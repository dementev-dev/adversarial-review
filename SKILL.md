---
name: adversarial-review
description: Adversarial AI code/plan review. Codex reviews, Claude fixes, iterative loop until approved. Auto-detects plan/code/code-vs-plan mode.
user_invocable: true
---

# Adversarial Code Review

> **Platform:** Claude Code only. This skill orchestrates Claude ↔ Codex interaction, where Claude is the executor and Codex is the external reviewer. Running this skill from Codex CLI itself creates a recursive loop — Codex would try to launch itself. If you are Codex — do NOT invoke this skill; perform the review directly.

Sends current work for adversarial review through an external AI model (OpenAI Codex by default). Auto-detects what to review: **plan** or **code**. Claude fixes issues based on reviewer feedback and resubmits until approved. Maximum 5 rounds.

---

## When to invoke

- `/adversarial-review` — auto-detect what to review
- `/adversarial-review plan` — force plan review
- `/adversarial-review code` — force code review
- `/adversarial-review <file-path>` — review a specific file (argument contains `/` or `.`)
- Override reasoning: `/adversarial-review xhigh` or `/adversarial-review medium` (one of: `medium`, `high`, `xhigh`)
- Override model: `/adversarial-review model:gpt-5.3-codex` (argument with `model:` prefix)

## Instructions

> **Placeholders:** `${REVIEW_ID}`, `${ATTEMPT_ID}`, `${CODEX_SESSION_ID}`, `${REPO_ROOT}`, and `${BASE_BRANCH}` in the steps below are template placeholders, NOT shell variables. Substitute literal values directly into each tool call. In particular:
> - `${REPO_ROOT}` is ALWAYS an absolute path captured at Step 2; never replace it with `$(pwd)`.
> - `${REVIEW_ID}` is stable for the entire review (used in file paths).
> - `${ATTEMPT_ID}` is a fresh 6-digit random integer generated **per launch** — a new value for the initial exec, for any retry of that exec, for every resume in Step 7, and for any fresh-exec fallback. The combined marker `${REVIEW_ID}-${ATTEMPT_ID}` is embedded in the prompt (HTML comment) so the filesystem session-id fallback identifies exactly THIS launch's rollout. Do NOT reuse a prior launch's ATTEMPT_ID — that would make multiple rollouts match and reintroduce silent session drift.

### Step 1: Determine review mode

Determine what to review. Check in priority order:

**1. Explicit argument** (`plan`, `code`, file path) → use it.
   - For `plan` → skip all git checks, proceed to step 2 (REVIEW_ID only).

**2. Claude Code Plan Mode** — if context contains the system message "Plan mode is active" → mode = `plan`, skip git. In Plan Mode code is not edited, so code/code-vs-plan are impossible.

**3. Auto-detect** (no explicit argument, not in Plan Mode):

1. Check for code changes (any non-empty output means changes exist):
   - `git diff --name-only` — unstaged
   - `git diff --cached --name-only` — staged
   - `git diff --name-only ${BASE_BRANCH}...HEAD` — branch commits
2. Check if a plan exists in the current conversation context (from plan mode, tasks, or discussion).

| Code changes? | Plan in context? | Mode |
|--------------|-----------------|------|
| No | Yes | **plan** — review the plan |
| Yes | Yes | **code-vs-plan** — review implementation against plan |
| Yes | No | **code** — review code changes |
| No | No | Ask the user what to review |

### Step 2: Generate Session ID, capture REPO_ROOT, determine base branch

**REVIEW_ID:** generate yourself, format `{unix_timestamp}-{random_8digit_number}`.
Example: `1711872000-48217593`. **Do NOT use bash** — substitute the value directly into commands in the following steps. 8-digit random makes collisions negligible (1 in 10^8 per same-second invocation).

**Capture REPO_ROOT:**

```bash
git rev-parse --show-toplevel
```

- **Exit 0, non-empty output** → absolute path. Save literally as `REPO_ROOT` (a template placeholder — substitute verbatim into codex commands; do NOT use `$(pwd)` anywhere).
- **Exit 128** (bare repo, or not in a work tree) → tell the user: `Cannot run adversarial review — current directory is not inside a git working tree.` Abort the skill.
- **Path contains single quote, double quote, `$`, backtick, newline** → tell the user: `REPO_ROOT path contains shell-special characters; cannot safely construct codex commands.` Abort.

**Submodule warning:** after capturing REPO_ROOT, run:

```bash
git rev-parse --show-superproject-working-tree
```

If this returns non-empty, the user is inside a git submodule. Tell the user: `You are inside a submodule. The review will be scoped to this submodule (${REPO_ROOT}), not the parent repo. If you meant to review the parent, invoke from there.` Proceed — this is a warning, not an abort.

**Determining base branch (only for `code` and `code-vs-plan` modes):**

For `plan` mode — skip base branch detection, proceed to step 3.

