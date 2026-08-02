# Ozore Provider Implementation Plan

> **For agentic workers:** Execute inline in this session. Spec: `docs/superpowers/specs/2026-08-02-ozore-provider-design.md`.

**Goal:** Add OpenAI-compatible Ozore provider with local catalog, exponential retries, capability + context probes — without touching `src/utils/models.json`.

**Architecture:** Verboo-style Bearer SSE client; tools via `openai_compat`; local `models.json` + `probe.py` + `limits_probe.py`.

**Tech Stack:** aiohttp, orjson, asyncio, loguru

## Global Constraints

- Do **not** edit `src/utils/models.json`
- Endpoint: `https://ozore.com/v1/chat/completions`
- Auth: Bearer (env `OZORE_API_KEYS`, default key from design dialogue)
- `send_message` retries: max 5, exponential backoff on 429/5xx/timeouts/empty streams; honor `Retry-After`; never retry after yielding chunks
- Context probe: 150k→300k ladder, concurrency 5, every model
- Commit only if user asks

---

### Task 1: OzoreClient

**Files:**
- Create: `src/providers/ozore/api.py`
- Create: `src/providers/ozore/__init__.py`

- [x] Implement client (see implementation)
- [x] Export `OzoreClient`

### Task 2: Local catalog

**Files:**
- Create: `src/providers/ozore/models.json`

- [x] Seed all upstream model ids from user list

### Task 3: Registration

**Files:**
- Modify: `src/providers/client.py`
- Modify: `src/tools/registry.py`

- [x] `"ozore": lambda: OzoreClient()`
- [x] `"ozore": "openai_compat"` + `get_provider_config`

### Task 4: Capability probe

**Files:**
- Create: `src/providers/ozore/probe.py`

- [x] Vision, reasoning params, tools simple→agentic

### Task 5: Limits probe

**Files:**
- Create: `src/providers/ozore/limits_probe.py`

- [x] 150k–300k, concurrency 5, all models

### Task 6: Live run + update catalog

- [ ] Run probes; write results JSON; update local `models.json` metadata
