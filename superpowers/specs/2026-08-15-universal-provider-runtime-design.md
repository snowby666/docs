# Universal Provider Runtime — Design

**Date:** 2026-08-15  
**Status:** Tasks 0–7 shipped (gonka/dahl chat on `AttemptRuntime`). Follow-on `2026-08-15-universal-runtime-per-provider.md` Tasks 1–7 shipped (57 live providers). Waves 6/8/9/10 are no-ops.  
**Scope:** `src/providers/` + `src/tools/` (chat, messages, responses, native tools)  
**Non-goals:** catalog 200k clamp, adding Gonka to `PROVIDER_STREAM_USAGE`, rewriting every `__main__` demo, inventing new overflow destinations

## Problem (measured, not invented)

Two stacks already exist for the same upstreams:

| Path | Entry | HTTP owner | Output |
|------|--------|------------|--------|
| Normal chat | `Client.send_message` | per-request `aiohttp.ClientSession` on the client | `{"response"}` / `{"usage"}` |
| Native tools | `openai_compat_tool_chunks` | `open_compat_session(flags)` | raw OpenAI SSE / JSON |

They share config constants (`CHAT_URL`, keys) but **reimplement** thinking payload, empty-visible retry, HTTP overflow, and `[DONE]`/usage policy. That is why Gonka chat recovered empty-visible live while tools streaming still forwards the first `[DONE]` before retry.

`ProviderFlags` is nine `provider == "..."` booleans. Adding Dahl required a new flag, a new fallback wrapper, and loop branches. The next Gonka-network broker will copy that again.

Process-global state is real and scattered: Dahl `_token` + lock, Gonka `_CURRENT_INDEX`, HyperFusion RR, PromptLayer bearers, Telnyx `AI_MODELS` mutation. Clients are already constructed per `__get_client` call (not a client singleton). The globals are **shared caches**, hidden as module variables.

`sse_chat.py` / `key_ring.py` have **no source** — only stale `.pyc`. Do not revive them as names without a new design.

INDEX: Dahl is **already** in both generated indexes (`providers/INDEX.md` row + `tools/providers/INDEX.md` handler map). “Update INDEX” means regenerate after registry/profile changes via `generate_indexes.py`, not hand-edit a new row.

## Research that constrains the design

