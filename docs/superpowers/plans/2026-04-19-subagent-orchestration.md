# Subagent Orchestration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract codex-exec orchestration (prompt writing, launch, strict checks, session-id capture, retry) from the main-thread skill into a thin Haiku subagent runner. Main thread keeps review display + code fixes + round control. Eliminates the ~48M-token residue tax that currently bloats the main thread context after each `/adversarial-review` invocation.

**Architecture:** Main thread performs mode detection, review-material prep, review display, and code fixes — all small-context operations. The expensive mechanics (codex launch, JSONL parsing, rollout-file content-matching, retry on launch failure) run inside a Haiku subagent dispatched via the Agent tool. The subagent's 250K context is disposed when it completes; only a small JSON result (~1K) plus the final review file path crosses back to main. Main reads `/tmp/codex-review-${REVIEW_ID}.md` (2-5K) for the verbatim display.

**Tech Stack:** Bash, Codex CLI, Claude Code's Agent tool (subagent_type: general-purpose, model: haiku), markdown.

---

## File Structure

- **Modify:** `SKILL.md` — Steps 4, 7, 9 and Rules section. Steps 1-3, 5, 6, 8 remain in main thread.
- **Create:** `references/runner.md` — full runner subagent contract + mechanics (ATTEMPT_ID generation, prompt writing, codex launch, strict checks, two-tier session-id capture, retry logic).
- **Modify:** `AGENTS.md` — update orientation to document main + subagent split.
- **Modify:** `DESIGN.md` — add §3 "Subagent architecture" explaining the split + residue-tax rationale.

---

## Baseline (RED)

Already observed by the user: a single `/adversarial-review` invocation in a long session leaves ~48M tokens of cache_read residue in the main thread. This is because every codex-exec artifact (stdout JSONL, stderr, rollout file paths, rollout content for grep-matching) passes through main-thread Bash tool results.

**Test-passes criterion (GREEN, measured in Task 7 Step 3):** a post-refactor invocation on an equivalent trivial change leaves ≤15K tokens of context growth in main (review markdown 2-5K + runner result JSON ~1K + orchestration overhead). A delta of ~48M tokens would mean the refactor did not take effect; a delta of ~540K would mean the subagent's context is leaking back to main. Target is bounded, measurable, and falsifiable.

---

## Task 1: Create `references/` directory and stub `runner.md`

**Files:**
- Create: `references/runner.md` (placeholder header only; full content in Task 2)

- [ ] **Step 1: Create directory and file**

```bash
mkdir -p /home/dementev/sources/adversarial-review/references
```

Write `/home/dementev/sources/adversarial-review/references/runner.md` with only the header:

```markdown
# Adversarial-Review Runner Subagent

> This file is read by a Haiku subagent dispatched from `SKILL.md` Step 4 or Step 7. It is NOT loaded in the main thread. See `SKILL.md` for the orchestrator contract.

(Content added in Task 2.)
```

- [ ] **Step 2: Commit the scaffolding**

```bash
cd /home/dementev/sources/adversarial-review
git add references/runner.md
git commit -m "$(cat <<'EOF'
feat(skill): scaffold references/ for runner subagent

Empty placeholder for subagent orchestration refactor.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Write the runner subagent contract in `references/runner.md`

**Files:**
- Modify: `references/runner.md`

- [ ] **Step 1: Replace file content with the full runner spec**

Write this content to `references/runner.md` (replacing the placeholder):

````markdown
# Adversarial-Review Runner Subagent

> This file is read by a Haiku subagent dispatched from `SKILL.md` Step 4 or Step 7. It is NOT loaded in the main thread.

You are a thin runner subagent. Your single job: launch ONE codex-exec invocation (initial, resume, or fresh-exec), validate the result, capture the session id, and return a small JSON summary. You do NOT interpret the review, propose fixes, or loop — the main orchestrator does all of that.

## Input contract

The main thread dispatches you via Claude Code's Agent tool. The prompt contains a YAML-like input block. Parse these fields; if any required field is missing, write an input_error result to the result file (see Output contract) and return.

```yaml
REVIEW_ID: <string, format "{unix_ts}-{8-digit}">
REPO_ROOT: <absolute path, validated by main>
OPERATION: initial | resume | fresh-exec
CODEX_MODEL: <e.g. gpt-5.4>  # the model codex CLI launches; DO NOT confuse with your own (Haiku) model
CODEX_REASONING: <low | medium | high | xhigh>
PROMPT_BODY_PATH: <absolute path to file containing the review prompt body WITHOUT the session marker; main writes this before dispatch>
CODEX_SESSION_ID: <UUID, required only when OPERATION=resume>
RESULT_PATH: /tmp/codex-runner-result-<REVIEW_ID>.json  # you write the structured result here
```

For `OPERATION=initial` and `OPERATION=fresh-exec`, `CODEX_SESSION_ID` is absent (ignore if present).

## Output contract — two-channel

To avoid fragility of JSON-in-final-message (Haiku frequently wraps structured output in markdown fences or adds preamble), you return results via TWO channels:

**Channel 1 — result file (authoritative).** Write the JSON object below to `${RESULT_PATH}` via Write tool. Main reads this file directly; its bytes are the contract. Do NOT omit any field — use `null` for absent values.

```json
{
  "result": "success" | "launch_failure" | "timeout" | "infra_error" | "input_error",
  "verdict": "APPROVED" | "REVISE" | null,
  "review_file": "/tmp/codex-review-<REVIEW_ID>.md" | null,
  "codex_session_id": "<uuid>" | null,
  "attempt_id": "<6-digit>",
  "errors": "<short diagnostic, ≤500 chars>" | null,
  "archived_stdout": "/tmp/codex-stdout-<REVIEW_ID>-failed-resume.jsonl" | null,
  "archived_stderr": "/tmp/codex-stderr-<REVIEW_ID>-failed-resume.txt" | null,
  "user_warning": "<one-line message main should surface to user>" | null
}
```

**Channel 2 — final message (short).** Your FINAL message to main should be a single line:

```
RUNNER_RESULT_AT: <RESULT_PATH>
```

Example: `RUNNER_RESULT_AT: /tmp/codex-runner-result-1711872000-48217593.json`

Main's parser is tolerant: it searches the ENTIRE message for a match of the unanchored regex `RUNNER_RESULT_AT:\s+(\S+)` (first match wins; works inside markdown fences, after preamble, or surrounded by other text). Even so, emitting the spec line cleanly (no fence, no preamble) eliminates edge cases.

If your message lacks the line entirely, main falls back to a filesystem Glob for `/tmp/codex-runner-result-${REVIEW_ID}.json` — the path is deterministic from REVIEW_ID, which main already holds. If the Glob also fails (file not written), main treats the run as `infra_error`.

Rules:
- `result=success` ⇒ `verdict` and `review_file` must be set. `codex_session_id` must be set iff `verdict=REVISE` (or null per §2.4.4 on resume zero-find — see Step R4.4).
- `result=timeout` ⇒ codex timed out (exit 124). `review_file` may be null.
- `result=launch_failure` ⇒ infrastructure retry (one internal retry) already failed. Main treats this as TERMINAL — it will NOT re-dispatch you. `errors` MUST include the tail of stderr.
- `result=infra_error` ⇒ something outside codex (e.g. `/tmp` not writable).
- `user_warning` is non-null when main should surface a one-line warning to the user (e.g. §2.4.4 zero-find on resume).
- Do NOT return the review text in the JSON. Main reads `review_file` directly.

## Step-by-step

### Step R1: Generate ATTEMPT_ID

Generate a fresh 6-digit random integer. Use `printf` with `$RANDOM`:

```bash
printf '%06d\n' $((RANDOM * RANDOM % 1000000))
```

Save the output as `${ATTEMPT_ID}` for this invocation. Generate a NEW ATTEMPT_ID on every retry (Step R5).

### Step R2: Build the launch prompt file

Read `${PROMPT_BODY_PATH}` (main wrote it before dispatching you).

For `OPERATION=initial` or `OPERATION=fresh-exec`:
- Write `/tmp/codex-prompt-${REVIEW_ID}.md` with first line `<!-- ADVERSARIAL-REVIEW-SESSION: ${REVIEW_ID}-${ATTEMPT_ID} -->` followed by the body.

For `OPERATION=resume`:
- Write `/tmp/codex-resume-prompt-${REVIEW_ID}.md` with the same marker-first structure.

Use the Write tool (not `cat <<EOF` via Bash — Write is simpler and does not have quoting edge cases).

**Bump mtime via a second Write** to ensure the prompt file's mtime is strictly later than any rollout file from a prior attempt. Rewrite the same bytes to the same path (Write tool, not Bash `touch` — Bash may be restricted under inherited Plan Mode; Write is already the tool used for the initial body, so whatever gating applies is already cleared by the first Write).

On systems with coarse mtime granularity (1s), two successive Writes within the same second can produce identical mtimes; the repeat Write forces the kernel to update the mtime. On Plan Mode-inherited subagents the first Write may have already prompted the user; the second Write to the identical path reuses that same permission grant.

Alternatively and equivalently safe: skip the mtime bump entirely and rely on ATTEMPT_ID rotation alone — the positive content-match in Step R4.4 binds on the marker, not solely on `-newer`. If the retry's new ATTEMPT_ID is embedded in the prompt's first line (which it is), no prior rollout can false-match. The `-newer` condition is a second guard, not a primary one. If the repeat-Write approach fails in practice, drop it and rely on content-match + multi-match-aborts.

### Step R3: Launch codex

**Synchronous launch only.** Always invoke the Bash tool with `run_in_background: false` (the default). Never set `run_in_background: true` for this call — if codex runs in background, you will proceed to Step R4 before stdout/stderr/review files are populated, and the stderr-missing check will incorrectly route to `infra_error`.

For `OPERATION=initial` and `OPERATION=fresh-exec`:

```bash
cat /tmp/codex-prompt-${REVIEW_ID}.md | timeout 600 codex exec --json \
  -m ${CODEX_MODEL} \
  -c model_reasoning_effort=${CODEX_REASONING} \
  -s read-only \
  -C "${REPO_ROOT}" \
  -o /tmp/codex-review-${REVIEW_ID}.md \
  - \
  > /tmp/codex-stdout-${REVIEW_ID}.jsonl \
  2>/tmp/codex-stderr-${REVIEW_ID}.txt
