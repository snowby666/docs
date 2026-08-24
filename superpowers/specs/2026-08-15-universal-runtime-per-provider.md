# Universal Runtime — Per-Provider Rework

**Date:** 2026-08-15  
**Status:** Waves 1–5 shipped. Wave 7 Microsoft import hygiene shipped. Waves 6 / 8 / 9 / 10 are explicit no-ops. Audit-corrected 2026-08-15; live roster updated after pplx / together / siliconflow deprecation.  
**Scope:** every remaining registered key in `src/providers/client.py` `providers_map_actions` (57)  
**Parent design:** `docs/superpowers/specs/2026-08-15-universal-provider-runtime-design.md` (Tasks 0–7 already shipped)  
**Plan:** `docs/superpowers/plans/2026-08-15-universal-runtime-per-provider.md`

This document is the file-level rework for each live provider. It does not invent overflow destinations, new codecs, or catalog rows.

## Audit corrections (2026-08-15)

Verified against live `api.py` / `loop.py` / `AttemptRuntime` and current OpenAI / LiteLLM / Groq docs. These replace the first-pass wrap advice.

1. **`AttemptRuntime` is not a generic SSE wrapper.** After the first HTTP 200 it `break`s the retry loop, then maybe thinking-off, then overflow. Verboo instead rotates a new key up to `MAX_RETRIES=5` on empty 200, yields usage live, never thinking-off-retries, then InferHub. Dropping Verboo into `chat_chunks` would InferHub after one empty stream and hold usage. **Wave 2 is extract (`yield_openai_sse_deltas` inside the existing 5-key loop), not wrap.**
2. **Thin clients are the same class as Modal, not Gonka.** Albert/Feather use `raise_for_status=True` plus `3**(i+1)` 429 sleep and first-chunk `lstrip`. Groq clamps temperature and hardcodes `max_tokens`. AttemptRuntime never sees those exceptions as `response.status`. **Wave 4 is inner-SSE extract; keep the outer shell.**
3. **`stream_mode="buffered"` is the openai_compat hold-until-accepted reader** (`stream_reader_for` / `loop.py`). It is not “this plugin buffers tools.” Setting venice/ionet/telnyx to buffered would lie in the profile and would hijack the tools reader if those names ever hit `openai_compat`. **Wave 7 does not change `stream_mode`.**
4. **Do not add unused `ProviderProfile` fields.** `thinking_default` already exists. `max_http_retries` / `overflow_name` / `marketplace` have no readers until a later flags rewrite. This plan does not add them.
5. **Key-cursor semantics are not one function.** Start-at-`-1` then increment (kilo, inferhub, SI, modal, zeroeval) matches `RoundRobinKeys.next()` (first call = key[0]). Start-at-`0` then increment-before-return (albert, feather, arliai) skips key[0] on the first call when `len>1`. Verboo is return-then-increment (matches). Preserve each family; do not “fix” Albert’s skip.
6. **Albert `include_usage` is invented product behavior.** Same rule as SiliconFlow / Gonka usage: no live pass, no add.
7. **`flags.*` cannot be deleted in one task.** `loop.py` still has many `flags.dahl` / `gonka` / `inferhub` / `si` / `verboo` / `zai` / `kilo` branches; `session.py` uses `flags.marketplace`; `attempts/__init__.py` and `payload_build.py` dispatch on the bools. Wave 10 is “do not delete.”
8. **INDEX scripts write `src/providers/INDEX.md` and `src/tools/providers/INDEX.md` only.** They do not write `src/tools/INDEX.md`.
9. **Groq PlayAI TTS is deprecated upstream** (shutdown 2025-12-31; Orpheus replacements). Out of scope for this runtime plan — do not migrate TTS while touching chat SSE.
10. **LiteLLM (2026):** retry the same deployment on 429 / empty-pre-first-chunk; fallback only after retries. That is Verboo’s outer loop, not AttemptRuntime’s break-on-first-200. OpenAI `include_usage` is a **final** `choices: []` chunk — holding usage across a retry is correct; emitting it mid-empty-attempt (today’s Verboo) can leak a usage frame before InferHub.

### When `AttemptRuntime.chat_chunks` is legal

Use it only if the client already matches this ladder:

| Step | Gonka/Dahl | Verboo | Albert/Feather/Groq |
|---|---|---|---|
| Non-200 | predicate → overflow or sleep/retry | sleep, next key, up to 5 | `raise_for_status` + custom sleep |
| Empty 200 | thinking-off once, then overflow | next key, up to 5, then InferHub | no empty retry |
| Usage | hold until visible accepted | yield live | none / live |
| Key per attempt | one key in `headers` (no `header_fn`) | `_next_key()` every attempt | one key per request |

Everyone else: extract `yield_openai_sse_deltas` **inside** the existing shell, or leave.

## Problem (measured)

`AttemptRuntime` + `ProviderProfile` already exist. Only **gonka** and **dahl** chat `send_message` use them. Tools already read `profile_for` for stream/hold/retry, but `flags.py` still gates InferHub / Surplus Intelligence / Verboo / Kilo / Z.AI / Dahl remint.

