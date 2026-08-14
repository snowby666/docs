# Surplus Intelligence Provider Implementation Plan

> **For agentic workers:** Execute task-by-task. Steps use checkbox syntax.

**Goal:** Ship `surplusintelligence` provider with Hybrid-C cheapest/low-latency/reliable routing, InferHub-parity wiring, and live-verified capabilities.

**Architecture:** In-memory catalog + order-book windows → `max_price_per_1m` + provider allow-lists → sticky provider family; chat via `SurplusIntelligenceClient`; tools via `openai_compat`.

**Tech Stack:** Python, aiohttp, orjson, FastAPI provider registry patterns already used by InferHub.

## Global Constraints

- Provider id exactly `surplusintelligence`
- Base URL `https://api.surplusintelligence.ai/v1`
- `max_price_per_1m` in USD; order book prices convert microdollars → USD
- Never put admin key in `valid_keys.txt`
- No `/min{N}` by default
- Strip `@effort` client-side before calling SI

---

### Task 1: Core package (params, models, routing, sticky, api)

**Files:**
- Create: `src/providers/prod/surplusintelligence/{__init__,params,models,routing,sticky,api}.py`
- Create: `src/providers/prod/surplusintelligence/valid_keys.txt`
- Create: `src/providers/prod/surplusintelligence/.gitignore`

- [x] Implement module files per design spec
- [x] Offline tests for effort + ladder + sticky (PASS)
- [x] Wire client + registry + openai_compat
- [x] Live probe smoke (8/8 PASS)

### Task 2: Offline tests

**Files:**
- Create: `src/providers/prod/surplusintelligence/tests/test_effort_offline.py`
- Create: `src/providers/prod/surplusintelligence/tests/test_routing_offline.py`

### Task 3: Registry wiring

**Files:**
- Modify: `src/providers/client.py`
- Modify: `src/tools/registry.py`
- Modify: `src/tools/providers/openai_compat.py`

### Task 4: Live probe

**Files:**
- Create: `src/providers/prod/surplusintelligence/probe.py`

---

Execution: inline in this session (user requested end-to-end implementation).
