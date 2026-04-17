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

> **Placeholders:** `${REVIEW_ID}`, `${CODEX_SESSION_ID}`, `${REPO_ROOT}`, and `${BASE_BRANCH}` in the steps below are template placeholders, NOT shell variables. Substitute literal values directly into each tool call. In particular, `${REPO_ROOT}` is ALWAYS an absolute path captured at Step 2; never replace it with `$(pwd)`.

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

### Step 4: Build the prompt and launch the first round

Build the prompt depending on the mode. All prompts use the adversarial stance.

**Prompt for plan review:**

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

**Prompt for code review (<= 50 files):**

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

**Prompt for code review (> 50 files):**

Same prompt as above, but the `<task>` section without the file list:
```
<task>
Review the code changes in this repo.
Changes include: <unstaged changes / staged changes / ...>.
Run <git diff commands> to see changed files and full diffs.
</task>
```

**Prompt for code-vs-plan review:**

Same prompt as code review, but the `<task>` section is extended:
```
<task>
Review the code changes in this repo against the implementation plan in <plan-path>.
Changed files:

<file list or empty if > 50>

Changes include: <type>.
Run <git diff commands> to see the full diffs.
</task>
```

And the following items are added to `<attack_surface>`:
```
- Completeness: does the implementation cover all plan steps?
- Deviations: where does the code differ from the plan? Are deviations justified?
- Missing: what from the plan is not yet implemented?
```

**Launching Codex — command template:**

Flags:
- `--json` — stdout becomes JSONL events (primary path for session-ID capture). In some sandbox configurations this stream ends up empty; the filesystem fallback in check 3 below handles that case.
- `-m gpt-5.4` — model (overridden by `model:...` argument)
- `-c model_reasoning_effort=high` — reasoning depth (overridden by `xhigh`, `low`, etc.)
- `-s read-only` — reviewer only reads, does not write
- `-C "${REPO_ROOT}"` — pin codex workdir to absolute repo root
- `-o /tmp/codex-review-${REVIEW_ID}.md` — file for capturing final agent text

**Prompt delivery:** write the prompt to `/tmp/codex-prompt-${REVIEW_ID}.md` via **Write tool**, then feed it to codex via `cat file | codex exec ... -`. This avoids shell quoting issues with long XML prompts and is environment-portable (the alternative `- < file` stdin-redirect form is accepted by codex but fails with `EXIT=1` in some Claude Code sandbox configurations).

**Plan Mode note:** Writing to `/tmp` via Write tool may trigger a permission prompt or exit Plan Mode. This is a known Claude Code limitation — Plan Mode restricts edits to the plan file only. If this happens, it does not affect review correctness: the review mode is already determined, and the skill only edits the plan file and `/tmp` temp files.

**Capture pre-exec timestamp** (for filesystem fallback of session-id; see check 3 below). Substitute the value literally:

```bash
date +%s
```

Save as `CODEX_SESSIONS_BEFORE` (a template placeholder — a Unix timestamp as an integer).

```bash
cat /tmp/codex-prompt-${REVIEW_ID}.md | timeout 600 codex exec --json \
  -m gpt-5.4 \
  -c model_reasoning_effort=high \
  -s read-only \
  -C "${REPO_ROOT}" \
  -o /tmp/codex-review-${REVIEW_ID}.md \
  - \
  > /tmp/codex-stdout-${REVIEW_ID}.jsonl \
  2>/tmp/codex-stderr-${REVIEW_ID}.txt
```

> **CRITICAL — the Bash tool result is NOT the review.** stdout is redirected to `/tmp/codex-stdout-${REVIEW_ID}.jsonl` (machine-readable JSONL events when populated, empty when the sandbox suppresses it — either way, never human-readable review text). The human-readable review exists ONLY in `/tmp/codex-review-${REVIEW_ID}.md`. Do not attempt to extract review text from the Bash result — there is none.