For other modes, determine the repository's base branch:

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||'
```

If the command returns empty (remote HEAD not configured), use fallback:

```bash
git rev-parse --verify main 2>/dev/null && echo main || echo master
```

Save the result as `BASE_BRANCH` — used in `git diff ${BASE_BRANCH}...HEAD` below.

### Step 3: Prepare review material

**Plan review:**

- If the plan already exists as a file (in `project/`, plan file from Plan Mode, memory, or somewhere in the repo) — use the path directly. Do NOT copy. In Claude Code Plan Mode the plan is always a file.
- If the plan is only in the conversation context (outside Plan Mode) — write via **Write tool** to `/tmp/codex-plan-${REVIEW_ID}.md`.
- **Always print the plan file path for the user** so they can open it in their IDE:
  `Plan for review: <file-path>`

**Code review:**

Collect the list of changed files:

1. `git diff --name-only` — unstaged changes
2. `git diff --cached --name-only` — staged changes

Merge unstaged + staged (unique paths). If both are empty:

3. `git diff --name-only ${BASE_BRANCH}...HEAD` — branch commits (fallback)

Branch diff is used ONLY when there are no local changes — otherwise context bloats.
For branch diff, include the command `git diff ${BASE_BRANCH}...HEAD` (full diff) in the prompt.

The reviewer has access to the repo and will read full diffs and files on its own.
In the prompt (step 4), pass the file list and which git diff commands to run.

**Many files (> 50):** if the combined list exceeds 50 paths,
pass only git commands without the file list — the reviewer will figure it out.

If all sources are empty — no changes to review, inform the user.

**Code-vs-plan review:** prepare the plan path AND collect the list of changed files (as above).

### Step 4: Build the prompt body, dispatch the runner subagent

Main thread composes the prompt BODY (without the session marker — the subagent adds it). Select the right template from below based on review mode.

**Prompt body for plan review:**

```
<role>
You are a senior adversarial reviewer of implementation plans.
Your job is to break confidence in the plan, not to validate it.
</role>

<operating_stance>
Default to skepticism. Assume the plan has gaps until the evidence says otherwise.
Do not give credit for good intent or likely follow-up work.
If something only works on the happy path, treat that as a real weakness.
</operating_stance>

<task>
Review the implementation plan in <plan-path>.
</task>

<attack_surface>
Check each area. Skip if not applicable:
- Feasibility — will this approach actually work given the codebase and constraints?
- Missing steps — what is forgotten or assumed but not stated?
- Risk areas — what could go wrong during implementation? Data loss? Downtime?
- Sequencing — are steps in the right order? Are there hidden dependencies?
- Alternatives — is there a simpler or more robust approach?
- Rollback — can this be safely reverted if it fails halfway?
- Security — auth, data exposure, injection, unsafe operations
</attack_surface>

<finding_bar>
Each finding MUST answer:
1. What can go wrong? (concrete scenario, not hypothetical)
2. Why is this plan vulnerable? (cite specific section)
3. Impact — what breaks and how badly?
4. Recommendation — specific change to the plan
</finding_bar>

<scope_exclusions>
DO NOT comment on: formatting, wording style, speculative issues without concrete trigger scenario.
</scope_exclusions>

<calibration>
Prefer one strong finding over several weak ones.
If the plan is solid, say so clearly — false positives erode trust.
</calibration>

<output_format>
Use markdown headers for sections: Summary, Findings, Verdict.

Summary: one paragraph — what this plan does and your overall assessment.

Findings: for each finding, use a sub-header with [severity: critical|high|medium] and title.
Include these fields per finding:
- **Section:** which part of the plan
- **What can go wrong:** ...
- **Why vulnerable:** ...
- **Impact:** ...
- **Recommendation:** ...

If no findings: "No actionable findings."

Verdict rules: approve if no findings or all low severity; revise if any high/critical.
Choose exactly one. The LAST line of your response must be one of:
VERDICT: APPROVED
VERDICT: REVISE
</output_format>
```

**Prompt body for code review (≤ 50 files):**

```
<role>
You are a senior adversarial code reviewer.
Your job is to break confidence in the change, not to validate it.
</role>

<operating_stance>
Default to skepticism. Assume the change can fail in subtle, high-cost,
or user-visible ways until the evidence says otherwise.
Do not give credit for good intent, partial fixes, or likely follow-up work.
If something only works on the happy path, treat that as a real weakness.
</operating_stance>

<task>
Review the code changes in this repo. Changed files:

<file list from --name-only>

Changes include: <unstaged changes / staged changes / unstaged + staged changes / branch changes vs ${BASE_BRANCH}>.
Run <git diff commands> to see the full diffs.
</task>

