# AGENTS.md — AI Agent Guidelines for decision-council

**Last Updated:** 2026-03-07
**Project:** Decision Council — Multi-Agent Debate Orchestration System

---

## Project Overview

This is a **configuration-driven multi-agent debate system** packaged as an Agent Skill. No traditional code, build system, or tests — just markdown files with YAML frontmatter that orchestrate structured decision-making debates.

**Core Concept:** Multiple AI agents with different perspectives (security, velocity, maintainability) debate propositions through structured rounds, producing synthesised recommendations for human decision-makers.

---

## Repository Structure

```
decision-council/
├── skills/                            # Canonical skill content
│   └── council/
│       ├── SKILL.md                   # Council moderator (entry point)
│       ├── perspectives/              # Perspective definitions
│       │   ├── security.md
│       │   ├── velocity.md
│       │   └── maintainability.md
│       └── references/                # Shared protocol files
│           ├── workflow.md            # Workflow phases, rules, templates
│           ├── summariser-prompt.md   # Synthesis instructions
│           └── synthesis-format.md    # 6-section output specification
├── .claude/
│   └── skills/
│       └── council -> ../../skills/council   # Symlink for Claude Code
├── docs/                              # Documentation and council artefacts
│   ├── council/                       # Council debate artefacts
│   │   ├── active/                    # In-progress councils
│   │   └── archive/                   # Completed councils
│   └── specs/                         # Design specifications
├── AGENTS.md                          # This file
├── README.md                          # Project overview
└── LICENSE
```

---

## Technology Stack

