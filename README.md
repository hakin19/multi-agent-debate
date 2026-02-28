# Multi-Agent Debate

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash command that orchestrates structured debates between two AI agents — **Claude** (via Task sub-agent) and **Codex** (via CLI) — to collaboratively diagnose and solve software engineering problems.

## How It Works

1. **Independent Diagnosis** — Both agents independently investigate the problem, identify root causes, and propose fixes.
2. **Cross-Review** — Each agent critically reviews the other's work, identifying agreements, disagreements, and missed issues.
3. **Iterative Revision** — Agents revise their proposals based on critique, defending or updating their positions with evidence.
4. **Consensus or Escalation** — The loop continues until both agents agree (consensus) or max rounds are reached (you decide).
5. **Implementation** — The agreed-upon fix is implemented automatically.

All agent outputs are saved to a `.debate/` workspace for full auditability.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and configured
- [Codex CLI](https://github.com/openai/codex) installed and configured

## Installation

Copy `debate.md` into your project's Claude Code commands directory:

```bash
mkdir -p .claude/commands
cp debate.md .claude/commands/debate.md
```

## Usage

From within Claude Code, run:

```
/debate <problem description>
```

### Options

- `--rounds N` or `-r N` — Set the maximum number of debate rounds (default: 5)

### Examples

```
/debate The config cache proxy returns stale data after a VyOS commit

/debate --rounds 3 Users report 502 errors when accessing the routing API

/debate https://github.com/your-org/your-repo/issues/42
```

GitHub issue URLs are automatically fetched and expanded.

## Output

Each debate session creates a workspace at `.debate/<session-id>/` containing:

```
.debate/<session-id>/
├── problem.md           # Problem statement
├── proposals/           # Each agent's diagnosis and fix per round
├── reviews/             # Cross-reviews between agents
├── raw/                 # Raw Codex JSONL output
├── arbiter/             # Disagreement evaluations and final verdict
└── implementation-log.md  # What was changed (if consensus reached)
```

## How the Debate Protocol Works

| Phase | What Happens |
|-------|-------------|
| **Diagnose** | Both agents independently read code and propose a root cause + fix |
| **Review** | Each agent critiques the other's proposal |
| **Check** | Facilitator checks if disagreements remain |
| **Revise** | Agents update proposals based on critique (or defend their position) |
| **Loop** | Repeat review → check → revise until consensus or max rounds |
| **Implement** | Consensus fix is applied; unresolved debates escalate to you |

The orchestrator (Claude Code) acts purely as a facilitator — it passes outputs between agents and checks for consensus without injecting its own technical opinions.

## License

MIT
