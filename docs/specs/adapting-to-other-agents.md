# Adapting the Council Pattern to Other Coding Agents

This document analyzes how the OpenCode Council's multi-perspective debate orchestration can be ported to Claude Code, Cursor, and Codex CLI. It maps each architectural component to its equivalent mechanism in each target agent, identifies gaps, and provides concrete implementation guidance.

---

## Table of Contents

1. [Architecture Summary](#architecture-summary)
2. [Component Mapping Matrix](#component-mapping-matrix)
3. [Claude Code Adaptation](#claude-code-adaptation)
4. [Cursor Adaptation](#cursor-adaptation)
5. [Codex CLI Adaptation](#codex-cli-adaptation)
6. [Cross-Platform Concerns](#cross-platform-concerns)
7. [Recommendation](#recommendation)

---

## Architecture Summary

The OpenCode Council has five architectural components that must be ported:

| Component | OpenCode Mechanism | Purpose |
|-----------|-------------------|---------|
| **Moderator** | `.opencode/agents/council-moderator.md` (primary agent) | Orchestrates 5-phase workflow, spawns advocates, gates transitions |
| **Advocates** | `.opencode/agents/advocate-*.md` (subagents) | Argue from one perspective per round (500-800 words) |
| **Summariser** | `.opencode/agents/council-summariser.md` (hidden subagent) | Produces 6-section synthesis from debate rounds |
| **Command** | `.opencode/commands/council.md` | Entry point: `/council <topic>` |
| **State** | `docs/council/active/<id>/council.yaml` + round files | Plain-text manifest and debate artifacts |

The workflow is: **Setup → Parallel Debate Rounds → User Gates → Synthesis → Archive**. Within each round all advocates execute simultaneously; rounds are sequential with user checkpoints between them.

---

## Component Mapping Matrix

| OpenCode Component | Claude Code | Cursor | Codex CLI |
|-------------------|-------------|--------|-----------|
| Primary agent (moderator) | Skill with `context: fork` | Custom mode in `.cursor/modes.json` or orchestrating command | Skill (`SKILL.md`) or Agents SDK orchestrator |
| Subagents (advocates) | Task tool subagents or Agent Teams | `.cursor/agents/advocate-*.md` subagents (v2.4+) | `codex exec` per perspective or MCP + Agents SDK |
| Hidden subagent (summariser) | Task tool subagent (not user-invocable) | Subagent with no slash command | `codex exec` call or Agents SDK agent |
| Slash command (`/council`) | `.claude/skills/council/SKILL.md` | `.cursor/commands/council.md` | `.codex/skills/council/SKILL.md` |
| Parallel execution | Task tool `run_in_background=true` (up to 7) | Native parallel subagents (up to 8, via git worktrees) | `codex exec &` (shell parallelism) or Agents SDK |
| Agent temperature | Not directly configurable per subagent | Model selection per subagent | Model/profile selection per `codex exec` call |
| Tool permissions | Skill `allowed-tools` frontmatter + hooks | Subagent tool scoping + hooks | Sandbox modes + `approval_policy` per profile |
| State persistence | File I/O to `docs/council/` | File I/O to `docs/council/` | File I/O to `docs/council/` |
| User gates | Conversation-driven (ask and wait) | Conversation-driven (ask and wait) | Interactive mode or scripted checkpoints |
| MCP integration (future) | Native MCP support | `.cursor/mcp.json` | `[mcp_servers]` in `config.toml` |

---

## Claude Code Adaptation

### Maturity: High

Claude Code has the strongest alignment with the OpenCode Council architecture. Its subagent system, skills framework, hooks, and experimental Agent Teams feature provide near-parity with every component.

### Directory Structure

```
.claude/
├── skills/
│   └── council/
│       └── SKILL.md              # /council command (moderator prompt)
├── agents/
│   ├── advocate-security.md      # Security subagent
│   ├── advocate-velocity.md      # Velocity subagent
│   ├── advocate-maintainability.md  # Maintainability subagent
│   └── council-summariser.md     # Synthesis subagent
├── rules/
│   └── council-protocol.md       # Council file conventions and workflow rules
├── settings.json                 # Hooks configuration
└── .mcp.json                     # MCP servers (optional, for external data)
```

### Component Implementation

#### 1. Slash Command → Skill

Create `.claude/skills/council/SKILL.md`:

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

You are the Council Moderator. You orchestrate structured multi-perspective
debates to support human decision-making.

## CRITICAL RULES

- You are neutral. You never advocate for any position.
- You always gate round transitions through user confirmation.
- You ask at most one question per message during setup.
- You propose sensible defaults (2 rounds, open visibility).
- All council files go to `docs/council/active/<council-id>/`.
- The manifest (`council.yaml`) is the single source of truth.
- You present brief round summaries (3-5 sentences per perspective).
- You invoke advocates in parallel per round using the Task tool
  with `run_in_background=true`, then collect results.

## WORKFLOW

[... remainder of council-moderator.md prompt ...]

The user's topic is:

$ARGUMENTS

Start the setup phase.
```

Key differences from OpenCode:
- Uses `allowed-tools` frontmatter instead of YAML `tools` block
- `$ARGUMENTS` placeholder works identically
- `description` field enables Claude to auto-invoke when appropriate

#### 2. Advocates → Custom Subagents

Create `.claude/agents/advocate-security.md`:

```yaml
---
name: advocate-security
description: >
  Security and compliance advocate that evaluates decisions from
  a risk-aware, defensive perspective. Council debate subagent.
tools: Read, Glob, Grep, Bash(cat *), Bash(ls *), Bash(grep *), Bash(find *)
user-invocable: false
---

# Security & Compliance Advocate

You are the Security and Compliance Advocate in a decision council.
[... remainder of advocate-security.md prompt ...]
```

Key differences:
- `user-invocable: false` replaces `hidden: true` (prevents users from invoking directly)
- Tool restrictions use the `tools` frontmatter field with Bash command patterns
- Temperature is **not directly configurable** per subagent in Claude Code — the model's behavior is shaped entirely by the system prompt. This is a minor gap; in practice, well-written prompts produce the same effect as temperature tuning.

#### 3. Parallel Execution → Task Tool

The moderator invokes advocates using Claude Code's Task tool, which supports `run_in_background`:

```
For each perspective:
  1. Construct task prompt (proposition + prior rounds)
  2. Invoke via Task tool with run_in_background=true
  3. Store the returned agent ID
  4. After all launched, read each agent's output file
```

Claude Code supports up to 7 concurrent subagents, sufficient for councils with 5-7 perspectives.

**Important difference:** In OpenCode, `background_output(task_id)` retrieves results synchronously. In Claude Code, background tasks write to an output file that the moderator reads with the Read tool or Bash `tail`. The moderator must poll or wait for completion.

#### 4. User Gates → Conversational

Identical pattern. The moderator presents summaries and asks the user to confirm before proceeding. Claude Code's interactive conversation model supports this naturally.

#### 5. State Persistence → Identical

Council artifacts (`council.yaml`, round files, `synthesis.md`) are written to `docs/council/` using Write/Edit tools. No changes needed.

#### 6. Alternative: Agent Teams (Experimental)

Claude Code's experimental Agent Teams feature (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) offers a higher-level approach:

```
Create an agent team to debate this decision. Spawn teammates:
- Security Advocate: argues from risk/compliance perspective
- Velocity Advocate: argues from speed/pragmatism perspective
- Maintainability Advocate: argues from code health perspective

Have each present their case on: [proposition]
Then have them respond to each other's arguments.
Finally, synthesize consensus, tensions, and recommendations.
```

**Trade-offs vs. subagent approach:**

| Aspect | Subagents (Task tool) | Agent Teams |
|--------|----------------------|-------------|
| Control | Full programmatic control over rounds, gates, file output | Less structured; team coordinates via task lists |
| Token cost | Lower (subagents return results to moderator) | Higher (each teammate has full context) |
| Parallelism | Up to 7 concurrent | Up to ~5 concurrent (display modes vary) |
| Debate protocol | Moderator enforces round structure | Teammates self-organize (less predictable) |
| User gates | Moderator explicitly asks at each gate | Must be prompted to checkpoint |

**Recommendation:** Use the subagent approach for production councils. Agent Teams work well for ad-hoc debates where strict round structure is less important.

### Gaps and Workarounds

| Gap | Impact | Workaround |
|-----|--------|------------|
| No per-subagent temperature | Low | Prompt engineering achieves equivalent effect |
| No `mode: primary` equivalent | Low | Skill with `context: fork` provides isolated session |
| Background task completion polling | Medium | Read output file; moderator prompt instructs polling pattern |
| No YAML permission syntax | Low | `tools` frontmatter + Bash command patterns provide equivalent restrictions |

---

## Cursor Adaptation

### Maturity: High

Cursor 2.4+ has native subagent support, custom commands, rules, hooks, and MCP — covering all council components. Its parallel agent execution (up to 8 via git worktrees) is the most performant option for debate rounds.

### Directory Structure

```
.cursor/
├── commands/
│   └── council.md                # /council slash command
├── agents/
│   ├── advocate-security.md      # Security subagent
│   ├── advocate-velocity.md      # Velocity subagent
│   ├── advocate-maintainability.md
│   └── council-summariser.md     # Synthesis subagent
├── rules/
│   └── council-protocol.mdc     # Council workflow rules
├── modes.json                    # Custom agent modes (optional)
├── hooks.json                    # Lifecycle hooks
└── mcp.json                      # MCP servers (optional)
```

### Component Implementation

#### 1. Slash Command → `.cursor/commands/council.md`

```markdown
You are the Council Moderator. You orchestrate structured multi-perspective
debates to support human decision-making.

## CRITICAL RULES
[... moderator instructions ...]

The user's topic is: {{input}}

Start the setup phase. Frame the proposition, identify available advocate
agents in `.cursor/agents/advocate-*.md`, and confirm debate parameters.
```

Cursor commands use `{{input}}` for user arguments (vs. OpenCode's `$ARGUMENTS`).

#### 2. Advocates → `.cursor/agents/*.md`

```yaml
---
name: advocate-security
description: Security and compliance advocate for council debates
model: claude-sonnet-4-20250514
tools: code_search, git_history
readonly: true
---

# Security & Compliance Advocate

You are the Security and Compliance Advocate in a decision council.
[... advocate prompt ...]
```

Key differences from OpenCode:
- **Model selection per subagent** — Cursor allows specifying different models per advocate. You could use a reasoning-heavy model (e.g., Claude Opus) for the security advocate and a faster model for velocity.
- **`readonly: true`** replaces the `write: false, edit: false` pattern.
- **Tool scoping** uses tool names from Cursor's tool registry rather than bash command patterns.

#### 3. Parallel Execution → Native Subagents

Cursor 2.4 subagents run in parallel natively with isolated contexts. The moderator (running as the main agent in Composer) spawns subagents and collects their outputs.

Each parallel agent can operate in its own git worktree, providing true filesystem isolation. This prevents the file-conflict concern that OpenCode addresses with unique round file names — though the same naming convention should still be used for organizational clarity.

#### 4. Rules for Council Protocol → `.cursor/rules/council-protocol.mdc`

```yaml
---
description: "Council debate orchestration protocol and file conventions"
alwaysApply: false
---

# Council Protocol

When orchestrating a council debate, follow these conventions:
- Council ID format: `council-YYYY-MM-DD-<slug>`
- Round files: `round-<N>-<perspective>.md` with YAML frontmatter
- Manifest: `council.yaml` is the single source of truth
- Synthesis: 6-section format (Consensus, Key Tensions, Risk Assessment,
  Recommended Path Forward, Minority Report, Suggested Actions)
```

Setting `alwaysApply: false` means the rule activates only when the AI determines it's relevant (i.e., during council debates), saving tokens during normal coding work.

#### 5. Hooks for Validation → `.cursor/hooks.json`

```json
{
  "afterFileEdit": [
    {
      "pattern": "docs/council/**/council.yaml",
      "command": "python scripts/validate-manifest.py $FILE"
    }
  ]
}
```

Cursor hooks provide lifecycle validation that OpenCode handles through moderator prompt instructions alone.

### Gaps and Workarounds

| Gap | Impact | Workaround |
|-----|--------|------------|
| Context bleed (~12% of cases) | Medium | Reinforce perspective identity in subagent prompts; validate outputs |
| No native debate protocol | Medium | Moderator prompt orchestrates rounds; no framework-level support |
| Token cost of parallel subagents | Medium | Each subagent = separate context window; 3-4x vs. sequential |
| Documentation gaps on subagents | Low | Configuration evolving; refer to community examples |
| `{{input}}` vs `$ARGUMENTS` | Trivial | Syntax difference only |

### Unique Advantages

- **Model mixing**: Use reasoning-heavy models for complex perspectives (security) and fast models for straightforward ones (velocity).
- **Background agents**: Prefix with `&` to run entire councils as background cloud agents, reviewable later at `cursor.com/agents`.
- **Browser integration**: Advocates could test proposals in a browser during debate (unique to Cursor).
- **Plan Mode**: Use a reasoning model for the moderator's synthesis planning, then a fast model for execution.

---

## Codex CLI Adaptation

### Maturity: Medium

Codex CLI has the weakest native multi-agent support of the three targets. It lacks built-in subagent spawning (as of February 2026), but its `codex exec` non-interactive mode, MCP server capability, and OpenAI Agents SDK integration provide viable — if more complex — paths.

### Three Implementation Approaches

#### Approach A: Shell-Scripted (Simplest)

Use `codex exec` to run each advocate as an independent non-interactive session:

```bash
#!/bin/bash
# council.sh — Shell-orchestrated decision council

TOPIC="$1"
COUNCIL_ID="council-$(date +%Y-%m-%d)-$(echo "$TOPIC" | tr ' ' '-' | tr '[:upper:]' '[:lower:]' | head -c 40)"
DIR="docs/council/active/$COUNCIL_ID"
mkdir -p "$DIR"

echo "$TOPIC" > "$DIR/topic.md"

# Round 1: All advocates in parallel
codex exec --full-auto --profile security \
  "Argue from security perspective on: $TOPIC. 500-800 words." \
  > "$DIR/round-1-security.md" &

codex exec --full-auto --profile velocity \
  "Argue from velocity/speed perspective on: $TOPIC. 500-800 words." \
  > "$DIR/round-1-velocity.md" &

codex exec --full-auto --profile maintainability \
  "Argue from maintainability perspective on: $TOPIC. 500-800 words." \
  > "$DIR/round-1-maintainability.md" &

wait

# User gate
echo "Round 1 complete. Review files in $DIR/round-1-*.md"
echo "Proceed to round 2? [y/n]"
read -r proceed
[ "$proceed" != "y" ] && exit 0

# Round 2: Advocates respond to each other
PRIOR=$(cat "$DIR"/round-1-*.md)

codex exec --full-auto --profile security \
  "Respond to these arguments from security perspective: $PRIOR" \
  > "$DIR/round-2-security.md" &
# ... (repeat for each advocate)
wait

# Synthesis
codex exec --full-auto \
  "Synthesize this council debate into 6 sections (Consensus, Key Tensions, \
   Risk Assessment, Recommended Path, Minority Report, Suggested Actions): \
   $(cat "$DIR"/round-*.md)" \
  > "$DIR/synthesis.md"

mv "$DIR" "docs/council/archive/$COUNCIL_ID"
echo "Council archived at docs/council/archive/$COUNCIL_ID/"
```

Profiles in `~/.codex/config.toml` provide per-advocate configuration:

```toml
[profiles.security]
model = "gpt-5-codex"
instructions = "You are a security and compliance advocate. Argue from a risk-aware, defensive perspective."

[profiles.velocity]
model = "gpt-5-codex"
instructions = "You are a velocity advocate. Argue for speed, pragmatism, and rapid iteration."

[profiles.maintainability]
model = "gpt-5-codex"
instructions = "You are a maintainability advocate. Argue for long-term code health and sustainability."
```

**Trade-offs:**
- Simple to implement and understand
- No interactive user gates (must be scripted externally)
- Each `codex exec` call is an independent session (no shared context)
- Round 2 prompts must include all prior round text (token-heavy)

#### Approach B: Skill-Based (Within Codex CLI)

Create `.codex/skills/council/SKILL.md`:

```yaml
---
name: council
description: >
  Run a structured multi-perspective debate on a technical decision.
---

You are the Council Moderator. Orchestrate a structured debate.

## WORKFLOW

1. Frame the proposition from the user's topic
2. For each round, run each perspective by asking the user to invoke
   `codex exec --profile <perspective>` for each advocate
3. Collect results and present summaries
4. After user confirms, produce synthesis

Topic: $ARGUMENTS
```

**Limitation:** Without native subagent support, the skill cannot spawn advocate sessions programmatically. The moderator must instruct the user to run separate `codex exec` calls or use shell scripting.

#### Approach C: MCP + Agents SDK (Full Multi-Agent)

This is the officially recommended approach for multi-agent Codex workflows.

**Architecture:**

```
┌─────────────────────────────┐
│  Python Orchestrator        │
│  (OpenAI Agents SDK)        │
│                             │
│  ┌──────────┐               │
│  │Moderator │               │
│  │  Agent   │               │
│  └────┬─────┘               │
│       │ hand-off             │
│  ┌────┴────┬────────┐       │
│  │Security │Velocity│Maint. │  ← Each calls codex MCP tool
│  │Advocate │Advocate│Advoc. │
│  └────┬────┴────┬───┴──┐   │
│       │         │       │   │
│  ┌────┴─────────┴───────┴┐  │
│  │   Summariser Agent    │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
         │
         ▼
   codex mcp (stdio)
```

```python
# orchestrator.py (simplified)
from openai_agents import Agent, Runner, handoff

moderator = Agent(
    name="Council Moderator",
    instructions="You orchestrate structured debates...",
    handoffs=[security_advocate, velocity_advocate, maintainability_advocate],
)

security_advocate = Agent(
    name="Security Advocate",
    instructions="You are the Security advocate...",
    tools=[codex_mcp_tool],  # Calls codex MCP server
)

# ... define other advocates and summariser

runner = Runner(agent=moderator, context={"topic": topic})
result = await runner.run()
```

**Trade-offs:**
- Full multi-agent orchestration with tracing and auditability
- Requires Python environment and Agents SDK dependency
- Most complex setup of the three approaches
- Best suited for teams already using the OpenAI ecosystem

### Gaps and Workarounds

| Gap | Impact | Workaround |
|-----|--------|------------|
| No native subagent spawning | High | Use `codex exec` (shell), MCP + Agents SDK (Python), or wait for native support |
| No interactive gates in `codex exec` | Medium | Script `read` prompts between rounds, or use interactive Codex session |
| No per-subagent temperature | Low | Profiles provide per-perspective model and instruction config |
| No background task collection | Medium | Shell `wait` + file output, or Agents SDK async patterns |
| Heavier orchestration burden | Medium | Shell script or Agents SDK handles what the moderator prompt does in OpenCode/Claude Code |

### Unique Advantages

- **Sandboxed execution**: Codex's Landlock/Seatbelt sandbox provides stronger isolation than prompt-based tool restrictions. Advocates genuinely cannot write files if sandbox is `read-only`.
- **Structured output**: `codex exec --output-schema schema.json` can enforce JSON structure on advocate responses, enabling programmatic processing.
- **CI/CD integration**: `codex exec` runs headlessly, making councils automatable in CI pipelines via GitHub Actions (`openai/codex-action@v1`).
- **Traces**: The Agents SDK produces full traces of every prompt, tool call, and hand-off — valuable for auditing council decisions.

---

## Cross-Platform Concerns

### 1. The Core Abstraction Is Portable

The council pattern's essential elements — a moderator prompt, perspective-specific subagent prompts, round-based debate structure, and a synthesis template — are plain text. The markdown prompts in `.opencode/agents/` can be reused with minimal syntactic changes across all three targets:

| Prompt Content | Changes Needed |
|---------------|----------------|
| Moderator instructions | Adjust tool invocation syntax (Task tool vs. `codex exec` vs. Cursor subagent API) |
| Advocate perspective prompts | Reusable as-is; only frontmatter format changes |
| Synthesis template | Reusable as-is |
| Round file format | Reusable as-is |
| Council manifest (`council.yaml`) | Reusable as-is |

### 2. Parallel Execution Model Varies

| Agent | Mechanism | Max Concurrent | Isolation |
|-------|-----------|---------------|-----------|
| OpenCode | `run_in_background=true` on Task tool | Framework-dependent | Separate context |
| Claude Code | Task tool with `run_in_background=true` | 7 | Separate context + output file |
| Cursor | Native parallel subagents | 8 | Git worktrees |
| Codex CLI | Shell `&` with `codex exec` | OS-limited | Separate processes |

### 3. Temperature vs. Prompt Engineering

OpenCode uses explicit `temperature` YAML fields (0.3 for security, 0.5 for velocity). Claude Code and Cursor don't expose per-subagent temperature controls. However, temperature primarily affects response randomness — the perspective differentiation in practice comes from the prompt itself (the advocate's values, argumentation style, and focus areas). Well-written prompts produce consistent perspective differentiation regardless of temperature settings.

### 4. User Gate Pattern

All three targets support interactive conversation, making user gates (the moderator asks "Proceed to round 2?" and waits for confirmation) portable without changes. The exception is Codex CLI's `codex exec` mode, which is non-interactive; gates must be implemented as shell script `read` prompts between invocations.

### 5. State Persistence

All targets have file I/O capabilities sufficient for the council's plain-text state model (`council.yaml` + round files + `synthesis.md`). No changes needed to the persistence layer.

### 6. Custom Advocate Extensibility

The "add a new perspective by creating `advocate-<name>.md`" pattern works across all targets:

| Agent | New Advocate Location | Discovery |
|-------|----------------------|-----------|
| OpenCode | `.opencode/agents/advocate-<name>.md` | Moderator globs `advocate-*.md` |
| Claude Code | `.claude/agents/advocate-<name>.md` | Moderator globs `advocate-*.md` |
| Cursor | `.cursor/agents/advocate-<name>.md` | Moderator globs `advocate-*.md` |
| Codex CLI | Profile in `config.toml` + optional `.codex/skills/advocate-<name>/SKILL.md` | Moderator reads config or skill directory |

---

## Recommendation

### Best Fit: Claude Code

Claude Code requires the fewest changes. Its subagent system (Task tool with `run_in_background`), skills framework (`.claude/skills/`), and file I/O model are near-identical to OpenCode's architecture. The moderator prompt, advocate prompts, and synthesis template can be ported with frontmatter syntax changes only. The experimental Agent Teams feature provides an additional high-level option for less structured debates.

### Strong Fit: Cursor

Cursor 2.4+ has native parallel subagents and custom commands that map directly to the council architecture. Its model-mixing capability (different models per advocate) and background agent execution are genuine advantages over the OpenCode implementation. The main concerns are subagent context bleed (~12%) and higher token costs from parallel execution.

### Viable with More Work: Codex CLI

Codex CLI lacks native subagent spawning, making the port more complex. The shell-scripted approach (`codex exec`) is functional but loses the conversational moderator experience. The MCP + Agents SDK approach provides full multi-agent orchestration but requires a Python environment and additional infrastructure. Codex CLI is best suited for teams that want automated, CI-integrated council execution rather than interactive debate.

### Migration Priority

1. **Claude Code** — Port prompts with frontmatter changes; test immediately
2. **Cursor** — Port prompts and commands; leverage model mixing and background execution
3. **Codex CLI** — Start with shell-scripted approach; graduate to Agents SDK if multi-agent orchestration is needed at scale
