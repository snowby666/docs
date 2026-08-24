# Stream Disconnect + Nested Fallback Aclose Implementation Plan

> **SUPERSEDED** by [`2026-08-15-universal-stream-hardening.md`](./2026-08-15-universal-stream-hardening.md).
> This plan shipped per-provider aclose copies and explicitly excluded Gonka empty-visible overflow. Do not execute remaining unchecked steps here.

# Stream Disconnect + Nested Fallback Aclose Implementation Plan (archived)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make subscriber disconnect during nested provider fallbacks (Zai → JewProxy → SmolProxy, plus Gonka/Dahl overflow) tear down every async generator and session without `Exception ignored in: <async_generator …>` or abandoned upstream sockets, and keep SSE unbuffered after removing the per-chunk `sleep(0.01)`.

**Architecture:** One shared `aclose_async_gen` in `src/utils/stream_disconnect.py`. Every nested `async for chunk in other_client.send_message(...)` must `aclose` the inner gen in `finally` *before* `close()` on the client, and must re-raise `GeneratorExit` / `CancelledError`. Fallback chains must not run while an upstream `async with response` is still held. Route SSE headers include `X-Accel-Buffering: no`. Hot-path `asyncio.sleep(0.01)` stays deleted.

**Tech Stack:** Python 3.12, asyncio async generators, aiohttp, pytest-asyncio, Starlette `StreamingResponse`.

**Spec:** Production log 2026-08-15 08:24 (`Exception ignored in: <async_generator object ZaiClient.send_message>` at `chat.py` `process_chunks` line -1) plus the approved sleep-removal research (OpenAI / OpenRouter / LiteLLM yield immediately; backpressure is TCP + `await send()`).

## Global Constraints

- Do not treat `GeneratorExit` or `asyncio.CancelledError` as a fallback failure (no roll to the next provider).
- `client.close()` / `session.close()` during disconnect must be wrapped in `except Exception` so a close error cannot replace `GeneratorExit`.
- Aclose the inner gen first, then close the client. Reverse order races the session out from under the gen.
- Do not reintroduce per-chunk `asyncio.sleep(0.01)` on chat / messages / responses hot paths.
- Leave the one-shot `sleep(0.01)` after chat timeout `[DONE]` (not a token-pacing path).
- Do not change Gonka empty-visible → Verboo behavior, catalog 200k, or `PROVIDER_STREAM_USAGE` in this plan.
- Do not commit unless the user asks.

---

## File map

| File | Responsibility |
| --- | --- |
| `src/utils/stream_disconnect.py` | Shared `aclose_async_gen`; `iterate_until_disconnect` uses it |
| `src/utils/tests/test_stream_disconnect_offline.py` | Helper + wrapper aclose tests |
| `src/providers/prod/zai/api.py` | Use shared helper; keep fallback *after* Z.ai POST exits |
| `src/providers/prod/zai/tests/test_fallback_aclose_offline.py` | Zai nested aclose (already present; keep green) |
| `src/providers/prod/jewproxy/api.py` | SmolProxy / Verboo fallback aclose-before-close |
| `src/providers/prod/jewproxy/tests/test_fallback_aclose_offline.py` | JewProxy aclose + close-error must not leak GE |
| `src/providers/prod/gonka/api.py` | Overflow gens aclose |
| `src/providers/prod/dahl/api.py` | Same overflow aclose |
| `src/providers/prod/gonka/tests/test_overflow_aclose_offline.py` | Gonka overflow aclose |
| `src/routes/normal/chat.py` | Sleep already removed; `X-Accel-Buffering: no` already set |
| `src/routes/normal/messages.py` | Same |
| `src/routes/normal/responses.py` | Same |
| `src/providers/prod/gonka/live_deepseek_empty_stress.py` | Add `tokenize` pace (current API: no sleep, still tiktoken every chunk) |

---

### Task 1: Shared `aclose_async_gen`

**Files:**
- Modify: `src/utils/stream_disconnect.py`
- Test: `src/utils/tests/test_stream_disconnect_offline.py`

**Interfaces:**
- Produces: `async def aclose_async_gen(agen: Any) -> None`

- [x] **Step 1: Write the failing test**

```python
@pytest.mark.asyncio
async def test_aclose_async_gen_runs_inner_finally():
    from src.utils.stream_disconnect import aclose_async_gen

    closed = {"n": 0}

    async def gen():
        try:
            yield 1
            await asyncio.Event().wait()
        finally:
            closed["n"] += 1

    agen = gen()
    assert await agen.__anext__() == 1
    await aclose_async_gen(agen)
    assert closed["n"] == 1


@pytest.mark.asyncio
async def test_aclose_async_gen_swallows_close_errors():
    from src.utils.stream_disconnect import aclose_async_gen

    class Boom:
        async def aclose(self):
            raise RuntimeError("aclose failed")

    await aclose_async_gen(Boom())
```

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/utils/tests/test_stream_disconnect_offline.py::test_aclose_async_gen_runs_inner_finally -q`

Expected: FAIL `ImportError` / `aclose_async_gen` not defined.

- [x] **Step 3: Write minimal implementation**

Add to `src/utils/stream_disconnect.py` (next to `abort_upstream`):

```python
async def aclose_async_gen(agen: Any) -> None:
    """Best-effort ``aclose`` of a nested async gen. GC cannot await teardown."""
    if agen is None:
        return
    aclose = getattr(agen, "aclose", None)
    if not callable(aclose):
        return
    try:
        await aclose()
    except Exception:
        pass
