# Remaining Runtime Rework

**Date:** 2026-08-15  
**Status:** Task 0 (stream hop 300 → 600) shipped. Tasks 1–10 are the remaining rework.  
**Scope:** the 39 registered keys that are not on `AttemptRuntime` and were not inner-SSE extracted in Waves 1–5  
**Parent:** `docs/superpowers/specs/2026-08-15-universal-runtime-per-provider.md`  
**Plan:** `docs/superpowers/plans/2026-08-15-remaining-runtime-rework.md`

This document is the file-level rework for what is left after the per-provider extract waves. It does not invent overflow destinations, new codecs, or catalog rows.

## Problem

`AttemptRuntime.chat_chunks` is legal only for gonka/dahl. Waves 1–5 extracted matching-cursor auth, Verboo/thin/partial inner SSE, and the InferHub/SI session factory. Thirty-nine live keys remain. None of them match the gonka/dahl HTTP ladder, so wrapping them is the rejected Approach C.

The remaining work is four independent slices plus explicit no-ops:

| Slice | Count | Owns |
|---|---|---|
| A — extract-sse | 5 | crofai → zai 200-path → openrouter → smolproxy chat → jewproxy chat last |
| B — timeout policy | cross-cutting | Named 600s hop budget (shipped). Tools keepalives. Do not raise quote/e2b/free-queue. |
| C — overflow helpers | 4 maps | Move zai / ionet / partyrock dest / jewproxy-verboo predicate into `runtime/overflow/` |
| D — new leaves | 6 | Later spec: `runtime/responses/`, curl_cffi iterator, vermal dual-codec. `cfplayground` is deprecated (keep `cf`). |
| E — leave | 23 | first_party, remaining dialect, prompt_sim, guest, media |

## Architecture

```
routes (chat / messages / responses)
        │
        ├─ gonka/dahl chat → AttemptRuntime (unchanged)
        ├─ already-extracted OpenAI SSE → existing shell + yield_openai_sse_deltas
        ├─ remaining OpenAI SSE chat → same extract, keep outer shell
        │     simple 200 body → yield_openai_sse_deltas
        │     mid-stream error/retry (jewproxy) → per-frame iter_openai_choice_chunks
        ├─ OpenAI SSE tools → openai_compat + SSE comment keepalives
        └─ other codec → plugin (profile row only)
```

Two codecs. One HTTP ladder. Plugins stay plugins.

### Per-frame emitter (new, Task 1)

`yield_openai_sse_deltas` owns the line loop. JewProxy cannot use that loop: it inspects each parsed frame for `_proxy_error_text` before thinking/content. Share the **frame** instead:

```
iter_openai_choice_chunks(data, think, progress, held_usage) -> Iterator[dict]
yield_openai_sse_deltas(...)                                  # splits lines, then calls the frame emitter
```

The stream helper must also split coalesced `data:` lines inside one aiohttp chunk. Today's helper treats the whole chunk as one line; Z.AI already splits on `\n` because upstream coalesces.

List-shaped `delta.content` (OpenRouter / SmolProxy / JewProxy) is flattened to a string inside the frame emitter. `◁`/`▷` replacement stays local to OpenRouter / JewProxy after the yield.

Think close tag stays the helper's `"\n</think>\n\n"`. Callers that used `"</think>\n\n"` accept that delta; do not add a close-tag flag.

