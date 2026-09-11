---
name: agent-eval
description: Design and plan evaluations for agent pipelines that combine skills, MCP servers, and CLI tools. Produces two concrete artifacts, an eval set file (tasks with trigger expectations, expected behavior, negative controls) and an eval pipeline plan with measurement points, metric table with provenance fields, baseline/arm matrix, and thresholds. Covers task completion, trajectory and behavior checks, security, plus token and time efficiency as first-class metrics. Uses NVIDIA skillevaluator as the optional executor for skill arms. Use this skill whenever the user wants to evaluate an agent, skill, tool, MCP server, CLI workflow, or agent pipeline, measure lift versus a baseline, check whether the right skill or tool fired, judge trajectories or security behavior, or asks "how do I know my agent/skill/tools are working", even if they never say the word "eval".
---

# Agent Eval Design

This skill produces two files via two subagents. You orchestrate. The subagents write. Talking about evals without writing the files is a failure mode, not a partial result.

| Artifact | Subagent | Inputs |
|---|---|---|
| `eval_set.yaml` | eval-set writer | target surface (skill/MCP/CLI paths), agreed case distribution |
| `eval_plan.md` | pipeline planner | `eval_set.yaml` + user's agent architecture |

Dependency: the pipeline plan needs the eval set to exist, so the second subagent runs only after the first finishes and the user has reviewed the eval set.

This skill plans only. It does not run pipelines or execute code. Implementation and execution happen later, outside this skill — except where the plan explicitly delegates an arm's execution to the skillevaluator CLI (see "Executor: skillevaluator").

## Why two subagents, not one session

Eval set quality depends on thinking through triggering, behavior, and negative controls with no preconception of the pipeline. Pipeline design depends on the eval set existing as a real file, not a hypothesis. Doing both in your own context produces lazy eval sets designed to fit a preconceived pipeline, and a pipeline plan built around a hallucinated dataset. Subagents isolate the two head spaces; your main context stays clean for orchestration.

## References (the subagents read these, not you)

| Reference | Read by | When |
|---|---|---|
| `references/case-guide.md` | eval-set writer | before drafting any case. Defines the case types (task, implicit, contextual, negative, security, multi-step) with a worked YAML example of each. |
| `references/provenance-eval.md` | pipeline planner | always. Invariants on recording tool/stage results (not just calls) and attribution. |
| `references/trajectory-eval.md` | pipeline planner | when the plan touches triggering, workflow order, or behavior checks. |
| `references/task-eval.md` | pipeline planner | when the plan touches task completion or answer grading. |
| `references/efficiency-eval.md` | pipeline planner | always. Token, time, and tool-call productivity as first-class metrics. |
| `references/ablation.md` | pipeline planner | when the plan compares arms (skill on/off, MCP vs CLI, component variants). |
| `references/skillevaluator.md` | pipeline planner | when delegating execution of an arm to the skillevaluator CLI. |

You do not need to read these yourself. The subagents do.

## Your job as orchestrator

You do not write the artifacts yourself. You gather inputs from the user, brief each subagent, and hand off. Stay out of the writing. If you start drafting cases or metrics in your own context, stop and dispatch a subagent instead.

## Phase 1: eval set

### Orchestrator steps

1. Ask the user what the target surface is: the skill (name, path), MCP servers (names, versions, tool lists), CLI tools (names, versions, command surface). Read the SKILL.md and any MCP/CLI config yourself to get a feel for what the surface can and cannot do. You need just enough to brief the subagent.
2. Ask how the user will run the eval: live agent runs, skillevaluator (Harbor sandbox), or a lighter harness. This decides whether cases need sandbox-friendly phrasing.
3. Agree a case distribution with the user. Default: roughly 50% task, 15% implicit, 10% contextual, 15% negative, 5% security, 5% multi-step. Adjust to the surface; the point is deliberate coverage, not this exact ratio.
4. Agree an output path for `eval_set.yaml`. Default: next to the skill or in a `evals/` directory.
5. Dispatch a subagent with the Agent tool (`subagent_type: general`). Use the brief in "Subagent brief" below, filling in the angle-bracket fields.
6. When the subagent returns, read `eval_set.yaml` yourself. Show the user the first 5 cases. Get their feedback. If they want changes, re-dispatch the same subagent (via its task ID) with the specific fixes, not a fresh one.
7. Only proceed to Phase 2 once the user accepts the eval set. If the user wants to skip Phase 1 and go straight to Phase 2, refuse: a pipeline plan written without a real eval set is speculation, not a plan.

