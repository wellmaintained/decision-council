# Council Workflow Specification

**Why this matters:** Understanding the workflow helps you use councils effectively and know when to intervene.

---

## Overview

A council progresses through 5 phases: Setup → Debate → Gate → Synthesis → Archive.

The key insight: **Rounds are sequential (with user gates), but perspectives within each round are parallel (for speed).**

---

## Phase 1: Setup (Interactive)

### Why It Exists
Clear problem framing is critical. A poorly framed proposition leads to scattered debate. The setup phase ensures everyone (you, moderator, perspectives) agrees on what's being debated.

### What Happens

1. **You invoke:** `/council <your topic>`
2. **Moderator proposes framing:** "Should we migrate authentication to OAuth 2.0 to improve security posture and reduce maintenance burden?"
3. **You confirm or refine:** If the framing misses the point, redirect it
4. **Moderator identifies perspectives:** Lists available perspectives (security, velocity, maintainability, etc.)
5. **You confirm perspectives:** Remove perspectives that don't apply, or note that you'll add custom ones
6. **Moderator confirms rounds:** Default is 2 (initial positions + responses), adjustable
7. **Council created:** Manifest written, directory structure created, debate begins

### User Control Points
- Refine the proposition framing
- Add/remove perspectives
- Adjust number of rounds (1-3 typical)

### Duration
2-4 exchanges, ~1 minute.

---

## Phase 2: Debate Rounds

### Why Multiple Rounds
Round 1 captures initial positions. Round 2+ allows perspectives to:
- Respond to each other's arguments
- Refine their positions based on new information
- Concede valid points
- Identify false dichotomies

This mimics real deliberation where hearing other viewpoints sharpens your own thinking.

### What Happens (Per Round)

#### 1. Launch (Parallel)
- Moderator constructs task prompt for each perspective
- **All perspectives invoked simultaneously** via `run_in_background=true`
- Manifest updated: each perspective marked `in-progress` with `task_id`

**Why parallel:** 3 perspectives complete in ~1 minute instead of ~3 minutes sequential. You don't wait for perspective A before perspective B starts thinking.

#### 2. Collection
- Moderator calls `background_output(task_id)` for each perspective
- Responses written to `round-<N>-<perspective>.md` files
- Manifest updated: each perspective marked `complete`, task_id removed

#### 3. Gate Checkpoint
- Moderator reads all contributions for this round
- Presents brief summaries (3-5 sentences per perspective)
- Asks: "Proceed to round [N+1]?" or "Proceed to synthesis?"

### Round Inputs

**Round 1:**
- Proposition only
- Perspectives argue from their core viewpoint without external context

**Round 2+:**
- Proposition
- The perspective's own prior round(s)
- All other perspectives' prior rounds

This visibility model ensures perspectives can respond to each other while maintaining their core viewpoint.

### User Control Points
- Review summaries at each gate
- Decide to continue, redirect, or conclude early
- Add a follow-up question for the next round (e.g., "Focus on operational costs in round 2")

### Duration (Per Round)
- Launch + collection: ~1 minute for 3 perspectives
- Gate review: ~2 minutes for you to read summaries and decide

---

## Phase 3: Gate Checkpoints

### Why Gates Exist
Gates give you control. Without them, you'd wait 6 minutes for all rounds to complete, then discover the debate went off-track in round 1.

Gates let you:
- **Steer:** "The security perspective is too abstract. Focus on specific attack vectors in round 2."
- **Conclude early:** "Clear consensus emerged. Skip round 2, go to synthesis."
- **Redirect:** "The velocity perspective missed the operational cost concern. Retry round 1 for that perspective."

### What the Moderator Presents

#### Per-Perspective Summaries
3-5 sentences capturing:
- Core argument
- Key evidence or reasoning
- Stance on the proposition (support, oppose, conditional)

#### Cross-Perspective Dynamics (Round 2+)
- Where perspectives converged
- Where disagreements intensified
- New arguments that emerged

### Your Decision
- **Continue:** Proceed to next round as planned
- **Adjust:** Provide additional context or redirection for next round
- **Conclude:** Skip remaining rounds, go straight to synthesis

### Duration
1-3 minutes for you to read and decide.

---

## Phase 4: Synthesis