The remaining live chat/media clients reimplement one or more of: SSE parse, key cursor, empty-visible retry, HTTP overflow, usage hold. Forcing every client through `AttemptRuntime` is the rejected Approach C — first-party, playground, prompt-sim, and media are different codecs. Waves 1–5 extracted matching-cursor auth, Verboo/thin/partial inner SSE, and the IH/SI session factory without wrapping those shells.

## Architecture (unchanged)

```
routes (chat / messages / responses)
        │
        ├─ gonka/dahl chat → AttemptRuntime(profile) → {"response"} / {"usage"}
        ├─ other OpenAI SSE chat → existing shell + yield_openai_sse_deltas
        ├─ OpenAI SSE tools → openai_compat loop → same profile predicates
        └─ other codec     → strategy plugin (profile row only)
```

Two codecs. One HTTP ladder. Plugins stay plugins.

`AttemptRuntime` owns: POST, OpenAI SSE iterate, `StreamProgress`, hold-until-accepted, thinking-off retry, named HTTP overflow, `forward_agen` teardown. Caller owns: session close, payload build, key/token mint, provider-specific overflow maps.

## Global constraints

Copy these into every task. Do not weaken them.

- Do not invent overflow destinations. Wire only maps that already exist (`resolve_gonka_overflow`, `map_verboo_to_inferhub`, `_map_zai_to_fallback`, `TELNYX_OVERFLOW_FALLBACK_MAP`, ArliAI GLM→Z.AI, IoNet→NVIDIA, PartyRock→Z.AI).
- Do not change `stream_buffered` flush for Verboo / InferHub / Surplus Intelligence.
- Do not add Gonka to `PROVIDER_STREAM_USAGE` without a live usage-accuracy pass.
- Do not revive `sse_chat.py` / `key_ring.py`.
- Do not edit `models.json`. In-memory `_UNREGISTERED_SOURCES` stays the catalog filter.
- Do not flatten `src/providers/runtime/`. New helpers go under an existing leaf (`auth/`, `overflow/`, `sse/`) or a new `runtime/marketplace/` leaf.
- Do not change `thinking=True` yield shape (`{"thinking": ...}` vs `<think>` inside `{"response"}`) without a live check. `yield_openai_sse_deltas` always tags reasoning into `{"response"}`. Albert / Feather / ArliAI / IoNet keep a custom branch when `thinking=True`.
- Do not call `AttemptRuntime.chat_chunks` unless the client already matches the gonka/dahl ladder (table above). Inner SSE extract uses `yield_openai_sse_deltas` only.
- Do not set `stream_mode="buffered"` on any provider that is not Verboo / InferHub / Surplus Intelligence.
- Do not add `stream_options.include_usage` to a client that does not already send it.
- Do not delete `ProviderFlags` bools while `loop.py` / `attempts/` / `payload_build.py` / `session.py` still read them.
- Preserve increment-first vs return-first key cursors. Do not “normalize” Albert/Feather/ArliAI onto `RoundRobinKeys` if that would start serving key[0] first when `len(API_KEYS)>1`.
- Nested `async for wrapper: yield` does not await teardown on outer `aclose`. Overflow paths must keep `forward_agen`.
- Surplus Intelligence / E2B “fireworks” strings are seller names, not the deprecated Fireworks client.
- INDEX: regenerate via `build_index_inventory` + `generate_indexes` + `verify_indexes`. Do not hand-edit generated rows.
- Do not revert the 16 deprecations (heliagent, helicone, prodia, gizai, mj, fireworks, chatgot, pollinations, promptlayer, rexua, zenmux, deepinfra, jamba, pplx, together, siliconflow).

## Disposition legend

| Disposition | Meaning |
|---|---|
| **done** | Chat already on `AttemptRuntime`. Keep. |
| **wrap** | Reserved for a future client that already matches the gonka/dahl ladder. **No remaining live provider qualifies.** |
| **auth** | Replace hidden `_CURRENT_INDEX` / `CURRENT_INDEX` with `RoundRobinKeys` (or `ProcessTokenCache`). No SSE change. |
| **extract** | Pull a shared helper (usage, SSE inner loop) out of a fat client. Do not wrap the whole `send_message`. |
| **profile** | Add/confirm a `ProviderProfile` row. Do not touch `send_message`. |
| **leave** | Media, guest, custom wire, or orchestration. Plugin forever. Profile default is enough. |

## Wave order

