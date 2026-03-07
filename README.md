# Decision Council

**Multi-perspective AI debate system for better technical decisions**

Version 0.5 | [Full Documentation](docs/README.md)

---

## What Is This?

Council orchestrates structured debates between AI agents representing different perspectives (security, velocity, maintainability, etc.) to help you make consequential technical decisions.

Instead of a single AI's opinion, you get:
- Multiple expert viewpoints arguing from specialised perspectives
- Structured rounds where perspectives respond to each other
- Synthesis documents surfacing consensus, tensions, and actionable recommendations

---

## Quick Start

```bash
/council Should we migrate to microservices?
```

The moderator will:
1. Frame the proposition clearly
2. Identify available perspectives (security, velocity, maintainability, etc.)
3. Orchestrate 2 rounds of debate (all perspectives per round execute in parallel)
4. Present synthesis with recommendations

**Typical duration:** 2–3 minutes for 2 rounds, 3 perspectives

---

## Example Output

After debating "Should we adopt GraphQL for our API?", you'll get a synthesis covering:

- **Consensus:** What all perspectives agree on
- **Key Tensions:** Fundamental trade-offs (e.g., velocity wants speed, security wants validation)
- **Risk Assessment:** What could go wrong with each path
- **Recommended Path Forward:** Concrete decision with reasoning
- **Minority Report:** Important dissenting views
- **Suggested Actions:** Numbered next steps

---

## When to Use

**Good for:**
- Architecture decisions (microservices, event-driven, monolith)
- Technology adoption (new framework, language, tool)
- Security vs velocity trade-offs (MVP scope, technical debt)
- Process changes (CI/CD strategy, code review practices)

**Not for:**
- Simple yes/no questions answerable by docs
- Bikeshedding (naming, formatting)
- Decisions already made (seeking validation)

---

## How It Works

### 1. Setup (Interactive)
You provide a proposition. Moderator confirms framing and perspectives.

### 2. Debate Rounds
- **Round 1:** All perspectives make their strongest case (parallel execution)
- **Gate:** You review summaries, decide to continue
- **Round 2:** Perspectives respond to each other (parallel execution)
- **Gate:** You review, decide to synthesise

### 3. Synthesis
Summariser produces structured 6-section document from all rounds.

### 4. Archive
Complete debate preserved under `docs/council/archive/` for future reference.

---

## Architecture

Implemented as an **Agent Skill** following the [Agent Skills standard](https://agentskills.io/specification). The skill lives in `skills/council/` and is symlinked from `.claude/skills/council/` for Claude Code discovery.

**Components:**
- `SKILL.md` — Council moderator (neutral orchestrator)
- `perspectives/security.md` — Risk-focused perspective
- `perspectives/velocity.md` — Speed-focused perspective
- `perspectives/maintainability.md` — Quality-focused perspective
- `references/workflow.md` — Workflow protocol
- `references/summariser-prompt.md` — Synthesis instructions
- `references/synthesis-format.md` — Output specification

**Custom perspectives:** Add `skills/council/perspectives/<name>.md`

**Performance:** Parallel execution within rounds reduces time by ~67% (3 perspectives: 3 min → 1 min per round)

---

## Documentation

**[Full Documentation](docs/README.md)**

### Key Specs
- [Workflow](docs/specs/workflow.md) — 5-phase council lifecycle
- [Parallel Execution](docs/specs/parallel-execution.md) — How perspectives run simultaneously
- [Synthesis Format](docs/specs/synthesis.md) — Output structure and guidelines
- [Extending](docs/specs/extending.md) — Add custom perspectives
- [Adapting to Other Agents](docs/specs/adapting-to-other-agents.md) — Cross-platform skill design

### For Developers
- [AGENTS.md](AGENTS.md) — AI agent guidelines for this codebase

---

## Installation

This is an Agent Skill, not a traditional package.

**Requirements:**
- An AI coding agent that supports the Agent Skills standard (Claude Code, OpenCode, Cursor)

**Setup:**
```bash
git clone <this-repo>
cd decision-council
# Claude Code discovers the skill via .claude/skills/council/ symlink
```

---

## Example Topics

**Architecture:**
- "Should we adopt event-driven architecture for order processing?"
- "Migrate from REST to GraphQL for mobile API?"

**Technology:**
- "Should we migrate from JavaScript to TypeScript?"
- "Use Terraform or Pulumi for infrastructure-as-code?"

**Security vs Velocity:**
- "Require MFA for all developer accounts?"
- "How much technical debt should we accept in this MVP?"

**Process:**
- "Adopt trunk-based development vs feature branches?"
- "Require code review approval before merging to main?"

---

## Status

**v0.5** — Production-ready

What works:
- Interactive setup with proposition framing
- Parallel perspective execution per round (60–70% faster)
- Multi-round debates with user gates
- Structured 6-section synthesis
- Custom perspective support
- Archive for debate history

What's planned:
- Preset council types (architecture, adoption, incident, hire)
- Blind visibility mode (perspectives don't see others' round 1)
- Resume support for interrupted councils
- MCP integration for external data sources

---

## Contributing

**Custom perspectives?** See [Extending](docs/specs/extending.md)

**Ideas or issues?** File an issue or submit a PR.

---

## Licence

MIT

---

## Credits

Inspired by the realisation that the best technical decisions emerge from structured debate between well-informed perspectives, not from single-viewpoint advice.
