# Universal Stream Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** One repo-level stream module owns nested-gen teardown, empty-visible overflow, and output-token metering so Gonka empty replies, Zai/JewProxy disconnect leaks, and per-chunk tiktoken are fixed once — not recopied in every provider.

**Architecture:** Nested fallbacks call `forward_agen` (teardown only). Empty-visible is a **same-broker retry with thinking off**, then the existing HTTP overflow target — not a hop to Verboo on the first blank. Tools keep today’s live-think buffer for Verboo/InferHub/SI. Routes stop the wasted per-chunk tiktoken when `max_tokens` is unset. No suffix token meter.

**Tech Stack:** Python 3.12, asyncio async generators, aiohttp, pytest-asyncio, existing `src/utils/stream_disconnect.py`, `src/providers/stream_usage.py`, `src/tools/providers/openai_compat`.

**Spec:** This document. It replaces `2026-08-15-stream-fallback-aclose.md` (aclose-only, per-provider, empty-visible out of scope). Evidence: 2026-08-15 08:24 Zai ignored `GeneratorExit`; live DeepSeek stress 9/12 `ok_content` + 3/12 `reasoning_only_empty_visible` with `would_fallback_n=0`.

## Global Constraints

- Module-level only. No new per-provider aclose/overflow/tokenize copies. New fallbacks call `forward_agen`. New overflow decisions call `should_overflow`.
- Do not overflow after visible content or tool_calls have been flushed to the subscriber (no duplicated answers).
- Reasoning-only is **not** success for overflow *decisions*. It is still valid live UX: do not hide `<think>` from Verboo/InferHub/SI or from Gonka clients that have thinking enabled.
- `should_overflow` runs only after the primary stream **completes** (or HTTP fails before any flush). Never mid-stream.
- Empty-visible: retry the **same** broker with `thinking=False` (or omit `chat_template_kwargs`). Only if that also fails, use the existing HTTP overflow map (Verboo/HyperFusion). Hopping to Verboo first with the same `max_tokens` + thinking reproduces the blank (live: think+256 → 3/3 empty; think+1024 → 2/3 empty; think off → 6/6 content).
- Hold `usage` frames in the same buffer as think. Drop them if we retry/overflow. Yielding Gonka usage then a Verboo body double-bills and double-`usage`s.
- Do not yield the dangling `</think>` closer before the retry/overflow decision.
- Do not change `_openai_payload_has_output` or `stream_buffered` flush for Verboo/InferHub/SI (live reasoning would stall until the first content token).
- Do not encode tiktoken suffixes. BPE is not prefix-stable (`encode("hello")+encode("world") != encode("hello world")`).
- `GeneratorExit` / `CancelledError` are never fallback failures.
- `close()` errors must not replace `GeneratorExit`.
- Do not reintroduce per-chunk `asyncio.sleep(0.01)`.
- Do not hunt every provider `__main__` demo loop.
- Do not commit unless the user asks.

## Why the previous plan was wrong

| Flaw | Effect |
| --- | --- |
| Empty-visible → Verboo listed as out of scope | The original Gonka ask was never planned |
| `aclose_async_gen` plus a 12-line try/finally pasted into Zai, JewProxy, Gonka, Dahl | Next provider will forget it; already four copies |
| `streamed_any = True` on reasoning | Overflow can never fire after `<think>` starts |
| Tools `track_empty_stream` excludes `gonka_family`; `_openai_payload_has_output` treats reasoning as useful | Tools path has the same empty-visible hole |
| `gonka_thinking_payload` ignores `thinking=` | `thinking=False` still forces think; small `max_tokens` → blank visible |
| `PROVIDER_STREAM_USAGE = {vermal, jewproxy}` + tokenize every other chunk in three routes | Sleep removal left the 100-chunk/s-class tiktoken tax on every non-allowlisted provider |
| Catalog 200k left untouched | Fine as a CF safety clamp (205k OK, 210k CF 502); not a stream bug — leave it |

## Safety audit (2026-08-15, before implementation)

Code read: `gonka/api.py` `send_message`, `dahl/api.py`, `chat.py` `process_chunks`, `messages.py` / `responses.py` send_message call sites, `openai_compat` `stream_buffered` / `payload.py` / `loop.py` / `fallbacks/gonka.py`+`dahl.py`, `helpers.__tokenize`, `openai_type.ChatData`.

### Confirmed root causes (do not “fix” the wrong layer)

