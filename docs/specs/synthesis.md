# Synthesis Format Specification

**Why this matters:** Synthesis documents extract actionable insights from raw debate rounds, making council outputs useful for decision-making.

---

## Purpose

The synthesis document is **not** a transcript. It's a structured extraction that:
- Surfaces what's agreed upon (low-controversy decisions)
- Highlights tradeoffs that require conscious choice
- Maps risks to the perspectives that identified them
- Recommends a path forward with reasoning
- Preserves important dissent

Think of it as an executive summary written by a neutral observer who attended the debate.

---

## The 6 Sections

### 1. Consensus

**What it contains:** Points where all or most perspectives agreed.

**Why it matters:** Consensus items represent low-controversy decisions. You can likely proceed without further debate.

**Example:**
```markdown
## Consensus

All perspectives agree that:
- Current authentication system has known vulnerabilities requiring remediation
- OAuth 2.0 is a well-established standard with strong ecosystem support
- Migration will require coordination across 4 services and mobile clients
```

**Summariser guidelines:**
- List only genuine agreement (not "security agrees with velocity's point about...")
- Be specific (not "everyone thinks security matters")
- Separate agreement on facts from agreement on recommendations

---

### 2. Key Tensions

**What it contains:** Fundamental disagreements between perspectives.

**Why it matters:** Tensions represent tradeoffs you must consciously choose. There's no "right" answer—you must weigh priorities.

**Structure per tension:**
- What the disagreement is about
- Which perspectives are on each side
- The strongest argument from each side

**Example:**
```markdown
## Key Tensions

### Migration Timeline: Fast vs Safe

**Velocity's position:** Migrate within 2 sprints to unblock mobile team and reduce maintenance burden.
- Argument: Current system requires manual intervention for password resets. Mobile team is blocked on SSO feature. Delaying adds 4 weeks to mobile roadmap.

**Security's position:** 4-week migration with comprehensive testing and staged rollout.
- Argument: OAuth misconfiguration is a common attack vector. Rushing increases risk of exposed credentials, broken access controls, or data leakage. Production incident would delay mobile team even more.

**No middle ground identified:** Both perspectives acknowledge the other's concerns but disagree on risk tolerance.
```

**Summariser guidelines:**
- Don't invent compromise ("maybe 3 weeks") if perspectives didn't converge
- Attribute arguments to specific perspectives
- Explain **why** they disagree (different risk tolerance, different information, different values)

---

### 3. Risk Assessment

**What it contains:** Material risks of each decision path, mapped to the perspectives that identified them.

**Why it matters:** Helps you evaluate worst-case scenarios and who's concerned about what.

**Example:**
```markdown
## Risk Assessment

### If we proceed with 2-sprint migration (Velocity's recommendation):

**Security risks:**
- OAuth misconfiguration leading to unauthorized access (CVSS 9.1)
- Insufficient testing of edge cases (password reset mid-migration, concurrent sessions)
- No rollback plan if production issues emerge

**Maintainability risks:**
- Technical debt from rushed implementation (hardcoded redirects, incomplete error handling)
- Documentation gaps making future changes risky

### If we proceed with 4-week migration (Security's recommendation):

**Velocity risks:**
- Mobile team blocked for additional 2 sprints, delaying Q2 OKRs
- Opportunity cost: 2 sprints of engineering time on infrastructure vs features
- Morale impact from extended timeline on aging system

**Maintainability risks:**
- Prolonged support of two authentication systems during migration phase
```

**Summariser guidelines:**
- Group risks by decision path, not by severity
- Map each risk to the perspective that raised it
- Include both technical and organizational risks

---

### 4. Recommended Path Forward

**What it contains:** A concrete recommendation with reasoning.

**Why it matters:** You asked for advice. This is where the council delivers it.

**Criteria for the recommendation:**
- Best risk/reward profile given the arguments
- **Not** a hollow compromise ("do both")
- Must be specific and actionable

**Example:**
```markdown
## Recommended Path Forward

**Adopt 3-week migration timeline with staged rollout and contingency plan.**

**Rationale:**

This balances velocity's urgency with security's risk concerns:

1. **Week 1:** Security team completes threat model and test plan. Engineering implements OAuth client config in non-prod.

2. **Week 2:** Migrate internal tools (lower risk) to validate implementation. Engineering completes remaining services.

3. **Week 3:** Staged rollout to production (10% → 50% → 100% over 3 days). Monitor error rates, auth failures, session anomalies.

**Contingency:** Maintain dual-auth support for 1 week post-migration. If critical issues emerge, rollback to legacy system while fixes are implemented.

**Why this is better than alternatives:**
- Faster than security's 4-week proposal (mobile team unblocked 1 week earlier)
- Safer than velocity's 2-week proposal (staged rollout + rollback plan mitigate misconfiguration risk)
- Addresses maintainability's concern about technical debt (proper test plan and documentation time)

**Tradeoffs accepted:**
- Still impacts Q2 OKRs (though less than 4-week option)
- Requires maintaining dual-auth temporarily (added complexity)
```

**Summariser guidelines:**
- Be specific: timelines, steps, responsibilities
- Explain **why** this recommendation balances the perspectives
- Acknowledge what tradeoffs are being accepted
- If no clear recommendation exists, say so and explain why

---

### 5. Minority Report

**What it contains:** Important dissenting positions that weren't adopted in the recommendation.

