# Gold Set Case Guide

How to write cases for `gold_set.yaml`. Read this before drafting cases; it defines the case types and shows a full example of each. Every case you write should be recognizable as one of these types, because each type targets a specific downstream failure.

This version of the skill covers four case types: base, multi-hop, ambiguous, edge. Adversarial (not-in-corpus) cases are deferred to a later version; do not emit them.

The handoff reality: this file will be loaded by an implementing agent and reviewed by an engineer. Cases must be self-describing: the question, the expected answer, and the source anchor all stand on their own, with no conversation context needed.

## Case types

### 1. Base case (factual)

The workhorse. Single source, single fact, phrased as a real user would ask it. Roughly 60% of the set. Tests the happy path through retrieval and generation.

```yaml
- id: q001
  question: How many vacation days do employees get after their first year?
  expected_answer: 15 days, increasing by 1 per year of service, capped at 20.
  source_refs:
    - doc: docs/handbook.pdf
      location: "p.12, Benefits section"
  stage: both
  difficulty: easy
  question_type: factual
```

Variants worth including: paraphrase the same fact two ways (keyword-style and conversational), so retrieval is tested for phrasing robustness, not just one lucky formulation.

### 2. Multi-hop case (hard)

Answer requires combining facts from two or more places. Roughly 25% of the set. Tests whether retrieval surfaces multiple sources and whether generation synthesizes rather than picks one.

```yaml
- id: q010
  question: Can a remote employee in their third year take a sabbatical?
  expected_answer: >
    Only employees with 4+ years of service are eligible for sabbatical
    (handbook p.14), and remote employees are subject to the same tenure
    rules as on-site staff (p.27). So a third-year remote employee is not
    eligible.
  source_refs:
    - doc: docs/handbook.pdf
      location: "p.14, Sabbatical policy"
    - doc: docs/handbook.pdf
      location: "p.27, Remote work rules"
  stage: both
  difficulty: hard
  question_type: multi-hop
```

Write the `expected_answer` to show the reasoning chain, so generation correctness judging can check each hop, not just the final yes/no.

### 3. Ambiguous case (hard)

The question is legitimately answerable in multiple ways, or its referent is unclear. Tests whether the system states its assumptions instead of silently guessing.

```yaml
- id: q020
  question: What's our refund policy?
  expected_answer: >
    Ambiguous: the corpus contains separate refund policies for retail
    purchases (store-policy.pdf p.3) and subscriptions (billing.pdf p.8).
    A good answer asks which one, or covers both explicitly.
  source_refs:
    - doc: docs/store-policy.pdf
      location: "p.3"
    - doc: docs/billing.pdf
      location: "p.8"
  stage: generation
  difficulty: hard
  question_type: ambiguous
  grading_note: judge should pass an answer that either disambiguates or covers both; fail one that silently picks a single interpretation
```

The `grading_note` field is optional but load-bearing here: ambiguity cases cannot be graded by string comparison, so the note tells the implementing agent what passing looks like. Use `grading_note` on any case whose correct behavior is not "reproduce expected_answer".

### 4. Edge case (boundary and contradiction)

Answers that exist but sit at boundaries: numbers just past a threshold, policies effective on a date, or (if the corpus has them) sources that contradict each other. Roughly 5%.

```yaml
- id: q040
  question: How much notice do I need to give to book the executive suite?
  expected_answer: 14 days, and same-day booking requires VP approval.
  source_refs:
    - doc: docs/facilities.pdf
      location: "p.5, Room reservations"
  stage: generation
  difficulty: medium
  question_type: edge
  grading_note: answer must include the approval exception, not just the 14-day rule
```

Edge cases also cover temporal boundaries ("what's the policy in 2027?" when the policy changes in 2026) and contradictions between documents, where the correct answer acknowledges the conflict.

## Distribution and sizing

Start at 30 to 50 cases: roughly 65% base, 25% multi-hop, 10% ambiguous/edge. Adversarial cases are deferred to a later version; do not include them. If the domain makes a different mix sensible (a legal corpus wants more ambiguity cases, say), adjust and note why in a comment on the distribution.

## Review checklist (for the engineer)

Before a gold set is accepted, each case must pass:

1. A human expert would answer it the same as `expected_answer`.
2. `source_refs` actually points at content supporting the answer.
3. The question reads like something a real user would type, not a keyword query.
4. Any case with non-obvious grading carries a `grading_note`.
5. Base case questions don't leak the answer's phrasing (no "15 days..." wording inside the question).