# OpenCode Council — Multi-Perspective Decision Support

**Version:** 0.5  
**Status:** Production-Ready

---

## What Is This?

OpenCode Council orchestrates structured debates between AI agents representing different perspectives (security, velocity, maintainability, etc.) to help you make better technical decisions.

Instead of a single AI's opinion, you get:
- **Multiple expert viewpoints** arguing from their specialized perspectives
- **Structured rounds** where perspectives respond to each other's arguments
- **Synthesis documents** that surface consensus, tensions, risks, and actionable recommendations

Think of it as having a panel of domain experts debate the tradeoffs before you make a decision.

---

## When Should I Use This?

Council is designed for **consequential technical decisions** where:
- Multiple valid perspectives exist (security vs velocity, pragmatism vs purity)
- Tradeoffs need to be surfaced and evaluated
- You want to avoid blind spots from a single viewpoint
- The decision warrants 5-10 minutes of structured debate

**Good use cases:**
- Architecture decisions (microservices vs monolith, GraphQL vs REST)
- Technology adoption (new language, framework, tool)
- Security vs velocity tradeoffs (MVP scope, technical debt paydown)
- Process changes (CI/CD strategy, testing approach)

**Not ideal for:**
- Simple yes/no questions answerable by docs
- Decisions already made (seeking validation)
- Bikeshedding (naming, formatting preferences)

---

## How It Works

### 1. Setup (Interactive)
You provide a proposition. The moderator:
- Frames it clearly
- Identifies available advocate perspectives
- Confirms debate parameters (rounds, perspectives)

### 2. Debate Rounds
For each round:
- **All advocates respond simultaneously** (parallel execution for speed)
- Round 1: Each perspective makes its strongest case
- Round 2+: Advocates respond to each other, refine positions, concede valid points

### 3. Gate Checkpoints
After each round, the moderator:
- Summarizes what each perspective argued (3-5 sentences each)
- Asks if you want to continue, redirect, or conclude early
- This is your opportunity to steer the debate

### 4. Synthesis
The summariser produces a structured document:
- **Consensus:** Points of agreement across perspectives
- **Key Tensions:** Fundamental disagreements with strongest arguments
- **Risk Assessment:** Material risks of each decision path
- **Recommended Path Forward:** Best risk/reward option (not hollow compromise)
- **Minority Report:** Important dissenting positions to consider
- **Suggested Actions:** Concrete, assignable next steps

### 5. Archive
The entire debate (all rounds, synthesis) is archived for future reference.

---

## Quick Start

### Generate a project-tailored council

```
/generate-council
```

The generator reads your project's codebase, tech stack, and MCP servers, asks a few questions, then scaffolds advocate agents tailored to your domain. Run this once to set up, then re-run whenever the project evolves.

### Run a council debate

```
/council Should we migrate our authentication to OAuth 2.0?
```

The moderator will guide you through setup, then orchestrate the debate. Typical council duration:
- **2 rounds, 3 advocates:** ~2-3 minutes
- **3 rounds, 5 advocates:** ~4-5 minutes

---

## Architecture Overview

### Components

| Component | Role | Mode |
|-----------|------|------|
| **council-generator** | Analyzes project, scaffolds tailored advocate agents | Primary agent |
| **council-moderator** | Orchestrates entire workflow, neutral facilitator | Primary agent |
| **council-summariser** | Produces synthesis from debate rounds | Hidden subagent |
| **advocate-\*** | Perspective agents (generated per-project) | Subagent |

The default advocates (security, velocity, maintainability) ship as examples. Use `/generate-council` to create advocates tailored to your project's domain, tech stack, and data sources.

### Debate Structure

```
User → /council <topic>
  ↓
Moderator: Setup (proposition framing, perspective selection)
  ↓
Round 1: All advocates argue simultaneously (parallel)
  ↓
Gate 1: User reviews round 1, decides to continue
  ↓
Round 2: All advocates respond to each other (parallel)
  ↓
Gate 2: User reviews round 2, decides to synthesize
  ↓
Summariser: Produces 6-section synthesis document
  ↓
Archive: Complete debate preserved for reference
```

