# Universal Runtime Per-Provider Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract shared OpenAI SSE parsing and matching key cursors without changing retry/overflow/thinking yield shapes.

**Architecture:** `AttemptRuntime.chat_chunks` stays gonka/dahl-only. Everyone else that speaks OpenAI SSE replaces the inner line loop with `yield_openai_sse_deltas` and keeps its own outer shell. Plugins do not change `stream_mode`.

**Tech Stack:** Python 3.11+, aiohttp, pytest, existing `src/providers/runtime/`

**Spec:** `docs/superpowers/specs/2026-08-15-universal-runtime-per-provider.md` (audit-corrected 2026-08-15)

**Shipped 2026-08-15:** Tasks 1–7. Live roster is **57** after pplx / together / siliconflow moved to `deprecated/` and `_UNREGISTERED_SOURCES`. together / pplx were extracted in Task 3, then unregistered — do not put them back on the thin-extract list. Task 8 is no-ops.

## Global Constraints

- Do not invent overflow destinations.
- Do not change `stream_buffered` flush for Verboo / InferHub / Surplus Intelligence.
- Do not add Gonka to `PROVIDER_STREAM_USAGE`.
- Do not revive `sse_chat.py` / `key_ring.py`.
- Do not edit `models.json`.
- Do not flatten `src/providers/runtime/`.
- Do not change `thinking=True` yield shape.
- Overflow paths must use `forward_agen`.
- Do not revert the 16 deprecations.
- INDEX only via `build_index_inventory` + `generate_indexes` + `verify_indexes`. Those write `src/providers/INDEX.md` and `src/tools/providers/INDEX.md` only — not `src/tools/INDEX.md`.
- Do not call `AttemptRuntime.chat_chunks` for any provider except gonka/dahl.
- Do not set `stream_mode="buffered"` on any name other than verboo / inferhub / surplusintelligence.
- Do not add `stream_options.include_usage` where it is absent.
- Do not add unused `ProviderProfile` fields (`max_http_retries`, `overflow_name`, `marketplace`).
- Do not delete `ProviderFlags` bools.
- Do not swap Albert / Feather / ArliAI onto `RoundRobinKeys` (increment-first skip).
- Stop after Task 2 for review. Do not start Tasks 3+ in the same session as Task 2.

## File map

| File | Role |
|---|---|
| `src/providers/prod/{verboo,kilo,inferhub,surplusintelligence,modal}/api.py` | `RoundRobinKeys` (matching-cursor families) |
| `src/providers/misc/zeroeval/api.py` | `RoundRobinKeys` |
| `src/providers/prod/verboo/api.py` | inner SSE extract |
| `src/providers/runtime/overflow/verboo.py` | optional mapper move + re-export |
| `src/providers/runtime/marketplace/__init__.py` | IH/SI session factory |
| `src/providers/tests/runtime/test_auth_cursor_offline.py` | Task 1 |
| `src/providers/prod/verboo/tests/test_inner_sse_offline.py` | Task 2 |
| `src/providers/tests/runtime/test_thin_extract_offline.py` | Task 3 |

---

### Task 1: RoundRobinKeys for matching cursors only

**Files:**
- Create: `src/providers/tests/runtime/test_auth_cursor_offline.py`
- Modify: `src/providers/prod/verboo/api.py`
- Modify: `src/providers/prod/kilo/api.py`
- Modify: `src/providers/prod/inferhub/api.py`
- Modify: `src/providers/prod/surplusintelligence/api.py`
- Modify: `src/providers/prod/modal/api.py`
- Modify: `src/providers/misc/zeroeval/api.py`

**Interfaces:**
- Consumes: `RoundRobinKeys.next() -> str`
- Produces: `VERBOO_KEYS`, `KILO_KEYS`, `INFERHUB_KEYS`, `SI_KEYS`, `MODAL_KEYS`, `ZEROEVAL_KEYS`

- [x] **Step 1: Write the failing test**

