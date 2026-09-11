# Efficiency Eval: Tokens, Time, and Tool Productivity

Efficiency metrics are first-class here, not a footnote. They are the numbers that answer "is the component worth it", which is the question most arm matrices exist to serve. Two of these (tokens, time) are exactly computed from traces and need no judge; the third (tool-call productivity) is a ratio with judgment in its denominator.

## Stage metrics

| Metric | Computation | Consumes | Provenance field |
|---|---|---|---|
| Tokens per case | input + output + tool-result tokens, summed per case; reported per case and per category | trace usage events | `provenance-exempt`: usage is not evidence-grounded |
| Wall-clock per case | end-to-end seconds per case; reported with the run's parallelism noted (parallel tool calls compress wall-clock without reducing work) | trace timestamps | `provenance-exempt` |
| Tool calls per case | count of tool/MCP/CLI invocations per case, split by surface | trace | trace: call list |
| Tool-call productivity | calls whose recorded results were used by a later step (or by the final answer) / total calls | reviewer or judge | `tool_calls.result_ids` + downstream usage |
| Cost per case | tokens x price + any tool usage cost, per case, in dollars | usage + pricing | `provenance-exempt` |
| Skill token overhead | tokens consumed loading the skill (SKILL.md + loaded sections), per activation | trace | trace: skill-loads |

Notes:
- Tool-call productivity is the efficiency metric that provenance buys (provenance-eval.md): "the retriever was called 9 times" is not a finding until you know how many calls' results were actually used. A productive-looking call count that returns nothing used is waste; the grounded-vs-prompt-only split applies here too.
- Report tokens and time per case, not per run: a run-level average over a mixed eval set hides the category where the arm exploded (typically multi-step or contextual cases).
- Wall-clock without the parallelism note misleads: an arm that parallelizes tool calls wins wall-clock and loses nothing, while a naive comparison concludes a speedup.

## The worth-it question

Every arm matrix row carries the cost column (ablation.md rule 6). When the decision is "does X earn its place", the standard comparison is:

```
completion delta (points) vs added cost per case (tokens, dollars, seconds)
```

A component that adds 3 completion points at +40k tokens/case may be worth it in a high-stakes domain and not in a bulk pipeline; the plan states the user's stakes instead of deciding by rule. Efficiency deltas inside the noise margin (ablation.md rule 5) are reported as "no measurable difference"; token counts are exact, so their margin is small but nonzero (1 token), while judge-based productivity margins derive from eval set size.

## Spec decisions the plan must pin down

1. **Token accounting rule.** What counts: LLM usage events only, or also embedding/sidecar calls? Cached-token treatment (counted at cache price or full price)? Name it once; mixed accounting makes arms incomparable.
2. **Pricing source and date.** The price list used for cost-per-case, with its date. Costs recompute when prices change; the plan states the source rather than embedding silent numbers.
3. **Wall-clock conditions.** What runs in parallel, whether sandbox startup time is included, and whether the baseline and treatment arms run under the same load. Time comparisons across different conditions are noise.
4. **Productivity join rule.** What "used" means: cited in the final answer, consumed by a downstream tool argument, or judged-necessary. Prefix-tolerant joins over-credit (provenance-eval.md spec decision 2); state the tolerance and the ceiling.
5. **Efficiency regression behavior.** Whether a token or time regression above the margin fails the run or flags for review. Defaults: fail on >2x time or >2x token regressions for the same completion rate; flag otherwise. State the chosen numbers with their reason, or mark TBD.