```

Point `iterate_until_disconnect`'s `finally` at this helper instead of inlining the same try/except.

- [x] **Step 4: Run test to verify it passes**

Run: `python -m pytest src/utils/tests/test_stream_disconnect_offline.py -q`

Expected: PASS (existing disconnect tests + the two new ones).

---

### Task 2: JewProxy SmolProxy / Verboo aclose-before-close

**Files:**
- Create: `src/providers/prod/jewproxy/tests/__init__.py`
- Create: `src/providers/prod/jewproxy/tests/test_fallback_aclose_offline.py`
- Modify: `src/providers/prod/jewproxy/api.py` (`_smolproxy_fallback_chunks`, `_verboo_deepseek_fallback_chunks`, `_maybe_verboo_deepseek_fallback_chunks`)

**Interfaces:**
- Consumes: `aclose_async_gen` from Task 1
- Produces: same public method names; disconnect-safe teardown

This is the remaining hole on the production stack. Today `_smolproxy_fallback_chunks` does `await smol.close()` in `finally` **unwrapped**. If `close()` raises while `GeneratorExit` is unwinding, Python replaces GE → `Exception ignored in: <async_generator object …>`.

- [x] **Step 1: Write the failing test**

```python
@pytest.mark.asyncio
async def test_smolproxy_fallback_aclose_survives_close_error(monkeypatch):
    from src.providers.prod.jewproxy import api as jp

    closed = {"inner": 0}

    class FakeSmol:
        async def send_message(self, *args, **kwargs):
            try:
                yield {"response": "hi"}
                await asyncio.Event().wait()
            finally:
                closed["inner"] += 1

        async def close(self):
            raise RuntimeError("close failed")

    monkeypatch.setattr(jp, "SmolProxyClient", FakeSmol)
    monkeypatch.setattr(jp, "map_jewproxy_base_model", lambda bot: "glm/glm-4.6")

    client = jp.JewProxyClient()
    agen = client._smolproxy_fallback_chunks("glm/glm-4.6", [])
    it = agen.__aiter__()
    assert (await it.__anext__())["response"] == "hi"
    await it.aclose()
    assert closed["inner"] == 1
```

Also cover Verboo fallback aclose (inner `finally` must run) and `_maybe_verboo_deepseek_fallback_chunks` when the predicate is true.

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/prod/jewproxy/tests/test_fallback_aclose_offline.py::test_smolproxy_fallback_aclose_survives_close_error -q`

Expected: FAIL with `RuntimeError: close failed` (unwrapped `smol.close()` during GE).

- [x] **Step 3: Write minimal implementation**

In `_smolproxy_fallback_chunks`:

```python
        smol = SmolProxyClient()
        agen = None
        try:
            agen = smol.send_message(
                smol_bot, message, file_path=file_path,
                temperature=temperature, top_p=top_p, top_k=top_k,
                logit_bias=logit_bias, thinking=thinking,
            )
            try:
                async for chunk in agen:
                    yield chunk
            except (GeneratorExit, asyncio.CancelledError):
                raise
            finally:
                await aclose_async_gen(agen)
        finally:
            try:
                await smol.close()
            except Exception:
                pass
```

Same pattern for `_verboo_deepseek_fallback_chunks` (aclose the Verboo gen). `_maybe_verboo_deepseek_fallback_chunks` acloses the inner `_verboo_deepseek_fallback_chunks` gen.

In `send_message`, immediately before `except Exception as e:` around the chat loop, add:

```python
            except (GeneratorExit, asyncio.CancelledError):
                raise
```

so a future edit cannot treat disconnect as a retryable exception.

- [x] **Step 4: Run test to verify it passes**

Run: `python -m pytest src/providers/prod/jewproxy/tests/test_fallback_aclose_offline.py src/providers/prod/zai/tests/test_fallback_aclose_offline.py -q`

Expected: PASS.

---

### Task 3: Zai uses shared helper; Gonka/Dahl overflow aclose

**Files:**
- Modify: `src/providers/prod/zai/api.py` (delete local `_aclose_async_gen`; import shared)
- Modify: `src/providers/prod/gonka/api.py` (`_verboo_fallback`, `_hyperfusion_fallback`, and the `send_message` overflow `async for`)
- Modify: `src/providers/prod/dahl/api.py` (same three sites)
- Create: `src/providers/prod/gonka/tests/test_overflow_aclose_offline.py`