**Why it matters:** Dissent often contains valuable insights. Even if you accept the majority recommendation, you should understand what's being sacrificed.

**Example:**
```markdown
## Minority Report

**Security's dissent on 3-week timeline:**

While security accepts the 3-week compromise, they emphasize that staged rollout does not eliminate misconfiguration risk—it only contains blast radius. 

Key concern not addressed in recommendation: OAuth token storage in mobile clients. If tokens are stored insecurely, compromise could affect all users regardless of rollout pace.

**Recommended action if proceeding:** Conduct mobile client security review in Week 1 as prerequisite. If insecure storage is found, extend timeline or mitigate before migration.

---

**Velocity's concern about contingency complexity:**

Maintaining dual-auth for 1 week post-migration adds operational complexity and delays full deprecation of legacy system.

Alternative: Accept rollback will be painful (require downtime) but don't build dual-auth scaffolding. This saves engineering time and simplifies Week 3.

**Recommendation:** If risk tolerance is high, consider velocity's alternative. If downtime is unacceptable, stick with dual-auth.
```

**Summariser guidelines:**
- Include only substantive dissent (not "velocity prefers 2 weeks but accepts 3")
- Explain **what risk or concern is unaddressed** by the recommendation
- If the dissenter offers an alternative action, include it

---

### 6. Suggested Actions

**What it contains:** Numbered, concrete next steps if the recommendation is accepted.

**Why it matters:** Bridges the gap between "we've decided" and "now what."

**Criteria for actions:**
- Specific enough to assign to a person or team
- Verifiable as complete
- Sequenced if order matters

**Example:**
```markdown
## Suggested Actions

1. **Security team:** Complete OAuth 2.0 threat model and test plan by EOW (Friday).
   - Include mobile token storage review
   - Define success criteria for staged rollout (error rate thresholds, monitoring plan)

2. **Engineering:** Implement OAuth client config in staging by Week 1, Day 3.
   - Use client_credentials grant for service-to-service
   - Use authorization_code + PKCE for mobile clients

3. **Mobile team:** Audit token storage implementation by Week 1, Day 5.
   - Verify tokens stored in iOS Keychain / Android Keystore
   - If insecure storage found, escalate immediately (timeline decision point)

4. **Engineering:** Migrate internal tools (admin panel, CI/CD) to OAuth in Week 2.
   - Validate error handling, session management, token refresh
   - Document any issues for main migration

5. **Engineering + Ops:** Execute staged production rollout in Week 3.
   - Day 1: 10% of users
   - Day 2: 50% of users (if error rate < 0.1%)
   - Day 3: 100% of users (if no critical issues)

6. **Ops:** Maintain dual-auth support for 1 week post-migration.
   - Monitor auth failure rates
   - Deprecate legacy system only after 1 week of stable OAuth-only usage
```

**Summariser guidelines:**
- Each action starts with a responsible party
- Include completion criteria or timeline
- Sequence matters: list actions in execution order
- If an action is conditional (depends on a decision point), note it

---

## What the Summariser Does NOT Do

### Does Not Invent Arguments

**Wrong:**
```markdown
Security likely would be concerned about OAuth token expiration times.
```

**Right:**
```markdown
Security did not explicitly address token expiration in their arguments.
```

If a perspective didn't mention something, the summariser doesn't infer it.

### Does Not Soften Disagreements

**Wrong:**
```markdown
While security and velocity have different preferences, there's room for compromise.
```

**Right:**
```markdown
Security and velocity identified fundamentally different risk tolerances. No middle ground emerged in debate.
```

If perspectives are irreconcilable, say so. Don't manufacture consensus.

### Does Not Editorialize

**Wrong:**
```markdown
Velocity's 2-week timeline is clearly too aggressive given the security concerns.
```

**Right:**
```markdown
Velocity proposes 2-week timeline. Security identifies this as high-risk. Recommendation adopts 3-week timeline to balance concerns.
```

The summariser reports and structures; it doesn't judge.

---

## Length Guidelines

**Total synthesis:** 1000-1500 words

**Per section:**
- Consensus: 100-200 words
- Key Tensions: 300-400 words (largest section)
- Risk Assessment: 200-300 words
- Recommended Path Forward: 200-300 words
- Minority Report: 100-200 words
- Suggested Actions: 100-200 words

**Why these lengths:**
- Long enough to be useful
- Short enough to read in 5 minutes
- Forces precision (no rambling)

---

## Quality Checklist

Before finalizing synthesis, verify:

- [ ] Every claim in Consensus is supported by perspective arguments
- [ ] Every Tension identifies specific perspectives on each side
- [ ] Every Risk is mapped to a perspective that raised it
- [ ] Recommended Path Forward is actionable (not vague)
- [ ] Minority Report captures substantive dissent (not noise)
- [ ] Every Suggested Action has a responsible party and completion criteria
- [ ] No arguments invented that weren't made by perspectives
- [ ] Disagreements not artificially softened
- [ ] Total length under 1500 words

---

## Example: Full Synthesis

See [example-synthesis-oauth-migration.md](../examples/synthesis-oauth-migration.md) for a complete synthesis document following this specification.

---

## Next Steps

- **[Workflow](workflow.md):** Understand when synthesis happens in the council lifecycle
- **[File Formats](file-formats.md):** Learn the YAML frontmatter for synthesis.md
- **[Extending](extending.md):** Customize synthesis for domain-specific output
