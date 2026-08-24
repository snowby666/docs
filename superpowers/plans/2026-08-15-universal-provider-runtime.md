# Universal Provider Runtime Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** One `ProviderProfile` + `AttemptRuntime` owns empty-visible retry, HTTP overflow, terminal-frame hold, and teardown for every chat and tools provider — so the next broker is a profile row, not a new `flags.dahl`.

**Architecture:** Move existing helpers into `src/providers/runtime/`. Chat `send_message` and `openai_compat` loop both call the same runtime. `flags_for` becomes a shim over `profile_for`. Auth caches become injectable stores. INDEX is regenerated, not hand-edited.

**Tech Stack:** Python 3.12, asyncio, aiohttp, pytest-asyncio, existing `stream_progress` / `forward_agen` / `generate_indexes.py`.

**Spec:** `docs/superpowers/specs/2026-08-15-universal-provider-runtime-design.md`

## Global Constraints

- Do not invent overflow destinations or new product behavior.
- Do not change `stream_buffered` flush for Verboo / InferHub / SI.
- Do not add Gonka to `PROVIDER_STREAM_USAGE` without a live usage-accuracy pass.
- Do not revive dead `sse_chat.py` / `key_ring.py` names at the old paths.
- Do not hand-edit `INDEX.md`; run `generate_indexes.py`.
- `GeneratorExit` / `CancelledError` are never fallback failures.
- Dahl 402 is remint on every POST, never overflow.
- Chat and tools must share the same overflow/empty-visible decisions.
- No client singleton. Shared caches are explicit injected stores.
- Do not commit unless the user asks.

## File map

| File | Responsibility |
| --- | --- |
| `src/providers/runtime/progress.py` | Move of `stream_progress.py` (re-export old path one release) |
| `src/providers/runtime/profile.py` | `ProviderProfile`, `profile_for` |
| `src/providers/runtime/overflow.py` | HTTP predicates + `resolve_gonka_overflow` (moved from gonka/api.py) |
| `src/providers/runtime/sse.py` | `yield_openai_sse_deltas`, `ThinkWrap` |
| `src/providers/runtime/attempt.py` | `AttemptRuntime` (hold terminal, thinking-off, overflow) |
| `src/providers/runtime/auth/keys.py` | `RoundRobinKeys` instance |
| `src/providers/runtime/auth/token_store.py` | `TokenStore` protocol + `ProcessTokenCache` |
| `src/tools/providers/openai_compat/loop.py` | Use profile + hold_terminal |
| `src/tools/providers/openai_compat/streaming/default.py` | Hold `[DONE]`/usage when profile says so |
| `src/providers/INDEX.md` / `src/tools/providers/INDEX.md` | Regenerated only |

---

### Task 0: Close the tools `[DONE]` hole (must land before the move)

**Files:**
- Modify: `src/tools/providers/openai_compat/streaming/default.py`
- Modify: `src/tools/providers/openai_compat/streaming/result.py`
- Modify: `src/tools/providers/openai_compat/loop.py`
- Modify: `src/providers/prod/dahl/api.py` (402 on thinking-off remint)
- Modify: `src/providers/prod/gonka/api.py` (`thinking: Optional[bool] = None`)
- Test: `src/tools/providers/openai_compat/tests/test_hold_terminal_offline.py` (new)
- Test: `src/providers/prod/dahl/tests/test_empty_visible_offline.py` (new)

**Interfaces:**
- Consumes: `StreamSession.progress`, `should_overflow`, `EMPTY_STREAM_TEXT`
- Produces: `StreamSession.held_frames: list[bytes]`; passthrough yields content/reasoning/tool_calls live; holds `usage` frames and `data: [DONE]` until `loop` accepts the attempt

- [x] **Step 1: Write the failing tools test**

