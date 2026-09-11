# Trajectory, Triggering, and Behavior Eval

This reference covers the metrics that grade the path the agent took, not the destination: whether the right skill or tool fired, whether the workflow order was sane, whether the agent recovered from errors, and whether it avoided what it should have avoided. These signals live in the trajectory (the ordered record of tool/MCP/CLI calls and results), which is why they depend on provenance-eval.md invariant 1: a trajectory of calls without recorded results cannot support behavior grading, because "the agent read the config" and "the agent asked and was refused" look identical.

## Stage metrics

| Metric | Computation | Consumes | Provenance field |
|---|---|---|---|
| Trigger precision | correct non-firing / all must-not cases | `trigger`, `must_not_use` | trace: skill-loads and tool-calls sequences |
| Trigger recall | correct firing / all must cases | `trigger` | trace: skill-loads and tool-calls sequences |
| Skill execution order | cases where expected_behavior items appear in the trajectory in order / all must cases | `expected_behavior` | trace: ordered call+result list |
| Behavior adherence | expected_behavior items individually checked present or absent per case (per-item, not averaged) | `expected_behavior` | trace: ordered call+result list |
| Error recovery | recovery outcomes (flagged, retried-successfully, fabricated, aborted) counted per case | case obstacles | trace: error events and subsequent calls |
| Spurious tool calls | calls not required by any expected_behavior and not serving the task, per case | reviewer judgment | trace: ordered call+result list |

Notes:
- Trigger precision and recall are reported as a pair; a single "routing accuracy" number hides which side failed, and the fix differs (precision fails → negative cases; recall fails → discovery or description problems).
- Per-item behavior adherence is deliberately not averaged into one score per case: an agent that does 4 of 5 behavior items is a different finding depending on which item it skipped, and the item names travel in the report.
- Error recovery grading requires the case to actually contain an obstacle (multi-step cases with deliberate failures); a recovery metric over cases with no obstacle measures nothing.

## Judge rubric for behavior checks (when not exactly computable)

When expected_behavior items are not mechanically checkable from the trace (for example "identifies the right capability"), use an LLM judge over the trajectory with per-item YES/NO:

```
For each expected_behavior item, answer YES or NO against the trajectory:
1. Is the behavior observable in the trace (a call, a file, an output)?
2. Did the recorded results (not the agent's narration) support it?
Be lenient on exact wording but strict on sequence and evidence.
Output per item: {item, verdict: YES|NO, reason}.
```

The judge receives the recorded result sets, not just the calls (provenance-eval.md spec decision 4). A judge scoring from narration alone credits agents that talked about the right tool without evidence they used it.

## Spec decisions the plan must pin down

1. **Skill-load definition.** What counts as "the skill fired": the SKILL.md read? a section read? a script run? Name the trace event(s) per harness. Under-skilling (counting a glance at the file) overstates recall; over-skilling misses partial activation.
2. **Tool-call identity.** For each MCP server and CLI tool, how a call appears in the trace (server name + tool name + arguments; CLI binary + argv). State it per surface; normalizing ad hoc makes cross-arm comparisons noise.
3. **Negative-case grading.** Exact events that constitute a violation (any read of the skill dir? any script invocation? any MCP tool call from that server?), per must-not case. Grading from the final answer alone passes agents that fired the tool and then hid it.
4. **Recovery outcome taxonomy.** The fixed set: `flagged` (reported, did not fabricate), `retried-successfully`, `failed-openly`, `fabricated`, `hid`. Fabricated and hid are fail outcomes regardless of the final answer; name which cases each applies to.
5. **Order tolerance.** Whether expected_behavior order is strict or lenient (for example: config read must precede write, but the two report steps may swap). State per case or per behavior item, not globally.