**Important:**
- Always wrap `codex exec` in `timeout 600` (10 minutes). If Codex hangs — the command exits with code 124.
- Use `timeout: 620000` parameter in Bash tool for headroom.
- The command is **synchronous**: when it returns, all four files (`-o`, stdout jsonl, stderr, prompt) are in their final state. Do **NOT** use a poll-loop.
- With `--json`, stderr is empty on success. It contains content only on errors (e.g. "Failed to write last message file ..."). Use stderr for diagnostics, NOT for session-id capture.

**Post-launch strict check order (do each before moving to the next):**

1. **Exit code.**
   - `124` → timeout. Tell the user "Reviewer did not respond within 10 minutes" and offer retry. Retry does NOT consume the round counter; max 1 retry per round.
   - `≠ 0 and ≠ 124` → launch error. Read `/tmp/codex-stderr-${REVIEW_ID}.txt` (if it exists), show its contents to the user, abort the skill.
   - `0` → proceed.

2. **Stderr sanity (even on exit 0).** Read `/tmp/codex-stderr-${REVIEW_ID}.txt`.
   - If file missing → redirect itself failed; tell user `Could not create stderr file — check /tmp writability`, abort.
   - If file contains a line matching `^Error:` or `Failed to write` → codex reported an infrastructure failure despite exit 0. Show stderr to user, route to launch-failure retry (max 1 per round; after retry failure → hard abort).
   - Otherwise → proceed.

3. **Capture `CODEX_SESSION_ID` — two-tier.**

   **What you are looking for.** A UUID string (format `[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}`) that identifies the codex session, so `codex exec resume <UUID>` in Step 7 can continue this conversation. Try the cheap source first, fall back to the filesystem only if needed.

   **Primary: first line of JSONL stdout.** Read `/tmp/codex-stdout-${REVIEW_ID}.jsonl`. If non-empty, the first line has the shape:
   ```json
   {"type":"thread.started","thread_id":"<uuid>","...":...}
   ```
   Parse it as JSON, take `thread_id`, save as `CODEX_SESSION_ID`, proceed to check 4.

   **Secondary: rollout filename.** In some Claude Code sandbox configurations the `--json` stdout file is empty (0 bytes) even when the review completes successfully (`-o` is populated, exit 0, stderr clean). In that case the session is still recoverable from disk: every `codex exec` writes a rollout file named `rollout-<ISO-timestamp>-<UUID>.jsonl` under `~/.codex/sessions/YYYY/MM/DD/` (see `DESIGN.md §2.3`). The trailing UUID in the filename is the session id.

   Run:

   ```bash
   find ~/.codex/sessions -name 'rollout-*.jsonl' -newermt "@${CODEX_SESSIONS_BEFORE}" -printf '%T@ %f\n' 2>/dev/null
   ```

   This prints zero or more lines of `<epoch-mtime> <filename>` for rollout files created after the pre-exec timestamp. From the result:

   - **Zero lines** → no rollout file was created; treat as launch failure, show stderr, retry once, then abort.
   - **One or more lines** → pick the line with the largest epoch-mtime (there is usually just one; multiple lines indicate a parallel codex invocation in the same second). Extract the trailing UUID from that filename (the 36-char hex-and-dashes pattern above) and save as `CODEX_SESSION_ID`.

   A single `find` invocation keeps the permission rule simple (`Bash(find ~/.codex/sessions*)`) and the parsing stays in your head — no shell pipeline needed.

   **Platform note.** `-newermt "@<epoch>"` and `-printf` are GNU extensions. On macOS (BSD find) they are unsupported — substitute an equivalent that achieves the same goal: "list rollout files modified since `${CODEX_SESSIONS_BEFORE}`, newest first". For example, `find ~/.codex/sessions -name 'rollout-*.jsonl' -type f` plus `stat -f '%m %N' <path>` per result, or `ls -t ~/.codex/sessions/*/*/*/rollout-*.jsonl` and filter by a reference file's mtime. The goal is what matters, not the exact flags.

   **Parallel-codex caveat:** if you happened to pick a rollout from a parallel codex invocation, Step 7's resume will either succeed against the wrong session (detected later via VERDICT / severity checks) or fail at the stderr/exit-code check and route to the standard fallback (§4.11). Either outcome is recoverable.