### Why Synthesis Matters
Raw debate rounds are useful, but synthesis extracts actionable insights. The summariser produces a structured document that:
- Highlights consensus (low-controversy decisions)
- Surfaces key tensions (tradeoffs you must consciously choose)
- Maps risks to perspectives (who's worried about what)
- Recommends a path forward (best risk/reward, not hollow compromise)
- Preserves minority positions (dissent that deserves consideration)

### What Happens

1. **Moderator invokes summariser:** Passes all round files + topic
2. **Summariser reads and extracts:** Not inventing arguments, just organizing what was said
3. **Moderator writes synthesis.md:** 6-section document (see [Synthesis Format](synthesis.md))
4. **Manifest updated:** Status set to `complete`

### User Control Points
None during synthesis itself, but:
- At prior gate, you decided whether sufficient debate occurred
- After synthesis, you decide whether to accept recommendation or conduct another council

### Duration
~30-60 seconds for synthesis generation.

---

## Phase 5: Archive

### Why Archive
Completed councils should move out of `active/` so:
- New councils have clean workspace
- Historical debates are preserved for reference
- You can compare councils on related topics over time

### What Happens

1. **Moderator moves directory:** `active/<council-id>` → `archive/<council-id>`
2. **Confirmation to user:** "Council archived at `docs/council/archive/<council-id>/`"

### What's Preserved
- `council.yaml` (manifest)
- `topic.md` (proposition)
- All round files (`round-1-security.md`, etc.)
- `synthesis.md` (final output)

### Duration
Instant.

---

## Error Handling

### Task Launch Failures
If a perspective fails to start (bad agent config, OpenCode error):
- Moderator reports which perspective failed
- Offers: retry, skip that perspective, cancel entire round

### Task Execution Failures
If `background_output` fails for a perspective (timeout, error in agent prompt):
- Moderator reports which perspective failed and error message
- Offers: retry just that perspective, continue without it, cancel round

### Partial Round Completion
If 2 of 3 perspectives succeed:
- Moderator asks whether to proceed with partial results or retry failures
- You decide whether missing perspective is critical

### Mid-Council Interruption
If you or the system interrupts before completion:
- Manifest and any completed round files remain in `active/`
- No automatic resume (limitation of current version)
- You can manually inspect files or start a fresh council

---

## Workflow Variants

### Single-Round Council
Useful for quick opinion gathering without cross-pollination:
- Setup: specify 1 round
- Round 1: All perspectives argue from initial positions
- Gate: Review, decide to synthesize
- Synthesis: Based on round 1 only

**Tradeoff:** Faster (~3 min total), but no opportunity for perspectives to respond to each other.

### Three-Round Council
Useful for complex decisions needing deep exploration:
- Round 1: Initial positions
- Round 2: Responses to each other
- Round 3: Convergence or final rebuttals

**Tradeoff:** More thorough (~6 min total), but diminishing returns if consensus emerges in round 2.

---

## Performance Characteristics

### Time Scaling

| Perspectives | Round Time (parallel) | 2-Round Council Total |
|-----------|----------------------|----------------------|
| 2 | ~1 min | ~3 min |
| 3 | ~1 min | ~3 min |
| 5 | ~1-2 min | ~4 min |

Time scales with **rounds**, not perspectives (thanks to parallel execution).

### Token Scaling

| Perspectives | Rounds | Approx. LLM Calls | Token Cost* |
|-----------|--------|-------------------|-------------|
| 3 | 2 | 7 | ~50K tokens |
| 5 | 2 | 11 | ~80K tokens |
| 3 | 3 | 10 | ~70K tokens |

*Token cost includes perspective responses + moderator gates + synthesis. Actual cost varies by response length.

Cost scales linearly with (perspectives × rounds).

---

## Best Practices

### When to Stop at Round 1
- All perspectives reached similar conclusions
- Proposition was simpler than expected
- You already got the insight you needed

### When to Add Round 3
- Round 2 introduced new arguments worth exploring
- Perspectives haven't converged and you need final positions
- Stakes are high and thoroughness matters

### When to Redirect Mid-Council
- A perspective misunderstood the proposition
- Important constraint wasn't mentioned in original framing
- Discussion drifted into irrelevant territory

### When to Cancel and Restart
- Proposition framing was fundamentally wrong
- You realized you need different perspectives
- External information invalidated the premise

---

## Next Steps

- **[Parallel Execution](parallel-execution.md):** Understand how perspectives run simultaneously
- **[Synthesis Format](synthesis.md):** Learn what the output document contains
- **[Extending](extending.md):** Add custom perspectives for your domain