Usage: the emitter **holds** usage in `held_usage` (same as today's helper). Callers that used to yield usage live flush `held_usage` after a successful attempt. Empty retries discard the list.

### When extract is legal

| Client | Extract how | Keep |
|---|---|---|
| crofai | helper inside `_stream_once` when `thinking` is False | empty-retry shell, GLM→Z.AI, `thinking=True` → `{"thinking":}` |
| zai | helper on HTTP 200 body only | `_run_fallbacks` after the POST context exits; media; `include_usage` already present |
| openrouter | helper on HTTP 200 body | key shuffle / used_keys; `◁`/`▷` post-filter on reasoning **and** content yields; `max_tokens` 0 |
| smolproxy | helper on chat 200 body | Responses / xAI multi-agent forks; param pin/drop; empty-retry via `progress.emitted_any` |
| jewproxy | **per-frame emitter only**, after `_proxy_error_text` is None, and only when `prompt_thinking` is False | Responses fork; SmolProxy fallback; Verboo DeepSeek hop; `prompt_thinking` stays on `convert_prompt_think` |

Do not call `AttemptRuntime.chat_chunks` for any of these.

## Timeout policy (Slice B)

`STREAM_TOTAL_SECONDS = 600` in `src/providers/runtime/timeouts.py` is the named hop budget.

**Shipped this session (Task 0):**

- Route `timeout_duration` in chat / messages / responses uses the constant.
- `marketplace_session` default `total` uses the constant (InferHub / SI chat **and** tools).
- Live client `request_timeout` defaults: inferhub, surplusintelligence, kilo, telnyx, k2think, modal, neuralwatt, zeroeval, jatevo, akash, markethub, googleapi.
- Hardcoded recreate totals: k2think, akash.
- Salesforce chat POST `total=600` (mint stays 60).
- NoteGPT `sock_read=600`. CF Playground was raised in Task 0, then deprecated (keep `cf`).

The route 600 is still one `asyncio.wait` cycle racing the 10s heartbeat. Heartbeat almost always wins, so the route path still almost never fires. Raising it is correct for the rare hang where no heartbeat task is scheduled; it is not the tools fix.

**Still open:**

- Native tools return a bare `StreamingResponse(tool_gen)` and never enter `generate_chunks`. Cloudflare / Koyeb idle (~100s) kills silent tools streams. Task 8 wraps that generator with SSE comment keepalives (`: keepalive\n\n`).
- Free-queue slot TTL stays **180s**. DevPass stays **600s**. Do not raise free-queue in this rework (worker contract + stuck-slot leak tradeoff). Separate spec if product wants it.
- Do not raise markethub quote `total=30`, e2b/mistral **180**, `/v1/models` cache TTL 300, probes, or deprecated clients.
- Do not raise ArliAI image `sock_read=300` (media download, not chat).

## Overflow helpers (Slice C)

Move predicates and dest maps only. Dispatch / `forward_agen` / client construction stay in the provider.

| Source | Move to | Symbols |
|---|---|---|
| `zai/api.py` | `runtime/overflow/zai.py` | `ZAI_FALLBACK_MODEL_MAP`, `map_zai_to_fallback`, `should_run_zai_fallback` |
| `ionet/api.py` | `runtime/overflow/ionet.py` | `IONET_NVIDIA_FALLBACK_MAP`, `resolve_ionet_nvidia_fallback` |
| `partyrock/api.py` | `runtime/overflow/partyrock.py` | `PARTYROCK_ZAI_FALLBACK_MODEL = "glm-4.6"` |
| `jewproxy/api.py` | `runtime/overflow/jewproxy.py` | `should_verboo_fallback_deepseek_context` (and `is_jewproxy_context_limit_error` if it has no other callers that must stay local) |

Telnyx already lives in `src/providers/prod/telnyx/overflow_fallback.py`. Do not move it again.

Re-export new symbols from `runtime/overflow/__init__.py`. Provider modules import from there and keep the old private names as aliases if existing tests patch `_map_zai_to_fallback`.

Do not invent destinations. Do not unify Z.AI's chain with Vermal/CrofAI.

## New leaves (Slice D) — later spec only

Do not start these in this plan:

- `runtime/responses/` for OpenAI + jewproxy/smolproxy Codex
- curl_cffi iterator for chutes (and later venice NDJSON)
- vermal dual-codec
- ~~cfplayground relay / Agents WS~~ — deprecated 2026-08-15; `cf` stays
- comparia dual-lane (stays a plugin even after a later spec)

## Leave (Slice E)

first_party (google, mistral, cohere, anthropic), remaining dialect (neuralwatt, hyperfusion), prompt_sim (salesforce, sakana, e2b, partyrock wire), guest (akash, athina, hotbot, jatevo, meituan, neuro, notegpt, exa, sambanova), media (grok, novelai, runware, higgsfield), k2think playground.

## Global constraints

Copy these into every task. Do not weaken them.

- Do not invent overflow destinations.
- Do not call `AttemptRuntime.chat_chunks` except gonka/dahl.
- Do not set `stream_mode="buffered"` except verboo / inferhub / surplusintelligence.
- Do not add `stream_options.include_usage` where it is absent.
- `thinking=True` clients that yield `{"thinking": ...}` never call `yield_openai_sse_deltas` on that branch (CrofAI, Albert, Feather, ArliAI, IoNet).
- Overflow paths that use `forward_agen` keep it. Z.AI fallbacks run **after** the primary POST context exits.
- Do not flatten `src/providers/runtime/`. New helpers go under `sse/`, `overflow/`, or `timeouts.py`.
- Do not revive `sse_chat.py` / `key_ring.py`.
- Do not edit `models.json`. Do not revert the 16 deprecations.
- INDEX only via `build_index_inventory` + `generate_indexes` + `verify_indexes`.
- Do not swap Albert / Feather / ArliAI onto `RoundRobinKeys`.
- Do not delete `ProviderFlags` bools.
- Do not raise free-queue 180, markethub quote 30, e2b/mistral 180, or models-cache 300.
- JewProxy / SmolProxy `/v1/responses` paths are out of scope.
- Stop after Task 1 for review. Do not start Tasks 2–6 in the same session as Task 1.