```python
@pytest.mark.asyncio
async def test_passthrough_holds_done_and_usage_until_release():
    from src.tools.providers.openai_compat.streaming.default import stream_passthrough
    from src.tools.providers.openai_compat.streaming.result import StreamSession

    class _Resp:
        async def __aiter__(self):
            yield 'data: {"choices":[{"delta":{"reasoning":"x"}}]}'
            yield 'data: {"usage":{"prompt_tokens":1,"completion_tokens":2}}'
            yield "data: [DONE]"

    session = StreamSession()
    session.hold_terminal = True
    out = []
    async for chunk in stream_passthrough(
        _Resp(), session=session, tools=None, display_model="m",
        repair_names=False, logging_data={},
    ):
        out.append(chunk)
    assert not any(b"[DONE]" in c for c in out)
    assert session.held_frames and any(b"[DONE]" in f for f in session.held_frames)
```

- [x] **Step 2: Run — expect FAIL** (`hold_terminal` missing)

Run: `python -m pytest src/tools/providers/openai_compat/tests/test_hold_terminal_offline.py -q`

- [x] **Step 3: Implement hold + loop release**

In `stream_passthrough`, if `session.hold_terminal`:
- live-yield deltas that `note_delta` marks as reasoning/content/tool_calls
- append usage and `[DONE]` to `session.held_frames` instead of yielding

In `loop.py` after a successful (non-overflow) stream:

```python
for frame in stream.held_frames:
    yield frame
```

On thinking-off `continue` or overflow `break`: drop `held_frames` (do not yield).

Dahl thinking-off 402: remint + retry the thinking-off POST (same as primary). Gonka/Dahl `thinking` default `None`.

- [x] **Step 4: Run hold + existing gonka/dahl/aclose offline tests — expect PASS**

Run: `python -m pytest src/tools/providers/openai_compat/tests/test_hold_terminal_offline.py src/providers/prod/gonka/tests src/providers/prod/dahl/tests src/utils/tests/test_stream_disconnect_offline.py -q`

- [x] **Step 5: Non-stream `retry_empty_visible`**

After JSON parse in `loop.py`, if `profile.retry_empty_visible` and no assistant `content`/`tool_calls`: same thinking-off once, then `last_error = EMPTY_STREAM_TEXT`. Do not treat reasoning-only as success.

---

### Task 1: `ProviderProfile` + `profile_for` (replace hardcoded flags)

**Files:**
- Create: `src/providers/runtime/__init__.py`
- Create: `src/providers/runtime/profile.py`
- Create: `src/providers/tests/test_profile_offline.py`
- Modify: `src/tools/providers/openai_compat/providers/flags.py` (shim)

**Interfaces:**
- Produces:

```python
@dataclass(frozen=True)
class ProviderProfile:
    name: str
    tool_type: str | None
    stream_mode: str  # passthrough | buffered | kilo
    retry_empty_visible: bool = False
    hold_terminal: bool = False
    seed_bust: bool = False
    thinking_default: bool | None = None

def profile_for(provider: str) -> ProviderProfile: ...
```

Seed data (only what exists today — do not invent flags):

| name | stream_mode | retry_empty_visible | hold_terminal | seed_bust |
| --- | --- | --- | --- | --- |
| gonka, dahl | passthrough | True | True | True |
| verboo, inferhub, surplusintelligence | buffered | False | False | False |
| kilo | kilo | False | False | False |
| others in openai_compat | passthrough | False | False | False |

- [x] **Step 1: Failing tests**

```python
from src.providers.runtime.profile import profile_for

def test_gonka_family_profiles():
    g = profile_for("gonka")
    d = profile_for("dahl")
    assert g.retry_empty_visible and d.retry_empty_visible
    assert g.hold_terminal and d.hold_terminal
    assert g.stream_mode == "passthrough"
    assert not profile_for("verboo").retry_empty_visible
    assert profile_for("verboo").stream_mode == "buffered"
    assert profile_for("kilo").stream_mode == "kilo"

def test_flags_shim_matches_profile():
    from src.tools.providers.openai_compat.providers.flags import flags_for
    assert flags_for("gonka").retry_empty_visible == profile_for("gonka").retry_empty_visible
    assert flags_for("verboo").track_empty_stream
    assert not flags_for("gonka").track_empty_stream
```