```python
# src/providers/tests/runtime/test_auth_cursor_offline.py
from src.providers.runtime.auth import RoundRobinKeys


def test_round_robin_cycles():
    rr = RoundRobinKeys(["a", "b", "c"])
    assert [rr.next() for _ in range(6)] == ["a", "b", "c", "a", "b", "c"]


def test_matching_families_use_named_store():
    from src.providers.prod.verboo import api as verboo
    from src.providers.prod.kilo import api as kilo
    from src.providers.prod.inferhub import api as inferhub
    from src.providers.prod.surplusintelligence import api as si
    from src.providers.prod.modal import api as modal
    from src.providers.misc.zeroeval import api as zeroeval

    assert isinstance(verboo.VERBOO_KEYS, RoundRobinKeys)
    assert isinstance(kilo.KILO_KEYS, RoundRobinKeys)
    assert isinstance(inferhub.INFERHUB_KEYS, RoundRobinKeys)
    assert isinstance(si.SI_KEYS, RoundRobinKeys)
    assert isinstance(modal.MODAL_KEYS, RoundRobinKeys)
    assert isinstance(zeroeval.ZEROEVAL_KEYS, RoundRobinKeys)
    assert not hasattr(verboo, "_CURRENT_INDEX")
    assert not hasattr(kilo, "_CURRENT_INDEX")


def test_increment_first_families_keep_int():
    from src.providers.prod.albert import api as albert
    from src.providers.prod.feather import api as feather
    from src.providers.prod.arliai import api as arliai

    assert hasattr(albert, "CURRENT_INDEX")
    assert hasattr(feather, "CURRENT_INDEX")
    assert hasattr(arliai, "CURRENT_INDEX")
    assert not hasattr(albert, "ALBERT_KEYS")
```

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/tests/runtime/test_auth_cursor_offline.py -v`

Expected: FAIL on `VERBOO_KEYS`

- [x] **Step 3: Write minimal implementation**

Verboo (return-then-increment, first call = key[0]):

```python
from src.providers.runtime.auth import RoundRobinKeys

VERBOO_KEYS = RoundRobinKeys(API_KEYS)

def _next_key() -> Optional[str]:
    key = VERBOO_KEYS.next()
    return key or None
```

Kilo / InferHub / SI / Modal / ZeroEval (start-at-`-1`, first call = key[0]): same swap. SI preferred-key pin stays outside `SI_KEYS.next()`.

Do not touch albert / feather / arliai / nvidia / openai / chutes / openrouter / groq.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/tests/runtime/test_auth_cursor_offline.py src/providers/tests/runtime/test_profile_offline.py -q`

Expected: PASS. Existing profile tests still pass (no new fields).

- [x] **Step 5: Commit**

```bash
git add src/providers/tests/runtime/test_auth_cursor_offline.py src/providers/prod/verboo/api.py src/providers/prod/kilo/api.py src/providers/prod/inferhub/api.py src/providers/prod/surplusintelligence/api.py src/providers/prod/modal/api.py src/providers/misc/zeroeval/api.py
git commit -m "refactor(auth): RoundRobinKeys for matching key-cursor families"
```

---

### Task 2: Verboo inner SSE extract (not AttemptRuntime)

**Files:**
- Create: `src/providers/prod/verboo/tests/test_inner_sse_offline.py`
- Optional create: `src/providers/runtime/overflow/verboo.py`
- Modify: `src/providers/prod/verboo/api.py`

**Interfaces:**
- Consumes: `yield_openai_sse_deltas(response, progress, think, held_usage)`
- Produces: same 5-key empty-200 loop + InferHub; usage emitted only after a visible attempt

- [x] **Step 1: Write the failing tests**

```python
# src/providers/prod/verboo/tests/test_inner_sse_offline.py
import inspect
from src.providers.prod.verboo import api as verboo


def test_send_message_does_not_construct_attempt_runtime():
    src = inspect.getsource(verboo.VerbooClient.send_message)
    assert "AttemptRuntime" not in src
    assert "yield_openai_sse_deltas" in src
    assert "MAX_RETRIES" in inspect.getsource(verboo)


class _EmptyResp:
    status = 200

    async def __aenter__(self):
        return self

    async def __aexit__(self, *a):
        return False

    @property
    def content(self):
        async def _gen():
            if False:
                yield b""
        return _gen()


class _EmptySession:
    def post(self, *a, **k):
        return _EmptyResp()

    async def close(self):
        return None


@pytest.mark.asyncio
async def test_empty_attempts_reach_inferhub(monkeypatch):
    calls = {"n": 0}

    async def _fake_ih(*a, **k):
        calls["n"] += 1
        yield {"response": "ih"}

    monkeypatch.setattr(verboo, "_inferhub_fallback", _fake_ih)
    monkeypatch.setattr(verboo, "API_KEYS", ["k1", "k2", "k3", "k4", "k5"])
    client = verboo.VerbooClient()
    client.session = _EmptySession()
    out = []
    async for c in client.send_message("pro/qwen3.6-27b", [{"role": "user", "content": "hi"}]):
        out.append(c)
    assert calls["n"] == 1
    assert out == [{"response": "ih"}]
```

