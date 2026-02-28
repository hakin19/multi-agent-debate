# Multi-Agent Debate

A structured debate protocol for AI coding agents. Two agents independently diagnose a problem, critique each other's work, and iterate toward consensus — producing better solutions than either agent alone.

The included implementation uses [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and [Codex CLI](https://github.com/openai/codex) as the two agents, but the protocol is framework-agnostic and can be adapted to any combination of AI coding assistants.

## Why Debate?

A single AI agent can miss things. Two agents reviewing each other's work catch more bugs, surface edge cases, and build confidence in the final fix — the same way human code review works. The structured protocol ensures agents engage critically rather than rubber-stamping each other's proposals.

## How It Works

1. **Independent Diagnosis** — Both agents independently investigate the problem, identify root causes, and propose fixes.
2. **Cross-Review** — Each agent critically reviews the other's work, identifying agreements, disagreements, and missed issues.
3. **Iterative Revision** — Agents revise their proposals based on critique, defending or updating their positions with evidence.
4. **Consensus or Escalation** — The loop continues until both agents agree (consensus) or max rounds are reached (you decide).
5. **Implementation** — The agreed-upon fix is implemented automatically.

All agent outputs are saved to a `.debate/` workspace for full auditability.

## The Protocol

The debate follows a structured loop with clear prompt templates for each phase:

| Phase | What Happens |
|-------|-------------|
| **Diagnose** | Both agents independently read code and propose a root cause + fix |
| **Review** | Each agent critiques the other's proposal |
| **Check** | Facilitator checks if disagreements remain |
| **Revise** | Agents update proposals based on critique (or defend their position) |
| **Loop** | Repeat review → check → revise until consensus or max rounds |
| **Implement** | Consensus fix is applied; unresolved debates escalate to you |

The orchestrator acts purely as a facilitator — it passes outputs between agents and checks for consensus without injecting its own technical opinions.

See [`debate.md`](debate.md) for the complete protocol specification, including prompt templates, session management, and execution rules.

## Reference Implementation: Claude Code + Codex

The included `debate.md` is a ready-to-use implementation for Claude Code with Codex CLI as the second agent.

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and configured
- [Codex CLI](https://github.com/openai/codex) installed and configured

### Installation

Copy `debate.md` into your project's Claude Code commands directory:

```bash
mkdir -p .claude/commands
cp debate.md .claude/commands/debate.md
```

### Usage

From within Claude Code, run:

```
/debate <problem description>
```

**Options:**

- `--rounds N` or `-r N` — Set the maximum number of debate rounds (default: 5)

**Examples:**

```
/debate The config cache proxy returns stale data after a commit

/debate --rounds 3 Users report 502 errors when accessing the API

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
├── raw/                 # Raw agent output for session management
├── arbiter/             # Disagreement evaluations and final verdict
└── implementation-log.md  # What was changed (if consensus reached)
```

## Adapting to Other Frameworks

The protocol in `debate.md` is a structured prompt — the core ideas work with any two AI coding agents that can:

1. **Read code** and produce structured analysis
2. **Accept prior context** (another agent's output) as input
3. **Be resumed** or given conversation history across rounds

To adapt it, replace the Claude Code Task tool calls and Codex CLI commands with equivalent invocations for your preferred agents. The prompt templates (Diagnose, Review, Revise) and the facilitator logic (disagreement checking, consensus detection) are agent-agnostic.

## License

MIT