- [x] **Step 2: Run — expect FAIL**
- [x] **Step 3: Implement `profile_for` + make `flags_for` read profiles for the four computed properties**
- [x] **Step 4: Run — expect PASS**

---

### Task 2: Move overflow + SSE helpers into `runtime/` (no behavior change)

**Files:**
- Create: `src/providers/runtime/progress.py` (move `stream_progress.py`; old module re-exports)
- Create: `src/providers/runtime/overflow.py` (`should_fallback_to_hyperfusion`, `should_fallback_from_dahl`, `resolve_gonka_overflow`, `gonka_thinking_payload`)
- Create: `src/providers/runtime/sse.py` (`ThinkWrap`, `yield_openai_sse_deltas`)
- Modify: `src/providers/prod/gonka/api.py`, `src/providers/prod/dahl/api.py` to import from runtime
- Modify: tools fallbacks / `payload_build.py` to import from runtime
- Test: existing `test_overflow_offline.py`, `test_stream_progress_offline.py` (update imports or keep re-exports)

**Interfaces:**
- Consumes: current function bodies (copy, then delete originals)
- Produces: same signatures; `gonka/api.py` re-exports for one cycle so `from src.providers.prod.gonka.api import resolve_gonka_overflow` still works

- [x] **Step 1: Move files; keep re-exports in old paths**
- [x] **Step 2: Run**

Run: `python -m pytest src/providers/prod/gonka/tests src/providers/prod/dahl/tests src/providers/tests/test_stream_progress_offline.py -q`

Expected: PASS, no logic change.

---

### Task 3: `AttemptRuntime` used by Gonka/Dahl chat (delete the duplicated send_message ladder)

**Files:**
- Create: `src/providers/runtime/attempt.py`
- Modify: `src/providers/prod/gonka/api.py` `send_message`
- Modify: `src/providers/prod/dahl/api.py` `send_message`
- Test: `src/providers/prod/gonka/tests/test_empty_visible_offline.py` (must still pass)

**Interfaces:**
- Produces:

```python
class AttemptRuntime:
    def __init__(self, profile: ProviderProfile, session, auth, *, log_label: str): ...
    async def chat_chunks(self, *, url, payload, headers, message, overflow_kwargs) -> AsyncIterator[dict]: ...
```

`chat_chunks` contains today’s sequence: primary SSE → `next_empty_visible_action` → thinking-off → `_hyperfusion_fallback` via `forward_agen`. Dahl passes a remint hook: `async def on_quota(status, body) -> bool`.

- [x] **Step 1: Extract without changing tests**
- [x] **Step 2: Run gonka/dahl empty-visible + overflow + aclose tests — expect PASS**

---

### Task 4: Tools loop calls the same runtime decisions

**Files:**
- Modify: `src/tools/providers/openai_compat/loop.py`
- Modify: `src/tools/providers/openai_compat/providers/fallbacks/gonka_family.py` to import overflow from `runtime/overflow.py`
- Delete after move: duplicate predicate imports from `prod/gonka/api.py` / `prod/dahl/api.py` in fallbacks (keep thin wrappers if `loop` still calls them by name)

**Interfaces:**
- Consumes: `profile_for(provider).retry_empty_visible`, `.hold_terminal`, `.stream_mode`
- Replaces `flags.track_empty_stream` with `profile.stream_mode == "buffered"`
- Replaces `flags.kilo` stream pick with `profile.stream_mode == "kilo"`

- [x] **Step 1: Offline test that `profile_for` drives stream function selection**
- [x] **Step 2: Switch loop to profiles; keep `flags.*` only where marketplace/inferhub/si/deepinfra still need custom attempt builders**
- [x] **Step 3: Run hold-terminal + overflow + flags shim tests — expect PASS**

