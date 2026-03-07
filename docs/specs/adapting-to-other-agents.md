# Adapting the Council Pattern: A Single Skill for All Platforms

This document describes how the decision council is implemented as a **single Agent Skill** that works across Claude Code, OpenCode, and Cursor without per-platform duplication.

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

---

## Background: The Agent Skills Standard

The **Agent Skills** open standard ([agentskills.io](https://agentskills.io/specification)), originated by Anthropic in late 2025, defines a portable format for packaging agent capabilities as markdown files with YAML frontmatter. It has been adopted by all three target platforms:

| Platform | Reads skills from | Also reads |
|----------|------------------|------------|
| **Claude Code** | `.claude/skills/*/SKILL.md` | `~/.claude/skills/*/SKILL.md` |
| **OpenCode** | `.opencode/skills/*/SKILL.md` | `.claude/skills/*/SKILL.md`, `.agents/skills/*/SKILL.md` |
| **Cursor** | `.cursor/skills/*/SKILL.md` | `.claude/skills/*/SKILL.md` |

The critical detail: **OpenCode and Cursor both natively scan `.claude/skills/`** for cross-compatibility. A skill placed in `.claude/skills/council/SKILL.md` is discovered by all three platforms without any symlinking or build steps.

The standard also specifies that **unknown frontmatter fields are silently ignored**. A skill using only the standard fields (`name`, `description`, `allowed-tools`, `licence`, `metadata`) works everywhere.

---

## Architecture: One Skill, Three Platforms

The council is implemented as a single skill directory. The canonical content lives in `skills/council/`, symlinked from `.claude/skills/council/`:

```
skills/council/                         # Canonical skill content
├── SKILL.md                            # Moderator prompt (the skill itself)
├── references/
│   ├── workflow.md                     # Workflow phases, rules, file conventions
│   ├── synthesis-format.md             # 6-section synthesis output spec
│   └── summariser-prompt.md            # Instructions for the synthesis step
└── perspectives/
    ├── security.md                     # Security & compliance perspective
    ├── velocity.md                     # Speed & pragmatism perspective
    └── maintainability.md              # Long-term code health perspective

.claude/skills/council -> ../../skills/council   # Symlink for platform discovery
```

This is the **entire implementation**. No per-platform wrappers, no generated files, no build step.

---

## Directory Structure

```
skills/council/                        # Canonical location for all skill content
├── SKILL.md                           # ← The moderator (entry point + orchestration)
├── references/
│   ├── workflow.md                    # ← Workflow protocol (phases, rules, templates)
│   ├── synthesis-format.md            # ← Synthesis output specification
│   └── summariser-prompt.md           # ← Summariser instructions
└── perspectives/                      # ← Perspective definitions
    ├── security.md                    #    Each file = one perspective
    ├── velocity.md                    #    No frontmatter, pure prompt content
    └── maintainability.md             #    Filename = perspective ID

.claude/skills/council                 # Symlink → ../../skills/council

docs/council/                          # ← Runtime output (unchanged)
├── active/                            #    Active councils
└── archive/                           #    Completed councils
```

Total files to maintain: **1 SKILL.md + 3 reference files + N perspective files**.
Platform-specific files: **zero** (just one symlink).

---

## The Skill Definition

### `skills/council/SKILL.md`

The SKILL.md contains YAML frontmatter (name, description, allowed-tools) followed by the moderator's system prompt. The moderator:

- Reads reference files at runtime for workflow protocol and synthesis format
- Discovers perspectives dynamically via glob
- Uses platform-agnostic language for subagent invocation

### Why Platform-Agnostic Language Works

The prompt uses natural language for subagent invocation:

- "Spawn a background subagent with this prompt" — not platform-specific API calls
- Each platform's agent knows its own tool set (Claude Code uses Agent tool, OpenCode uses Task tool, Cursor uses its native mechanism)
- The prompt describes **what to do**, not **which API to call**

The only platform-specific token is `$ARGUMENTS` — the standard placeholder for user input in skills.

---

## Perspective Files

Each perspective is a standalone markdown file with no frontmatter. The moderator reads these at runtime and injects them into subagent task prompts.

Structure:
- `## Your Perspective` — What this perspective argues for
- `## How You Argue` — Argumentation strategy
- `## Key Areas of Focus` — Domain-specific concerns
- `## In Round 2 and Beyond` — Multi-round engagement rules
- `## Tone` — Communication style
- `## Response Format` — Word count and structure guidelines

---

## Shared Protocol Files

### `references/workflow.md`

The moderator's operating manual: workflow phases, rules, file conventions, task prompt templates, manifest YAML structure, round file frontmatter format, and error handling procedures.

### `references/synthesis-format.md`

The six-section synthesis output specification: Consensus, Key Tensions, Risk Assessment, Recommended Path Forward, Minority Report, Suggested Actions. Plus rules for attribution, honesty about disagreements, and word limits.

### `references/summariser-prompt.md`

Instructions for the synthesis subagent: what inputs it receives, how to follow the synthesis format, YAML frontmatter for the output file, and the constraint that it synthesises rather than creates.

---

## How It Works at Runtime

```
User types: /council Should we migrate from REST to GraphQL?

┌─────────────────────────────────────────────────────────────┐
│ Platform discovers .claude/skills/council/SKILL.md           │
│ (via symlink → skills/council/SKILL.md)                      │
│ Loads SKILL.md body as the moderator's prompt                │
│ Passes "Should we migrate..." as $ARGUMENTS                  │
├─────────────────────────────────────────────────────────────┤
│ SETUP PHASE                                                  │
│                                                              │
│ Moderator reads references/workflow.md                       │
│ Moderator globs perspectives/*.md → finds 3 perspectives     │
│ Moderator frames proposition, proposes defaults               │
│ User confirms: 2 rounds, all 3 perspectives                  │
│ Moderator creates docs/council/active/<id>/council.yaml      │
├─────────────────────────────────────────────────────────────┤
│ ROUND 1                                                      │
│                                                              │
│ For each perspective (in parallel):                           │
│   Moderator reads perspectives/<id>.md                       │
│   Moderator constructs task prompt from workflow template     │
│   Moderator spawns background subagent with this prompt      │
│                                                              │
│ Moderator collects all 3 results                             │
│ Moderator writes round-1-security.md, etc.                   │
│ Moderator updates council.yaml                               │
├─────────────────────────────────────────────────────────────┤
│ GATE 1                                                       │
│                                                              │
│ Moderator presents 3–5 sentence summary per perspective      │
│ User confirms → proceed to round 2                           │
├─────────────────────────────────────────────────────────────┤
│ ROUND 2 (same as round 1, but task prompt includes           │
│          prior round files as PRIOR ARGUMENTS)               │
├─────────────────────────────────────────────────────────────┤
│ GATE 2 → User confirms → proceed to synthesis                │
├─────────────────────────────────────────────────────────────┤
│ SYNTHESIS                                                    │
│                                                              │
│ Moderator reads references/summariser-prompt.md              │
│ Moderator reads references/synthesis-format.md               │
│ Moderator constructs synthesis task prompt                    │
│ Moderator spawns subagent (foreground, waits for result)     │
│ Moderator writes synthesis.md                                │
│ Moderator archives council                                   │
└─────────────────────────────────────────────────────────────┘
```

The entire flow is platform-agnostic. Each platform uses its own native mechanism for "spawn a background subagent" and "read a file," but the moderator prompt never references those mechanisms by name.

---

## Adding a New Perspective

### Step 1: Create `skills/council/perspectives/cost.md`

```markdown
# Cost & Financial Perspective

You are the Cost and Financial Perspective in a decision council.
Your role is to argue from a financial and resource allocation
perspective.

## Your Perspective
[... follows standard structure ...]
```

### Step 2: There is no step 2.

The moderator discovers perspectives via glob. The new file is available on all platforms immediately (via the symlink).

---

## Platform-Specific Considerations

### Argument Passing

| Platform | Skill argument mechanism | `$ARGUMENTS` supported? |
|----------|-------------------------|-------------------------|
| Claude Code | `$ARGUMENTS` placeholder in SKILL.md | Yes (native) |
| OpenCode | `$ARGUMENTS` placeholder | Yes (native) |
| Cursor | Arguments passed to skill context | Yes (via compat layer) |

### Subagent Spawning

| Platform | What "spawn a background subagent" translates to |
|----------|--------------------------------------------------|
| Claude Code | `Agent(subagent_type="general-purpose", run_in_background=true)` |
| OpenCode | `Task(run_in_background=true)` via built-in task tool |
| Cursor | Native parallel subagent spawning |

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

You can **override** the shared skill for a specific platform by placing a modified copy in the platform-specific directory, whilst the `skills/council/` version (via `.claude/skills/` symlink) serves as the default.

### Model and Temperature

The Agent Skills standard does not include `temperature` or `model` fields. All perspectives run at the platform's default temperature and model. Perspective differentiation comes from the prompt content, not sampling parameters.

Claude Code's extended fields (`allowed-tools`, `argument-hint`, `context`, `model`) are silently ignored by OpenCode and Cursor per the spec's forward-compatibility rule.
