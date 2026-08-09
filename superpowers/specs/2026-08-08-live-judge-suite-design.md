# Live tool-judge suite design (2026-08-08)

## Goal
Exercise Gonka MiniMax-M2.7 → Kilo hy3 tool-judge on complex agentic edge cases, plus Venice/VeniceDev pipelines with scripted upstream and live judge backstop. Score strictly (schema-valid tools / exact labels). No soft passes.

## Layout
- `live_judge_core_bench.py` — collector, continuation modes, RAG classify
- `live_judge_pipeline_bench.py` — Venice (+ VeniceDev) e2e FakeClient + live judge
- `live_judge_bench.py` — runs both, one JSON report

## Cleanup in same pass
- Dedupe judge hook resolution; unify continuation token cap alias; fix stale PartyRock judge logs
