# Parallel Execution Specification

**Why this matters:** Parallel execution makes councils 60-70% faster without sacrificing correctness or user control.

---

## The Model

**Within each round:** All advocates execute simultaneously (parallel)  
**Between rounds:** Sequential with user gates (control)

This balances **speed** (don't wait for advocate A before B starts) with **control** (review round N before round N+1 begins).

---

## Why Parallel Within Rounds?

### The Sequential Problem

**Sequential execution** (old model):
```
Round 1:
  Security writes (1 min) → done
  Velocity writes (1 min) → done
  Maintainability writes (1 min) → done
Total: 3 minutes
```

You sit idle while each advocate finishes. Adding a 4th advocate adds another minute.

### The Parallel Solution

**Parallel execution** (current model):
```
Round 1:
  Security, Velocity, Maintainability all write simultaneously
Total: ~1 minute (longest advocate)
```

Adding a 4th advocate doesn't change duration—still ~1 minute.

### Performance Impact

| Advocates | Sequential | Parallel | Time Saved |
|-----------|------------|----------|------------|
| 2 | 2 min | 1 min | 50% |
| 3 | 3 min | 1 min | 67% |
| 5 | 5 min | 1-2 min | 60-80% |

**For a 2-round council with 3 advocates:**
- Sequential: ~6 minutes
- Parallel: ~2 minutes
- **4 minutes saved**

---

## Why Sequential Between Rounds?

### Round Dependencies

Round 2 advocates **need** to see round 1 results:
- Security's round 2 argument responds to velocity's round 1 concerns
- Velocity's round 2 refines its position based on maintainability's round 1 evidence

If all rounds launched simultaneously, advocates would argue in isolation—no cross-pollination, no refinement.

### User Control

Gates between rounds let you:
- **Detect drift early:** "Security is too abstract, refocus on practical risks"
- **Conclude early:** "Clear consensus in round 1, no need for round 2"
- **Redirect:** "Add cost considerations to round 2"

Without gates, you'd wait for all rounds to complete, then discover the debate went off-track.

---

## Technical Implementation

### 1. Launch Phase

**Moderator actions:**
```
For each perspective in current round:
  1. Construct task prompt (proposition + prior rounds if applicable)
  2. Invoke via Task tool with run_in_background=true
  3. Store task_id
  4. Update manifest: perspective marked "in-progress" with task_id
```

**Result:** All advocates start thinking simultaneously.

### 2. Collection Phase

**Moderator actions:**
```
For each task_id:
  1. Call background_output(task_id=...)
  2. Write response to round-N-perspective.md
  3. Update manifest: perspective marked "complete", task_id removed
```

**Result:** All responses collected before proceeding to gate.

### 3. Manifest Tracking

During parallel execution:
```yaml
perspectives:
  - id: security
    rounds:
      1: { status: in-progress, file: round-1-security.md, task_id: task_abc123 }
  - id: velocity
    rounds:
      1: { status: in-progress, file: round-1-velocity.md, task_id: task_def456 }
```

After collection:
```yaml
perspectives:
  - id: security
    rounds:
      1: { status: complete, file: round-1-security.md }
  - id: velocity
    rounds:
      1: { status: complete, file: round-1-velocity.md }
```

---

## Safety Guarantees

### No File Conflicts

**Problem:** What if two advocates try to write the same file?

**Solution:** Each advocate writes to a **unique file**: `round-<N>-<perspective>.md`
- Security writes: `round-1-security.md`
- Velocity writes: `round-1-velocity.md`

No possibility of conflict.

### No Manifest Conflicts

**Problem:** What if two advocates update the manifest simultaneously?

**Solution:** Manifest is updated **by the moderator**, not advocates:
1. Moderator launches all advocates (no manifest writes during execution)
2. Advocates execute (only write their round files)
3. Moderator collects results and updates manifest atomically

### Consistent Input

**Problem:** What if advocates see different prior round states?

**Solution:** All advocates receive **identical prior round files**:
- Round 1: All advocates see only the proposition (no prior rounds exist)
- Round 2: All advocates see **completed** round 1 files

No advocate sees another's in-progress round N response—only completed round N-1.

---

## Error Handling

### Launch Failures

**Scenario:** Task tool fails to invoke an advocate (bad config, permission error)

**Moderator response:**
```
"Failed to launch advocate-security for round 1: [error message]

Options:
1. Retry advocate-security
2. Skip security perspective for this round
3. Cancel entire round

What would you like to do?"
```

**Result:** Other advocates' tasks continue running. You decide per-failure.

### Execution Failures

**Scenario:** `background_output` returns error (advocate timeout, prompt error, model failure)

**Moderator response:**
```
"advocate-velocity failed during round 1 execution: [error message]

Options:
1. Retry advocate-velocity for round 1
2. Continue with security and maintainability only
3. Cancel entire round

What would you like to do?"
```

**Result:** Successful advocates' results are preserved. Granular recovery.

### Partial Completion

**Scenario:** 2 of 3 advocates succeed, 1 fails

**Moderator response:**
```
"Round 1 complete for security and maintainability.
advocate-velocity failed: [error message]

Continue with 2 perspectives, or retry velocity?"
```

**Your decision:** Is the missing perspective critical? If velocity's input is essential, retry. If not, proceed.

---

## Cost Implications

### Token Cost

**Parallel execution does NOT increase token cost:**
- Same number of advocates
- Same number of rounds
- Same amount of text generated

**What changes:** Cost accumulates **faster** (all advocates billing simultaneously).

### Cost Visibility

**Sequential:**
```
Advocate A: $0.10 → done
Advocate B: $0.10 → done
Advocate C: $0.10 → done
Total: $0.30 over 3 minutes
```

**Parallel:**
```
All advocates simultaneously: $0.30
Total: $0.30 in 1 minute
```

Same cost, faster accumulation. If you're monitoring spend in real-time, parallel execution shows larger spikes.

---

## Comparison: Parallel Per Round vs Fully Parallel

### Parallel Per Round (Current)

```
Setup → Round 1 (parallel advocates) → Gate 1 → Round 2 (parallel advocates) → Gate 2 → Synthesis
```

**Pros:**
- User gates between rounds for control
- Round 2 sees completed round 1 results
- Can redirect or conclude early

**Cons:**
- Slower than fully parallel (but only marginally)

### Fully Parallel (Rejected)

```
Setup → All Rounds (parallel advocates × rounds) → Single Gate → Synthesis
```

**Pros:**
- Fastest possible (all debate at once)

**Cons:**
- No mid-debate control
- Round 2 advocates must predict what round 2 will contain (can't see actual round 1)
- Can't conclude early if consensus emerges
- Harder to redirect if debate drifts

**Why we chose parallel per round:** Control is more valuable than marginal time savings.

---

## Performance Characteristics

### Best Case (All Advocates Similar Speed)

3 advocates, 2 rounds:
- Round 1: 1 minute (all finish ~same time)
- Gate 1: 2 minutes (you review)
- Round 2: 1 minute (all finish ~same time)
- Gate 2: 2 minutes (you review)
- Synthesis: 30 seconds

**Total: ~6.5 minutes** (vs ~12 minutes sequential)

### Worst Case (One Slow Advocate)

3 advocates, 2 rounds:
- Round 1: 2 minutes (one advocate takes longer)
- Gate 1: 2 minutes
- Round 2: 2 minutes (same advocate still slower)
- Gate 2: 2 minutes
- Synthesis: 30 seconds

**Total: ~8.5 minutes** (still better than 12 minutes sequential)

### Scaling

**Adding advocates:** Minimal impact on time (parallel execution)
**Adding rounds:** Linear impact on time (sequential rounds)

**Implication:** A 5-advocate, 2-round council takes roughly the same time as a 3-advocate, 2-round council.

---

## Future Enhancements

### Staggered Starts

**Idea:** Add small delays between advocate launches to avoid "thundering herd"

**Benefit:** Reduces instantaneous load on OpenCode task scheduler

**Cost:** Adds 5-10 seconds per round

### Timeout Detection

**Idea:** Detect advocates taking unusually long and offer to cancel/retry

**Benefit:** Don't wait 5 minutes for a stuck advocate

**Implementation:** Track task duration, prompt user if exceeds threshold

### Progress Indicators

**Idea:** Show which advocates are still in-progress during parallel execution

**Benefit:** Transparency ("Security and maintainability done, waiting on velocity")

**Implementation:** Poll task status, update user display

---

## Best Practices

### When Parallel Execution Matters Most

- **Many advocates:** 5+ perspectives benefit significantly from parallelism
- **Fast models:** If advocates finish in 30-60 seconds, parallel saves more time
- **Multiple rounds:** Savings compound across rounds

### When It Matters Less

- **Few advocates:** 2 advocates, parallel saves only ~1 minute
- **Slow models:** If advocates take 5+ minutes each, parallel vs sequential is less noticeable
- **Single round:** Only one opportunity for parallel execution

### Monitoring Execution

Watch for:
- **One advocate consistently slow:** Check agent prompt complexity
- **Frequent failures:** Check agent configurations and permissions
- **High variance:** Some advocates may need temperature adjustments

---

## Next Steps

- **[Workflow](workflow.md):** Understand the full 5-phase council lifecycle
- **[File Formats](file-formats.md):** Learn how task_id is tracked in manifests
- **[Extending](extending.md):** Add custom advocates that execute efficiently