<attack_surface>
Check each area. Skip if not applicable to this change:
- Auth & permissions: bypasses, privilege escalation, missing checks
- Data integrity: loss, corruption, partial writes, constraint violations
- Race conditions: TOCTOU, concurrent access, deadlocks
- Rollback safety: can this change be safely reverted?
- Schema drift: migrations, backward compatibility, data format changes
- Error handling: swallowed errors, missing retries, cascading failures
- Observability: will operators know when this breaks?
</attack_surface>

<finding_bar>
Each finding MUST answer:
1. What can go wrong? (concrete scenario, not hypothetical)
2. Why is this code vulnerable? (cite specific file and lines)
3. Impact — what breaks and how badly? (data loss > downtime > degraded UX)
4. Recommendation — specific fix with code reference
</finding_bar>

<scope_exclusions>
DO NOT comment on: code style, formatting, naming conventions,
speculative issues without concrete trigger scenario,
"nice to have" improvements unrelated to correctness or safety.
</scope_exclusions>

<calibration>
Prefer one strong finding over several weak ones.
Severity: critical (data loss/security) > high (bug in prod) > medium (edge case).
If the change is solid, say so clearly — false positives erode trust.
</calibration>

<output_format>
Use markdown headers for sections: Summary, Findings, Verdict.

Summary: one paragraph — what this change does and your overall assessment.

Findings: for each finding, use a sub-header with [severity: critical|high|medium] and title.
Include these fields per finding:
- **File:** path/to/file.ext lines N-M
- **What can go wrong:** ...
- **Why vulnerable:** ...
- **Impact:** ...
- **Recommendation:** ...

If no findings: "No actionable findings."

