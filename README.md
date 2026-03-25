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

## Quick Start

### 1. Install the skill

Claude Code automatically discovers skills placed in `~/.claude/skills/`. Clone this repo there:

```bash
# User-level (available in all projects)
git clone https://github.com/kenwilliford/claude-peer-review.git \
    ~/.claude/skills/peer-review

# Or project-level (only in one project)
git clone https://github.com/kenwilliford/claude-peer-review.git \
    .claude/skills/peer-review
```

**Important:** Clone the entire repo, not just `SKILL.md`. The skill references template files in `references/` that must be present alongside it.

### 2. Install at least one reviewer CLI

Requires [Node.js](https://nodejs.org/) 18+ for npm installs.

| Reviewer | Install | Auth |
|----------|---------|------|
| **Codex** (recommended) | `npm install -g @openai/codex` | `codex login` |
| **Gemini** | `npm install -g @google/gemini-cli@latest` | [Google AI API key](https://ai.google.dev/) or Google OAuth |
| **Claude** (self-review) | Included with Claude Code | N/A |

Cross-model review (Codex or Gemini) is recommended. Self-review provides structural value but less diversity of perspective.

### 3. Verify installation

Start (or restart) Claude Code and type `/peer-review` — it should appear in autocomplete. If it doesn't, check that `~/.claude/skills/peer-review/SKILL.md` exists.

### 4. Run your first review

```bash
# In Claude Code, with a plan/spec file ready:
/peer-review path/to/plan.md
```

Claude will detect available reviewer CLIs, ask you to choose one, then run the review. Output files appear alongside your plan.

## Usage

```bash
/peer-review path/to/plan.md
/peer-review path/to/plan.md --output-dir=reviews/
/peer-review path/to/plan.md --rounds=3

# Autonomous mode (no human interaction — for use in automation)
/peer-review path/to/plan.md --auto-go --reviewer=codex
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

| File | Author | Purpose |
|------|--------|---------|
| `my-plan-review.md` | Reviewer | Round 1 review |
| `my-plan-review-response.md` | Claude | Round 1 response (describes revisions) |
| `my-plan-review2.md` | Reviewer | Round 2 review |
| `my-plan-review2-response.md` | Claude | Round 2 response |
| **`my-plan-revised.md`** | **Claude** | **Revised spec — implementation-ready** |

The original plan file is never modified. After the editor says GO, Claude produces `*-revised.md` by applying all accepted revisions from every round into a clean, standalone document. The review/response pairs are the audit trail showing how it got there.

## Updating

If you installed via `git clone`, update with:

```bash
cd ~/.claude/skills/peer-review && git pull
```

## Uninstalling

```bash
rm -rf ~/.claude/skills/peer-review
```

## Design decisions

- **Plan immutability during review**: The original plan stays unchanged during review rounds so reviewers see a consistent base. After GO, Claude produces a separate `*-revised.md` with all accepted changes applied.
- **Temp file + stdin**: Prompts are written to a temp file and piped via stdin to avoid argument length limits and shell metacharacter issues.
- **Structured editor decision**: After the final round, you're presented with explicit options (GO / Additional round / Different reviewer / NO-GO) rather than an open-ended prompt.
- **Resume support**: If Round 1 artifacts already exist, the skill offers to resume at Round 2.

## License

MIT
