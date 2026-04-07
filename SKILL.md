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

> **Placeholders:** `${REVIEW_ID}`, `${CODEX_SESSION_ID}` and `${BASE_BRANCH}` in the steps below are template placeholders, NOT shell variables. Substitute literal values directly into each tool call.

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

### Step 2: Generate Session ID and determine base branch

Generate a unique `REVIEW_ID` yourself, format: `{unix_timestamp}-{random_4digit_number}`.
Example: `1711872000-4821`. **Do NOT use bash** — substitute the value directly into commands in the following steps.

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
- `-m gpt-5.4` — model (overridden by `model:...` argument)
- `-c model_reasoning_effort=high` — reasoning depth (overridden by `xhigh`, `low`, etc.)
- `-s read-only` — reviewer only reads, does not write
- `-o /tmp/codex-review-${REVIEW_ID}.md` — file for capturing output

**Prompt delivery:** write the prompt to `/tmp/codex-prompt-${REVIEW_ID}.md` via **Write tool**, then pass via stdin redirection (`- < file`). This avoids shell quoting issues with long XML prompts.

**Plan Mode note:** Writing to `/tmp` via Write tool may trigger a permission prompt or exit Plan Mode. This is a known Claude Code limitation — Plan Mode restricts edits to the plan file only. If this happens, it does not affect review correctness: the review mode is already determined, and the skill only edits the plan file and `/tmp` temp files.

```bash
timeout 600 codex exec \
  -m gpt-5.4 \
  -c model_reasoning_effort=high \
  -s read-only \
  -o /tmp/codex-review-${REVIEW_ID}.md \
  - < /tmp/codex-prompt-${REVIEW_ID}.md \
  2>/tmp/codex-stderr-${REVIEW_ID}.txt
```

**Important:**
- Always wrap `codex exec` in `timeout 600` (10 minutes). If Codex hangs — the command exits with code 124.
- Use `timeout: 620000` parameter in Bash tool for headroom.
- The command is **synchronous**: when it returns, the `-o` file is ready. Do **NOT** use a poll-loop (`while/sleep`).
- If exit code = 124 (timeout) — inform the user and offer to retry.
- stderr is redirected to a temp file for session ID capture and error diagnostics.

