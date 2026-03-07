# Council Workflow Protocol

## Phases

A council progresses through 5 phases:

1. **Setup** — Frame proposition, confirm perspectives and rounds
2. **Debate Rounds** — Parallel perspective invocation per round
3. **Gate Checkpoints** — User reviews summaries between rounds
4. **Synthesis** — Summariser produces structured output
5. **Archive** — Move completed council to archive

## Rules

- **Neutral moderator.** Never advocate for any position.
- **User gates.** Always gate round transitions through user confirmation.
- **One question per message.** Propose sensible defaults, don't ask open-ended questions.
- **Sensible defaults.** 2 rounds, open visibility.
- **Manifest is truth.** `council.yaml` is the single source of truth. Read and update at every transition.
- **Parallel execution.** Invoke all perspectives simultaneously within each round. Rounds remain sequential with gates between them.

## File Conventions

### Council ID Format
`council-YYYY-MM-DD-<slugified-topic>`

### Directory Structure
```
docs/council/
├── active/
│   └── <council-id>/
│       ├── council.yaml
│       ├── topic.md
│       ├── round-1-<perspective>.md
│       ├── round-2-<perspective>.md
│       └── synthesis.md
└── archive/
    └── <council-id>/
```

### Round File Naming
`round-<N>-<perspective-id>.md`

### Council Manifest Structure

```yaml
id: council-2026-02-08-example-topic
created: 2026-02-08T14:30:00Z
topic_summary: "Should we adopt X?"
status: setup | round-1 | gate-1 | round-2 | gate-2 | synthesis | complete | archived
total_rounds: 2
current_round: 1
visibility: open
perspectives:
  - id: security
    rounds:
      1: { status: complete, file: round-1-security.md }
      2: { status: pending, file: round-2-security.md }
  - id: velocity
    rounds:
      1: { status: complete, file: round-1-velocity.md }
      2: { status: pending, file: round-2-velocity.md }
```

### Round File Frontmatter

```yaml
---
perspective: security
round: 1
timestamp: 2026-02-08T14:35:00Z
council_id: council-2026-02-08-example-topic
responding_to: []
word_count: 650
---
```

For round 2+, the `responding_to` field lists prior round files shown:
```yaml
responding_to:
  - round-1-velocity.md
  - round-1-maintainability.md
```

## Task Prompt Templates

### Round 1

```
You are participating in a structured council debate.

PROPOSITION:
[contents of topic.md]

YOUR ROLE:
[contents of the perspective file]

INSTRUCTIONS:
- Present your strongest arguments regarding the proposition from your perspective.
- Be specific and concrete. Reference relevant standards, patterns, or evidence.
- Structure your response with clear sections.
- Do not attempt to be balanced. Your job is to make the strongest case from your perspective.
- Keep your response to approximately 500-800 words.
```

### Round 2+

```
You are participating in round [N] of a structured council debate.

PROPOSITION:
[contents of topic.md]

YOUR ROLE:
[contents of the perspective file]

PRIOR ARGUMENTS:
[contents of round-(N-1)-*.md files from other perspectives]

INSTRUCTIONS:
- Respond to the arguments made by other perspectives in the prior round.
- Identify where you agree, where you disagree, and where you see false dichotomies.
- Strengthen or refine your position based on the discussion.
- You may concede points where the evidence warrants it.
- Keep your response to approximately 500-800 words.
```

## Error Handling

- **Subagent failure:** Inform user, offer to retry or skip that perspective.
- **Manifest corruption:** Report error, ask user to inspect file.
- **Missing perspectives:** If fewer than 2 found, inform user and halt.
- **Interrupted session:** Manifest and completed round files remain. User must start a new council.

## Archive

After synthesis, move the council directory:
```
docs/council/active/<council-id> → docs/council/archive/<council-id>
```
