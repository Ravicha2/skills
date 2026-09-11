# Executor: skillevaluator

NVIDIA [SkillEvaluator](https://github.com/NVIDIA/SkillEvaluator) (`skillevaluator` CLI, installed via `uv tool install "skillevaluator[all] @ git+https://github.com/NVIDIA/SkillEvaluator"`) evaluates agent skills through Harbor sandboxed live runs. This skill plans the evaluation; skillevaluator can execute the skill-arm runs underneath. This file tells the planner what to delegate, what to keep, and what to translate.

## Division of labor

| Concern | Owner | Why |
|---|---|---|
| Eval set design (cases, distribution, negative/security cases) | this skill's eval set | skillevaluator's synthetic 4-bucket generator (explicit/implicit/contextual/negative) is a subset of the case guide; use the user's real surface instead |
| Metric table, provenance, thresholds, arm matrix | this skill's plan | skillevaluator scores 6 raw metrics mapped to 5 dimensions; the plan needs provenance fields and user thresholds it does not have |
| Skill-arm execution (sandboxed runs, trajectories, reward scores) | skillevaluator tier 3 | this is its strength; do not rebuild Harbor-style run collection |
| Baseline (without-skill) runs | skillevaluator, when the harness choice is acceptable | its lift methodology is exactly the minimal arm matrix (ablation.md) |
| MCP/CLI arms | the user's harness | skillevaluator sees MCP/CLI only as generic tool calls; per-tool attribution stays in the plan's provenance design |
| Token/time efficiency | skillevaluator reports token_efficiency unscored; the plan's efficiency-eval.md rows own it | first-class here |

## What the plan must translate

skillevaluator's dataset schema (`evals/evals.json`) differs from this skill's `eval_set.yaml`. When delegating an arm to skillevaluator, the plan adds a translation row per case: `question` ← `task`, `expected_output`/`ground_truth` ← `expected_output`, `expected_behavior` ← `expected_behavior`, `should_trigger` ← `trigger: must` / `must-not`. Cases with no skillevaluator equivalent (security-only traps, cases needing the user's MCP surface) stay in the user's harness arm; the plan lists which cases went which way and why — this is an arm-definition entry in the ablation.md sense.

## CLI surface the plan may reference

- `skillevaluator create-eval-dataset <skill-dir> --full` — 4-bucket synthetic dataset; useful as a smoke test, not as the eval set (generic cases, not the user's real surface).
- `skillevaluator tier3 evaluate <skill-dir>` — live sandboxed runs with trajectories, 6 raw metrics (security, skill_execution, skill_efficiency, accuracy, goal_accuracy, behavior_check) mapped to 5 dimensions (Security, Correctness, Discoverability, Effectiveness, Efficiency), with-skill/baseline lift.
- `skillevaluator compare` — results across agents.
- `skillevaluator view` / `harbor-view` — HTML report, trajectory browser.
- `skillevaluator validate <skill-dir> --harbor-contract` — run before delegating; fixes missing eval files and schema mismatches cheaply.

The 5-criterion accuracy rubric (tool identified, actions correct, factually accurate, task addressed, actionable) is the source of task-eval.md's adapted rubric; when skillevaluator runs the arm, its rubric result can stand in for the plan's answer-correctness row — the plan notes the substitution and keeps the provenance requirement: skillevaluator's judge sees the sandbox trace, which satisfies provenance-eval.md invariant 1 for that arm.

## Health and readiness

Before promising a delegated arm, the plan includes a readiness step: `skillevaluator health-check` (CLI + backend) and `skillevaluator doctor` (runtime). Full tier 3 needs a provider key and a sandbox backend (Docker, local OS, or cloud). If the user lacks these, the arm falls back to the user's harness and the plan says so in the arm table — a delegated arm that cannot run is a plan error, not an implementation surprise.

## What not to delegate

- The eval set itself: skillevaluator's synthetic generation is for bootstrap and smoke tests; the accepted eval set is the ruler (ablation.md rule 2) and is never replaced by a regenerated synthetic one.
- Negative and security grading for the user's surface: skillevaluator's `check_negative_case` covers skill-activation leakage; the plan's trace-scan rules (trajectory-eval.md spec decision 3) cover the user's actual MCP/CLI surface.
- Threshold decisions: skillevaluator reports scores and lift; thresholds and their reasons come from the plan's section 4, derived from the user's stakes.