| Wave | Disposition | Providers | Why this order |
|---|---|---|---|
| 0 | done | gonka, dahl | Already shipped. Only legal `AttemptRuntime` chat clients. |
| 1 | auth | kilo, inferhub, surplusintelligence, modal, zeroeval, verboo | Start-at-`-1` or return-then-increment families only. **Not** albert/feather/arliai (increment-first skip). |
| 2 | extract (not wrap) | verboo | Inner `yield_openai_sse_deltas`; keep 5-key empty-200 loop + InferHub + live usage. |
| 3 | extract (marketplace, not wrap) | inferhub, surplusintelligence, kilo, markethub | Bid ladder / proxy matrix / quote-then-route do not fit a single-POST `AttemptRuntime`. |
| 4 | extract (thin SSE) | albert, feather, groq, llm7, cerebras, xai, modal | Inner SSE only. Keep `raise_for_status`, backoff, `lstrip`, `thinking=True`, payload quirks. together / pplx were extracted then deprecated. |
| 5 | extract (partial SSE) | nvidia, arliai, cf (chat path only) | Keep proxy/cooldown/GLM/media shells. |
| 6 | leave | crofai, chutes, zai, vermal, openai, openrouter | Fat openai_compat. Helpers only in a **new** spec. siliconflow was deprecated (not a Wave 6 leave). |
| 7 | leave (profile already correct) | google, mistral, cohere, anthropic, ionet, neuralwatt, telnyx, hyperfusion, partyrock, salesforce, sakana, venice, venicedev, e2b, cfplayground, k2think | `tool_type` already comes from `registry.py`. Do **not** set `stream_mode="buffered"`. |
| 8 | leave | akash, athina, comparia, hotbot, jatevo, meituan, neuro, notegpt, zeroeval, exa, grok, novelai, runware, higgsfield, microsoft, sambanova | Guest / media / custom coalesce. Microsoft: lazy voices only. |
| 9 | leave (last) | jewproxy, smolproxy | Meta-proxies. Touch only after Waves 1–5 settle. |
| 10 | leave | `flags.py` | Do not delete bools while loop/attempts/payload/session still read them. |

SambaNova is **leave**, not wrap: it coalesces deltas (`max_chunks` 100/1000) and posts a nested `body.messages` playground payload. Wrapping would change chunk granularity.

---

## Wave 0 — already on AttemptRuntime

### gonka

- **Files:** `src/providers/prod/gonka/api.py` (342), `src/providers/runtime/overflow/gonka.py`, `src/tools/providers/openai_compat/` gonka branches
- **Session:** aiohttp. **Auth:** `GONKA_KEYS = RoundRobinKeys(API_KEYS)` from `GONKA_API_KEYS`
- **Tools:** `openai_compat`
- **Overflow:** `resolve_gonka_overflow` → Verboo alias for `deepseek-ai/DeepSeek-V4-Flash-0731`, else `hyperfusion` `gonka/{mid}`. Predicate `should_fallback_to_hyperfusion` (429 + body regex).
- **Thinking/seed:** `gonka_chat_payload` + `gonka_thinking_payload` + `seed=random.randint(1, 2_000_000_000)`
- **KEEP:** AttemptRuntime, overflow map, seed bust, `hyperfusion_fallback` flag, passthrough stream
- **CHANGE:** none required for chat
- **DO NOT TOUCH:** overflow destinations, `gonka_thinking_payload`, adding Gonka to `PROVIDER_STREAM_USAGE`
- **Test bar:** existing `src/providers/prod/gonka/tests/test_empty_visible_offline.py`, `test_overflow_offline.py`, `test_overflow_aclose_offline.py`. Live: do not restage `results/deepseek_empty_stress.json`

### dahl

- **Files:** `src/providers/prod/dahl/api.py` (301), `src/providers/runtime/overflow/dahl.py`
- **Session:** aiohttp. **Auth:** `DAHL_TOKENS = ProcessTokenCache()` mint `POST /tokens`; 402 remint via `on_quota`
- **Tools:** `openai_compat`
- **Overflow:** reuses gonka `_hyperfusion_fallback` + `should_fallback_from_dahl` (gonka predicate + 413/502 Cloudflare/HTML)
- **KEEP:** token mint, Cloudflare UA, overflow via gonka family map, AttemptRuntime
- **CHANGE:** optionally pass `auth=DAHL_TOKENS` into AttemptRuntime (currently `auth=None`; mint stays in the client)
- **DO NOT TOUCH:** mint URLs, 402 remint cap, Cloudflare 502/413 extension
- **Test bar:** `src/providers/prod/dahl/tests/test_empty_visible_offline.py`, `test_overflow_offline.py`

---

## Wave 1 — auth store (no SSE change)

Replace hidden module ints with named `RoundRobinKeys` instances **only when the first call already returns key[0]**.

| Provider | Today | First call | Target |
|---|---|---|---|
| verboo | `_CURRENT_INDEX=0`, return then `+=1` | key[0] | `VERBOO_KEYS = RoundRobinKeys(API_KEYS)` |
| kilo | `_CURRENT_INDEX=-1` then increment | key[0] | `KILO_KEYS` |
| inferhub | `_CURRENT_INDEX=-1` then increment | key[0] | `INFERHUB_KEYS` |
| surplusintelligence | `_CURRENT_INDEX=-1` + preferred-key pin | key[0] (or pin) | `SI_KEYS`; pin stays outside the cursor |
| modal | `_CURRENT_INDEX=-1` then increment | key[0] | `MODAL_KEYS` |
| zeroeval | `_CURRENT_INDEX=-1` then increment | key[0] | `ZEROEVAL_KEYS` |
| albert | `CURRENT_INDEX=0`, increment **then** return | key[1] if `len>1` | **LEAVE the int.** One key today, so a swap is a latent footgun. |
| feather | same as albert | key[1] if `len>1` | **LEAVE the int.** |
| arliai | same as albert | key[1] if `len>1` | **LEAVE the int.** |

**DO NOT TOUCH:** NVIDIA `COOLDOWN_KEYS` / `random.sample`, OpenAI `KEY_COOLDOWNS` / `random.sample`, Chutes `random.shuffle`, OpenRouter free vs premium shuffle, Groq `ACCOUNT_INDEX` (used; one account so it is a no-op, still do not delete). HyperFusion `_RR_INDEX` (random start + return-then-increment — optional later, not this wave).