Verdict rules: approve if no findings or all low severity; revise if any high/critical.
Choose exactly one. The LAST line of your response must be one of:
VERDICT: APPROVED
VERDICT: REVISE
</output_format>
```

**Prompt body for code review (> 50 files):**

Same as ≤ 50 files above, but the `<task>` section is replaced with:

```
<task>
Review the code changes in this repo.
Changes include: <unstaged changes / staged changes / ...>.
Run <git diff commands> to see changed files and full diffs.
</task>
```

**Prompt body for code-vs-plan review:**

Same as code review (≤ 50 or > 50 variant depending on file count), but:
- `<task>` is extended to reference the plan file: `Review the code changes in this repo against the implementation plan in <plan-path>.`
- `<attack_surface>` appends these three items:
  ```
  - Completeness: does the implementation cover all plan steps?
  - Deviations: where does the code differ from the plan? Are deviations justified?
  - Missing: what from the plan is not yet implemented?
  ```

**Substitute template placeholders BEFORE writing to disk:**

The inlined prompt bodies above contain template placeholders that main must resolve with real captured values before the Write. Placeholders per mode:

| Placeholder | Value source | Applies to |
|---|---|---|
| `${BASE_BRANCH}` | captured at Step 2 (code & code-vs-plan only) | code, code-vs-plan |
| `<plan-path>` | captured at Step 3 | plan, code-vs-plan |
| `<file list from --name-only>` | result of `git diff --name-only` + `git diff --cached --name-only` (or branch diff) from Step 3 | code, code-vs-plan (≤50 files only) |
| `<unstaged changes / staged changes / ...>` | human-readable description derived from which diff commands had content | code, code-vs-plan |
| `<git diff commands>` | the exact commands main determined at Step 3 (e.g. `git diff`, `git diff --cached`, `git diff ${BASE_BRANCH}...HEAD`) | code, code-vs-plan |

Substitute `${BASE_BRANCH}` first (it appears nested inside `<unstaged changes / staged changes / ...>`), then compute the outer human-readable description based on which diffs have content. Main writes the substituted string to the Write tool — no template placeholders should remain in the body file sent to the runner.

**Capture user overrides for `CODEX_MODEL` / `CODEX_REASONING` at Step 1:**

The skill supports overrides like `/adversarial-review xhigh`, `/adversarial-review medium`, `/adversarial-review model:gpt-5.3-codex`. At Step 1, capture:

- `CODEX_MODEL` — default `gpt-5.4`. Overridden by any argument matching `^model:(.+)$`; use the capture group.
- `CODEX_REASONING` — default `high`. Overridden by any argument exactly matching `low`, `medium`, `high`, or `xhigh`.

These are passed into the runner YAML input block below.

**Write the prompt body to disk via Write tool:**

Write `/tmp/codex-body-${REVIEW_ID}.md` containing the substituted body text (no session marker — the runner adds it).

> **Plan Mode note:** Writing to `/tmp` via Write tool may trigger a permission prompt or exit Plan Mode. This is a known Claude Code limitation. Additionally, dispatching a subagent under Plan Mode may inherit the restriction — empirical behavior documented in DESIGN.md §12.7.

**Resolve the runner spec path:**

The runner spec lives at `references/runner.md` within the skill's install directory. Main cannot reliably introspect Claude Code's skill-invocation header from inside its own context (there is no tool for reading one's own system prompt — any attempt would be a hallucination risk). Therefore the discovery uses only concrete filesystem checks, in this priority order:

1. **User-scoped install** (primary): check `~/.claude/skills/adversarial-review/references/runner.md`:

```bash
ls ~/.claude/skills/adversarial-review/references/runner.md 2>/dev/null
```

If exit 0, set `RUNNER_SPEC_PATH` to the expanded absolute path and proceed.

2. **Plugin-marketplace install** (secondary): Claude Code's plugin system installs skills at paths like `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/skills/adversarial-review/`. Glob to find it:

```bash
ls ~/.claude/plugins/cache/*/*/*/skills/adversarial-review/references/runner.md 2>/dev/null | head -1
```

If the Glob returns one or more paths, take the first and set `RUNNER_SPEC_PATH`.

3. **Dev checkout** (tertiary): if neither above, try `$(git rev-parse --show-toplevel)/references/runner.md`:

```bash
REPO=$(git rev-parse --show-toplevel 2>/dev/null) && ls "$REPO/references/runner.md" 2>/dev/null
```

4. **Abort**: if no path yields a readable file, tell the user: `Could not locate references/runner.md. Expected locations: (1) ~/.claude/skills/adversarial-review/references/runner.md, (2) ~/.claude/plugins/cache/*/*/*/skills/adversarial-review/references/runner.md, (3) $(git rev-parse --show-toplevel)/references/runner.md. Re-install the skill.` Abort the skill.

Save the resolved absolute path as `RUNNER_SPEC_PATH`.

**Dispatch the runner subagent via Agent tool:**

Read `${RUNNER_SPEC_PATH}` once (main thread). This is the subagent's full spec.

Invoke the Agent tool with:
- `subagent_type: "general-purpose"`
- `model: "haiku"`
- `description: "Adversarial-review runner, round N"` (N is the current round number)
- `prompt:` the concatenation of:
  1. The full text of `references/runner.md` you just read (as instructions).
  2. A blank line.
  3. A literal YAML input block with the values substituted (use `CODEX_MODEL` and `CODEX_REASONING` — the codex-CLI model/effort; distinct from the subagent's own Haiku model):

```yaml
---
REVIEW_ID: 1711872000-48217593
REPO_ROOT: /home/dementev/sources/myproject
OPERATION: initial
CODEX_MODEL: gpt-5.4
CODEX_REASONING: high
PROMPT_BODY_PATH: /tmp/codex-body-1711872000-48217593.md
RESULT_PATH: /tmp/codex-runner-result-1711872000-48217593.json
---
```

Substitute real values. `RESULT_PATH` always follows the pattern `/tmp/codex-runner-result-${REVIEW_ID}.json`.

**Do NOT run the Agent tool call in background.** Wait for the subagent to return. (Runner's own codex exec is also synchronous per runner Step R3.)

**Parse the subagent's response — two-channel protocol:**

Apply the regex `RUNNER_RESULT_AT:\s+(\S+)` (UNANCHORED — matches anywhere in the Agent tool's result text, tolerant of markdown fences and preamble). Take the first match's capture group as the result-file path.

If the regex finds NO match in the subagent's response, fall back to a Glob for the deterministic path `/tmp/codex-runner-result-${REVIEW_ID}.json` — REVIEW_ID is already known to main. If Glob also returns nothing, treat as `infra_error` with `errors: "runner did not write result file at deterministic path and did not emit RUNNER_RESULT_AT line"` and abort.

Read the file at the resolved path. Parse as JSON. Extract `result`, `verdict`, `review_file`, `codex_session_id`, `errors`, `user_warning`, `archived_stdout`, `archived_stderr`.

**If `user_warning` is non-null, surface it as a SEPARATE short user-visible message BEFORE the Step 5 verbatim-review message.** Format:

```
⚠ <user_warning contents>
```

Emit this on its own turn — do NOT concatenate into the Step 5 `## Adversarial Review — Round N` header message (that message's body must remain the review's verbatim content, nothing else). Emit the warning FIRST, then the Step 5 message. This preserves both the pre-refactor §2.4.4 "no-op refresh" diagnostic AND the Step 5 verbatim-display contract.

Dispatch based on `result`:

| `result` value | Main thread action |
|---|---|
| `success` | Save `codex_session_id` (keep prior if `null` per §2.4.4). Surface `user_warning` if set. Proceed to Step 5. |
| `timeout` | **TERMINAL — do NOT re-dispatch.** Runner already attempted twice internally (R4.1 + R5 retry = 2 × 10min). Tell user: "Reviewer timed out after two attempts (20 minutes total)." Abort the skill. User can re-invoke `/adversarial-review` to start a fresh review. |
| `launch_failure` | **TERMINAL — do NOT re-dispatch.** The runner already retried once internally (Step R5). Show `errors` to user, abort the skill. This keeps the total-attempts-per-round invariant at 2 (matches pre-refactor: 1 initial + 1 retry). |
| `infra_error` | Show `errors` to user (infrastructure: /tmp not writable, stderr file missing, RUNNER_RESULT_AT line absent). Abort. |
| `input_error` | Bug in orchestration. Show `errors` to user. Abort. |