Keep `src/providers/prod/gonka/tests/test_overflow_aclose_offline.py` passing. If `map_verboo_to_inferhub` moves, re-export it from `verboo.api`.

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/prod/verboo/tests/test_inner_sse_offline.py -v`

Expected: FAIL — `yield_openai_sse_deltas` not in `send_message`

- [x] **Step 3: Write minimal implementation**

Keep `for attempt in range(MAX_RETRIES)`, `_next_key()`, empty-200 continue, InferHub after exhaustion. Inside a 200 response:

```python
progress = StreamProgress()
think = ThinkWrap()
held_usage: list = []
async for chunk in yield_openai_sse_deltas(response, progress, think, held_usage):
    yield chunk
if think.open and not think.closed:
    yield {"response": "\n</think>\n\n"}
if progress.visible:
    if held_usage:
        yield held_usage[0]
    return
# else: next key; do not yield held_usage
```

Do not call `AttemptRuntime`. Do not change tools `stream_buffered`. Sleep stays `0.5 * (2 ** attempt)`.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/prod/verboo/tests src/providers/prod/gonka/tests/test_overflow_aclose_offline.py src/providers/tests/runtime/test_profile_offline.py -q`

Expected: PASS

- [x] **Step 5: Commit**

```bash
git add src/providers/prod/verboo/api.py src/providers/prod/verboo/tests/test_inner_sse_offline.py src/providers/runtime/overflow/verboo.py
git commit -m "refactor(verboo): extract inner OpenAI SSE without AttemptRuntime"
```

**Stop here for review.** Do not start Task 3 until the user approves.

---

### Task 3: Thin-client inner SSE extract

**Files:**
- Create: `src/providers/tests/runtime/test_thin_extract_offline.py`
- Modify: `src/providers/prod/albert/api.py` (thinking=False branch only)
- Modify: `src/providers/prod/feather/api.py` (thinking=False branch only)
- Modify: `src/providers/prod/groq/api.py` (chat only)
- Modify: `src/providers/prod/llm7/api.py`
- Modify: `src/providers/misc/cerebras/api.py`
- Modify: `src/providers/misc/xai/api.py`
- Modify: `src/providers/prod/modal/api.py` (inner loop only)

**Interfaces:**
- Consumes: `yield_openai_sse_deltas`
- Produces: unchanged outer retry / `raise_for_status` / payload / `thinking=True`

- [x] **Step 1: Write the failing test**

```python
# src/providers/tests/runtime/test_thin_extract_offline.py
import inspect

EXTRACT = [
    "src.providers.prod.albert.api",
    "src.providers.prod.feather.api",
    "src.providers.prod.groq.api",
    "src.providers.prod.llm7.api",
    "src.providers.misc.cerebras.api",
    "src.providers.misc.xai.api",
    "src.providers.prod.modal.api",
]


def test_thin_clients_extract_sse_not_attempt_runtime():
    for mod_name in EXTRACT:
        mod = __import__(mod_name, fromlist=["*"])
        src = inspect.getsource(mod)
        assert "yield_openai_sse_deltas" in src
        assert "AttemptRuntime" not in src
```

Albert/Feather: `thinking=True` path still contains `{"thinking"` and does not call the helper. Together: `convert_msg` still exists. Groq: `generate_audio` still exists; `playai-tts` ids unchanged. Albert: no new `include_usage`.

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/tests/runtime/test_thin_extract_offline.py -v`

Expected: FAIL — helper not imported

- [x] **Step 3: Write minimal implementation**

Replace only the default inner `async for line in response.content` OpenAI parse. Keep session `raise_for_status`, Albert `3**(i+1)` 429 sleep, Groq payload clamps, Together `convert_msg` + two-model `"<think>\n"` pre-yield, Modal outer key loop. After a successful inner gen, yield held usage if the client already requested `include_usage` (Feather, Groq, Together, Modal). Albert does not.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/tests/runtime/test_thin_extract_offline.py src/providers/tests/runtime/test_profile_offline.py -q`

Expected: PASS

- [x] **Step 5: Commit**

