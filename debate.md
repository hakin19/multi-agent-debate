---
description: Multi-agent debate between Claude and Codex to collaboratively diagnose and solve problems
---

You are orchestrating a structured debate between two AI agents — **Claude** (via Task sub-agent) and **Codex** (via CLI). They independently diagnose a problem, critique each other's work, and iterate toward consensus.

## Argument Parsing

Parse `$ARGUMENTS`:
- If arguments contain `--rounds N` or `-r N`, extract N as **MAX_ROUNDS** (remove it from arguments)
- Default MAX_ROUNDS: **5**
- The remaining text is the **PROBLEM_STATEMENT**
- If the problem statement is a GitHub issue URL, fetch details with `gh issue view <url>` and use the full issue body

## Setup

1. Generate session ID: `debate-$(date +%Y%m%d-%H%M%S)`
2. Create the workspace:
   ```
   .debate/{session-id}/
   ├── problem.md
   ├── proposals/      # Agent diagnoses and fixes per round
   ├── reviews/        # Agent critiques of each other
   ├── raw/            # Codex JSONL output (for session ID extraction)
   └── arbiter/        # Your evaluations and final verdict
   ```
3. Write the PROBLEM_STATEMENT to `.debate/{session-id}/problem.md`
4. Track state internally:
   - `CLAUDE_AGENT_ID` — set after first Task call
   - `CODEX_SESSION_ID` — parsed from first Codex JSONL output
   - `ROUND` — current round number

## Agent Session Management

### Codex

**First call** — use `--json` to capture session ID, `-o` for clean response, and `-C` to pin the working directory:
```bash
codex exec --json --full-auto \
  -C "$(pwd)" \
  -o .debate/{session-id}/proposals/codex-r1.md \
  "PROMPT_TEXT" \
  > .debate/{session-id}/raw/codex-r1.jsonl 2>&1
```
After execution, extract the Codex session ID from the JSONL file — look for a JSON field containing `session_id`, `conversation_id`, or `id` with a UUID value. Store as `CODEX_SESSION_ID`.

**Resume calls** — `codex exec resume` does NOT support the `-o` flag. Use `| tee` to capture output, then clean it up:
```bash
codex exec resume {CODEX_SESSION_ID} --full-auto \
  "PROMPT_TEXT" 2>&1 | tee .debate/{session-id}/raw/codex-r{N}-raw.txt
```
After execution, the raw output will contain CLI noise (headers, thinking lines, etc.) mixed with the agent's actual response. Read the raw file, extract just the agent's analysis content (look for the structured markdown headers like `## Root Cause Analysis`, `## Agreement Points`, etc.), and write the clean version to the appropriate debate file (e.g., `proposals/codex-r{N}.md` or `reviews/codex-on-claude-r{N}.md`) using the Write tool.

**Fallback** — if session ID extraction fails or resume errors, include full context (problem + prior proposals + reviews) in a fresh `codex exec` prompt. Note the fallback in the arbiter log.

Use **600000ms timeout** (10 minutes) for all Codex Bash calls. If Codex times out, note the failure and continue with Claude's output only for that round.

### Claude

**First call** — use Task tool with `subagent_type="general-purpose"`. Store returned agent ID as `CLAUDE_AGENT_ID`. Write the response to the appropriate debate file using the Write tool.

**Resume calls** — use Task tool with `resume=CLAUDE_AGENT_ID`. The agent retains full conversation context automatically. Write each response to the debate file.

## Prompt Templates

### DIAGNOSE (Round 1 — Both Agents)

Replace `{PROBLEM}` with contents of problem.md:

```
You are a senior software engineer performing an independent technical diagnosis.

PROBLEM:
{PROBLEM}

INSTRUCTIONS:
1. Investigate the codebase — read relevant source files to understand the code involved
2. Identify the root cause with specific file paths and line numbers
3. Propose a concrete fix with specific code changes (show before/after or describe precisely)
4. Identify edge cases or risks with your proposed fix
5. Do NOT make any code changes — analysis and proposal only

OUTPUT FORMAT (use these exact headers):

## Root Cause Analysis
[Your analysis with file:line references]

## Proposed Fix
[Specific code changes — what to change, where, and why]
```

