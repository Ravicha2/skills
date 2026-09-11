# Task Completion and Answer Eval

This reference covers the metrics that grade the destination: did the task produce the right final artifact or answer. It is the agent-pipeline counterpart of generation-eval: same judging discipline (per-case criteria, grading_notes, judges that see recorded evidence), applied to `expected_output` and completion rather than grounded answers.

## Stage metrics

| Metric | Computation | Consumes | Provenance field |
|---|---|---|---|
| Task completion rate | cases where the final artifact matches expected_output / all cases with expected_output | `expected_output` | `generation.final_answer` + recorded results used |
| Answer correctness | per-case rubric below / cases with expected_output | `expected_output` | recorded result sets |
| Goal achievement | did the run end with the user's goal met (not just a well-formed answer) | task + `expected_output` | full trajectory |
| Artifact validity | produced files parse/compile/open as claimed | `expected_output` | file-system events |

The 5-criterion rubric adapted for agent pipelines (borrowed from skillevaluator's accuracy rubric; use when the artifact is prose or a mixed answer rather than a file):

```
Score each criterion against the expected_output. For each, answer YES or NO.
1. TOOL_IDENTIFIED: did the agent use the expected surface (skill/MCP/CLI) where the case expects it?
2. ACTION_CORRECT: are the actions taken the ones the task requires?
3. FACTUALLY_ACCURATE: are the factual claims consistent with the recorded results?
4. TASK_ADDRESSED: does the output directly address the user's request?
5. ACTIONABLE: does the output provide usable information (not just acknowledgment)?
score = count(YES) / 5
Be lenient on exact wording but strict on factual correctness.
```

Caveats:
- Criterion 1 belongs to trajectory-eval when the case is implicit or negative; in those cases grading tool use here double-counts. The plan states which reference owns criterion 1 per category.
- FACTUALLY_ACCURATE is judged against recorded results (ids plus text), not against the agent's own summary of what it saw (provenance-eval.md spec decision 4). An answer consistent with a hallucinated observation is not accurate.
- A case whose `grading_note` redefines passing passes when its note is satisfied, regardless of the rubric score. The note wins; the rubric is the default, not an override.

## Per-case pass vs aggregate

- Per-case pass/fail comes from `expected_output` match, the rubric, or the `grading_note` — decided per case, never from an aggregate crossing a threshold.
- Aggregates (completion rate, mean rubric score) are for tracking and arms comparison, not for passing individual cases.
- When a plan runs an M2-only arm (trajectory captured, final answers not graded), completion metrics cannot be computed in that arm; the plan states this so the implementer does not guess.

## Spec decisions the plan must pin down

1. **Artifact match rule.** For file artifacts: exact content, structural match (parses, right schema), or spot-checked fields? State which per artifact type; "correct file" under-specified gets graded three different ways by three implementers.
2. **Answer judge inputs.** The judge for correctness receives: the task, `expected_output`, the final answer, and the recorded result sets. State this in the plan; a judge without the recorded results grades plausibility, not grounding.
3. **Partial credit policy.** Whether multi-part tasks get per-part credit (each part pass/fail, reported separately) or all-or-nothing. Default per-part: a reconciliation that is right except for the flagged row is a different finding from a wrong reconciliation, and the delta between arms lives in the parts.
4. **Completion vs quality split.** Task completion (artifact exists and is structurally valid) and answer quality (rubric) are separate rows with separate thresholds; one number hides arms that ship fast wrong answers.