Do **not** rewrite InferHub/SI ladders in this task. Those stay as attempt builders keyed by profile name until a later plan.

---

### Task 5: Auth stores — kill hidden module globals on Gonka/Dahl only

**Files:**
- Create: `src/providers/runtime/auth/keys.py`
- Create: `src/providers/runtime/auth/token_store.py`
- Modify: `src/providers/prod/gonka/api.py` (`_CURRENT_INDEX` → `RoundRobinKeys(API_KEYS)` constructed in `GonkaClient.__init__` or a module-level **named** `GONKA_KEYS = RoundRobinKeys(...)` passed into the client)
- Modify: `src/providers/prod/dahl/api.py` (`_token` → `ProcessTokenCache` instance `DAHL_TOKENS` injected; `current_dahl_keys()` reads the store)

**Interfaces:**

```python
class RoundRobinKeys:
    def __init__(self, keys: list[str]): ...
    def next(self) -> str: ...

class TokenStore(Protocol):
    def get(self) -> str | None: ...
    def set(self, token: str, available: int | None = None) -> None: ...
    def invalidate(self, token: str | None = None) -> None: ...
```

A process cache **instance** is allowed (mint must be reused across requests). A hidden `_token = None` at module bottom is not. Tests inject a fake store.

- [x] **Step 1: Tests with a fake `TokenStore` proving remint does not touch a second global**
- [x] **Step 2: Wire Gonka/Dahl only. Do not “fix” PromptLayer/Telnyx globals in this plan.**
- [x] **Step 3: Run dahl overflow + mint-related offline tests**

---

### Task 6: INDEX regenerate (Dahl already listed)

**Files:**
- Modify: `src/providers/tests/generate_indexes.py` `NOTES` only if profile text changed
- Regenerate: `src/providers/INDEX.md`, `src/tools/providers/INDEX.md`

Dahl is already in both files. This task is **regenerate after Tasks 1–5**, then diff:

- `dahl` still in registration map + tools handler map
- tools specials bullet still says mint + overflow
- cache column may stay `untested` until Task 7

- [x] **Step 1:** `python -m src.providers.tests.generate_indexes`
- [x] **Step 2:** `git diff -- src/providers/INDEX.md src/tools/providers/INDEX.md` — no invented rows; only inventory/note drift

---

### Task 7: Full-suite live (concurrency 15)

**Do not invent a new harness.** Run the three that already cover registered providers:

- [x] `python -m src.providers.tests.live_cache_matrix --concurrency 15`
- [x] `python -m src.providers.tests.live_usage_surface --concurrency 15`
- [x] `python -m src.tools.providers.openai_compat.tests.live_tools_matrix --concurrency 15`

Plus the existing Gonka client probe (proves thinking-off still works after the move):

- [x] `python -m src.providers.prod.gonka.tests.live_deepseek_empty_stress --concurrency 8 --rounds 1`

**Pass bar:**
- [x] Tools matrix: gonka + dahl stream scenarios must not end on a think-only `[DONE]` when the client path would retry (compare `client_probe.ok` / logs `empty visible; retrying`).
- [x] No `Exception ignored in: <async_generator`.
- [x] Cache/usage matrices: no new provider-wide fail vs the 2026-08-11 baseline except documented upstream outages.

---

## Self-review

- Spec coverage: tools `[DONE]` → Task 0. Profiles → Task 1. Move helpers → Task 2. Chat runtime → Task 3. Tools loop → Task 4. Auth injection → Task 5. INDEX → Task 6. Live 15 → Task 7.
- Placeholder scan: none. Live commands are the existing modules.
- Type consistency: `ProviderProfile.retry_empty_visible` / `hold_terminal` / `stream_mode` used in Tasks 1, 3, 4.
- Out of scope (still): PromptLayer/Telnyx global cleanup, `PROVIDER_STREAM_USAGE` + Gonka, catalog 200k, InferHub ladder rewrite.