- **Format:** Agent Skill (markdown + YAML)
- **Standard:** Agent Skills open standard ([agentskills.io](https://agentskills.io/specification))
- **Discovery:** `.claude/skills/council/SKILL.md` (symlinked from `skills/council/`)

**No build system, no tests, no linters.** This is a configuration-driven system — verification is interactive.

---

## Build & Test Commands

### There Are None (Intentionally)

This project has **no traditional build, lint, or test commands**. Verification is interactive:

```bash
/council Should we use TypeScript or JavaScript for the new microservice?
```

---

## Skill Structure

### Entry Point: `skills/council/SKILL.md`

The SKILL.md file defines the council moderator. It contains:
- YAML frontmatter (`name`, `description`, `allowed-tools`)
- The moderator's system prompt

The moderator reads reference files at runtime and discovers perspectives via glob.

### Perspective Files (`skills/council/perspectives/*.md`)

Each perspective is a standalone markdown file. No frontmatter — the moderator reads these at runtime and injects them into subagent task prompts.

**Required structure:**
```markdown
# <Perspective Name> Perspective

## Your Perspective
[What this perspective argues for]

## How You Argue
[Argumentation strategy]

## Key Areas of Focus
[Domain-specific concerns]

## In Round 2 and Beyond
[Multi-round engagement rules]

## Tone
[Communication style]

## Response Format
[Word count, structure guidelines]
```

### Reference Files (`skills/council/references/`)

- **workflow.md** — Workflow phases, rules, file conventions, task prompt templates, manifest structure
- **summariser-prompt.md** — Instructions for the synthesis subagent
- **synthesis-format.md** — 6-section synthesis output specification

### Council Artefacts (`docs/council/active/<id>/`)

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
    rounds:
      1: { status: pending | in-progress | complete, file: round-1-<id>.md }
      2: { status: pending | in-progress | complete, file: round-2-<id>.md }
```

**Round Files (round-N-<perspective>.md):**
```markdown
---
perspective: <perspective-id>
round: 1
timestamp: <ISO8601>
council_id: council-YYYY-MM-DD-<slug>
responding_to: []
word_count: <number>
---

# [Perspective Name] — Round [N]

[Perspective's argument content]
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
## Key Tensions
## Risk Assessment
## Recommended Path Forward
## Minority Report
## Suggested Actions
```

---

## Naming Conventions

**Files:**
- Perspective files: `<name>.md` (lowercase, hyphen-separated)
- Council ID: `council-YYYY-MM-DD-<slug>` (date + slugified topic)
- Round files: `round-<N>-<perspective-id>.md` (number + perspective)

**Perspectives:**
- Use full words: `maintainability` (not `maintain`)
- Lowercase with hyphens: `security`, `velocity`, `maintainability`

---

## Content Style Guidelines

### Markdown Content

**DO:**
- Use clear section headers (`##` for main sections)
- Keep response guidelines to 500–800 words for perspectives
- Structure arguments with bullet points for clarity
- Include specific examples or standards when applicable

**DON'T:**
- Make perspectives balanced — each argues ONE viewpoint forcefully
- Allow the moderator to advocate for any position (must remain neutral)
- Soften disagreements in synthesis (if irreconcilable, say so)

---

## Workflow Phases (5 Phases)

1. **Setup** — Interactive proposition framing + perspective selection + parameter confirmation
2. **Rounds** — Parallel perspective invocation via subagents, responses written to round files
3. **Gates** — User checkpoints: present round summaries, ask to proceed
4. **Synthesis** — Invoke summariser with all round files, write synthesis.md
5. **Archive** — Move council from `active/` to `archive/`, confirm to user

**Default:** 2 rounds, open visibility (perspectives see prior rounds)

---

## Error Handling

**If subagent fails:**
- Moderator informs user and offers to retry or skip that perspective

**If manifest corrupted:**
- Moderator reports error and asks user to inspect `council.yaml`

**If fewer than 2 perspectives found:**
- Moderator halts setup and informs user

**If interrupted:**
- Manifest and completed round files remain. User must start a new council.

---

## Adding New Perspectives

1. Create `skills/council/perspectives/<name>.md`
2. Use existing perspective files as templates
3. Define the perspective clearly in the prompt
4. Instruct to argue from ONE viewpoint (not balanced)
5. Include round 2+ response guidance
6. Keep responses to 500–800 words
7. Test with `/council <topic>` to verify the moderator discovers the new perspective

**Example perspectives:** cost-optimisation, user-experience, scalability, compliance, innovation, technical-simplicity

---

## Git Conventions

**Commit Message Format:**
```
feat(council): <description>
fix(council): <description>
docs: <description>
```

**Conventional Commit Types:**
- `feat`: New perspective, skill change, or feature
- `fix`: Bug fix in configuration
- `docs`: Documentation updates
- `chore`: Config changes

**What to commit:**
- Skill files (SKILL.md, perspectives, references)
- Documentation updates
- Council manifests/round files (if preserving debates)

**What NOT to commit:**
- Active council directories (use `docs/.gitignore` if you want to exclude them)

---

## Key Design Principles

1. **Human-in-the-loop:** User controls workflow via gate confirmations
2. **Plain text persistence:** All state in markdown/YAML (git-trackable, inspectable)
3. **Agent-agnostic perspectives:** Users bring their own perspective prompts
4. **Minimal viable orchestration:** Simplest workflow with value
5. **Parallel execution:** All perspectives within a round run simultaneously

---

## Scope (Current vs Future)

**In Scope:**
- Moderator + summariser + 3 default perspectives
- Custom perspectives (user can add perspectives)
- Parallel perspective execution per round
- Two-round debate with user gating
- Structured 6-section synthesis
- `/council` command
- Archive completed councils

**Out of Scope (Future):**
- Preset council types (architecture, adopt, incident, hire)
- Blind visibility mode (perspectives don't see prior rounds)
- Resume support for interrupted councils
- MCP integration for external data sources

---

## References

- **Skill definition:** `skills/council/SKILL.md`
- **Workflow protocol:** `skills/council/references/workflow.md`
- **Synthesis format:** `skills/council/references/synthesis-format.md`
- **Summariser instructions:** `skills/council/references/summariser-prompt.md`

---

## Quick Start for AI Agents

**To understand the system:**
1. Read `skills/council/SKILL.md` (entry point)
2. Read `skills/council/references/workflow.md` (protocol)
3. Look at one perspective file (e.g., `skills/council/perspectives/security.md`)

**To test the system:**
```bash
/council Should we migrate to microservices?
```

**To add a new perspective:**
1. Create `skills/council/perspectives/<your-perspective>.md`
2. Follow the structure of existing perspective files
3. Test by running `/council <topic>` and verifying the moderator finds it

**To modify workflow:**
- Edit `skills/council/references/workflow.md`
- DO NOT change file formats without updating all files that read/write those formats