**Test:** `src/providers/tests/runtime/test_auth_cursor_offline.py` — each named store returns keys in the same cyclic order as the old int.

---

## Wave 2 — Verboo inner SSE extract (not AttemptRuntime)

### verboo

- **Files:** `src/providers/prod/verboo/api.py` (316), `src/tools/providers/openai_compat/providers/attempts/verboo.py`
- **Session:** aiohttp. **Auth:** Wave 1 `VERBOO_KEYS`
- **Tools:** `openai_compat`, `stream_mode="buffered"` (already). Do not change.
- **Overflow (exists):** `_inferhub_fallback` → `InferHubClient.send_message(map_verboo_to_inferhub(bot), ...)`. Aliases: `glm-4.7-flash`→`glm-4.7`, `qwen3.6-27b`→`qwen3.6-plus`
- **Thinking:** payload always `"thinking": {"type": "enabled"}`. `thinking` param is forwarded to InferHub only
- **Rework:** keep the `for attempt in range(MAX_RETRIES)` key loop, empty-200 continue, InferHub after exhaustion. Replace only the inner `async for line in response.content` parse with `yield_openai_sse_deltas`. Yield held usage **live** after each attempt that produced visible content (do not wait for a later retry). Empty attempt: discard held usage, next key.
- **KEEP:** `map_verboo_to_inferhub`, InferHub as sole overflow, buffered tools flush, `MAX_RETRIES=5`, `0.5 * (2 ** attempt)` sleep, live usage on success
- **CHANGE:** inner SSE parse only; optional move of `map_verboo_to_inferhub` to `runtime/overflow/verboo.py` with re-export
- **DO NOT TOUCH:** tools `stream_buffered` flush, InferHub alias table, calling `AttemptRuntime.chat_chunks`
- **Risk:** `yield_openai_sse_deltas` holds usage in a list. After a successful attempt, yield `held_usage[0]` before return. After an empty attempt, clear it. Do not emit usage from an empty attempt (today Verboo can; fixing that leak is allowed — it is a retry-correctness fix, not a new destination).
- **Test bar:** offline: 5 empty 200s → InferHub with mapped model; one visible attempt → no InferHub; `aclose` on InferHub uses `forward_agen`. Live: tools path still buffered.

---

## Wave 3 — marketplace extract (do not wrap the ladder)

### inferhub

- **Files:** `src/providers/prod/inferhub/api.py` (799), `sticky.py`, `routing.py`, `models.py`, `src/tools/providers/openai_compat/providers/attempts/inferhub.py`
- **Session:** aiohttp `ssl=False`, total=300. Import seeds catalog via `ensure_catalog()` (sync disk, not `asyncio.run`)
- **Auth:** Wave 1 `INFERHUB_KEYS`
- **Overflow:** internal bid ladder only. No external provider
- **KEEP:** entire ladder, sticky, balance gate, `_flush_pending` / codex hold
- **CHANGE:** extract session factory + key cursor into `src/providers/runtime/marketplace/` (shared with SI). Chat `send_message` stays the ladder owner
- **DO NOT TOUCH:** ladder advancement, `_flush_pending`, tools buffered flush, `flags.inferhub` until Wave 10
- **Why not wrap:** `AttemptRuntime` is one URL + one payload. InferHub is N bids with price headers and sticky remember/fail

### surplusintelligence

- **Files:** `src/providers/prod/surplusintelligence/api.py` (813), Hybrid-C `sticky.py` / `routing.py`
- **Auth:** Wave 1 `SI_KEYS` + preferred-key pin on first attempt
- **KEEP:** Hybrid-C, `MIN_DISCOUNT_PCT`, media/audio helpers, preferred-key pin
- **CHANGE:** same `runtime/marketplace/` session + key as InferHub
- **DO NOT TOUCH:** route ladder, discount gate, tools buffered flush
- **Why not wrap:** same as InferHub

### kilo

- **Files:** `src/providers/prod/kilo/api.py` (388), `src/tools/providers/openai_compat/providers/attempts/kilo.py`
- **Session:** aiohttp `ssl=False`, total=300. **Auth:** Wave 1 `KILO_KEYS`. Proxy per attempt via `get_random_proxy()`
- **Overflow:** none. Exhaustion yields `{"response": ""}`
- **KEEP:** proxy+key matrix, `sanitize_kilo_sse_line`, kilo stream reader, empty-response terminal
- **CHANGE:** key cursor only. Optional later: AttemptRuntime with a `proxy=` hook — not this wave
- **DO NOT TOUCH:** proxy manager, SSE noise strip, `stream_mode="kilo"`

### markethub

- **Files:** `src/providers/markethub/api.py` (356), `src/providers/markethub/sticky.py`
- **Auth:** none at hub. Delegates to InferHub / SI children
- **Overflow:** `pick_route` quote-then-route
- **KEEP:** quote, precommit usage hold, child ladders, SI-only media
- **CHANGE:** none for AttemptRuntime
- **DO NOT TOUCH:** `pick_route` order, commit semantics
- **Tools:** early `markethub_tool_chunks` in loop.py (not `flags_for`)

