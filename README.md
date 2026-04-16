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
// Git: diff, status, branch detection
"Bash(git diff*)",
"Bash(git status*)",
"Bash(git symbolic-ref*)",
"Bash(git rev-parse*)",
// Codex: review execution (always wrapped in timeout)
"Bash(timeout 600 codex exec *)",
// Temp files: prompts, plans, output capture
"Write(/tmp/codex-plan-*)",
"Write(/tmp/codex-prompt-*)",
"Read(/tmp/codex-review-*)",
"Read(/tmp/codex-stderr-*)",
// Cleanup and output piping
"Bash(rm -f /tmp/codex-*)",
"Bash(tee *)"
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
      "Bash(timeout 600 codex exec *)",
      "Write(/tmp/codex-plan-*)",
      "Write(/tmp/codex-prompt-*)",
      "Read(/tmp/codex-review-*)",
      "Read(/tmp/codex-stderr-*)",
      "Bash(rm -f /tmp/codex-*)",
      "Bash(tee *)"
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
The skill uses `codex exec resume <session-id>` for rounds 2+. If the session
expired or the ID was not captured, the skill falls back to a fresh `codex exec`
automatically. No action needed.

**Plan Mode exits when writing temp files.**
In Claude Code Plan Mode, writing to `/tmp` may trigger a permission prompt
or exit Plan Mode. This is a known Claude Code limitation. It does not affect
review correctness.

## Known limitations

- **Plan Mode and `/tmp` writes.** Writing review prompts to `/tmp` may trigger
  a permission prompt or cause Plan Mode to exit. Does not affect review correctness.
- **`resume` inherits sandbox.** `codex exec resume` does not accept `-s` —
  sandbox is inherited from the original session (always `read-only`).

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