1. **Empty visible is thinking-budget, not “Gonka is down.”** Live stress: `medium` + forced `chat_template_kwargs.thinking=true` + `max_tokens=256` → `finish=length`, ~1000 chars reasoning, 0 content, `would_fallback=False`. Same prompt with thinking off → 6/6 real paragraphs. Hopping to Verboo Flash with the same budget will likely blank again.
2. **`streamed_any` includes reasoning** (`gonka/api.py` ~404–416). HTTP overflow after a 200 think-only stream is impossible by design. That guard is *correct* for “don’t duplicate after flush”; it is the wrong predicate for “no visible answer.”
3. **Routes never pass `max_tokens` or `thinking` into Gonka/Dahl** (`chat.py` ~1069, `responses.py` ~829). `ChatData.max_tokens` defaults to `None`. `messages.py` passes `thinking` but `gonka_thinking_payload(bot)` ignores it. Production blanks are: user-set small `max_tokens` cut *in the route after flush*, or `thinking_enabled=false` (`REASONING_MODE_EXCLUDE`) stripping `<think>` so the client sees nothing.
4. **Gonka already yields `usage` chunks** (`stream_options.include_usage=True`) but chat/messages/responses **drop them** unless `provider in PROVIDER_STREAM_USAGE | PROVIDER_CACHE_USAGE`. Gonka is in neither. Adding a suffix meter is the wrong lever.

### Unsafe ideas in the first draft of this plan (killed)

| Draft idea | Why it is unsafe |
| --- | --- |
| Buffer all think, then overflow to Verboo | Hides live thinking for every Gonka request until first content. Long thinks (tens of seconds) add that delay to TTFT. If overflow uses same `max_tokens`+thinking, Verboo can also return empty. If Gonka `usage` was already yielded, route bills 256 think tokens *plus* Verboo. |
| `should_overflow(status=200, not visible)` without `stream_completed` | Mid-stream, before the first delta, this is true → immediate false overflow. |
| Yield `</think>` then check overflow (`gonka/api.py` ~418–420 today) | Closer is a yield. After that the subscriber has think tags; overflow duplicates. |
| Change `stream_buffered` to flush only on visible content | Changes Verboo/InferHub/SI tools: reasoning is held until content. `loop.py` ~529 treats `saw_useful_output` as “already flushed, do not retry.” Reasoning-only would start retrying those providers. |
| `OutputTokenMeter` encode-only-the-suffix | `cl100k_base` is not prefix-stable. Measured: `encode("hello")+encode("world")` ≠ `encode("hello world")`. Billing drift. |
| `track_empty_stream` += `gonka_family` *and* treat reasoning as not useful | Combined with the flush change, tools users lose live CoT. |

### Safe observations (keep)

- `forward_agen` is safe: aclose inner, then `abort_upstream`, swallow close errors, re-raise `GeneratorExit`/`CancelledError`. Double-`aclose` on an exhausted gen is a no-op.
- Gonka closes `self.session` in `finally` **before** today’s HTTP overflow block. Overflow already uses a fresh Verboo/HF client. Keep that order.
- `except Exception` in Gonka’s stream loop does **not** swallow `GeneratorExit`/`CancelledError` (both `BaseException`). Do not widen it.
- `tool-judge` sets `hyperfusion_fallback=False`. Empty-visible retry must honor that (retry thinking-off on the same client only; no forced Verboo).
- Disconnect while buffering: `iterate_until_disconnect` cancels `__anext__` and acloses the provider gen. Buffer is discarded. No leak if `forward_agen`/`aclose` runs.
- Heartbeat is 10s and independent of the first provider yield. Buffering a 2s think does not trip the 300s wait. A 60s think-only *plus* Verboo hop would, which is another reason not to hop first.
- Per-chunk tiktoken in `chat.py` ~1153–1157 runs in the `else` of `if max_tokens` — i.e. **even when `max_tokens` is None**. `final_output_tokens` tokenizes again at end (~1206). The per-chunk call is unused for cutoff. Deleting that else-branch is the safe, universal win.

### Required edge-case tests (any implementation must include)

