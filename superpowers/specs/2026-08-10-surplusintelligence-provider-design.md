# Surplus Intelligence Provider Design

Date: 2026-08-10  
Status: Approved for implementation  
Provider id: `surplusintelligence`

## Goal

Add an OpenAI-compatible marketplace provider for [Surplus Intelligence](https://www.surplusintelligence.ai/docs) that prioritizes **cheapest inference**, **low latency**, and **high reliability**, following the InferHub provider patterns (catalog, sticky routing, `@effort`, vision/tools/cache probing, detailed logs).

## Why not copy InferHub bids 1:1

InferHub requires client `x-max-*-price` bid headers across marketplace prefixes.  
Surplus Intelligence already routes cheapest-first server-side and exposes:

- `max_price_per_1m` / `X-Max-Price-Per-1M` (**USD per 1M tokens**, not microdollars)
- `provider` pin or allow-list (slugs: `venice`, `bankr`, `inferhub`, `zai`, …)
- Order book: `GET /api/markets/:model` (prices in **microdollars**)
- Response observability headers: `x-si-provider-family`, `x-si-marketplace-attempts`, `x-si-routing-decision-ms`, `x-si-cache-*`
- Body `cost.usd`, `reasoning_content`, `prompt_tokens_details.cached_tokens`

Live probes showed single-provider pins often return `503 no_healthy_sellers` even when the order book marks that seller healthy. **Allow-lists** succeed.

## Approach: Hybrid C

1. Build a cheapest-first ladder of **provider allow-list windows** from the healthy order book.
2. **Budget gate (InferHub-style):** require ≥ ``SURPLUSINTELLIGENCE_MIN_DISCOUNT_PCT`` (default **70**) off order-book `direct_price` on **both** IN and OUT. Drop near-list offers; if none qualify, **STOP** (empty ladder — do not burn budget).
3. Cap each attempt with `max_price_per_1m = min(window ceiling, budget ceiling)` (USD/$1M).
4. Soft-sticky the last successful `provider_family` (from response headers), with fail cooldowns and epsilon cheapest-first exploration.
5. Escalate window / price on `max_price_exceeded` / `no_healthy_sellers` / 5xx; keep first attempt as **one HTTP call** so SI can failover inside the allow-list. Chat path uses SI ``/min{N}/v1/chat/completions`` as defense-in-depth.

## Package

`src/providers/prod/surplusintelligence/`

| File | Role |
|------|------|
| `api.py` | `SurplusIntelligenceClient` streaming chat |
| `models.py` | In-memory `/v1/models` + buyer `/v1/buyer/me` credit gate; order book cache |
| `routing.py` | Offer → slug map, window ladder, price caps, error parse, vision helpers |
| `sticky.py` | Sticky last-good provider family + cooldowns + epsilon |
| `params.py` | `@effort` → `reasoning_effort` (same tokens as InferHub) |
| `probe.py` / `live_tests.py` | Live capability suite |
| `valid_keys.txt` | Buyer `inf_` key(s) only |
| `tests/` | Offline unit tests |

## Wiring

- `src/providers/client.py` → register `surplusintelligence`
- `src/tools/registry.py` → `openai_compat` + config keys/URL
- `src/tools/providers/openai_compat.py` → ladder path parallel to InferHub (price/provider headers, sticky, empty-stream retry)
- Local catalog only (like InferHub); selective `utils/models.json` entries are out of scope for v1 unless needed later

## Routing details

- Price unit conversion: order book microdollars `/ 1e6` → USD
- Sort key: `max(in,out)` then `in + 3*out` (matches live `$1` reject / `$10` fill)
- Default window size: 3 provider slugs
- Budget: pay at most `(100 - min_discount)%` of direct IN and OUT (default 30% of list). Env `0` disables the gate.
- `max_price_per_1m = min(unit_ceiling * 1.05, budget_unit * 1.05)`
- Last resort when budget on: unpinned **but still budget-capped** (no uncapped fill)
- Chat URL: `/min{N}/v1/chat/completions` when `N = MIN_DISCOUNT_PCT > 0`
- Vision: only image-capable catalog models; refuse empty vision ladder
- Long prompts: prefer cache-capable offers inside near-cheapest band when catalog advertises cache pricing

## Capabilities

| Feature | Behavior |
|---------|----------|
| `@effort` | Strip client-side; SI treats `model@high` as unknown model (404) |
| Reasoning | Forward `reasoning_effort`; set `include_reasoning=True` when effort set; wrap `reasoning_content` like InferHub |
| Vision | OpenAI `image_url` parts; vision ladder from catalog modalities |
| Tools | Native via `openai_compat` (live-verified) |
| Cache | Parse usage cache fields + log `x-si-cache-*` headers |
| Credits | Soft-gate on `/v1/buyer/me` (`credit_balance_usdc` micro-USD + `balance_usdc`) |
| Image | `POST /min{N}/v1/images/generations` (+ `/edits`); cheapest seller; sync. See [Image Generations](https://www.surplusintelligence.ai/docs/api-reference/image-generations) |
| Video | `POST /min{N}/v1/video/generations` then poll `GET …/:id` (+ `X-Job-Token`); cheapest seller; async T2V/I2V. See [Video Generations](https://www.surplusintelligence.ai/docs/api-reference/video-generations) |

Helpers live in `media.py` (`generate_images` / `generate_video`); client methods
`create_image` / `create_video` are wired through `routes/normal/image.py` and
`video.py`. `user_db` usage counters (`image_gen` / `image_edit` / `video_gen`)
need no schema change.

Media budget: `SURPLUSINTELLIGENCE_MEDIA_MIN_DISCOUNT_PCT` (default **50** — media
books often clear ~60–68% but miss chat's 70%). On
`minimum_discount_not_met` / `no_healthy_sellers`, one fallback to plain
`/v1/images|video|audio/...` (still SI cheapest-first).

| Audio TTS | `POST /min{N}/v1/audio/speech` — OpenAI body (`model`,`input`,`voice`,…) |
| Audio STT | `POST /min{N}/v1/audio/transcriptions` — multipart file, max 25MB |

Platform catalog packs (copy/paste into `utils/models.json` yourself):
`python -m src.providers.prod.surplusintelligence.export_platform_models`
writes `src/providers/prod/surplusintelligence/models/{chat,images,videos,audio_speech,audio_transcriptions,music,embeddings}.json`.

Pack rules: public **id / name / owned_by / description** use the original
model creator (OpenAI, Alibaba, Hexgrad, …) — never Venice / Surplus branding.
`baseModel` remains the SI wire id; `source` remains `surplusintelligence`.
Legacy SI aliases are skipped.

## Latency / soft gates

- ``/buyer/me`` is a soft balance gate only via **async aiohttp**
  (``ensure_usage`` / ``asyncio.create_task`` refresh — no threads).
  Short timeout (default 2.5s), stale-while-revalidate, negative cache on
  5xx, fail-open when unknown. Never block the event loop or chat path.
- On ``minimum_discount_not_met`` while ``/min{N}/`` is active: skip remaining
  allow-list windows and jump to ``unpinned-capped`` (path already enforces floor).
- Fill logs include ``save≈N% vs direct`` from catalog list prices.

## Logging

Every pick / escalate / sticky hit must log:

- attempt index, model, allow-list, `max_price_per_1m`, ask in/out USD
- reason: `sticky` / `epsilon` / `escalate_max_price` / `escalate_window` / `cooldown` / `empty_stream` / `http_N`
- on success: `x-si-provider-family`, marketplace attempts, routing ms, cache headers, `cost.usd`, save%

## Out of scope (v1)

- Responses / Anthropic skins as first-class client surfaces
- Image/video/music generation endpoints
- BYOK priority provider registration
- Admin key usage
- Full merge into `src/utils/models.json`

## Auth notes

- Inference: `Authorization: Bearer inf_…`
- Admin key is not used for chat; do not place it in `valid_keys.txt`
- Env override: `SURPLUSINTELLIGENCE_API_KEYS` (comma-separated)

## Verification

1. Offline: effort strip, slug map, ladder windows, sticky reorder
2. Live probe: plain chat, vision, tools (simple/parallel/round-trip), reasoning on GLM, cache fields when present, price-cap escalate, provider allow-list
3. Tools path through `openai_compat` with `provider=surplusintelligence`
