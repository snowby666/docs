# Provider Routing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Route each catalog model through a `providers[]` list (OpenRouter-shaped): cheapest eligible healthy row by default, skip rows whose context window cannot hold this prompt, honor pins (`gnk/kimi-k2.6`, `provider.only`, user settings), remap vision per row, and bill/log the selected row’s price.

**Architecture:** One shared resolver (`src/providers/runtime/route/`) turns a catalog row + request prefs + “has images” + estimated prompt size into a `ResolvedRoute`. Chat / messages / responses stop unpacking `modelData["source"]` / `baseModel` / `tokens` themselves. Top-level `pricing` / `tokens` / `metadata.vision` are derived listing fields. Runtime billing (`UserManager.update_usage`), per-user `api_usage`, platform `ModelManager`, and `request_logs` all key the **selected public `provider_id`** and that row’s `pricing`. Optional `rpm` / `max_concurrency` are **Mongo-atomic** (`db.provider_capacity`). Live hop health is a **separate** Mongo circuit breaker (`db.provider_health`, keyed by wire `source`) — Envoy-shaped ejection with decay, not a mutating catalog `weight`. Both collections coordinate **3 Koyeb instances** the same way `src/utils/infra/lease.py` already does (`api.py` prefers `workers=1` and scales by replica).

**Tech Stack:** existing FastAPI routes, Pydantic request models (`ChatData`, `AnthropicMessagesRequest`, `ResponseRequest`), `src/utils/catalog/models.json` + `AI_MODELS`, `providers_map_actions` / `is_registered_provider`, `apply_native_reasoning_effort`.

**Spec:** this document (schema + routing rules below). There is no separate spec file.

## Global Constraints

- Do **not** flatten `src/providers/runtime/`. Add a new `route/` folder beside `overflow/`, `profile/`, `attempt/`.
- Do **not** revive `sse_chat.py` / `key_ring.py`.
- Do **not** revert deprecations; do **not** delete `deprecated/cfplayground`.
- `AttemptRuntime.chat_chunks` stays gonka/dahl only. `stream_mode="buffered"` stays verboo / inferhub / surplusintelligence only.
- Do **not** raise free-queue 180, markethub quote 30, e2b/mistral 180, models-cache 300.
- JewProxy / SmolProxy `/v1/responses` is out of scope.
- Top-level capability INDEX only via `build_index_inventory` + `generate_indexes` + `verify_indexes`. Folder maps via `python -m src.providers.tests.generate_folder_indexes` (must **not** overwrite `src/providers/INDEX.md`).
- Do **not** mass-migrate ionet/hyperfusion/kilo/telnyx/partyrock SSE loops.
- Do **not** add 403 to `should_fallback_from_dahl`.
- Do **not** expose internal `source` (`jewproxy`, `smolproxy`, …) on `/v1/models` or in `X-ElectronHub-Provider`.
- Bill the **selected catalog provider row**, not the advertised cheapest top-level `pricing`, and not the vision dest unless that dest is the selected row.
- Overflow maps (`ZAI_FALLBACK_MODEL_MAP`, gonka/dahl/ionet/partyrock/jewproxy) stay until a model’s `providers[]` already lists that dest. Do **not** auto-promote Venice/Chutes TEE into `providers[]`.
- Do **not** hand-edit 31k-line `models.json` except via the mechanical migration script (Task 6) or the already-done public:false prune (Task 0).
- Do **not** commit unless the user explicitly asks.
- PowerShell: use `;` not `&&`.
- Windows: `python -m pytest …` from the repo root.
- After **every** task: run that task’s **Audit sweep** before starting the next. A green unit test is not enough — the catalog and the resolver must stay the single source of truth.
- Do **not** leave a second copy of a rule in a route after the resolver owns it (vision swap, cheapest pick, context filter, pin merge).
- Do **not** add a `db.providers` usage/analytics collection. OpenRouter has no such document. Provider is a **dimension** on model×day activity. Capacity / health / perf stay their own collections (`provider_capacity`, `provider_health`, `provider_perf`).

---

## Source of truth

One object, one owner. If two files compute the same fact, the later file is wrong.

| Fact | Owner | Everyone else |
|------|-------|----------------|
| Which endpoints exist for a model | `model_data["providers"]` (or `providers_of` shim) | Routes never read top-level `source` / `baseModel` / `vision_provider` after Task 4/9. |
| Listing `tokens` / `pricing` / `metadata.vision` | `derive_listing_fields` at catalog load (Task 6) | Do not hand-type these after migration. Runtime billing ignores them. |
| Which row runs this request | `resolve_route` (rank) then `acquire_route` (Task 13 capacity + HTTP walk; Task 14 skips ejected hops) | Routes bind `provider, baseModel, tokensLimit, pricing` from `ResolvedRoute` only. |
| Public slug clients see | `ResolvedRoute.provider_id` (`providers[].id`) | Never `source`. Never `jewproxy` / `smolproxy`. |
| Price charged / logged | `ResolvedRoute.pricing` | Never listing `modelData["pricing"]` on a routed request. |
| Vision remap | selected row’s `vision_provider` | No model-root `vision_provider` after Task 10. |
| Account pin / ignore | `auth_data["routing"]` + `merge_prefs` | No second user fetch. |
| Live hop performance (TTFT / tok/s / uptime %) | Task 16 `db.provider_perf` keyed by **`model_id::provider_id`** | Default rank never reads this. Only `sort=latency\|throughput`. Never fall back to `_id: gnk`. |
| RPM / in-flight | `db.provider_capacity` keyed by public `id` | No in-process dict. No Redis. |
| Live hop health | `db.provider_health` keyed by **wire `source`** (Task 14) | Never mutate `providers[].weight`. Fail-open if Mongo errors. |
| Per-provider analytics | `api_usage.models.*.providers.{id}` (user lifetime), `ModelManager.providers.{id}` on the **model** doc (platform; Task 12 lifetime, Task 12b calendar windows), `request_logs.provider` (30d event) | Winning row only. Grain is **model × provider**, never a singleton provider doc. Hub-wide “gnk this month” = sum leaves at read time. |

### Per-task audit protocol (required)

Copy this checklist into the task’s last step. Do not mark the task done until every box is true.

1. **Owner grep.** Search the files this task touched for the facts it now owns. Confirm no leftover `modelData["source"]` / `["baseModel"]` / `["vision_provider"]` in those files (unless the task is explicitly pre-Task-10 and still on the shim).
2. **Cross-file drift.** If this task added a helper (`providers_of`, `cheapest_pricing`, `merge_prefs`, `prefs_from_request`), grep for a second implementation of the same rule. Delete the duplicate.
3. **Contract tests.** Re-run this task’s pytest file **and** every earlier route-test file that would break if the contract changed. One command, listed on the task.
4. **Catalog invariant** (Tasks 6–10 only). `test_every_row_has_providers` + `test_listing_fields_match_derived` still pass. After Task 10, `test_no_legacy_top_level_route_fields` too.
5. **Error copy.** New 400s use the envelopes in Task 2 (`no_vision`, `context`, `unknown_provider_id`, `no_providers`). Do not invent a second message for the same code.
6. **Progress table.** Set this task’s row to `[x]` and one-line notes. Do not start the next task with a dirty table.
7. **No drive-by.** Do not “while I’m here” edit `models.json` by hand, overflow maps, or INDEX.md.

### Who owns failover (locked — four layers)

These are not the same system. Do not collapse them.

| System | Task | What it does |
|--------|------|----------------|
| `resolve_route` / `iter_eligible_rows` | 2 | Rank + hard filters. Sync. No HTTP. No Mongo. |
| Overflow maps (`ZAI_FALLBACK_MODEL_MAP`, gonka→hyperfusion, …) | 8 de-dupes; maps stay | Intra-provider dest swap **inside** one `source`. Not a second billed `providers[]` peer. Hop-local empty recovery (`should_overflow` / thinking-off) stays here. |
| `acquire_route` + HTTP walk | **13** | Same-request recovery: Mongo capacity skip, then retryable hop errors **before the first visible token**. Recovers *this* request. |
| `db.provider_health` circuit breaker | **14** | Cross-request memory: eject a sick **wire `source`** so the *next* requests skip it. Envoy-shaped cooldown with decay. Deprioritize, do not delete. Panic-bypass if every remaining hop is ejected. |

Tasks 4 and 5 pick **one** row (`resolve_route`). They do **not** walk. Task 8 does **not** walk HTTP. Task 5 must not say “walking is Task 8’s job”. Task 13 must not mutate catalog `weight`. Task 14 must not invent a second retryable-error classifier — it calls Task 13’s `is_retryable_hop_error`.

---

## Progress

| Task | Status | Notes |
|------|--------|-------|
| 0. Delete `public: false` catalog rows | [x] | 488 hidden rows removed; `gpt-5.3-codex-low` stays public |
| 1. Route types + cheapest / max helpers | [x] | `route/types.py` + `pricing.py`; cheapest/max/vision/listing helpers |
| 2. `resolve_route` + per-provider vision remap | [x] | sync rank + vision remap; `RouteError` codes; no Mongo |
| 3. Legacy shim (`source`/`baseModel` → one row) | [x] | `providers_of` wraps legacy rows; id==source until Task 6; catalog smoke green |
| 4. Wire chat / messages / responses | [x] | resolve after tokenize; vision via resolver; keep-effort via providers_of |
| 5. Request `provider` object (OpenRouter-shaped) | [x] | `ProviderPreferences` on 3 request models; `prefs_from_request` → `bind_text_route`; no HTTP walk |
| 6. Mechanical `models.json` → `providers[]` | [x] | 1098 rows wrapped; `id` from `owned_by` (gonka→`gnk`); listing fields derived at load |
| 7. Public `/v1/models` projection + header | [x] | sanitized `providers`; `X-ElectronHub-Provider` = `route.provider_id` |
| 8. Overflow de-dupe for listed peers | [x] | listed `(source, baseModel)` dropped from Z.AI overflow; Venice/Chutes stay |
| 9. Media routes + leftover readers | [x] | `bind_media_route`; inventory/cache read `providers_of`; unregistered = every hop |
| 10. Drop legacy top-level `source` / `baseModel` / `vision_provider` | [x] | stripped; listing tokens/pricing/vision restamped from providers[]; Telnyx writers finalize |
| 11. Pin syntax + account routing settings | [x] | prefix pin + merge_prefs; auth `routing`; GET/PATCH `/v1/users/settings/routing` |
| 12. Per-provider analytics + request-log provider | [x] | `api_usage` + `ModelManager` provider leaves; request-log `provider` + sanitized pricing |
| 12b. Provider calendar windows (no new collection) | [x] | Rollover `usage_daily/weekly/monthly` on existing `db.models` provider leaves; reuse parent date markers; no `db.providers` |
| 13. Optional `weight` / Mongo `rpm` / `max_concurrency` + HTTP walk | [x] | `db.provider_capacity`; `is_retryable_hop_error`; walk before first token |
| 14. Mongo hop health (circuit breaker + decay) | [x] | `db.provider_health` by wire `source`; eject / decay / panic-bypass; no catalog-weight mutation |
| 15. OpenRouter pref contract + `max_price` | [x] | `extra=forbid`; hard `max_price` from catalog row pricing; unknown keys 422 |
| 16. Hop performance windows + opt-in speed sort | [x] | Writer + `sort=latency\|throughput` / `:nitro`/`:floor`; default pick still cheapest |

---

## Corrected catalog schema

Internal catalog (what `models.json` stores). Public `/v1/models` is a **projection** (Task 7).

### Field glossary

| Field | Who sees it | Meaning |
|-------|-------------|---------|
| `providers[].id` | clients + logs | **Cosmetic** public slug. Unique **within the model**. Used by `provider.order` / `only` / `ignore`, `X-ElectronHub-Provider`, and `api_usage` / `ModelManager` / `request_logs`. **Default = slug of top-level `owned_by`.** Never a hidden hop (`jewproxy`, `smolproxy`, `partyrock`, `salesforce`, …). **Must not contain `.` or `$`**. Example: Salesforce Claude → `id=anthropic`, `source=salesforce`. |
| `providers[].name` | clients + logs | **Cosmetic** display name. **Default = pretty form of `owned_by`** (`OpenAI`, `Anthropic`, `ZhipuAI`). Listing and analytics only. Gonka exception: `GNK`. |
| `providers[].source` | internal only | **The real hop.** Key in `providers_map_actions`. Routes, clients, overflow, and capacity internals use this. |
| `providers[].baseModel` | internal only | Upstream id for **text** (and for images only when `vision_provider` is absent — which means that row cannot take images). |
| `providers[].tokens` | clients (sanitized) | Context limit of **this** endpoint. |
| `providers[].pricing` | clients (sanitized) | Price of **this** endpoint. Runtime billing uses this object for the selected row. |
| `providers[].vision_provider` | internal only | Optional. If present, this row can take images and dispatch remaps to `{source, baseModel}` — **which may be a different provider**. If absent, the row is text-only. |
| `providers[].weight` | internal (optional) | Relative share among **same-price** eligible rows. Default `1`. `0` = drain (keep in catalog, never pick unless pinned). |
| `providers[].rpm` | internal (optional) | Soft **global** cap for this public `id` across all models and all Koyeb replicas. Mongo `db.provider_capacity`. If at limit, skip the row (try the next eligible). Not a user-facing 429. |
| `providers[].max_concurrency` | internal (optional) | Same collection, `in_flight` field. Protects the upstream across 3 instances. |
| Top-level `tokens` | clients | `max(providers[].tokens)`. Keep the name; `/v1/models` already ships it (+ context aliases). |
| Top-level `pricing` | clients | **Derived at load**, not hand-typed. Same `type` only; cheapest = lowest `input`, then `output`, then `cache_read`. Listing only. |
| Top-level `metadata.vision` | clients | `true` if **any** provider row has `vision_provider`. |
| Top-level `source` / `baseModel` / `vision_provider` | — | Legacy. Shim in Task 3; deleted in Task 10. |

### Public `id` / `name` (Task 6)

`id` and `name` are labels. Do **not** invent a brand per hop. Task 6 assigns them as follows:

1. **Default:** `id = slug(owned_by)`, `name = pretty(owned_by)`. Salesforce GPT → `openai` / OpenAI. PartyRock Claude → `anthropic` / Anthropic. JewProxy Qwen → `alibaba` / Alibaba. InferHub Kimi → `moonshotai` / MoonshotAI.
2. **Gonka only:** always `gnk` / GNK, even when `owned_by` is `moonshotai` / `minimax`. That is the pin (`gnk/kimi-k2.6`).
3. **Never emit a hop as `id`:** `jewproxy`, `smolproxy`, `partyrock`, `salesforce`, `athina`, `kilo`, `venice`, `verboo`, `albert`, `inferhub`, `surplusintelligence`, `exa`, `e2b`, `notegpt`, `comparia`, `dahl`, `telnyx`, `zeroeval`, `llm7`, `k2think`, `hotbot`. If `owned_by` is missing or equals one of these, the script exits 1.
4. **JewProxy / SmolProxy:** still use `owned_by` (already a lab). Prefix map is only a fallback when `owned_by` is missing.
5. **Empty-catalog hops** (no rows in `models.json` today): `comparia`, `dahl`, `hotbot`, `k2think`, `llm7`, `notegpt`, `telnyx`, `zeroeval`. No migrate work. When a row is added (or Telnyx bootstraps), apply the same helper — `owned_by` if it is a lab, else fail.
6. **Later collision:** `id` must be unique inside one model. If a second hop is added and `owned_by` matches the first row’s `id`, the new peer gets an **explicit** id (`byteplus`, `moonshotai`). Do not auto-copy `owned_by` in that case. v1 migrate is one row per model, so this does not fire.

`pretty(owned_by)` uses a small display map (`openai`→`OpenAI`, `x-ai`→`xAI`, `zhipuai`→`ZhipuAI`, `huggingface`→`Hugging Face`, `deepseekai`→`DeepSeek`, `moonshotai`→`MoonshotAI`, `mistralai`→`Mistral`, `blackforestlabs`→`Black Forest Labs`, …). Unknown slugs Title-Case.

### `vision_provider` is per provider, not per model

Today the catalog has **one** model-level `vision_provider`. That is wrong once a model has multiple text endpoints: Gonka cannot see images, InferHub can; Z.ai text vs BytePlus text may remap vision differently.

**Rule:** each provider row that can serve a request with images **must** declare its own remap:

```json
"vision_provider": {
  "baseModel": "kimi-k2.6",
  "source": "inferhub"
}
```

Same shape as today’s model-level object. The dest `source` does **not** have to match the row’s `source`. The dest does **not** have to be a sibling in `providers[]` (Athina text → OpenAI vision is still valid).

- Missing `vision_provider` → this row is skipped when the request has images.
- Present `vision_provider` → after the row is selected, swap dispatch `source`/`baseModel` to the remap. Re-apply `apply_native_reasoning_effort` against `{baseModel: vision.baseModel, source: vision.source}` so `@high` is not dropped (see `src/utils/tests/test_reasoning_effort_offline.py::test_vision_provider_keeps_the_effort_suffix`).
- Native-vision row (same mapping): still write the object, e.g. kimi-k3 today (`baseModel` + `source` both `inferhub` / `kimi-k3`).
- Do **not** invent `"vision_provider": true` shorthand in v1.
- Do **not** put `vision_provider` back at the model root after Task 10.