### REVIEW (Cross-Review — Each Agent Reviews the Other)

Replace `{PROBLEM}` and `{OTHER_PROPOSAL}`:

```
Another engineer has analyzed the same problem and produced their diagnosis and proposed fix. Your job is to critically evaluate their work.

ORIGINAL PROBLEM:
{PROBLEM}

THEIR DIAGNOSIS AND PROPOSED FIX:
{OTHER_PROPOSAL}

INSTRUCTIONS:
1. Evaluate their root cause analysis — is it correct and complete?
2. Review their proposed fix — will it actually work? Any bugs or oversights?
3. Check for missed edge cases or unintended side effects
4. Identify what they got right that strengthens confidence
5. Do NOT simply agree to be polite — your job is to find problems. If you genuinely agree with everything, explain specifically WHY their analysis is correct with evidence from the codebase.

OUTPUT FORMAT (use these exact headers):

## Agreement Points
[What they got right and why you're confident in those parts]

## Disagreements
[Where you disagree, with specific evidence from the codebase]

## Missed Issues
[Anything they overlooked]

## Overall Assessment
[Do you think their fix is correct? Would you merge it as-is?]
```

### REVISE (Round 2+ — Agent Updates Their Proposal After Critique)

Replace `{REVIEW_OF_YOUR_WORK}`:

```
The other engineer has reviewed your diagnosis and proposed fix. Here is their critique:

THEIR REVIEW OF YOUR WORK:
{REVIEW_OF_YOUR_WORK}

INSTRUCTIONS:
1. Consider each criticism on its merits
2. If they found genuine issues, update your proposal with specific corrections
3. If you disagree with a criticism, defend your position with evidence from the codebase
4. Do NOT capitulate just to reach agreement — defend your analysis if you believe it's correct

OUTPUT FORMAT (use these exact headers):

## Response to Critique
[Address each point they raised — agree or rebut with evidence]

## Updated Proposal
[Your revised fix, or reaffirmation of original with justification]

## Remaining Disagreements
[Points you still disagree on, if any — be specific about what and why]
```

## Execution Protocol

### Phase 1: Independent Diagnosis

Tell the user: **"Round 1 — Both agents are independently diagnosing the problem..."**

Run BOTH in parallel:
1. **Claude** — Task tool, DIAGNOSE prompt → save to `proposals/claude-r1.md`, store CLAUDE_AGENT_ID
2. **Codex** — Bash tool, codex exec with DIAGNOSE prompt → auto-saved to `proposals/codex-r1.md` via `-o`, parse CODEX_SESSION_ID from JSONL

After both complete, briefly summarize each agent's position to the user (2-3 sentences each).

### Phase 2: Iterative Cross-Review Loop

For ROUND = 1 to MAX_ROUNDS:

**Step A — Cross-Review** (run in parallel):
1. **Claude reviews Codex**: Resume Claude (Task resume), REVIEW prompt with Codex's latest proposal
   → Write to `reviews/claude-on-codex-r{ROUND}.md`
2. **Codex reviews Claude**: Resume Codex (codex exec resume), REVIEW prompt with Claude's latest proposal
   → Save to `reviews/codex-on-claude-r{ROUND}.md`

Tell the user: **"Round {ROUND} — Cross-reviews complete. Evaluating consensus..."**

**Step B — Disagreement Check**:

Your ONLY job here is to determine whether any disagreements remain. Do NOT judge the technical correctness of either agent's position.

Read both reviews. Check the **Disagreements** sections:
- If BOTH reviews have empty or "None" disagreements → **NO DISAGREEMENTS REMAIN**
- If EITHER review lists any disagreements → **DISAGREEMENTS REMAIN**

