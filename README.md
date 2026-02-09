# Claude Peer Review

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that runs multi-round scientific peer reviews between Claude and an external AI model.

**Reviewer** (external model) critiques the plan. **Author** (Claude) responds and revises. **Editor** (you) makes the final call.

## How it works

```
 You provide a plan/spec/design document
     |
 Claude launches an external reviewer (Codex, Gemini, or Claude CLI)
     |
 Reviewer writes a structured review with GO/NO-GO assessment
     |
 Claude responds to each finding and describes revisions
     |
 Repeat for N rounds (default 2), then you decide:
     GO | Additional round | Different reviewer | NO-GO
```

Each round produces a pair of artifacts (`*-review.md` and `*-review-response.md`) that form a complete audit trail.

## Install

Copy the skill into your Claude Code skills directory:

```bash
# User-level (available in all projects)
git clone https://github.com/kenwilliford/claude-peer-review.git \
    ~/.claude/skills/peer-review

# Or project-level
git clone https://github.com/kenwilliford/claude-peer-review.git \
    .claude/skills/peer-review
```

## Prerequisites

At least one reviewer CLI must be installed:

| Reviewer | Install | Auth |
|----------|---------|------|
| **Codex** | `npm install -g @openai/codex` | `codex login` |
| **Gemini** | `npm install -g @google/gemini-cli@latest` | Google OAuth or API key |
| **Claude** (self-review) | Included with Claude Code | N/A |

Cross-model review (Codex or Gemini) is recommended. Self-review provides structural value but less diversity of perspective.

## Usage

```bash
/peer-review path/to/plan.md
/peer-review path/to/plan.md --output-dir=reviews/
/peer-review path/to/plan.md --rounds=3
```

The skill will detect available CLIs and ask you to choose a reviewer.

## Permissions

This skill works in Claude Code's **default permission mode** -- no `--dangerously-skip-permissions` needed. The skill's bash usage is limited to:

- **CLI detection**: `which`, `--version`, config file reads
- **Temp file management**: `mktemp`, `cat` (heredoc), `rm`
- **Reviewer invocation**: `timeout 300 <cli> ...`

On first use you'll be prompted to approve each command (~4-5 prompts per review round). For uninterrupted operation, add these rules to your project's `.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Bash(which:*)",
      "Bash(grep:*)",
      "Bash(mktemp:*)",
      "Bash(cat:*)",
      "Bash(timeout:*)",
      "Bash(rm -f /tmp/peer-review:*)",
      "Bash(PROMPT_FILE:*)",
      "Bash(codex:*)",
      "Bash(gemini:*)",
      "Bash(claude:*)"
    ]
  }
}
```

You can trim this to just the reviewer you use. For example, if you only use Codex, you don't need `Bash(gemini:*)` or `Bash(claude:*)`.

The Claude self-review subprocess runs with `--tools ""` (all tools disabled) -- it only reads stdin and writes to stdout.

## Output

For a plan named `my-plan.md` with 2 rounds:

| File | Author | Round |
|------|--------|-------|
| `my-plan-review.md` | Reviewer | 1 |
| `my-plan-review-response.md` | Claude | 1 |
| `my-plan-review2.md` | Reviewer | 2 |
| `my-plan-review2-response.md` | Claude | 2 |

The final `*-response.md` is the key decision artifact -- it summarizes the full review process, residual risks, and implementation readiness.

## Design decisions

- **Plan immutability**: The original plan file is never modified. All revisions are described in response documents. This keeps the review record coherent and auditable.
- **Temp file + stdin**: Prompts are written to a temp file and piped via stdin to avoid argument length limits and shell metacharacter issues.
- **Structured editor decision**: After the final round, you're presented with explicit options (GO / Additional round / Different reviewer / NO-GO) rather than an open-ended prompt.
- **Resume support**: If Round 1 artifacts already exist, the skill offers to resume at Round 2.

## License

MIT