- Reasoning-only + `stream_completed=True` + `flushed=False` → same-broker thinking-off retry, not Verboo.
- After thinking-off retry still empty → HTTP overflow map (0731 → Verboo) via `forward_agen`.
- Visible content flushed → no retry, no overflow.
- `usage` chunk during think-only → not forwarded; dropped on retry.
- `hyperfusion_fallback=False` → thinking-off retry only; no Verboo.
- Disconnect during think-only (no yield yet) → inner gen aclosed, no ignored `GeneratorExit`.
- `thinking_enabled=false` / `REASONING_MODE_EXCLUDE`: provider may still emit `<think>`; route strips it. Retry thinking-off is what actually produces visible text.
- Do not pass a new `max_tokens` into a retry that is *smaller* than the think already consumed.
- Dahl 402 remint stays a remint, not an empty-visible overflow.

---

## File map

| File | Responsibility |
| --- | --- |
| `src/utils/stream_disconnect.py` | `aclose_async_gen`, `abort_upstream`, **new** `forward_agen` |
| `src/providers/stream_progress.py` | **New.** `StreamProgress`, `note_delta`, `should_overflow` |
| `src/providers/stream_usage.py` | Existing `usage_chunk` only — **no** suffix token meter |
| `src/tools/providers/openai_compat/payload.py` | Split visible vs reasoning for buffer flush |
| `src/tools/providers/openai_compat/providers/flags.py` | `track_empty_stream` includes `gonka_family` |
| `src/tools/providers/openai_compat/providers/fallbacks/gonka.py` | Overflow on empty-visible `last_error` too |
| `src/providers/prod/gonka/api.py` | Use `forward_agen` + `StreamProgress`; honor `thinking=` |
| `src/providers/prod/dahl/api.py` | Same (already shares `resolve_gonka_overflow`) |
| `src/providers/prod/zai/api.py` | Collapse local try/finally to `forward_agen` |
| `src/providers/prod/jewproxy/api.py` | Collapse local try/finally to `forward_agen` |
| `src/routes/normal/chat.py` | Delete unused per-chunk tokenize when `max_tokens` is unset |
| `src/routes/normal/messages.py` | Same |
| `src/routes/normal/responses.py` | Same |
| Tests | Module tests first; provider tests only prove they call the module |

Do **not** add `src/providers/prod/*/tests/test_*_aclose_offline.py` for the next provider. One `forward_agen` test covers teardown.

---

### Task 1: `forward_agen` — the only nested-yield API

**Files:**
- Modify: `src/utils/stream_disconnect.py`
- Test: `src/utils/tests/test_stream_disconnect_offline.py`

**Interfaces:**
- Consumes: `aclose_async_gen`, `abort_upstream`
- Produces: `async def forward_agen(agen: Any, client: Any = None) -> AsyncIterator[Any]`

- [ ] **Step 1: Write the failing tests**

```python
@pytest.mark.asyncio
async def test_forward_agen_yields_then_aclose_and_close():
    from src.utils.stream_disconnect import forward_agen

    closed = {"inner": 0, "client": 0}

    async def inner():
        try:
            yield {"response": "hi"}
            await asyncio.Event().wait()
        finally:
            closed["inner"] += 1

    class Client:
        async def close(self):
            closed["client"] += 1
            raise RuntimeError("close failed")

    agen = forward_agen(inner(), Client())
    it = agen.__aiter__()
    assert (await it.__anext__())["response"] == "hi"
    await it.aclose()
    assert closed == {"inner": 1, "client": 1}


@pytest.mark.asyncio
async def test_forward_agen_reraises_cancelled():
    from src.utils.stream_disconnect import forward_agen

    async def inner():
        yield 1
        raise asyncio.CancelledError()

    with pytest.raises(asyncio.CancelledError):
        async for _ in forward_agen(inner()):
            pass
```

- [ ] **Step 2: Run tests — expect FAIL** (`forward_agen` not defined)

Run: `python -m pytest src/utils/tests/test_stream_disconnect_offline.py::test_forward_agen_yields_then_aclose_and_close -q`

- [ ] **Step 3: Implement**

```python
async def forward_agen(agen: Any, client: Any = None) -> AsyncIterator[Any]:
    """Yield from a nested async gen; always aclose it, then close ``client``.

    Use this for every ``other_client.send_message(...)`` / overflow gen.
    ``client.close()`` errors are swallowed so they cannot replace GeneratorExit.
    """
    try:
        async for chunk in agen:
            yield chunk
    except (GeneratorExit, asyncio.CancelledError):
        raise
    finally:
        await aclose_async_gen(agen)
        await abort_upstream(client)
```

- [ ] **Step 4: Collapse the four existing copies** to one line each.

Zai / JewProxy / Gonka / Dahl today:

```python
            try:
                async for chunk in agen:
                    yield chunk
            except (GeneratorExit, asyncio.CancelledError):
                raise
            finally:
                await aclose_async_gen(agen)
                await abort_upstream(client)
```

Replace with:

```python
            async for chunk in forward_agen(agen, client):
                yield chunk
```

JewProxy `_smolproxy_fallback_chunks` already has an extra `smol.close()` finally — pass `smol` as `client` and delete the extra finally.

- [ ] **Step 5: Run**

Run: `python -m pytest src/utils/tests/test_stream_disconnect_offline.py src/providers/prod/zai/tests/test_fallback_aclose_offline.py src/providers/prod/jewproxy/tests/test_fallback_aclose_offline.py src/providers/prod/gonka/tests/test_overflow_aclose_offline.py -q`

Expected: PASS. Existing provider aclose tests stay as regression that they still call the shared path.

---

### Task 2: `StreamProgress` + `should_overflow` (fixes Gonka empty-visible)

**Files:**
- Create: `src/providers/stream_progress.py`
- Create: `src/providers/tests/test_stream_progress_offline.py`

**Interfaces:**
- Produces:

```python
@dataclass
class StreamProgress:
    visible: bool = False      # assistant content or tool_calls
    reasoning: bool = False    # reasoning / <think> only
    flushed: bool = False      # already yielded visible/tools to subscriber

    @property
    def emitted_any(self) -> bool:
        return self.visible or self.reasoning

def note_delta(
    progress: StreamProgress,
    *,
    content: str = "",
    reasoning: str = "",
    tool_calls: Any = None,
) -> None: ...

def should_overflow(
    progress: StreamProgress,
    *,
    status: int = 200,
    body: str = "",
    http_predicate: Optional[Callable[[int, str], bool]] = None,
    allow_empty_visible: bool = True,
) -> bool: ...
```

Rules (lock these; do not reinterpret per provider):

1. If `progress.flushed` → never retry/overflow.
2. `http_predicate(status, body)` and `not progress.flushed` → HTTP overflow (existing map).
3. `allow_empty_visible` and `stream_completed` and `status==200` and `not progress.visible` → **same-broker thinking-off retry**, not Verboo. Only if that retry also ends with `not visible` does HTTP overflow run.
4. `should_overflow` / retry MUST receive `stream_completed=True`. Default `False` so a missing flag cannot mid-stream hop.

- [ ] **Step 1: Write the failing tests**

```python
from src.providers.stream_progress import StreamProgress, note_delta, should_overflow

def test_reasoning_only_overflows_if_not_flushed():
    p = StreamProgress()
    note_delta(p, reasoning="think")
    assert p.visible is False
    assert p.reasoning is True
    assert should_overflow(p, status=200, stream_completed=True) is True
    assert should_overflow(p, status=200, stream_completed=False) is False

def test_visible_content_does_not_overflow():
    p = StreamProgress()
    note_delta(p, content="hello")
    p.flushed = True
    assert should_overflow(p, status=200) is False

def test_http_429_overflows_if_not_flushed():
    p = StreamProgress()
    assert should_overflow(
        p, status=429, body="rate limit",
        http_predicate=lambda s, b: s == 429,
    ) is True

def test_flushed_blocks_http_overflow():
    p = StreamProgress()
    note_delta(p, content="hi")
    p.flushed = True
    assert should_overflow(
        p, status=429, http_predicate=lambda s, b: s == 429,
    ) is False

def test_tool_calls_are_visible():
    p = StreamProgress()
    note_delta(p, tool_calls=[{"id": "1"}])
    assert p.visible is True
    assert should_overflow(p, status=200) is False
```

- [ ] **Step 2: Run — expect FAIL** (module missing)

Run: `python -m pytest src/providers/tests/test_stream_progress_offline.py -q`

- [ ] **Step 3: Implement `src/providers/stream_progress.py`**

```python
from __future__ import annotations

from dataclasses import dataclass
from typing import Any, Callable, Optional


@dataclass
class StreamProgress:
    visible: bool = False
    reasoning: bool = False
    flushed: bool = False

    @property
    def emitted_any(self) -> bool:
        return self.visible or self.reasoning


def note_delta(
    progress: StreamProgress,
    *,
    content: str = "",
    reasoning: str = "",
    tool_calls: Any = None,
) -> None:
    if reasoning:
        progress.reasoning = True
    if content:
        progress.visible = True
    if tool_calls:
        progress.visible = True


def should_overflow(
    progress: StreamProgress,
    *,
    status: int = 200,
    body: str = "",
    http_predicate: Optional[Callable[[int, str], bool]] = None,
    allow_empty_visible: bool = True,
    stream_completed: bool = False,
) -> bool:
    if progress.flushed:
        return False
    if http_predicate is not None and http_predicate(status, body):
        return True
    if (
        allow_empty_visible
        and stream_completed
        and status == 200
        and not progress.visible
    ):
        return True
    return False
```