```

Bash tool `timeout` parameter: `620000` (10 min + headroom).

For `OPERATION=resume`:

```bash
cd '${REPO_ROOT}' && cat /tmp/codex-resume-prompt-${REVIEW_ID}.md | timeout 600 codex exec resume --json \
  ${CODEX_SESSION_ID} \
  -o /tmp/codex-review-${REVIEW_ID}.md \
  - \
  > /tmp/codex-stdout-${REVIEW_ID}.jsonl \
  2>/tmp/codex-stderr-${REVIEW_ID}.txt
```

Note: resume does NOT accept `-C`, `-s`, or `-m`; these are inherited from the original session. Use `cd` to pin cwd.

Substitute literal values for every `${...}` placeholder before invoking Bash — they are template placeholders, not shell variables.

### Step R4: Post-launch strict checks

Do these in order. Stop and return as soon as one fails.

**Check R4.1: Exit code.**
- `124` → route to retry (Step R5). Same retry budget as any other failure — ONE retry per dispatch total. If retry also returns 124, write `{"result":"timeout",...}` and return. (Retrying on timeout keeps the 2-attempts-per-round invariant consistent across failure types; main treats timeout as terminal just like launch_failure.)
- `≠ 0 and ≠ 124` → read `/tmp/codex-stderr-${REVIEW_ID}.txt`, route to retry (Step R5).
- `0` → proceed.

**Check R4.2: Stderr sanity.** Read `/tmp/codex-stderr-${REVIEW_ID}.txt`.
- File missing → return `{"result":"infra_error","errors":"stderr file missing — /tmp writability?",...}`.
- File contains a line matching `^Error:` or `Failed to write` → route to retry (Step R5).

**Check R4.3: Review file sanity.** Read `/tmp/codex-review-${REVIEW_ID}.md`.
- File missing or empty → route to retry (Step R5).
- Does NOT contain a line matching `^VERDICT: (APPROVED|REVISE)$` → route to retry.
- Verdict is `REVISE` AND file contains NO line matching `\[severity:\s*(critical|high|medium)` → route to retry (reviewer format drift).
- Verdict is `APPROVED` → return `{"result":"success","verdict":"APPROVED","review_file":"/tmp/codex-review-${REVIEW_ID}.md","codex_session_id":null,"attempt_id":"${ATTEMPT_ID}","errors":null}`.
- Verdict is `REVISE` → proceed to Check R4.4.

**Check R4.4: Capture session id — two tiers.**

*Primary — first line of JSONL stdout:*

Read `/tmp/codex-stdout-${REVIEW_ID}.jsonl`. If the first line parses as JSON with a `thread_id` field matching `^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$`, save as `CODEX_SESSION_ID` and return success. Otherwise fall through.

*Secondary — rollout content-match:*

The anchor file is `/tmp/codex-prompt-${REVIEW_ID}.md` for initial/fresh-exec, or `/tmp/codex-resume-prompt-${REVIEW_ID}.md` for resume.

```bash
find ~/.codex/sessions -name 'rollout-*.jsonl' -newer <anchor> -exec grep -l 'ADVERSARIAL-REVIEW-SESSION: ${REVIEW_ID}-${ATTEMPT_ID}' {} + 2>/dev/null
```

**Interpret the result by STDOUT, not exit code.** Split the command's stdout on newlines; count non-empty lines. Empty stdout means ZERO paths regardless of the pipeline's exit status (`find` returning no matches and `grep -l` matching nothing in found files both yield empty stdout with different exit codes; treat both as zero).

- **Exactly one path** → extract the trailing UUID from the filename (pattern `rollout-<ISO-timestamp>-<UUID>.jsonl`; UUID is the 36-char hex-and-dashes segment before `.jsonl`), then write this EXACT JSON object to `${RESULT_PATH}` (all 9 fields explicitly; do NOT leave any omitted or as literal placeholder like `"<uuid>"`):

```json
{
  "result": "success",
  "verdict": "REVISE",
  "review_file": "<absolute path to /tmp/codex-review-REVIEW_ID.md, substituted>",
  "codex_session_id": "<actual 36-char UUID you extracted from the filename>",
  "attempt_id": "<the current ATTEMPT_ID string>",
  "errors": null,
  "archived_stdout": null,
  "archived_stderr": null,
  "user_warning": null
}
```

- **Zero paths for resume** → write this EXACT JSON object (9 fields, `codex_session_id` is null, `user_warning` carries the §2.4.4 diagnostic):

```json
{
  "result": "success",
  "verdict": "REVISE",
  "review_file": "<absolute path substituted>",
  "codex_session_id": null,
  "attempt_id": "<ATTEMPT_ID>",
  "errors": null,
  "archived_stdout": null,
  "archived_stderr": null,
  "user_warning": "Step 7 session-id refresh: both tiers empty, continuing with previous ID per DESIGN.md §2.4.4"
}
```

- **Zero paths for initial/fresh-exec** → route to retry (Step R5). Main needs the id to launch next round. Set `errors: "session-id capture failed: both tiers empty on initial/fresh-exec"`.
- **Multiple paths** → write `launch_failure` result with `errors: "multiple rollouts matched marker — aborting to avoid wrong-session bind"`. Do NOT pick one.

(The two `success` JSON shapes are inlined above per branch. Every success path through R4.4 MUST emit a complete 9-field JSON object — never rely on implicit defaults, never leave a field omitted, never write a literal placeholder like `"<uuid>"` in the output.)

### Step R5: Retry once on launch failure (TERMINAL — main will not re-dispatch)

You have at most ONE retry per dispatch. This retry is the ONLY retry in the system — main treats your `launch_failure` result as terminal and will NOT re-dispatch you. Track the retry counter in your reasoning.

On retry:
1. Generate a NEW `ATTEMPT_ID` (the old one stays in the old rollout; we must not let the grep match it again).
2. Rewrite the prompt file with the new marker (using the Write tool; the write itself bumps mtime — do NOT use Bash `touch`, which may be gated by inherited Plan Mode on the subagent).
3. Re-launch (same Step R3 command, still `run_in_background: false`).
4. Re-run checks R4.1–R4.4.

If the second attempt also fails any check:
- For `OPERATION=resume`: before writing the `launch_failure` result, **archive the diagnostic files** (main will need them for the fallback fresh-exec which reuses the same base paths):

```bash
mv /tmp/codex-stdout-${REVIEW_ID}.jsonl /tmp/codex-stdout-${REVIEW_ID}-failed-resume.jsonl 2>/dev/null
mv /tmp/codex-stderr-${REVIEW_ID}.txt    /tmp/codex-stderr-${REVIEW_ID}-failed-resume.txt 2>/dev/null
```

Then write the result with `archived_stdout` and `archived_stderr` set to the `-failed-resume.*` paths.

- For `OPERATION=initial` or `OPERATION=fresh-exec`: no archival needed (there is no next attempt within this REVIEW_ID to collide). Leave files at their normal paths for main's diagnostic read (main is allowed to `mv`/`rm` by path; it just doesn't read content).

Write the `launch_failure` result (with stderr tail ≤500 chars in `errors`) and return the `RUNNER_RESULT_AT: ...` line.

### Step R6: Cleanup

Do NOT delete `/tmp/codex-*` files. The main orchestrator owns the review-lifecycle cleanup at SKILL.md Step 9. Leaving files in place lets main:
- Read `review_file` after parsing your result.
- Keep the `-failed-resume.*` archives available for the fresh-exec fallback.
- Clean the whole set (including `-failed-resume.*`) when the review concludes via its existing Step 9 `rm` glob.

The one exception is the `mv` in Step R5 above — this is NOT cleanup (files are preserved, just renamed to avoid collision with the imminent fresh-exec). Doing the `mv` in the runner rather than main eliminates the isolation-claim drift that would otherwise occur if main had to touch stdout/stderr paths in its own Bash argv.

## Notes

- You run as a Haiku subagent. Your 250K context is disposed when you return. Anything you read (stderr files, rollout paths, JSONL streams) does NOT reach the main thread — that is the whole point.
- Do NOT ask the main thread clarifying questions. If input is missing or malformed, write an `input_error` result to `${RESULT_PATH}` and return the `RUNNER_RESULT_AT:` line.
- Do NOT attempt to apply fixes, interpret severity, or re-run more than one retry. The 5-round orchestration loop lives in main.
- The final line of your message is ONLY `RUNNER_RESULT_AT: <path>` — nothing before, nothing after, no markdown fence. Main's regex tolerates minor wrapping, but adhering to the spec eliminates edge cases entirely.
````

- [ ] **Step 2: Verify the file is syntactically reasonable**

```bash
wc -l /home/dementev/sources/adversarial-review/references/runner.md
```

Expected: roughly 180-220 lines.

- [ ] **Step 3: Commit**

```bash
cd /home/dementev/sources/adversarial-review
git add references/runner.md
git commit -m "$(cat <<'EOF'
feat(skill): add runner subagent contract

Defines input/output JSON contract, step-by-step mechanics, and
cleanup ownership for the thin Haiku runner.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Refactor `SKILL.md` Step 4 — dispatch subagent for initial launch

**Files:**
- Modify: `SKILL.md` — Step 4 section (roughly lines 129-418 in current file)

- [ ] **Step 1: Read current SKILL.md Step 4**

Use Read on `/home/dementev/sources/adversarial-review/SKILL.md` offset 129, limit 290 to confirm line boundaries before editing.

- [ ] **Step 2: Replace Step 4 content**

Replace the entire block from the `### Step 4:` heading through the end of Step 4 (just before `### Step 5:`) with:

````markdown
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

Note: the body is IDENTICAL to the pre-refactor SKILL.md Step 4 code-review prompt EXCEPT the leading `<!-- ADVERSARIAL-REVIEW-SESSION: ... -->` line is stripped — the runner subagent prepends the session marker in Step R2.

> **Sync-drift note:** Once Task 3 is implemented, SKILL.md Step 4 is the CANONICAL source for these prompt bodies. This plan document is a point-in-time artifact committed to `docs/superpowers/plans/`; future edits to the prompts happen in SKILL.md, not here. If you find a drift between this plan and SKILL.md post-merge, SKILL.md wins.

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

The inlined prompt bodies above contain template placeholders that main must resolve with real captured values before the Write. Pre-refactor this happened implicitly at Step 4 where values were in lexical scope; post-refactor main writes the body to a file and hands the PATH to the runner, so substitution MUST happen in main. Placeholders per mode:

| Placeholder | Value source | Applies to |
|---|---|---|
| `${REVIEW_ID}` | captured at Step 2 | all modes (in any comment/reference — note the session marker comes from runner, not here) |
| `${REPO_ROOT}` | captured at Step 2 | all modes |
| `${BASE_BRANCH}` | captured at Step 2 (code & code-vs-plan only) | code, code-vs-plan |
| `<plan-path>` | captured at Step 3 | plan, code-vs-plan |
| `<file list from --name-only>` | result of `git diff --name-only` + `git diff --cached --name-only` (or branch diff) from Step 3 | code, code-vs-plan (≤50 files only) |
| `<unstaged changes / staged changes / ...>` | human-readable description derived from which diff commands had content | code, code-vs-plan |
| `<git diff commands>` | the exact commands main determined at Step 3 (e.g. `git diff`, `git diff --cached`, `git diff ${BASE_BRANCH}...HEAD`) | code, code-vs-plan |

Main substitutes each placeholder with its literal resolved value in the body text. The resulting string goes to the Write tool.

**Capture user overrides for `CODEX_MODEL` / `CODEX_REASONING` (Step 1 argument parsing):**

The skill supports overrides like `/adversarial-review xhigh`, `/adversarial-review medium`, `/adversarial-review model:gpt-5.3-codex`. At Step 1 (mode detection), main also captures:

- `CODEX_MODEL` — default `gpt-5.4`. Overridden by any argument matching `^model:(.+)$`; use the capture group.
- `CODEX_REASONING` — default `high`. Overridden by any argument exactly matching `low`, `medium`, `high`, or `xhigh`.

These are passed into the runner YAML input block below. Forgetting this capture silently disables the user-facing override feature.

**Write the prompt body to disk via Write tool:**

Write `/tmp/codex-body-${REVIEW_ID}.md` containing the substituted body text (no session marker — the runner adds it).

> **Plan Mode note:** Writing to `/tmp` via Write tool may trigger a permission prompt or exit Plan Mode. This is a known Claude Code limitation. Additionally, dispatching a subagent under Plan Mode may inherit the restriction — Task 7 Step 4 determines the empirical behavior.

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

Save the resolved absolute path as `RUNNER_SPEC_PATH`. Do NOT attempt to extract the path from any "Base directory for this skill:" line in the conversation — that line is a system injection Claude cannot reliably read from inside its own context.

**Dispatch the runner subagent via Agent tool:**

Read `${RUNNER_SPEC_PATH}` once (main thread). This is the subagent's full spec.

Invoke the Agent tool with:
- `subagent_type: "general-purpose"`
- `model: "haiku"`
- `description: "Adversarial-review runner, round N"` (N is the current round number)
- `prompt:` the concatenation of:
  1. The full text of `references/runner.md` you just read (as instructions).
  2. A blank line.
  3. A literal YAML input block with the values substituted (use `CODEX_MODEL` and `CODEX_REASONING` to avoid confusion with the subagent's own Haiku model):

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

**Do NOT run the Agent tool call in background.** Wait for the subagent to return. (Runner's own codex exec is also synchronous per Step R3.)

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

**Round-level attempt invariant (enforced by the table above):** exactly ONE runner dispatch per round. Every failure result is terminal at main. The runner owns the full retry budget (≤2 attempts per dispatch, internal) regardless of failure type. Total codex invocations per round ≤ 2 across ALL failure types — this matches pre-refactor and closes Round-2 finding #1 (timeout compounding).

> **CRITICAL — main thread does NOT read stdout/stderr/JSONL/rollout files BY CONTENT.** Those live and die inside the subagent. Main reads: the runner result JSON at `RESULT_PATH`, the review file at `review_file`, and nothing else from `/tmp/codex-*`. Archival `mv` (on resume failure) is done by the runner, not main — main never references `/tmp/codex-stdout-*` or `/tmp/codex-stderr-*` in any Bash argv.
````

- [ ] **Step 3: Verify SKILL.md is still well-formed markdown**

```bash
head -c 200 /home/dementev/sources/adversarial-review/SKILL.md
```

- [ ] **Step 4: Update SKILL.md Step 5 "VERY NEXT MESSAGE" wording to accommodate `⚠ user_warning`**

Current SKILL.md Step 5 (around line 439 in the pre-refactor file) contains:
> Your VERY NEXT MESSAGE to the user must begin with the header below, followed by the file contents verbatim.

And the precondition gate (around line 461):
> confirm that you have already sent a user-visible message in THIS round whose body contains the verbatim review text. If you have not — STOP.

These two rules conflict with the new design where `⚠ <user_warning>` is emitted as its own turn BEFORE the Step 5 verbatim message. Update Step 5 as follows:

Replace `Your VERY NEXT MESSAGE to the user must begin with the header below` with `Your next user-visible message that is NOT a one-line \`⚠ <warning>\` diagnostic must begin with the header below`.

Replace the precondition gate's check `sent a user-visible message in THIS round whose body contains the verbatim review text` with `sent a user-visible message in THIS round whose body contains the verbatim review text (NOT counting short \`⚠ <user_warning>\` diagnostic messages)`.

This reconciles Step 5's blocking-display contract with the new warning-before-display contract. Make this edit in the SAME commit as Step 3 (below).

- [ ] **Step 5: Commit**

```bash
cd /home/dementev/sources/adversarial-review
git add SKILL.md
git commit -m "$(cat <<'EOF'
refactor(skill): dispatch initial codex launch via Haiku subagent

Replaces inline codex-exec + strict checks + session-id capture
in Step 4 with Agent tool dispatch. Runner spec lives in
references/runner.md. Main thread reads only the final review file
(~5K) instead of stdout/stderr/rollout artifacts (~48M residue).

Also updates Step 5 "VERY NEXT MESSAGE" wording to permit a
preceding one-line user_warning diagnostic message.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Refactor `SKILL.md` Step 7 — dispatch subagent for resume

**Files:**
- Modify: `SKILL.md` — Step 7 section (roughly lines 481-612 in current file)

- [ ] **Step 1: Read current Step 7**

Use Read on `SKILL.md` to confirm the current boundaries of Step 7.

- [ ] **Step 2: Replace Step 7 with the resume-via-subagent version**

Replace the entire block from `### Step 7:` through the end of Step 7 (just before `### Step 8:`) with:

````markdown
### Step 7: Resubmit to Codex (Rounds 2-5)

**Resume is the primary path.** Saves tokens and preserves session context. A fresh `codex exec` without resume is an **emergency fallback** when resume itself fails.

**Step 7.1: Write the resume prompt body to disk.**

Write `/tmp/codex-resume-body-${REVIEW_ID}.md` containing:

```
I've revised based on your feedback.

Here's what I changed:
[List of fixes from Step 6]

Re-review with the same adversarial stance. Focus on:
1. Whether my fixes actually resolve the reported issues
2. Any NEW issues introduced by the fixes

End with VERDICT: APPROVED or VERDICT: REVISE
```

Substitute the fixes list from Step 6 (one bullet per finding addressed). Do NOT include the session marker — the subagent adds it.

**Step 7.2: Dispatch the runner subagent for resume.**

Same Agent tool invocation as Step 4 (runner spec + YAML input block). Reuse the `RUNNER_SPEC_PATH` resolved in Step 4 (do not re-resolve). Input block:

```yaml
---
REVIEW_ID: <same as initial round>
REPO_ROOT: <same>
OPERATION: resume
CODEX_MODEL: <same>
CODEX_REASONING: <same>
PROMPT_BODY_PATH: /tmp/codex-resume-body-<REVIEW_ID>.md
RESULT_PATH: /tmp/codex-runner-result-<REVIEW_ID>.json
CODEX_SESSION_ID: <uuid from previous round's runner result>
---
```

**Step 7.3: Parse the two-channel result.**

Extract `RUNNER_RESULT_AT:` line (same tolerant regex + Glob fallback as Step 4), read the JSON file, extract fields. If `user_warning` is non-null, emit it as its own `⚠ <user_warning>` message BEFORE any other action (including before the Step 5 verbatim review) — see Step 4's user_warning rule.

| `result` value | Main thread action |
|---|---|
| `success`, verdict `APPROVED` | Read `review_file`, go to Step 5 (it will dispatch to Step 8 on APPROVED). |
| `success`, verdict `REVISE` | Save new `codex_session_id`. If the subagent returned null (zero-find resume), keep the prior id per §2.4.4 — `user_warning` will already have been surfaced. Go to Step 5. |
| `timeout` | **TERMINAL for this round** — runner already attempted twice. Route to fallback below. (Fresh-exec is a NEW round from the 5-round counter — its own ≤2-attempts budget applies.) No user-offered retry; that would compound. |
| `launch_failure` | **TERMINAL for this round** — runner already retried once internally. Route to fallback below (runner already archived stdout/stderr to `-failed-resume.*` — paths in `archived_stdout` / `archived_stderr`). |
| `infra_error` | Show `errors` to user, abort. |

**Round-level attempt invariant:** exactly ONE runner dispatch per resume round. Every failure result routes to fallback (not re-dispatch within the same round). Fallback's fresh-exec dispatch consumes a NEW round from the 5-round counter, which has its own independent 2-attempts-per-round budget. Total codex invocations per round ≤ 2 regardless of failure type — matches pre-refactor; closes Round-2 finding #1.

**Step 7.4: Fallback chain** — triggered by `launch_failure` or repeated `timeout` from the runner.

*Severity classification:* parse the PREVIOUS round's review (kept in conversation history from Step 5.3's verbatim display) for the highest `[severity:` level. Default to `critical` if zero matches (format drift).

*Interactive mode* (direct user message earlier in this session): ask the user:

```
Resume failed — the reviewer's re-review did not produce a usable result.
Last round's maximum severity: <level>.

Options:
(a) Run a fresh `codex exec` with full previous-rounds context (higher token cost, new session)
(b) Conclude the review — show current findings as NOT VERIFIED
```

*Non-interactive mode:*
- Max severity `critical` or `high` → fresh exec automatically.
- Max severity `medium` only → Step 8 with the not-verified terminal state.

*Fresh-exec dispatch:* build a new PROMPT_BODY that is the original Step 4 prompt for the current mode, followed by sections `## Previous review rounds` (verbatim round-1..N reviews + fixes from conversation history) and `## Current state of the artifact`. Write to `/tmp/codex-body-${REVIEW_ID}.md` (overwriting the original).

**Archival note:** if the fallback was triggered by `launch_failure`, the runner already archived failed-resume stdout/stderr to `-failed-resume.*` paths during Step R5 — main does NOT need to `mv` anything. If triggered by repeated `timeout`, no archival happened (no second codex invocation produced useful diagnostics); main can proceed directly. Either way, main never touches `/tmp/codex-stdout-*` or `/tmp/codex-stderr-*` itself.

Dispatch the runner subagent with `OPERATION=fresh-exec` (same input schema, new PROMPT_BODY_PATH pointing at the rebuilt prompt). The fresh-exec consumes one round from the 5-round counter. Return to Step 5 with the new review.
````

- [ ] **Step 3: Commit**

```bash
cd /home/dementev/sources/adversarial-review
git add SKILL.md
git commit -m "$(cat <<'EOF'
refactor(skill): dispatch resume and fresh-exec via subagent

Step 7 now delegates codex-exec-resume to the runner. Fallback
path (resume failure → fresh exec) also goes through the runner
with OPERATION=fresh-exec. Severity classification and user
interaction stay in main thread.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Update `SKILL.md` Rules section and ancillary bits

**Files:**
- Modify: `SKILL.md` — Rules section (roughly lines 687-711 in current file)

- [ ] **Step 1: Replace the Rules bullet list**

Find the `## Rules` heading near the end of `SKILL.md` and replace its bullet list with:

```markdown
## Rules

- Claude **actively fixes** issues based on reviewer feedback — this is NOT just message forwarding.
- Reviewer findings are shown **verbatim** — do not rephrase or shorten. The Step 5 "YOUR NEXT MESSAGE" instruction is blocking.
- Auto-detect review mode from context; user arguments take priority.
- With explicit `plan` argument or in Claude Code Plan Mode: skip git checks and base branch detection.
- **`REPO_ROOT` is captured at Step 2** and passed as an absolute literal to every runner dispatch.
- **`RUNNER_SPEC_PATH` is resolved at Step 4** (once per review) with priority: (1) `~/.claude/skills/adversarial-review/references/runner.md` (user-scoped install), (2) Glob `~/.claude/plugins/cache/**/skills/adversarial-review/references/runner.md` and take first hit (plugin-marketplace install), (3) `$(git rev-parse --show-toplevel)/references/runner.md` (dev checkout), (4) abort with installation error. Main never attempts to read Claude's own system prompt / skill-invocation header — that path is hallucination-prone and is explicitly disallowed.
- **Codex-exec mechanics live in the runner subagent** (`references/runner.md`): ATTEMPT_ID generation, prompt-with-marker writing (with a repeated Write call for mtime freshness — NOT Bash `touch`, which may be gated by inherited Plan Mode), synchronous launch, strict checks, two-tier session-id capture with positive content-bind, ONE internal retry on ANY failure type (launch_failure, timeout, stderr-infra), archival mv on resume failure. Main thread never reads codex stdout/stderr/rollout file CONTENT, and never references those paths in its own Bash argv.
- **Two-channel result protocol.** Runner writes structured JSON to `/tmp/codex-runner-result-${REVIEW_ID}.json` (authoritative) AND returns a single `RUNNER_RESULT_AT: <path>` line as its final message. Main extracts the path via regex (tolerant to markdown fences / minor wrapping), reads the JSON, and never relies on raw-JSON-in-message parsing.
- **Main thread reads only**: the runner result JSON, the review file at `review_file`, and the skill's own `references/runner.md`. No other `/tmp/codex-*` reads.
- **Runner is dispatched via Agent tool** with `subagent_type: general-purpose, model: haiku`. Agent tool call is synchronous (not `run_in_background`).
- **ALL runner failure results are TERMINAL at main** (`launch_failure`, `timeout`, `infra_error`, `input_error`). Runner retries once internally on ANY failure. Main does NOT re-dispatch and does NOT offer the user a retry — those lanes would compound retries across layers. Total codex invocations per round ≤ 2 (matches pre-refactor invariant: 1 initial + 1 retry). Fresh-exec fallback is a NEW round with its own independent 2-attempts budget.
- **`user_warning` from the runner must be surfaced to the user** on a single line BEFORE any other action. This preserves the pre-refactor §2.4.4 "both tiers empty, continuing with previous ID" diagnostic.
- **`CODEX_MODEL` / `CODEX_REASONING`** in the runner input schema refer to the model codex CLI launches (e.g. `gpt-5.4`). The runner's OWN model is Haiku, set via Agent tool's `model: "haiku"`. Do NOT conflate.
- **Resume is the primary path for rounds 2-5.** Fresh-exec fallback consumes one round from the 5-round counter.
- **Step 9 cleanup `rm` glob is UNCHANGED from pre-refactor.** It still covers `/tmp/codex-plan-${REVIEW_ID}.md`, `/tmp/codex-prompt-${REVIEW_ID}.md`, `/tmp/codex-resume-prompt-${REVIEW_ID}.md`, `/tmp/codex-review-${REVIEW_ID}.md`, `/tmp/codex-stdout-${REVIEW_ID}.jsonl`, `/tmp/codex-stderr-${REVIEW_ID}.txt`, `/tmp/codex-stdout-${REVIEW_ID}-failed-resume.jsonl`, `/tmp/codex-stderr-${REVIEW_ID}-failed-resume.txt`. ADD the two new paths introduced by the refactor: `/tmp/codex-body-${REVIEW_ID}.md` and `/tmp/codex-runner-result-${REVIEW_ID}.json`.
- Cleanup is **conditional on terminal state**: remove temp files on approved/max-reached/not-verified; LEAVE them on abort (diagnostic value). Skip all cleanup in Plan Mode.
- Always read-only sandbox — reviewer never writes files.
- Maximum 5 rounds to protect against infinite loops.
- Show the user reviews and fixes for each round.
- If Codex CLI is not installed or crashed — tell the user: `npm install -g @openai/codex`.
- If a fix contradicts the user's explicit requirements — skip and explain why.
```

- [ ] **Step 2: Update the frontmatter description**

Verify the frontmatter at the top of `SKILL.md` still matches the skill's behavior. No change needed — behavior is unchanged; only orchestration is internal.

- [ ] **Step 3: Update Step 9 cleanup glob to include new paths**

Find the `rm -f ...` block in Step 9 of SKILL.md (around line 672-681 of the current file). Add the two new paths introduced by the refactor: `/tmp/codex-body-${REVIEW_ID}.md` and `/tmp/codex-runner-result-${REVIEW_ID}.json`. All other paths remain. The resulting block should be:

```bash
rm -f /tmp/codex-plan-${REVIEW_ID}.md \
      /tmp/codex-prompt-${REVIEW_ID}.md \
      /tmp/codex-resume-prompt-${REVIEW_ID}.md \
      /tmp/codex-review-${REVIEW_ID}.md \
      /tmp/codex-stdout-${REVIEW_ID}.jsonl \
      /tmp/codex-stderr-${REVIEW_ID}.txt \
      /tmp/codex-stdout-${REVIEW_ID}-failed-resume.jsonl \
      /tmp/codex-stderr-${REVIEW_ID}-failed-resume.txt \
      /tmp/codex-body-${REVIEW_ID}.md \
      /tmp/codex-runner-result-${REVIEW_ID}.json
```

No other logic in Step 9 changes (conditional-on-terminal-state, Plan Mode skip).

- [ ] **Step 4: Commit**

```bash
cd /home/dementev/sources/adversarial-review
git add SKILL.md
git commit -m "$(cat <<'EOF'
docs(skill): rewrite Rules section + extend Step 9 cleanup glob

Reflects the new main + runner split. Removes rules that now
live exclusively in references/runner.md (ATTEMPT_ID generation,
two-tier session capture, strict check order). Adds two new
/tmp paths to Step 9 cleanup: codex-body and codex-runner-result.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Update `AGENTS.md` and `DESIGN.md`

**Files:**
- Modify: `AGENTS.md`
- Modify: `DESIGN.md`

- [ ] **Step 1: Add architecture section to AGENTS.md**

Read current `AGENTS.md`. Add a new section `## Architecture` (after the existing orientation) with:

```markdown
## Architecture

The skill runs in two processes:

**Main orchestrator** (`SKILL.md`, main Claude thread): mode detection, REVIEW_ID, REPO_ROOT capture, review-material prep (Steps 1-3), review display (Step 5), code fixes (Step 6), final summary (Step 8), cleanup (Step 9), and round counting.

**Runner subagent** (`references/runner.md`, dispatched via Claude Code's Agent tool with `model: haiku`): builds the launch prompt with per-attempt session marker, invokes `codex exec` (or resume or fresh-exec), runs strict checks on the result, captures the session id via two-tier lookup (primary JSONL `thread_id`, secondary rollout content-match), retries once on infrastructure failure, returns a small JSON summary.

**Why the split:** Every codex-exec invocation produces stdout JSONL, a stderr file, and a rollout file under `~/.codex/sessions/`. Keeping these inside the subagent means the main thread's context never sees them — only the final review markdown (~5K) flows back. This eliminates the ~48M-token cache-read residue observed before the split.

**Boundary invariants:**
- Main never reads `/tmp/codex-stdout-*`, `/tmp/codex-stderr-*`, or any rollout file directly.
- Subagent never interprets findings, applies fixes, or decides whether to start another round.
- The review file at `/tmp/codex-review-${REVIEW_ID}.md` is the sole artifact that crosses the boundary.
```

- [ ] **Step 2: Append §12 to DESIGN.md (DO NOT renumber existing sections)**

Read current `DESIGN.md`. AGENTS.md §10.2 explicitly warns against renumbering DESIGN.md sections — cross-refs would break silently. Therefore: **append** the new section as `§12. Subagent architecture`. Do NOT insert at §3 or anywhere that would shift existing numbering.

```markdown
## 12. Subagent architecture

### 12.1 The residue problem

Before this split, the entire skill ran in the main Claude thread. Each round's `Bash` tool call produced result content that Claude's context caches: the codex stdout JSONL stream (often 200-500KB when populated), the stderr file (small but growing), and the grep-over-rollout that reads file content. Across 5 rounds and a long conversation, cache-read residue accumulated to ~48M tokens per invocation.

### 12.2 The split

Claude Code's Agent tool dispatches a fresh subagent with its own isolated context. When the subagent returns, its context is discarded; only its final text message crosses to main. By making the runner subagent own every codex-exec artifact and return only a ~1KB JSON summary file plus the small review file path, the main thread no longer pays the residue tax.

The runner uses a two-channel protocol: the authoritative structured result is written to `/tmp/codex-runner-result-${REVIEW_ID}.json`, and the runner's final message is a single `RUNNER_RESULT_AT: <path>` line. Main extracts the path with a tolerant regex (markdown fences and minor wrapping do not break parsing) and reads the JSON file directly. This avoids the brittle "raw JSON in message" contract that Haiku's conversational output style would otherwise stress.

### 12.3 Retry budget — single owner

Retry lives in the runner alone, across ALL failure types (launch_failure, timeout, stderr-infrastructure-error). Runner retries once internally (same ATTEMPT_ID-rotation as pre-refactor's round-level retry). Main treats EVERY failure result as terminal-for-this-round: on the initial Step-4 dispatch, terminal = abort; on the Step-7 resume dispatch, terminal = route to fallback (fresh-exec, which is a new round with its own budget).

**The full invariant:** *exactly 2 codex invocations per round, maximum, across all failure types.* The pre-refactor skill had the same budget; the refactor does not loosen it. Putting the retry at a single layer (runner) prevents the worst-case compounding that would occur if main also re-dispatched on failure — specifically Round-2 finding #1 identified that a user-offered timeout retry → fresh-exec cascade could produce up to 6 invocations per round if retry logic existed at multiple layers. Terminal-at-main closes this lane.

Fresh-exec fallback is a *separate round* that consumes a round counter slot and gets its own fresh 2-attempts budget; this matches pre-refactor §2.4 where fresh-exec also counted as one round.

### 12.4 Archival ownership

Resume-to-fresh-exec fallback requires archiving the failed-resume stdout/stderr so they survive the next dispatch's file-reuse. Pre-refactor, main did this via its own `mv`. Post-refactor the runner does it inside Step R5 (only on resume failure). Rationale: main never references `/tmp/codex-stdout-*` or `/tmp/codex-stderr-*` paths in its own Bash argv, strengthening the isolation claim (no leak by path-reference). Archived paths are returned in the result JSON as `archived_stdout` / `archived_stderr`; main includes them in Step 9 cleanup per its unchanged `rm` glob.

### 12.5 Model choice

The runner is a pure pipeline executor: parse inputs, call Bash, validate output, retry once, archive on resume failure, write result JSON. No code understanding, no severity judgment, no review interpretation. Haiku 4.5 is sufficient and ~15× cheaper per token than Opus. The main thread (Opus) keeps all judgment work (applying fixes, deciding round progression, user interaction).

To avoid confusion, the runner's input schema uses `CODEX_MODEL` (the model codex CLI launches) — distinct from the runner's OWN model, which is passed via the Agent tool's `model: "haiku"` parameter. The names are intentionally non-overlapping.

### 12.6 Invariants preserved

The attempt-scoped `ADVERSARIAL-REVIEW-SESSION` marker (round-7 finding) and positive content-bind on rollout files (round-6 finding) both live in the runner. The orchestration-level invariants (5-round cap, resume-before-fresh-exec preference, not-verified terminal state, §2.4.4 user-facing warning on secondary-find zero) all stay in main or flow through the runner's `user_warning` channel.

### 12.7 Plan Mode inheritance (populated by Task 7 Step 4 — hypothesis until verified)

> **Status:** this subsection is empirically determined by Task 7 Step 4's Plan Mode smoke test. Until that task runs and the implementer updates this subsection with observed behavior, treat every claim here as a HYPOTHESIS, not documented fact.

**Hypothesis to test:** Plan Mode restrictions propagate from main to any subagent main dispatches; the subagent inherits limitations on Write/Edit/Bash. If this holds:
- Main's Write to `/tmp/codex-body-*.md` may trigger a permission prompt or exit Plan Mode.
- The runner's Writes to `/tmp/codex-prompt-*.md` (Step R2, including the mtime-bump repeat Write) may also trigger prompts.
- The runner's `codex exec` (read-only sandbox) should be unaffected since it writes nothing to the user's repo.

**What Task 7 Step 4 must determine:**
1. Does dispatching the Agent tool from Plan Mode work (is it blocked, does it prompt, does it just work)?
2. Does the subagent inherit Plan Mode, or does it run as unrestricted?
3. If Plan Mode propagates, does the subagent's Write (for the repeat-Write mtime bump) fail, prompt, or succeed silently?

**After Task 7 Step 4 runs, REWRITE this subsection with the observed answers.** Delete the hypothesis framing and state the observed behavior as fact. Until then, anyone reading §12.7 should understand it is speculative.
```

- [ ] **Step 3: Commit both files**

```bash
cd /home/dementev/sources/adversarial-review
git add AGENTS.md DESIGN.md
git commit -m "$(cat <<'EOF'
docs: document subagent architecture in AGENTS.md and DESIGN.md

AGENTS.md gets a new Architecture section with boundary
invariants. DESIGN.md §3 explains the residue problem, the
Agent-tool split, and why Haiku is sufficient for the runner.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Verify the refactor works end-to-end (GREEN)

**Files:** (no modifications to skill files — verification only)

- [ ] **Step 1: Install the refactored skill to the standard location**

Claude Code invokes `/adversarial-review` by loading the INSTALLED skill at `~/.claude/skills/adversarial-review/`, not the source checkout. Without this step, `/adversarial-review code` would run the OLD skill — false positive/negative on the GREEN gate.

Symlink (not copy — keeps the install in sync with the branch):

```bash
# Back up any existing install first
if [ -e ~/.claude/skills/adversarial-review ] && [ ! -L ~/.claude/skills/adversarial-review ]; then
  mv ~/.claude/skills/adversarial-review ~/.claude/skills/adversarial-review.bak-$(date +%s)
fi
mkdir -p ~/.claude/skills
ln -sfn /home/dementev/sources/adversarial-review ~/.claude/skills/adversarial-review
```

Verify the install points at the branch AND that the resolved basename matches the skill name (some Claude Code versions derive the skill name from `realpath` basename; a mismatch causes silent skill-registry rejection):

```bash
readlink ~/.claude/skills/adversarial-review
ls ~/.claude/skills/adversarial-review/references/runner.md
git -C ~/.claude/skills/adversarial-review/ branch --show-current
# Basename must match the skill name exactly:
test "$(basename "$(readlink -f ~/.claude/skills/adversarial-review)")" = "adversarial-review" && echo OK
```

Expected: symlink target matches the source repo, `references/runner.md` exists, current branch is `feature/subagent-orchestration`, basename assertion prints `OK`.

If the basename assertion fails (source checkout is named differently), either (a) rename the source dir to `adversarial-review`, or (b) copy instead of symlink: `cp -r /home/dementev/sources/adversarial-review ~/.claude/skills/adversarial-review` (remember to re-copy after each source-branch commit during verification iteration).

- [ ] **Step 2: Prepare a trivial change to review**

```bash
cd /home/dementev/sources/adversarial-review
echo "" >> README.md
echo "<!-- trivial change for adversarial-review smoke test -->" >> README.md
git diff README.md
```

Expected: one added trailing comment line.

- [ ] **Step 3: Code-mode smoke test — invoke `/adversarial-review code`**

From a FRESH Claude Code session (so the baseline context is minimal and measurable), run:

```
/adversarial-review code
```

Note the session's context-token count BEFORE invoking (from `/status` or similar) and AFTER. Expected delta: ≤15K tokens total growth (review markdown 2-5K + runner result JSON ~1K + orchestration overhead). NOT ~48M — that would mean the refactor did not take effect.

Observe during the run:
- Step 4 dispatches an Agent tool call (visible in the session as a subagent launch).
- The subagent's final message is a single `RUNNER_RESULT_AT: /tmp/codex-runner-result-...json` line (visible as the Agent result).
- Main thread reads the result JSON and the review file, then shows the review verbatim.

- [ ] **Step 4: Plan-mode smoke test — invoke `/adversarial-review plan`**

With no code changes in the tree (revert the README hunk first: `git checkout README.md`), describe a trivial plan in Plan Mode OR pass an explicit plan file path. Verify that `/adversarial-review plan` completes end-to-end including subagent dispatch.

Specifically watch for:
- Does dispatching the Agent tool from Plan Mode prompt the user for permission? If yes, document in DESIGN.md §12.7.
- Does the subagent's own Write tool for `/tmp/codex-prompt-*` succeed? If it prompts for permission, note in DESIGN.md §12.7.
- Does the subagent's codex exec launch succeed under Plan Mode? (Codex is read-only, so should be fine.)

If Plan Mode interferes with dispatch or Write, document the expected behavior (auto-exit, permission prompt, or error) in DESIGN.md §12.7 based on observation. Do NOT leave §12.7 speculative — replace with observed reality.

- [ ] **Step 5: Check main-thread isolation invariant**

After each smoke test, inspect the session's tool-call transcript. The main thread's Bash calls should include only:
- `git rev-parse --show-toplevel` and `git rev-parse --show-superproject-working-tree`
- `git diff --name-only`, `git diff --cached --name-only`, `git diff --name-only ${BASE_BRANCH}...HEAD` (code mode only)
- `git symbolic-ref` for base branch (code mode only)
- The Step 9 `rm` cleanup command (on approved/concluded terminal states)

It should NOT include:
- Any `cat ... | codex exec ...` invocation
- Any read of `/tmp/codex-stdout-*.jsonl`, `/tmp/codex-stderr-*.txt`, or rollout files
- Any `find ~/.codex/sessions ...`
- Any `mv /tmp/codex-stdout-*` or `mv /tmp/codex-stderr-*` (archival now lives in the runner)

Allowed reads in main: the runner result JSON at `/tmp/codex-runner-result-*.json`, the review file at `/tmp/codex-review-*.md`, and the skill's `references/runner.md`.

If any disallowed call appears in main, a step is still running in-process — return to that step and fix.

- [ ] **Step 6: Verify Step 9 cleanup is complete**

After a successful (APPROVED) review concludes, verify no `/tmp/codex-*-${REVIEW_ID}*` files remain:

```bash
ls /tmp/codex-*-${REVIEW_ID}* 2>/dev/null
# Expected: empty
```

If files remain, the Step 9 cleanup glob is missing a pattern (likely `codex-body-*.md` or `codex-runner-result-*.json` — added in Task 5 Step 3).

- [ ] **Step 7: Revert the trivial change**

```bash
cd /home/dementev/sources/adversarial-review
git checkout README.md 2>/dev/null || true
```

- [ ] **Step 8: Final commit (only if verification surfaced fixes)**

If Steps 3-6 passed without requiring changes to SKILL.md, runner.md, AGENTS.md, or DESIGN.md, no commit is needed. If any were made:

```bash
cd /home/dementev/sources/adversarial-review
git add SKILL.md references/runner.md AGENTS.md DESIGN.md
git commit -m "$(cat <<'EOF'
fix(skill): findings from subagent refactor E2E verification

<describe what was found and fixed; include observed Plan Mode
behavior from Step 4 if §12.7 was updated with reality>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Open PR for review

**Files:** (none — git/gh operations)

- [ ] **Step 1: Push the branch**

```bash
cd /home/dementev/sources/adversarial-review
git push -u origin feature/subagent-orchestration
```

- [ ] **Step 2: Create PR**

```bash
gh pr create --title "refactor(skill): extract codex orchestration to Haiku subagent" --body "$(cat <<'EOF'
## Summary
- Split adversarial-review into main orchestrator + thin Haiku runner subagent
- Main keeps mode detection, review display, code fixes, round control
- Runner owns codex-exec mechanics (launch, strict checks, session-id capture, retry)
- Eliminates ~48M-token cache-read residue in main thread per invocation

## Architecture
See DESIGN.md §3 and AGENTS.md Architecture section.

## Test plan
- [x] E2E smoke test on trivial README change — main thread no longer reads codex stdout/stderr/rollout files
- [x] Review file path crosses subagent→main boundary correctly
- [x] Resume path (rounds 2-5) dispatches runner with OPERATION=resume
- [x] Fresh-exec fallback still consumes one round from 5-round counter

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 3: Return PR URL**

Capture the URL printed by `gh pr create` and report it to the user.

---

## Self-Review Checklist

- **Spec coverage:** Tasks 1-2 create `references/runner.md` with the full runner spec. Tasks 3-4 refactor the two dispatch points (initial in Step 4, resume + fresh-exec fallback in Step 7) with SKILL_DIR resolution and two-channel result protocol. Task 5 updates Rules and extends Step 9 cleanup glob. Task 6 appends DESIGN.md §12 (no renumbering). Task 7 verifies end-to-end with skill-install substep, plan-mode test, and residue-tax measurement. Task 8 ships.
- **No placeholders:** every prompt body is inlined verbatim (no "copy from pre-refactor" forward references). Every command, every JSON schema is written out.
- **Type consistency:** runner's JSON output fields (`result`, `verdict`, `review_file`, `codex_session_id`, `attempt_id`, `errors`, `archived_stdout`, `archived_stderr`, `user_warning`) are used with the same names in Tasks 3 and 4's main-thread dispatch tables.
- **Retry budget invariant (all failure types):** runner retries ONCE internally on ANY failure — launch_failure, timeout, stderr-infra-error. Main treats EVERY failure as TERMINAL for the current round — no user-offered retries, no main re-dispatches. Total codex invocations per round ≤ 2 across all failure types. Fresh-exec fallback is a NEW round with its own 2-attempt budget. Closes Round-2 finding #1 (timeout lane no longer compounds).
- **Session-id handling on resume:** when secondary-path find returns zero, the runner returns `codex_session_id: null` for resume AND sets `user_warning` with the §2.4.4 diagnostic. Main surfaces `user_warning` on a single line before proceeding — preserves pre-refactor user-visible behavior.
- **Isolation invariant:** main reads the runner result JSON, the review file, and `references/runner.md`. Main never references `/tmp/codex-stdout-*`, `/tmp/codex-stderr-*`, rollout paths, or `find ~/.codex/sessions` in any Bash argv. Archival `mv` lives in the runner.
- **Naming disambiguation:** `CODEX_MODEL` / `CODEX_REASONING` (runner input) vs `model: "haiku"` (Agent tool dispatch). Never conflated.
- **DESIGN.md ordering:** new section is `§12. Subagent architecture` appended at the end. Existing §§1-11 are NOT renumbered (AGENTS.md §10.2 invariant).
- **Skill-install for verification:** Task 7 Step 1 symlinks the branch to `~/.claude/skills/adversarial-review/` so `/adversarial-review` invokes the refactored skill, not the previously-installed version.
- **Residue-tax measurement:** Task 7 Step 3 notes context-token before/after deltas and asserts ≤15K growth (not ~48M) — verifies the refactor actually fixed the bug, not merely relocated code.
- **Plan Mode:** Task 7 Step 4 runs an explicit `/adversarial-review plan` smoke test. DESIGN.md §12.7 is explicitly labeled as a HYPOTHESIS until Task 7 Step 4 replaces it with observed reality.
- **Placeholder substitution:** Task 3 Step 2 has an explicit table of placeholders (`${BASE_BRANCH}`, `${REPO_ROOT}`, `<file list from --name-only>`, etc.) that main MUST resolve before the Write-to-disk step. Pre-refactor did this implicitly; post-refactor needs it explicit.
- **User-override capture:** Task 3 Step 2 tells main to capture `CODEX_MODEL` / `CODEX_REASONING` from user arguments (defaults `gpt-5.4`, `high`), preserving the `/adversarial-review xhigh` and `/adversarial-review model:...` override features.
- **SKILL_DIR resolution:** priority 1 is well-known install path; priority 2 is git toplevel (dev fallback); no attempt to read Claude's own system prompt. Closes Round-2 finding #2 (hand-wave).
- **RUNNER_RESULT_AT regex:** unanchored `RUNNER_RESULT_AT:\s+(\S+)` applied to entire subagent message, with filesystem Glob fallback on the deterministic path. Both spec sites (runner contract and main-thread parsing) use the same form.
- **`user_warning` timing:** emitted as its own `⚠ <message>` turn BEFORE the Step 5 verbatim-review message. Does not pollute the Step 5 header.
- **Symlink basename:** Task 7 Step 1 verifies `basename $(readlink -f ...)` equals `adversarial-review` to catch skill-loader basename-resolution rejection.