Write to `arbiter/eval-r{ROUND}.md`:
```
## Round {ROUND} Disagreement Check
- Claude's disagreements with Codex: [List, or "None"]
- Codex's disagreements with Claude: [List, or "None"]
- Decision: [CONSENSUS / CONTINUE / MAX_ROUNDS_REACHED]
```

Tell the user the decision and list any remaining disagreements.

**If CONSENSUS (no disagreements remain)** → go to Implementation
**If ROUND >= MAX_ROUNDS** → go to Escalation (unresolved disagreements)

**Step C — Revision** (run in parallel):
1. **Claude revises**: Resume Claude, REVISE prompt with Codex's review of Claude's work
   → Write to `proposals/claude-r{ROUND+1}.md`
2. **Codex revises**: Resume Codex, REVISE prompt with Claude's review of Codex's work
   → Save to `proposals/codex-r{ROUND+1}.md`

Tell the user: **"Round {ROUND+1} — Revised proposals ready. Starting cross-review..."**

→ Loop back to Step A with ROUND+1

### Phase 3a: Implementation (Consensus Reached)

When both agents agree (no remaining disagreements):

1. Write `.debate/{session-id}/arbiter/verdict.md`:
```markdown
# Debate Verdict

**Session:** {session-id}
**Rounds completed:** {N}
**Outcome:** Consensus

## Problem
[One-line summary]

## Agreed Fix
[The consensus fix — summarize what both agents agreed on]
```

2. Tell the user: **"Consensus reached after {N} rounds. Both agents agree on the fix. Handing off to Codex for implementation..."**

3. Resume the Codex session and instruct it to implement the consensus fix:
```bash
codex exec resume {CODEX_SESSION_ID} --full-auto \
  "The debate has concluded. Both you and the other engineer agreed on the fix. Implement the agreed changes now. Here is the consensus: {AGREED_FIX_SUMMARY}" 2>&1 | tee .debate/{session-id}/raw/implementation-raw.txt
```

4. After Codex completes, read the raw output, extract a clean summary of what was implemented, and write it to `.debate/{session-id}/implementation-log.md`. Inform the user what was changed.

### Phase 3b: Escalation (Max Rounds, No Consensus)

When MAX_ROUNDS is reached with disagreements still remaining:

1. Write `.debate/{session-id}/arbiter/verdict.md`:
```markdown
# Debate Verdict

**Session:** {session-id}
**Rounds completed:** {N}
**Outcome:** Unresolved — max rounds reached

## Problem
[One-line summary]

## Claude's Final Position
[2-3 sentence summary]

## Codex's Final Position
[2-3 sentence summary]

## Remaining Disagreements
[Specific points they still disagree on]
```

2. Present both positions and the remaining disagreements to the user.

3. Ask the user: **"The agents could not reach consensus after {N} rounds. Which approach would you like to go with — Claude's or Codex's? Or would you like to provide your own direction?"**

4. Once the user decides, resume the appropriate agent to implement their chosen approach.

## Rules

1. **You are a facilitator, not a judge** — Do NOT assess the technical correctness of either agent's position. Your only job is to pass outputs between agents and determine when disagreements have been resolved. Pass the other agent's output verbatim — never inject your own opinions or analysis.
2. **No code changes during debate** — Agents investigate and propose only. Implementation happens AFTER consensus is reached (by Codex) or after the user chooses an approach (on escalation).
3. **Audit trail** — Every agent output MUST be written to the debate directory before proceeding to the next step. Never skip writing a file.
4. **Parallel execution** — Run both agents simultaneously whenever they're working independently (diagnosis, reviews, revisions).
5. **Session persistence** — Always use resume/session-id for rounds after the first. This avoids agents re-reading the same files and keeps context efficient.
6. **Transparency** — Briefly update the user after each phase with a summary of where things stand and what disagreements remain (if any).
7. **Graceful degradation** — If one agent fails (timeout, error, session resume failure), note the failure, continue with the working agent's output, and inform the user.