### Subagent brief

Paste this into the subagent prompt, filling the angle-bracket fields:

```
You are the eval-set writer subagent for the agent-eval skill. Do not do any pipeline design. Your only job is to write eval_set.yaml.

Read these files in full first:
- <skill-dir>/references/case-guide.md
- the schema and per-case invariants in the "Eval set schema" section of <skill-dir>/SKILL.md

Target surface:
- Skill: <name and path to SKILL.md, or "none">
- MCP servers: <names, tool lists, or "none">
- CLI tools: <names and command surface, or "none">
Read the SKILL.md and configs to learn what the surface can and cannot do, and what tools exist.

Will the eval run via: <live agent / skillevaluator Harbor sandbox / light harness>.

Write <count, 20 to 40> cases to <output path>/eval_set.yaml using the schema from SKILL.md. Use this case distribution: <distribution agreed with user>.

Per-case invariants (every case must satisfy these):
- The task reads like something a real user would type, not a synthetic instruction.
- expected_behavior lists observable actions (tool calls with arguments, files touched, outputs produced) that a reviewer can check against a trajectory, not vague goals.
- expected_output exists when the task has a checkable final artifact; omit it when correctness is behavioral only.
- Negative cases state which skill/tool must NOT fire, and what acceptable alternative behavior looks like.
- Security cases state the forbidden action (secret leak, destructive command, unauthorized access) precisely enough that a trace scan can check it.
- Any case whose pass condition is not obvious carries a grading_note telling the implementing agent what passing looks like.
- The file must parse as valid YAML. Free-text values containing `:` must be quoted, or `yaml.safe_load` fails. Before returning, verify with `python3 -c "import yaml; yaml.safe_load(open('<output path>/eval_set.yaml'))"`. If it raises, fix the quoting and re-check.

Use the Write tool to write the file. Do not return the YAML in your message, just confirm the file path and give a one-line summary: case count by category and by trigger.
```

## Phase 2: eval pipeline plan

### Orchestrator steps

8. Confirm the architecture: how the agent runs (which harness, which model), where traces land, what the skillevaluator role is if any. Read configs yourself until you can brief the subagent on the real pipeline shape.
9. Confirm the eval set path from Phase 1.
10. Dispatch a subagent with the Agent tool (`subagent_type: general`). Use the brief in "Subagent brief" below.
11. When the subagent returns, read `eval_plan.md` yourself. Show the user the structure: the measurement-points diagram, the metric table, the arm matrix, the thresholds, the build order. If they want changes, re-dispatch the same subagent with the specific fixes.
12. End by telling the user, in one line: wrote `eval_set.yaml` and `eval_plan.md`; implementation and runs happen outside this skill (except any arm delegated to skillevaluator, which the plan names explicitly).

### Subagent brief

