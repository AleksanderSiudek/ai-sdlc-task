---
name: evaluate-plan
description: Checks that TASK.md and context/PLAN.md exist, then launches the evaluator agent to produce context/PLAN_REVIEW.md
disable-model-invocation: true
context: fork
agent: plan-evaluator
background: false
allowed-tools: Bash(test *)
---

## Preflight

TASK.md present: !`test -f TASK.md && echo yes`
PLAN.md present: !`test -f context/PLAN.md && echo yes`

## Task

Review `context/PLAN.md` against `TASK.md` and write `context/PLAN_REVIEW.md`,
following the rules in your agent definition.