Billing stays on the **selected text row**. Vision is a dispatch remap, not a second billed catalog peer, unless that dest is itself selected as the row (same `id`).

### Worked examples

`kimi-k2.6` as it actually works today (Gonka text, InferHub vision), plus a later Moonshot peer:

```json
"kimi-k2.6": {
  "name": "MoonshotAI: Kimi K2.6",
  "description": "…",
  "endpoints": ["/v1/chat/completions", "/v1/responses", "/v1/messages"],
  "premium_model": true,
  "object": "model",
  "owned_by": "moonshotai",
  "public": true,
  "tokens": 500000,
  "pricing": {
    "type": "per_million_tokens",
    "input": 0.6,
    "output": 3,
    "cache_read": 0.16
  },
  "providers": [
    {
      "id": "gnk",
      "name": "GNK",
      "source": "gonka",
      "baseModel": "moonshotai/Kimi-K2.6",
      "tokens": 262000,
      "pricing": {
        "type": "per_million_tokens",
        "input": 0.6,
        "output": 3,
        "cache_read": 0.16
      },
      "vision_provider": {
        "baseModel": "kimi-k2.6",
        "source": "inferhub"
      }
    },
    {
      "id": "moonshotai",
      "name": "MoonshotAI",
      "source": "jewproxy",
      "baseModel": "moonshot/kimi-k2.6",
      "tokens": 500000,
      "pricing": {
        "type": "per_million_tokens",
        "input": 0.8,
        "output": 3,
        "cache_read": 0.2
      },
      "vision_provider": {
        "baseModel": "moonshot/kimi-k2.6",
        "source": "jewproxy"
      }
    }
  ],
  "metadata": {
    "vision": true,
    "function_call": true,
    "web_search": false,
    "reasoning": true,
    "supported_reasoning_efforts": ["none", "minimal", "low", "medium", "high", "xhigh", "max"]
  }
}
```

`glm-5.2` (no vision on either row):

```json
"glm-5.2": {
  "name": "ZhipuAI: GLM 5.2",
  "description": "…",
  "endpoints": ["/v1/chat/completions", "/v1/responses", "/v1/messages"],
  "premium_model": true,
  "object": "model",
  "owned_by": "zhipuai",
  "public": true,
  "tokens": 1000000,
  "pricing": {
    "type": "per_million_tokens",
    "input": 0.8,
    "output": 3,
    "cache_read": 0.2
  },
  "providers": [
    {
      "id": "zhipuai",
      "name": "ZhipuAI",
      "source": "zai",
      "baseModel": "glm-5.2",
      "tokens": 1000000,
      "pricing": {
        "type": "per_million_tokens",
        "input": 1,
        "output": 3.2,
        "cache_read": 0.26
      }
    },
    {
      "id": "byteplus",
      "name": "BytePlus",
      "source": "jewproxy",
      "baseModel": "glm/glm-5.2",
      "tokens": 500000,
      "pricing": {
        "type": "per_million_tokens",
        "input": 0.8,
        "output": 3,
        "cache_read": 0.2
      }
    }
  ],
  "metadata": {
    "vision": false,
    "function_call": true,
    "web_search": false,
    "reasoning": true,
    "supported_reasoning_efforts": ["none", "minimal", "low", "medium", "high", "xhigh", "max"]
  }
}
```

The BytePlus row is a **later** catalog PR. Task 6 only wraps today’s single hop as `id=zhipuai` (`owned_by`). When a second hop is added and `owned_by` is already used, that peer gets an explicit id (`byteplus`).

Invalid JSON in the original draft (extra `}` parking `metadata` outside the model, trailing commas, `//` comments) must not land in `models.json`.

### Request body (OpenRouter-shaped, additive)

Add the same optional object to `ChatData`, `AnthropicMessagesRequest`, and `ResponseRequest`:

```json
"provider": {
  "order": ["byteplus", "zai"],
  "only": ["zai"],
  "ignore": ["gnk"],
  "allow_fallbacks": true,
  "sort": "price"
}
```

| Field | Default | Meaning |
|-------|---------|---------|
| omitted `provider` | cheapest eligible | After hard filters (registered, vision, context, health/capacity, ignore). Ties broken by higher `weight`, then catalog order. |
| `order` | — | Public `id`s, tried first in this sequence; remaining eligible rows follow. Disables cheapest default (OpenRouter-compatible). |
| `only` | — | Intersect. Unknown id → 400 `invalid_request_error` / `param: provider`. |
| `ignore` | — | Subtract. Merged with account `routing.ignore`. |
| `allow_fallbacks` | `true` | On retryable HTTP **or upstream context-length errors**, walk the remaining eligible list. OpenRouter-compatible: `only` / `order` in the body keep this default **true**. Only the **model-id prefix pin** (`gnk/kimi-k2.6`) defaults it to `false`. |
| `sort` | unset (= price among eligible) | `"price"` (same as default, no load-balance) or `"context"` (widest window first). `"latency"` / `"throughput"` are already **422** via `Literal` (not silently cheapest-routed). Task 15 closes `extra="ignore"` on the other keys (`preferred_*`, `max_price`, `zdr`, …). Task 16 is the only reader of hop windows. |

### Resolver algorithm

`resolve_route` is **sync and Mongo-free** (rank + hard filters except live capacity). Tests in Task 2 stay offline.

`async def acquire_route(...)` (Task 13) walks the ranked list, admits Mongo capacity, and (same task) retries the next row on retryable HTTP / upstream context-length **before the first visible token**. Routes call `resolve_route` in Tasks 4–12; they switch the call site to `acquire_route` in Task 13. Do not add a third walker.

`resolve_route(model_data, *, has_images, prefs=None, prompt_tokens=None, max_output_tokens=None) -> ResolvedRoute`

`needed = (prompt_tokens or 0) + (max_output_tokens or 0)` when either is set. If `max_output_tokens` is omitted, filter on prompt size only (the model may still generate into the remaining window). `max_tokens == 0` is treated as omitted.

**Routes must estimate `prompt_tokens` before the first resolve** (Task 4, shipped). Chat / messages / responses tokenize, then `bind_text_route`. Do not unpack `source` / `tokens` before tokenize.

Chicken-and-egg: the accurate TAGS / hotbot tokenize path needs `provider`. **Do not** resolve twice. Locked order:

1. After `__split_content`, compute a **provider-agnostic** estimate: `__tokenize_messages(text_messages, image_urls)`, and if `tools` or `tools_in_messages` then `__tokenize_payload(native_tool_messages, tools=tools)`.
2. `resolve_route(..., prompt_tokens=estimate, max_output_tokens=max_tokens)`.
3. After resolve, TAGS / hotbot may recompute. If the refined count now exceeds `route.tokens` → existing 400 copy. **Do not** re-resolve. TAGS models are one-row.
4. Web-search `token_budget` (today uses unpacked `tokensLimit` *before* tokenize) uses listing `modelData["tokens"]` (widest window). That field stays after Task 6.

1. `rows = providers_of(model_data)` — Task 3 shim if `providers` missing.
2. Drop rows whose `source` is not in `providers_map_actions`.
3. Drop rows with `weight == 0` unless the public `id` is in `prefs.only` **or** `prefs.order` (any position) **or** the model-id prefix pin. A drain row the user named is not drained.
4. Apply `merge_prefs` (account then request) against `id` (never `source`). See Pinning.
5. Unknown ids: `only` containing an id not on this model → 400 `unknown_provider_id`. `ignore` / `order` unknown ids are skipped silently (OpenRouter-compatible).
6. If `has_images`: keep only rows that have a `vision_provider` object with string `source` + `baseModel`. If none left → 400 `"This model does not support image inputs."` A pin to a text-only row with images is `no_vision`, even if a sibling can see.
7. If `needed > 0`: drop rows with `tokens < needed`. If none left → 400 `context` with the Task 2 envelope (`This request exceeds the context window of all eligible providers.`). **Do not** fail the whole model when a cheaper/smaller peer cannot fit — that is the point of this filter. Do **not** invent a second message that names the largest window.
8. Live `rpm` / `max_concurrency` are **not** applied in `resolve_route`. Task 13 `acquire_route` drops rows currently over cap (**Mongo** `db.provider_capacity`, key = public `id`, shared by all models and all Koyeb replicas). If every remaining row is at cap → 400 `No eligible providers for this request.` (not a user RPM 429). Live health is also **not** applied in `resolve_route` — Task 14 `acquire_route` deprioritizes ejected wire sources.
9. If `prefs.order` is set: stable-sort so listed ids come first in that sequence, then the rest by the default key. Else if `prefs.sort == "context"`: sort by `tokens` descending. Else: sort by price key (see `cheapest_pricing`), then `-weight`, then catalog index.
10. If the eligible list is empty → 400, not a silent primary.
11. Take `rows[0]` as the selected catalog row (`resolve_route`). `acquire_route` then **admits** the Mongo capacity slot for `row.id`. If the admit races and now exceeds the cap, skip to the next ranked row.
12. Dispatch:
    - text: `source=row.source`, `baseModel=row.baseModel`, `tokens=row.tokens`
    - images: `source=row.vision_provider.source`, `baseModel=row.vision_provider.baseModel`, `tokens=row.tokens` (acceptance already used the text row’s window)
13. `pricing = row.pricing` (selected catalog row, not vision dest, not top-level cheapest).
14. `provider_id = row.id` for `X-ElectronHub-Provider`, `api_usage`, `ModelManager`, and request logs.
15. Caller runs `apply_native_reasoning_effort(dispatch_baseModel, effort, {**model_data, "baseModel": dispatch_baseModel, "source": dispatch_source})`.
16. **HTTP failover (Task 13 walk + Task 14 health):** on a retryable hop error (`is_retryable_hop_error`), if `allow_fallbacks`, **release** the current capacity slot and try the next eligible row **before the first SSE/JSON token**. Retryable = 429 / 502 / 503 / 504 / connect / empty-before-token (after hop-local recovery) / generic 500 / upstream context-length. After the first visible token, or after a tool adapter has called `update_usage`, stop walking. Do **not** failover on 401 / 403 / 404. Do **not** add 403 to `should_fallback_from_dahl`. Do **not** also run `ZAI_FALLBACK_MODEL_MAP` for a dest already in `providers[]` (Task 8). Always release the slot in `finally` (success, fail, disconnect). Task 14 records failure/success on the **wire `source`** per the health table above.
17. `X-ElectronHub-Provider` is the **winning** id. `StreamingResponse` headers are frozen at construction (`chat.py` `streaming_response` ~1506). Therefore the HTTP walk must finish **before** `streaming_response()` / `non_streaming_response()` / the tools `StreamingResponse` is constructed. Do not walk inside `generate_chunks` after that constructor has run.

**Listing vs runtime (intentional, OpenRouter-like):** top-level `tokens` = max window; top-level `pricing` = cheapest row. Those two fields may come from **different** rows (glm-5.2: 1M from Z.ai, $0.8 from BytePlus). Runtime never uses the listing pair.

### Pinning (Task 11)

Three ways to force a provider. They are **not** identical on fallbacks:

| Form | Prefs | `allow_fallbacks` default |
|------|-------|---------------------------|
| `gnk/kimi-k2.6` prefix | `only=["gnk"]` | **false** (a pin) |
| Body `provider.only` / `order` | as sent | **true** (OpenRouter) |
| Account `routing.only` / `ignore` | merged under request | **true** unless the request/prefix says otherwise |

1. **Model id prefix** (OpenRouter-inverted, matches the user’s `gnk/kimi-k2.6`):
   - Partition `@` first (chat in-chat image), then parse `{provider_id}/{catalog_id}` when the left segment is a known public provider `id` **on that catalog row** and the remainder is a catalog key.
   - No catalog key today contains `/` or `@`, so this does not collide.
   - `electronhub/` continues to strip first (`_strip_model_prefix`). Chat strips `:reasoning-exclude` before the pin.
   - Public `id` for Gonka is **`gnk`** (not `gonka`) so the pin string matches the brand. `source` stays `gonka`.
2. **Request body:** `provider: { "only": ["gnk"] }` or `order: ["gnk"]` + `allow_fallbacks: false`.
3. **Account settings** (user doc, not a new `user_db.py`): `routing: { "only": ["gnk"], "ignore": ["byteplus"] }`. Merged the same way OpenRouter merges account-wide ignored/allowed providers with the request object. Request wins on conflict.

Unknown pin (prefix or body `only`) → 400 `RouteError("unknown_provider_id")` / `param: provider`. One envelope. Do **not** use `param: model` for the prefix — `bind_route._ROUTE_PARAMS` maps the code, not the parse site.

`merge_prefs(account, request, prefix=None)` (Task 11) — request wins on `only`; ignores union; prefix is a hard pin:

```
ignore = (set(account.ignore) - set(request.only or [])) | set(request.ignore or [])
only   = request.only if request.only is not None else account.only
# prefix pin: only=[id], allow_fallbacks=False always.
# request.order / sort may still apply; request.allow_fallbacks=True cannot reopen the pin.
allow_fallbacks = False if prefix else (request.allow_fallbacks if request.allow_fallbacks is not None else True)
```

`electronhub/gnk/kimi-k2.6`: `_strip_model_prefix` first, then pin parse. Chat in-chat image: partition `@` **before** the slash split so `gnk/kimi-k2.6@flux-2` → catalog key `kimi-k2.6@flux-2` + `only=["gnk"]`. Messages / responses do not use `@image` (an `@` suffix there is an invalid model). No catalog key contains `/` or `@` (verified 2026-08-16, 610 keys).

Media routes (`bind_media_route`) do **not** read account/request prefs or prefix pins. Image generation is not chat.

`PATCH /v1/users/settings/routing` **replaces** both lists. Omitting `ignore` writes `ignore: []` (clears it). It is not a merge.

### Billing + analytics (Task 12)

There is **no** `src/manager/user_db.py`. Three write paths already run from `update_usage` and **all three** must learn `provider_id`. Request logs alone are not analytics (TTL 30 days, capped 10k, no rollup).

| Sink | File | Today | After |
|------|------|-------|-------|
| User by-model | `UserManager.update_api_usage` → `db.api_usage` | `models.{sanitized}.requests/tokens/cost` | Same totals **plus** `models.{sanitized}.providers.{provider_id}.*` (same five counters). `/v1/user/models` stays additive. |
| Platform by-model | `ModelManager.increment_model_usage` → model docs | in-process batch → `$inc` usage/in/out | Same **plus** `providers.{provider_id}.{usage,in,out}`. Batching stays; 3 replicas already flush independently and Mongo sums (see `model.py` comments). |
| Platform global | `APIManager` → `db.api` `type=global` | requests/tokens/consumption only | No per-provider split in v1 (hub totals stay one number). |
| Request log | `UserManager.log_request` | no provider | `provider` + sanitized `pricing` snapshot. |

Callers must pass **`route.pricing`** into `update_usage` (never listing `modelData["pricing"]`) and **`route.provider_id`**. Increment user + platform counters on the **selected catalog row id**, including when vision remaps dispatch elsewhere (billing identity ≠ wire dest).

`get_api_usage` already attaches catalog `owned_by` (vendor). Keep that. Add `providers` under each model. Do not replace `owned_by` with the route slug.

Sanitize `provider_id` with the same dot→underscore helper if a slug ever contains a dot; the catalog contract is “no `.` / `$` in `id`”.

### Weight / RPM / concurrency (Task 13)

Industry split:

- OpenRouter: no public `weight`; default is inverse-square-of-price among providers without a 30s outage. `sort`/`order` disable that.
- LiteLLM: default `simple-shuffle` by `weight`/`rpm`; they warn usage-based Redis routing is slow. `rpm`/`tpm`/`max_parallel_requests` are capacity filters.
- Portkey: explicit `weight` (0 = drain). Fallback is a separate strategy.

**We add the fields. We do not copy OpenRouter’s inverse-square random in v1.** Default is deterministic cheapest-among-eligible.

- `weight`: tie-break only (and drain at 0). In-process, no Mongo.
- `rpm` / `max_concurrency`: **Mongo**, not in-process, not Redis. `src/utils/lease.py` already states the API runs as **N Koyeb replicas**; `api.py` prefers `workers=1` and scales by instance. An in-process counter would admit **3×** the cap. User rate limits already use `find_one_and_update` on Mongo (`src/moderation/auth.py`) — copy that pattern.
- Collection: `db.provider_capacity`, `_id` = public `id` (global across models). One Gonka cap covers every model that routes to `gnk`.
- Admit: atomic `$inc` `in_flight` + rolling-minute `rpm_count` (reset `rpm_window_start` when the minute changes), same shape as `rate_limits`.
- Release: `$inc in_flight: -1` in `finally` (success, fail, failover, disconnect).
- Stale `in_flight` after a crashed replica: if `updated_at < now - STREAM_TOTAL_SECONDS`, the next admit treats `in_flight` as 0 (same recovery idea as lease `expires_at`).
- Do **not** 429 the user for this cap — skip to the next eligible row.
- Do **not** add throughput/latency `sort` in this task. Task 16 owns that later. Do **not** add Redis.

### Production failover (locked 2026-08-16 — research)

Industry split (OpenRouter, LiteLLM, Portkey, Envoy, Cloudflare AI Gateway):

