---
name: rag-eval
description: Design and plan evals for RAG/knowledge agents (gold set, eval plan, optional ablation matrix). Enforces result capture and provenance on every metric. Use for eval design, quality metrics or thresholds, faithfulness or hallucination judging, ablation studies, stage or tool attribution, or "how do I know my agent is working".
---

# Agent Eval Design

This skill produces two files via two subagents. You orchestrate. The subagents write. Talking about evals without writing the files is a failure mode, not a partial result.

| Artifact | Subagent | Inputs |
|---|---|---|
| `gold_set.yaml` | gold-set writer | source document path, agreed case distribution |
| `eval_plan.md` | pipeline planner | `gold_set.yaml` + user's system architecture |

Dependency: the pipeline plan needs the gold set to exist, so the second subagent runs only after the first finishes and the user has reviewed the gold set.

This skill plans only. It does not run pipelines or execute code. Implementation and execution happen later, outside this skill.

## Why two subagents, not one session

Gold set quality depends on careful reading of source documents with no preconception of the pipeline. Pipeline design depends on the gold set existing as a real file, not a hypothesis. Doing both in your own context produces lazy gold sets designed to fit a preconceived pipeline, and a pipeline plan built around a hallucinated dataset. Subagents isolate the two head spaces: each one sees only what its task needs, and your main context stays clean for orchestration.

## References (the subagents read these, not you)

| Reference | Read by | When |
|---|---|---|
| `references/gold-set-cases.md` | gold-set writer | before drafting any case. Defines the four case types (base, multi-hop, ambiguous, edge) with a worked YAML example of each. |
| `references/ingestion-eval.md` | pipeline planner | when the plan touches parsing, chunking, metadata. |
| `references/retrieval-eval.md` | pipeline planner | when the plan touches retrieval. |
| `references/generation-eval.md` | pipeline planner | when the plan touches answer generation. |
| `references/provenance-eval.md` | pipeline planner | always, before writing any metric table. Not optional. |
| `references/ablation.md` | pipeline planner | only when the orchestrator decided the plan needs an ablation matrix (step 8). |

You do not need to read these yourself. The subagents do.

## Your job as orchestrator

You do not write the artifacts yourself. You gather inputs from the user, brief each subagent, and hand off. Stay out of the writing. If you start drafting cases or metrics in your own context, stop and dispatch a subagent instead.

## Phase 1: gold set

### Orchestrator steps

1. Ask the user where the source documents live. Read a small sample yourself (one or two docs) to confirm the path and get a feel for structure: sections, tables, metadata fields. You need just enough to brief the subagent.
2. Agree a case distribution with the user. Default: roughly 65% base, 25% multi-hop, 10% ambiguous/edge. Adjust to the domain; the point is deliberate coverage, not this exact ratio. (Adversarial / not-in-corpus cases are deferred to a later version of this skill; do not emit them now.)
3. Agree an output path for `gold_set.yaml`. Default: next to the source documents.
4. Dispatch a subagent with the Agent tool (`subagent_type: general-purpose`). Use the brief in "Subagent brief" below, filling in the angle-bracket fields.
5. When the subagent returns, read `gold_set.yaml` yourself. Show the user the first 5 cases. Get their feedback. If they want changes, re-dispatch the same subagent (via SendMessage with its agent ID) with the specific fixes, not a fresh one.
6. Only proceed to Phase 2 once the user accepts the gold set. If the user wants to skip Phase 1 and go straight to Phase 2, refuse: a pipeline plan written without a real gold set is speculation, not a plan.

### Subagent brief

Paste this into the subagent prompt, filling the angle-bracket fields:

```
You are the gold-set writer subagent for the rag-eval skill. Do not do any pipeline design. Your only job is to write gold_set.yaml.

Read these files in full first:
- /Users/<user>/.claude/skills/rag-eval/references/gold-set-cases.md
- the schema and per-case invariants in the "Gold set schema" section of /Users/<user>/.claude/skills/rag-eval/SKILL.md

Source documents live at: <path>
Read a sample of them to learn the structure (sections, tables, metadata fields).

Write <count, 30 to 50> cases to <output path>/gold_set.yaml using the schema from SKILL.md. Use this case distribution: <distribution agreed with user>.

Per-case invariants (every case must satisfy these):
- The question reads like something a real user would type, not a keyword query.
- expected_answer quotes or tightly paraphrases the source. For multi-hop, the answer shows the reasoning chain so each hop can be checked.
- source_refs points at the document and the location inside it. Every case must be anchored; do not emit cases with empty source_refs.
- Any case whose correct behavior is not "reproduce expected_answer" (ambiguous, edge) carries a grading_note telling the implementing agent what passing looks like.
- The file must parse as valid YAML. Free-text values containing `:` (like `description:` or `expected_answer:`) must be quoted, or `yaml.safe_load` fails. Before returning, verify with `python3 -c "import yaml; yaml.safe_load(open('<output path>/gold_set.yaml'))"`. If it raises, fix the quoting and re-check.

Use the Write tool to write the file. Do not return the YAML in your message, just confirm the file path and give a one-line summary: case count by question_type.
```

## Phase 2: eval pipeline plan

### Orchestrator steps

7. Ask the user for their system architecture. If they have a diagram, a repo, or a config, read it yourself. You need to understand the real pipeline shape well enough to brief the subagent.
8. Decide whether the plan needs an ablation matrix. Ablation is a comparison design: the same frozen gold set run through pipeline variants that differ in one deliberate change, so a quality delta can be attributed to a component or parameter. It is necessary when the user is choosing between designs (reranker vs none, chunk size A vs B, hybrid vs dense), attributing a regression to a stage, or deciding whether a component earns its cost. It is scope creep for a first baseline or a fixed-pipeline health check, where evaluating one config is the whole job. If the trigger is not obvious from what the user said, ask them one question and default to no. When ablation applies, add the ablation block to the planner brief; when it does not, do not mention ablation in the brief at all, because a planner told to consider ablation will invent variants nobody asked for.
9. Confirm the gold set path from Phase 1.
10. Dispatch a subagent with the Agent tool (`subagent_type: general-purpose`). Use the brief in "Subagent brief" below.
11. When the subagent returns, read `eval_plan.md` yourself. Show the user the structure: the measurement-points diagram, the metric table, the thresholds, the build order, and the variant matrix if present. If they want changes, re-dispatch the same subagent with the specific fixes.
12. End by telling the user, in one line: wrote `gold_set.yaml` and `eval_plan.md`; implementation and runs happen outside this skill.

### Subagent brief

