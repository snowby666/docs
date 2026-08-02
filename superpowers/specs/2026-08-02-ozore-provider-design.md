# Ozore Provider Design

**Date:** 2026-08-02  
**Status:** Approved (design dialogue)  
**Repo:** uvicorn-electron

## Goal

Add an OpenAI-compatible **Ozore** provider (`https://ozore.com/v1/chat/completions`) with Bearer auth, register it for chat + native tools, keep a **local** model catalog under `src/providers/ozore/`, and live-probe every model for vision, reasoning params, tool calling (simple→agentic), and context limits (150k→300k @ concurrency 5).

## Non-goals

- Do **not** edit `src/utils/models.json`.
- Do not invent a custom tool adapter unless live probes prove Ozore does not return OpenAI-style `tool_calls`.
- Do not require JWT/account login (static Bearer key only).

## Architecture

Mirror **Verboo / Albert**: thin streaming client + shared `openai_compat` tool path.

```
Chat route (source=ozore)
  ├─ plain chat  → OzoreClient.send_message (SSE)
  └─ tools       → openai_compat_tool_chunks (registry: ozore → openai_compat)
Local catalog    → src/providers/ozore/models.json (not loaded into AI_MODELS)
Probes           → probe.py + limits_probe.py → JSON results in ozore folder
```

### Endpoint & auth

| Item | Value |
|------|--------|
| Base | `https://ozore.com/v1` (env override `OZORE_API_BASE`) |
| Chat | `{base}/chat/completions` |
| Auth | `Authorization: Bearer <key>` |
| Keys | Env `OZORE_API_KEYS` (comma-separated), default includes the provided `oz-...` key |
| Rotation | Round-robin / next-key per attempt |

### Client (`OzoreClient.send_message`)

- Payload: `{model, messages, stream: true, stream_options: {include_usage: true}}` plus optional `temperature` / `top_p` / `top_k` / `logit_bias` when provided.
- Forward messages **verbatim** (OpenAI multimodal `image_url` parts stay as-is). Ignore `file_path` (images already inlined by routes).
- Parse SSE `data: {...}` lines; stop on `[DONE]`.
- Reasoning: read `delta.reasoning` / `delta.reasoning_content` (and choice-level if present); wrap in `<think>...</think>` for normal streaming; honor `thinking=True` split if implemented consistently with peers.
- **Exponential retry** (required):
  - Retry on: HTTP **429**, **5xx**, connect/timeouts, and **empty completed streams** (no content and no reasoning after a finished response).
  - Prefer honoring `Retry-After` on 429 when present; otherwise `sleep(2^(attempt+1) + jitter)`.
  - Max attempts: **5**.
  - **Never retry after any chunk has been yielded** (no duplicate partial output).
  - Rotate API key on each retry when multiple keys exist.

### Registration

| File | Change |
|------|--------|
| `src/providers/ozore/__init__.py` | Export `OzoreClient` |
| `src/providers/ozore/api.py` | Client + `API_KEYS` + `CHAT_URL` |
| `src/providers/client.py` | `"ozore": lambda: OzoreClient()` |
| `src/tools/registry.py` | `"ozore": "openai_compat"` + `get_provider_config` → `CHAT_URL` + `API_KEYS` |
| `src/utils/models.json` | **Untouched** |

### Local catalog (`src/providers/ozore/models.json`)

Seed from the upstream `/v1/models` list provided by the user, including:

- `auto`, `ozore/fusion`, `ozore/auto-fusion`
- All `ozore/<pin>` model ids

Each entry stores at least: upstream `id` as `baseModel`, `description`, `owned_by`, advertised `context_window` / `input`, plus probe-filled fields:

- `tokens` — measured max accepted context (or advertised if probe inconclusive)
- `metadata.vision` / `function_call` / `reasoning` — from live probes
- Optional notes for unsupported request params (`reasoning_effort`, `reasoning_tokens`, `chat_template`, etc.)

Catalog is for operator/docs/probe consumption only until a later decision to promote rows into the global catalog.

## Capability probing

### Phase A — `probe.py` (capabilities)

Run against a representative matrix (at least one model per family + `auto`), then spot-check remaining ids for basic chat smoke:

1. Plain streaming chat
2. Vision: small PNG as `image_url` data URI; confirm color/content recognition
3. Reasoning surface: whether separate reasoning deltas appear; try `reasoning_effort`, `reasoning_tokens`, `chat_template` (and similar) — record accept vs ignore vs error
4. Tools (native OpenAI shape):
   - simple function call
   - tool round-trip (assistant tool_calls → tool result → final answer)
   - parallel tools
   - `tool_choice` forced/required
   - short agentic multi-step
5. Persist `probe_results.json` under `src/providers/ozore/`

### Phase B — `limits_probe.py` (context)

- **Every** model in the local catalog
- Token ladder: **150k → 300k** (e.g. 150000, 175000, 200000, 225000, 250000, 275000, 300000)
- Concurrency: **5**
- For each size: accept/reject; on accept, needle recall (front/mid/back as practical)
- Persist `limits_probe_results.json`; update local `models.json` `tokens` / notes from results

## Error handling & safety

- Log HTTP status + truncated body on non-200
- Do not leak full API keys in logs
- Large probe result artifacts may be gitignored if oversized; keep summary JSON

## Success criteria

1. `OzoreClient` streams chat successfully with exponential retry behavior as specified.
2. Provider registered in `client.py` and tools registry as `openai_compat`.
3. Local `models.json` lists all provided models with metadata updated from probes.
4. Probe reports cover vision, reasoning params, tool ladder, and per-model context 150k–300k @ concurrency 5.
5. `src/utils/models.json` has **zero** edits from this work.

## Out of scope (later)

- Promoting ozore models into the public/global `AI_MODELS` catalog
- Pricing / billing coefficients for ozore rows
- Custom Fusion/Auto-Fusion dashboard configuration beyond calling those model ids
