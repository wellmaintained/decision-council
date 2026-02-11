# Adapting the Council Pattern: A Single Skill for All Platforms

This document describes how to implement the decision council as a **single Agent Skill** that works across OpenCode, Claude Code, and Cursor without per-platform duplication.

---

## Table of Contents

1. [Background: The Agent Skills Standard](#background-the-agent-skills-standard)
2. [Architecture: One Skill, Three Platforms](#architecture-one-skill-three-platforms)
3. [Directory Structure](#directory-structure)
4. [The Skill Definition](#the-skill-definition)
5. [Perspective Files](#perspective-files)
6. [Shared Protocol Files](#shared-protocol-files)
7. [How It Works at Runtime](#how-it-works-at-runtime)
8. [Adding a New Perspective](#adding-a-new-perspective)
9. [Platform-Specific Considerations](#platform-specific-considerations)
10. [Comparison to Alternatives](#comparison-to-alternatives)

---

## Background: The Agent Skills Standard

The **Agent Skills** open standard ([agentskills.io](https://agentskills.io/specification)), originated by Anthropic in late 2025, defines a portable format for packaging agent capabilities as markdown files with YAML frontmatter. It has been adopted by all three target platforms:

| Platform | Reads skills from | Also reads |
|----------|------------------|------------|
| **Claude Code** | `.claude/skills/*/SKILL.md` | `~/.claude/skills/*/SKILL.md` |
| **OpenCode** | `.opencode/skills/*/SKILL.md` | `.claude/skills/*/SKILL.md`, `.agents/skills/*/SKILL.md` |
| **Cursor** | `.cursor/skills/*/SKILL.md` | `.claude/skills/*/SKILL.md` |

The critical detail: **OpenCode and Cursor both natively scan `.claude/skills/`** for cross-compatibility. A skill placed in `.claude/skills/council/SKILL.md` is discovered by all three platforms without any symlinking or build steps.

The standard also specifies that **unknown frontmatter fields are silently ignored**. A skill using only the standard fields (`name`, `description`, `allowed-tools`, `license`, `metadata`) works everywhere.

---

## Architecture: One Skill, Three Platforms

The council is implemented as a single skill directory. Everything lives in one place:

```
.claude/skills/council/
├── SKILL.md                        # Moderator prompt (the skill itself)
├── references/
│   ├── workflow.md                 # Workflow phases, rules, file conventions
│   ├── synthesis-format.md         # 6-section synthesis output spec
│   └── summariser-prompt.md        # Instructions for the synthesis step
└── perspectives/
    ├── security.md                 # Security & compliance perspective
    ├── velocity.md                 # Speed & pragmatism perspective
    └── maintainability.md          # Long-term code health perspective
```

This is the **entire implementation**. No per-platform wrappers, no generated files, no build step. The skill directory is committed to the repo and discovered by all three platforms.

### Why `.claude/skills/` and Not a Vendor-Neutral Path?

OpenCode also reads `.agents/skills/`, which is more vendor-neutral. However:

- `.claude/skills/` is read by **all three** target platforms today
- `.agents/skills/` is read by OpenCode but not yet by Cursor
- The Agent Skills standard originated with Claude Code; `.claude/skills/` is the de facto canonical path
- If a truly vendor-neutral path gains universal adoption, moving the directory is a one-line `git mv`

### What About OpenCode's Agent/Command System?

The current OpenCode implementation uses `.opencode/agents/council-moderator.md` + `.opencode/commands/council.md`. Moving to a skill changes the invocation model:

| | Current (agent + command) | Proposed (skill) |
|-|--------------------------|-------------------|
| Entry point | `/council` via command file | `/council` via skill discovery |
| Moderator prompt | Baked into agent file | SKILL.md body |
| Perspective prompts | Separate agent files | Reference files in skill directory |
| OpenCode-specific files | 6 files (.opencode/agents/ + .opencode/commands/) | 0 files |

OpenCode discovers skills from `.claude/skills/` and exposes them as `/council` in its command menu. The migration path is: keep the existing `.opencode/` files during transition, then remove them once the skill is validated.

---

## Directory Structure

```
.claude/skills/council/             # Discovered by Claude Code, OpenCode, and Cursor
├── SKILL.md                        # ← The moderator (entry point + orchestration)
├── references/
│   ├── workflow.md                 # ← Workflow protocol (phases, rules, templates)
│   ├── synthesis-format.md         # ← Synthesis output specification
│   └── summariser-prompt.md        # ← Summariser instructions
└── perspectives/                   # ← Perspective definitions
    ├── security.md                 #    Each file = one perspective
    ├── velocity.md                 #    No frontmatter, pure prompt content
    └── maintainability.md          #    Filename = perspective ID

docs/council/                       # ← Runtime output (unchanged)
├── active/                         #    Active councils
└── archive/                        #    Completed councils
```

Total files to maintain: **1 SKILL.md + 3 reference files + N perspective files**.
Platform-specific files: **zero**.

---

## The Skill Definition

### `.claude/skills/council/SKILL.md`

```yaml
---
name: council
description: >
  Run a structured multi-perspective debate on a technical decision.
  Spawns perspective subagents that argue from different viewpoints,
  manages debate rounds with user gate checkpoints between them,
  and produces a structured synthesis. Use when facing architectural
  decisions, technology choices, or trade-off analysis.
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
---
```

```markdown
You are the Council Moderator. You orchestrate structured
multi-perspective debates to support human decision-making.

## YOUR ROLE

You are neutral. You never advocate for any position. You manage
the process, not the arguments. You present information clearly
and let the perspectives speak for themselves.

## PROTOCOL

Read these files before proceeding:
- `.claude/skills/council/references/workflow.md` — workflow phases,
  rules, file conventions, task prompt templates, manifest structure
- `.claude/skills/council/references/synthesis-format.md` — synthesis
  output specification
- `.claude/skills/council/references/summariser-prompt.md` — instructions
  for the synthesis step

## PERSPECTIVE DISCOVERY

Discover available perspectives by globbing:
`.claude/skills/council/perspectives/*.md`

Each file name (without extension) is the perspective ID. Read each
file to get the perspective's full definition.

## ADVOCATE INVOCATION

For each perspective in a round:

1. Read the perspective file
2. Construct a task prompt using the round template from
   `workflow.md`, inserting:
   - The proposition as the PROPOSITION section
   - The perspective file content as the YOUR ROLE section
   - Prior round files (if round 2+) as the PRIOR ARGUMENTS section
3. Spawn a background subagent with this prompt
4. Repeat for all perspectives in the round — all in parallel
5. Collect all results before proceeding

## SUMMARISER INVOCATION

After the final round, construct a synthesis task prompt combining:
- The topic
- All round files
- The instructions from `summariser-prompt.md`
- The format from `synthesis-format.md`

Invoke as a subagent (not background — wait for result).
Write the output to `synthesis.md` in the council directory.

## GATES

Between every round, present a brief summary of each perspective's
arguments (3-5 sentences each) and ask the user to confirm before
proceeding. Do not proceed without explicit confirmation.

## SETUP

When invoked, immediately:
1. Read the workflow protocol
2. Discover available perspectives
3. Frame the user's topic as a clear proposition
4. Propose defaults (2 rounds, all discovered perspectives)
5. Ask the user to confirm or adjust — one question only

$ARGUMENTS
```

### Why This Works Across Platforms

The prompt uses **platform-agnostic language** for subagent invocation:

- "Spawn a background subagent with this prompt" — not `Task(subagent_type=general-purpose, run_in_background=true)` or `background_output(task_id=...)`
- Each platform's agent knows its own tool set. Claude Code will use the Task tool. OpenCode will use its Task tool. Cursor will use its native subagent mechanism.
- The prompt describes **what to do**, not **which API to call**

The only platform-specific token is `$ARGUMENTS` — the standard placeholder for user input in skills. OpenCode and Claude Code both support it. Cursor skills also receive user arguments, though the variable name may differ; the agent sees the arguments in context regardless.

---

## Perspective Files

Each perspective is a standalone markdown file with no frontmatter. The moderator reads these at runtime and injects them into subagent task prompts.

### `perspectives/security.md`

```markdown
# Security & Compliance Perspective

You are the Security and Compliance Perspective in a decision council.
Your role is to represent the cautious, risk-aware perspective that
prioritises security, compliance, and risk mitigation above other
considerations.

## Your Perspective

You approach every decision through the lens of security risk and
compliance requirements. You are not balanced or neutral — you
advocate strongly for the security viewpoint. Your job is to:

1. Identify vulnerabilities and attack vectors in proposed solutions
2. Assess compliance implications against relevant standards
3. Evaluate data protection and privacy risks
4. Challenge assumptions about security controls
5. Propose security-first alternatives that reduce risk

## How You Argue

- Lead with risk: start by identifying the most critical security gaps
- Use standards and frameworks: reference OWASP, CIS, NIST, etc.
- Think like an attacker: consider exploitation scenarios
- Quantify impact: describe consequences of security failures
- Demand evidence: ask for threat models, audits, test results
- Propose mitigations: offer concrete security controls

## Key Areas of Focus

- Authentication & authorisation
- Data protection (encryption, access control, logging)
- Vulnerability management
- Compliance (regulatory requirements, audit trails)
- Incident response (detection, investigation, response plans)
- Supply chain security

## In Round 2 and Beyond

1. Acknowledge valid points from other perspectives
2. Highlight security trade-offs in their proposals
3. Propose security-compatible alternatives
4. Challenge risk acceptance with business impact analysis
5. Escalate critical issues that should not proceed without mitigation

## Tone

- Firm but professional
- Evidence-based
- Constructive: offer solutions, not just problems
- Persistent: security is non-negotiable

## Response Format

- 500-800 words
- Clear sections with headers
- Bullet points for lists of concerns or recommendations
- End with a clear position statement
```

### `perspectives/velocity.md` and `perspectives/maintainability.md`

Same structure, different content. Each defines: perspective values, argumentation strategy, key focus areas, multi-round engagement rules, tone, and response format. See the existing OpenCode perspective files for the full content — the body transfers directly, minus the YAML frontmatter.

---

## Shared Protocol Files

### `references/workflow.md`

Contains the workflow phases, rules, file conventions, task prompt templates, manifest YAML structure, round file frontmatter format, and error handling procedures. This is the moderator's operating manual — identical to what was previously baked into the moderator agent's system prompt.

Key sections:
- **Phases**: Setup, Debate Rounds, Gate Checkpoints, Synthesis, Archive
- **Rules**: Neutral moderator, user gates, one question per message, defaults
- **File Conventions**: Council ID format, directory structure, file naming
- **Task Prompt Templates**: Round 1 and Round 2+ templates with placeholder sections
- **Manifest Structure**: YAML schema for `council.yaml`
- **Error Handling**: Subagent failure, manifest corruption, minimum perspective count

### `references/synthesis-format.md`

The six-section synthesis output specification: Consensus, Key Tensions, Risk Assessment, Recommended Path Forward, Minority Report, Suggested Actions. Plus rules for attribution, honesty about disagreements, and word limits.

### `references/summariser-prompt.md`

Instructions for the synthesis subagent: what inputs it receives, how to follow the synthesis format, YAML frontmatter for the output file, and the constraint that it synthesises rather than creates.

---

## How It Works at Runtime

```
User types: /council Should we migrate from REST to GraphQL?

┌─────────────────────────────────────────────────────────────┐
│ Platform discovers .claude/skills/council/SKILL.md          │
│ Loads SKILL.md body as the moderator's prompt               │
│ Passes "Should we migrate..." as $ARGUMENTS                 │
├─────────────────────────────────────────────────────────────┤
│ SETUP PHASE                                                 │
│                                                             │
│ Moderator reads references/workflow.md                      │
│ Moderator globs perspectives/*.md → finds 3 perspectives    │
│ Moderator frames proposition, proposes defaults              │
│ User confirms: 2 rounds, all 3 perspectives                 │
│ Moderator creates docs/council/active/<id>/council.yaml     │
├─────────────────────────────────────────────────────────────┤
│ ROUND 1                                                     │
│                                                             │
│ For each perspective (in parallel):                          │
│   Moderator reads perspectives/<id>.md                      │
│   Moderator constructs task prompt from workflow template    │
│   ┌───────────────────────────────────────────────────────┐ │
│   │ PROPOSITION: Should we migrate from REST to GraphQL?  │ │
│   │ YOUR ROLE: [security.md content]                      │ │
│   │ INSTRUCTIONS: Present your strongest arguments...     │ │
│   └───────────────────────────────────────────────────────┘ │
│   Moderator spawns background subagent with this prompt     │
│                                                             │
│ Moderator collects all 3 results                            │
│ Moderator writes round-1-security.md, etc.                  │
│ Moderator updates council.yaml                              │
├─────────────────────────────────────────────────────────────┤
│ GATE 1                                                      │
│                                                             │
│ Moderator presents 3-5 sentence summary per perspective     │
│ User confirms → proceed to round 2                          │
├─────────────────────────────────────────────────────────────┤
│ ROUND 2 (same as round 1, but task prompt includes          │
│          prior round files as PRIOR ARGUMENTS)              │
├─────────────────────────────────────────────────────────────┤
│ GATE 2 → User confirms → proceed to synthesis               │
├─────────────────────────────────────────────────────────────┤
│ SYNTHESIS                                                   │
│                                                             │
│ Moderator reads references/summariser-prompt.md             │
│ Moderator reads references/synthesis-format.md              │
│ Moderator constructs synthesis task prompt                   │
│ Moderator spawns subagent (foreground, waits for result)    │
│ Moderator writes synthesis.md                               │
│ Moderator archives council                                  │
└─────────────────────────────────────────────────────────────┘
```

The entire flow is platform-agnostic. Each platform uses its own native mechanism for "spawn a background subagent" and "read a file," but the moderator prompt never references those mechanisms by name.

---

## Adding a New Perspective

### Step 1: Create `.claude/skills/council/perspectives/cost.md`

```markdown
# Cost & Financial Perspective

You are the Cost and Financial Perspective in a decision council.
Your role is to argue from a financial and resource allocation
perspective.

## Your Perspective

You approach every decision through the lens of financial impact
and resource efficiency. You are not balanced — you advocate for
fiscal responsibility. Your job is to:

1. Quantify costs (implementation, maintenance, opportunity)
2. Surface hidden financial implications
3. Challenge ROI assumptions
4. Propose cost-effective alternatives
5. Evaluate total cost of ownership

## How You Argue
[... same structure as other perspectives ...]
```

### Step 2: There is no step 2.

The moderator discovers perspectives via glob. The new file appears on all three platforms immediately.

---

## Platform-Specific Considerations

### Argument Passing

| Platform | Skill argument mechanism | `$ARGUMENTS` supported? |
|----------|-------------------------|-------------------------|
| Claude Code | `$ARGUMENTS` placeholder in SKILL.md | Yes (native) |
| OpenCode | `$ARGUMENTS` placeholder | Yes (native) |
| Cursor | Arguments passed to skill context | Yes (via compat layer) |

If Cursor's skill system doesn't expand `$ARGUMENTS` literally, the user's topic still appears in the conversation context. The moderator prompt's final line ("$ARGUMENTS") acts as a signal to use whatever topic text is available. This is a graceful degradation, not a failure.

### Subagent Spawning

The moderator prompt says "spawn a background subagent." Each platform interprets this using its native mechanism:

| Platform | What "spawn a background subagent" translates to |
|----------|--------------------------------------------------|
| Claude Code | `Task(subagent_type="general-purpose", run_in_background=true)` |
| OpenCode | `Task(run_in_background=true)` via built-in task tool |
| Cursor | Native parallel subagent spawning |

The prompt does not prescribe the mechanism. The agent uses whatever subagent tool is available. All three platforms support parallel subagent execution sufficient for 3-7 concurrent perspectives.

### Parallel Execution Limits

| Platform | Max concurrent subagents | Sufficient for council? |
|----------|-------------------------|------------------------|
| Claude Code | ~7 | Yes |
| OpenCode | Framework-dependent | Yes (typically 5+) |
| Cursor | ~8 (via git worktrees) | Yes |

### Skill Discovery Path Priority

If the same skill name exists in multiple locations, platform-specific paths take priority:

| Platform | Priority order |
|----------|---------------|
| Claude Code | `.claude/skills/` (only reads its own path) |
| OpenCode | `.opencode/skills/` > `.claude/skills/` > `.agents/skills/` |
| Cursor | `.cursor/skills/` > `.claude/skills/` |

This means you can **override** the shared skill for a specific platform by placing a modified copy in the platform-specific directory, while the `.claude/skills/` version serves as the default.

### Temperature and Model Selection

The Agent Skills standard does not include `temperature` or `model` fields (these are Claude Code extensions). All perspectives run at the platform's default temperature and model. Perspective differentiation comes from the prompt content, not sampling parameters.

Claude Code's extended fields (`allowed-tools`, `argument-hint`, `context`, `model`) are silently ignored by OpenCode and Cursor per the spec's forward-compatibility rule.

---

## Comparison to Alternatives

| Approach | Platform-specific files | Perspective duplication | New perspective effort | Moderator duplication |
|----------|------------------------|------------------------|----------------------|-----------------------|
| **Current** (OpenCode-only agents) | 6 (agents + command) | N/A (single platform) | 1 agent file | N/A |
| **Naive port** (per-platform agents) | 6 per platform (18 total) | 3x per perspective | 3 files | 3x moderator prompt |
| **Previous proposal** (shared perspectives + platform wrappers) | 1 per platform (3 total) | None | 1 file | ~60% shared, 40% per-platform |
| **Single skill** (this proposal) | 0 | None | 1 file | None |

The single-skill approach achieves **zero platform-specific files** and **zero duplication** by leveraging the Agent Skills standard's cross-platform discovery. The entire council — moderator, perspectives, workflow protocol, synthesis format — lives in one directory that all three platforms read natively.

### When You Might Still Want Platform-Specific Files

- **OpenCode `temperature` tuning**: If per-perspective temperature control matters, keep an `.opencode/agents/council-moderator.md` with `temperature: 0.2` alongside the skill. The agent file handles orchestration config; the skill provides the perspectives.
- **Cursor model mixing**: If you want different models per perspective, a `.cursor/agents/` override can specify `model` per subagent.
- **Platform-specific hooks**: Validation hooks (e.g., manifest schema checking) use platform-specific config files (`.claude/settings.json`, `.cursor/hooks.json`).

These are **optional enhancements**, not requirements. The skill works without them.
