# MarketHub Provider Implementation Plan

> **For agentic workers:** Implement task-by-task. Spec: `docs/superpowers/specs/2026-08-10-markethub-provider-design.md`.

**Goal:** Parallel quote-then-route hub over InferHub + SurplusIntelligence, registered as `markethub`.

**Architecture:** Thin adapters + board; reuse `quote_route` + existing clients; tools remap once into IH/SI openai_compat paths.

### Task 1: Export `quote_route` on IH + SI
### Task 2: Build `src/providers/markethub/` core
### Task 3: Wire `client.py` + tools
### Task 4: Offline tests + smoke
