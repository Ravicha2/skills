# Retrieval Eval

Retrieval evals answer one question: for a query, did the right chunks land in the top-k window passed to generation? Everything the generation stage can do is bounded by this. It is usually the highest-leverage place to measure, and the cheapest to wire, so it belongs first in the build order.

## What can go wrong

| Failure | Symptom downstream |
|---|---|
| Right chunk ranked below k | Generation cannot answer, or hallucinates to fill the gap |
| Right chunk retrieved, wrong rank | Weak answer; the good chunk gets diluted by noise around it |
| Near-duplicate chunks fill top-k | k slots wasted, no new information |
| Keyword/semantic mismatch | Systematic failure on a whole question phrasing style |
| Multi-hop gap | Each hop retrievable alone, but no query formulation retrieves both |

## Metrics

All of these consume the gold set's `source_refs`: a retrieved chunk counts as relevant if it matches (by chunk, section, or location overlap) a case's expected source. Define the matching rule in the plan precisely. Two options, pick one and state it:
- chunk-id equality: exact, requires stable chunk ids between gold-set authoring and retrieval runs.
- section overlap: the pragmatic fallback when chunk boundaries shift. A retrieved chunk counts as relevant if `chunk.doc == gold source_refs[].doc` AND the chunk text contains the section heading derived from `source_refs[].location`. Specify the derivation with a worked example, and get the direction right. The `location` is typically shaped like `"<Section name> section, <detail>"` (e.g., `"Vacation section, line 4"`); the heading is the text BEFORE the first comma, with a trailing ` section` suffix stripped, lower-cased, so `"Vacation section, line 4"` -> `"vacation"`. Taking the text AFTER the comma yields the detail (`"line 4"`), not the heading, and makes the matching rule fire for zero chunks, which degenerates every M2 metric to 0 or N/A. Do not leave the derivation to the implementer.

| Metric | Computation | Threshold is set from |
|---|---|---|
| Recall@k | relevant chunks found in top-k / total relevant | the cost of missing information for that question type: set per `question_type` segment, not system-wide, because base-case and multi-hop recall measure different abilities |
| Hit rate@k | % of cases where at least one relevant chunk is in top-k | what fraction of "cannot answer" outcomes users will tolerate |
| Precision@k | relevant chunks in top-k / k | usually report-only; becomes a threshold when token cost or context dilution measurably degrades generation |
| MRR | mean of 1/rank of first relevant chunk | how strongly generation quality depends on the best chunk's position (weak models degrade faster with low ranks) |
| nDCG@k | ranking quality, rewards relevant chunks near the top | relative comparison across retrieval configs; rarely a standalone pass/fail threshold |

Recall and hit rate matter most: generation can tolerate junk in the context (it can ignore it) but cannot recover from the answer simply not being there. Precision matters more as an efficiency concern (token cost, dilution) than a correctness one.

## Spec decisions the plan must pin down

Each of these is a concrete rule, not an open question. State the rule in the plan exactly as given here, adapted only where the note says it depends on the system. An implementer must not have to invent a derivation; the worker run on iteration 2 showed that an under-specified derivation gets invented backwards and degenerates every metric.

1. **Total relevant for Recall@k.** "Total relevant" is the count of distinct chunks that satisfy the matching rule for at least one source_ref, not the count of source_refs. State this.

2. **Aggregation method.** Every metric is reported two ways: per `question_type` segment, AND as an aggregate over all cases. The aggregate is a micro-average over all per-case values (sum of per-case numerators / sum of per-case denominators), not a macro-average of segment means. State both. When a segment has zero relevant chunks, Recall@k for that segment is 0/0: report N/A for that segment and exclude it from the aggregate so it does not crash the run. Per-case nDCG when total_relevant == 0 is defined as 0.0 by convention; state this.

3. **nDCG@k IDCG.** Use the standard IDCG: top-k positions filled with relevant items, i.e. IDCG@k = sum_{i=1..min(k, total_relevant)} 1/log2(i+1). State this.

4. **Retrieval tie-breaking.** Equal similarity scores produce ambiguous rankings and unstable MRR/nDCG. Break ties by chunk_id ascending so re-runs reproduce the same numbers. State this; do not leave tie-breaking to the implementer.

5. **Token-counting tokenizer.** Name the tokenizer used for chunk size and the p50/p95 metrics. For OpenAI embeddings, use tiktoken `cl100k_base`. For a stand-in or non-OpenAI system, name the actual tokenizer or state "whitespace split as a proxy". The choice changes the numbers; state it.

6. **Stage field semantics.** The gold set carries a `stage` field (`retrieval | generation | both`). M2 runs on every case regardless of `stage`; the field is informational about what the case was designed to test, not a filter for which metrics apply. State this so an implementer does not exclude `stage: generation` cases from M2.

7. **Chunk section_heading.** A chunk's `section_heading` is the `## ` heading at its start byte if the chunk begins with one; otherwise it is the nearest preceding `## ` heading in the document. Worked example: a chunk whose text starts with `## Vacation` has section_heading `vacation`; a chunk that starts mid-section inherits the `## ` heading above it. State this. Getting "at start byte" vs "nearest preceding" wrong makes every chunk inherit the previous section's heading.

8. **Precision@k micro form.** Per-case Precision@k = relevant_chunks_in_top_k / k. The aggregate is the micro-average: sum(relevant_chunks_in_top_k) / (k * n_cases). State this.

9. **Recorded retrieval results.** Every retrieval run records the full retrieved identity list per case (the ordered top-k chunk ids), not just the metrics derived from it. A metric table recomputed offline from a summary number can never be re-joined against the gold set when a matching rule needs revisiting; the recorded result list is the raw material every M2 metric and every provenance join (references/provenance-eval.md) consumes. State the trace field that holds it (for example `retrieval.result_chunk_ids`); it is the provenance field for every retrieval metric row.

## Segmenting results

Aggregate numbers hide systematic failures. Report every metric broken down by the gold set's `question_type` field. Common pattern: recall is fine on `factual`, collapses on `multi-hop`. That pattern points at retrieval strategy (query formulation), not at ranking tuning, and the segmentation is what makes it visible.

## Measurement flow

```mermaid
sequenceDiagram
    participant Q as Gold case
    participant R as Retriever
    participant G as Gold source_refs
    Q->>R: question
    R-->>R: top-k chunks; record ordered result id list
    R->>G: match chunks against source_refs
    G-->>G: compute recall@k, MRR, nDCG@k
    G-->>G: provenance join: which recorded results carry each anchor (attribution coverage + distribution)
```

Per-run cadence: every pipeline or corpus change, plus a smoke subset (10 cases) on model or embedding changes.

## Threshold guidance

Do not copy thresholds from here or anywhere else; this file deliberately contains no numbers. Set each threshold case-by-case in the pipeline session, from the dependency listed in the table, with the user, and record the reason in the plan. Thresholds set before the first real run are provisional: after that run, recalibrate to slightly below the observed score per segment, so regressions get caught without false alarms.