**Round-level attempt invariant:** exactly ONE runner dispatch per round. Every failure result is terminal at main. The runner owns the full retry budget (≤2 attempts per dispatch, internal) regardless of failure type. Total codex invocations per round ≤ 2.

> **CRITICAL — main thread does NOT read stdout/stderr/JSONL/rollout files BY CONTENT.** Those live and die inside the subagent. Main reads: the runner result JSON at `RESULT_PATH`, the review file at `review_file`, and nothing else from `/tmp/codex-*`. Archival `mv` (on resume failure) is done by the runner, not main — main never references `/tmp/codex-stdout-*` or `/tmp/codex-stderr-*` in any Bash argv.

### Step 5: Read the review, show it, then check the verdict

**1. Read the review file.** Read `/tmp/codex-review-${REVIEW_ID}.md`.

**2. Semantic sanity checks.** The file MUST pass all of these:

- Exists and is non-empty.
- Contains a line exactly matching `^VERDICT: (APPROVED|REVISE)$`.
- If verdict is `REVISE` → file must also contain at least one line matching `\[severity:\s*(critical|high|medium)` (i.e. at least one structured finding).

If any check fails → this is a **launch failure** (model produced no actionable review):

- Show the user the `/tmp/codex-stderr-${REVIEW_ID}.txt` contents (if any) AND the raw review file.
- Offer ONE retry of Step 4 (re-launch the same round). Retry does NOT consume the round counter — the round counter advances only when a valid review is produced.
- Track the retry counter in your **current round's** reasoning only. The counter resets at the start of every new round.
- After a failed retry → hard abort the skill. Do NOT route to the Step 7 fresh-exec fallback (that path is for resume failures in rounds 2+, and depends on prior-round content).

**3. Show the review to the user. This is mandatory and blocking.**

> Your next user-visible message that is NOT a one-line `⚠ <warning>` diagnostic must begin with the header below, followed by the file contents **verbatim**. Not "I've received the review", not "The reviewer said:", not a summary — the literal file content.
>
> Do NOT wrap the review in a code fence (the review is already markdown, and an outer fence would break on inner fences).
>
> Do NOT call any Edit, Write, or fix-applying tool in the same message as the review. The review output is a standalone user-visible message.

Message format:

```
## Adversarial Review — Round N (mode: <plan|code|code-vs-plan>, model: gpt-5.4)

<verbatim contents of /tmp/codex-review-${REVIEW_ID}.md>
```

**4. Only AFTER the review message has been sent** — parse the VERDICT line and dispatch:

- `VERDICT: APPROVED` → Step 8 (Done).
- `VERDICT: REVISE` → Step 6 (Fixes).
- Maximum rounds reached (5 rounds) → Step 8 with the max-rounds note.

### Step 6: Apply fixes

> **Precondition gate (check first).** Before calling any Edit, Write, or other fix-applying tool: confirm that you have already sent a user-visible message in THIS round whose body contains the verbatim review text (short `⚠ <user_warning>` diagnostic messages do NOT count). If you have not — STOP. Go back to Step 5 and send the review message now. This is the same rule that protects the "user sees the review" contract; a literal reader may otherwise slip past it.

Based on the reviewer's findings:

**For plan review:** fix the plan — address each finding. Update the plan file (or temp file). Show the user:

```
### Fixes (Round N)
- [What was changed and why, one item per finding]
```

**For code review:** fix the code directly — edit files, run tests if applicable. Show the user:

```
### Fixes (Round N)
- [What was fixed and why, one item per finding]
```

**Skip** a fix if it contradicts the user's explicit requirements — note this for the user.

### Step 7: Resubmit to Codex (Rounds 2-5)

**Resume is the primary path.** Saves tokens and preserves session context. A fresh `codex exec` without resume is an **emergency fallback** — costly in tokens, and requires rebuilding prior-round context.

**1. Write the resume prompt** to `/tmp/codex-resume-prompt-${REVIEW_ID}.md` via **Write tool**. Use a separate file from the initial prompt so round-1 material remains available for diagnostics. **Generate a fresh `${ATTEMPT_ID}` for this resume launch** (different from the initial exec's ATTEMPT_ID and from every prior resume's ATTEMPT_ID). The resume prompt must begin with the per-launch marker:

```
<!-- ADVERSARIAL-REVIEW-SESSION: ${REVIEW_ID}-${ATTEMPT_ID} -->

I've revised based on your feedback.

Here's what I changed:
[List of fixes]

Re-review with the same adversarial stance. Focus on:
1. Whether my fixes actually resolve the reported issues
2. Any NEW issues introduced by the fixes

End with VERDICT: APPROVED or VERDICT: REVISE
```

