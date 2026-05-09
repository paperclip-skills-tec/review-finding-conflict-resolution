---
name: review-finding-conflict-resolution
description: "Use when two or more specialist reviewers reference the same file, function, or diff region with different severity or conclusion — before drafting the synthesis verdict. Invoke during the conflict detection phase of review synthesis when reviewers disagree: one says PASS, another says FAIL; same code area but different severity ratings; or one reviewer implicitly contradicts another's clean verdict. Required for Code Review Lead to detect, classify, and resolve reviewer conflicts with documented rationale. Do not resolve conflicts freehand."
---

# Review Finding Conflict Resolution

When five specialist reviewers independently assess the same diff, they will sometimes reach
different conclusions about the same code. Without a structured protocol, conflicts are either
silently ignored (one reviewer's finding dominates) or force an undocumented judgment call.

This skill defines the detection pass, conflict classification, resolution hierarchy, and output
format required before the synthesis verdict can be posted.

**Invoke this skill from Step 3 of `review-synthesis-verdict`**, after extracting all specialist
findings in Step 1 and before applying verdict decision logic in Step 5.

---

## Step 1 — Conflict Detection Pass

Before drafting any synthesis text, scan all extracted findings for overlapping references.

Build a working index of findings by location:

| Finding | Reviewer | Severity | File / Function / Diff region | Conclusion |
|---------|----------|----------|-------------------------------|------------|
| … | … | … | … | … |

**Match on any of these anchors:**
- Same file path (even if different line ranges within the file)
- Same function or class name cited in the finding
- Same diff region (e.g., "the auth middleware", "the pagination handler")

For every pair of reviewers who reference the same anchor, flag it as a **conflict candidate**.
Do not filter at this stage — classification happens in Step 2.

If no overlapping references exist across all five reviewers, write:
> "Conflict detection pass complete. No overlapping references found."

Then exit this skill and continue with Step 4 of `review-synthesis-verdict`.

---

## Step 2 — Classify Each Conflict Candidate

For each conflict candidate, determine its type:

### Type A — True Contradiction
One reviewer concludes the code is correct / safe / acceptable for the specific concern they
were assigned; another reviewer concludes the same code is broken / unsafe / unacceptable on
the same dimension.

**Hallmarks:** one says PASS and one says FAIL on the same specific question. The conclusions
are mutually exclusive — both cannot be right.

**Examples:**
- Correctness says "null input is handled" — Test Coverage says "null path has no test and the
  handler was observed to throw during QA"
- Security says "this route is auth-protected" — Correctness says "the middleware is bypassed in
  the redirect flow"

→ Proceed to Step 3 (Resolution Hierarchy).

### Type B — Complementary Finding
Both reviewers reference the same file or function, but their concerns are on different
dimensions and do not contradict each other. Both findings can be true simultaneously.

**Hallmarks:** the findings address different properties of the same code — one flags design,
the other flags a security issue; one flags performance, the other flags naming. There is no
logical conflict.

**Action:** Not a real conflict. Handle each finding independently per its severity tier.
Document: "Architecture and Security both flag `[file]`; concerns are independent — handled
separately."

### Type C — Severity Disagreement
Both reviewers flag the same issue as real, but they rate its severity differently. Neither is
saying PASS while the other says FAIL — they agree the problem exists but disagree on how
serious it is.

**Hallmarks:** both findings describe the same defect; the disagreement is one says BLOCKING
and the other says ADVISORY (or one ADVISORY and one OBSERVATION).

**Action:** Apply the stricter rating. Document which reviewer rated it higher and that the
stricter rating was adopted. No further resolution logic needed.

---

## Step 3 — Resolve True Contradictions (Type A Only)

For each Type A conflict, apply the resolution criteria in this order. Stop at the first
criterion that resolves the conflict.

### Resolution Criterion 1 — Reviewer Domain Precedence

Each reviewer owns one domain. When two reviewers contradict each other on a question that
falls squarely in one domain, the domain owner's conclusion takes precedence:

```
Security > Correctness > Code Quality > Architecture > Test Coverage
```

**Apply when:** the conflict is on a question within one reviewer's explicit domain and the
other reviewer's finding is incidental (they noticed something outside their primary remit).

**Document:** "Security's conclusion prevails — auth correctness falls within the Security
Reviewer's primary domain. [Correctness Reviewer]'s observation noted but does not override."

### Resolution Criterion 2 — Defer to the Stricter Finding

When domain precedence does not resolve the conflict (e.g., both reviewers have equal
standing on the question), default to the stricter finding. A false positive is recoverable;
a missed defect is not.

**Apply when:** neither reviewer's domain clearly owns the question, or both have equal claim.

**Document:** "Domain precedence does not resolve this conflict. Adopting [Reviewer]'s BLOCKING
classification per the defer-to-stricter rule."

### Resolution Criterion 3 — Evidence Weight

When both criteria above leave the conflict open (e.g., both reviewers cite evidence and both
have domain standing), weigh the evidence directly:

- Reviewer who cited a specific code path, test result, or reproduction step > reviewer who
  cited a general principle or "likely" concern
- Concrete evidence beats reasoned inference

**Document:** "[Reviewer A] cited [specific evidence]. [Reviewer B] cited [general principle].
Resolving in favor of [A] on evidence weight."

---

## Step 4 — Escalation Condition

Do NOT apply the resolution hierarchy if the conflict meets any of these conditions. Instead,
flag it for human judgment and do not issue a verdict until the flag is resolved:

- **Safety-critical:** the contradiction bears on data integrity, authentication, or a
  security vulnerability — and you cannot determine from the evidence which reviewer is right
- **Architecturally fundamental:** the contradiction is about a design decision that affects
  multiple current or future features, not just the diff under review
- **Spec mismatch:** the reviewers are operating from different assumptions about what the
  feature is supposed to do — a code-level review cannot resolve a requirements disagreement

**When escalating:**

```markdown
### Conflict Escalation — Human Judgment Required

**Reviewers:** [A] vs [B]
**File/Area:** [anchor]
**[A] finding:** [summary]
**[B] finding:** [summary]
**Reason for escalation:** [safety-critical | architecturally fundamental | spec mismatch]
**Question to resolve:** [one crisp sentence stating what the human must decide]

⚠️ Verdict is blocked pending resolution of this conflict. Do not merge until this is addressed.
```

Set the verdict to BLOCKED and name the escalation explicitly in the synthesis comment.

---

## Step 5 — Output Requirement

Every resolved conflict (Type A or C) must appear in the synthesis comment's **Conflicts**
section with this structure:

```markdown
### Conflict: [Reviewer A] vs [Reviewer B] — [file/area]

**[Reviewer A]:** [one-sentence finding summary]
**[Reviewer B]:** [one-sentence finding summary]
**Type:** [True Contradiction | Severity Disagreement]
**Resolution:** [Domain precedence — Security > Correctness | Defer-to-stricter | Evidence weight]
**Applied finding:** [the finding that was adopted, with its severity tier]
**Verdict impact:** [how this affects the overall verdict — e.g., "BLOCKING finding upheld; verdict remains BLOCKED"]
```

Type B (complementary) conflicts are documented with a single line:
```
[Reviewer A] and [Reviewer B] both reference `[file]`; concerns are independent — no conflict.
```

**Silence is not acceptable.** If you resolved a conflict, it must appear in the output.
A synthesis comment with unacknowledged conflicts is not a valid verdict.

---

## Quick Reference

| Conflict Type | Resolution | Output required |
|---|---|---|
| True Contradiction (A) | Domain precedence → defer-to-stricter → evidence weight | Full conflict block with resolution rule cited |
| Complementary (B) | Handle independently — not a conflict | One-line note: "independent concerns" |
| Severity Disagreement (C) | Apply stricter rating | Full conflict block (abbreviated) |
| Escalation condition | Block verdict, flag for human | Escalation block with crisp question |

**Domain precedence (for criterion 1):**
```
Security > Correctness > Code Quality > Architecture > Test Coverage
```

**Never silently pick one finding over another.** Document the resolution rule applied.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