```
You are the pipeline planner subagent for the rag-eval skill. Do not write any gold set cases. Your only job is to write eval_plan.md.

Read these files in full first:
- /Users/<user>/.claude/skills/rag-eval/SKILL.md (the "Session 2" logic and the "Writing style for artifacts" section)
- /Users/<user>/.claude/skills/rag-eval/references/provenance-eval.md (in full; its two invariants and the provenance metric-table column are mandatory)
- /Users/<user>/.claude/skills/rag-eval/references/ingestion-eval.md
- /Users/<user>/.claude/skills/rag-eval/references/retrieval-eval.md
- /Users/<user>/.claude/skills/rag-eval/references/generation-eval.md
- the gold set at <path to gold_set.yaml>
- the user's system architecture: <path or description>

<Only when the orchestrator decided in step 8 that ablation applies, add this block; otherwise omit it entirely.>
Ablation: this eval exists to decide <the user's actual decision, e.g. "whether the cohere reranker earns its added latency">. Also read /Users/<user>/.claude/skills/rag-eval/references/ablation.md in full before writing, and add a section 5 "Variant matrix" to the plan following its format and rules. Sections 1 to 4 are unchanged: the metric table, thresholds, and matching rules apply to every variant identically.
<End ablation block.>

Write eval_plan.md to <path next to gold_set.yaml>. The plan must have these sections, in order:

1. Measurement points. A mermaid flowchart of the user's actual pipeline (from the architecture you read) with measurement points M1, M2, ... marked where data is captured. Every measurement point that captures a tool, retriever, or stage call also captures that call's result identity set (the ids of what came back, per provenance-eval.md invariant 1). A call recorded with only name and arguments is metadata, not evidence.
2. Metric table. One row per metric with columns: measurement point, stage, metric name, how it is computed (formula or judge rubric, drawn from the stage reference for that stage), the gold set field it consumes, the provenance field it consumes. The provenance field names the trace field recording which call's result supplied the evidence behind the metric (per provenance-eval.md invariant 2); cost, latency, and count-only metrics may be marked `provenance-exempt` with a one-line reason, but any quality metric on matches, answers, rankings, citations, or false positives may not. Every metric must trace to a gold set field. Cut any metric with no data source. If a metric's provenance field does not exist in the user's trace yet, it becomes a build-order row: instrument the capture first.
3. Thresholds and pass conditions. For every metric and threshold, state: the number; the reason it is that number (domain stakes, cost of a wrong answer, user requirement, or first-run calibration, never a number copied from a reference); whether the threshold is segmented per question_type (a single system-wide recall target is usually wrong); how per-case pass criteria are applied (cases with a grading_note pass when their note is satisfied, not when an aggregate crosses a threshold); recalibration policy (what evidence updates thresholds after the first real run); regression behavior (fail the run, or flag for review). Also note explicitly: any per-case pass criterion that requires M3 signals (answer correctness, faithfulness) cannot be applied during an M2-only run, so an M2-only implementer cannot produce per-case pass/fail from M2 alone. The plan must say this so the implementer does not guess.
4. Build order. Which measurement point to instrument first, chosen by cheapest-to-wire plus highest-signal. Instrument result capture (invariant 1) at a measurement point before any metric that consumes that measurement point's provenance field; provenance joins are useless over a trace that recorded calls only. Retrieval metrics usually go first.

The references deliberately contain no threshold numbers. If you cannot derive a number from the user's domain stakes or requirements, write "TBD: needs user input" in that row and list what input is needed. Do not invent numbers.

The "Spec decisions the plan must pin down" sections in references/provenance-eval.md, references/retrieval-eval.md, and references/ingestion-eval.md are concrete rules with worked examples, not suggestions. Copy each rule into the plan verbatim (adapt only the system-specific bits the rule itself names, like the tokenizer or the target token count). Do not paraphrase, do not invent your own derivation, do not pick a different convention. A previous run showed that paraphrasing a spec rule produced a backwards derivation that made every metric degenerate; copying verbatim is the fix.

Use the Write tool to write the file. Do not return the plan in your message, just confirm the file path and give a one-line summary: section count and any TBD rows.
```

## Gold set schema

```yaml
version: 1
created: 2026-08-29
source_documents:
  - path: docs/handbook.pdf
    description: employee handbook, 40 pages
cases:
  - id: q001
    question: How many vacation days do employees get after their first year?
    expected_answer: 15 days, increasing by 1 per year of service up to 20.
    source_refs:
      - doc: docs/handbook.pdf
        location: "p.12, Benefits section"
    stage: generation        # retrieval | generation | both
    difficulty: easy          # easy | medium | hard
    question_type: factual    # factual | multi-hop | ambiguous | edge
    grading_note: only for cases whose correct behavior is not "reproduce expected_answer"; see references/gold-set-cases.md
```

## Writing style for artifacts

- Artifacts are handoff documents: an agent will load and implement them, and an engineer will review them. Write for both. Self-contained, no reliance on conversation context, exact field names, exact formulas, exact rubrics. "Assess answer quality" is not a metric.
- Every judgment call (a threshold, a case distribution, a rubric choice) states its reason inline. The implementing agent needs the decision; the reviewing engineer needs the reason.
- Name real files and real fields from the user's system, not placeholders, wherever the user has told you them.