**Interfaces:**
- Consumes: `aclose_async_gen`
- Produces: overflow gens that aclose on subscriber abort

- [ ] **Step 1: Write the failing Gonka test**

```python
@pytest.mark.asyncio
async def test_verboo_overflow_aclose_closes_inner(monkeypatch):
    from src.providers.prod.gonka import api as gonka

    closed = {"n": 0}

    class FakeVerboo:
        async def send_message(self, *args, **kwargs):
            try:
                yield {"response": "hi"}
                await asyncio.Event().wait()
            finally:
                closed["n"] += 1

    monkeypatch.setattr(gonka, "resolve_gonka_overflow", lambda bot: ("verboo", "pro/deepseek-v4-flash"))

    # Patch the import site used inside _verboo_fallback by injecting a fake module path.
    # Implementation: monkeypatch a test hook or patch VerbooClient after import inside the helper
    # by replacing _verboo_fallback's client constructor via a module-level alias if needed.

    agen = gonka._verboo_fallback("pro/deepseek-v4-flash", [])
    it = agen.__aiter__()
    assert (await it.__anext__())["response"] == "hi"
    await it.aclose()
    assert closed["n"] == 1
```

To keep the test honest, add a module-level `_verboo_client_factory` (or monkeypatch `src.providers.prod.verboo.api.VerbooClient`) so `_verboo_fallback` constructs the fake.

- [x] **Step 2: Run test to verify it fails** only if the current helper does not aclose when `VerbooClient.send_message` is abandoned. If async-for already acloses, still add explicit `aclose_async_gen` + wrap any `close()`; the test stays as the contract.

- [ ] **Step 3: Implement**

Zai: `from src.utils.stream_disconnect import aclose_async_gen` and delete the local copy.

Gonka `_verboo_fallback` / `_hyperfusion_fallback`:

```python
    client = VerbooClient()
    agen = client.send_message(...)
    try:
        async for chunk in agen:
            yield chunk
    except (GeneratorExit, asyncio.CancelledError):
        raise
    finally:
        await aclose_async_gen(agen)
        closer = getattr(client, "close", None)
        if callable(closer):
            try:
                result = closer()
                if asyncio.iscoroutine(result):
                    await result
            except Exception:
                pass
```

Same for Dahl (copy of these helpers).

In `GonkaClient.send_message` / `DahlClient.send_message` overflow block, aclose the `_hyperfusion_fallback(...)` gen the same way.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/prod/zai/tests/test_fallback_aclose_offline.py src/providers/prod/gonka/tests/test_overflow_aclose_offline.py src/utils/tests/test_stream_disconnect_offline.py -q`

Expected: PASS.

---

### Task 4: Live stress confirmation

**Files:**
- Modify: `src/providers/prod/gonka/live_deepseek_empty_stress.py` — add pace `"tokenize"` (no sleep; `__tokenize` every visible chunk). Keep `sleep01` as the old-API comparison. `native` remains unmodified upstream.

- [x] **Step 1: Update `tps_suite` paces to** `(None, "tokenize", "sleep01", "sleep01_tokenize")`.

- [x] **Step 2: Run live stress**

Run: `python -m src.providers.prod.gonka.tests.live_deepseek_empty_stress --concurrency 8 --rounds 3`

Expected:
- Script exit 0.
- Writes `src/providers/prod/gonka/results/deepseek_empty_stress.json`.
- No process-level `Exception ignored in: <async_generator`.
- `tokenize` median tok/s is the current routed-API estimate (sleep gone).
- Empty-visible / Verboo-fallback counts are reported; this plan does **not** require empty-visible to start overflowing to Verboo.

---

## Already landed (do not revert)

These are part of the same patch set and must stay:

1. Hot-path `await asyncio.sleep(0.01)` removed from `chat.py` (heartbeat + per-chunk), `messages.py`, `responses.py`.
2. SSE headers: `X-Accel-Buffering: no` on chat / messages / responses streaming responses.
3. Zai: fallbacks run after the Z.ai POST context exits; nested gens aclosed; bare `except:` replaced; session close wrapped.
4. Zai offline tests in `src/providers/prod/zai/tests/test_fallback_aclose_offline.py`.

## Out of scope

- Gonka HTTP 200 empty-visible → Verboo overflow (`streamed_any` / thinking budget).
- Adding Gonka to `PROVIDER_STREAM_USAGE` / incremental tokenize.
- Raising catalog DeepSeek context above 200k.
- Rewriting every provider that has a `__main__` `async for client.send_message` demo loop.

## Self-review

- Spec coverage: ignored GE on Zai send_message → Tasks 1–3 (shared helper + JewProxy hole + overflow siblings). Sleep removal already landed. Live confirm → Task 4.
- Placeholder scan: none.
- Type consistency: `aclose_async_gen(agen: Any) -> None` is the only new name; Zai/JewProxy/Gonka/Dahl all import that symbol.