```bash
git add src/providers/tests/runtime/test_thin_extract_offline.py src/providers/prod/albert/api.py src/providers/prod/feather/api.py src/providers/prod/groq/api.py src/providers/prod/together/api.py src/providers/prod/llm7/api.py src/providers/prod/pplx/api.py src/providers/misc/cerebras/api.py src/providers/misc/xai/api.py src/providers/prod/modal/api.py
git commit -m "refactor(chat): extract inner OpenAI SSE from thin clients"
```

---

### Task 4: Partial extract (nvidia, arliai, cf chat) + ArliAI import hygiene

**Files:**
- Modify: `src/providers/prod/nvidia/api.py`
- Modify: `src/providers/prod/arliai/api.py`
- Modify: `src/providers/prod/cf/api.py` (chat path only)
- Create: `src/providers/prod/arliai/tests/test_import_no_asyncio_run_offline.py`

**Interfaces:**
- Consumes: `yield_openai_sse_deltas`
- Produces: NVIDIA cooldown/proxy unchanged; `ARLIAI_TO_ZAI_MODEL_MAP` unchanged; CF media unchanged

- [x] **Step 1: Write the failing test**

```python
def test_arliai_import_does_not_call_asyncio_run(monkeypatch):
    import asyncio
    import importlib
    called = []
    monkeypatch.setattr(asyncio, "run", lambda *a, **k: called.append(True))
    import src.providers.prod.arliai.api as arliai
    importlib.reload(arliai)
    assert called == []
```

ArliAI `thinking=True` still does not call the helper. CF `create_image` still present. No `AttemptRuntime` in these three modules.

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/prod/arliai/tests/test_import_no_asyncio_run_offline.py -v`

Expected: FAIL — import still hits `asyncio.run(_fetch_arliai_img_models())`

- [x] **Step 3: Write minimal implementation**

ArliAI: lazy-fetch image models on first image call. NVIDIA/CF/ArliAI chat: inner SSE only. Keep GLM semaphore overflow.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/prod/arliai/tests src/providers/tests/runtime/test_thin_extract_offline.py -q`

Expected: PASS

- [x] **Step 5: Commit**

```bash
git add src/providers/prod/nvidia/api.py src/providers/prod/arliai/api.py src/providers/prod/cf/api.py src/providers/prod/arliai/tests/test_import_no_asyncio_run_offline.py
git commit -m "refactor(chat): extract OpenAI SSE from nvidia, arliai, cf"
```

---

### Task 5: Marketplace session helper (no ladder wrap)

**Files:**
- Create: `src/providers/runtime/marketplace/__init__.py`
- Create: `src/providers/tests/runtime/test_marketplace_session_offline.py`
- Modify: `src/providers/prod/inferhub/api.py`
- Modify: `src/providers/prod/surplusintelligence/api.py`

**Interfaces:**
- Consumes: existing IH/SI timeout (`ssl=False`, total=300)
- Produces: `def marketplace_session() -> aiohttp.ClientSession`

- [x] **Step 1: Write the failing test**

```python
import aiohttp
from src.providers.runtime.marketplace import marketplace_session


def test_marketplace_session_defaults():
    session = marketplace_session()
    try:
        assert session.timeout.total == 300
        assert isinstance(session, aiohttp.ClientSession)
    finally:
        session._connector._closed or None
        # close without running a loop if possible
```

Do not read private `connector._ssl`. After creating the session, close it (`session.close()` needs a running loop — use `pytest.mark.asyncio` and `await session.close()`).

```python
@pytest.mark.asyncio
async def test_marketplace_session_defaults():
    from src.providers.runtime.marketplace import marketplace_session
    session = marketplace_session()
    try:
        assert session.timeout.total == 300
    finally:
        await session.close()
```

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/tests/runtime/test_marketplace_session_offline.py -v`

Expected: FAIL — module missing

- [x] **Step 3: Write minimal implementation**

```python
# src/providers/runtime/marketplace/__init__.py
import aiohttp

def marketplace_session() -> aiohttp.ClientSession:
    return aiohttp.ClientSession(
        connector=aiohttp.TCPConnector(ssl=False, enable_cleanup_closed=True),
        timeout=aiohttp.ClientTimeout(total=300),
    )
```

Point InferHub and SI session construction at this helper. Do not move the bid ladder.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/tests/runtime/test_marketplace_session_offline.py src/providers/tests/runtime/test_profile_offline.py -q`

Expected: PASS

- [x] **Step 5: Commit**