| Layer | Who | What we copy |
|-------|-----|----------------|
| Same-request walk | OpenRouter `allow_fallbacks`, LiteLLM fallbacks, Cloudflare Dynamic Routing | Task 13 `acquire_route` |
| Cross-request cooldown | OpenRouter “significant outage in the last 30s” (deprioritize, not remove); LiteLLM `allowed_fails` + `cooldown_time`; Portkey circuit breaker (min 30s); Envoy outlier ejection | Task 14 `db.provider_health` |
| Decay | Envoy: ejection time = `base_ejection_time × streak`, cap `max_ejection_time`; healthy intervals decrement the streak | Task 14. **Not** a decaying catalog `weight` |
| Panic | Portkey: if every target is OPEN, bypass breaker. LiteLLM: cooldown skipped if every peer is cold | Task 14 panic-bypass |
| Traffic `weight` | Portkey / LiteLLM | Static catalog field. `0` = operator drain. **Never written by live errors.** |
| Inverse-square random | OpenRouter default among stable providers | **Out of scope.** We pick deterministically cheapest. Inverse-square is a **price** weight (`1/price²`), not a latency statistic. Task 14 cooldown is the reliability half. |

**Do not invent a live `providers[].weight` EWMA.** That mixes operator drain with hop sickness, races across 3 replicas, and poisons listing / pins. Health is a cooldown clock next to the hop.

**Health key = wire `source`**, after vision remap (`wire_source(row, has_images=…)`). Capacity / analytics stay on public `id`. InferHub vision 502 cools `inferhub`, not `gnk`. A jewproxy 429 on `byteplus` cools every jewproxy-backed row.

**Classifier (one function, Task 13 owns it):**

```python
HopErrorKind = Literal["429", "502", "503", "504", "connect", "empty", "500", "context_length", "fatal"]

def is_retryable_hop_error(kind: HopErrorKind) -> bool:
    return kind in {"429", "502", "503", "504", "connect", "empty", "500", "context_length"}
```

| kind | Walk this request (13) | Health write (14) |
|------|------------------------|-------------------|
| `429` / `502` / `503` / `504` / `connect` | yes | **Immediate eject** (streak += 1) |
| `empty` | yes, only after hop-local `should_overflow` / thinking-off exhausted, and still no visible token | `fail_count += 1`; eject at **3** |
| `500` | yes | `fail_count += 1`; eject at **2** (first 500 is often request-shaped) |
| `context_length` | yes | **none** — hop is not sick |
| `fatal` (401 / 403 / 404 / other 400 / after first token / after `update_usage`) | no | **none** |

Ejection duration: `min(300_000, 30_000 * streak)` ms (Envoy default 30s base, 300s cap). `is_ejected` is true only while `now < ejected_until`. After expiry the hop is half-open: the next request may try it. **Success** on a half-open hop: `fail_count = 0`, `ejection_streak = max(0, streak - 1)`. A later failure while half-open increments streak again (longer cooldown — flapping hops stay out).

One `record_failure` per `(request_id, source)` when that hop is abandoned. Hop-local retries (thinking-off) are not health events.

Mongo errors on health: **fail-open** (not ejected). Availability beats a stuck breaker.

Prefix pin (`allow_fallbacks=false`): try the pinned hop even if ejected; do not walk.

`acquire_route` order after Task 14:

1. `iter_eligible_rows` (rank).
2. Partition: non-ejected first, ejected last (deprioritize). If `allow_fallbacks` is false, do not skip the pinned row.
3. If every remaining row is ejected → panic-bypass (use full rank order).
4. Admit capacity (Task 13). At-cap → next row, no health write.
5. Dispatch. Retryable → `release`, `record_failure` (except `context_length`), next row.
6. Success (first token about to be sent, or non-stream JSON ok) → `record_success`.
7. `finally` release capacity.

### Defaults we locked (revised 2026-08-16)

1. Default pick = **cheapest eligible** after vision + context + capacity + ignore + **non-ejected health** (Task 14). Not raw catalog order. Ejected hops are deprioritized (OpenRouter 30s-outage equivalent), not deleted.
2. Context is a **hard filter**, not a sort. Cheapest 240k is skipped when the prompt needs 300k; a 500k peer is tried. If none fit → 400.
3. `allow_fallbacks` default **true**, except the **model-id prefix pin**. Body `only` stays OpenRouter-true.
4. Hide `source` / `baseModel` / `vision_provider` on `/v1/models`.
5. Do not auto-promote Venice/Chutes from the Z.AI map into `providers[]`.
6. One resolver module; routes do not each reimplement filters.
7. Bill, log, and increment analytics on the selected public `id` at that row’s price.
8. Capacity counters live in Mongo (`db.provider_capacity`), shared by all Koyeb replicas.
9. HTTP / capacity walk lives in Task 13 `acquire_route`, not Task 4/5/8. Cross-request health lives in Task 14. Do **not** mutate `providers[].weight` as a health score.
10. Failed hops never write success analytics. Winning row only.
11. `model_supports_native_effort` after Task 10: keep-effort uses `any(providers_of.source in NATIVE_EFFORT_PROVIDERS)`; `@effort` apply uses the **dispatch** overlay `source` only.
12. Public `id` / `name` = `owned_by` (pretty). Only override: `gonka` → `gnk` / GNK. Never invent a hop brand. Never emit a hidden hop as `id`.
13. A `provider` key that would change the pick if honored, but is not implemented yet, is **400** — never silently dropped. Task 15 owns the reject list; Task 16 removes `latency` / `throughput` / `preferred_*` from it.
14. Latency and tok/s **never** affect the default pick. OpenRouter does not use them on the default path either (only 30s-outage deprioritize + inverse-square price). Our equivalent of that reliability half is Task 14. Speed sort is opt-in (Task 16).
15. No inverse-square random. No Auto Exacto. No `models[]` / `openrouter/auto` / `:exacto`. No `sort.partition` (that only exists for cross-model fallbacks we do not have).
16. `max_price` is a **hard** filter (Task 15, catalog `row.pricing`). `preferred_min_throughput` / `preferred_max_latency` are **soft** (Task 16): deprioritize, never 400 if every remaining hop misses.

### OpenRouter routing map (locked 2026-08-16 — research)