**2. Run resume.** Resume does NOT accept `-C`, so prefix the command with an explicit `cd` to `${REPO_ROOT}` (captured at Step 2). Use single quotes around `${REPO_ROOT}` — the path was validated at Step 2 to contain no single quotes.

The resume prompt file (`/tmp/codex-resume-prompt-${REVIEW_ID}.md`) acts as the anchor for this resume's filesystem fallback, the same way the initial prompt file anchors Step 4. Launch resume via the `cat | ... -` pattern:

```bash
cd '${REPO_ROOT}' && cat /tmp/codex-resume-prompt-${REVIEW_ID}.md | timeout 600 codex exec resume --json \
  ${CODEX_SESSION_ID} \
  -o /tmp/codex-review-${REVIEW_ID}.md \
  - \
  > /tmp/codex-stdout-${REVIEW_ID}.jsonl \
  2>/tmp/codex-stderr-${REVIEW_ID}.txt
```

Use `timeout: 620000` in Bash tool parameters.

**Note:** Resume does NOT accept `-s` (sandbox — inherited from the original session; always `read-only` here) or `-C` (see above). It DOES accept `--json`, `-o`, `-m`, and `-i`.

**3. Post-resume strict check order (do each before moving to the next):**

1. **Exit code.**
   - `124` → timeout. Tell the user and offer retry. Retry does not consume the round counter.
   - `≠ 0` → resume failed. Do NOT update `CODEX_SESSION_ID`. Route to fallback.
   - `0` → proceed.

2. **Stderr error check** (exit 0 can hide `Error:` or `thread/resume failed`). Read `/tmp/codex-stderr-${REVIEW_ID}.txt`:
   - If file is missing → redirect failed; tell user `/tmp not writable`, abort.
   - If contains a line matching `thread/resume failed` or `^Error:` → route to fallback. Do NOT update `CODEX_SESSION_ID`.

3. **Review file sanity.** Read `/tmp/codex-review-${REVIEW_ID}.md` and apply the same checks as Step 5.2:
   - Missing / empty / no `^VERDICT: (APPROVED|REVISE)$` line / REVISE without `[severity:` lines → route to fallback. Do NOT update `CODEX_SESSION_ID`.

