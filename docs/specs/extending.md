# Extending OpenCode Council

**Why this matters:** Council is designed to be extended. Custom perspectives let you adapt the system to your domain, organization, and decision-making needs.

---

## Recommended: Use `/generate-council`

The fastest way to create project-tailored advocates is the generator:

```
/generate-council
```

The generator:
1. Reads your project's codebase, tech stack, and dependencies
2. Detects available MCP servers
3. Asks 2-3 targeted questions about your decision-making needs
4. Generates 3-5 advocate agents with project-specific domain knowledge, calibrated temperatures, and perspective-appropriate bash permissions

Re-run `/generate-council` whenever the project evolves — new MCP servers, new domain concerns, or team growth.

The sections below describe how to create advocates manually, which is useful for fine-tuning generated advocates or creating highly specialized perspectives.

---

## Adding Custom Advocate Perspectives (Manual)

### When to Add a New Perspective

**Good reasons:**
- Your decisions have a recurring dimension not covered by default advocates
- Your organization values a specific concern (cost, compliance, UX, scalability)
- You're applying councils to a non-engineering domain (hiring, marketing, procurement)

**Examples of useful custom perspectives:**
- **advocate-cost:** Financial implications, ROI, budget constraints
- **advocate-ux:** User experience, accessibility, product intuition
- **advocate-compliance:** Regulatory requirements, audit trails, data sovereignty
- **advocate-scalability:** Growth considerations, capacity planning
- **advocate-simplicity:** Reducing complexity, avoiding over-engineering

