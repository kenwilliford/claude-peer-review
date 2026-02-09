---
name: peer-review
description: >-
  Initiate a multi-round scientific peer review between Claude and an external
  model (Codex, Gemini, or Claude CLI). Use when you have a plan, spec, or
  design document that needs rigorous multi-model review before implementation.
---

# Peer Review Skill

## Purpose

Run a multi-round scientific peer review process where:
- **External model** acts as Reviewer (critical evaluation)
- **Claude** acts as Author (responds to feedback, revises)
- **Human** acts as Editor (makes final GO/NO-GO decision based on artifacts)

The skill detects available reviewer CLIs and lets the user choose which model
to use as the reviewer. Communication uses direct pipe-and-capture — the
reviewer CLI is invoked with the full prompt and its stdout is captured as the
review. The default is 2 rounds, but more can be requested.

## Invocation

```
/peer-review <plan-file> [--output-dir=<dir>] [--rounds=<N>]
```

**Arguments:**
- `<plan-file>`: Path to the plan/spec/design document to review
- `--output-dir`: Directory for review artifacts (default: same as plan file)
- `--rounds`: Number of review rounds (default: 2)

## Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                    STEP 0: MODEL SELECTION                          │
├─────────────────────────────────────────────────────────────────────┤
│  [Claude]    Detects available CLIs → asks user to choose reviewer  │
├─────────────────────────────────────────────────────────────────────┤
│                    ROUND 1 .. N (default N=2)                       │
├─────────────────────────────────────────────────────────────────────┤
│  [Claude]    Constructs prompt → pipes to reviewer CLI              │
│      ↓                                                              │
│  [Reviewer]  Reads prompt (plan + prior artifacts) → outputs review │
│      ↓                                                              │
│  [Claude]    Captures stdout → writes plan-reviewN.md               │
│              Analyzes review → writes plan-reviewN-response.md      │
│      ↓                                                              │
│  If reviewer issues NO-GO with Critical findings AND rounds remain: │
│      → continue to next round automatically                         │
│  Else if final round reached:                                       │
│      → proceed to Editor Decision                                   │
├─────────────────────────────────────────────────────────────────────┤
│                      EDITOR DECISION                                │
├─────────────────────────────────────────────────────────────────────┤
│  [Human]     Presented with structured options:                     │
│              GO — proceed to implementation                         │
│              Additional round — run another review round             │
│              Different reviewer — switch to a different model        │
│              NO-GO — abandon or rework outside this process          │
└─────────────────────────────────────────────────────────────────────┘
```

## Output Artifacts

For N rounds, the following files are produced:

| File | Author | Contains |
|------|--------|----------|
| `<name>-review.md` | Reviewer | Round 1 review with GO/NO-GO |
| `<name>-review-response.md` | Claude | Round 1 response addressing all concerns |
| `<name>-review2.md` | Reviewer | Round 2 review with GO/NO-GO |
| `<name>-review2-response.md` | Claude | Round 2 response |
| `<name>-review<N>.md` | Reviewer | Round N review (if additional rounds) |
| `<name>-review<N>-response.md` | Claude | Round N response (final = key decision artifact) |

The last `*-response.md` is always the **key decision artifact** for the editor.

## Execution Steps

### Step 0: Detect Available Reviewers and Ask User

When invoked, Claude MUST detect which reviewer CLIs are available and present
a choice to the user. **Do NOT skip this step. Do NOT default to self-review.**

**Detection commands** (run in parallel):

```bash
which codex 2>/dev/null && codex --version 2>&1    # Codex CLI
which gemini 2>/dev/null && gemini --version 2>&1   # Gemini CLI
which claude 2>/dev/null && claude --version 2>&1   # Claude CLI (separate instance)
```

**Detect Codex default model** (if codex is available):

```bash
grep '^model ' ~/.codex/config.toml 2>/dev/null | head -1 | sed 's/model *= *"\(.*\)"/\1/'
```

If this returns a model name, use it in the Codex option label. If it fails or
returns empty, fall back to displaying just "Codex" without a model name.

**Present options** using AskUserQuestion with header "Reviewer":

- For each detected CLI, list it as an option with model info
- Always include "Claude (self-review)" as last option — but label it clearly
  as single-model review (less rigorous than cross-model)
- Recommend cross-model options first

Example options (model name is detected dynamically):
- `Codex (<detected-model>)` — "Cross-model review using OpenAI reasoning model"
- `Gemini` — "Cross-model review using Google model"
- `Claude (self-review)` — "Same-model review — less diversity of perspective"

**If user selects Gemini**, ask a follow-up question for model choice with
header "Model":
- `gemini-2.5-pro (Recommended)` — "Strong reasoning, default model"
- `gemini-2.5-flash` — "Faster, lower quota usage"
- `gemini-3-pro` — "Highest quality (requires previewFeatures enabled)"

### Step 1: Read Plan and Prepare

After user selects reviewer:

1. Read the plan file (error if not found)
2. Read `references/REVIEW_TEMPLATE.md` for the review format
3. **Check for existing artifacts:** Look for `<name>-review.md` and
   `<name>-review-response.md` in the output directory. If found, ask the user
   via AskUserQuestion with header "Resume" and options:
   - `Resume at Round 2` — "Round 1 artifacts found, continue from where you left off"
   - `Start fresh` — "Discard existing artifacts and begin Round 1"
   If resuming, skip to the appropriate round with existing artifacts loaded.
4. Construct the reviewer prompt (see "Reviewer Prompt Construction" below)
5. Display: "Review initiated. Launching [Model] reviewer..."

### Step 2: Launch Reviewer (Round 1)

Write the constructed prompt to a temp file, then pipe it via stdin to the
selected reviewer CLI. Wrap each invocation in a 5-minute timeout.

**Why temp file + stdin:** Passing the prompt as a CLI argument would hit
argument length limits and break on shell metacharacters in the plan content.
Using a heredoc with quoted delimiter (`'PROMPT_EOF'`) prevents shell expansion.

```bash
# Write prompt to temp file (quoted delimiter prevents shell expansion)
PROMPT_FILE=$(mktemp /tmp/peer-review-XXXXXX.md)
cat > "$PROMPT_FILE" <<'PROMPT_EOF'
<reviewer-prompt content here>
PROMPT_EOF
```

Then invoke the reviewer CLI using stdin redirection:

#### Codex

```bash
timeout 300 codex exec --full-auto - < "$PROMPT_FILE"
```

The `-` argument tells Codex to read the prompt from stdin.

#### Gemini

**CRITICAL:** Always use `-o text` to force headless output. Without it, Gemini
renders its TUI which hangs on piped/non-interactive input.

```bash
timeout 300 gemini -o text --yolo -m <model> < "$PROMPT_FILE"
```

**Model selection:** Ask the user which Gemini model to use:

| Model | Flag value | When to recommend |
|-------|-----------|-------------------|
| `gemini-2.5-pro` | `-m gemini-2.5-pro` | Default — strong reasoning |
| `gemini-2.5-flash` | `-m gemini-2.5-flash` | Faster, lower quota usage |
| `gemini-3-pro` | `-m gemini-3-pro` | Best quality (requires previewFeatures) |

If the user doesn't specify, default to `gemini-2.5-pro`.

#### Claude CLI (self-review fallback)

```bash
timeout 300 claude -p --tools "" < "$PROMPT_FILE"
```

The `--tools ""` flag disables all tool use in the child process — it only
reads stdin and writes to stdout. No filesystem access, no bash execution.

**Note:** This runs a separate Claude instance. It provides structural review
value (second pass) but not model-diversity value. Label the review output
as "Claude (self-review)" to distinguish from cross-model reviews.

#### Cleanup

After capturing the reviewer's output, remove the temp file:

```bash
rm -f "$PROMPT_FILE"
```

### Step 3: Process Round 1 Review

After capturing the reviewer's stdout:

1. Save it as `<name>-review.md`
2. Analyze each finding
3. Describe revisions in the response document (see "Plan Immutability" below)
4. Write `<name>-review-response.md` using response template
5. If reviewer issued NO-GO with Critical findings AND more rounds remain,
   display: "Round 1 complete. Critical issues found — launching Round 2..."
   Otherwise display: "Round 1 complete. Launching Round 2 review..."

**If timeout or CLI failure:** Display error with the specific reviewer CLI
that failed and suggest troubleshooting. **Do NOT fall back to self-review
silently.**

#### Plan Immutability

Claude MUST NOT modify the original plan file during the review process. All
revisions are described in the response document (`*-review-response.md`).
The "original plan" passed to subsequent rounds is always the unmodified
original. After the full process completes, the human editor decides
whether and how to apply revisions to the plan.

**Rationale:** Preserving the original makes the review record coherent and
auditable — reviewers in later rounds can see exactly what changed and why.

### Step 4: Launch Reviewer (Round 2+)

Construct a follow-up round prompt that includes:
- The original plan (unmodified — see Plan Immutability)
- All prior review and response artifacts
- Instructions to assess whether concerns were addressed

Pipe to the same reviewer CLI using the same temp-file + stdin pattern as Step 2.

Repeat Steps 4–5 for each round up to the configured round cap.

### Step 5: Process Follow-up Review

After capturing the reviewer's stdout for round N:

1. Save as `<name>-review<N>.md` (e.g., `<name>-review2.md`)
2. Analyze concerns (focusing on residual issues)
3. Describe revisions in the response document (do not modify the original plan)
4. Write `<name>-review<N>-response.md`
5. If more rounds remain AND reviewer issued NO-GO with Critical findings,
   continue to next round automatically
6. If this is the final round, proceed to Step 6

### Step 6: Editor Decision

After the final round, present the human with a structured decision using
AskUserQuestion with header "Editor Decision":

- `GO — proceed to implementation` — "Review complete, plan is ready"
- `Additional round — run another review round` — "Extend the review by one more round"
- `Different reviewer — switch to a different model` — "Re-run with a different reviewer CLI"
- `NO-GO — abandon or rework outside this process` — "Stop review, plan needs significant rework"

Also display:
- Summary of all review rounds (findings count, assessments)
- Final GO/NO-GO from the reviewer (identify which model reviewed)
- Path to the final `*-response.md` for detailed review

If the user selects "Additional round," increment the round cap by 1 and loop
back to Step 4. If "Different reviewer," return to Step 0 (model selection).

## Reviewer Prompt Construction

The reviewer prompt sent to the external CLI MUST include:

1. **Role description**: "You are a peer reviewer..."
2. **Plan content**: The full plan text (inline in the prompt)
3. **Review format**: The structured review template (from `references/REVIEW_TEMPLATE.md`)
4. **Output instructions**: Write the review directly to stdout

**Round 1 Template:**

```
You are a scientific peer reviewer. Your task is to critically evaluate the
plan document below and write a structured review.