```
You are the pipeline planner subagent for the agent-eval skill. Do not write any eval cases. Your only job is to write eval_plan.md.

Read these files in full first:
- <skill-dir>/SKILL.md (the "Writing style for artifacts" section)
- <skill-dir>/references/provenance-eval.md (always; its invariants apply to every metric)
- <skill-dir>/references/trajectory-eval.md (when the plan touches triggering, workflow order, behavior checks)
- <skill-dir>/references/task-eval.md (when the plan touches task completion grading)
- <skill-dir>/references/efficiency-eval.md (always; token and time metrics are first-class)
- <skill-dir>/references/ablation.md (when the plan compares arms)
- <skill-dir>/references/skillevaluator.md (when any arm's execution is delegated to skillevaluator)
- the eval set at <path to eval_set.yaml>
- the user's agent architecture: <path or description>

Write eval_plan.md to <path next to eval_set.yaml>. The plan must have these sections, in order:

1. Measurement points. A mermaid flowchart of the user's actual agent loop (task in, triggering, tool calls, final answer) with measurement points M1, M2, ... marked where data is captured. Each capture includes the call AND the result identity set per references/provenance-eval.md invariant 1.
2. Metric table. One row per metric with columns: measurement point, stage, metric name, how it is computed (formula, trace scan, or judge rubric, drawn from the stage reference for that stage), the eval set field it consumes, the provenance field it consumes, and whether it is a completion, trajectory, security, or efficiency metric. Every metric must trace to an eval set field and a provenance field (or be marked provenance-exempt with a reason, per provenance-eval.md invariant 2). Cost, token, and latency metrics are first-class rows, not an afterthought. Cut any metric with no data source.
3. Arms and baselines. The runs the plan requires: baseline arm(s) and treatment arm(s). Minimum viable: same task set with and without the skill (lift). If the user is deciding between components (skill vs MCP tool vs CLI, reranker vs none), apply references/ablation.md: one variable per variant, noise margins, stop rule, cost column. State which arm(s) run via skillevaluator and which run in the user's harness.
4. Thresholds and pass conditions. For every metric and threshold, state: the number; the reason it is that number (domain stakes, cost of a wrong answer, user requirement, or first-run calibration, never a number copied from a reference); whether the threshold is segmented per case category (a single system-wide pass target is usually wrong); how per-case pass criteria are applied (cases with a grading_note pass when their note is satisfied, not when an aggregate crosses a threshold); recalibration policy (what evidence updates thresholds after the first real run); regression behavior (fail the run, or flag for review). If a per-case criterion requires signals from a later measurement point than the arm captures, state that the arm cannot apply it.
5. Build order. Which measurement point to instrument first, chosen by cheapest-to-wire plus highest-signal. Usually: trace capture with result identities first (provenance-eval.md invariant 1), then triggering checks, then task grading, then efficiency rollups.

The references deliberately contain no threshold numbers. If you cannot derive a number from the user's domain stakes or requirements, write "TBD: needs user input" in that row and list what input is needed. Do not invent numbers.

The "Spec decisions the plan must pin down" sections in the references are concrete rules with worked examples, not suggestions. Copy each rule into the plan verbatim (adapt only the system-specific bits the rule itself names). Do not paraphrase, do not invent your own derivation, do not pick a different convention. A previous run showed that paraphrasing a spec rule produced a backwards derivation that made every metric degenerate; copying verbatim is the fix.

Use the Write tool to write the file. Do not return the plan in your message, just confirm the file path and give a one-line summary: section count, arm count, and any TBD rows.
```

## Eval set schema

```yaml
version: 1
created: 2026-09-10
target:
  name: my-agent-pipeline
  skill: skills/my-skill        # path to SKILL.md dir, or omit
  mcps: [filesystem, github]    # server names as configured, or omit
  clis: [gh, ffmpeg]            # tool names, or omit
  harness: live-agent           # live-agent | skillevaluator | custom
cases:
  - id: a001
    task: Find the largest CSV in ~/data and summarize its top row.
    trigger: must               # must | must-not | optional
    expected_behavior:
      - lists files in ~/data
      - runs a size/count comparison before opening any file
      - summarizes the first data row of the largest file
    expected_output: a one-paragraph summary of the header row
    tools_allowed: [fs, csvkit]
    category: task              # task | implicit | contextual | negative | security | multi-step
    difficulty: medium          # easy | medium | hard
    grading_note: only for cases whose pass condition is not obvious; see references/case-guide.md
```

## Writing style for artifacts

- Artifacts are handoff documents: an agent will load and implement them, and an engineer will review them. Write for both. Self-contained, no reliance on conversation context, exact field names, exact formulas, exact rubrics. "Assess agent quality" is not a metric.
- Every judgment call (a threshold, a case distribution, an arm choice, a rubric choice) states its reason inline. The implementing agent needs the decision; the reviewing engineer needs the reason.
- Name real tools, real servers, real commands from the user's system, not placeholders, wherever the user has told you them.