**4. Only if all three checks pass AND the verdict is REVISE** → refresh `CODEX_SESSION_ID` using two tiers (primary = first JSONL line of `/tmp/codex-stdout-${REVIEW_ID}.jsonl`; secondary = rollout file that is both newer than `/tmp/codex-resume-prompt-${REVIEW_ID}.md` AND contains the `ADVERSARIAL-REVIEW-SESSION: ${REVIEW_ID}-${ATTEMPT_ID}` marker for THIS resume's ATTEMPT_ID, with UUID extracted from the basename — same positive-binding approach as Step 4 check 4 but anchored on the resume prompt). On APPROVED verdict, skip the refresh — there is no round N+1.

> **Important — NOT identical to Step 4 check 4 on the failure side.** Step 4 check 4 treats "no matching rollout" as a launch failure because in Step 4 the session id is needed for resume to even happen. In Step 7 the resume has **already succeeded** (checks 1-3 passed), and per `DESIGN.md §2.4.4` the thread id does not rotate across resumes — so if both tiers yield nothing here, **do NOT abort and do NOT retry**: keep the previous `CODEX_SESSION_ID` unchanged, log a one-line warning to the user (`"Step 7 session-id refresh: both tiers empty, continuing with previous ID per §2.4.4"`), and continue to Step 5.

After the refresh (or the no-op refresh on zero-find), return to **Step 5** with the new review.

---

**Fallback chain** — triggered when any of the three resume checks above fails.

> `--last` is deliberately NOT used. `codex exec resume --last` picks the newest session in the current cwd, which may be an unrelated codex invocation and cannot be distinguished from the intended one until after damage is done.

**Severity classification** — parse the **previous** round's review file (which is still in `/tmp/codex-review-${REVIEW_ID}.md` only if the resume overwrote the current-round result but not the previous-round; in general, rely on **conversation history** where prior rounds were shown verbatim per Step 5.3).

Parse case-insensitively for `\[severity:\s*(critical|high|medium)\b` and take the highest. If zero matches (reviewer format drift), default to `critical` to force re-verification in non-interactive mode.

**Interactive mode** (you received a direct user message earlier in this session, not a trigger/cron):

Ask the user:

```
Resume failed — the reviewer's re-review did not produce a usable result.
Last round's maximum severity: <level>.

Options:
(a) Run a fresh `codex exec` with full previous-rounds context (higher token cost, new session)
(b) Conclude the review — show current findings as NOT VERIFIED
```

- (a) → fresh-exec path below.
- (b) → Step 8 with the **not-verified** terminal state (same as maximum-reached, but with a different header).

**Non-interactive mode** (headless, scheduled run, no direct user message in this conversation):

- Max severity `critical` or `high` → fresh exec automatically. The risk of silently skipping a serious finding outweighs the token cost.
- Max severity `medium` only → Step 8 with the **not-verified** terminal state.

**Fresh-exec prompt template.** The lead rebuilds prior-round context from the conversation (all prior rounds were shown verbatim in Step 5.3 user messages, so they are available in context). Generate a fresh `${ATTEMPT_ID}` for this fresh-exec launch, then begin the prompt with the per-launch marker:

```
<!-- ADVERSARIAL-REVIEW-SESSION: ${REVIEW_ID}-${ATTEMPT_ID} -->
[Original adversarial prompt for the current mode, from Step 4]

## Previous review rounds

### Round 1 findings (verbatim from earlier in this conversation):
<copy the round-1 review text that you already output as a user message>

### Round 1 fixes:
<copy the round-1 fixes list you output in Step 6>

### Round 2 findings (verbatim):
<...>

### Round 2 fixes:
<...>

## Current state of the artifact
[plan mode] Full current plan text: <insert>
[code mode] Run `git diff` from ${REPO_ROOT} to see the current changes.

Re-review. Focus on whether prior fixes resolved the reported issues and on any NEW issues introduced by the fixes.

End with VERDICT: APPROVED or VERDICT: REVISE.
```

**Archive failed-resume diagnostics BEFORE launching the fresh exec** (the fresh exec reuses the same stderr/stdout paths and would overwrite them):

```bash
mv /tmp/codex-stdout-${REVIEW_ID}.jsonl /tmp/codex-stdout-${REVIEW_ID}-failed-resume.jsonl 2>/dev/null
mv /tmp/codex-stderr-${REVIEW_ID}.txt /tmp/codex-stderr-${REVIEW_ID}-failed-resume.txt 2>/dev/null
```

If the fresh exec later needs investigating, both the failed-resume trail (`*-failed-resume.*`) and the fresh-exec trail (the unsuffixed files) survive side-by-side. Cleanup at Step 9 removes both (the cleanup glob `/tmp/codex-*-${REVIEW_ID}*` covers the suffixed variants).

Write the fresh-exec prompt to `/tmp/codex-prompt-${REVIEW_ID}.md` (overwriting the original is acceptable).

Launch using the **same command template as Step 4** (`cat file | timeout 600 codex exec --json ... -` with `-C`, `-o`, stdout jsonl, stderr). Because this fresh-exec path **overwrites** `/tmp/codex-prompt-${REVIEW_ID}.md` with new content just written above, that file's mtime is automatically the post-write moment — it serves as the `-newer` anchor for the two-tier secondary session-id capture on the fresh exec's rollout, the same way Step 4 uses it on the initial exec's rollout. Apply the same post-launch strict check order including the two-tier session-id capture, then return to **Step 5**.

> This fresh exec consumes one round from the 5-round counter — same as a successful resume would have.

### Step 8: Final result

**Approved:**
```
## Adversarial Review — Summary (mode: <mode>, model: gpt-5.4)

**Status:** Approved after N round(s)

[Final review]

---
**Reviewed and approved by the reviewer. Awaiting your decision.**
```

**Maximum rounds reached:**
```
## Adversarial Review — Summary (mode: <mode>, model: gpt-5.4)

**Status:** Maximum reached (5 rounds) — not fully approved

**Remaining findings:**
[Unresolved issues]

---
**The reviewer still has findings. Please review them and decide how to proceed.**
```

**Not verified** (resume failed and the operator chose to conclude, or headless with only medium severity):
```
## Adversarial Review — Summary (mode: <mode>, model: gpt-5.4)

**Status:** NOT VERIFIED — fixes applied, reviewer did not re-verify

**Last round's findings:**
[Verbatim findings from the last successful round]

**Applied fixes:**
[List of fixes per finding]

---
**WARNING: This is NOT an approval. Fixes were applied but never verified by the reviewer. Manual review is required before merging.**
```

### Step 9: Cleanup

**Conditional on terminal state:**

| Terminal state | Cleanup behavior |
|---|---|
| Approved | Remove all temp files |
| Maximum rounds reached | Remove all temp files |
| Not verified (fallback conclude) | Remove all temp files |
| Aborted (launch failure, redirect failure, infrastructure error) | **LEAVE files in place** for diagnostics |

**In Claude Code Plan Mode:** skip all cleanup (including deferred). `rm` will trigger a permission prompt. Files will be cleaned up on the next invocation outside Plan Mode.

**Outside Plan Mode, on a cleanup-eligible terminal state:**

```bash
rm -f /tmp/codex-plan-${REVIEW_ID}.md \
      /tmp/codex-prompt-${REVIEW_ID}.md \
      /tmp/codex-resume-prompt-${REVIEW_ID}.md \
      /tmp/codex-review-${REVIEW_ID}.md \
      /tmp/codex-stdout-${REVIEW_ID}.jsonl \
      /tmp/codex-stderr-${REVIEW_ID}.txt \
      /tmp/codex-stdout-${REVIEW_ID}-failed-resume.jsonl \
      /tmp/codex-stderr-${REVIEW_ID}-failed-resume.txt
```

If the user declined `rm` — continue without error.

Do NOT delete plan files that existed before the review (only temp files created by this skill). On abort paths, old temp files remain for diagnostics and will be cleaned up by the OS on reboot, or overwritten by the next invocation using the same REVIEW_ID (collision probability is ~10⁻⁸ per same-second run).

## Rules

- Claude **actively fixes** issues based on reviewer feedback — this is NOT just message forwarding.
- Reviewer findings are shown **verbatim** — do not rephrase or shorten. The Step 5 "YOUR NEXT MESSAGE" instruction is blocking: no edit/fix tool may be called until that message has been sent.
- Auto-detect review mode from context; user arguments take priority.
- With explicit `plan` argument or in Claude Code Plan Mode: skip git checks and base branch detection.
- **`REPO_ROOT` is captured at Step 2** via `git rev-parse --show-toplevel` and substituted as an absolute literal path into every codex command. Never use `$(pwd)` inside codex commands — cwd drift between Bash calls makes it unreliable.
- **Resume requires `cd '${REPO_ROOT}' && ...`** because `codex exec resume` has no `-C` flag; cwd is inherited from the shell. The initial exec uses `-C "${REPO_ROOT}"` instead.
- **`CODEX_SESSION_ID` is updated only on full success** — ALL of (exit=0 AND stderr has no `Error:`/`thread/resume failed` line AND review file contains a valid `VERDICT:` line with findings on REVISE). On any failure, leave it unchanged and route to the fallback.
- **Session ID capture is two-tier.** Primary: `thread_id` from the first JSONL line of stdout. Secondary (primary empty / malformed / missing `thread_id`): the rollout file that is both `-newer` than the prompt file AND contains the `ADVERSARIAL-REVIEW-SESSION: ${REVIEW_ID}-${ATTEMPT_ID}` marker for THIS launch, with UUID from the filename. Positive content-binding eliminates wrong-session hazard from both parallel codex invocations AND same-review retries.
- **Every prompt starts with `<!-- ADVERSARIAL-REVIEW-SESSION: ${REVIEW_ID}-${ATTEMPT_ID} -->` as its first line.** Generate a **fresh 6-digit ATTEMPT_ID per launch** (initial exec, any retry of that exec, each resume in Step 7, any fresh-exec fallback). Never reuse an ATTEMPT_ID within the same review — doing so would let a prior attempt's rollout match the current launch's grep, reintroducing session drift.
- **Secondary-path multi-match is fail-closed, not pick-any.** If the `find ... -exec grep -l ... {} +` returns two or more rollout paths for a single `${REVIEW_ID}-${ATTEMPT_ID}`, abort the round with a diagnostic. This should not happen under correct attempt-scoping; if it does, something is structurally wrong and silent picking would mask it.
- **Prompt delivery is `cat file | codex exec ... -`.** The `- < file` stdin-redirect form is accepted by codex but exits 1 with empty stderr in some Claude Code sandbox configurations. Pipe is portable across both envs observed.
- **The `--json` stdout stream is never human-readable review text** — JSONL events when populated, empty when suppressed by sandbox. Never treat Bash result as review content; the review lives exclusively in `/tmp/codex-review-*.md`.
- **Launch-failure retry** is capped at 1 per round and does NOT consume the 5-round counter. The retry counter is per-round; it resets at the start of every new round and is tracked only in that round's reasoning.
- **Resume is the primary path for rounds 2-5.** Fresh exec is a fallback that runs only when resume fails; it consumes one round from the counter just as a successful resume would.
- **`--last` is never used** — cwd filtering is insufficient to distinguish the current skill session from unrelated parallel codex invocations in the same repo.
- **Fallback after resume failure:** interactive → ask the user (fresh exec vs conclude as not-verified); non-interactive → auto fresh exec if max severity is critical/high, auto conclude-as-not-verified if only medium.
- Cleanup is **conditional on terminal state**: remove temp files on approved/max-reached/not-verified; LEAVE them on abort (diagnostic value). Skip all cleanup in Plan Mode.
- Always read-only sandbox — reviewer never writes files.
- Maximum 5 rounds to protect against infinite loops.
- Show the user reviews and fixes for each round.
- If Codex CLI is not installed or crashed — tell the user: `npm install -g @openai/codex`.
- If a fix contradicts the user's explicit requirements — skip and explain why.
