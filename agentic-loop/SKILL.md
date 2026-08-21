---
name: agentic-loop
description: Loop engineering reference — run one self-prompting agentic-loop cycle (orchestrator → discover → plan → fan-out specialists → verify against goal → synthesize → write memory), then optionally wire the weekly schedule. Reproduces the Builder/Scout/Growth demo and generalizes to any goal.
---

> Run this playbook: `iris playbook run agentic-loop `
> Steps: plan → build → scout → growth → verify → synthesize → write-memory → schedule → summary
# Agentic Loop (loop engineering)

A runnable reference for the "set the goal once, the agents prompt themselves" pattern:

```
GOAL → DISCOVER/PLAN → EXECUTE (Builder · Scout · Growth) → VERIFY → SHIP/ITERATE
   + MEMORY (next-steps, outside the conversation)   + weekly SCHEDULE
```

Each specialist below is a `prompt` step you can later swap for a real agent fanned out
across the Hive — `iris hive run <node> "iris agents chat <specialistId> '…' --bloq <mem>"`
— for true parallel execution. See `iris how-to view agentic-loops`.

All AI steps use **gpt-4.1-nano** (cheap, closed-loop economics). Memory persists to a
local next-steps file (the video's "memory outside the conversation") and, if `--bloq` is
given, is ingested into that knowledge base for recall next cycle.

## Steps


---
"""
with open(mem, "a") as f:
    f.write(entry)
print(f"Memory appended -> {mem}")
PY

BLOQ="${{args.bloq}}"
if [ -n "$BLOQ" ] && [ "$BLOQ" != "0" ] && [ "$BLOQ" != "null" ]; then
  echo "Ingesting memory into bloq $BLOQ for RAG recall next cycle…"
  iris bloqs ingest "$BLOQ" "$MEM" && echo "Ingested into bloq $BLOQ" || echo "(bloq ingest skipped — check the bloq id)"
else
  echo "No --bloq given; memory is the local file only. Pass --bloq <id> to make it RAG-recallable."
fi
```