```bash
git add src/providers/runtime/marketplace/__init__.py src/providers/tests/runtime/test_marketplace_session_offline.py src/providers/prod/inferhub/api.py src/providers/prod/surplusintelligence/api.py
git commit -m "refactor(marketplace): share InferHub/SI session factory"
```

---

### Task 6: Microsoft import hygiene only

**Files:**
- Modify: `src/providers/prod/microsoft/api.py`
- Create: `src/providers/prod/microsoft/tests/test_import_no_asyncio_run_offline.py`

**Interfaces:**
- Consumes: existing `load_all_voices`
- Produces: first `generate_audio` call loads voices; import does not call `asyncio.run`

- [x] **Step 1: Write the failing test**

```python
def test_microsoft_import_does_not_call_asyncio_run(monkeypatch):
    import asyncio
    import importlib
    called = []
    monkeypatch.setattr(asyncio, "run", lambda *a, **k: called.append(True))
    import src.providers.prod.microsoft.api as microsoft
    importlib.reload(microsoft)
    assert called == []
```

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/prod/microsoft/tests/test_import_no_asyncio_run_offline.py -v`

Expected: FAIL — line 234 `asyncio.run(load_all_voices())`

- [x] **Step 3: Write minimal implementation**

Delete the module-level `asyncio.run(load_all_voices())`. Call `await load_all_voices()` from `generate_audio` if `AVAILABLE_VOICES` is empty. Do not change any plugin `stream_mode`.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/prod/microsoft/tests src/providers/tests/runtime/test_profile_offline.py -q`

Expected: PASS. `profile_for("venice").stream_mode == "passthrough"` still.

- [x] **Step 5: Commit**

```bash
git add src/providers/prod/microsoft/api.py src/providers/prod/microsoft/tests/test_import_no_asyncio_run_offline.py
git commit -m "fix(microsoft): lazy-load TTS voices without import-time asyncio.run"
```

---

### Task 7: INDEX regen (only if Tasks 1–6 changed registry/profile)

**Files:** generated INDEX files via the existing scripts

- [x] **Step 1:** `python -m src.providers.tests.build_index_inventory`
- [x] **Step 2:** `python -m src.providers.tests.generate_indexes`
- [x] **Step 3:** `python -m src.providers.tests.verify_indexes`
- [x] **Step 4:** Commit only if those two INDEX files or `index_inventory.json` changed

```bash
git add src/providers/tests/results/index_inventory.json src/providers/INDEX.md src/tools/providers/INDEX.md
git commit -m "docs(index): regenerate after runtime extract"
```

Do not add `src/tools/INDEX.md`. Do not hand-edit INDEX rows.

---

### Task 8: Explicit no-ops

These have **no implementation task**. Starting one in this plan is a spec violation.

Leave: akash, athina, comparia, hotbot, jatevo, meituan, neuro, notegpt, exa, sambanova, grok, novelai, runware, higgsfield, jewproxy, smolproxy, chutes, zai, vermal, openai, openrouter, crofai, google, mistral, cohere, anthropic, ionet, neuralwatt, telnyx, hyperfusion, partyrock, salesforce, sakana, venice, venicedev, e2b, cfplayground, k2think, albert/feather/arliai **cursors**, `flags.py` bools, Groq PlayAI TTS ids. siliconflow / pplx / together are deprecated, not leave-in-place.

A future spec may: grow `AttemptRuntime` with empty-200 key retry + live-usage mode (then Verboo wrap becomes legal); migrate Groq TTS to Orpheus; shrink `ProviderFlags`.

---

## Self-review

**Spec coverage:**
- Wave 0 gonka/dahl: no task
- Wave 1 matching-cursor auth: Task 1 (albert/feather/arliai excluded)
- Wave 2 Verboo extract: Task 2
- Wave 3 marketplace session: Task 5
- Wave 4 thin extract: Task 3
- Wave 5 partial + ArliAI import: Task 4
- Wave 6 fat leave: Task 8
- Wave 7 no buffered plugins: Task 6 (Microsoft only) + Task 8
- Wave 8/9 leave: Task 8
- Wave 10 flags stay: Task 8
- INDEX paths corrected: Task 7
- No unused profile fields: Global Constraints
- Groq TTS drift: Task 8 / out of scope

**Placeholder scan:** none. Task 2 includes a full `_EmptySession` fixture.

**Type consistency:** no new profile fields. Helper name is `yield_openai_sse_deltas` everywhere. No `AttemptRuntime` except gonka/dahl.