---

## Wave 4 — thin OpenAI SSE extract (not AttemptRuntime)

Shared pattern: keep payload, session, `raise_for_status`, backoff, and key loops. Replace only the default (`thinking=False`) inner `async for line` parse with `yield_openai_sse_deltas`. After the inner gen finishes, yield any held usage (these clients do not retry empty 200).

`thinking=True` clients (Albert, Feather, ArliAI) **must not** use `yield_openai_sse_deltas` for that branch. Keep the existing `{"thinking": ...}` loop.

Do not construct `AttemptRuntime`. Do not pass a dummy `run_overflow`.

### albert

- **Files:** `src/providers/prod/albert/api.py` (166)
- **Session:** aiohttp, connect 15s, `raise_for_status=True`
- **Auth:** leave `CURRENT_INDEX` (increment-first). One key today.
- **Overflow:** none. 429 retry `3**(i+1)` via `raise_for_status` exception text
- **KEEP:** multimodal flatten (keep `image_url`, flatten text-only lists), first-chunk `lstrip`, `choice.get("reasoning")` fallback if we stay on the custom parser for `thinking=True`
- **CHANGE:** default (`thinking=False`) inner SSE → `yield_openai_sse_deltas`. Note: that helper only reads `delta.reasoning` / `delta.reasoning_content`, not `choice.reasoning`. If Albert ever puts reasoning only on the choice object, keep a one-line copy into the delta before calling the helper — do not invent a new field.
- **DO NOT TOUCH:** `thinking=True` yield shape, `raise_for_status=True`, do **not** add `stream_options.include_usage`
- **Test:** offline flatten + 429 still sleeps `3**(i+1)`; `thinking=True` never calls `yield_openai_sse_deltas`

### feather

- **Files:** `src/providers/prod/feather/api.py` (170)
- **Already has** `usage_chunk` + `stream_options.include_usage`
- **KEEP:** default `temperature or 0.3`, `top_p or 0.7`, `stop: []`
- **CHANGE:** default (`thinking=False`) inner SSE → `yield_openai_sse_deltas`
- **DO NOT TOUCH:** `thinking=True` yield shape, increment-first `CURRENT_INDEX`, default `temperature or 0.3` / `top_p or 0.7`

### groq

- **Files:** `src/providers/prod/groq/api.py` (222)
- **Auth:** `get_account_round_robin()` uses `ACCOUNT_INDEX` (one account, so a no-op). Keep it.
- **Tools:** none
- **KEEP:** `generate_audio` entirely. PlayAI model ids are **deprecated upstream** (shutdown 2025-12-31; Orpheus replacements). Do not migrate TTS in this plan.
- **CHANGE:** chat inner SSE → `yield_openai_sse_deltas`. Keep payload clamps (`temperature>1` → 1, per-model `max_tokens`).
- **DO NOT TOUCH:** audio endpoints, org header, `raise_for_status=True`

### together — deprecated after extract

Moved to `src/providers/deprecated/together/`. Unregistered. Surplus Intelligence `"together"` / `"together ai"` strings are seller aliases, not this client.

### llm7

- **Files:** `src/providers/prod/llm7/api.py` (139)
- **Auth:** `API_KEYS` list, 3-attempt RR
- **CHANGE:** inner SSE → `yield_openai_sse_deltas`. Key loop stays as written.
- **KEEP:** 3-attempt budget

### pplx — deprecated after extract

Moved to `src/providers/deprecated/pplx/`. Unregistered.

### cerebras

- **Files:** `src/providers/misc/cerebras/api.py` (104)
- **Auth:** static Bearer
- **CHANGE:** inner SSE → `yield_openai_sse_deltas`
- **KEEP:** static header

### xai

- **Files:** `src/providers/misc/xai/api.py` (180)
- **KEEP:** `_sanitize_xai_messages`
- **CHANGE:** inner SSE → `yield_openai_sse_deltas`; shuffled `API_KEYS` stays per-request

### modal

- **Files:** `src/providers/prod/modal/api.py` (319)
- **Auth:** Wave 1 `MODAL_KEYS`. `MAX_RETRIES = max(1, min(len(API_KEYS), 8))`. Rotate on 429/empty
- **KEEP:** key-rotation **outer** loop. `:thinking` strip (no payload field). `_flatten_message_content` drops images
- **CHANGE:** inner SSE per key → `yield_openai_sse_deltas`. Outer key loop stays.
- **DO NOT TOUCH:** empty-stream → next key

---

## Wave 5 — partial SSE extract

### nvidia

- **Files:** `src/providers/prod/nvidia/api.py` (445)
- **Session:** aiohttp `ssl=False`, proxy per request `get_next_proxy()`
- **Auth:** `random.sample` + `COOLDOWN_KEYS` 12h on 403
- **Suffix:** `:thinking` → `chat_template_kwargs.thinking=True`. Pre-yield `"<think>\n"` for deepseek-r1/qwq. `BUILT_IN_THINKING_BOTS` skips close tag
- **max_tokens:** `MAX_TOKENS_MAP.get(bot, 4096)`
- **KEEP:** proxy, cooldown, suffix, max_tokens, pre-yield
- **CHANGE:** default-path inner SSE → `yield_openai_sse_deltas`. After the two-bot `"<think>\n"` pre-yield, set `think.open = True` so the helper does not open again. `BUILT_IN_THINKING_BOTS` keeps the custom unwrap. Do not construct `AttemptRuntime`.
- **DO NOT TOUCH:** cooldown JSON on disk, 403 policy