Sources (fetched 2026-08-16): [Provider routing](https://openrouter.ai/docs/guides/routing/provider-selection), [Model routing blog](https://openrouter.ai/blog/insights/model-routing/) (2026-06-12), [Performance blog](https://openrouter.ai/blog/insights/evaluate-llm-provider-performance/) (2026-07-28), [Auto Exacto](https://openrouter.ai/docs/guides/routing/auto-exacto), [Provider integration](https://openrouter.ai/docs/guides/community/for-providers).

**How OpenRouter actually picks a hop**

Two layers: **which model** (`model` / `models[]` / `openrouter/auto`) then **which provider** for that model. We only own the second layer. Cross-model fallback and Auto Router stay out of scope.

Default provider path (no `sort`, no `order`, no tools):

1. Deprioritize providers with a significant outage in the last **30 seconds** (not removed).
2. Among the stable set, pick from the lowest-cost candidates **weighted by inverse square of price** (A=$1, C=$3 → A is 9× more likely than C).
3. Remaining providers are same-request fallbacks (`allow_fallbacks` default true).

Setting `sort` or `order` **disables** that load balance and walks in deterministic order.

**Do they use latency / tok/s on the default path?** No. Those numbers are:

| When | What OpenRouter does |
|------|----------------------|
| Default (no tools) | Price-weighted among hops that were not just out. Latency/tok/s unused. |
| Request has `tools` | **Auto Exacto** (on by default): reorder by throughput + tool-call success + GPQA/Tau2 benchmarks; price kept **inside** each quality tier. Opt out with `sort: "price"`, `:floor`, or account default sort=price. |
| Client sets `sort: "latency"` | Deterministic lowest TTFT. No load balance. |
| Client sets `sort: "throughput"` or `:nitro` | Deterministic highest tok/s. No load balance. |
| Client sets `sort: "price"` or `:floor` | Deterministic cheapest. No load balance. |
| `preferred_max_latency` / `preferred_min_throughput` | Soft: hops that miss are deprioritized, never blocked. Number = p50; or `{p50,p75,p90,p99}`. |
| `max_price` | Hard: if no hop is under the cap, the request fails. |
| `sort.partition: "none"` | Only with `models[]`: sort endpoints globally across models. We have no `models[]`. |

Uptime % (5m / 30m / 1d) is published and used for traffic shaping (≥95% normal, 80–94% degraded, &lt;80% fallback-only) after 100+ samples. 400/413 do not hurt uptime; 429 is tracked separately; 401/402/404/5xx/mid-stream do.

**Field-by-field owner (no silent holes)**

| OpenRouter `provider` / slug | Our v1–14 | Owner | Rule if the client sends it early |
|------------------------------|-----------|-------|-----------------------------------|
| `order` / `only` / `ignore` | shipped (Task 5+11) | rank | as specified |
| `allow_fallbacks` | stored; walk is Task 13 | 13 | default true; prefix pin false |
| `sort: "price"` | shipped (same as default) | 2 | ok |
| `sort: "context"` | shipped (ours, not OR) | 2 | ok |
| `sort: "latency"` / `"throughput"` | not implemented | **16** | already **422** via `Literal`; stay rejected until 16 |
| `sort: { by, partition }` | no `models[]` | never | **400** |
| `preferred_min_throughput` / `preferred_max_latency` | needs hop windows | **16** | **400** until 16 |
| `max_price` | catalog-only, no stats needed | **15** | implement; hard filter |
| `:nitro` / `:floor` | aliases of sort throughput/price | **16** | strip after `electronhub/`, before pin; `:nitro` 400 until 16 |
| `:exacto` | Auto Exacto explicit | never | **400** |
| `:reasoning-exclude` | already ours | chat | unchanged |
| `quantizations` / `zdr` / `data_collection` / `require_parameters` / `enforce_distillable_text` | no catalog flags | later catalog PR | **400** |
| `models[]` / `openrouter/auto` | model layer | never | ignore `models[]` if a client sends it (OpenAI extra); do not walk other catalog keys |
| Auto Exacto (tools present) | no tool-schema hop score | never in this plan | cheapest-even-if-prompt-sim (already out of scope as v1.1 tools filter) |
| Inverse-square default | — | never | deterministic cheapest |
| 30s outage deprioritize | Task 14 eject | 14 | equivalent reliability half |
| Account default sort | we have `routing.only/ignore` only | 16 (optional `routing.sort`) | not in 15 |

`ProviderPreferences` today is `extra="ignore"`. `sort: "latency"` is **already 422** (`Literal["price", "context"]`) — it does **not** cheapest-route. The real hole is extra keys (`preferred_*`, `max_price`, `zdr`, `quantizations`, …) which are dropped. Task 15 forbids that and implements `max_price`.

### Edge-case catalog (locked)

| Case | Behaviour |
|------|-----------|
| Cheapest row window too small; sibling fits | Skip cheap row; pick sibling. Not a 400. |
| No row fits `prompt + max_output` | 400 `context` — Task 2 envelope (generic; do not name the largest window). |
| Images + pin to text-only row | 400 `no_vision` even if a sibling has `vision_provider`. |
| Images + default | Cheapest row **that has** `vision_provider`. Bill that row; dispatch remap. |
| `weight == 0` unpinned | Drain (never pick). |
| `weight == 0` in `only` / `order` / prefix | Pick it. |
| `only` unknown id | 400 `unknown_provider_id`. |
| `ignore` / `order` unknown id | Skip silently. |
| Account `ignore: [gnk]` + request `only: [gnk]` | Request wins → gnk stays. |
| Account `only: [gnk]` + request `only: [byteplus]` | Request wins → byteplus. |
| Prefix `gnk/kimi-k2.6` | `only=[gnk]`, `allow_fallbacks=false`. Request body cannot set it back to true. |
| Body `provider.only` | `allow_fallbacks` default **true**. |
| Mid-stream 502 after first token | No walk. Finish or error on that row. Header already the current id. |
| Retryable error before first token | Task 13 walk if `allow_fallbacks`. Header not yet sent. Task 14 may eject the failed **wire `source`**. |
| Empty 200, no visible tokens, hop-local recovery exhausted | Task 13 walk (same freeze as mid-stream). Task 14 counts toward empty threshold (eject at 3), not first-hit. |
| Generic 500 | Task 13 walk. Task 14 ejects only after **2 consecutive** 500s on that source. First 500 is request-shaped until proven otherwise. |
| Upstream context-length | Task 13 walk to a larger-window peer. **Do not** eject — the hop is not sick. |
| Tool adapter already called `update_usage` | No walk. No health write. |
| 401 / 403 / 404 / client 400 (not context-length) | No walk. No health write. |
| Wire dest ≠ selected `id` (vision remap) | Health keys the **dispatch `source`** (inferhub 502 cools inferhub, not `gnk`). Capacity / analytics still use public `id`. |
| Every remaining hop ejected | Panic-bypass: ignore health, try rank order (Portkey / LiteLLM). Still honor capacity + `allow_fallbacks`. |
| Prefix pin `allow_fallbacks=false` | Try the pinned hop even if ejected. No walk. |
| Mongo health read/write fails | Fail-open: treat as not ejected. Walk still runs. |
| Overflow dest already in `providers[]` | Task 8 drops it from the overflow map. Not a second billed hop. |
| Vision dest ≠ selected row | Bill selected row. Analytics `provider_id` = selected `id`. |
| `apply_reasoning_effort_to_model` before resolve | OpenAI `gpt-5.5`→`gpt-5.5-high` stays here (new catalog key). Native keep uses `any(providers_of)` so Task 10 does not drop `@max` on glm-5.2. |
| `apply_native_reasoning_effort` after resolve | Overlay dispatch `source` / `baseModel`. Gonka text on kimi-k2.6 does **not** get `@high`; InferHub vision dest does. |
| Effort-suffix catalog keys (`gpt-5.5-high`, `gpt-5.3-codex-low`) | Separate rows. Task 6 wraps each from its own `source`. Do not copy the parent’s future `providers[]`. |
| Tools present, cheapest is prompt-sim | v1 accepts that (same as today). Not a resolve filter. `supports_native_tools` keys off **dispatch** `source`. |
| TAGS stringify inflates tokens past `route.tokens` | 400 existing copy. No re-resolve. |
| In-chat image gen / title helper | Own catalog key. Resolve that key (or shim). Never reuse the chat `ResolvedRoute`. |
| `music.py` `pricing.coefficient` | `route.pricing["coefficient"]`. `cheapest_pricing` compares `coefficient` when type is `coefficient`. |
| `cache_usage._catalog_meta` after Task 10 | `baseModel` from `providers_of(entry)[0]`. `owned_by` stays top-level. |
| Telnyx / Mistral / ArliAI writers | Must emit `providers[]` (Task 10). A post-strip sync that writes legacy-only fields undoes Task 10. |
| `title_gen.py` | Reads `meta.source` / `meta.baseModel` (~64). Task 9. |
| `stats.py` | `owned_by or source` fallback. After strip, `owned_by` is enough; do not group by internal `source`. |
| DevPass `auth()` twin (~2179) | No `routing` today. Task 11 adds it on both return dicts. DevPass may keep `routing: {}`. |
| Inner `jewproxy`/`smolproxy` `update_usage` | Tool-adapter billing. Route-level walk must not start after those calls. |
| `APIManager` global totals | No per-provider split in v1. |
| Listing vs runtime | glm-5.2 listing: 1M from Z.ai, $0.8 from BytePlus. Runtime uses the selected row only. |
| Public id default | `slug(owned_by)`. Salesforce Claude → `anthropic`. JewProxy Qwen → `alibaba`. |
| Gonka public id | Always `gnk` / GNK (pin `gnk/kimi-k2.6`). |
| Hidden hop as `owned_by` or missing | Script exits 1. JewProxy/SmolProxy may fall back to `baseModel` prefix. |
| Second hop, same `owned_by` | Explicit id on the new peer (`byteplus`). v1 migrate does not hit this. |
| Empty-catalog hops | `comparia`, `dahl`, `hotbot`, `k2think`, `llm7`, `notegpt`, `telnyx`, `zeroeval`. Apply the same helper when a row appears. |

### Public `/v1/models` projection

Keep today’s extras (`name`, `description`, `id`, `object`, `created`, `owned_by`, `tokens`, `pricing`, `endpoints`, `premium_model`, metadata, context aliases). Additive:

```json
"providers": [
  { "id": "zhipuai", "name": "ZhipuAI", "tokens": 1000000, "pricing": { "type": "per_million_tokens", "input": 1, "output": 3.2, "cache_read": 0.26 } }
]
```

Never emit `source`, `baseModel`, or `vision_provider`. A separate `/v1/models/{id}/endpoints` can wait.

TTS/STT parameter metadata today keys off `values["source"]` + `values["baseModel"]` in `src/routes/normal/model.py` (`_build_public_model_dict`). After migration, use `providers[0].source` / `providers[0].baseModel` (media models stay one-row).

---

## File map

| File | Role |
|------|------|
| `src/providers/runtime/route/__init__.py` | Re-export `ResolvedRoute`, `ProviderPrefs`, `resolve_route`, `providers_of`, `cheapest_pricing`, `max_tokens`. |
| `src/providers/runtime/route/types.py` | Dataclasses / TypedDicts. |
| `src/providers/runtime/route/pricing.py` | `cheapest_pricing`, `max_tokens`, `derive_listing_fields`. |
| `src/providers/runtime/route/resolve.py` | `providers_of`, `resolve_route`. |
| `src/providers/runtime/route/pin.py` | `split_provider_model`, `merge_prefs` (Task 11). |
| `src/providers/runtime/route/capacity.py` | Mongo admit/release for `rpm` / `in_flight` (Task 13). |
| `src/providers/runtime/route/health.py` | Mongo eject / decay / panic-bypass (Task 14). Key = wire `source`. |
| `src/providers/runtime/route/errors.py` | `is_retryable_hop_error` + `HopErrorKind` (Task 13; Task 14 reuses). |
| `src/providers/runtime/route/tests/test_pricing_offline.py` | Cheapest / max / vision-OR tests. |
| `src/providers/runtime/route/tests/test_resolve_offline.py` | Filters, vision remap, shim, unknown source. |
| `src/manager/users/usage.py` | `update_api_usage(..., provider_id)` nested counters. |
| `src/manager/users/request_logs.py` | `log_request(..., provider, pricing)`. |
| `src/manager/model.py` | `increment_model_usage(..., provider_id)` batched `$inc`. |
| `src/utils/openai_type.py` | `ProviderPreferences` + field on `ChatData` / `ResponseRequest`. |
| `src/utils/anthropic_type.py` | Same field on `AnthropicMessagesRequest`. |
| `src/routes/normal/chat.py` | Replace unpack + model-level vision swap. |
| `src/routes/normal/messages.py` | Same. |
| `src/routes/normal/responses.py` | Same. |
| `src/routes/normal/model.py` | Sanitized `providers` on the public dict; TTS/STT via first provider row. |
| `src/utils/helpers.py` | `model_supports_native_effort`: overlay `source` if present, else `any(providers_of.source in NATIVE_EFFORT_PROVIDERS)`. Overlay `baseModel` if present, else `providers_of[0].baseModel`. |
| `src/utils/cache_usage.py` | `_catalog_meta` after Task 10: `baseModel` via `providers_of`. |
| `src/utils/models.json` | Task 0 prune (done). Task 6 wrap into `providers[]`. |
| `src/utils/__init__.py` | After Task 6: `derive_listing_fields` at load. After Task 10: `_UNREGISTERED_SOURCES` filter reads `providers_of` sources. |
| `src/moderation/auth.py` | Task 11: `"routing": user.get("routing") or {}` on both auth return dicts (~3009 and DevPass ~2179). |
| `src/routes/normal/settings.py` | Task 11: GET/PATCH `/v1/users/settings/routing` beside thinking. |
| `src/providers/runtime/overflow/zai.py` | Task 8: skip dests already listed on the model. |
| `misc/migrate_models_providers.py` | Task 6 one-shot script. |
| `src/routes/normal/{image,embedding,tts,audio,video,music,moderation,title_gen}.py` | Task 9: `resolve_route` or shim. |
| `src/routes/platform/stats.py` | Task 10: stop using `source` as `owned_by` fallback. |
| `src/providers/prod/{mistral,arliai}/sync_models.py` | Task 10: emit `providers[]`. |
| `src/providers/prod/telnyx/{tts,stt}.py` | Task 10: `bootstrap_into_ai_models` writes `providers[]`. |
| `src/providers/tests/build_index_inventory.py` | Task 9: count from `providers_of` sources. |
| `src/providers/runtime/INDEX.md` | Regenerated by folder-index script after `route/` exists. |

Do **not** add `resolve_route` to `src/providers/runtime/__init__.py` unless a caller already imports from that barrel and needs it. Prefer `from src.providers.runtime.route import resolve_route`.

---

### Task 0: Delete `public: false` catalog rows

**Files:**
- Modify: `src/utils/models.json`
- Keep: `gpt-5.3-codex-low` (`public: false`) — `apply_reasoning_effort_to_model("gpt-5.3-codex", "low")` rewrites to this key (`REASONING_EFFORT_MODELS` in `src/utils/helpers.py`). Deleting it would make `test_every_effort_variant_resolves_to_a_real_entry` fail and drop low-effort Codex onto the medium catalog row.

**Status:** done in the same session that wrote this plan. 488 hidden rows removed; 1 required hidden effort variant kept.

- [x] **Step 1: Inventory** — 1098 total, 489 `public: false`, 0 missing `public`.
- [x] **Step 2: Delete** all `public: false` except `gpt-5.3-codex-low`.
- [x] **Step 3: Reload JSON** and confirm `public: false` count is 1 (`gpt-5.3-codex-low` only).

---

### Task 1: Route types + cheapest / max helpers

**Files:**
- Create: `src/providers/runtime/route/types.py`
- Create: `src/providers/runtime/route/pricing.py`
- Create: `src/providers/runtime/route/__init__.py`
- Test: `src/providers/runtime/route/tests/test_pricing_offline.py`
- Test: `src/providers/runtime/route/tests/__init__.py` (empty)

**Interfaces:**
- Consumes: nothing from later tasks.
- Produces:
  - `ProviderRow` TypedDict (total=False): `id: str`, `name: str`, `source: str`, `baseModel: str`, `tokens: int`, `pricing: dict`, `vision_provider: dict | None`, `weight: int`, `rpm: int`, `max_concurrency: int`
  - `ProviderPrefs` dataclass: `order: list[str] | None = None`, `only: list[str] | None = None`, `ignore: list[str] | None = None`, `allow_fallbacks: bool = True`, `sort: Literal["price", "context"] | None = None`
  - `ResolvedRoute` dataclass: `provider_id: str`, `source: str`, `baseModel: str`, `tokens: int`, `pricing: dict`, `selected_row: dict`, `used_vision: bool`
  - `cheapest_pricing(rows: list[dict]) -> dict`
  - `max_tokens(rows: list[dict]) -> int`
  - `any_vision(rows: list[dict]) -> bool` — True if any row has a `vision_provider` dict with `source` and `baseModel`
  - `derive_listing_fields(rows: list[dict]) -> dict` — `{"tokens", "pricing", "metadata_vision"}`

- [x] **Step 1: Write the failing tests**

```python
from src.providers.runtime.route.pricing import cheapest_pricing, max_tokens, any_vision

ZAI = {"id": "zai", "tokens": 1_000_000, "pricing": {"type": "per_million_tokens", "input": 1, "output": 3.2, "cache_read": 0.26}}
BYTE = {"id": "byteplus", "tokens": 500_000, "pricing": {"type": "per_million_tokens", "input": 0.8, "output": 3, "cache_read": 0.2}}
GONKA = {"id": "gnk", "tokens": 262_000, "pricing": {"type": "per_million_tokens", "input": 0.6, "output": 3, "cache_read": 0.16}}
MOON = {
    "id": "moonshotai",
    "tokens": 500_000,
    "pricing": {"type": "per_million_tokens", "input": 0.8, "output": 3, "cache_read": 0.2},
    "vision_provider": {"baseModel": "moonshot/kimi-k2.6", "source": "jewproxy"},
}

def test_cheapest_is_lowest_input_then_output():
    assert cheapest_pricing([ZAI, BYTE])["input"] == 0.8
    assert cheapest_pricing([GONKA, MOON])["input"] == 0.6

def test_max_tokens_is_the_widest_window():
    assert max_tokens([ZAI, BYTE]) == 1_000_000

def test_any_vision_is_or_across_rows():
    assert any_vision([GONKA, MOON]) is True
    assert any_vision([ZAI, BYTE]) is False

def test_cheapest_does_not_mix_pricing_types():
    image = {"id": "img", "tokens": 0, "pricing": {"type": "per_image", "input": 0.04}}
    assert cheapest_pricing([ZAI, image])["type"] == "per_million_tokens"
```

- [x] **Step 2: Run tests to verify they fail**

Run: `python -m pytest src/providers/runtime/route/tests/test_pricing_offline.py -v`

Expected: FAIL (module not found).

- [x] **Step 3: Implement types + helpers**

`cheapest_pricing`: among rows whose `pricing.type` equals the majority type (or the first row’s type if tied), pick:

- `per_million_tokens`: min `(input, output, cache_read or 0)`
- `coefficient`: min `(coefficient,)`
- `per_image` / `fixed`: min `(input or 0,)`

Never compare `per_million_tokens` to `per_image` / `coefficient`. Single-row media models return that row’s pricing unchanged.

`max_tokens`: `max(int(row["tokens"]) for row in rows)` ; empty list raises `ValueError`.

`any_vision`: `vision_provider` must be a dict with non-empty string `source` and `baseModel`.

- [x] **Step 4: Run tests to verify they pass**

Run: `python -m pytest src/providers/runtime/route/tests/test_pricing_offline.py -v`

Expected: PASS.

- [x] **Step 5: Audit sweep**

- Grep `src/providers/runtime/route/` for a second cheapest/max implementation. There must be one.
- Confirm `__init__.py` re-exports `cheapest_pricing`, `max_tokens`, `any_vision`, `derive_listing_fields`, `ProviderPrefs`, `ResolvedRoute`.
- Progress table: Task 1 `[x]`.

---

### Task 2: `resolve_route` + per-provider vision remap

**Files:**
- Create: `src/providers/runtime/route/resolve.py`
- Modify: `src/providers/runtime/route/__init__.py`
- Test: `src/providers/runtime/route/tests/test_resolve_offline.py`

**Interfaces:**
- Consumes: `ProviderPrefs`, `ResolvedRoute`, types from Task 1.
- Produces:
  - `class RouteError(ValueError)` with `.code: str` (`no_providers` | `unknown_provider_id` | `no_vision` | `context` | `unregistered_source`)
  - `providers_of(model_data: dict) -> list[dict]` (shim lands in Task 3; Task 2 may require `providers`)
  - `resolve_route(model_data: dict, *, has_images: bool, prefs: ProviderPrefs | None = None, prompt_tokens: int | None = None, max_output_tokens: int | None = None) -> ResolvedRoute` (sync; no Mongo)
  - `iter_eligible_rows(...)` — same filters, yields ranked rows (used by `acquire_route`)

- [x] **Step 1: Write the failing tests** (use in-memory dicts, not `AI_MODELS`)

```python
from src.providers.runtime.route import ProviderPrefs, resolve_route
from src.providers.runtime.route.resolve import RouteError

KIMI = {
    "providers": [
        {
            "id": "gnk",
            "name": "GNK",
            "source": "gonka",
            "baseModel": "moonshotai/Kimi-K2.6",
            "tokens": 262000,
            "pricing": {"type": "per_million_tokens", "input": 0.6, "output": 3},
            "vision_provider": {"baseModel": "kimi-k2.6", "source": "inferhub"},
        },
        {
            "id": "moonshotai",
            "name": "MoonshotAI",
            "source": "jewproxy",
            "baseModel": "moonshot/kimi-k2.6",
            "tokens": 500000,
            "pricing": {"type": "per_million_tokens", "input": 0.8, "output": 3},
            "vision_provider": {"baseModel": "moonshot/kimi-k2.6", "source": "jewproxy"},
        },
    ]
}

GLM = {
    "providers": [
        {"id": "zai", "name": "Z.ai", "source": "zai", "baseModel": "glm-5.2", "tokens": 1000000, "pricing": {"type": "per_million_tokens", "input": 1, "output": 3.2}},
        {"id": "byteplus", "name": "BytePlus", "source": "jewproxy", "baseModel": "glm/glm-5.2", "tokens": 500000, "pricing": {"type": "per_million_tokens", "input": 0.8, "output": 3}},
    ]
}

def test_default_picks_cheapest_eligible():
    route = resolve_route(GLM, has_images=False)
    assert route.provider_id == "byteplus"
    assert route.source == "jewproxy"
    assert route.pricing["input"] == 0.8

def test_long_prompt_skips_cheap_small_window():
    route = resolve_route(GLM, has_images=False, prompt_tokens=600_000)
    assert route.provider_id == "zai"
    assert route.tokens == 1_000_000

def test_sort_price_picks_byteplus():
    route = resolve_route(GLM, has_images=False, prefs=ProviderPrefs(sort="price"))
    assert route.provider_id == "byteplus"
    assert route.source == "jewproxy"
    assert route.baseModel == "glm/glm-5.2"

def test_images_use_selected_row_vision_provider_not_model_root():
    route = resolve_route(KIMI, has_images=True)
    assert route.provider_id == "gnk"
    assert route.source == "inferhub"
    assert route.baseModel == "kimi-k2.6"
    assert route.used_vision is True
    assert route.pricing["input"] == 0.6  # billed as Gonka, not InferHub

def test_images_on_explicit_moonshot_stay_on_jewproxy():
    route = resolve_route(KIMI, has_images=True, prefs=ProviderPrefs(only=["moonshotai"]))
    assert route.source == "jewproxy"
    assert route.baseModel == "moonshot/kimi-k2.6"

def test_glm_images_are_rejected():
    try:
        resolve_route(GLM, has_images=True)
    except RouteError as exc:
        assert exc.code == "no_vision"
    else:
        raise AssertionError("expected RouteError")

def test_only_unknown_id_is_rejected():
    try:
        resolve_route(GLM, has_images=False, prefs=ProviderPrefs(only=["not-a-slug"]))
    except RouteError as exc:
        assert exc.code == "unknown_provider_id"
    else:
        raise AssertionError("expected RouteError")

def test_unregistered_source_is_dropped():
    data = {"providers": [
        {"id": "dead", "name": "Dead", "source": "not-registered", "baseModel": "x", "tokens": 100, "pricing": {"type": "per_million_tokens", "input": 0, "output": 0}},
        GLM["providers"][0],
    ]}
    route = resolve_route(data, has_images=False)
    assert route.provider_id == "zai"

def test_ignore_unknown_id_is_silent():
    route = resolve_route(GLM, has_images=False, prefs=ProviderPrefs(ignore=["not-a-slug"]))
    assert route.provider_id == "byteplus"

def test_weight_zero_is_drained_unless_named():
    data = {"providers": [{**GLM["providers"][1], "weight": 0}, GLM["providers"][0]]}
    assert resolve_route(data, has_images=False).provider_id == "zai"
    assert resolve_route(data, has_images=False, prefs=ProviderPrefs(only=["byteplus"])).provider_id == "byteplus"
    assert resolve_route(data, has_images=False, prefs=ProviderPrefs(order=["byteplus"])).provider_id == "byteplus"
```

`is_registered_provider` is used as-is. Fixture sources (`zai`, `jewproxy`, `gonka`, `inferhub`) are in `providers_map_actions`. Do not mock the map in these tests.

- [x] **Step 2: Run tests to verify they fail**

Run: `python -m pytest src/providers/runtime/route/tests/test_resolve_offline.py -v`

Expected: FAIL (`resolve_route` missing).

- [x] **Step 3: Implement `resolve_route`**

Use `is_registered_provider` from `src.providers.client`. Do not import `__get_client` (that constructs clients).

Map `RouteError.code` so routes can raise the existing HTTP envelopes:

| code | status | message (keep current copy where it exists) |
|------|--------|-----------------------------------------------|
| `no_vision` | 400 | `This model does not support image inputs.` |
| `unregistered_source` | 400 | `Provider {source!r} is not available.` |
| `unknown_provider_id` | 400 | `Unknown provider id.` / `param: provider` |
| `no_providers` | 400 | `Invalid model.` is wrong here — use `No eligible providers for this request.` |
| `context` | 400 | `This request exceeds the context window of all eligible providers.` |

- [x] **Step 4: Run tests to verify they pass**

Run: `python -m pytest src/providers/runtime/route/tests/test_resolve_offline.py src/providers/runtime/route/tests/test_pricing_offline.py -v`

Expected: PASS.

- [x] **Step 5: Audit sweep**

- Confirm `resolve_route` does **not** import Motor / `capacity.py`.
- Confirm vision remap never reads `model_data["vision_provider"]` when `providers` is present.
- Confirm empty eligible list raises `RouteError`, never returns a dummy row.
- Progress table: Task 2 `[x]`.

---

### Task 3: Legacy shim

**Files:**
- Modify: `src/providers/runtime/route/resolve.py` (`providers_of`)
- Test: `src/providers/runtime/route/tests/test_resolve_offline.py` (add cases)

**Interfaces:**
- Consumes: Task 2 `resolve_route`.
- Produces: `providers_of(model_data)` that understands both shapes.

```python
def providers_of(model_data: dict) -> list[dict]:
    rows = model_data.get("providers")
    if isinstance(rows, list) and rows:
        return rows
    source = model_data.get("source")
    base = model_data.get("baseModel")
    if not source or not base:
        return []
    row = {
        "id": source,  # legacy: public id == source; jewproxy leak only until Task 6 assigns brand slugs
        "name": source,
        "source": source,
        "baseModel": base,
        "tokens": model_data.get("tokens") or 0,
        "pricing": model_data.get("pricing") or {},
    }
    vision = model_data.get("vision_provider")
    if isinstance(vision, dict) and vision.get("source") and vision.get("baseModel"):
        row["vision_provider"] = {"source": vision["source"], "baseModel": vision["baseModel"]}
    return [row]
```

- [x] **Step 1: Add shim tests**

```python
def test_legacy_row_synthesizes_one_provider():
    data = {
        "source": "gonka",
        "baseModel": "moonshotai/Kimi-K2.6",
        "tokens": 262000,
        "pricing": {"type": "per_million_tokens", "input": 0.6, "output": 3},
        "vision_provider": {"baseModel": "kimi-k2.6", "source": "inferhub"},
    }
    route = resolve_route(data, has_images=True)
    assert route.source == "inferhub"
    assert route.baseModel == "kimi-k2.6"
    assert route.provider_id == "gonka"  # shim id == legacy source; Task 6 maps gonka → gnk

def test_legacy_text_keeps_source():
    data = {"source": "zai", "baseModel": "glm-5.2", "tokens": 1000000, "pricing": {"type": "per_million_tokens", "input": 1, "output": 3.2}}
    route = resolve_route(data, has_images=False)
    assert route.source == "zai"
    assert route.baseModel == "glm-5.2"
```

- [x] **Step 2: Run the new tests — expect FAIL, then implement `providers_of`, then PASS**

Run: `python -m pytest src/providers/runtime/route/tests/test_resolve_offline.py -v`

- [x] **Step 3: Smoke the live catalog through the shim** (still no `providers[]` on disk)

```python
from src.utils import AI_MODELS
from src.providers.runtime.route import resolve_route
from src.providers.runtime.route.resolve import RouteError

def test_every_public_chat_model_resolves_without_images():
    failures = []
    for mid, row in AI_MODELS.items():
        if not row.get("public"):
            continue
        if "/v1/chat/completions" not in row.get("endpoints", []):
            continue
        try:
            resolve_route(row, has_images=False)
        except RouteError as exc:
            failures.append(f"{mid}: {exc.code}")
    assert not failures, failures
```

Expected: PASS against current `models.json` (legacy fields still present).

- [x] **Step 4: Audit sweep**

- `providers_of` is the only shim. Grep `src/providers/runtime/route/` for a second legacy wrap.
- Shim `id` is `source` (so `gonka`, not `gnk`) until Task 6. Do not “fix” the test to expect `gnk`.
- Progress table: Task 3 `[x]`.

---

### Task 4: Wire chat / messages / responses

**Files:**
- Modify: `src/routes/normal/chat.py` (unpack ~359, vision swap ~480–492, tokenize ~521–554, title helper ~765, in-chat image ~828)
- Modify: `src/routes/normal/messages.py` (unpack ~191, vision swap ~297–309)
- Modify: `src/routes/normal/responses.py` (unpack ~206, vision swap ~282–295)
- Modify: `src/utils/helpers.py` — `model_supports_native_effort` (keep-effort must survive Task 10)
- Test: extend `src/utils/tests/test_reasoning_effort_offline.py` so vision effort uses `resolve_route` + `apply_native_reasoning_effort` (keep the kimi-k3 / gemini-3.6-flash assertions).

**Interfaces:**
- Consumes: `resolve_route`, `ProviderPrefs` (prefs stay `None` until Task 5), `RouteError`.
- Produces: routes bind `provider`, `baseModel`, `tokensLimit`, `pricing` from `ResolvedRoute`. **No HTTP walk in this task.**

Replace this pattern (all three files):

```python
baseModel, tokensLimit, endpoints, premiumModel, provider = (
    modelData["baseModel"], modelData["tokens"], modelData["endpoints"],
    modelData["premium_model"], modelData["source"],
)
```

**Locked order (no “resolve twice”, no “or”):**

1. `model in AI_MODELS` + `ultimate_model_check`.
2. `apply_reasoning_effort_to_model` (OpenAI variant rewrite: `gpt-5.5`+high → `gpt-5.5-high`). Re-lookup `modelData` if the key changed.
3. Endpoint / premium / cache-breakpoint checks that do **not** need `source`.
4. `__split_content` → `image_urls_length`.
5. Provider-agnostic token estimate (`__tokenize_messages` + `__tokenize_payload` when tools). **Then** resolve.
6. `resolve_route` / `RouteError` → HTTP 400 envelopes from Task 2.
7. Bind `provider, baseModel, tokensLimit, pricing = route.source, route.baseModel, route.tokens, route.pricing`. Keep `route` in scope for Tasks 7/12/13.
8. `apply_native_reasoning_effort(baseModel, effort, {**modelData, "baseModel": baseModel, "source": provider})`.
9. JewProxy xAI multi-agent guards, keyed off **dispatch** `provider` + `baseModel`.
10. Existing `prompt_tokens > tokensLimit` / `max_tokens + prompt_tokens > tokensLimit` checks against `route.tokens` (belt-and-suspenders; resolver already filtered).
11. Balance check against `route.pricing`, not listing pricing.
12. Claude-opus long-prompt swap stays, keyed off dispatch `baseModel`.

```python
try:
    route = resolve_route(
        modelData,
        has_images=image_urls_length > 0,
        prompt_tokens=estimated_prompt_tokens,
        max_output_tokens=max_tokens,
    )
except RouteError as exc:
    raise HTTPException(
        detail={"error": {"message": str(exc), "type": "error", "param": None, "code": 400}},
        status_code=400,
    ) from exc
provider, baseModel, tokensLimit, pricing = route.source, route.baseModel, route.tokens, route.pricing
if not is_registered_provider(provider):
    raise HTTPException(...)  # keep existing copy
baseModel = helpers.apply_native_reasoning_effort(
    baseModel, reasoning_effort, {**modelData, "baseModel": baseModel, "source": provider}
)
```

Delete the `if "vision_provider" in modelData:` blocks in all three files. The resolver owns that swap.

**`model_supports_native_effort` (bake-in — would silently drop `@high` after Task 10):**

```python
def _effort_source(model_data: dict) -> str | None:
    if model_data.get("source"):
        return model_data["source"]
    from src.providers.runtime.route import providers_of
    for row in providers_of(model_data):
        if row.get("source") in NATIVE_EFFORT_PROVIDERS:
            return row["source"]
    return None

def _effort_base(model_data: dict) -> str | None:
    if isinstance(model_data.get("baseModel"), str):
        return model_data["baseModel"]
    from src.providers.runtime.route import providers_of
    rows = providers_of(model_data)
    base = rows[0].get("baseModel") if rows else None
    return base if isinstance(base, str) else None
```

- Keep-effort (`apply_reasoning_effort_to_model` **before** resolve): `_effort_source` so glm-5.2 still keeps `max` when top-level `source` is gone.
- Apply-suffix (`apply_native_reasoning_effort` **after** resolve): still requires the overlay `source` (dispatch). A cheapest gonka text hop does not get `@high`; the InferHub vision dest does.
- Update `test_native_effort_models_stay_on_a_provider` and `test_declared_efforts_only_live_on_forwarding_providers` to read `providers_of(entry)` sources, not `entry["source"]`, **when** they start failing (Task 6/10). In this task they still pass on legacy fields.

**Title helper** (`chat.py` ~765): `AI_MODELS[prompt_model]["baseModel"]` → `providers_of(AI_MODELS[prompt_model])[0]["baseModel"]` (or `resolve_route` on that key). Do not reuse the chat route.

**In-chat image gen** (`chat.py` ~828): different catalog key. Leave legacy unpack until Task 9, but do **not** point it at the chat `ResolvedRoute`. Task 9 must convert it.

Web-search `token_budget` uses listing `modelData["tokens"]` (step 3, before resolve).

- [x] **Step 1: `model_supports_native_effort` helpers + tests still pass on legacy catalog**
- [x] **Step 2: Move resolve-after-split+tokenize in chat.py; delete model-level vision block; fix title helper**
- [x] **Step 3: Same resolve-after-split in messages.py and responses.py**
- [x] **Step 4: Run offline tests**

Run: `python -m pytest src/utils/tests/test_reasoning_effort_offline.py src/providers/runtime/route/tests -v`

Expected: PASS. `test_vision_provider_keeps_the_effort_suffix` still reads catalog `vision_provider` until Task 6; after Task 6 update it to `providers_of(entry)[n]["vision_provider"]`.

- [x] **Step 5: Audit sweep**

- Grep `src/routes/normal/chat.py`, `messages.py`, `responses.py` for `modelData["source"]`, `modelData["baseModel"]`, `modelData["vision_provider"]`. Chat in-chat image ~828 may still hit until Task 9. Title helper must not.
- Confirm there is no HTTP walk / `allow_fallbacks` loop in these files yet.
- Confirm `update_usage` still receives `pricing` from the bound `route.pricing` variable (not `modelData["pricing"]`).
- Progress table: Task 4 `[x]`.

---

### Task 5: Request `provider` object

**Files:**
- Modify: `src/utils/openai_type.py` — add `ProviderPreferences` and optional `provider` on `ChatData` and `ResponseRequest`
- Modify: `src/utils/anthropic_type.py` — same on `AnthropicMessagesRequest`
- Modify: chat / messages / responses to pass `ProviderPrefs` from the body
- Test: `src/utils/tests/test_response_request_schema_offline.py` (add parse cases); new `src/utils/tests/test_provider_prefs_offline.py`

**Interfaces:**
- Consumes: Task 2 `ProviderPrefs`.
- Produces: parsed request field.

```python
class ProviderPreferences(BaseModel):
    model_config = ConfigDict(extra="ignore")
    order: Optional[List[str]] = None
    only: Optional[List[str]] = None
    ignore: Optional[List[str]] = None
    allow_fallbacks: Optional[bool] = True
    sort: Optional[Literal["price", "context"]] = None
```

Field name on the three request models: `provider: Optional[ProviderPreferences] = None`.

OpenRouter clients already send this key. Extra unknown keys inside `provider` are ignored.

```python
def prefs_from_request(data) -> ProviderPrefs | None:
    raw = getattr(data, "provider", None)
    if raw is None:
        return None
    return ProviderPrefs(
        order=raw.order,
        only=raw.only,
        ignore=raw.ignore,
        allow_fallbacks=True if raw.allow_fallbacks is None else raw.allow_fallbacks,
        sort=raw.sort,
    )
```

- [x] **Step 1: Schema tests** — `ChatData.model_validate({..., "provider": {"sort": "price"}})` keeps `provider.sort == "price"`; unknown `sort: "latency"` → validation error; omitted `provider` is `None`.
- [x] **Step 2: Implement models + `prefs_from_request` in `src/providers/runtime/route/resolve.py`**
- [x] **Step 3: Pass prefs into `resolve_route` in the three routes**
- [x] **Step 4: Run** `python -m pytest src/utils/tests/test_provider_prefs_offline.py src/utils/tests/test_response_request_schema_offline.py src/providers/runtime/route/tests -v`

Failover walking (`allow_fallbacks`) on live HTTP is **not** this task. Selecting the first eligible row is enough. The walk is Task 13 (`acquire_route`). Task 8 is overflow de-dupe only.

- [x] **Step 5: Audit sweep**

- Confirm `ProviderPreferences` lives in the type modules and `prefs_from_request` is the only body→`ProviderPrefs` converter.
- Confirm extra keys inside `provider` are ignored (`extra="ignore"`).
- Confirm `sort: "latency"` still 400s (not silently ignored).
- Progress table: Task 5 `[x]`.

---

### Task 6: Mechanical `models.json` → `providers[]`

**Files:**
- Create: `misc/migrate_models_providers.py` (one-shot, idempotent)
- Modify: `src/utils/models.json` (script output only)
- Modify: `src/utils/__init__.py` — after load, `derive_listing_fields` onto each row that has `providers`
- Test: `src/providers/runtime/route/tests/test_migrate_contract_offline.py`

**Interfaces:**
- Consumes: `providers_of`, `derive_listing_fields`.
- Produces: every catalog row has `providers` with ≥1 entry; top-level `tokens` / `pricing` / `metadata.vision` match derived values.

Script rules:

1. If `providers` already exists and is a non-empty list, leave the row (re-derive listing fields only).
2. Else wrap `source` / `baseModel` / `tokens` / `pricing` into `providers[0]`.
3. `id` / `name` from **`owned_by`**, not from `source`. See “Public id / name” above. Shared helper (script + Telnyx/Mistral/ArliAI writers):

```python
HIDDEN_HOPS = frozenset({
    "jewproxy", "smolproxy", "partyrock", "salesforce", "athina", "kilo",
    "venice", "venicedev", "verboo", "albert", "inferhub", "surplusintelligence",
    "exa", "e2b", "notegpt", "comparia", "dahl", "telnyx", "zeroeval",
    "llm7", "k2think", "hotbot",
})

SOURCE_OVERRIDE = {
    "gonka": ("gnk", "GNK"),
}

OWNED_BY_DISPLAY = {
    "openai": "OpenAI",
    "anthropic": "Anthropic",
    "google": "Google",
    "x-ai": "xAI",
    "zhipuai": "ZhipuAI",
    "huggingface": "Hugging Face",
    "deepseekai": "DeepSeek",
    "moonshotai": "MoonshotAI",
    "mistralai": "Mistral",
    "blackforestlabs": "Black Forest Labs",
    "alibaba": "Alibaba",
    "meta": "Meta",
    "nvidia": "Nvidia",
    "amazon": "Amazon",
    "minimax": "MiniMax",
    "bytedance": "ByteDance",
    "stabilityai": "Stability AI",
    "nousresearch": "Nous Research",
    "thinkingmachines": "Thinking Machines",
    "lumalabs": "Luma",
    "xiaohongshu": "Xiaohongshu",
    "aion-labs": "Aion Labs",
    "hidream-ai": "HiDream",
    "falai": "Fal",
    "xlabsai": "xLabs",
    "neta-art": "Neta Art",
    "myshell": "MyShell",
    "poolside": "Poolside",
    "stepfun": "StepFun",
    "gryphe": "Gryphe",
    "writer": "Writer",
    "recraft": "Recraft",
    "tencent": "Tencent",
    "xiaomi": "Xiaomi",
    "krea": "Krea",
    "microsoft": "Microsoft",
    "cohere": "Cohere",
    "novelai": "NovelAI",
    "meituan": "Meituan",
    "sakana": "Sakana",
    "replicate": "Replicate",
}

# Prefix fallback only when jewproxy/smolproxy owned_by is missing.
JEWPROXY_PREFIX = {
    "qwen/": ("alibaba", "Alibaba"),
    "openai/": ("openai", "OpenAI"),
    "google-ai/": ("google", "Google"),
    "xai/": ("x-ai", "xAI"),
    "deepseek/": ("deepseekai", "DeepSeek"),
    "groq/": ("groq", "Groq"),
    "openrouter/": ("openrouter", "OpenRouter"),
    "glm/": ("byteplus", "BytePlus"),
    "moonshot/": ("moonshotai", "MoonshotAI"),
}
SMOLPROXY_PREFIX = {
    "openai/": ("openai", "OpenAI"),
}

def public_id_name(row: dict) -> tuple[str, str]:
    source = row.get("source") or ""
    if source in SOURCE_OVERRIDE:
        return SOURCE_OVERRIDE[source]
    owned = str(row.get("owned_by") or "").strip()
    if "." in owned or "$" in owned:
        raise SystemExit(f"owned_by {owned!r} is not a safe public id")
    if not owned or owned in HIDDEN_HOPS:
        if source in ("jewproxy", "smolproxy"):
            # prefix fallback, then fail
            ...
        raise SystemExit(f"{row.get('id') or source}: cannot derive public id from owned_by={owned!r}")
    name = OWNED_BY_DISPLAY.get(owned) or owned.replace("-", " ").title()
    return owned, name
```

Do **not** keep a `SOURCE_TO_PUBLIC` map keyed by hop. First-party `openai` / `google` / `nvidia` where `owned_by == source` is correct (they are the lab).

4. Move model-level `vision_provider` onto `providers[0].vision_provider` (do not keep both after write).
5. Recompute top-level `tokens`, `pricing`, `metadata.vision` via `derive_listing_fields`.
6. Keep top-level `source` / `baseModel` for one release (shim + leftover media routes). Task 10 deletes them.
7. Media / TTS / image / video: one-element `providers[]`. Do not invent peers.
8. `gpt-5.3-codex-low` stays `public: false`.
9. Effort-suffix keys (`gpt-5.5-high`, `gpt-5.3-codex-low`, …) are **separate rows**. Wrap each from its own `source`. Do not copy a parent’s `providers[]`.
10. `id` unique within a model. `id` must not contain `.` or `$`. Script exits 1 on violation.
11. Write `indent=4`, `ensure_ascii=False`, trailing newline, LF.

- [x] **Step 1: Write the script + a dry-run mode** (`--check` exits 0 if every row already has `providers`)
- [x] **Step 2: Run** `python misc/migrate_models_providers.py --check` — expect non-zero before migration
- [x] **Step 3: Run** `python misc/migrate_models_providers.py` then `--check` — expect 0
- [x] **Step 4: Contract tests**

```python
from src.utils import AI_MODELS
from src.providers.runtime.route.pricing import derive_listing_fields
from src.providers.runtime.route.resolve import providers_of

def test_every_row_has_providers():
    missing = [mid for mid, row in AI_MODELS.items() if not providers_of(row)]
    assert not missing

def test_listing_fields_match_derived():
    bad = []
    for mid, row in AI_MODELS.items():
        derived = derive_listing_fields(providers_of(row))
        if row.get("tokens") != derived["tokens"]:
            bad.append(f"{mid} tokens")
        if row.get("pricing") != derived["pricing"]:
            bad.append(f"{mid} pricing")
        meta = row.get("metadata") or {}
        if "/v1/chat/completions" in row.get("endpoints", []) and bool(meta.get("vision")) != derived["metadata_vision"]:
            bad.append(f"{mid} vision")
    assert not bad, bad[:20]

def test_kimi_k26_vision_stays_on_the_gonka_row():
    rows = providers_of(AI_MODELS["kimi-k2.6"])
    gonka = next(r for r in rows if r["source"] == "gonka")
    assert gonka["vision_provider"]["source"] == "inferhub"
    assert gonka["vision_provider"]["baseModel"] == "kimi-k2.6"

HIDDEN_HOPS = frozenset({
    "jewproxy", "smolproxy", "partyrock", "salesforce", "athina", "kilo",
    "venice", "verboo", "albert", "inferhub", "surplusintelligence",
    "exa", "e2b", "notegpt", "comparia", "dahl", "telnyx", "zeroeval",
    "llm7", "k2think", "hotbot",
})

def test_public_id_is_owned_by_except_gonka():
    bad = []
    for mid, row in AI_MODELS.items():
        for p in providers_of(row):
            if p["id"] in HIDDEN_HOPS:
                bad.append(f"{mid} leaked hop id={p['id']}")
            if p["source"] == "gonka" and p["id"] != "gnk":
                bad.append(f"{mid} gonka id={p['id']}")
            elif p["source"] != "gonka" and p["id"] != row.get("owned_by"):
                bad.append(f"{mid} id={p['id']} owned_by={row.get('owned_by')}")
    assert not bad, bad[:20]
```

- [x] **Step 5: Update `test_vision_provider_keeps_the_effort_suffix`** to read `providers_of(entry)[n]["vision_provider"]` (kimi-k3 / gemini-3.6-flash).
- [x] **Step 6: Run** `python -m pytest src/utils/tests/test_reasoning_effort_offline.py src/utils/tests/test_cache_pricing_offline.py src/providers/runtime/route/tests -v`

- [x] **Step 7: Audit sweep**

- `--check` exits 0. No `id` is in `HIDDEN_HOPS`. Gonka rows are `gnk`. Every other row’s `id` equals `owned_by`.
- `test_public_id_is_owned_by_except_gonka` is the source of truth for public labels.
- `test_listing_fields_match_derived` is the source of truth for top-level `tokens` / `pricing` / `metadata.vision`. Do not hand-edit those fields after this.
- Update `test_native_effort_models_stay_on_a_provider` if it still reads `entry["source"]` and now fails.
- Progress table: Task 6 `[x]`.

---

### Task 7: Public listing + provider header

**Files:**
- Modify: `src/routes/normal/model.py` `_build_public_model_dict`
- Modify: chat / messages / responses — set `X-ElectronHub-Provider: {route.provider_id}` on both stream and JSON responses (add to the same header map that already carries `X-RateLimit-*` if that is where response headers are attached; otherwise set on `StreamingResponse` / `JSONResponse` headers directly)
- Test: new `src/routes/normal/tests/test_models_public_shape_offline.py` (or under `src/utils/tests/` if that folder is where route-shape tests already live)

**Interfaces:**
- Consumes: `providers_of`.
- Produces: public dict with sanitized `providers`.

```python
def _public_providers(values: dict) -> list[dict]:
    out = []
    for row in providers_of(values):
        item = {
            "id": row["id"],
            "name": row.get("name") or row["id"],
            "tokens": row["tokens"],
            "pricing": row.get("pricing") or {},
        }
        assert "source" not in item and "baseModel" not in item
        out.append(item)
    return out
```

TTS/STT: `row0 = providers_of(values)[0]`; use `row0["source"]` / `row0["baseModel"]` instead of `values.get("source")`.

- [x] **Step 1: Unit-test `_build_public_model_dict` on a fixture row** — public JSON has `providers`, no `source` / `baseModel` / `vision_provider`
- [x] **Step 2: Implement projection**
- [x] **Step 3: Add header on the three text routes**

Set `X-ElectronHub-Provider: {route.provider_id}` on the same header map `streaming_response` already builds (~1506) and on `JSONResponse` headers in `non_streaming_response`. Tools `StreamingResponse` (~651) must get it too. After Task 13 the value is the **winning** id because the walk finishes before these constructors.

- [x] **Step 4: Run the new test + `python -m pytest src/utils/tests/test_json_response_offline.py -v`**

- [x] **Step 5: Audit sweep**

- Public fixture JSON has `providers` and does **not** contain `source`, `baseModel`, `vision_provider`, `jewproxy`, `smolproxy`.
- Header uses `route.provider_id`, not `route.source`.
- Progress table: Task 7 `[x]`.

---

### Task 8: Overflow de-dupe

**Files:**
- Modify: `src/providers/runtime/overflow/zai.py` (and the Z.AI dispatch in `src/providers/prod/zai/api.py` that calls `map_zai_to_fallback`)
- Test: `src/providers/runtime/overflow/` existing offline tests if present; else add `src/providers/runtime/route/tests/test_overflow_dedupe_offline.py`

**Rule:** if `providers_of(model_data)` already contains a row with the same `(source, baseModel)` as a fallback dest, drop that dest from `map_zai_to_fallback` for this request. Venice / Chutes TEE stay in the map until they are real billed `providers[]` peers.

```python
def fallbacks_not_in_catalog(zai_model: str, model_data: dict) -> list:
    listed = {(r["source"], r["baseModel"]) for r in providers_of(model_data)}
    return [pair for pair in map_zai_to_fallback(zai_model) if pair not in listed]
```

Do **not** change ionet-nvidia / partyrock / telnyx / jewproxy-context maps in this task.

This task is **not** the HTTP provider walk. Overflow stays an intra-`source` dest swap. A listed `(source, baseModel)` must not be tried twice (once as a billed peer, once as overflow).

- [x] **Step 1: Test** — glm-5.2 with BytePlus already in `providers[]` does not return `("jewproxy", "glm/glm-5.2")` again
- [x] **Step 2: Implement + wire the Z.AI client to the filtered list**
- [x] **Step 3: Run overflow + route tests**

- [x] **Step 4: Audit sweep**

- Venice / Chutes TEE remain in `ZAI_FALLBACK_MODEL_MAP`.
- No new HTTP walker in `overflow/`.
- Progress table: Task 8 `[x]`.

---

### Task 9: Media routes + leftover readers

**Files:**
- Modify: `src/routes/normal/image.py` (two unpack sites ~127, ~456)
- Modify: `src/routes/normal/embedding.py` ~84
- Modify: `src/routes/normal/tts.py` ~91
- Modify: `src/routes/normal/audio.py` ~241
- Modify: `src/routes/normal/video.py` ~65
- Modify: `src/routes/normal/music.py` ~43
- Modify: `src/routes/normal/moderation.py` ~71
- Modify: `src/routes/normal/title_gen.py` ~64 (`meta.get("source")` / `meta.get("baseModel")`)
- Modify: `src/routes/normal/chat.py` in-chat image unpack ~828 (left on legacy in Task 4)
- Modify: `src/utils/__init__.py` `_UNREGISTERED_SOURCES` filter — a row is dropped if **every** `providers_of(row)` source is unregistered (or the legacy `source` is unregistered)
- Modify: `src/providers/tests/build_index_inventory.py` — count / source from `providers_of` (all sources, so a multi-source model is counted once per source). Do **not** regenerate `src/providers/INDEX.md` unless you also run the full inventory + `verify_indexes` pair.

Each media unpack becomes `resolve_route(modelData, has_images=False)` (image **generation** is not chat-vision). One-row models keep today’s behaviour. `music.py` binds `coefficient = route.pricing["coefficient"]`. Image / video / TTS / embedding `update_usage` uses `route.pricing`.

- [x] **Step 1: Switch each file to `ResolvedRoute`**
- [x] **Step 2: title_gen + in-chat image helper**
- [x] **Step 3: Run** `python -m pytest src/utils/tests/test_audio_offline.py src/utils/tests/test_image_token_billing_offline.py src/utils/tests/test_stt_upload_validation_offline.py -v`

- [x] **Step 4: Audit sweep**

- Grep `src/routes/normal/` for `modelData["source"]` / `["baseModel"]` / `["vision_provider"]`. Must be empty (title_gen and in-chat image included).
- Progress table: Task 9 `[x]`.

---

### Task 10: Drop legacy top-level fields

**Only after** Tasks 3, 4, 6, 7, 9 are green.

**Files:**
- Modify: `misc/migrate_models_providers.py` — add `--strip-legacy` that deletes top-level `source`, `baseModel`, `vision_provider` from every row that has `providers`
- Modify: `src/utils/models.json` via that flag
- Modify: leftover readers listed below **before** running `--strip-legacy`
- Test: `test_every_row_has_providers` plus `test_no_legacy_top_level_route_fields`

```python
LEGACY = ("source", "baseModel", "vision_provider")

def test_no_legacy_top_level_route_fields():
    leaked = [
        f"{mid}.{key}"
        for mid, row in AI_MODELS.items()
        for key in LEGACY
        if key in row
    ]
    assert not leaked
```

Grep the repo for `modelData["source"]`, `["baseModel"]`, `["vision_provider"]`, `entry["source"]`, `values.get("source")` and fix leftovers **before** `--strip-legacy`. Named leftovers (a post-strip writer that emits legacy-only fields undoes this task):

| File | What to do |
|------|------------|
| `src/utils/helpers.py` | Task 4 helpers must already handle missing top-level `source` / `baseModel`. Add a test: catalog row **without** those keys still keeps `max` on glm-5.2 via `providers_of`, and still applies `@high` when the overlay source is `inferhub`. |
| `src/utils/cache_usage.py` `_catalog_meta` | `baseModel = meta.get("baseModel") or (providers_of(meta)[0]["baseModel"] if providers_of(meta) else model_id)`. `owned_by` stays top-level. |
| `src/utils/__init__.py` | `_UNREGISTERED_SOURCES` filter uses `providers_of`. |
| `src/utils/tests/test_reasoning_effort_offline.py` | `test_vision_provider_keeps_the_effort_suffix`, `test_native_effort_models_stay_on_a_provider`, `test_declared_efforts_only_live_on_forwarding_providers` read `providers_of`. |
| `src/providers/prod/mistral/sync_models.py` | Emit `providers[]` via `public_id_name` (`id` from `owned_by`). Stop writing top-level `source` / `baseModel` / `vision_provider` as the only route fields. |
| `src/providers/prod/arliai/sync_models.py` | Same. `values.get("source") != "arliai"` skip-guards become “any `providers_of` source is arliai”. |
| `src/providers/prod/telnyx/{tts,stt}.py` `bootstrap_into_ai_models` | Dynamic rows get one-element `providers[]` (`id` from `owned_by`, `source=telnyx`). Never `id=telnyx` unless `owned_by` is the lab and that is intended. |
| `src/providers/prod/albert/models.py` | If this overlay is still imported as live rows, wrap the same way. |
| `src/routes/normal/model.py` | TTS/STT already via `providers_of[0]` (Task 7). |
| `src/routes/platform/stats.py` | `owned_by or source` → `owned_by` only (do not group by internal `source`). |
| `src/providers/tests/build_index_inventory.py` | Already on `providers_of` (Task 9). |
| Tests that fixture a bare `{"source": "zai", "baseModel": ...}` | Fine — shim still exists for **in-memory** dicts. Catalog JSON must not. |

Ignore false positives: `devpass.py` appeal `"source"`, provider-internal config, tool-adapter `"source"` image blocks.

- [ ] **Step 1: Grep + fix leftover readers (table above)**
- [ ] **Step 2: `--strip-legacy`**
- [ ] **Step 3: Run** `python -m pytest src/utils/tests/test_reasoning_effort_offline.py src/utils/tests/test_cache_pricing_offline.py src/providers/runtime/route/tests -v`
- [ ] **Step 4: `python -m src.providers.tests.generate_folder_indexes`** so `runtime/INDEX.md` lists `route/`. Do **not** run `generate_indexes` unless inventory was rebuilt.

- [ ] **Step 5: Audit sweep**

- `test_no_legacy_top_level_route_fields` is the catalog source of truth.
- Repo grep for `AI_MODELS[...]["source"]` / `["baseModel"]` / `["vision_provider"]` is empty (or only in the shim / comments).
- A dry-run of mistral/arliai/telnyx writers against a temp dict produces `providers` and does not re-add top-level route fields.
- Progress table: Task 10 `[x]`.

---

### Task 11: Pin syntax + account routing settings

**Files:**
- Create: `src/providers/runtime/route/pin.py` — `split_provider_model(model: str, catalog: dict) -> tuple[str, ProviderPrefs | None]`
- Modify: chat / messages / responses model lookup (after `_strip_model_prefix`; chat: after `:reasoning-exclude`, before `image_mode_check`)
- Modify: `src/manager/users/identity.py` — `get_routing` / `set_routing` (`routing: {only?: list[str], ignore?: list[str]}` on the user document). Not `profile.py`.
- Test: `src/providers/runtime/route/tests/test_pin_offline.py`

```python
def test_prefix_pin_selects_gnk_only():
    model, prefs = split_provider_model("gnk/kimi-k2.6", {"kimi-k2.6": KIMI})
    assert model == "kimi-k2.6"
    assert prefs.only == ["gnk"]
    assert prefs.allow_fallbacks is False
    route = resolve_route(KIMI, has_images=False, prefs=prefs)
    assert route.provider_id == "gnk"

def test_unknown_prefix_is_not_a_model_id():
    try:
        split_provider_model("not-a-slug/kimi-k2.6", {"kimi-k2.6": KIMI})
    except RouteError as exc:
        assert exc.code == "unknown_provider_id"
    else:
        raise AssertionError("expected RouteError")

def test_account_ignore_merges_with_request():
    prefs = merge_prefs(account={"ignore": ["byteplus"]}, request=ProviderPrefs(sort="price"))
    route = resolve_route(GLM, has_images=False, prefs=prefs)
    assert route.provider_id == "zai"

def test_prefix_pin_keeps_in_chat_image_suffix():
    model, prefs = split_provider_model("gnk/kimi-k2.6@flux-2", {"kimi-k2.6": KIMI})
    assert model == "kimi-k2.6@flux-2"
    assert prefs.only == ["gnk"]
```

**Auth (shipped — three return dicts):**

The user doc is already loaded. `"routing": user.get("routing") or {}` is on:

- `src/moderation/auth.py` normal `auth()` return
- DevPass twin
- DevPass refusal stub

No second Mongo fetch on the hot path. Routes pass `merge_prefs(auth_data.get("routing") or {}, prefs_from_request(data), prefix=prefix_prefs)` into `resolve_route`.

**Settings API (same task, thin, beside thinking):**

`GET` + `PATCH /v1/users/settings/routing` in `src/routes/normal/settings.py`. Body `{ "only": ["gnk"], "ignore": ["byteplus"] }`. Master key only (same rule as thinking). Persist on the user document as `routing`. Validate each id is a non-empty string without `.` / `$`. Do not validate against the live catalog (ids are global slugs).

- [x] **Step 1: Write pin + merge_prefs tests (expect FAIL)**
- [x] **Step 2: Implement `split_provider_model` + `merge_prefs`**
- [x] **Step 3: Wire into the three text routes after `_strip_model_prefix`, before `AI_MODELS[model]`**
- [x] **Step 4: Expose `routing` on both auth return dicts; add GET/PATCH settings**
- [x] **Step 5: Run** `python -m pytest src/providers/runtime/route/tests/test_pin_offline.py src/providers/runtime/route/tests/test_resolve_offline.py -v`

- [x] **Step 6: Audit sweep**

- Prefix pin defaults `allow_fallbacks=false` and **stays false** even if the request body sets it true; body `only` stays true.
- `electronhub/gnk/kimi-k2.6` strips then pins. `gnk/kimi-k2.6@flux-2` pins and keeps the `@image` suffix (chat `image_mode_check` is after the pin).
- Unknown prefix → `param: provider` (same `RouteError` as body `only`).
- Account ignore ∪ request ignore; request `only` wins over account ignore.
- `auth_data["routing"]` present on all three auth return dicts. No extra `get_user` in chat/messages/responses.
- PATCH routing **replaces** `only` + `ignore` (omitted list → `[]`).
- Progress table: Task 11 `[x]`.

---

### Task 12: Per-provider analytics + request-log provider

**Files:**
- Modify: `src/manager/users/usage.py` — `update_usage(..., provider: str = None)` and `update_api_usage(..., provider_id: str = None)`
- Modify: `src/manager/users/request_logs.py` — `log_request(..., provider: str = None, pricing: dict = None)`
- Modify: `src/manager/users/helpers.py` `public_request_log_entry` — keep `provider`; keep sanitized `pricing` (`type`/`input`/`output`/`cache_read` only)
- Modify: `src/manager/model.py` — `increment_model_usage(..., provider_id: str = None)`; flush `$inc` `providers.{id}.usage/in/out`
- Modify: chat / messages / responses `update_usage` call sites — `pricing=route.pricing`, `provider=route.provider_id`
- Test: `src/manager/tests/test_usage_billing_offline.py`; `src/manager/tests/test_api_stats_batch_offline.py` if ModelManager flush tests live there

`update_usage` already bills from the `pricing` dict. The bug to prevent is callers still passing listing `modelData["pricing"]`.

```python
def test_public_request_log_keeps_provider_and_row_pricing():
    pub = public_request_log_entry({
        "model": "kimi-k2.6",
        "provider": "gnk",
        "pricing": {"type": "per_million_tokens", "input": 0.6, "output": 3, "cache_read": 0.16, "source": "gonka"},
        "credits": 0.01,
    })
    assert pub["provider"] == "gnk"
    assert pub["pricing"]["input"] == 0.6
    assert "source" not in pub["pricing"]

def test_update_api_usage_incs_model_and_provider_leaves(monkeypatch):
    captured = {}

    class Fake:
        async def update_one(self, filt, update, upsert=False):
            captured["inc"] = update["$inc"]

    monkeypatch.setattr(UserManager, "api_usage_db", Fake())
    await UserManager.update_api_usage(
        "u1", "kimi-k2.6", 10, 4, 0.01, provider_id="gnk",
    )
    inc = captured["inc"]
    assert inc["models.kimi-k2_6.requests"] == 1
    assert inc["models.kimi-k2_6.providers.gnk.requests"] == 1
    assert inc["models.kimi-k2_6.providers.gnk.input_tokens"] == 10
```

`update_api_usage` `$inc` (same upsert as today):

```
models.{sanitized}.requests / input_tokens / output_tokens / total_cost / cache_*
models.{sanitized}.providers.{provider_id}.<same five>
```

Omit the `providers.*` leaves when `provider_id` is missing (legacy callers / media until Task 9). Model totals remain the sum of traffic, not “unknown-provider only”.

`get_api_usage` / `GET /v1/user/models`: each model dict gains optional `providers: { gnk: {requests, ...}, ... }`. Keep `owned_by` as the catalog vendor. When reading, pass through the nested `providers` object already stored; do not re-derive it from the catalog.

`ModelManager` batch slot grows `providers: { id: {usage, in, out} }`. Flush writes those `$inc` paths on the same model document. Analytics-only (crash window already accepted in `model.py`). **Not** billing.

**Failed hops must not look like success.** Only the **winning** row writes `update_api_usage` / `increment_model_usage` / success `log_request`. A 429/5xx hop still `release()`s capacity (Task 13). Do not bill listing price on a hop that never completed. `status != "success"` logs may record the last attempted `provider_id` (optional); they must not increment the per-provider success counters.

`update_usage` already bills from the `pricing` dict. Thread `provider=route.provider_id` from the three text routes (and media after Task 9). Tool-adapter inner `update_usage` calls (jewproxy/smolproxy/tools) may omit `provider` until those adapters are taught; omit `providers.*` leaves when missing.

- [x] **Step 1: Extend public-log + api_usage increment tests**
- [x] **Step 2: Add kwargs; increment both leaves; strip internal keys from public logs**
- [x] **Step 3: Thread `route.provider_id` / `route.pricing` from the three text routes into `update_usage`**
- [x] **Step 4: Run** `python -m pytest src/manager/tests/test_usage_billing_offline.py src/manager/tests/test_request_logs_collection_offline.py src/manager/tests/test_update_usage_motor_future_offline.py -v`

- [x] **Step 5: Audit sweep**

- Grep chat/messages/responses `update_usage(` — every success path passes `pricing=route.pricing` (or the bound `pricing` from the route) and `provider=route.provider_id`.
- Public log has `provider`, sanitized `pricing`, no `source`.
- `/v1/user/models` still has `owned_by` + new `providers`.
- Progress table: Task 12 `[x]`.

---

### Task 12b: Provider calendar windows (no new collection)

**Locked after OpenRouter research (2026-08-16):** do **not** create `db.providers` for usage. OpenRouter’s public surfaces are:

| Their surface | Grain | Time | Provider role |
|---|---|---|---|
| `GET /api/v1/generation` | one request | `created_at` | `provider_name` + `provider_responses[]` |
| `GET /api/v1/activity` | date + model + `endpoint_id` | last 30 completed UTC days | `provider_name` is a **column** |
| `POST /api/v1/analytics/query` | ≤2 dimensions + granularity | minute / hour / day / week / month | `provider` is a filterable **dimension** |
| `GET /api/v1/key` `usage_daily/weekly/monthly` | one API key | spend-cap reset at midnight UTC | **not** hop analytics |

They never maintain a singleton provider document with rolling counters. `usage_daily` on a key is a credit limit, not a hop window.

**What Task 12 already shipped (keep):**

- Event: `request_logs.provider` + sanitized listing `pricing` (30-day TTL). This is their generation row.
- User lifetime: `api_usage.models.{m}.providers.{id}.{requests,input_tokens,output_tokens,total_cost,cache_*}`
- Platform lifetime: `ModelManager` `$inc` `providers.{id}.{usage,in,out}` on the **same** model document.

User-level model usage has **never** had daily/weekly/monthly. Do not add them on provider leaves either — subscriber time series is `request_logs` (same 30-day bound as `GET /activity`).

**What this task adds:** calendar windows on the existing platform provider leaves, same grain OpenRouter exposes (`kimi × gnk this week`).

**Files:**
- Modify: `src/manager/model.py` `_flush_pending` Op 2 — `$cond` rollover under `providers.{id}` using the **parent** `current_date` / `current_week` / `current_month` (hops reset in lockstep with the model)
- Modify: `src/manager/model.py` `get_top_models` / `get_all_model_usage_docs` projections — include `providers` so hub/stats can read hop windows
- Test: `src/manager/tests/test_usage_billing_offline.py` — flush writes `providers.gnk.usage_daily` and resets when `current_date` changes

```
providers.{id}.usage              # lifetime (already Task 12)
providers.{id}.usage_daily
providers.{id}.usage_weekly
providers.{id}.usage_monthly
providers.{id}.in / in_daily / in_weekly / in_monthly
providers.{id}.out / out_daily / out_weekly / out_monthly
```

Rules (do not weaken):

- No `db.providers` collection. No second flush loop. Same 2s batch, same two `UpdateOne`s per model.
- Reuse the model document’s date markers. Do **not** store per-hop `current_date` (marker explosion).
- Do **not** copy `buckets_15m`, `rpm`, or `rph` onto each hop (that is what blew ModelManager doc size; OpenRouter keeps short windows on rankings, Task 16).
- Hub-wide “gnk this month, all models”: sum `providers.gnk.usage_monthly` across model docs at **read** time. Catalog size is small.
- Failed hops still do not increment (Task 12 `provider_leaf` only on `status == "success"`).
- `api_usage` stays lifetime. `request_logs` stays the user-level time axis.

- [x] **Step 1: Write failing flush tests** (`usage_daily` incs; date change resets hop daily, not lifetime)
- [x] **Step 2: Extend Op 2 rollover for each pending `providers.{id}`**
- [x] **Step 3: Run** `python -m pytest src/manager/tests/test_usage_billing_offline.py src/manager/tests/test_request_logs_collection_offline.py -v`

- [x] **Step 4: Audit sweep**

- Grep `db["providers"]` / `provider_usage` — none.
- `buckets_15m` still model-only.
- Progress table: Task 12b `[x]`.

---

### Task 13: Optional weight + Mongo RPM / concurrency

**Files:**
- Modify: `src/providers/runtime/route/types.py` — optional `rpm` / `max_concurrency` on `ProviderRow` (`weight` + drain + `-weight` tie-break already shipped in Task 2 `resolve.py`; do not re-implement)
- Create: `src/providers/runtime/route/capacity.py` — `admit(provider_id, *, rpm, max_concurrency) -> bool`, `release(provider_id)`, `async def acquire_route(...) -> ResolvedRoute`
- Do **not** call admit from `resolve_route` (stays sync and Mongo-free). Capacity lives only in `acquire_route`.
- Test: `src/providers/runtime/route/tests/test_capacity_offline.py` (fake Motor collection). Reuse `test_weight_zero_is_drained_unless_named` in `test_resolve_offline.py` — do not duplicate.

```python
def test_zero_weight_is_drained_unless_pinned():
    data = {"providers": [
        {**GLM["providers"][1], "weight": 0},
        GLM["providers"][0],
    ]}
    assert resolve_route(data, has_images=False).provider_id == "zai"
    pinned = resolve_route(data, has_images=False, prefs=ProviderPrefs(only=["byteplus"]))
    assert pinned.provider_id == "byteplus"

def test_same_price_prefers_higher_weight():
    a = {"id": "a", "source": "zai", "baseModel": "x", "tokens": 1000, "pricing": {"type": "per_million_tokens", "input": 1, "output": 1}, "weight": 1}
    b = {**a, "id": "b", "weight": 5}
    assert resolve_route({"providers": [a, b]}, has_images=False).provider_id == "b"

async def test_admit_rejects_when_rpm_full():
    coll = FakeCapacity(docs={"gnk": {"rpm_count": 10, "rpm_window_start": now_minute, "in_flight": 0}})
    assert await admit("gnk", rpm=10, max_concurrency=None, coll=coll) is False

async def test_admit_is_global_across_models():
    # kimi-k2.6 and kimi-k2.7-code both id=gnk share one doc
    assert capacity_key("kimi-k2.6", "gnk") == "gnk"
    assert capacity_key("kimi-k2.7-code", "gnk") == "gnk"
```

Mongo doc (`db.provider_capacity`):

```
{ _id: "gnk", rpm_count: int, rpm_window_start: int, in_flight: int, updated_at: int }
```

Admit uses `find_one_and_update` (same family as `src/moderation/auth.py` `rate_limits` and `src/utils/lease.py`). If `rpm` and `max_concurrency` are both unset, skip the collection (no write). Release is a no-op when admit was skipped.

Do **not** add Redis. Do **not** add latency/throughput sort here (Task 16). Do **not** 429 the user when a row is at RPM — skip to the next eligible row. Do **not** use an in-process dict as the source of truth (3 Koyeb replicas).

**HTTP walk (this task, not Task 8):**

Create `src/providers/runtime/route/errors.py` with `HopErrorKind` + `is_retryable_hop_error` (table in **Production failover**). Task 14 imports this — do not copy the set.

Also export `wire_source(row, *, has_images) -> str` from `resolve.py` (vision dest `source` when `has_images`, else `row.source`).

`acquire_route` yields the next admitted row. The three text routes replace `resolve_route(...)` with a short loop **before** `streaming_response` / `non_streaming_response` / tools `StreamingResponse`:

1. `route = await acquire_route(...)` (capacity admit; Task 14 also skips ejected wire sources).
2. Obtain client / first upstream error. Classify with `HopErrorKind` (connect, 429/502/503/504, empty-after-hop-local-recovery, generic 500, context-length, fatal).
3. On retryable + `allow_fallbacks`: `release(route.provider_id)`, `record_failure` except `context_length` (Task 14; no-op until then), next row. Do not write analytics.
4. On success (or non-retryable / last row / first token about to be sent): `record_success` (Task 14), construct the response with `X-ElectronHub-Provider: route.provider_id`.
5. `finally`: `release` if this request still holds a slot (stream end, disconnect, error).

Do not walk inside `generate_chunks` after `DisconnectAwareStreamingResponse` is constructed. Do not walk after a tool adapter has called `update_usage`. Empty-200 is retryable only after the hop’s own `should_overflow` / thinking-off path has already run and still produced no visible token.

- [x] **Step 1: Write fake-Mongo capacity + HTTP-walk tests** (weight drain/tie-break already green in `test_resolve_offline.py`)
- [x] **Step 2: Implement admit/release + `acquire_route`** (do not re-implement drain/tie-break; do not admit inside `resolve_route`)
- [x] **Step 3: Switch Task 4 call sites from `resolve_route` to `acquire_route`; release in `finally`; HTTP walk before StreamingResponse**
- [x] **Step 4: Run** `python -m pytest src/providers/runtime/route/tests -v`

- [x] **Step 5: Audit sweep**

- No in-process RPM dict. Capacity key is public `id` only (`gnk` shared across models).
- Header is the winning id. Failed hops did not increment `api_usage` / `ModelManager`.
- `resolve_route` remains sync and Mongo-free (offline tests still call it).
- Progress table: Task 13 `[x]`.

---

### Task 14: Mongo hop health (circuit breaker + decay)

**Files:**
- Create: `src/providers/runtime/route/health.py` — `is_ejected`, `record_failure`, `record_success`, `HEALTH_BASE_EJECTION_MS = 30_000`, `HEALTH_MAX_EJECTION_MS = 300_000`
- Modify: `src/providers/runtime/route/capacity.py` `acquire_route` — skip ejected wire sources (deprioritize), panic-bypass, call record_* 
- Modify: `src/providers/runtime/route/__init__.py` — do **not** need to export health helpers unless a test imports the package
- Test: `src/providers/runtime/route/tests/test_health_offline.py` (fake Motor collection)

**Interfaces:**
- Consumes: Task 13 `is_retryable_hop_error`, `wire_source`, `acquire_route`.
- Produces: Mongo `db.provider_health` keyed by wire `source`.

```
{ _id: "inferhub", fail_count: int, ejected_until: int, ejection_streak: int, last_kind: str, updated_at: int }
```

```python
async def is_ejected(source: str, *, now_ms: int | None = None, coll=None) -> bool
async def record_failure(source: str, *, kind: HopErrorKind, now_ms: int | None = None, coll=None) -> None
async def record_success(source: str, *, now_ms: int | None = None, coll=None) -> None
```

Rules (do not weaken):

- Key = `wire_source`, never public `id`, never catalog `weight`.
- Immediate eject kinds: `429`, `502`, `503`, `504`, `connect`. `ejection_streak += 1`; `ejected_until = now + min(300_000, 30_000 * streak)`.
- Threshold kinds: `empty` ejects at `fail_count >= 3`; `500` at `>= 2`. Other kinds do not increment `fail_count`.
- `context_length` and `fatal`: `record_failure` is a no-op (callers should not call it; the function must still ignore them).
- `is_ejected` is `now_ms < ejected_until`. Expired = half-open (not ejected).
- `record_success`: `fail_count = 0`; if half-open or healthy, `ejection_streak = max(0, streak - 1)` and `ejected_until = 0`.
- Panic-bypass: if every remaining eligible row’s `wire_source` is ejected, ignore health for this request.
- Pin (`allow_fallbacks=false`): never skip the pinned row for health.
- Mongo error: fail-open (`is_ejected` → False; `record_*` swallows).
- One failure write per `(request, source)` when the hop is abandoned. Thinking-off retries are not writes.
- No Redis. No in-process dict as source of truth. No inverse-square. No latency sort.

- [x] **Step 1: Write the failing tests**

```python
import pytest
from src.providers.runtime.route.health import (
    HEALTH_BASE_EJECTION_MS,
    is_ejected,
    record_failure,
    record_success,
)

@pytest.mark.asyncio
async def test_429_ejects_immediately(coll):
    await record_failure("gonka", kind="429", now_ms=1_000, coll=coll)
    assert await is_ejected("gonka", now_ms=1_000, coll=coll) is True
    assert await is_ejected("gonka", now_ms=1_000 + HEALTH_BASE_EJECTION_MS, coll=coll) is False

@pytest.mark.asyncio
async def test_first_500_does_not_eject(coll):
    await record_failure("zai", kind="500", now_ms=1, coll=coll)
    assert await is_ejected("zai", now_ms=1, coll=coll) is False
    await record_failure("zai", kind="500", now_ms=2, coll=coll)
    assert await is_ejected("zai", now_ms=2, coll=coll) is True

@pytest.mark.asyncio
async def test_context_length_does_not_eject(coll):
    await record_failure("jewproxy", kind="context_length", now_ms=1, coll=coll)
    assert await is_ejected("jewproxy", now_ms=1, coll=coll) is False

@pytest.mark.asyncio
async def test_success_decays_streak(coll):
    await record_failure("gonka", kind="429", now_ms=1, coll=coll)
    await record_success("gonka", now_ms=1 + HEALTH_BASE_EJECTION_MS, coll=coll)
    assert (await coll.find_one({"_id": "gonka"}))["ejection_streak"] == 0

@pytest.mark.asyncio
async def test_second_eject_lasts_longer(coll):
    await record_failure("gonka", kind="502", now_ms=1, coll=coll)
    await record_failure("gonka", kind="502", now_ms=2, coll=coll)
    # streak 2 → 60s
    assert await is_ejected("gonka", now_ms=2 + HEALTH_BASE_EJECTION_MS, coll=coll) is True
    assert await is_ejected("gonka", now_ms=2 + 2 * HEALTH_BASE_EJECTION_MS, coll=coll) is False

@pytest.mark.asyncio
async def test_vision_wire_source_is_dest_not_row_id():
    from src.providers.runtime.route.resolve import wire_source
    row = {"id": "gnk", "source": "gonka", "vision_provider": {"source": "inferhub", "baseModel": "kimi-k2.6"}}
    assert wire_source(row, has_images=False) == "gonka"
    assert wire_source(row, has_images=True) == "inferhub"
```

Also add an `acquire_route` test: ejected cheap hop is skipped; when every hop is ejected, the cheap hop is tried (panic-bypass).

- [x] **Step 2: Run tests — expect FAIL, then implement, then PASS**

Run: `python -m pytest src/providers/runtime/route/tests/test_health_offline.py src/providers/runtime/route/tests/test_capacity_offline.py src/providers/runtime/route/tests/test_resolve_offline.py src/providers/runtime/route/tests/test_pricing_offline.py -v`

- [x] **Step 3: Audit sweep**

- Grep `providers[].weight` writes — only catalog / Task 6. Health never assigns `weight`.
- Health key is `source`, capacity key is public `id`. Do not merge the collections.
- `is_retryable_hop_error` has one definition.
- Fail-open on Mongo errors is tested.
- Progress table: Task 14 `[x]`.

---

### Task 15: OpenRouter pref contract + `max_price`

**Why:** OpenRouter clients already send `preferred_*`, `max_price`, `zdr`. `extra="ignore"` drops those keys. `sort: "latency"` is already 422 via `Literal` — do not claim it cheapest-routes. `max_price` is the one remaining hard filter we can honor from catalog `row.pricing` with no live stats.

**Files:**
- Modify: `src/utils/openai_type.py` + anthropic twin — `ProviderPreferences`
- Modify: `src/providers/runtime/route/types.py` — `max_price` on `ProviderPrefs`
- Modify: `src/providers/runtime/route/resolve.py` — hard filter; new `RouteError("max_price")`
- Modify: `src/providers/runtime/route/resolve.py` `prefs_from_request`
- Modify: `src/routes/normal/bind_route.py` — map `max_price` → `param: "provider"`
- Test: `src/providers/runtime/route/tests/test_prefs_contract_offline.py`

```python
def test_sort_latency_is_rejected_until_perf_exists():
    from pydantic import ValidationError
    try:
        ProviderPreferences(sort="latency")
    except ValidationError:
        return
    raise AssertionError("sort=latency must not be accepted before Task 16")

def test_preferred_throughput_is_rejected():
    try:
        ProviderPreferences(preferred_min_throughput=50)
    except ValidationError:
        return
    raise AssertionError("preferred_* must be rejected until Task 16")

def test_max_price_skips_over_budget_row():
    prefs = ProviderPrefs(max_price={"prompt": 0.9, "completion": 3.1})
    route = resolve_route(GLM, has_images=False, prefs=prefs)
    assert route.provider_id == "byteplus"  # 0.8 / 3 vs zai 1 / 3.2

def test_max_price_none_eligible():
    try:
        resolve_route(GLM, has_images=False, prefs=ProviderPrefs(max_price={"prompt": 0.1}))
    except RouteError as exc:
        assert exc.code == "max_price"
    else:
        raise AssertionError("expected RouteError")
```

**Contract:**
- `extra="forbid"` **or** an explicit allow-list. Unknown keys **422** (Pydantic), not a Task 2 400.
- Accepted now: `order`, `only`, `ignore`, `allow_fallbacks`, `sort` in `{price, context}`, `max_price`.
- Rejected now: `sort` in `{latency, throughput}` and `sort` as object stay **422** (Pydantic `Literal` / type), same as any other invalid ChatData field. Extra keys (`preferred_*`, `quantizations`, `zdr`, `data_collection`, `require_parameters`, `enforce_distillable_text`) become **422** via `extra="forbid"` (or an explicit allow-list). Do not invent a second 400 dialect just for `provider.*` validation — FastAPI already 422s bad request bodies. `max_price` over-cap after a valid body is `RouteError("max_price")` / 400 `param: provider`.
- `max_price` keys: `prompt` / `completion` (USD per million, same unit as catalog `input` / `output`). Optional `request` later. Ignore `image` / `audio` in v1 (media routes do not read prefs).
- Compare against the **selected row’s** `pricing.input` / `pricing.output`. Coefficient-priced rows: if `max_price` is set, skip the row (cannot compare). Do not use listing `modelData["pricing"]`.
- If every eligible row is over cap → `RouteError("max_price")` / 400. Do not fall through to a more expensive hop.
- Do **not** implement `:nitro` / `:floor` here (`:nitro` would 400 via sort=throughput).

- [x] **Step 1: Write the four tests.** `test_sort_latency_is_rejected_until_perf_exists` already PASSES (Literal). The other three FAIL until this task (`preferred_*` extra-ignore, `max_price` missing).
- [x] **Step 2: Tighten `ProviderPreferences` + `max_price` filter**
- [x] **Step 3: Run** `python -m pytest src/providers/runtime/route/tests/test_prefs_contract_offline.py src/providers/runtime/route/tests/test_resolve_offline.py src/providers/runtime/route/tests/test_pin_offline.py -v`

- [x] **Step 4: Audit sweep**

- Grep `extra="ignore"` on `ProviderPreferences` — gone.
- `sort=latency` remains 422 (cannot cheapest-route). Extra keys 422. Over-cap `max_price` is 400 `RouteError`.
- Progress table: Task 15 `[x]`.

---

### Task 16: Hop performance windows + opt-in speed sort

**Depends on:** 12 (winning `provider_id` on the request), 13 (walk before first token), 14 (health). Do not start before 14 is `[x]`.

**Why:** OpenRouter’s `sort: "latency"` / `"throughput"` / `:nitro` and `preferred_*` need rolling p50/p90 of **this hop**, not model volume. Default rank still ignores these windows.

**Files:**
- Create: `src/providers/runtime/route/perf.py` — `record_hop_sample(...)`, `window_stats(*, model_id, provider_id) -> {ttft_ms, tok_s, uptime}`
- Create: `src/providers/runtime/route/tests/test_perf_offline.py`
- Modify: stream path (chat / messages / responses) — stamp start, first visible token, last token, output tokens, hop outcome class
- Modify: `ProviderPrefs.sort` — add `"latency" | "throughput"`
- Modify: `split_provider_model` / a sibling suffix helper — `:nitro` → sort throughput, `:floor` → sort price (after `electronhub/`, before `gnk/` pin, after `:reasoning-exclude`)
- Modify: `resolve.py` `_rank_key` — latency = lowest p50 TTFT; throughput = highest p50 tok/s; missing window sorts **after** hops that have one (OpenRouter “insufficient data” middle tier)
- Optional: account `routing.sort` merged in `merge_prefs` when request sort is unset

Mongo `db.provider_perf` (not `provider_health`, not `provider_capacity`):

```
{ _id: "<model_id>::<provider_id>",  # kimi-k2.6::gnk — NOT public id alone
  model_id: "kimi-k2.6",
  provider_id: "gnk",
  samples: [ { ts, ttft_ms, tok_s, ok: bool, kind: str, source: str } ],
  updated_at: int }
```

Capacity stays provider-global (`gnk`). Health stays wire `source`. Perf does not. `kimi-k2.6 × gnk` and `deepseek-v4-flash × gnk` are different endpoints. Old `_id: gnk` samples cannot be attributed — `purge_legacy_provider_perf` deletes them on startup. Do **not** copy them onto model keys. Do **not** fall back to provider-only docs on read. `perf_key` returns `None` if either side is empty or contains `::`. Never parse `_id` for logic.

Formulas (copy OpenRouter, do not invent a third):

- TTFT = first **visible** token − hop start (clock starts immediately before `send_message`, not tokenize/bind). Reasoning / `<think>` does not count. Whitespace-only content does not count. Tool-call payload counts as visible. Reasoning-only streams use stream-end as TTFT.
- tok/s = `output_tokens / max(generation_s, 0.001)` where generation_s is hop start → last upstream token (fetch + queue + TTFT + stream). **Not** `elapsed - ttft`. Omit `tok_s` when `output_tokens == 0`. Call `finish()` when the LLM loop ends, before image-gen / keepalive sleep.
- Uptime window = `ok / (ok + provider_error)`. Provider error = Task 13 kinds `{429,502,503,504,connect,empty,500}` plus mid-stream after first token. **Exclude** user 400, `context_length`, `fatal` 401/403/404. 429 counts as provider_error for uptime here (OpenRouter tracks 429 separately and excludes it from the published % — we store `kind` so both views are possible; published uptime excludes 429).
- Windows: 5m / 30m / 1d. Percentiles p50/p75/p90/p99. Need ≥20 samples in-window before `sort=latency|throughput` uses the hop (else “insufficient data”).
- Writer is fail-open: Mongo error does not fail the user request.
- Key = catalog model × public `id` (`kimi-k2.6::gnk`). Sort still compares public ids **on that model**. Stamp `source` on the sample so we can debug vision remaps. Do **not** sort on wire source. In-chat image uses the catalog text id after `image_mode_check`. Latency/throughput without `model_id` does not call `window_stats`.

**Still forbidden in this task:**
- Reading perf in the **default** pick (unset sort).
- Inverse-square.
- Auto Exacto / tool-schema error rate as a ranker.
- Mutating catalog `weight`.
- Merging `provider_perf` into `provider_health`.

```python
def test_sort_latency_picks_faster_ttft(monkeypatch):
    monkeypatch.setattr("src.providers.runtime.route.perf.window_stats", lambda **kw: {
        "gnk": {"ttft_ms": {"p50": 800}, "samples": 40},
        "moonshotai": {"ttft_ms": {"p50": 200}, "samples": 40},
    }[kw["provider_id"]])
    route = resolve_route(KIMI, has_images=False, model_id="kimi-k2.6", prefs=ProviderPrefs(sort="latency"))
    assert route.provider_id == "moonshotai"

def test_default_still_ignores_faster_expensive_hop(monkeypatch):
    monkeypatch.setattr("src.providers.runtime.route.perf.window_stats", lambda **kw: {
        "zai": {"ttft_ms": {"p50": 100}, "tok_s": {"p50": 80}, "samples": 40},
        "byteplus": {"ttft_ms": {"p50": 900}, "tok_s": {"p50": 20}, "samples": 40},
    }[kw["provider_id"]])
    route = resolve_route(GLM, has_images=False)
    assert route.provider_id == "byteplus"

def test_nitro_suffix_is_throughput_sort():
    model, prefs = split_provider_model("kimi-k2.6:nitro", {"kimi-k2.6": KIMI})
    assert model == "kimi-k2.6"
    assert prefs.sort == "throughput"
```

- [x] **Step 1: Writer tests + default-still-cheapest test** (`test_perf_offline.py`; default rank still ignores perf)
- [x] **Step 2: Implement `record_hop_sample` + stream stamps** (chat / messages / responses; finish before image-gen / keepalive sleep)
- [x] **Step 3: Enable `sort` latency/throughput + `:nitro`/`:floor`; lift Task 15 rejects**
- [x] **Step 4: Run** `python -m pytest src/providers/runtime/route/tests/test_perf_offline.py src/providers/runtime/route/tests/test_prefs_contract_offline.py src/providers/runtime/route/tests/test_resolve_offline.py src/providers/runtime/route/tests/test_pin_offline.py -v`

- [x] **Step 5: Audit sweep**

- Default path greps `window_stats` / `provider_perf` — only inside `sort in {latency, throughput}` or `preferred_*`.
- Health and perf are different collections. Perf is model × public id, not public id alone.
- No leftover `window_stats(provider_id)` / `_id: provider_id` on perf. No provider-only read fallback.
- `:exacto` still 400.
- Progress table: Task 16 `[x]`.

---

## Out of scope

- Flattening `runtime/`
- Exposing `jewproxy` / `smolproxy` as public `id`s
- OpenRouter inverse-square random load-balance (cooldown in Task 14 is the reliability half; we do not pick randomly among stable peers)
- Mutating catalog `providers[].weight` as a live health score (operator drain only)
- Redis (capacity, health, perf, and analytics use Mongo, like leases and user rate limits)
- Per-provider split on `APIManager` global totals (`db.api` `type=global`)
- Rewriting `/v1/messages` and `/v1/responses` error dialects
- JewProxy `/v1/responses`
- Promoting Venice / Chutes TEE into `providers[]`
- Hand-editing individual models to add BytePlus / Moonshot peers (that is a later catalog PR; this plan only migrates the current single-source rows and preserves today’s vision remaps)
- Raising timeout constants
- Deleting `gpt-5.3-codex-low`
- Context-compression / middle-truncate (OpenRouter plugin). We fail 400 if no provider window fits.
- Tools as a resolve filter (v1.1: if `tools` is present, prefer rows whose `source` is in the tools registry). v1 accepts cheapest-even-if-prompt-sim. Auto Exacto / `:exacto` / tool-schema hop scoring stay out.
- `models[]` cross-model fallback, `sort.partition`, `openrouter/auto`
- Catalog flags we do not store: `quantizations`, `zdr`, `data_collection`, `require_parameters`, `enforce_distillable_text` (400 if requested; later catalog PR)
- Settings UI. Task 11 ships GET/PATCH `/v1/users/settings/routing` only.
- Using TTFT / tok/s / uptime % on the **default** pick (Task 16 is opt-in only)
- Falling back to provider-only `provider_perf` docs (`_id: gnk`) or copying those samples onto model keys
- Computing hop windows on `GET /v1/models/{id}/providers` (snapshot only; 1d uptime from `buckets_1m`, not a larger raw-sample cap)
- A `db.providers` usage/analytics collection (OpenRouter research 2026-08-16; Task 12b extends the model document instead)

---

## Self-review

**Spec coverage**

| Requirement | Task |
|-------------|------|
| `providers[]` with public `id` ≠ internal `source` | 1, 6 |
| Public `id` / `name` from `owned_by` (gonka → `gnk`) | 6 |
| Per-provider `vision_provider` remap (possibly other source) | 2, 6 |
| Top-level cheapest pricing + max tokens | 1, 6 |
| `metadata.vision` = any row has vision remap | 1, 6 |
| Default cheapest-among-eligible; context is a hard filter | 2 |
| OpenRouter `provider` request object | 5 |
| Pin via `gnk/kimi-k2.6` + account `routing` + settings GET/PATCH | 11 |
| `auth()` exposes `routing` (all three return dicts) | 11 |
| Per-provider user + platform analytics + request-log provider | 12 |
| Platform provider daily/weekly/monthly on the model doc (no `db.providers`) | 12b |
| Failed hops do not increment success counters | 12, 13 |
| Weight + Mongo RPM/concurrency across Koyeb replicas | 13 |
| HTTP walk before first token / before StreamingResponse | 13 |
| Retryable classifier (`is_retryable_hop_error`) including empty-200 + 500 | 13 |
| Mongo hop health: 30s×streak eject, decay, panic-bypass, wire-`source` key | 14 |
| Silent-ignore of unimplemented `provider.*` is forbidden | 15 |
| Hard `max_price` from catalog row pricing | 15 |
| Hop TTFT / tok/s / uptime windows; opt-in `sort` latency\|throughput; `:nitro`/`:floor` | 16 |
| Hide internals on `/v1/models` | 7 |
| Incremental migrate via shim | 3, 6, 10 |
| Overflow not double-hitting listed dests | 8 |
| Media + title_gen + in-chat image | 6, 9 |
| Native effort survives Task 10 strip | 4, 10 |
| Catalog writers (mistral/arliai/telnyx) emit `providers[]` | 10 |
| `public: false` prune + keep Codex-low | 0 |
| Per-task audit sweep (source of truth) | every task, last step |

**Inconsistencies closed this revision**

- Failover owner was Task 5→8, algorithm→13, and gap→4/5. Now: rank in 2, overflow de-dupe in 8, HTTP+capacity walk in 13, hop health in 14.
- `resolve_route` no longer claims to drop live RPM (that is Mongo, Task 13).
- Weight-0 pin is `only` **or** `order` (any position), not “order first item” only.
- Task 4 no longer offers “resolve twice”. Tokenize-then-resolve is locked; TAGS refine cannot re-resolve.
- `allow_fallbacks` defaults: prefix false, body `only` true (unchanged) — Task 5 no longer assigns the walk to Task 8.
- Native-effort keep vs apply split is written into Task 4, not a leftover note.
- Settings GET/PATCH is in Task 11 (was “can wait”).
- Leftover readers are a Task 10 table, including `title_gen.py`, `stats.py`, `cache_usage.py`, sync/bootstrap writers.
- Public labels are `owned_by`, not a per-hop brand map. Gonka is the only override (`gnk`). Hidden hops never appear as `id`.
- Failover research (2026-08-16): same-request walk (13) + Mongo cooldown (14). Catalog `weight` is never a live score. Health key is wire `source`. Context-length walks but does not eject. Empty/500 use thresholds. Panic-bypass + pin override. Inverse-square stays out.
- OpenRouter routing research (2026-08-16): default is price-weighted among recently-stable hops, **not** latency/tok/s. Speed sort is opt-in. `sort: "latency"` is already 422; `extra="ignore"` still swallows `preferred_*` / `max_price` / `zdr` — Task 15 forbids those extras and implements `max_price`. Task 16 is the only reader of hop windows, and only when `sort`/`preferred_*` is set. Auto Exacto / `models[]` / inverse-square stay out.
- Pin + `@image` (2026-08-16 audit): `split_provider_model` partitions `@` first. Prefix `allow_fallbacks` cannot be reopened. Unknown prefix `param: provider`. Context 400 stays the Task 2 generic envelope. Weight drain already lives in Task 2; Task 13 must not admit inside `resolve_route`.
- Provider analytics (2026-08-16): OpenRouter stamps the hop on every generation and rolls **date × model × endpoint** rows. They do not give providers their own usage Mongo. Task 12 is lifetime leaves + `request_logs.provider`. Task 12b adds calendar windows on those same `db.models` leaves. `usage_daily` on OpenRouter **keys** is a spend cap — do not clone that onto a provider collection.

**Placeholder scan:** none of TBD / “handle edge cases” / “tests for the above” without code.

**Type consistency:** `ResolvedRoute`, `ProviderPrefs`, `RouteError`, `providers_of`, `resolve_route`, `acquire_route`, `cheapest_pricing`, `derive_listing_fields`, `merge_prefs`, `prefs_from_request`, `split_provider_model`, `is_retryable_hop_error`, `HopErrorKind`, `wire_source`, `is_ejected`, `record_failure`, `record_success`, `record_hop_sample`, `window_stats` are named the same in every task.

Intentionally not on the default path: latency, tok/s, uptime %, Auto Exacto. `:floor`/`:nitro` are Task 16 aliases only. Still out: `previous_response_id` stickiness, `models[]`, inverse-square, mutating catalog `weight`, `APIManager` per-provider totals, Redis, tools-as-resolve-filter, `:exacto`. 30s-outage **deprioritize** is Task 14, not inverse-square pick.
