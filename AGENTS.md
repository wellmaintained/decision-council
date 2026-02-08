# AGENTS.md — AI Agent Guidelines for decision-council

**Last Updated:** 2026-02-08  
**Project:** OpenCode Council — Multi-Agent Debate Orchestration System

---

## Project Overview

This is a **configuration-driven multi-agent debate system** built on OpenCode's agent framework. No traditional code, build system, or tests—just markdown agents with YAML frontmatter that orchestrate structured decision-making debates.

**Core Concept:** Multiple AI agents with different perspectives (security, velocity, maintainability) debate propositions through structured rounds, producing synthesized recommendations for human decision-makers.

---

## Repository Structure

```
decision-council/
├── .opencode/                          # OpenCode framework directory
│   ├── agents/                         # Agent definitions (5 files)
│   │   ├── council-moderator.md        # Primary orchestrator
│   │   ├── council-summariser.md       # Synthesis generator (hidden)
│   │   ├── advocate-security.md        # Security perspective
│   │   ├── advocate-velocity.md        # Speed perspective
│   │   └── advocate-maintainability.md # Quality perspective
│   ├── commands/                       # Custom commands
│   │   └── council.md                  # /council <topic> command
│   ├── council/                        # Council debate artifacts
│   │   ├── active/                     # In-progress councils
│   │   └── archive/                    # Completed councils
│   ├── package.json                    # Dependencies (@opencode-ai/plugin)
│   └── .gitignore                      # Ignore node_modules
├── docs/                               # Documentation
│   ├── poc-prd.md                      # Complete PRD (544 lines)
│   └── IMPLEMENTATION-COMPLETE.md      # Verification checklist
├── .gitignore                          # Root ignore (Go-focused)
└── README.md                           # Project overview
```

---

## Technology Stack

- **Framework:** OpenCode agent platform
- **Language:** Markdown + YAML (no compiled code)
- **Package Manager:** Bun (bun.lock present)
- **Runtime:** Node.js (implied by package.json)
- **Dependencies:** `@opencode-ai/plugin` v1.1.53

**No build system, no tests, no linters.** This is a POC demonstrating agent orchestration via configuration files.

---

## Build & Test Commands

### There Are None (Intentionally)

This project has **no traditional build, lint, or test commands**. Verification is interactive:

```bash
# Test the council workflow
/council Should we use TypeScript or JavaScript for the new microservice?
```

**Verification checklist:** See `docs/IMPLEMENTATION-COMPLETE.md` for expected file structures and manual verification steps.

**Success criteria:** Defined in `docs/poc-prd.md` section 9.

---

## File Structure Standards

### Agent Configuration Files (`.opencode/agents/*.md`)

**Required structure:**
```markdown
---
description: Brief description of agent purpose (string or multiline with >)
mode: primary | subagent
temperature: 0.1-0.9
tools:
  read: true/false
  write: true/false
  edit: true/false
  bash: true/false
  glob: true/false
  grep: true/false
  task: true/false
  webfetch: true/false
permission:
  bash:
    "*": deny
    "cat *": allow
    "ls *": allow
  task:
    "*": deny
    "advocate-*": allow
hidden: true | false (optional, default: false)
---

# Agent Name

[Agent instructions and prompt]
```

**Temperature Guidelines:**
- **0.1-0.2:** Deterministic tasks (orchestration, structured extraction)
- **0.3-0.4:** Measured reasoning (security, maintainability)
- **0.5-0.6:** Creative reasoning (velocity, pragmatic tradeoffs)

**Permission Syntax:**
- Pattern matching with `"*"` wildcard
- **Last match wins** (order matters!)
- Use `deny` default + `allow` specific commands for safety

### Council Files (`.opencode/council/active/<id>/`)

**Manifest (council.yaml):**
```yaml
id: council-YYYY-MM-DD-<slug>
created: <ISO8601 timestamp>
topic_summary: "Question or proposition"
status: setup | round-1 | gate-1 | round-2 | gate-2 | synthesis | complete | archived
total_rounds: 2
current_round: 1
visibility: open
perspectives:
  - id: <perspective-id>
    agent: advocate-<perspective-id>
    rounds:
      1: { status: pending | in-progress | complete, file: round-1-<id>.md }
      2: { status: pending | in-progress | complete, file: round-2-<id>.md }
```

**Round Files (round-N-<perspective>.md):**
```markdown
---
agent: advocate-<perspective>
perspective: <perspective-id>
round: 1
timestamp: <ISO8601>
council_id: council-YYYY-MM-DD-<slug>
responding_to: []  # Empty for round 1, list of prior round files for round 2+
word_count: <number>
---

# [Perspective Name] — Round [N]

[Agent's argument content]
```

