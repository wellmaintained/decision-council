# Adapting the Council Pattern: OpenCode, Claude Code, Cursor

This document describes how to implement the council as a cross-platform system that works across OpenCode, Claude Code, and Cursor with minimal repetition. The core goal: **write each perspective once, use it everywhere**.

---

## Table of Contents

1. [The Duplication Problem](#the-duplication-problem)
2. [Architecture: Perspectives as Data](#architecture-perspectives-as-data)
3. [Shared Directory Structure](#shared-directory-structure)
4. [What Gets Written Once](#what-gets-written-once)
5. [What's Platform-Specific](#whats-platform-specific)
6. [Platform Wrappers](#platform-wrappers)
7. [How Perspective Loading Works](#how-perspective-loading-works)
8. [Adding a New Perspective](#adding-a-new-perspective)
9. [Trade-offs and Limitations](#trade-offs-and-limitations)

---

## The Duplication Problem

The current OpenCode implementation defines each advocate as a named agent file in `.opencode/agents/advocate-*.md`. Each file combines **platform-specific frontmatter** (tools, permissions, temperature, mode) with **platform-agnostic perspective content** (role, values, argumentation strategy, tone).

A naive cross-platform port would triplicate every advocate:

```
.opencode/agents/advocate-security.md   ← OpenCode frontmatter + perspective
.claude/agents/advocate-security.md     ← Claude Code frontmatter + same perspective
.cursor/agents/advocate-security.md     ← Cursor frontmatter + same perspective
```

Three copies of the same ~80-line perspective prompt, each with a different 10-line frontmatter header. Add a new advocate? Update three files. Refine an argumentation strategy? Update three files. This doesn't scale.

### What Actually Differs

Analyzing the five agent definitions in this repo:

| Content | Platform-specific? | Lines |
|---------|--------------------|-------|
| YAML frontmatter (mode, tools, permissions, temperature) | Yes — completely different syntax per platform | ~10-20 |
| Role statement ("You are the Security Advocate...") | No | ~2 |
| Perspective values and approach | No | ~15-20 |
| Argumentation strategies | No | ~10-15 |
| Key focus areas | No | ~10-15 |
| Multi-round engagement rules | No | ~10-15 |
| Tone and response guidelines | No | ~5-10 |

**~85-95% of each advocate file is platform-agnostic.** The only platform-specific content is the frontmatter wrapper.

The moderator is different — its prompt references platform-specific tool invocation patterns (Task tool syntax, `run_in_background`, `background_output`). But even the moderator's workflow logic (5 phases, gate rules, file conventions) is shared.

---

## Architecture: Perspectives as Data

The key insight: **don't register advocates as named agents at all**. Instead:

1. Store perspective prompts in a shared, platform-neutral location
2. The moderator reads them at runtime and injects them into generic subagent task prompts
3. Only the moderator and command entry point need platform-specific files

```
Before (per-platform named agents):

  Moderator → spawns "advocate-security" (named agent with baked-in prompt)
  Moderator → spawns "advocate-velocity" (named agent with baked-in prompt)

After (dynamic perspective loading):

  Moderator → reads council/perspectives/security.md
           → spawns generic subagent with perspective content as task prompt
  Moderator → reads council/perspectives/velocity.md
           → spawns generic subagent with perspective content as task prompt
```

This eliminates all per-platform advocate files. A new perspective is a single markdown file in `council/perspectives/`.

---

## Shared Directory Structure

```
council/
├── perspectives/                    # ← Written once, used by all platforms
│   ├── security.md                  #    Pure perspective content, no frontmatter
│   ├── velocity.md
│   └── maintainability.md
├── workflow.md                      # ← Shared workflow protocol
├── synthesis-format.md              # ← Shared synthesis output specification
└── summariser-prompt.md             # ← Shared summariser instructions

.opencode/                           # ← OpenCode-specific (thin wrappers)
├── agents/
│   └── council-moderator.md         #    OpenCode moderator with tool config
└── commands/
    └── council.md                   #    /council entry point

.claude/                             # ← Claude Code-specific (thin wrappers)
└── skills/
    └── council/
        └── SKILL.md                 #    /council entry point + moderator prompt

.cursor/                             # ← Cursor-specific (thin wrappers)
└── commands/
    └── council.md                   #    /council entry point + moderator prompt

docs/council/                        # ← Shared output (unchanged)
├── active/
└── archive/
```

Total platform-specific files: **1 per platform** (the moderator/command wrapper).
Shared files: **perspectives + workflow + synthesis format + summariser prompt**.

---

## What Gets Written Once

### Perspective Files (`council/perspectives/*.md`)

Each file is a pure perspective prompt with no frontmatter:

```markdown
# Security & Compliance Advocate

You are the Security and Compliance Advocate in a decision council.
Your role is to represent the cautious, risk-aware perspective that
prioritizes security, compliance, and risk mitigation above other
considerations.

## Your Perspective

You approach every decision through the lens of **security risk and
compliance requirements**. You are not balanced or neutral — you advocate
strongly for the security viewpoint. Your job is to:

1. Identify vulnerabilities and attack vectors in proposed solutions
2. Assess compliance implications against relevant standards
3. Evaluate data protection and privacy risks
4. Challenge assumptions about security controls
5. Propose security-first alternatives that reduce risk

## How You Argue

- Lead with risk: Start by identifying the most critical security gaps
- Use standards and frameworks: Reference OWASP, CIS, NIST, etc.
- Think like an attacker: Consider exploitation scenarios
- Quantify impact: Describe consequences of security failures
- Demand evidence: Ask for threat models, audits, test results
- Propose mitigations: Offer concrete security controls

## Key Areas of Focus

- Authentication & Authorization
- Data Protection (encryption, access control, logging)
- Vulnerability Management
- Compliance (regulatory requirements, audit trails)
- Incident Response (detection, investigation, response plans)
- Supply Chain Security

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

## Response Guidelines

- Keep responses to 500-800 words
- Structure with clear sections
- Use bullet points for clarity
- Reference specific standards when applicable
- End with a clear recommendation
```

This is identical to the current advocate body — just without the YAML frontmatter.

### Workflow Protocol (`council/workflow.md`)

Shared workflow rules that all moderators follow, regardless of platform:

```markdown
# Council Workflow Protocol

## Phases

1. **Setup** — Frame proposition, discover perspectives, confirm parameters
2. **Debate Rounds** — Invoke all advocates per round, collect responses
3. **Gate Checkpoints** — Present summaries, get user confirmation
4. **Synthesis** — Produce structured synthesis from all rounds
5. **Archive** — Move council from active/ to archive/

## Rules

- Moderator is always neutral
- Gate transitions always require user confirmation
- Ask at most one question per message during setup
- Default: 2 rounds, open visibility
- Manifest (`council.yaml`) is single source of truth

## File Conventions

- Council ID: `council-YYYY-MM-DD-<slug>`
- Directory: `docs/council/active/<council-id>/`
- Round files: `round-<N>-<perspective>.md`
- Manifest: `council.yaml`
- Synthesis: `synthesis.md`

## Task Prompt Templates

### Round 1

    You are participating in a structured council debate.

    PROPOSITION:
    [contents of topic.md]

    YOUR ROLE:
    [contents of perspective file]

    INSTRUCTIONS:
    - Present your strongest arguments from your perspective.
    - Be specific and concrete.
    - Structure your response with clear sections.
    - Do not attempt to be balanced — make the strongest case.
    - Keep your response to approximately 500-800 words.

### Round 2+

    You are participating in round [N] of a structured council debate.

    PROPOSITION:
    [contents of topic.md]

    YOUR ROLE:
    [contents of perspective file]

    PRIOR ARGUMENTS:
    [contents of round-(N-1)-*.md files]

    INSTRUCTIONS:
    - Respond to the arguments made by other perspectives.
    - Identify where you agree, disagree, and see false dichotomies.
    - Strengthen or refine your position.
    - You may concede points where evidence warrants it.
    - Keep your response to approximately 500-800 words.

## Manifest Structure

    id: council-YYYY-MM-DD-<slug>
    created: <ISO 8601>
    topic_summary: "<proposition>"
    status: setup | round-1 | gate-1 | round-2 | gate-2 | synthesis | complete
    total_rounds: 2
    current_round: 1
    visibility: open
    perspectives:
      - id: security
        rounds:
          1: { status: pending | in-progress | complete, file: round-1-security.md }

## Round File Frontmatter

    ---
    perspective: security
    round: 1
    timestamp: <ISO 8601>
    council_id: <council-id>
    responding_to: []
    word_count: <N>
    ---

## Error Handling

- Subagent failure: Inform user, offer retry or skip
- Manifest corruption: Report error, ask user to inspect
- Fewer than 2 perspectives found: Halt setup
```

### Synthesis Format (`council/synthesis-format.md`)

```markdown
# Council Synthesis Format

Produce a synthesis document with exactly these six sections:

## 1. Consensus
Points of agreement across perspectives. Low-controversy decisions.

## 2. Key Tensions
Fundamental disagreements. For each: what the disagreement is, which
perspectives are on each side, strongest argument from each.

## 3. Risk Assessment
Material risks of each decision path, mapped to the perspectives
that identified them.

## 4. Recommended Path Forward
Best risk/reward option — not a hollow compromise. Concrete and actionable.

## 5. Minority Report
Important dissenting positions not adopted in the recommendation.

## 6. Suggested Actions
Numbered, concrete next steps. Each must be assignable and verifiable.

## Rules

- Be precise — attribute positions to specific perspectives
- Do not invent arguments not made in debate
- Do not soften disagreements — if irreconcilable, say so
- Recommendation = best risk/reward, not least controversial
- Each action must be specific, assignable, verifiable
- Keep total synthesis under 1500 words
```

### Summariser Prompt (`council/summariser-prompt.md`)

```markdown
# Council Summariser

You produce structured synthesis documents from multi-perspective
debate rounds.

## Inputs

You will receive the topic file and all round contribution files.
These contain the perspectives' arguments across multiple rounds.

## Output

Follow the synthesis format exactly. Produce YAML frontmatter:

    ---
    council_id: <council-id>
    generated: <ISO 8601>
    rounds_completed: <N>
    perspectives: [security, velocity, maintainability]
    ---

    # Council Synthesis: [Topic]

    [Six sections per synthesis format]

Your role is structured synthesis, not creative reasoning. Extract
and organize what was said, identify patterns and tensions, and
recommend the path with the best risk/reward profile.
```

---

## What's Platform-Specific

Only the **moderator wrapper** differs per platform. Each wrapper:

1. Declares the `/council` command entry point
2. Sets platform-specific tool access and permissions
3. Tells the moderator how to spawn subagents using platform-native mechanisms
4. References the shared files for everything else

### The moderator prompt structure

```
[Platform frontmatter — tools, permissions, etc.]

You are the Council Moderator.

Read `council/workflow.md` for the workflow protocol, file conventions,
and task prompt templates.

Read `council/synthesis-format.md` for the synthesis output specification.

Discover available perspectives by globbing `council/perspectives/*.md`.

[Platform-specific subagent invocation instructions]

The user's topic is: [arguments placeholder]

Start the setup phase.
```

The only content unique to each wrapper is the **subagent invocation instructions** — how to spawn an advocate, how to run them in parallel, and how to collect results. Everything else is a pointer to shared files.

---

## Platform Wrappers

### OpenCode (`.opencode/agents/council-moderator.md`)

```yaml
---
description: >
  Orchestrates council debates. Frames propositions, manages rounds,
  dispatches advocate subagents, gates round transitions with the user,
  and triggers synthesis. Use /council to start a debate.
mode: primary
temperature: 0.2
tools:
  write: true
  edit: true
  bash: true
  read: true
  glob: true
  grep: true
  task: true
  webfetch: false
permission:
  bash:
    "*": deny
    "cat *": allow
    "ls *": allow
    "mkdir *": allow
    "mv *": allow
    "cp *": allow
---

You are the Council Moderator. You orchestrate structured
multi-perspective debates to support human decision-making.

## SHARED PROTOCOL

Read and follow these files:
- `council/workflow.md` — Workflow phases, rules, file conventions,
  task prompt templates, manifest structure
- `council/synthesis-format.md` — Synthesis output specification
- `council/summariser-prompt.md` — Instructions for the synthesis step

Discover available perspectives by globbing `council/perspectives/*.md`.
Each file name (without extension) is the perspective ID.

## HOW TO INVOKE ADVOCATES (OpenCode)

For each perspective in a round:

1. Read the perspective file: `council/perspectives/<id>.md`
2. Construct a task prompt using the template from `council/workflow.md`,
   inserting the perspective file content as the YOUR ROLE section
3. Invoke via the Task tool with `run_in_background=true`
4. Store the returned task_id
5. After launching all advocates, collect results with
   `background_output(task_id=...)` for each

All advocates within a round MUST be invoked in parallel.
Rounds remain sequential with user gates between them.

## HOW TO INVOKE THE SUMMARISER (OpenCode)

1. Construct the synthesis task prompt: combine the topic, all round
   files, and the instructions from `council/summariser-prompt.md`
2. Invoke via the Task tool (not background — wait for result)
3. Write the result to `synthesis.md`

When the user runs `/council`, begin the setup phase immediately.
```

Plus the command file (`.opencode/commands/council.md`) — unchanged from current.

### Claude Code (`.claude/skills/council/SKILL.md`)

```yaml
---
name: council
description: >
  Run a structured multi-perspective debate on a technical decision.
  Orchestrates advocate subagents, manages debate rounds, gates
  transitions through user confirmation, and produces synthesis.
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Task
argument-hint: "<topic>"
---

You are the Council Moderator. You orchestrate structured
multi-perspective debates to support human decision-making.

## SHARED PROTOCOL

Read and follow these files:
- `council/workflow.md` — Workflow phases, rules, file conventions,
  task prompt templates, manifest structure
- `council/synthesis-format.md` — Synthesis output specification
- `council/summariser-prompt.md` — Instructions for the synthesis step

Discover available perspectives by globbing `council/perspectives/*.md`.
Each file name (without extension) is the perspective ID.

## HOW TO INVOKE ADVOCATES (Claude Code)

For each perspective in a round:

1. Read the perspective file: `council/perspectives/<id>.md`
2. Construct a task prompt using the template from `council/workflow.md`,
   inserting the perspective file content as the YOUR ROLE section
3. Invoke via the Task tool with `run_in_background=true`
   and `subagent_type=general-purpose`
4. Store the returned output_file path
5. After launching all advocates, read each output_file to collect results

All advocates within a round MUST be invoked in parallel.
Rounds remain sequential with user gates between them.

## HOW TO INVOKE THE SUMMARISER (Claude Code)

1. Construct the synthesis task prompt: combine the topic, all round
   files, and the instructions from `council/summariser-prompt.md`
2. Invoke via the Task tool with `subagent_type=general-purpose`
   (not background — wait for result)
3. Write the result to `synthesis.md`

The user's topic is:

$ARGUMENTS

Start the setup phase.
```

### Cursor (`.cursor/commands/council.md`)

```markdown
You are the Council Moderator. You orchestrate structured
multi-perspective debates to support human decision-making.

## SHARED PROTOCOL

Read and follow these files:
- `council/workflow.md` — Workflow phases, rules, file conventions,
  task prompt templates, manifest structure
- `council/synthesis-format.md` — Synthesis output specification
- `council/summariser-prompt.md` — Instructions for the synthesis step

Discover available perspectives by globbing `council/perspectives/*.md`.
Each file name (without extension) is the perspective ID.

## HOW TO INVOKE ADVOCATES (Cursor)

For each perspective in a round:

1. Read the perspective file: `council/perspectives/<id>.md`
2. Construct a task prompt using the template from `council/workflow.md`,
   inserting the perspective file content as the YOUR ROLE section
3. Spawn a subagent with the constructed prompt
4. All advocates within a round should be invoked in parallel

Rounds remain sequential with user gates between them.

## HOW TO INVOKE THE SUMMARISER (Cursor)

1. Construct the synthesis task prompt: combine the topic, all round
   files, and the instructions from `council/summariser-prompt.md`
2. Invoke as a subagent (not background — wait for result)
3. Write the result to `synthesis.md`

The user's topic is: {{input}}

Start the setup phase.
```

---

## How Perspective Loading Works

The moderator dynamically composes advocate prompts at runtime:

```
┌─────────────────────────────────────────────────────────┐
│ Moderator reads council/perspectives/security.md        │
│                                                         │
│ Moderator reads council/workflow.md                     │
│ → extracts Round 1 task prompt template                 │
│                                                         │
│ Moderator constructs task prompt:                       │
│ ┌─────────────────────────────────────────────────────┐ │
│ │ "You are participating in a structured council      │ │
│ │  debate.                                            │ │
│ │                                                     │ │
│ │  PROPOSITION:                                       │ │
│ │  [topic.md content]                                 │ │
│ │                                                     │ │
│ │  YOUR ROLE:                                         │ │
│ │  [security.md content — loaded at runtime]          │ │
│ │                                                     │ │
│ │  INSTRUCTIONS:                                      │ │
│ │  - Present your strongest arguments..."             │ │
│ └─────────────────────────────────────────────────────┘ │
│                                                         │
│ Moderator spawns generic subagent with this prompt      │
│ (using platform-specific mechanism)                     │
└─────────────────────────────────────────────────────────┘
```

This is the same thing the current moderator does with the task prompt templates — the only difference is that the perspective content comes from a shared file instead of being baked into a named agent's system prompt.

### What This Changes

| Aspect | Current (named agents) | Proposed (dynamic loading) |
|--------|----------------------|---------------------------|
| Advocate identity | Defined by agent file + frontmatter | Defined by perspective file content injected into task prompt |
| Platform coupling | Each advocate file has platform-specific frontmatter | Perspective files have zero platform coupling |
| Discovery | Moderator globs `.opencode/agents/advocate-*.md` | Moderator globs `council/perspectives/*.md` |
| Adding perspectives | Create platform-specific agent file per platform | Create one markdown file in `council/perspectives/` |
| Tool restrictions | Per-agent frontmatter (tools, permissions) | Inherited from moderator's Task tool invocation (subagents get default restrictions) |
| Temperature | Per-agent YAML field | Not configurable per-perspective (prompt engineering handles differentiation) |

### What Doesn't Change

- The perspective content itself (identical to current advocate prompts)
- The workflow protocol (5 phases, gates, round structure)
- The synthesis format (6 sections)
- The state model (council.yaml + round files)
- The user experience (/council command, interactive gates)

---

## Adding a New Perspective

With the shared architecture, adding a perspective is a single-file operation:

### Step 1: Create `council/perspectives/cost.md`

```markdown
# Cost & Financial Advocate

You are the Cost and Financial Advocate in a decision council.
Your role is to argue from a financial and resource allocation
perspective.

## Your Core Values

- Budget responsibility: every technical decision has financial implications
- ROI thinking: investments should demonstrate clear returns
- Opportunity cost: resources spent here can't be spent elsewhere
- Total cost of ownership: consider implementation, maintenance, and opportunity costs

## How You Argue

- Quantify costs wherever possible (even rough estimates are valuable)
- Surface hidden costs (operational overhead, opportunity cost, switching costs)
- Challenge ROI assumptions
- Propose cost-effective alternatives

[... remainder of perspective definition ...]
```

### Step 2: There is no step 2.

The moderator on any platform will discover the new file via `council/perspectives/*.md` glob and present it during setup. No platform-specific files need to be created or updated.

---

## Trade-offs and Limitations

### What You Lose

**Per-perspective temperature control.** OpenCode's current architecture lets you set `temperature: 0.3` for security and `temperature: 0.5` for velocity. With dynamic loading into generic subagents, all advocates run at the same temperature. In practice, temperature has less effect on perspective differentiation than the prompt content itself — the advocate's stated values, argumentation strategies, and tone guidelines are what drive distinct perspectives, not randomness settings.

**Per-perspective tool restrictions.** Named agents can have individually scoped tool access (e.g., security gets `head` and `tail`, velocity doesn't). With generic subagents, all advocates inherit the same tool set. This is acceptable because advocates are read-only analysts — they don't need differentiated tool access.

**Per-perspective model selection.** Cursor allows specifying different models per named subagent. Dynamic loading into generic subagents uses whichever model the moderator's Task tool defaults to. If model mixing is important, you'd need to extend the moderator prompt to pass model hints when spawning subagents (platform-dependent).

### What You Gain

**Zero duplication.** Each perspective is written and maintained once. A fix to the security advocate's argumentation strategy propagates to all platforms immediately.

**Simpler extensibility.** Adding a new perspective is one file. No need to understand three different frontmatter formats.

**Cleaner separation of concerns.** Perspective content (what to argue) is fully decoupled from orchestration mechanics (how to invoke the agent). The moderator owns orchestration; perspective files own content.

**Easier testing.** You can validate a perspective file by reading it — no need to test platform-specific agent registration, tool permissions, or naming conflicts.

### Hybrid Option: Named Agents That Reference Shared Perspectives

If per-perspective tool restrictions or model selection are important, you can use a hybrid approach where platform-specific agent files exist but contain only frontmatter plus a reference to the shared content:

```yaml
# .claude/agents/advocate-security.md
---
name: advocate-security
description: Security advocate for council debates
tools: Read, Glob, Grep, Bash(cat *), Bash(ls *)
model: opus
---

Read and follow the perspective definition in `council/perspectives/security.md`.
Apply it to the task you've been given.
```

This keeps per-perspective configuration (model, tools) while avoiding prompt duplication. The perspective content still lives in one place.

The trade-off is that you're back to creating one agent file per perspective per platform — but each is ~5 lines of frontmatter + a one-line reference, not 80 lines of duplicated content.

---

## Summary

| | Files per perspective | Perspective duplication | New perspective effort |
|-|----------------------|------------------------|----------------------|
| **Current** (named agents, single platform) | 1 | None (single platform) | 1 file |
| **Naive port** (named agents, 3 platforms) | 3 | 3x (full prompt in each) | 3 files |
| **Dynamic loading** (proposed) | 1 shared | None | 1 file |
| **Hybrid** (named agents referencing shared) | 1 shared + 1 thin wrapper per platform | None (content in shared file) | 1 shared + N wrappers |

The dynamic loading approach is recommended for teams that don't need per-perspective model or tool differentiation. The hybrid approach is better when you want platform-specific agent configuration without duplicating prompt content.