### arliai

- **Files:** `src/providers/prod/arliai/api.py` (695)
- **Auth:** leave increment-first `CURRENT_INDEX` (do **not** add `ARLIAI_KEYS`)
- **Overflow (exists):** GLM models when `GLM_SEMAPHORE.locked()` → `_redirect_to_zai` via `ARLIAI_TO_ZAI_MODEL_MAP`. Semaphores `REQUEST_SEMAPHORE(6)`, `GLM_SEMAPHORE(2)`
- **Import:** lazy `_ensure_img_models()` on first image call. No `asyncio.run` at import.
- **KEEP:** GLM overflow map, image endpoints, semaphores
- **CHANGE:** chat SSE block only; import-time `asyncio.run` removed
- **DO NOT TOUCH:** `ARLIAI_TO_ZAI_MODEL_MAP` destinations, `thinking=True` yield shape

### cf (chat path only)

- **Files:** `src/providers/prod/cf/api.py` (477)
- **Chat:** AI Gateway `compat/chat/completions` OpenAI SSE (`cf-aig-authorization`)
- **Media:** Workers REST image/audio with a different Bearer
- **KEEP:** all media methods
- **CHANGE:** optional chat inner SSE → `yield_openai_sse_deltas`. Media stays. Do not construct `AttemptRuntime`.
- **DO NOT TOUCH:** Workers tokens, image/audio

---

## Wave 6 — fat openai_compat (helpers only)

Do not wrap `send_message`. Extract only helpers that are already duplicated.

### crofai

- **Files:** `src/providers/prod/crofai/api.py` (477)
- **Auth:** single `_crofai_api_key()`
- **Suffix:** `:no-thinking` → `reasoning_effort="none"` else `"high"`
- **Overflow (exists):** `_fallback_to_zai` when two empty attempts and `bot` starts with `glm-`
- **KEEP:** `_normalize_messages` (remote image → base64), Z.AI fallback, think-tag without newline after `<think>`
- **CHANGE:** none required. Optional: share `usage_chunk` (already used)
- **DO NOT TOUCH:** Z.AI fallback trigger

### chutes

- **Files:** `src/providers/prod/chutes/api.py` (656)
- **Session:** `curl_cffi.requests.AsyncSession` `impersonate="chrome110"` + proxy. httpx for image upload
- **Auth:** `random.shuffle` of `CHUTES_AI_API_KEYS`
- **Thinking:** `:thinking` injects ◁/▷ system prompt; not AttemptRuntime think wrap
- **Import:** `asyncio.set_event_loop_policy(WindowsSelectorEventLoopPolicy())` on Windows
- **max_tokens:** hardcoded 4096
- **KEEP:** curl_cffi, fake OpenRouter headers, prompt injection, multimodal `format_content`
- **CHANGE:** none
- **DO NOT TOUCH:** Windows event loop policy, ◁/▷ replacement

### zai

- **Files:** `src/providers/prod/zai/api.py` (731), `src/tools/providers/openai_compat/providers/fallbacks/zai.py`
- **Auth:** single `API_KEY`
- **Overflow (exists, chat):** `_run_fallbacks` jewproxy → venice → chutes → vermal → crofai → telnyx via `forward_agen`
- **Overflow (exists, tools):** `zai_fallback_chain` telnyx → jewproxy → venice → chutes (no vermal/crofai)
- **KEEP:** both chains as-is (they are allowed to differ until a later explicit unify task)
- **CHANGE:** none in this wave. Do not delete `flags.zai`.
- **DO NOT TOUCH:** effort suffix, `thinking_payload`, inventing a unified chain

### vermal

- **Files:** `src/providers/prod/vermal/api.py` (1554)
- **Wire:** dual — Anthropic `/v1/messages` and OpenAI `/v1/chat/completions`
- **KEEP:** ShallowHide, hybrid memory, Anthropic thinking budget, inline tools in `send_message`, `_vermal_usage_chunk`
- **CHANGE:** none
- **DO NOT TOUCH:** Anthropic path, `:no-thinking` skip-tags

### openai

- **Files:** `src/providers/prod/openai/api.py` (1555)
- **Tools:** native `"openai"` handler, not openai_compat
- **KEEP:** `/v1/responses` fork (`RESPONSE_ENDPOINT`), hybrid memory, 24h `KEY_COOLDOWNS`, `strip_effort_suffix`
- **CHANGE:** none
- **DO NOT TOUCH:** responses API, cooldown window

### openrouter

- **Files:** `src/providers/prod/openrouter/api.py` (872)
- **KEEP:** hybrid memory, free vs premium keys, `transforms: ["middle-out"]`, `include_reasoning`, `:no-thinking` strip, `max_tokens` default **0**
- **CHANGE:** none
- **DO NOT TOUCH:** `max_tokens: 0` default (product)

### siliconflow — deprecated

Moved to `src/providers/deprecated/siliconflow/`. Unregistered. JewProxy catalog ids like `siliconflow/deepseek-v3.2` are seller prefixes, not this client.