**Not useful:**
- Duplicates of existing perspectives (don't create "advocate-security-2")
- Perspectives that don't have distinct viewpoints ("advocate-common-sense")
- Joke perspectives that dilute serious debate

---

## Creating an Advocate Agent

### Step 1: Copy a Template

```bash
cp .opencode/agents/advocate-security.md .opencode/agents/advocate-cost.md
```

### Step 2: Update the YAML Frontmatter

```yaml
---
description: >
  Cost and financial perspective for council debates. Argues from
  budget constraints, ROI considerations, and opportunity cost framing.
mode: subagent
temperature: 0.4
tools:
  read: true
  glob: true
  grep: true
  bash: true
  write: false
  edit: false
  task: false
  webfetch: false
permission:
  bash:
    "*": deny
    "cat *": allow
    "ls *": allow
    "grep *": allow
    "find *": allow
---
```

**Key fields:**
- **description:** What this perspective argues for (shown to user during setup)
- **mode:** Always `subagent` for advocates
- **temperature:** 0.3-0.6 depending on perspective personality
  - 0.3-0.4: Conservative, risk-focused (security, compliance)
  - 0.4-0.5: Balanced, evidence-based (maintainability, cost)
  - 0.5-0.6: Creative, pragmatic (velocity, UX, simplicity)
- **tools:** Read-only + limited bash (never write/edit)
- **permission:** Restrict bash to safe read commands

### Step 3: Write the Perspective Prompt

```markdown
# Cost & Financial Perspective

You are the **Cost advocate** in a structured council debate. Your role is to argue from a financial and resource allocation perspective.

## Your Core Values

- **Budget responsibility:** Every technical decision has financial implications
- **ROI thinking:** Investments should demonstrate clear returns
- **Opportunity cost:** Resources spent here can't be spent elsewhere
- **Total cost of ownership:** Consider implementation, maintenance, and opportunity costs

## What You Argue For

- Solutions that optimize for long-term financial sustainability
- Quantifiable cost-benefit analysis where possible
- Awareness of hidden costs (operational overhead, opportunity cost, switching costs)
- Pragmatic tradeoffs when perfect solutions are prohibitively expensive

## What You Argue Against

- Expensive solutions without clear ROI justification
- Hidden costs that haven't been surfaced
- "Gold plating" that adds cost without proportional value
- Ignoring opportunity cost of resource allocation

## How to Argue

### Round 1: Present Your Strongest Case

Structure your argument:
1. **Direct costs:** Implementation, licensing, infrastructure
2. **Indirect costs:** Maintenance, training, opportunity cost
3. **Financial risks:** Budget overruns, unexpected expenses
4. **ROI considerations:** What return justifies the investment?

Be specific. Quantify where possible (even rough estimates are valuable).

**Example:**
- Don't say: "This will be expensive"
- Do say: "Migrating to Kubernetes requires: $50K/year managed cluster fees, 4 weeks of eng time ($40K opportunity cost), and ongoing 10% overhead for ops. Total first-year cost: ~$100K. What's the quantifiable return?"

### Round 2+: Respond to Other Perspectives

- **When you agree:** Acknowledge valid points about value creation
- **When you disagree:** Challenge ROI assumptions, surface hidden costs
- **Identify false dichotomies:** "Security says we must do X, but cheaper alternatives exist"

Maintain your financial framing, but engage with their evidence.

## Guidelines

- **Keep responses to 500-800 words**
- **Reference specific standards or frameworks when relevant** (e.g., TCO models, NPV calculations)
- **Structure with clear sections** (Direct Costs, Hidden Costs, ROI Analysis, Recommendation)
- **You are not balanced** — make the strongest financial case, don't hedge

## Round-Specific Context

**Round 1:** You will receive the proposition and nothing else. Argue from first principles.

**Round 2+:** You will receive:
- Your own prior round arguments
- Other perspectives' prior arguments

Build on your position. Respond to others. Refine your recommendation.

---

When invoked, present your argument following these guidelines.
```

### Step 4: Test Your Advocate

```
/council Should we migrate to Kubernetes for our container orchestration?
```

During setup, confirm that your new perspective appears in the list and behaves as expected.

### Step 5: Refine Based on Output

After first council:
- Did the advocate stay in character?
- Were arguments too abstract or too specific?
- Did temperature feel right? (too random = too high, too rigid = too low)
- Did 500-800 words feel sufficient?

Adjust prompt and temperature accordingly.

---

## Temperature Tuning Guide

Temperature controls how creative vs deterministic the advocate's responses are.

### 0.2-0.3: Highly Deterministic
**Personality:** Strict, standards-based, risk-averse  
**Good for:** Compliance, security, regulatory perspectives  
**Behavior:** Cites standards, repeats established patterns, minimal creativity

**Example perspectives:**
- Security (follows OWASP, NIST)
- Compliance (follows regulations, audit requirements)

### 0.4-0.5: Balanced
**Personality:** Evidence-based, pragmatic, measured  
**Good for:** Cost, maintainability, scalability perspectives  
**Behavior:** Balances creativity with consistency, argues from principles but adapts

**Example perspectives:**
- Cost (considers multiple financial models)
- Maintainability (balances long-term health with pragmatism)

### 0.6-0.7: Creative
**Personality:** Innovative, opportunistic, flexible  
**Good for:** Velocity, UX, simplicity perspectives  
**Behavior:** Generates novel arguments, challenges assumptions, thinks laterally

**Example perspectives:**
- Velocity (finds creative shortcuts)
- UX (proposes unconventional solutions)
- Simplicity (challenges complexity)

### 0.8+: Highly Creative (Usually Too High)
**Personality:** Unpredictable, experimental  
**Risk:** Arguments may drift off-topic or become inconsistent  
**Use case:** Rare — only for perspectives that intentionally challenge norms

---

## Domain-Specific Councils

### Engineering Councils (Default)
Perspectives: security, velocity, maintainability  
Use cases: Architecture, technology adoption, process changes

### Product Councils
Perspectives: ux, cost, velocity, simplicity  
Use cases: Feature prioritization, product strategy, design decisions

### Hiring Councils
Perspectives: skills-match, culture-fit, growth-potential, diversity  
Use cases: Candidate evaluation, leveling decisions

### Procurement Councils
Perspectives: cost, vendor-reliability, integration-complexity, support-quality  
Use cases: Tool selection, vendor evaluation

### Incident Response Councils
Perspectives: impact-mitigation, root-cause, prevention, communication  
Use cases: Post-incident retrospectives, runbook development

---

## Advanced Customization

### Custom Synthesis Sections

You can extend the synthesis format by customizing `council-summariser.md`:

**Example: Add "Cost-Benefit Analysis" section**

```markdown
## OUTPUT STRUCTURE:
Produce a synthesis document with exactly these sections:
1. Consensus
2. Key Tensions
3. Risk Assessment
4. **Cost-Benefit Analysis** (if cost advocate participated)
5. Recommended Path Forward
6. Minority Report
7. Suggested Actions
```

**When to add sections:**
- Domain-specific concerns that don't fit existing structure
- Organization-specific reporting requirements
- Integration with existing decision frameworks (e.g., RFC templates)

### Custom Round Structures

Future enhancement (not yet supported): Specialized round types

**Example ideas:**
- **Convergence round:** Advocates explicitly seek common ground
- **Rebuttal round:** Each advocate responds to one specific other perspective
- **Synthesis round:** Advocates propose compromise positions

---

## Best Practices

### DO:
- Give perspectives clear identity and values
- Make trade-offs explicit (what this perspective optimizes for vs against)
- Provide concrete examples in the prompt
- Test with real propositions before deploying
- Adjust temperature based on actual behavior

### DON'T:
- Create "balanced" perspectives (defeats the purpose of multi-perspective debate)
- Overlap significantly with existing perspectives
- Add perspectives for every decision (3-5 is optimal)
- Expect perfection on first iteration (prompts require tuning)

### Naming Conventions:
- File: `advocate-<perspective-id>.md` (lowercase, hyphenated)
- ID: `<perspective-id>` (used in manifest, file names)
- Description: "X perspective" (e.g., "Cost and financial perspective")

---

## Troubleshooting

### Advocate Stays Off-Topic
- **Cause:** Temperature too high, prompt too vague
- **Fix:** Lower temperature by 0.1-0.2, add specific examples to prompt

### Advocate Too Generic
- **Cause:** Prompt lacks clear values/trade-offs
- **Fix:** Strengthen "What You Argue For/Against" sections

### Advocate Doesn't Engage in Round 2
- **Cause:** Prompt doesn't emphasize responding to others
- **Fix:** Add explicit "Round 2: Respond to other perspectives" section

### Advocate Arguments Too Long/Short
- **Cause:** Word count guideline not emphasized
- **Fix:** Repeat "500-800 words" in multiple places in prompt

---

## Contributing Perspectives

If you create a generally useful perspective, consider contributing it upstream:

1. Test with 5+ different propositions
2. Document the perspective's purpose and tuning rationale
3. Submit a PR with the advocate file and examples

Valuable contributions:
- Domain-specific perspectives (healthcare, finance, education)
- Organization-specific perspectives (startup vs enterprise)
- Cross-cutting concerns (accessibility, internationalization)

---

## Next Steps

- **[Workflow](workflow.md):** Understand how custom advocates fit into the council lifecycle
- **[Synthesis](synthesis.md):** Learn how to customize synthesis output
- **[File Formats](file-formats.md):** Understand advocate configuration structure
