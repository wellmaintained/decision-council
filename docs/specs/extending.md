# Extending Decision Council

**Why this matters:** Council is designed to be extended. Custom perspectives let you adapt the system to your domain, organisation, and decision-making needs.

---

## Adding Custom Perspectives

### When to Add a New Perspective

**Good reasons:**
- Your decisions have a recurring dimension not covered by default perspectives
- Your organisation values a specific concern (cost, compliance, UX, scalability)
- You're applying councils to a non-engineering domain (hiring, marketing, procurement)

**Examples of useful custom perspectives:**
- **cost:** Financial implications, ROI, budget constraints
- **ux:** User experience, accessibility, product intuition
- **compliance:** Regulatory requirements, audit trails, data sovereignty
- **scalability:** Growth considerations, capacity planning
- **simplicity:** Reducing complexity, avoiding over-engineering

**Not useful:**
- Duplicates of existing perspectives (don't create "security-2")
- Perspectives that don't have distinct viewpoints ("common-sense")
- Joke perspectives that dilute serious debate

---

## Creating a Perspective

### Step 1: Copy a Template

```bash
cp skills/council/perspectives/security.md skills/council/perspectives/cost.md
```

### Step 2: Write the Perspective Prompt

Follow the same structure as existing perspectives:

```markdown
# Cost & Financial Perspective

You are the **Cost perspective** in a structured council debate. Your role
is to argue from a financial and resource allocation perspective.

## Your Perspective

You approach every decision through the lens of financial impact and
resource efficiency. You are not balanced — you advocate for fiscal
responsibility. Your job is to:

1. Quantify costs (implementation, maintenance, opportunity)
2. Surface hidden financial implications
3. Challenge ROI assumptions
4. Propose cost-effective alternatives
5. Evaluate total cost of ownership

## How You Argue

- Lead with financial impact: how does the decision affect the bottom line?
- Quantify where possible: even rough estimates are valuable
- Think like a CFO: what's the ROI? What's the payback period?
- Reference TCO models, NPV calculations where relevant
- Propose cost-effective alternatives

## Key Areas of Focus

- Direct costs: implementation, licensing, infrastructure
- Indirect costs: maintenance, training, opportunity cost
- Financial risks: budget overruns, unexpected expenses
- ROI considerations: what return justifies the investment?
- Total cost of ownership: full lifecycle cost analysis

## In Round 2 and Beyond

1. Acknowledge valid points about value creation
2. Highlight financial trade-offs in their proposals
3. Propose cost-compatible alternatives
4. Challenge ROI assumptions with business impact analysis
5. Escalate if a proposal lacks clear financial justification

## Tone

- Pragmatic but principled: fiscal responsibility is an investment
- Evidence-based: grounded in financial analysis
- Constructive: offer cost-effective solutions, not just objections
- Persistent: cost discipline is non-negotiable

## Response Format

- 500–800 words
- Clear sections with headers
- Bullet points for cost comparisons
- End with a clear position statement
```

### Step 3: Test Your Perspective

```
/council Should we migrate to Kubernetes for our container orchestration?
```

During setup, confirm that your new perspective appears in the list and behaves as expected.

### Step 4: Refine Based on Output

After first council:
- Did the perspective stay in character?
- Were arguments too abstract or too specific?
- Did 500–800 words feel sufficient?

Adjust the prompt accordingly.

---

## Domain-Specific Councils

### Engineering Councils (Default)
Perspectives: security, velocity, maintainability
Use cases: Architecture, technology adoption, process changes

### Product Councils
Perspectives: ux, cost, velocity, simplicity
Use cases: Feature prioritisation, product strategy, design decisions

### Hiring Councils
Perspectives: skills-match, culture-fit, growth-potential, diversity
Use cases: Candidate evaluation, levelling decisions

### Procurement Councils
Perspectives: cost, vendor-reliability, integration-complexity, support-quality
Use cases: Tool selection, vendor evaluation

### Incident Response Councils
Perspectives: impact-mitigation, root-cause, prevention, communication
Use cases: Post-incident retrospectives, runbook development

---

## Advanced Customisation

### Custom Synthesis Sections

You can extend the synthesis format by customising `skills/council/references/summariser-prompt.md` and `skills/council/references/synthesis-format.md`.

**When to add sections:**
- Domain-specific concerns that don't fit existing structure
- Organisation-specific reporting requirements
- Integration with existing decision frameworks (e.g., RFC templates)

### Custom Round Structures

Future enhancement (not yet supported): Specialised round types

**Example ideas:**
- **Convergence round:** Perspectives explicitly seek common ground
- **Rebuttal round:** Each perspective responds to one specific other perspective
- **Synthesis round:** Perspectives propose compromise positions

---

## Best Practices

### DO:
- Give perspectives clear identity and values
- Make trade-offs explicit (what this perspective optimises for vs against)
- Provide concrete examples in the prompt
- Test with real propositions before deploying
- Adjust based on actual behaviour

### DON'T:
- Create "balanced" perspectives (defeats the purpose of multi-perspective debate)
- Overlap significantly with existing perspectives
- Add perspectives for every decision (3–5 is optimal)
- Expect perfection on first iteration (prompts require tuning)

### Naming Conventions:
- File: `<perspective-id>.md` (lowercase, hyphenated)
- ID: `<perspective-id>` (used in manifest, file names)

---

## Troubleshooting

### Perspective Stays Off-Topic
- **Cause:** Prompt too vague
- **Fix:** Add specific examples and sharper framing

### Perspective Too Generic
- **Cause:** Prompt lacks clear values/trade-offs
- **Fix:** Strengthen "Your Perspective" and "How You Argue" sections

### Perspective Doesn't Engage in Round 2
- **Cause:** Prompt doesn't emphasise responding to others
- **Fix:** Add explicit guidance in "In Round 2 and Beyond" section

### Perspective Arguments Too Long/Short
- **Cause:** Word count guideline not emphasised
- **Fix:** Repeat "500–800 words" in multiple places in prompt

---

## Contributing Perspectives

If you create a generally useful perspective, consider contributing it upstream:

1. Test with 5+ different propositions
2. Document the perspective's purpose and tuning rationale
3. Submit a PR with the perspective file and examples

Valuable contributions:
- Domain-specific perspectives (healthcare, finance, education)
- Organisation-specific perspectives (startup vs enterprise)
- Cross-cutting concerns (accessibility, internationalisation)

---

## Next Steps

- **[Workflow](workflow.md):** Understand how custom perspectives fit into the council lifecycle
- **[Synthesis](synthesis.md):** Learn how to customise synthesis output