---

## Wave 7 — plugins already have the right profile

`profile_for` already returns `tool_type` from `src/tools/registry.py` and `stream_mode="passthrough"` by default. That is correct. Dialect / prompt_sim tools buffer **inside their own handlers**, not via `stream_reader_for`. **Do not set `stream_mode="buffered"` here.** This wave is documentation + Microsoft voice-load hygiene only.

### first_party

#### google (`src/providers/prod/googleapi/` — registered as `"google"`)

- **Wire:** Gemini `streamGenerateContent?alt=sse`. `gemini_usage_chunk` (not interchangeable with `usage_chunk`)
- **Tools:** `src/tools/providers/first_party/google.py`
- **Auth:** `REGULAR_API_KEYS` / `EXP_API_KEYS` + restoration manager
- **Overflow:** exp pool → regular keys. 429 → restoration queue
- **KEEP / DO NOT TOUCH:** Gemini parts, `gemini_usage_chunk`, key restoration
- **CHANGE:** none. `tool_type` already `"google"`

#### mistral

- **Wire:** OpenAI-compat SSE but thinking is `delta.content[]` `type=="thinking"`
- **Tools:** `first_party/mistral.py` (`tool_choice=="required"` → `"any"`)
- **KEEP:** thinking list parse, tool_choice map
- **CHANGE:** none. `tool_type` already `"mistral"`
- **Why not wrap:** thinking list would be dropped by `yield_openai_sse_deltas`

#### cohere

- **Wire:** Cohere v2 events (`delta.message.content.text` / `.thinking`)
- **Tools:** `first_party/cohere.py`
- **KEEP:** v2 event translation
- **CHANGE:** none. `tool_type` already `"cohere"`

#### anthropic

- **Wire:** Anthropic Messages SSE (`delta.text` / `delta.thinking`)
- **Tools:** none in `registry.py`. Native tools live in `src/tools/messages_handler.py`
- **KEEP:** 32k checkpoint, system merge, weighted `ANTHROPIC_KEYS`
- **CHANGE:** none. `tool_type` is already `None`
- **DO NOT TOUCH:** do not add a chat-tools registry row unless a separate product task asks

### dialect

#### ionet

- **Wire:** guest `_create_guest_chat` then `POST /chats/{id}/messages`. ◁/▷ `_ThinkMarkerStream`. Empty DeepSeek V4 → `NvidiaClient`
- **Tools:** `dialect/ionet.py` buffers + `emit_buffered_completion`
- **CHANGE:** none. Do not set `stream_mode="buffered"`.
- **DO NOT TOUCH:** guest auth, marker stream, NVIDIA fallback map

#### neuralwatt

- **Wire:** OpenAI-shaped SSE plus `tool_call_chunk` / `finish_reason` yields
- **Tools:** `dialect/neuralwatt.py`
- **CHANGE:** none. Do not set `stream_mode="buffered"`.
- **DO NOT TOUCH:** native tool delta yields

#### telnyx

- **Wire:** playground `https://telnyx.com/api/inference` (not official `/v2`)
- **Overflow (exists):** `TELNYX_OVERFLOW_FALLBACK_MAP` + `run_overflow_fallbacks()` on 413
- **CHANGE:** none. Do not set `stream_mode="buffered"`.
- **DO NOT TOUCH:** overflow map, `payload_too_large`, playground message adapter

#### hyperfusion

- **Wire:** JWT login then custom concatenated `data:{json}` (`_extract_events`, no `\n\n`)
- **Auth:** `ACCOUNTS`, `_TOKEN_CACHE`, `_RR_INDEX`
- **CHANGE:** none. Do not set `stream_mode="buffered"`. Do not move `_RR_INDEX` in this plan.
- **DO NOT TOUCH:** `_extract_events`, JWT rotation. AttemptRuntime `hyperfusion_fallback` is a **different** concept (gonka family overflow)

### prompt_sim

All inject `<tool_call>` (or Venice GLM XML). No upstream `tools` field.

| Provider | Wire | Overflow | DO NOT TOUCH |
|---|---|---|---|
| partyrock | PartyRock proxy `invocation` blocks | tokens-not-ready → Z.AI; long → Gemini continuation | block conversion, proxy `/tokens` |
| salesforce | single JSON `chat-generations`, fake stream | token refresh only | Aura/OAuth mint |
| sakana | Namazu NDJSON + Fugu conductor | guest rotate | Fugu/conductor, guest pool |
| venice | NDJSON `kind=="content"` | 413 compact once; pro-only account rotate | compact, GLM TEE/workaround |
| venicedev | same `VeniceClient(compact=False)` | same | compact=False contract |
| e2b | Fragments `complete()` fake stream | rate-limit rebuild | Fragments schema / `extract_reply` |

**CHANGE:** none. Chat and tools handlers stay. `stream_mode` stays passthrough.

### playground

#### cfplayground

- **Files:** `src/providers/prod/cfplayground/api.py` (1668)
- **Wire:** relay OpenAI SSE **or** local Agents SDK WebSocket
- **Tools:** `playground/cfplayground.py`
- **LEAVE** send_message. Profile `tool_type="cfplayground"`

#### k2think