- **LiteLLM (2025–2026):** hardcoded model-substring checks keep missing new ids. They moved to capability flags on a registry (`ProviderCapabilities` / `supports_*` in the model map). Custom handlers must be dispatched **before** name matching or they are silently bypassed ([PR 30032](https://github.com/BerriAI/litellm/pull/30032), [PR 21379](https://github.com/BerriAI/litellm/pull/21379), [issue 23352](https://github.com/BerriAI/litellm/issues/23352)).
- **llm-open-proxy / llm_adapter:** one internal request model, protocol codecs, and a **fallback router** that is host-owned — not `if provider ==` inside every stream loop. Retry 429/5xx/404; do not retry 400/401/403/422 as “another upstream.”
- **OpenAI-compat adapters:** `max_tokens` vs `max_completion_tokens` and thinking flags are **profile fields**, not chat-loop special cases ([opencode #25096](https://github.com/anomalyco/opencode/issues/25096)).

Implication for this repo: a new broker is a **profile + optional hooks**, not a new `flags.dahl` and a second overflow file.

## Approaches

### A — ProviderProfile + one AttemptRuntime (recommended)

Every registered chat/tools provider declares a `ProviderProfile` (data). One `AttemptRuntime` owns: POST, SSE iterate, `StreamProgress`, hold-until-accepted (`usage` + `[DONE]`), thinking-off retry, HTTP overflow map, `forward_agen` teardown.

- Chat adapter: runtime → `{"response"}` chunks (existing think wrap stays in one SSE helper).
- Tools adapter: runtime → OpenAI SSE bytes (existing finalize / tool_calls).
- Same predicates, same overflow map, same hold policy.

### B — Keep two stacks, extract more helpers

What we just did. Helpers (`should_overflow`, `gonka_family_overflow`) reduce copies but loop.py and `send_message` still diverge. Tools `[DONE]` bug is the proof this is not enough.

### C — Force all chat through `openai_compat`

Rejected. Chat must wrap `<think>`, filter `REASONING_MODE_EXCLUDE`, and yield route-shaped chunks. Tools must emit native `tool_calls` SSE and **bypass** `chat.py` reasoning remappers (INDEX already documents this). One HTTP runtime, two **codecs** — not one wire format.

## Recommended architecture

```
routes (chat / messages / responses)
        │
        ├─ no tools → ProviderClient.send_message
        │                 │
        │                 └─ AttemptRuntime(profile) ──► upstream
        │
        └─ tools → native_tool_* → handler
                      openai_compat → AttemptRuntime(profile) ──► same upstream
```

### ProviderProfile (data, opt-in, defaults False)

```python
@dataclass(frozen=True)
class ProviderProfile:
    name: str
    tool_type: Optional[str]          # None | openai_compat | jewproxy | ...
    stream_mode: str                  # passthrough | buffered | kilo
    retry_empty_visible: bool = False
    hold_terminal: bool = False       # hold usage + [DONE] until attempt accepted
    seed_bust: bool = False
    thinking_default: Optional[bool] = None  # None = omit; True/False = explicit
    http_overflow: Optional[OverflowSpec] = None
    auth: AuthKind = AuthKind.bearer_keys
```

`flags_for("gonka")` becomes `profile_for("gonka")`. Loop branches on `profile.retry_empty_visible`, not `flags.gonka_family`.

Overflow is data:

```python
OverflowSpec(
    http_predicate=should_fallback_to_hyperfusion,  # or dahl's CF-502 extender
    resolve=resolve_gonka_overflow,                 # → (provider, model)
)
```

A third Gonka broker adds a profile row + optional predicate. No new `flags.xyz`.

### AttemptRuntime (instance, not singleton)

Constructed per request with:

- `profile`
- `session` (injected; caller owns close)
- `auth` object (injected `KeyRing` / `TokenStore` **instance**, not module `_token`)
- `progress: StreamProgress`

Module-level Dahl `_token` becomes `DahlTokenStore` with an explicit shared cache **passed in** (process cache is allowed; hiding it in the module is not). Gonka `_CURRENT_INDEX` becomes `RoundRobinKeys(keys)` on the client or store.

### Codecs (two, not N)

- `chat_codec`: `yield_openai_sse_deltas` (already shared by Gonka/Dahl)
- `tools_codec`: `stream_passthrough` / `stream_buffered` / `stream_kilo` — but passthrough **must** honor `hold_terminal` when `retry_empty_visible`

### Folder layout (move, don’t invent new product features)

```
src/providers/
  runtime/
    profile.py          # ProviderProfile, profile_for, OVERFLOW specs
    progress.py         # move stream_progress.py
    usage.py            # move stream_usage.py
    send_kwargs.py      # move
    sse.py              # yield_openai_sse_deltas, ThinkWrap
    overflow.py         # resolve maps + http predicates used by chat AND tools
    auth/
      keys.py           # RoundRobinKeys instance
      token_store.py    # injectable cache protocol
  client.py             # providers_map_actions only
  prod/…                # thin clients: URL + profile name + quirks
src/tools/providers/openai_compat/
  loop.py               # uses profile_for + AttemptRuntime
  providers/flags.py    # thin shim → profile_for (delete after callers move)
```

Dead: do not recreate `sse_chat.py` / `key_ring.py` at the old paths. New auth lives under `runtime/auth/`.

### Consistency rules (chat ↔ tools)

1. Empty-visible: same `should_overflow` + one thinking-off retry + same overflow map.
2. Hold `usage` and tools `[DONE]` until the attempt is accepted. Drop on retry/overflow.
3. Do not yield a dangling `</think>` before the retry/overflow **decision**; after deciding to retry, close think, then stream the retry.
4. `GeneratorExit` / `CancelledError` are never fallback failures. Nested gens use `async with forward_agen`.
5. 402 on Dahl is remint on **every** POST (primary and thinking-off), never overflow.
6. `hyperfusion_fallback=False` (tool-judge) = thinking-off only, no Verboo/HF.
7. Do not change `stream_buffered` flush for Verboo/IH/SI (live CoT).

### Live verification (after wiring, concurrency 15)

Existing harnesses — do not invent a new matrix:

- `python -m src.providers.tests.live_cache_matrix --concurrency 15`
- `python -m src.providers.tests.live_usage_surface --concurrency 15`
- `python -m src.tools.providers.openai_compat.tests.live_tools_matrix --concurrency 15`

54/66 prod folders have no dedicated `live_*.py`. The cross-provider matrices **are** the full-suite. Per-provider scripts stay for Gonka/Dahl/Telnyx edge probes.

## Success criteria

- Adding a Gonka-like broker: profile + predicate, zero new `flags.*`, zero new `fallbacks/foo.py` if the overflow map is the same.
- Chat empty-visible and tools empty-visible produce the same retry/overflow decisions (proven by shared unit tests, not two copies).
- Tools streaming clients see thinking-off / overflow **before** `[DONE]`.
- No new process-global client. Shared caches are named stores with injection points.
- INDEX regenerated, not hand-patched.
