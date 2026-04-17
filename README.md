# Adversarial Review

Claude Code skill for adversarial AI code and plan review.

One AI writes the code. Another tears it apart. Iterate until approved.

## What is this

Most AI code review tools validate your changes — "looks good, maybe add tests."
Adversarial review does the opposite: the reviewer **defaults to skepticism**
and tries to break confidence in the change. It looks for what will fail
in production, not what might be nice to improve.

This is a [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code)
— a single `SKILL.md` file that teaches Claude how to run adversarial reviews
through an external AI model (currently OpenAI Codex).

## Key features

- **Plan review** — review the plan BEFORE writing code. Catch architecture
  mistakes, missing steps, and risks early
- **Code review** — review the implementation. Bugs, security, data loss
- **Code-vs-plan** — verify the implementation matches the plan
- **Iterative** — Claude fixes issues based on reviewer feedback and resubmits
  for re-review. Up to 5 rounds until approved
- **Lightweight** — one `SKILL.md` file, no server, no broker. Compare with
  [codex-plugin-cc](https://github.com/openai/codex-plugin-cc):
  ~15 JS modules, App Server, JSON-RPC broker, lifecycle hooks

## How it works

```
┌─────────┐     ┌──────────┐     ┌─────────┐
│  Claude  │────>│ Reviewer  │────>│  Claude  │
│  (code)  │     │ (Codex)   │     │  (fix)   │
└─────────┘     └──────────┘     └─────────┘
     ^                                 │
     │         ┌──────────┐            │
     └─────────│ Reviewer  │<───────────┘
               │(re-review)│
               └──────────┘
                    │
              VERDICT: APPROVED
```

### Three modes

| Mode | What it reviews | When to use |
|------|----------------|-------------|
| `plan` | Implementation plan | Before writing code |
| `code` | Git diff (unstaged, staged, or branch) | After writing code |
| `code-vs-plan` | Code changes against the plan | Verify implementation matches plan |

Mode is auto-detected from context, or you can force it with an argument.

## Quick start

### 1. Prerequisites

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) and
[OpenAI Codex CLI](https://github.com/openai/codex) must be installed.

Verify both are available:

```bash
claude --version   # Claude Code CLI
codex --version    # OpenAI Codex CLI (>= 0.115.0)
```

If Codex is missing: `npm install -g @openai/codex`

**Authentication.** Codex needs an OpenAI account. Either:
- Sign in interactively: `codex` (opens browser)
- Or set `CODEX_API_KEY` env var for non-interactive use

### 2. Install the skill

```bash
git clone https://github.com/dementev-dev/adversarial-review.git
ln -s "$(pwd)/adversarial-review" ~/.agents/skills/adversarial-review
```

Verify the skill is visible to Claude Code:

```bash
ls -la ~/.agents/skills/adversarial-review/SKILL.md
```

### 3. Add permissions

The skill runs `git`, `codex exec`, and writes temp files to `/tmp`.
Without pre-approved permissions, Claude Code will prompt for each action.

**Where to add.** Since the skill is installed globally
(`~/.agents/skills/`), permissions should go into the global config
so they work in any project:

| Install scope | Config file |
|---------------|-------------|
| Global (recommended) | `~/.claude/settings.json` |
| Single project | `<project>/.claude/settings.local.json` |

Merge the following rules into the `permissions.allow` array of the
chosen config file:

```jsonc
// --- adversarial-review permissions ---
// Git: diff, status, branch detection, repo root, submodule check
"Bash(git diff*)",
"Bash(git status*)",
"Bash(git symbolic-ref*)",
"Bash(git rev-parse*)",
// Pre-exec timestamp capture (for session-id filesystem fallback)
"Bash(date +%s)",
// Codex: initial launch (uses -C; prompt fed via cat | pipe for env portability)
"Bash(cat /tmp/codex-prompt-* | timeout 600 codex exec *)",
// Codex: resume (cd prefix because resume has no -C flag; prompt via cat | pipe)
"Bash(cd * && cat /tmp/codex-resume-prompt-* | timeout 600 codex exec resume *)",
// Session-id filesystem fallback (newest rollout file in ~/.codex/sessions/)
"Bash(find ~/.codex/sessions*)",
// Diagnostic aid when filesystem fallback finds nothing
"Bash(ls -t ~/.codex/sessions*)",
// Temp files: prompts (initial + resume), plans, review output, JSONL stdout, stderr
"Write(/tmp/codex-plan-*)",
"Write(/tmp/codex-prompt-*)",
"Write(/tmp/codex-resume-prompt-*)",
"Read(/tmp/codex-review-*)",
"Read(/tmp/codex-stdout-*)",
"Read(/tmp/codex-stderr-*)",
// Cleanup
"Bash(rm -f /tmp/codex-*)"
```

<details>
<summary>Full example (if the config file is empty or does not exist)</summary>

```jsonc
{
  "permissions": {
    "allow": [
      // adversarial-review
      "Bash(git diff*)",
      "Bash(git status*)",
      "Bash(git symbolic-ref*)",
      "Bash(git rev-parse*)",
      "Bash(date +%s)",
      "Bash(cat /tmp/codex-prompt-* | timeout 600 codex exec *)",
      "Bash(cd * && cat /tmp/codex-resume-prompt-* | timeout 600 codex exec resume *)",
      "Bash(find ~/.codex/sessions*)",
      "Bash(ls -t ~/.codex/sessions*)",
      "Write(/tmp/codex-plan-*)",
      "Write(/tmp/codex-prompt-*)",
      "Write(/tmp/codex-resume-prompt-*)",
      "Read(/tmp/codex-review-*)",
      "Read(/tmp/codex-stdout-*)",
      "Read(/tmp/codex-stderr-*)",
      "Bash(rm -f /tmp/codex-*)"
    ]
  }
}
```

</details>

**Security note:** The `codex exec` rule allows any `codex exec` invocation
wrapped in `timeout 600`. The skill only uses read-only mode (`-s read-only`),
but Claude Code's permission patterns are prefix-based and cannot enforce flag
constraints. If you prefer tighter control, omit the `codex exec` rule and
approve each invocation manually.

### 4. Use

```bash
/adversarial-review                       # auto-detect mode
/adversarial-review plan                  # force plan review
/adversarial-review code                  # force code review
/adversarial-review path/to/f             # review a specific file
/adversarial-review xhigh                 # higher reasoning effort
/adversarial-review model:gpt-5.3-codex   # use a different model
```

## Prompt architecture

The skill uses XML-structured prompts with adversarial stance:

- **`<role>`** — adversarial reviewer, defaults to skepticism
- **`<operating_stance>`** — break confidence, not validate
- **`<attack_surface>`** — concrete checklist: auth, data integrity,
  race conditions, rollback safety, schema drift, error handling, observability
- **`<finding_bar>`** — every finding must answer 4 questions:
  what can go wrong, why vulnerable, impact, recommendation
- **`<scope_exclusions>`** — no style, naming, or speculative comments
- **`<calibration>`** — one strong finding > five weak ones

## Example output

See [examples/review-output.md](examples/review-output.md) for a sample review.

## Troubleshooting

**`codex exec` exits with model error.**
Some models are unavailable with ChatGPT accounts (e.g. `o3-mini`).
The default `gpt-5.4` works with both ChatGPT and API key auth.
Override with `/adversarial-review model:<name>`.

**Permission prompts on every action.**
Add the permissions from the [setup section](#3-add-permissions). Check that
the file is valid JSON and in the right location (project `.claude/settings.local.json`
or global `~/.claude/settings.json`).

**Codex hangs / timeout (exit code 124).**
All `codex exec` calls are wrapped in `timeout 600` (10 minutes). If you see
exit code 124, the reviewer did not respond in time. Retry — this is usually
transient.

**Resume fails with session error.**
The skill uses `codex exec resume <session-id>` for rounds 2+. On failure
(non-zero exit, `thread/resume failed` in stderr, or a malformed review),
the skill does NOT silently fall back. In an interactive session it asks
whether to run a fresh `codex exec` (higher token cost) or conclude the
review as NOT VERIFIED. In headless runs it decides based on the maximum
severity of the last successful round's findings: critical/high → fresh
exec; medium-only → conclude as NOT VERIFIED.

**`--json` stdout is empty in my Claude Code session (session ID capture noise).**
In some Claude Code sandbox configurations codex's `--json` event stream is
suppressed when stdout is redirected to a file — the `/tmp/codex-stdout-*.jsonl`
ends up 0 bytes even though the review itself (`-o /tmp/codex-review-*.md`)
completes correctly. The skill handles this automatically via a filesystem
fallback: when the JSONL stream is empty it extracts the session UUID from
the newest `~/.codex/sessions/YYYY/MM/DD/rollout-*-<UUID>.jsonl` filename
created since the pre-exec timestamp. Resume continues to work normally.

**"NOT VERIFIED" result.**
The skill applied fixes but the reviewer did not re-verify them (resume
failed or the operator chose to conclude). This is not an approval —
manually review the applied fixes before merging.

**Running inside a git submodule.**
`git rev-parse --show-toplevel` returns the submodule path, not the parent
repo. The skill warns you and scopes the review to the submodule. If you
meant to review the parent, invoke the skill from the parent working tree.

**Bare repository or not inside a work tree.**
The skill aborts at Step 2 with a clear message. Run it from inside a
git working tree.

**Plan Mode exits when writing temp files.**
In Claude Code Plan Mode, writing to `/tmp` may trigger a permission prompt
or exit Plan Mode. This is a known Claude Code limitation. It does not affect
review correctness.

## Known limitations

- **Plan Mode and `/tmp` writes.** Writing review prompts to `/tmp` may trigger
  a permission prompt or cause Plan Mode to exit. Does not affect review correctness.
- **`resume` inherits sandbox.** `codex exec resume` does not accept `-s` —
  sandbox is inherited from the original session (always `read-only`).
- **`resume` has no `-C` flag.** The skill captures `REPO_ROOT` via
  `git rev-parse --show-toplevel` at Step 2 and prefixes every resume with
  `cd '<REPO_ROOT>' && ...`. This requires paths without single quotes;
  pathological paths (containing `'`, `"`, `$`, backtick, newline) cause
  the skill to abort at Step 2.
- **Submodule scoping.** When invoked inside a submodule, the review is
  scoped to the submodule — `git rev-parse --show-toplevel` does not walk
  up to the parent. A warning is printed; invoke from the parent repo if
  you want parent scope.
- **GNU find on macOS.** The secondary session-id capture uses
  `find -newermt "@<epoch>"` and `-printf`, both GNU extensions. On
  macOS (BSD find) the skill's default command does not work; the skill
  states the *goal* of the step in SKILL.md and invites the model (or
  user) to substitute an equivalent BSD-compatible command. The skill
  has not been end-to-end tested on macOS.

## Roadmap

- [ ] Gemini as alternative reviewer backend
- [ ] Local model support (Ollama, llama.cpp)
- [ ] CI integration (GitHub Actions)
- [ ] Multi-reviewer mode (parallel review by multiple models)

## Inspiration

Adversarial prompt structure developed after studying
[openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) (Apache-2.0).

Borrowed ideas: XML-structured prompts, adversarial stance, attack surface
checklist, finding bar, calibration rules.

What we did differently:
- **Iterative loop** — Claude fixes issues and resubmits (not "stop and ask user")
- **Plan review** — reviews plans before code, not just code
- **Single file** — one SKILL.md vs 15+ JS modules
- **Verbatim output** — reviewer findings shown as-is, not rephrased

## License

Apache-2.0 — see [LICENSE](LICENSE).