**After launch:** extract the session ID from the stderr file using **Read tool** on `/tmp/codex-stderr-${REVIEW_ID}.txt`. Find the line `session id: <uuid>` (format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`) and extract the UUID value.

Save the result as `CODEX_SESSION_ID` — needed for `resume` in subsequent rounds. If the session ID is not found or does not match UUID format — resume will not work (fallback to fresh exec).

**Notes:**
- Default model: `gpt-5.4` with `model_reasoning_effort=high`. User can override via arguments.
- Always `-s read-only` — reviewer must not write files.
- `-o` captures model output to file. Session ID goes to stderr (captured separately). Do **NOT** run in background.

### Step 5: Read the review and check the verdict

1. Read `/tmp/codex-review-${REVIEW_ID}.md`
2. Show the user **verbatim** — do not rephrase the reviewer's findings:

```
## Adversarial Review — Round N (mode: <plan|code|code-vs-plan>, model: gpt-5.4)

[Reviewer's response — verbatim]
```

3. Check the verdict:
   - **VERDICT: APPROVED** → proceed to Step 8 (Done)
   - **VERDICT: REVISE** → proceed to Step 6 (Fixes)
   - No clear verdict → treat as parse failure, run resume/fallback requesting a clear verdict
   - Maximum reached (5 rounds) → proceed to Step 8 with a note

### Step 6: Apply fixes

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

**Resume is the primary path.** Saves tokens and preserves session context. A fresh `codex exec` without resume is an **emergency fallback** that costs significantly more tokens. Use only if resume fails.

1. Write the resume prompt to `/tmp/codex-prompt-${REVIEW_ID}.md` via **Write tool** (overwrite previous prompt):

```
I've revised based on your feedback.

Here's what I changed:
[List of fixes]

Re-review with the same adversarial stance. Focus on:
1. Whether my fixes actually resolve the reported issues
2. Any NEW issues introduced by the fixes

End with VERDICT: APPROVED or VERDICT: REVISE
```

2. Run resume via stdin redirection:

```bash
timeout 600 codex exec resume ${CODEX_SESSION_ID} \
  - < /tmp/codex-prompt-${REVIEW_ID}.md \
  2>/tmp/codex-stderr-${REVIEW_ID}.txt
```

Use `timeout: 620000` in Bash tool parameters.

**Note:** `resume` does not accept `-s` (sandbox) — sandbox is inherited from the original session. Do not pass `-s` to resume. The `-m` (model) flag is accepted if you need to override the model.

stderr is redirected to temp file for diagnostics. On resume failure, check `/tmp/codex-stderr-${REVIEW_ID}.txt` for details.

3. Check the result by exit code:
   - **exit 0** — success. stdout contains clean review. Show to user directly (Write to file and Read are **not needed**). Check VERDICT: the last non-empty line of stdout = `VERDICT: APPROVED` or `VERDICT: REVISE`. If verdict is missing → output may have been truncated, proceed to Fallback. Then apply verdict handling from Step 5 (APPROVED → Step 8, REVISE → Step 6).
   - **exit 124** — timeout. Tell the user: "Reviewer did not respond within 10 minutes" and offer to retry.
   - **other exit code** — tell the user: "Resume failed (exit code N)". Proceed to Fallback. Check `/tmp/codex-stderr-${REVIEW_ID}.txt` for diagnostics.

**Fallback** — if `resume` did not work (session expired, session ID not captured, error):

1. Collect the list of changed files (same as Step 3).
2. Write the fallback prompt (with description of previous rounds) to `/tmp/codex-prompt-${REVIEW_ID}.md`.
3. Launch a fresh `codex exec` using the **same command template as Step 4** (stdin via `- < file`, stderr to temp file, `-o` for output).
4. **Extract the new session ID** from `/tmp/codex-stderr-${REVIEW_ID}.txt` and **refresh `CODEX_SESSION_ID`** — otherwise subsequent resume attempts will use the stale session ID.

Return to **Step 5**.

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

### Step 9: Cleanup

**In Claude Code Plan Mode:** skip all cleanup (including deferred). rm will trigger a permission prompt. Files will be cleaned up on the next invocation outside Plan Mode.

**Outside Plan Mode:**

```bash
rm -f /tmp/codex-prompt-${REVIEW_ID}.md /tmp/codex-review-${REVIEW_ID}.md \
      /tmp/codex-stderr-${REVIEW_ID}.txt /tmp/codex-plan-${REVIEW_ID}.md
```

If the user declined rm — continue without error.

Do NOT delete plan files that existed before the review (only temp files created by this skill). Old temp files from previous sessions are harmless in /tmp and will be cleaned up by the OS on reboot.

## Rules

- Claude **actively fixes** issues based on reviewer feedback — this is NOT just message forwarding
- Reviewer findings are shown **verbatim** — do not rephrase or shorten
- Auto-detect review mode from context; user arguments take priority
- With explicit `plan` argument or in Claude Code Plan Mode: skip git checks and base branch detection
- Resume is the primary path for subsequent rounds. Fresh exec is an emergency fallback (expensive in tokens)
- Cleanup is best-effort: skip in Plan Mode, continue without error if declined
- Prefer existing files, do not create unnecessary copies
- Always read-only sandbox — reviewer never writes files
- Maximum 5 rounds to protect against infinite loops
- Show the user reviews and fixes for each round
- If Codex CLI is not installed or crashed — tell the user: `npm install -g @openai/codex`
- If a fix contradicts the user's explicit requirements — skip and explain why