**Synthesis File (synthesis.md):**
```markdown
---
council_id: council-YYYY-MM-DD-<slug>
generated: <ISO8601>
rounds_completed: 2
perspectives: [security, velocity, maintainability]
---

# Council Synthesis: [Topic]

## Consensus
[Points of agreement across perspectives]

## Key Tensions
[Fundamental disagreements with strongest arguments from each side]

## Risk Assessment
[Material risks of each decision path, mapped to perspectives]

## Recommended Path Forward
[Concrete recommendation with best risk/reward profile]

## Minority Report
[Important dissenting positions not adopted in recommendation]

## Suggested Actions
[Numbered, concrete, assignable next steps]
```

---

## Naming Conventions

**Files:**
- Agent files: `<name>.md` (lowercase, hyphen-separated)
- Council ID: `council-YYYY-MM-DD-<slug>` (date + slugified topic)
- Round files: `round-<N>-<perspective-id>.md` (number + perspective)

**Agents:**
- Primary agents: `council-<name>` (e.g., `council-moderator`)
- Subagents: `<category>-<name>` (e.g., `advocate-security`)
- Hidden agents: Add `hidden: true` to YAML frontmatter

**Perspectives:**
- Use full words: `maintainability` (not `maintain`)
- Lowercase with hyphens: `security`, `velocity`, `maintainability`

---

## Code Style Guidelines

### YAML Frontmatter

**DO:**
- Use consistent indentation (2 spaces)
- Quote strings with special characters
- Use multiline strings with `>` for long descriptions
- Always include required fields: `description`, `mode`, `temperature`

**DON'T:**
- Use tabs (spaces only)
- Omit required fields
- Use invalid permission syntax (see pattern-matching format above)

### Markdown Content

**DO:**
- Use clear section headers (`##` for main sections)
- Keep response guidelines to 500-800 words for advocates
- Structure arguments with bullet points for clarity
- Include specific examples or standards when applicable

**DON'T:**
- Make agents balanced—each advocate argues ONE perspective forcefully
- Allow moderator to advocate for any position (must remain neutral)
- Soften disagreements in synthesis (if irreconcilable, say so)

---

## Agent Role Specifications

### council-moderator (Primary Agent)
- **Mode:** primary
- **Temperature:** 0.2 (deterministic)
- **Tools:** Full (write, edit, bash, task) with restricted permissions
- **Role:** Neutral orchestrator—frames propositions, manages rounds, gates transitions, triggers synthesis, archives councils
- **Critical Rules:**
  - Never advocate for any position
  - Always gate round transitions through user confirmation
  - Ask at most one question per message during setup
  - Propose sensible defaults, ask for confirmation (not open-ended questions)
  - Write all files to `.opencode/council/active/<council-id>/`
  - Update `council.yaml` at every workflow transition
  - Present brief round summaries (3-5 sentences per perspective) at each gate
  - Invoke advocates sequentially, one Task call per advocate per round

### council-summariser (Hidden Subagent)
- **Mode:** subagent
- **Temperature:** 0.1 (structured extraction)
- **Tools:** Read-only (read, glob, grep)
- **Hidden:** true (not in @ autocomplete)
- **Role:** Produces 6-section synthesis from all round files
- **Rules:**
  - Be precise—attribute positions to specific perspectives
  - Do not invent arguments that were not made
  - Do not soften disagreements (if irreconcilable, say so)
  - Recommendation should be best risk/reward, not hollow compromise
  - Each suggested action must be specific, assignable, verifiable
  - Keep total synthesis under 1500 words

### advocate-* (Perspective Subagents)
- **Mode:** subagent
- **Temperature:** 0.3-0.6 (varies by perspective)
- **Tools:** Read-only + limited bash (`cat`, `ls`, `grep`, `find`)
- **Role:** Argue forcefully from ONE perspective (not balanced)
- **Guidelines:**
  - Present strongest arguments from your perspective (500-800 words)
  - Be specific and concrete—reference standards, patterns, evidence
  - Structure with clear sections
  - In round 2+, respond to other advocates' arguments
  - Acknowledge valid points, but always return to your core perspective
  - Identify where you agree, disagree, and where you see false dichotomies

**Perspective-Specific Temperatures:**
- **Security** (0.3): Conservative, risk-focused, standards-based
- **Velocity** (0.5): Pragmatic, speed-focused, opportunity cost framing
- **Maintainability** (0.4): Balanced, quality-focused, long-term thinking