Read the plan carefully, then write your review following the template format
at the end. Output your review directly — nothing else.

Be rigorous, specific, and constructive. Every concern must include a
concrete recommendation. Acknowledge strengths as well as problems. Include
a GO/NO-GO assessment.

## Plan Document

{PLAN_CONTENT}

## Review Format

{REVIEW_TEMPLATE_CONTENT}
```

**Follow-up Round Template (Round 2+):**

```
You are a scientific peer reviewer conducting a follow-up review round.

In prior rounds, you reviewed the plan and raised concerns. The author has
responded and described revisions. Your task is to:

1. Assess whether prior concerns were adequately addressed
2. Identify any new issues introduced by revisions
3. Evaluate residual risk
4. Provide a GO/NO-GO assessment

Output your review directly — nothing else.

## Original Plan

{PLAN_CONTENT}

## Prior Review and Response Artifacts

{ALL_PRIOR_ARTIFACTS}

## Review Format

{REVIEW_TEMPLATE_CONTENT}

In addition to the standard review sections, include a "Prior Concern
Resolution" table at the top of your findings:

| Finding | Status | Notes |
|---------|--------|-------|
| F1 | Resolved / Partially Resolved / Unresolved | [Brief note] |
```

## Review Document Format

See `references/REVIEW_TEMPLATE.md` for the review format.
See `references/RESPONSE_TEMPLATE.md` for the response format.
See `references/CODEX_INSTRUCTIONS.md` for detailed reviewer role description.

## GO/NO-GO Criteria

Each review and response MUST include a GO/NO-GO assessment:

| Assessment | Meaning |
|------------|---------|
| **GO** | Plan is sound, ready for implementation |
| **CONDITIONAL GO** | Minor issues that can be addressed during implementation |
| **NO-GO** | Significant issues that must be resolved before proceeding |

## Error Handling

| Scenario | Action |
|----------|--------|
| No external CLI available | Warn user; offer Claude self-review with clear labeling |
| Reviewer CLI fails to start | Display error with CLI name; do NOT silently self-review |
| Reviewer timeout (5 min) | Display timeout error; suggest checking CLI auth/config |
| Plan file not found | Error with helpful message |
| Empty reviewer output | Display warning; reviewer may have failed silently |

**CRITICAL: Never silently fall back to Claude self-review.** If the selected
external reviewer fails, report the failure and let the user decide next steps.

## Prerequisites Per Reviewer

### Codex
- `codex` CLI installed (`npm install -g @openai/codex` or via bun)
- OpenAI API authentication (`codex login`)

### Gemini
- `gemini` CLI installed (`npm install -g @google/gemini-cli@latest`)
- Google AI API key or Google OAuth authentication

### Claude CLI (self-review)
- `claude` CLI installed

## Example Usage

```bash
# Invoke peer review (will detect available CLIs and ask)
/peer-review docs/specs/PDS_INGESTION_SPEC.md

# With custom output directory
/peer-review specs/NEW_FEATURE.md --output-dir=reviews/

# With 3 review rounds
/peer-review specs/COMPLEX_FEATURE.md --rounds=3
```
