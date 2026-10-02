---
description: Mandatory stop before anything that calls a paid API (judge, hosted eval, RunPod).
---
Before launching any paid run:
1. State the run: script, question count, model, estimated EUR (use rag-service/scripts/run_costs.py for the last comparable run).
2. STOP and wait for an explicit yes from the operator. "Go ahead" on an earlier plan is not a yes for this run.
3. On yes: append the ledger row to data/eval/run-costs.jsonl FIRST, then launch.
Never start a pod or a judge run without steps 1-3. Never send user log text, secrets or invite links to a third-party model.
