# MarketHub Provider Design

Date: 2026-08-10  
Status: Approved for implementation  
Provider id: `markethub`

## Goal

Add a top-level **marketplace intelligence hub** that quotes registered markets **in parallel**, picks the cheapest budget-qualified route, then streams from that market’s existing client — with **no reimplementation** of InferHub bids or Surplus Intelligence Hybrid-C ladders.

v1 markets:

- `inferhub` → `src/providers/prod/inferhub/`
- `surplusintelligence` → `src/providers/prod/surplusintelligence/`

Future markets register as thin adapters only.

## Decisions (locked)

| Topic | Choice |
|-------|--------|
| Selection | **Quote-then-route** — compare asks, call one backend |
| Quote latency | **Parallel** `asyncio.gather` + per-market timeout |
| Completion | **Sequential** on winner (no parallel completion race) |
| Naming | **Bare family** + optional `market:` pin (`C`) |
| Code reuse | Hub adapters call exported `quote_route` + existing `*Client.send_message` |
| Tools | Hub picks market, then **delegates** into existing `openai_compat` IH/SI paths |

## Package

```
src/providers/markethub/
  __init__.py
  api.py                 # MarketHubClient
  board.py               # parallel quote, score, order, escalate
  aliases.py             # parse pin / bare / @effort; resolve per-market ids
  sticky.py              # last-good market sticky (hub-level)
  types.py               # MarketQuote shared dataclass
  adapters/
    __init__.py          # registry of adapters
    base.py              # MarketAdapter protocol
    inferhub.py
    surplusintelligence.py
  tests/
    test_board_offline.py
    test_aliases_offline.py
```

## Model naming

1. Strip known `@effort` (`minimal|low|medium|high|none`) from the **rightmost** `@`.
2. If remaining id matches `^(inferhub|surplusintelligence):(.+)$` → **pin** that market; remainder is the market-local model id.
3. Else **bare** family (e.g. `kimi-k3`): each adapter resolves to a local id via:
   - optional override in `aliases.py` / `ALIASES` map
   - else exact catalog id
   - else first catalog row whose `family_key` matches
4. Pass `model_id` + effort back into the winning client as `model[@effort]` so each market’s `translate_model_params` runs unchanged.

Pins never quote other markets. Bare names quote all registered adapters that resolve.

## Quote contract (reuse surface)

Each market exports (or adapter wraps):

```python
def quote_route(
    model_id: str,
    *,
    require_image: bool = False,
    messages: Any = None,
) -> Optional[RouteQuote]:
    """Cheapest *available* head after that market's own filters.
    Must NOT apply market sticky reorder — sticky stays inside send_message.
    """
```

| Market | Implementation |
|--------|----------------|
| SI | `build_route_ladder(...)[0]` after budget gate / vision fallback; `None` if empty (STOP) |
| InferHub | `build_bid_ladder(...)[0]`; `None` if empty |

Shared quote fields (`markethub.types.MarketQuote`):

- `market: str`
- `model_id: str` (resolved local id)
- `ask_in` / `ask_out` (USD / 1M)
- `score = ask_in + 3 * ask_out` (common sort key)
- `supports_image` / `supports_cache`
- `label` (debug)

## Board algorithm (latency)

```
resolve(request) → pin?, effort, per-market model ids
warm in-memory catalogs for target markets
for each eligible adapter (parallel, timeout QUOTE_TIMEOUT_MS default 8000):
    await adapter.quote(..., session=aiohttp)  # no ThreadPoolExecutor
drop None / timeout / exception (log; do not fail the board)
on SI timeout: asyncio.create_task warm that model's order book
order by score asc; hub sticky may promote last-good market if
    its score <= best * (1 + STICKY_BAND) (default 5%)
for quote in ordered:
    stream child.send_message(model[@effort], ...)
    if any non-empty response/thinking chunk → success; sticky market; return
escalate to next quote
if all fail → yield {"response": ""}
```

Env knobs:

- `MARKETHUB_QUOTE_TIMEOUT_MS` (default `8000`; cold SI OB often 2.8–6.5s)
- `MARKETHUB_SI_ORDERBOOK_TIMEOUT_SEC` (default `7.0`; SI adapter OB budget)
- `MARKETHUB_STICKY_BAND` (default `0.05`)
- `MARKETHUB_STICKY_TTL_SEC` (default `900`, clamped 300–1800)
- `MARKETHUB_STICKY_EPSILON` (default `0.08` — force pure cheapest reorder)
- `MARKETHUB_MARKETS` (optional comma allow-list; default all registered)

## Execution reuse

`MarketHubClient` holds lazy child clients:

- `InferHubClient`
- `SurplusIntelligenceClient`

`send_message` signature and yield shape match other providers. Children keep their own ladders, budgets, balance gates, and sticky. Hub does **not** copy bid headers / allow-lists.

On child soft-fail (empty generator / only empty strings): escalate. Hub does not parse child HTTP errors (children already exhausted their ladders).

## Tools path

`tools/registry.py`:

- `"markethub": "openai_compat"`
- `get_provider_config("markethub")` → placeholder url + empty keys (unused after remap)

`openai_compat_tool_chunks`:

1. If `provider == "markethub"`: run board pick (same as chat).
2. Recurse **once** with `provider=<winner.market>` and `base_model=<winner.model_id>[@effort]`.
3. Existing IH/SI ladder branches handle the request — **zero** duplicated ladder code in hub tools.

Guard with an internal flag / only remap when provider is markethub so recursion cannot loop.

## Wiring

- `src/providers/client.py` → `"markethub": lambda: MarketHubClient()`
- `src/tools/registry.py` → tool type + config
- `src/tools/providers/openai_compat.py` → one-shot remap branch

## Edge cases (must hold)

| Case | Behavior |
|------|----------|
| SI budget STOP, IH quotes | IH wins |
| Both empty | empty response + error log |
| Pin SI near-list | SI only; may empty (no IH fallthrough) |
| Vision message | `require_image=True` on all quotes; markets apply own vision filters |
| `@high` bare | strip at hub parse; reattach for child |
| `inferhub:cb/foo@high` | pin IH, model `cb/foo`, effort high |
| Quote timeout on one market | other market still competes |
| Nested SI→InferHub seller | still one SI quote; billing is SI’s ask (acceptable) |
| Chat vs tools sticky | same key: `family\|modality\|effort\|pin` |
| Child balance / empty sentinel | swallow until first useful chunk; then commit (no escalate) |
| Mid-stream child failure after commit | stop (do not concatenate next market) |
| IH free/combo/unpriced $0 | excluded from cross-market board; allowed when pinned to InferHub |
| Unknown bare family | all quotes None → empty |
| Fail cooldown | scoped `market\|sticky_key` (not whole market) |

## Media (image / video)

InferHub has no OpenAI-compatible image/video surface today. Hub media methods
**delegate to Surplus Intelligence only** (min-discount cheapest seller):

- `create_image` → SI `POST /min{N}/v1/images/generations` or `/edits`
- `create_video` → SI async `POST /min{N}/v1/video/generations` + poll

Routes: `src/routes/normal/image.py` / `video.py` accept
`provider in ("surplusintelligence", "markethub")`. Platform catalog entries use
`source: surplusintelligence|markethub` (e.g. `si-venice-z-image-turbo`).

## Out of scope (v1)

- Parallel completion race
- Full auto-import of SI catalog into `utils/models.json` (sample media models only)
- Admin / BYOK
- New markets beyond IH + SI (adapter slot ready)
- Music / other SI media modalities

## Verification

1. Offline: alias parse/pin, score order, sticky band, pin isolation, remap guard
2. Offline: SI/IH `quote_route` returns head consistent with ladder[0]
3. Live smoke (optional): bare `kimi-k3` / `deepseek-v4-pro` fills; pin forces market
4. `python -m src.providers.markethub.api` smoke chat
