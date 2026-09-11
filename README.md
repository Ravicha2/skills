# Eval Skills

Skills for designing and running evaluations on AI agents and RAG systems.

## rag-eval

Design and plan evals for RAG / knowledge agents: gold sets, eval plans, and optional ablation matrices. Enforces result capture and provenance on every metric. Covers ingestion, retrieval, generation, and provenance eval, with gold-set case design and faithfulness judging.

```bash
npx skills add Ravicha2/skills@rag-eval
```

## agent-eval

Design and plan evaluations for agent pipelines that combine skills, MCP servers, and CLI tools. Produces an eval set file (tasks with trigger expectations, expected behavior, negative controls) and an eval pipeline plan with measurement points, metric table, baseline/arm matrix, and thresholds. Optional NVIDIA skillevaluator executor.

```bash
npx skills add Ravicha2/skills@agent-eval
```

## Install both

```bash
npx skills add Ravicha2/skills
```

## Feedback

Issues and PRs welcome — open an issue on this repo describing your use case and what felt off.

[![skills.sh](https://skills.sh/b/Ravicha2/skills)](https://skills.sh/Ravicha2/skills)