---

## Workflow Phases (5 Phases)

1. **Setup** — Interactive proposition framing + advocate selection + parameter confirmation
2. **Rounds** — Sequential advocate invocation via Task tool, write responses to round files
3. **Gates** — User checkpoints: present round summaries, ask to proceed
4. **Synthesis** — Invoke summariser with all round files, write synthesis.md
5. **Archive** — Move council from `active/` to `archive/`, confirm to user

**Default:** 2 rounds, open visibility (advocates see prior rounds)

---

## Error Handling

**If subagent fails:**
- Moderator informs user and offers to retry or skip that perspective

**If manifest corrupted:**
- Moderator reports error and asks user to inspect `council.yaml`

**If fewer than 2 advocates found:**
- Moderator halts setup and informs user

**If interrupted:**
- POC does not support resume—user must start new council (manifest/files preserved for reference)

---

## Adding New Advocate Agents

1. Create `.opencode/agents/advocate-<name>.md`
2. Use existing advocate files as templates
3. Set appropriate temperature (0.3-0.6 based on perspective personality)
4. Define perspective clearly in prompt
5. Instruct to argue from ONE viewpoint (not balanced)
6. Include round 2+ response guidance
7. Keep responses to 500-800 words
8. Test with `/council <topic>` to verify moderator discovers the new agent

**Example perspectives:** cost-optimization, user-experience, scalability, compliance, innovation, technical-simplicity

---

## Git Conventions

**Commit Message Format:**
```
feat(council): <description>
fix(council): <description>
docs: <description>
```

**Conventional Commit Types:**
- `feat`: New agent, command, or feature
- `fix`: Bug fix in configuration
- `docs`: Documentation updates
- `chore`: Dependency updates, config changes

**What to commit:**
- Agent configuration files
- Command definitions
- Documentation updates
- Council manifests/round files (if preserving debates)

**What NOT to commit:**
- `node_modules/` (already ignored)
- Active council directories (use `.opencode/.gitignore`)

---

## Key Design Principles

1. **Human-in-the-loop:** User controls workflow via gate confirmations
2. **Plain text persistence:** All state in markdown/YAML (git-trackable, inspectable)
3. **Agent-agnostic perspectives:** Users bring their own advocate prompts
4. **Minimal viable orchestration:** POC demonstrates simplest workflow with value
5. **Sequential execution only:** No parallel subagent calls in POC

---

## Scope (POC vs Future)

**In Scope (POC):**
- ✅ Moderator + summariser + 3 default advocates
- ✅ Custom perspectives (user can add advocates)
- ✅ Sequential subagent invocation
- ✅ Two-round debate with user gating
- ✅ Structured 6-section synthesis
- ✅ `/council` command
- ✅ Archive completed councils

**Out of Scope (Future):**
- ❌ Preset council types (architecture, adopt, incident, hire)
- ❌ Parallel subagent execution
- ❌ Blind visibility mode (advocates don't see prior rounds)
- ❌ Resume support for interrupted councils
- ❌ MCP integration for external data sources
- ❌ Progress tracking / real-time status

---

## References

- **Complete PRD:** `docs/poc-prd.md` (544 lines, authoritative specification)
- **Implementation Status:** `docs/IMPLEMENTATION-COMPLETE.md` (verification checklist)
- **Agent Examples:** `.opencode/agents/*.md` (5 working agent configurations)
- **Active Council Example:** `.opencode/council/active/council-2026-02-08-*/` (real debate in progress)

---

## Quick Start for AI Agents

**To understand the system:**
1. Read `docs/poc-prd.md` (complete specification)
2. Examine `.opencode/agents/council-moderator.md` (primary orchestrator)
3. Look at one advocate file (e.g., `advocate-security.md`) for perspective pattern

**To test the system:**
```bash
/council Should we migrate to microservices?
```

**To add a new perspective:**
1. Copy `.opencode/agents/advocate-security.md`
2. Rename to `advocate-<your-perspective>.md`
3. Modify description, temperature, and prompt for your perspective
4. Test by running `/council <topic>` and verifying moderator finds the new agent

**To modify workflow:**
- Edit `.opencode/agents/council-moderator.md` (workflow phases defined in prompt)
- DO NOT change file formats without updating all agents that read/write those files

---

**Questions? Issues?**
- Check `docs/poc-prd.md` for specifications
- Check `docs/IMPLEMENTATION-COMPLETE.md` for verification steps
- Verify agent YAML frontmatter syntax (especially `permission` fields)
- Ensure `council.yaml` manifest is valid YAML