`should_overflow(...)` on empty-visible means “retry thinking-off or overflow”; the caller distinguishes HTTP vs empty via `http_predicate` match first.

- [ ] **Step 4: Run tests — expect PASS**

---

### Task 3: Wire overflow into Gonka-family chat + tools (no second copy in Dahl)

**Files:**
- Modify: `src/providers/prod/gonka/api.py` (`send_message` loop + `gonka_thinking_payload`)
- Modify: `src/providers/prod/dahl/api.py` (same loop shape; keep using `should_fallback_from_dahl` as the **http** predicate only)
- Modify: `src/tools/providers/openai_compat/payload.py`
- Modify: `src/tools/providers/openai_compat/providers/flags.py`
- Modify: `src/tools/providers/openai_compat/providers/fallbacks/gonka.py`
- Modify: `src/tools/providers/openai_compat/loop.py` (empty-stream `last_error` already exists; gonka fallback must accept it)
- Test: `src/providers/prod/gonka/tests/test_overflow_offline.py` (extend)
- Test: `src/tools/providers/openai_compat` existing flag tests if any

**Interfaces:**
- Consumes: `StreamProgress`, `note_delta`, `should_overflow`, `forward_agen`
- Produces: chat + tools overflow on reasoning-only / empty 200

Critical chat-path change in `GonkaClient.send_message` / `DahlClient.send_message`:

Keep **live** think yields (do not buffer the happy path). Track `StreamProgress` on the raw delta. After the SSE loop ends, if `not progress.visible` and nothing was flushed except think/usage:

1. Drop buffered `usage` (do not forward the think-only usage frame).
2. Do **not** yield the dangling `</think>`.
3. If `thinking` was not already False: one same-client retry with `gonka_thinking_payload(bot, thinking=False)` (new POST; session still open — move `session.close()` to after retries, or open a new session for the retry).
4. If still `not visible` and `hyperfusion_fallback`: `forward_agen(_hyperfusion_fallback(..., thinking=False))`.
5. If still nothing: yield the original think buffer (today’s behavior) so we do not swallow evidence.

Do **not** hold think until first content on the success path. That would stall every Gonka CoT stream.

`gonka_thinking_payload` must honor the caller's thinking flag:

```python
def gonka_thinking_payload(bot: str, thinking: Optional[bool] = None) -> dict:
    if (bot or "").strip() not in _DEEPSEEK_THINKING_DEFAULT:
        return {}
    if thinking is False:
        return {}
    return {"chat_template_kwargs": {"thinking": True}}
```

Pass `thinking` from `send_message`. Chat and responses must start passing `thinking=thinking_enabled` and `max_tokens=max_tokens` into `send_message` (messages already passes `thinking`).

Tools path — **do not** change `stream_buffered` flush or `_openai_payload_has_output`. Only:

1. `flags.track_empty_stream` stays Verboo/IH/SI (live CoT). Add a separate `retry_empty_visible: bool` on `gonka_family` that, after passthrough, if no assistant `content`/`tool_calls` accumulated, sets `last_error = {status:200, text:"empty stream"}` and continues.
2. `gonka_hyperfusion_fallback` / `dahl_hyperfusion_fallback`: `empty stream` first triggers a same-provider tools retry with thinking off (via `request_extras={}` / no `gonka_thinking_payload`). Second failure uses the existing Verboo/HF map. Share `is_empty_stream_error(status, text)` in `stream_progress.py`. Do not fork the `"empty stream"` string.

- [ ] **Step 1: Tests first**

Extend `test_overflow_offline.py`:

```python
def test_should_overflow_reasoning_only_uses_module():
    from src.providers.stream_progress import StreamProgress, note_delta, should_overflow
    p = StreamProgress()
    note_delta(p, reasoning="x")
    assert should_overflow(
        p, status=200, stream_completed=True,
        http_predicate=should_fallback_to_hyperfusion,
    )

def test_thinking_payload_honors_false():
    assert gonka_thinking_payload(DEEPSEEK_0731, thinking=False) == {}
    assert gonka_thinking_payload(DEEPSEEK_0731, thinking=True) == {
        "chat_template_kwargs": {"thinking": True}
    }
```

