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

**Two stages** — works on both planning and implementation:
- **Plan review** — review the plan BEFORE writing code. Catch architecture
  mistakes, missing steps, and risks early
- **Code review** — review the implementation. Bugs, security, data loss
- **Code-vs-plan** — verify the implementation matches the plan

**Lightweight** — one file, no server, no broker, no dependencies beyond
the reviewer CLI. Compare with [codex-plugin-cc](https://github.com/openai/codex-plugin-cc):
~15 JS modules, App Server, JSON-RPC broker, lifecycle hooks.
This skill is a text instruction that any AI agent can interpret.

**Iterative** — Claude doesn't just show the review and stop.
It actively fixes issues based on reviewer feedback and resubmits
for re-review. Up to 5 rounds until approved.

**Designed for extensibility** — the skill relies on basic agent capabilities:
run a command, read a file, edit a file. The prompts and review workflow
are model-agnostic. Currently uses OpenAI Codex as the reviewer;
adding other backends (Gemini, local models) is on the roadmap.

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

## Installation

### Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- [OpenAI Codex CLI](https://github.com/openai/codex): `npm install -g @openai/codex`
- OpenAI API key (`OPENAI_API_KEY` environment variable)

### Setup

```bash
# Clone the repository
git clone https://github.com/<your-username>/adversarial-review.git

# Symlink into Claude Code skills directory
ln -s "$(pwd)/adversarial-review" ~/.agents/skills/adversarial-review
```

After symlinking, the skill is available as `/adversarial-review` in Claude Code.

### Recommended permissions

The skill runs git, codex, and `/tmp` write commands that will trigger
permission prompts. To avoid repeated confirmations, add these to your
`.claude/settings.local.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(git diff*)",
      "Bash(git status*)",
      "Bash(git symbolic-ref*)",
      "Bash(git rev-parse*)",
      "Bash(timeout 600 codex exec *)",
      "Write(/tmp/codex-plan-*)",
      "Write(/tmp/codex-prompt-*)"
    ]
  }
}
```

**Note:** The `codex exec` rule allows any `codex exec` invocation wrapped
in `timeout 600`. The skill only uses read-only mode (`-s read-only`), but
Claude Code's permission patterns are prefix-based and cannot enforce flag
constraints. If you prefer tighter control, omit the `codex exec` rule and
approve each review invocation manually.

## Usage

```
# Auto-detect what to review
/adversarial-review

# Review a plan
/adversarial-review plan

# Review code changes
/adversarial-review code

# Review a specific file
/adversarial-review path/to/plan.md

# Use higher reasoning effort
/adversarial-review xhigh

# Use a different model
/adversarial-review model:gpt-5.3-codex
```

## Prompt architecture

The skill uses XML-structured prompts inspired by adversarial review methodology:

- **`<role>`** — adversarial reviewer, defaults to skepticism
- **`<operating_stance>`** — break confidence, not validate
- **`<attack_surface>`** — concrete checklist: auth, data integrity,
  race conditions, rollback safety, schema drift, error handling, observability
- **`<finding_bar>`** — every finding must answer 4 questions:
  what can go wrong, why this code is vulnerable, impact, recommendation
- **`<scope_exclusions>`** — no style, naming, or speculative comments
- **`<calibration>`** — one strong finding > five weak ones

## Example output

See [examples/review-output.md](examples/review-output.md) for a sample
adversarial review output.

## Known limitations

- **Plan Mode and `/tmp` writes.** In Claude Code Plan Mode, writing review
  prompts to `/tmp` may trigger a permission prompt or cause Plan Mode to exit.
  This does not affect review correctness — the review mode is already determined,
  and only the plan file and temp files are modified.
- **`resume` inherits sandbox.** The `codex exec resume` command does not accept
  `-s` (sandbox) — sandbox is inherited from the original session. The first
  `codex exec` call sets the sandbox to `read-only`, and all subsequent resume
  rounds use the same setting.

## Roadmap

- [ ] Gemini as alternative reviewer backend
- [ ] Local model support (Ollama, llama.cpp)
- [ ] CI integration (GitHub Actions)
- [ ] Multi-reviewer mode (parallel review by multiple models)

## Inspiration

The adversarial prompt structure was developed after studying
[openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) (Apache-2.0)
— the official OpenAI plugin for code review with Codex in Claude Code.

What we borrowed as ideas:
- XML-structured prompts (`<role>`, `<operating_stance>`, `<attack_surface>`, etc.)
- Adversarial stance: "break confidence, not validate"
- Attack surface checklist approach
- Finding bar: 4 questions each finding must answer
- Calibration rules: prefer strong findings over weak ones

What we did differently:
- **Iterative loop** — Claude actively fixes issues and resubmits (vs "stop and ask user")
- **Plan review** — reviews plans before code, not just code
- **Single file** — one SKILL.md vs 15+ JS modules with App Server
- **Verbatim output** — reviewer findings shown as-is, not rephrased

## License

Apache-2.0 — see [LICENSE](LICENSE).