### Performance

**Parallel execution within rounds:**
- 3 advocates complete a round in ~1 minute (vs ~3 minutes sequential)
- Rounds remain sequential with user gates for control
- Total debate time scales with rounds, not advocates

---

## Key Design Principles

### Human-in-the-Loop
You control the workflow through gate confirmations. The moderator never proceeds to the next round without your approval. This lets you:
- Redirect a perspective that's off-track
- Conclude early if consensus emerges
- Add a follow-up question for the next round

### Plain Text Persistence
All council state lives in markdown and YAML files under `docs/council/`. This means:
- Git-trackable debate history
- Human-readable at every stage
- No opaque databases
- Easy to inspect, archive, or share

### Agent-Agnostic Perspectives
The moderator doesn't care what perspectives you use. You bring your own advocate agents. This makes the system:
- Adaptable to any domain (not just engineering)
- Extensible by users (add `advocate-cost.md`, done)
- Composable (different perspective sets for different decisions)

### Parallel Execution for Speed
All advocates within a round execute simultaneously. This maximizes throughput while maintaining:
- Sequential rounds for human control
- Consistent input (all advocates see same prior rounds)
- Clean output (no race conditions on file writes)

---

## Documentation

### Specifications

- **[Workflow](specs/workflow.md)** — Detailed breakdown of the 5-phase council lifecycle
- **[Parallel Execution](specs/parallel-execution.md)** — How advocates run simultaneously within rounds
- **[Synthesis Format](specs/synthesis.md)** — Structure and guidelines for synthesis documents
- **[File Formats](specs/file-formats.md)** — Council manifest, round files, and frontmatter schemas
- **[Extending](specs/extending.md)** — How to add custom advocate perspectives

### For Developers

- **[AGENTS.md](../AGENTS.md)** — AI agent guidelines for working with this codebase
- **[Architecture Decision Log](specs/decisions.md)** — Why we made key design choices

---

## Example Council Topics

### Architecture Decisions
- "Should we adopt microservices for our user service?"
- "GraphQL vs REST for our new mobile API?"
- "Event-driven vs request-response for inter-service communication?"

### Technology Adoption
- "Should we migrate from JavaScript to TypeScript?"
- "Should we use Terraform or Pulumi for infrastructure-as-code?"
- "Should we adopt React Server Components?"

### Security vs Velocity
- "Should we mandate SBOM generation for all internal services?"
- "Should we require MFA for all developer accounts?"
- "How much technical debt should we accept in this MVP?"

### Process Changes
- "Should we require code review approval before merging to main?"
- "Should we adopt trunk-based development?"
- "Should we implement automated rollback for failed deployments?"

---

## Current Limitations

**Open visibility only:**  
Advocates always see prior round contributions from other perspectives. "Blind mode" (independent arguments without cross-pollination) is not yet supported.

**No resume support:**  
If a council is interrupted, you must start a new one. The manifest and completed round files remain intact for reference.

**Fixed round structure:**  
All rounds use the same format. Dynamic round structures (e.g., "rebuttal round", "convergence round") are not yet supported.

**MCP integration is manual:**
The generator can wire MCP servers to relevant advocates when detected, but you must configure MCP servers in `.opencode/config.yaml` first.

---

## What's Next?

### Planned Features (v0.6+)

- **Blind visibility mode:** Advocates argue independently in round 1, see others in round 2+
- **Dynamic round structures:** Specialized round types for convergence, escalation, etc.
- **Resume support:** Continue interrupted councils from their last completed round

### Contribute

Custom perspectives? Workflow improvements? File an issue or submit a PR.

---

## License & Credits

Built on [OpenCode](https://opencode.sh) agent framework.

Inspired by the realization that the best technical decisions emerge from structured debate between well-informed perspectives, not from single-viewpoint advice.