4. **Review file sanity.** (Performed again in Step 5, but note upfront.) `/tmp/codex-review-${REVIEW_ID}.md` must exist and contain a line matching `^VERDICT: (APPROVED|REVISE)$`. If not → Step 5 will handle it via retry/abort.

**Where `thread_id` / session id is NOT:**

- NOT in `/tmp/codex-review-${REVIEW_ID}.md` (only contains the final agent text)
- NOT in stderr file under `--json` (empty on success; error text only on failure)
- NOT in the middle or tail of stdout — only the **first line** of the JSONL file (when it is populated at all)

**Notes:**
- Default model: `gpt-5.4` with `model_reasoning_effort=high`. User can override via arguments.
- Always `-s read-only` — reviewer must not write files.
- Do **NOT** run in background.

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

> Your VERY NEXT MESSAGE to the user must begin with the header below, followed by the file contents **verbatim**. Not "I've received the review", not "The reviewer said:", not a summary — the literal file content.
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

> **Precondition gate (check first).** Before calling any Edit, Write, or other fix-applying tool: confirm that you have already sent a user-visible message in THIS round whose body contains the verbatim review text. If you have not — STOP. Go back to Step 5 and send the review message now. This is the same rule that protects the "user sees the review" contract; a literal reader may otherwise slip past it.

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

**1. Write the resume prompt** to `/tmp/codex-resume-prompt-${REVIEW_ID}.md` via **Write tool**. Use a separate file from the initial prompt so round-1 material remains available for diagnostics.

```
I've revised based on your feedback.

Here's what I changed:
[List of fixes]

Re-review with the same adversarial stance. Focus on:
1. Whether my fixes actually resolve the reported issues
2. Any NEW issues introduced by the fixes

End with VERDICT: APPROVED or VERDICT: REVISE
```

**2. Run resume.** Resume does NOT accept `-C`, so prefix the command with an explicit `cd` to `${REPO_ROOT}` (captured at Step 2). Use single quotes around `${REPO_ROOT}` — the path was validated at Step 2 to contain no single quotes.

Capture a pre-resume timestamp for the secondary session-id fallback (same pattern as Step 4):

```bash
date +%s
```

Save as `CODEX_SESSIONS_BEFORE`. Then launch resume via the same `cat | ... -` pattern:

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

**4. Only if all three checks pass** → refresh `CODEX_SESSION_ID` with the same two-tier approach from Step 4 check 3 (primary = first JSONL line of `/tmp/codex-stdout-${REVIEW_ID}.jsonl`; secondary = newest rollout filename with mtime > `CODEX_SESSIONS_BEFORE`, UUID extracted from the basename).

Per `DESIGN.md §2.4.4`, successful resume does not rotate the thread id — the new value equals the previous one, so this refresh is defensive. If both tiers yield nothing but the three checks passed → the resume itself was fine; keep the previous `CODEX_SESSION_ID` unchanged and continue.

After the refresh, return to **Step 5** with the new review.

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

**Fresh-exec prompt template.** The lead rebuilds prior-round context from the conversation (all prior rounds were shown verbatim in Step 5.3 user messages, so they are available in context):

```
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

Write this prompt to `/tmp/codex-prompt-${REVIEW_ID}.md` (overwriting the original is acceptable here; diagnostic files for the failed resume remain in `/tmp/codex-stderr-*` and `/tmp/codex-stdout-*`).

Launch using the **same command template as Step 4** (`cat file | timeout 600 codex exec --json ... -` with `-C`, `-o`, stdout jsonl, stderr; also re-capture `CODEX_SESSIONS_BEFORE` immediately before the call), apply the same post-launch strict check order including the two-tier session-id capture, then return to **Step 5**.

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
      /tmp/codex-stderr-${REVIEW_ID}.txt
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
- **Session ID capture is two-tier.** Primary: `thread_id` from the first JSONL line of stdout. Secondary (when stdout is empty — env-specific): UUID from the trailing component of the newest `~/.codex/sessions/**/rollout-*.jsonl` filename with mtime > `CODEX_SESSIONS_BEFORE`. Capture `CODEX_SESSIONS_BEFORE=$(date +%s)` **before** every `codex exec` / `codex exec resume` call.
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