Flag test:

```python
def test_gonka_family_retries_empty_visible_not_buffered():
    from src.tools.providers.openai_compat.providers.flags import flags_for
    assert flags_for("gonka").retry_empty_visible
    assert flags_for("dahl").retry_empty_visible
    assert not flags_for("gonka").track_empty_stream
    assert flags_for("verboo").track_empty_stream
```

- [ ] **Step 2: Run — expect FAIL** on thinking=False and flag
- [ ] **Step 3: Implement the wiring above**
- [ ] **Step 4: Run overflow + flags + gonka/dahl offline tests — expect PASS**

---

### Task 4: Stop wasted per-chunk tiktoken (all providers)

**Files:**
- Modify: `src/routes/normal/chat.py` (`process_chunks` ~1141–1157)
- Modify: `src/routes/normal/messages.py` (same pattern)
- Modify: `src/routes/normal/responses.py` (same pattern)

**Do not** add `OutputTokenMeter` or suffix encoding.

In `chat.py` today, when `max_tokens` is unset (the default), the `else` branch still `__tokenize`s the full accum every chunk, then `final_output_tokens` tokenizes again at end. Delete the else-branch tokenize. Only tokenize during the stream when `max_tokens` is set (cutoff). Always tokenize once at end for billing (or use provider usage when allowlisted).

```python
                if max_tokens and provider not in _PROVIDER_STREAM_USAGE:
                    chunk_token = await helpers.__tokenize(last_text + reasoning_text)
                    if chunk_token >= max_tokens:
                        finish_reason = "length"
                        stream_completed_normally = False
                        break
                elif output_tokens_from_provider is not None:
                    chunk_token = output_tokens_from_provider
```

- [ ] **Step 1: Grep the three routes for `__tokenize(last_text` in the per-chunk loop; write a tiny offline test that a stub process loop with `max_tokens=None` calls `__tokenize` only at finalize**
- [ ] **Step 2: Apply the else-branch deletion in chat / messages / responses**
- [ ] **Step 3: Run existing route/offline tests**

Do **not** add Gonka to `PROVIDER_STREAM_USAGE` until a live usage-accuracy pass exists.

---

### Task 5: Live confirm (same script, new expectations)

**Files:**
- Modify: `src/providers/prod/gonka/live_deepseek_empty_stress.py` — report `would_fallback` using `should_overflow` (reasoning-only + not flushed), not `should_fallback_to_hyperfusion and not streamed_any`.

- [ ] **Step 1: Update `_would_fallback` to use `should_overflow`**
- [ ] **Step 2: Run**

Run: `python -m src.providers.prod.gonka.tests.live_deepseek_empty_stress --concurrency 8 --rounds 3`

Expected:

- Exit 0.
- Raw-HTTP stress will still show `reasoning_only_empty_visible` (it bypasses `GonkaClient`). That is expected.
- A follow-up live call through `GonkaClient.send_message(..., max_tokens=256)` on the medium prompt must retry thinking-off and return visible content (log line for the retry). Verboo only if the retry is also empty.
- Per-chunk tokenize absence: `tokenize` pace in the raw script is unchanged; route-level tok/s improves only on the API path when `max_tokens` is unset.
- No `Exception ignored in: <async_generator`.

---

## Already landed (keep; then collapse into Task 1)

- Hot-path `sleep(0.01)` removed; `X-Accel-Buffering: no` on SSE.
- `aclose_async_gen` + Zai fallback-after-POST + JewProxy close-wrap.
- Per-provider aclose copies — **delete in Task 1** in favor of `forward_agen`.

## Out of scope (still)

- Raising catalog `tokens: 200000` on `deepseek-v4-flash-0731:dev`. Live: 205k OK, 210k Cloudflare 502. 200k is a product clamp, not a stream bug.
- Adding Gonka to `PROVIDER_STREAM_USAGE` without a live usage-accuracy pass.
- Rewriting every provider `__main__` demo `async for`.

## Self-review

- Spec coverage: ignored GE → Task 1. Gonka empty-visible chat + tools → Tasks 2–3. Thinking flag → Task 3. Tiktoken tax → Task 4. Live proof → Task 5.
- No per-provider teardown/overflow/tokenize logic after Task 1–4.
- `should_overflow` / `forward_agen` / `retry_empty_visible` names are stable across later tasks.
- Placeholder scan: none.
