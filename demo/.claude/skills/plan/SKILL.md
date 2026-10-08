---
name: plan
description: Validates that TASK.md exists, then launches the planner agent to produce context/PLAN.md
disable-model-invocation: true
context: fork
agent: planner
background: false
allowed-tools: Bash(test *)
---

## Preflight

TASK.md present: !`test -f TASK.md && echo yes`

## Task

Read `TASK.md` and inspect the repository, then produce `context/PLAN.md`
following the rules in your agent definition.

If `context/PLAN_REVIEW.md` exists, run in revision mode instead.