- **Files:** `src/providers/prod/k2think/api.py` (441)
- **Tools:** `playground/k2think.py` via non-stream `complete()` only
- **LEAVE** send_message. Profile `tool_type="k2think"`

---

## Wave 8 — leave (guest / media / custom)

### Guest / custom chat (no tools)

| Provider | Lines | Why leave |
|---|---:|---|
| akash | 452 | Vercel AI data-stream `0:"..."` + guest session cookie |
| athina | 109 | Athina prompt-execution stream; payload hardcodes `tools:[]` |
| comparia | 1663 | curl_cffi + Altcha/CapMonster + dual-lane arena |
| hotbot | 325 | scrape then **WebSocket** `wss://assistant.hbsvc.com/` |
| jatevo | 399 | anonymous playground; strips `tools` |
| meituan | 523 | LongCat oversea SSE, no key |
| neuro | 157 | JSON poll loop, not SSE |
| notegpt | 305 | cookie stream |
| zeroeval | 416 | session create + GET stream; Wave 1 auth only |
| exa | 57 | Vercel `0:"..."` guest (registered as search/chat hybrid) |
| sambanova | 102 | nested playground payload + **coalesced** deltas |

**CHANGE:** zeroeval auth cursor (Wave 1). Everyone else: profile default only.

### Media / non-chat

| Provider | Entry | Why leave |
|---|---|---|
| grok | `create_image` only | X GraphQL + cookies. No `send_message` |
| novelai | `create_image` | image lock |
| runware | `create_image` | image auth task |
| higgsfield | `create_image` | broker JWT job poll |
| microsoft | `generate_audio` only | TTS. Lazy `load_all_voices()` on first `generate_audio`; import must not call `asyncio.run` |

---

## Wave 9 — proxies last

### jewproxy

- **Files:** `src/providers/prod/jewproxy/api.py` (2863), `src/tools/providers/proxy/jewproxy.py`
- **Role:** meta-proxy. Routes `/proxy/<service>/v1/chat/completions` or `/v1/responses`. SmolProxy fallback on outage
- **KEEP:** service routing, Codex/xAI multi-agent, image/video/embeddings/moderation
- **CHANGE:** none until Waves 1–6 are done. Then optional: inner OpenAI SSE via `yield_openai_sse_deltas` only where the upstream is already OpenAI SSE
- **DO NOT TOUCH:** gate key, service map, SmolProxy fallback destinations

### smolproxy

- **Files:** `src/providers/prod/smolproxy/api.py` (540), `src/tools/providers/proxy/smolproxy.py`
- **KEEP:** `resolve_endpoint(bot)`, key pool, embeddings
- **CHANGE:** same as jewproxy — last
- **DO NOT TOUCH:** per-service base URL derivation

---

## Wave 10 — do not delete flags bits

`loop.py`, `attempts/__init__.py`, `payload_build.py`, and `session.py` still read `flags.dahl` / `gonka` / `inferhub` / `si` / `verboo` / `zai` / `kilo` / `marketplace`. A later spec may replace those bools with profile fields. **This plan does not delete them.**

---

## Profile fields — do not add in this plan

`ProviderProfile` already has `thinking_default`. Do not add `max_http_retries`, `overflow_name`, or `marketplace` until a task has a reader. Reserved names for a future flags rewrite only.

Do not add `AuthKind`. Dahl stays `ProcessTokenCache` in the client.

## File map (new)

| Path | Responsibility |
|---|---|
| `src/providers/runtime/marketplace/__init__.py` | Session factory (`ssl=False`, `connect=60`, `total=request_timeout`) |
| `src/providers/runtime/overflow/verboo.py` | optional move of `map_verboo_to_inferhub` (re-export from `verboo/api.py`) |
| `src/providers/tests/runtime/test_auth_cursor_offline.py` | Wave 1 |
| `src/providers/prod/verboo/tests/test_inner_sse_offline.py` | Wave 2 |
| `src/providers/tests/runtime/test_thin_extract_offline.py` | Wave 4 |
| `src/providers/tests/runtime/test_marketplace_session_offline.py` | Wave 3 |
| `src/providers/prod/arliai/tests/test_import_no_asyncio_run_offline.py` | Wave 5 |
| `src/providers/prod/microsoft/tests/test_import_no_asyncio_run_offline.py` | Wave 7 hygiene |

Existing live clients keep their `api.py`. Deprecated clients live under `src/providers/deprecated/` only — do not leave a second copy under `prod/`.

## Out of scope

- Catalog 200k clamp / `models.json` edits
- Gonka `PROVIDER_STREAM_USAGE`
- Unifying Z.AI chat vs tools fallback lists
- Wrapping first_party / playground / prompt_sim / media
- Reviving deprecated providers
- Changing JewProxy/SmolProxy routing
- New overflow destinations
- `thinking=True` codec unification
- Adding unused profile fields
- Setting plugin `stream_mode="buffered"`
- Groq PlayAI → Orpheus TTS migration
- Deleting `ProviderFlags` bools

## Approval gate

Waves 1–5 are shipped. Wave 7 is Microsoft voice-load hygiene only. Do not start Waves 6 / 8 / 9 implementations — those are Task 8 no-ops. Do not wrap anyone else in `AttemptRuntime`.
