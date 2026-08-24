# Remaining Runtime Rework Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Share one OpenAI choice-frame emitter across the five remaining SSE chat clients, move existing overflow maps into `runtime/overflow/`, and add tools-stream keepalives — without wrapping anyone in `AttemptRuntime`.

**Architecture:** `iter_openai_choice_chunks` is the per-frame primitive. `yield_openai_sse_deltas` splits coalesced `data:` lines and calls it. Simple 200 bodies (crofai / zai / openrouter / smolproxy chat) use the stream helper. JewProxy keeps its error/retry line loop and calls the frame emitter after `_proxy_error_text` is None. Overflow maps move; dispatch stays in the provider. Tools keepalives wrap `tool_gen` only.

**Tech Stack:** Python 3.11+, aiohttp, pytest, existing `src/providers/runtime/`

**Spec:** `docs/superpowers/specs/2026-08-15-remaining-runtime-rework-design.md`

**Shipped 2026-08-15:** Task 0 (`STREAM_TOTAL_SECONDS = 600` on routes, marketplace, live client totals, CF Playground / NoteGPT `sock_read`). Do not re-do it.

## Global Constraints

- Do not invent overflow destinations.
- Do not call `AttemptRuntime.chat_chunks` except gonka/dahl.
- Do not set `stream_mode="buffered"` except verboo / inferhub / surplusintelligence.
- Do not add `stream_options.include_usage` where it is absent.
- `thinking=True` branches that yield `{"thinking": ...}` never call `yield_openai_sse_deltas`.
- Overflow paths keep `forward_agen`. Z.AI fallbacks run after the primary POST context exits.
- Do not flatten `src/providers/runtime/`.
- Do not revive `sse_chat.py` / `key_ring.py`.
- Do not edit `models.json`. Do not revert the 16 deprecations.
- INDEX only via `build_index_inventory` + `generate_indexes` + `verify_indexes`.
- Do not swap Albert / Feather / ArliAI onto `RoundRobinKeys`.
- Do not delete `ProviderFlags` bools.
- Do not raise free-queue 180, markethub quote 30, e2b/mistral 180, or models-cache 300.
- JewProxy / SmolProxy `/v1/responses` paths are out of scope.
- Stop after Task 1 for review. Do not start Tasks 2–6 in the same session as Task 1.

## File map

| File | Role |
|---|---|
| `src/providers/runtime/timeouts.py` | `STREAM_TOTAL_SECONDS = 600` (Task 0, shipped) |
| `src/providers/runtime/sse/choice_chunks.py` | `flatten_delta_content`, `iter_openai_choice_chunks` |
| `src/providers/runtime/sse/deltas.py` | newline-split + call frame emitter |
| `src/providers/runtime/sse/__init__.py` | re-export new symbols |
| `src/providers/prod/crofai/api.py` | Task 2 extract |
| `src/providers/prod/zai/api.py` | Task 3 extract |
| `src/providers/prod/openrouter/api.py` | Task 4 extract |
| `src/providers/prod/smolproxy/api.py` | Task 5 chat extract |
| `src/providers/prod/jewproxy/api.py` | Task 6 per-frame extract |
| `src/providers/runtime/overflow/{zai,ionet,partyrock,jewproxy}.py` | Task 7 maps |
| `src/routes/normal/tools_keepalive.py` | Task 8 |
| `src/providers/tests/runtime/test_choice_chunks_offline.py` | Task 1 |
| `src/providers/prod/crofai/tests/test_inner_sse_offline.py` | Task 2 |
| `src/providers/prod/zai/tests/test_inner_sse_offline.py` | Task 3 |
| `src/providers/prod/openrouter/tests/test_inner_sse_offline.py` | Task 4 |
| `src/providers/prod/smolproxy/tests/test_inner_sse_offline.py` | Task 5 |
| `src/providers/prod/jewproxy/tests/test_choice_chunks_offline.py` | Task 6 |
| `src/providers/tests/runtime/test_overflow_maps_offline.py` | Task 7 |
| `src/routes/normal/tests/test_tools_keepalive_offline.py` | Task 8 |

---

### Task 0: Stream hop 300 → 600 (SHIPPED)

**Files:** already modified. Lock tests: `src/providers/tests/runtime/test_timeouts_offline.py`, `src/providers/tests/runtime/test_marketplace_session_offline.py`.

- [x] **Step 1:** `STREAM_TOTAL_SECONDS = 600` in `src/providers/runtime/timeouts.py`
- [x] **Step 2:** Routes, marketplace default, live client defaults, k2think/akash recreate, Salesforce chat POST, CF Playground / NoteGPT `sock_read`
- [x] **Step 3:** Lock tests pass (`10 passed`)

Do not re-open this task. Do not raise free-queue / quote / e2b.

---

### Task 1: Per-frame emitter + newline-split helper

**Files:**
- Create: `src/providers/runtime/sse/choice_chunks.py`
- Create: `src/providers/tests/runtime/test_choice_chunks_offline.py`
- Modify: `src/providers/runtime/sse/deltas.py`
- Modify: `src/providers/runtime/sse/__init__.py`

**Interfaces:**
- Consumes: `ThinkWrap`, `StreamProgress`, `note_delta`, `usage_chunk`
- Produces:
  - `flatten_delta_content(content) -> str`
  - `iter_openai_choice_chunks(data: dict, think: ThinkWrap, progress: StreamProgress, held_usage: list) -> Iterator[dict]`
  - `yield_openai_sse_deltas` still `async def (response, progress, think, held_usage)`

- [x] **Step 1: Write the failing test**

```python
# src/providers/tests/runtime/test_choice_chunks_offline.py
from src.providers.runtime.progress import StreamProgress
from src.providers.runtime.sse import (
    ThinkWrap,
    flatten_delta_content,
    iter_openai_choice_chunks,
    yield_openai_sse_deltas,
)


def test_flatten_delta_content():
    assert flatten_delta_content("hi") == "hi"
    assert flatten_delta_content([{"type": "text", "text": "a"}, "b"]) == "ab"
    assert flatten_delta_content(None) == ""
    assert flatten_delta_content(12) == ""


def test_iter_choice_chunks_think_then_content():
    think = ThinkWrap()
    progress = StreamProgress()
    held = []
    data = {
        "choices": [{"delta": {"reasoning_content": "hmm", "content": "hi"}}],
        "usage": {"prompt_tokens": 1, "completion_tokens": 1},
    }
    chunks = list(iter_openai_choice_chunks(data, think, progress, held))
    assert chunks[0] == {"response": "<think>\n"}
    assert chunks[1] == {"response": "hmm"}
    assert chunks[2] == {"response": "\n</think>\n\n"}
    assert chunks[3] == {"response": "hi"}
    assert held and held[0]["usage"]["prompt_tokens"] == 1
    assert progress.emitted_any is True
    assert "usage" not in "".join(str(c) for c in chunks)


class _CoalescedResp:
    @property
    def content(self):
        async def _gen():
            yield (
                b'data: {"choices":[{"delta":{"content":"a"}}]}\n'
                b'data: {"choices":[{"delta":{"content":"b"}}]}\n'
            )
        return _gen()


import pytest


@pytest.mark.asyncio
async def test_yield_openai_sse_deltas_splits_coalesced_lines():
    progress = StreamProgress()
    think = ThinkWrap()
    held = []
    out = []
    async for chunk in yield_openai_sse_deltas(_CoalescedResp(), progress, think, held):
        out.append(chunk)
    assert [c.get("response") for c in out] == ["a", "b"]
```

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/tests/runtime/test_choice_chunks_offline.py -v`

Expected: FAIL on `flatten_delta_content` import.

- [x] **Step 3: Write minimal implementation**

`src/providers/runtime/sse/choice_chunks.py`:

```python
"""Per-frame OpenAI chat delta → platform chunks."""
from __future__ import annotations

from ..progress import StreamProgress, note_delta
from ..usage import usage_chunk
from .think import ThinkWrap


def flatten_delta_content(content) -> str:
    if isinstance(content, str):
        return content
    if isinstance(content, list):
        parts = []
        for item in content:
            if isinstance(item, dict):
                parts.append(item.get("text") or "")
            elif isinstance(item, str):
                parts.append(item)
        return "".join(parts)
    return ""


def iter_openai_choice_chunks(data, think: ThinkWrap, progress: StreamProgress, held_usage: list):
    _usage = usage_chunk(data)
    if _usage:
        held_usage.clear()
        held_usage.append(_usage)

    choices = data.get("choices") or []
    if not choices:
        return
    delta = choices[0].get("delta") or {}
    content = flatten_delta_content(delta.get("content"))
    reasoning = delta.get("reasoning") or delta.get("reasoning_content")
    if not isinstance(reasoning, str):
        reasoning = str(reasoning) if reasoning else ""

    note_delta(
        progress,
        content=content,
        reasoning=reasoning,
        tool_calls=delta.get("tool_calls") or delta.get("function_call"),
    )

    if reasoning and not think.closed:
        if not think.open:
            think.open = True
            yield {"response": "<think>\n"}
        yield {"response": reasoning}

    if content:
        if think.open and not think.closed:
            think.closed = True
            yield {"response": "\n</think>\n\n"}
        progress.flushed = True
        yield {"response": content}
```

Replace the body of `yield_openai_sse_deltas` in `deltas.py` so each aiohttp chunk is split on newlines, each `data:` line is parsed, and each object is passed to `iter_openai_choice_chunks`. Keep `[DONE]` / bad JSON skips. Re-export `flatten_delta_content` and `iter_openai_choice_chunks` from `sse/__init__.py`.

Existing thin-extract / Verboo / NVIDIA tests must still pass: the helper still holds usage, still opens `<think>\n`, still closes with `\n</think>\n\n`.

- [x] **Step 4: Run tests to verify they pass**

Run:

```
python -m pytest src/providers/tests/runtime/test_choice_chunks_offline.py src/providers/tests/runtime/test_thin_extract_offline.py src/providers/prod/verboo/tests/test_inner_sse_offline.py src/providers/prod/nvidia/tests/test_prethink_offline.py -v
```

Expected: PASS

- [ ] **Step 5: Commit** (not run — wait for an explicit commit request)

```bash
git add src/providers/runtime/sse/choice_chunks.py src/providers/runtime/sse/deltas.py src/providers/runtime/sse/__init__.py src/providers/tests/runtime/test_choice_chunks_offline.py
git commit -m "feat: share per-frame OpenAI choice emitter and split coalesced SSE"
```

**Stop here for review. Do not start Task 2 in this session.**

---

### Task 2: CrofAI inner SSE extract

**Files:**
- Create: `src/providers/prod/crofai/tests/__init__.py` (empty, if missing)
- Create: `src/providers/prod/crofai/tests/test_inner_sse_offline.py`
- Modify: `src/providers/prod/crofai/api.py` (`_stream_once` only)

**Interfaces:**
- Consumes: `yield_openai_sse_deltas`, `ThinkWrap`, `StreamProgress`
- Produces: `_stream_once` still sets `_last_status` / `_last_had_content`; `send_message` retry + GLM→Z.AI unchanged

- [x] **Step 1: Write the failing test**

```python
# src/providers/prod/crofai/tests/test_inner_sse_offline.py
import inspect

from src.providers.prod.crofai import api as crofai


def test_stream_once_extracts_helper_not_attempt_runtime():
    src = inspect.getsource(crofai.CrofaiClient._stream_once)
    assert "yield_openai_sse_deltas" in src
    assert "AttemptRuntime" not in inspect.getsource(crofai)
    assert "if thinking:" in src
    assert '{"thinking"' in src


def test_send_message_keeps_empty_retry_and_zai():
    src = inspect.getsource(crofai.CrofaiClient.send_message)
    assert "_stream_once" in src
    assert "_fallback_to_zai" in src
    assert "_MAX_ATTEMPTS" in inspect.getsource(crofai)
```

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/prod/crofai/tests/test_inner_sse_offline.py -v`

Expected: FAIL on `yield_openai_sse_deltas`

- [x] **Step 3: Write minimal implementation**

In `_stream_once`, after `status != 200` return, keep the `thinking` branch that yields `{"thinking": reasoning}` / `{"response": content}` and sets `_last_had_content`.

When `thinking` is False, replace the manual `readline` parse with:

```python
from src.providers.runtime.progress import StreamProgress
from src.providers.runtime.sse import ThinkWrap, yield_openai_sse_deltas

progress = StreamProgress()
think = ThinkWrap()
held_usage: list = []
async for chunk in yield_openai_sse_deltas(response, progress, think, held_usage):
    self._last_had_content = progress.emitted_any
    yield chunk
if think.open and not think.closed:
    yield {"response": "\n</think>\n\n"}
    think.closed = True
for usage in held_usage:
    yield usage
```

Do not call the helper when `thinking` is True. Do not wrap `send_message` in `AttemptRuntime`. Do not change `_fallback_to_zai`. Do not add `include_usage` (already present).

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/prod/crofai/tests/test_inner_sse_offline.py -v`

Expected: PASS

- [ ] **Step 5: Commit** (not run — wait for an explicit commit request)

```bash
git add src/providers/prod/crofai/api.py src/providers/prod/crofai/tests/test_inner_sse_offline.py
git commit -m "refactor: extract CrofAI inner SSE via shared helper"
```

---

### Task 3: Z.AI HTTP 200 extract

**Files:**
- Create: `src/providers/prod/zai/tests/test_inner_sse_offline.py`
- Modify: `src/providers/prod/zai/api.py` (200-path only)

**Interfaces:**
- Consumes: `yield_openai_sse_deltas`, `ThinkWrap`, `StreamProgress`
- Produces: `_run_fallbacks` / `_map_zai_to_fallback` / `_should_run_fallback` unchanged (Task 7 moves the maps)

- [x] **Step 1: Write the failing test**

```python
# src/providers/prod/zai/tests/test_inner_sse_offline.py
import inspect

from src.providers.prod.zai import api as zai


def test_send_message_extracts_200_path_only():
    src = inspect.getsource(zai.ZaiClient.send_message)
    assert "yield_openai_sse_deltas" in src
    assert "AttemptRuntime" not in inspect.getsource(zai)
    assert "_run_fallbacks" in src
    assert "forward_agen" in src
    assert "include_usage" in src


def test_thinking_true_still_wraps_in_response():
    src = inspect.getsource(zai.ZaiClient.send_message)
    assert '{"thinking"' not in src
```

Also re-run `src/providers/prod/zai/tests/test_fallback_aclose_offline.py` after the change.

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/prod/zai/tests/test_inner_sse_offline.py -v`

Expected: FAIL on `yield_openai_sse_deltas`

- [x] **Step 3: Write minimal implementation**

Inside the `status == 200` branch only, replace the manual `async for chunk in response.content` loop with the helper. Flush `held_usage` after the helper, then `return` so fallbacks still run only after the POST context exits.

```python
progress = StreamProgress()
think = ThinkWrap()
held_usage: list = []
async for chunk in yield_openai_sse_deltas(response, progress, think, held_usage):
    yield chunk
if think.open and not think.closed:
    yield {"response": "\n</think>\n\n"}
    think.closed = True
for usage in held_usage:
    yield usage
return
```

Do not move `_run_fallbacks` in this task. Do not construct `AttemptRuntime`. Media (`create_image` / upload) stays.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/prod/zai/tests/test_inner_sse_offline.py src/providers/prod/zai/tests/test_fallback_aclose_offline.py -v`

Expected: PASS

- [ ] **Step 5: Commit** (not run — wait for an explicit commit request)

```bash
git add src/providers/prod/zai/api.py src/providers/prod/zai/tests/test_inner_sse_offline.py
git commit -m "refactor: extract Z.AI 200-path SSE via shared helper"
```

---

### Task 4: OpenRouter inner SSE extract

**Files:**
- Create: `src/providers/prod/openrouter/tests/__init__.py` (empty, if missing)
- Create: `src/providers/prod/openrouter/tests/test_inner_sse_offline.py`
- Modify: `src/providers/prod/openrouter/api.py` (200 body only)

**Interfaces:**
- Consumes: `yield_openai_sse_deltas`, `ThinkWrap`, `StreamProgress`
- Produces: key shuffle / `used_keys` / non-200 `continue` unchanged

- [x] **Step 1: Write the failing test**

```python
# src/providers/prod/openrouter/tests/test_inner_sse_offline.py
import inspect

from src.providers.prod.openrouter import api as openrouter


def test_send_message_extracts_helper_not_attempt_runtime():
    src = inspect.getsource(openrouter.OpenRouterClient.send_message)
    assert "yield_openai_sse_deltas" in src
    assert "AttemptRuntime" not in inspect.getsource(openrouter)
    assert "used_keys" in src
    assert "◁" in src
```

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/prod/openrouter/tests/test_inner_sse_offline.py -v`

Expected: FAIL on `yield_openai_sse_deltas`

- [x] **Step 3: Write minimal implementation**

On HTTP 200, replace the inner `async for line in response.content` parse with the helper. After each helper chunk, if the chunk is a `response` string containing `◁` or `▷`, apply `.replace("◁", "<").replace("▷", ">")` on **both** reasoning and content (live code already did this; the earlier “reasoning-only / think-open” note was wrong). Track `content_received` as `progress.emitted_any`. Flush `held_usage` on a successful attempt. Keep the existing empty-stream / next-key behavior.

Do not change hybrid memory, `max_tokens` 0, or key shuffle.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/prod/openrouter/tests/test_inner_sse_offline.py -v`

Expected: PASS (`6 passed`, plus the shared extract suite)

- [ ] **Step 5: Commit** (not run — wait for an explicit commit request)

```bash
git add src/providers/prod/openrouter/api.py src/providers/prod/openrouter/tests/test_inner_sse_offline.py
git commit -m "refactor: extract OpenRouter inner SSE via shared helper"
```

---

### Task 5: SmolProxy chat SSE extract

**Files:**
- Create: `src/providers/prod/smolproxy/tests/test_inner_sse_offline.py` (package `__init__.py` already exists)
- Modify: `src/providers/prod/smolproxy/api.py` (chat 200 body only)

**Interfaces:**
- Consumes: `yield_openai_sse_deltas`, `ThinkWrap`, `StreamProgress`
- Produces: Responses / xAI multi-agent forks unchanged; param pin/drop / key rotate unchanged

- [x] **Step 1: Write the failing test**

```python
# src/providers/prod/smolproxy/tests/test_inner_sse_offline.py
import inspect

from src.providers.prod.smolproxy import api as smolproxy


def test_chat_path_extracts_helper():
    src = inspect.getsource(smolproxy.SmolProxyClient.send_message)
    assert "yield_openai_sse_deltas" in src
    assert "AttemptRuntime" not in inspect.getsource(smolproxy)
    assert "_send_responses_message" in src
    assert "is_openai_codex_model" in src
    assert "is_xai_multi_agent" in src


def test_responses_helper_untouched():
    src = inspect.getsource(smolproxy.SmolProxyClient._send_responses_message)
    assert "yield_openai_sse_deltas" not in src
```

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/prod/smolproxy/tests/test_inner_sse_offline.py -v`

Expected: FAIL on `yield_openai_sse_deltas`

- [x] **Step 3: Write minimal implementation**

Only the chat `response.status == 200` body. Replace the manual delta loop with the helper. Set `content_received = progress.emitted_any`. On empty, keep `retry_backoff` + `continue`. On success, close think if needed, flush `held_usage`, `return`.

Do not touch `_send_responses_message`. Do not change `sock_read=600` / `total=None`. Do not add `AttemptRuntime`.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/prod/smolproxy/tests/test_inner_sse_offline.py -v`

Expected: PASS (`11 passed`; shared extract suite `52 passed`)

- [ ] **Step 5: Commit** (not run — wait for an explicit commit request)

```bash
git add src/providers/prod/smolproxy/api.py src/providers/prod/smolproxy/tests/test_inner_sse_offline.py
git commit -m "refactor: extract SmolProxy chat SSE via shared helper"
```

---

### Task 6: JewProxy chat per-frame extract (last)

**Files:**
- Create: `src/providers/prod/jewproxy/tests/test_choice_chunks_offline.py`
- Modify: `src/providers/prod/jewproxy/api.py` (chat 200 happy-path frames only)

**Interfaces:**
- Consumes: `iter_openai_choice_chunks`, `ThinkWrap`, `StreamProgress` — **not** `yield_openai_sse_deltas` on this client
- Produces: `_proxy_error_text` / param pin / SmolProxy fallback / Verboo DeepSeek hop / `prompt_thinking` unchanged

- [x] **Step 1: Write the failing test**

```python
# src/providers/prod/jewproxy/tests/test_choice_chunks_offline.py
import inspect

from src.providers.prod.jewproxy import api as jewproxy


def test_chat_uses_per_frame_emitter_not_stream_helper():
    src = inspect.getsource(jewproxy.JewProxyClient.send_message)
    assert "iter_openai_choice_chunks" in src
    assert "yield_openai_sse_deltas" not in src
    assert "AttemptRuntime" not in inspect.getsource(jewproxy)
    assert "_proxy_error_text" in src
    assert "_smolproxy_fallback_chunks" in src
    assert "should_verboo_fallback_deepseek_context" in inspect.getsource(jewproxy)
    assert "convert_prompt_think" in src


def test_responses_path_untouched():
    src = inspect.getsource(jewproxy)
    assert "_send_responses" in src or "responses" in src.lower()
```

If `send_message` is too large for a single `getsource` slice, assert on the module source instead — same strings.

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/prod/jewproxy/tests/test_choice_chunks_offline.py src/providers/prod/jewproxy/tests/test_fallback_aclose_offline.py -v`

Expected: FAIL on `iter_openai_choice_chunks`

- [x] **Step 3: Write minimal implementation**

Keep the `async for raw in response.content` loop and the `_proxy_error_text` block exactly. After `err_text is None`, replace the manual reasoning/content/usage parse with:

```python
for chunk in iter_openai_choice_chunks(data, think, progress, held_usage):
    text = chunk.get("response")
    if isinstance(text, str) and ("◁" in text or "▷" in text):
        chunk = {**chunk, "response": text.replace("◁", "<").replace("▷", ">")}
    if chunk.get("response"):
        content_received = True
    if prompt_thinking and chunk.get("response") and not (think.open and not think.closed and chunk["response"].startswith("<think>")):
        piece, think_start, think_end = convert_prompt_think(
            chunk["response"], think_start, think_end
        )
        if piece:
            yield {"response": piece}
        continue
    yield chunk
```

Wire `think_start` / `think_end` from `think.open` / `think.closed` at the end of the attempt so existing retry flags stay consistent (`content_received = progress.emitted_any` is enough if you drop the locals).

`prompt_thinking` stays on the old `convert_prompt_think` path; the frame emitter runs only when `prompt_thinking` is False. Triangle replace on the emitter path applies to both reasoning and content. That is the acceptable minimal extract.

Flush `held_usage` only on a successful return (not on empty retry / proxy error retry).

Do not extract the Responses fork. Do not construct `AttemptRuntime`. Do not change overflow destinations.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/providers/prod/jewproxy/tests/test_choice_chunks_offline.py src/providers/prod/jewproxy/tests/test_fallback_aclose_offline.py -v`

Expected: PASS (`14 passed` here; shared extract suite `66 passed`)

- [ ] **Step 5: Commit** (not run — wait for an explicit commit request)

```bash
git add src/providers/prod/jewproxy/api.py src/providers/prod/jewproxy/tests/test_choice_chunks_offline.py
git commit -m "refactor: emit JewProxy chat deltas via shared per-frame helper"
```

---

### Task 7: Move overflow maps into `runtime/overflow/`

**Files:**
- Create: `src/providers/runtime/overflow/zai.py`
- Create: `src/providers/runtime/overflow/ionet.py`
- Create: `src/providers/runtime/overflow/partyrock.py`
- Create: `src/providers/runtime/overflow/jewproxy.py`
- Create: `src/providers/tests/runtime/test_overflow_maps_offline.py`
- Modify: `src/providers/runtime/overflow/__init__.py`
- Modify: `src/providers/prod/zai/api.py`
- Modify: `src/providers/prod/ionet/api.py`
- Modify: `src/providers/prod/partyrock/api.py`
- Modify: `src/providers/prod/jewproxy/api.py`

**Interfaces:**
- Consumes: existing maps/predicates (copy verbatim)
- Produces:
  - `ZAI_FALLBACK_MODEL_MAP`, `map_zai_to_fallback(zai_model: str) -> list`, `should_run_zai_fallback(status: int, error_code: str, error_msg: str) -> bool`
  - `IONET_NVIDIA_FALLBACK_MAP`, `resolve_ionet_nvidia_fallback(bot: str) -> str | None`
  - `PARTYROCK_ZAI_FALLBACK_MODEL: str` (`"glm-4.6"`)
  - `should_verboo_fallback_deepseek_context(service: str, text: str) -> bool`

- [x] **Step 1: Write the failing test**

```python
# src/providers/tests/runtime/test_overflow_maps_offline.py
from src.providers.runtime.overflow import (
    PARTYROCK_ZAI_FALLBACK_MODEL,
    map_zai_to_fallback,
    resolve_ionet_nvidia_fallback,
    should_run_zai_fallback,
    should_verboo_fallback_deepseek_context,
)


def test_zai_map_glm52():
    chain = map_zai_to_fallback("glm-5.2")
    assert chain[0] == ("jewproxy", "glm/glm-5.2")
    assert chain[1][0] == "venice"
    assert chain[2][0] == "chutes"


def test_zai_should_fallback():
    assert should_run_zai_fallback(429, "", "") is True
    assert should_run_zai_fallback(500, "1234", "") is True
    assert should_run_zai_fallback(400, "", "nope") is False


def test_ionet_nvidia_preserves_thinking():
    assert resolve_ionet_nvidia_fallback("deepseek-ai/DeepSeek-V4-Pro") == (
        "deepseek-ai/deepseek-v4-pro"
    )
    assert resolve_ionet_nvidia_fallback("deepseek-ai/DeepSeek-V4-Pro:thinking") == (
        "deepseek-ai/deepseek-v4-pro:thinking"
    )
    assert resolve_ionet_nvidia_fallback("other") is None


def test_partyrock_dest_unchanged():
    assert PARTYROCK_ZAI_FALLBACK_MODEL == "glm-4.6"


def test_jewproxy_verboo_predicate():
    assert should_verboo_fallback_deepseek_context("deepseek", "context length exceeded") is True
    assert should_verboo_fallback_deepseek_context("glm", "context length exceeded") is False


def test_providers_import_from_runtime():
    import inspect
    from src.providers.prod.ionet import api as ionet
    from src.providers.prod.partyrock import api as partyrock
    from src.providers.prod.zai import api as zai

    assert "src.providers.runtime.overflow" in inspect.getsource(zai)
    assert "src.providers.runtime.overflow" in inspect.getsource(ionet)
    assert "src.providers.runtime.overflow" in inspect.getsource(partyrock)
    assert "TELNYX_OVERFLOW_FALLBACK_MAP" not in inspect.getsource(
        __import__("src.providers.runtime.overflow", fromlist=["*"])
    )
```

Copy the exact DeepSeek context-limit strings from `is_jewproxy_context_limit_error` so the predicate test matches today's function, not a guessed phrase. Live needle is ``context size limit`` (not ``context length exceeded``). `is_jewproxy_context_limit_error` moved with the predicate so `proxy_error_is_retryable` can import it back without a circular import.

- [x] **Step 1: Write the failing test** (live needle; ImportError RED)

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/providers/tests/runtime/test_overflow_maps_offline.py -v`

Expected: FAIL on import

- [x] **Step 3: Write minimal implementation**

Move the functions **verbatim**. In each provider, import the public names and alias the old privates so existing monkeypatches keep working:

```python
from src.providers.runtime.overflow import map_zai_to_fallback as _map_zai_to_fallback
from src.providers.runtime.overflow import should_run_zai_fallback as _should_run_fallback
```

`test_fallback_aclose_offline.py` patches `zai_api._map_zai_to_fallback` — the alias is required.

Do not move Telnyx. Do not add destinations. Do not move `_run_fallbacks` or IoNet's NVIDIA client construction.

- [x] **Step 4: Run tests**

Run:

```
python -m pytest src/providers/tests/runtime/test_overflow_maps_offline.py src/providers/prod/zai/tests/test_fallback_aclose_offline.py src/providers/prod/jewproxy/tests/test_fallback_aclose_offline.py -v
```

Expected: PASS (`71 passed` on the shared extract + overflow + dispatch suite)

- [ ] **Step 5: Commit** (not run — wait for an explicit commit request)

```bash
git add src/providers/runtime/overflow src/providers/prod/zai/api.py src/providers/prod/ionet/api.py src/providers/prod/partyrock/api.py src/providers/prod/jewproxy/api.py src/providers/tests/runtime/test_overflow_maps_offline.py
git commit -m "refactor: move zai/ionet/partyrock/jewproxy overflow maps into runtime"
```

---

### Task 8: Tools-stream SSE keepalives

**Files:**
- Create: `src/routes/normal/tools_keepalive.py`
- Create: `src/routes/normal/tests/test_tools_keepalive_offline.py` (and `src/routes/normal/tests/__init__.py` if missing)
- Modify: `src/routes/normal/chat.py` (streaming tools `return StreamingResponse` only)
- Modify: `src/routes/normal/messages.py` (streaming native-tools return only, if it also returns a bare `tool_gen`)
- Modify: `src/routes/normal/responses.py` (same)

**Interfaces:**
- Consumes: an async iterator of SSE bytes
- Produces: `with_sse_keepalives(agen, interval: float = 10) -> AsyncIterator[bytes]` yielding `b": keepalive\n\n"` when idle

- [x] **Step 1: Write the failing test**

```python
# src/routes/normal/tests/test_tools_keepalive_offline.py
import asyncio
import inspect

import pytest

from src.routes.normal.tools_keepalive import with_sse_keepalives


@pytest.mark.asyncio
async def test_keepalive_emits_comment_before_chunk():
    async def silent_then_chunk():
        await asyncio.sleep(0.05)
        yield b'data: {"ok":true}\n\n'

    out = []
    async for item in with_sse_keepalives(silent_then_chunk(), interval=0.01):
        out.append(item)
    assert b": keepalive\n\n" in out
    assert out[-1] == b'data: {"ok":true}\n\n'


def test_chat_tools_path_wraps_keepalive():
    from src.routes.normal import chat

    src = inspect.getsource(chat._chat_completions_impl)
    assert "with_sse_keepalives" in src
    assert "native_tool_chat_chunks" in src
```

- [x] **Step 2: Run test to verify it fails**

Run: `python -m pytest src/routes/normal/tests/test_tools_keepalive_offline.py -v`

Expected: FAIL on import

- [x] **Step 3: Write minimal implementation**

```python
# src/routes/normal/tools_keepalive.py
from __future__ import annotations

import asyncio
from typing import AsyncIterator


KEEPALIVE_FRAME = b": keepalive\n\n"


async def with_sse_keepalives(agen, interval: float = 10) -> AsyncIterator[bytes]:
    it = agen.__aiter__()
    pending = asyncio.ensure_future(it.__anext__())
    try:
        while True:
            done, _ = await asyncio.wait({pending}, timeout=interval)
            if not done:
                yield KEEPALIVE_FRAME
                continue
            try:
                yield pending.result()
            except StopAsyncIteration:
                return
            pending = asyncio.ensure_future(it.__anext__())
    finally:
        if not pending.done():
            pending.cancel()
        aclose = getattr(agen, "aclose", None)
        if aclose is not None:
            await aclose()
```

In `chat.py` streaming tools return:

```python
return StreamingResponse(
    with_sse_keepalives(tool_gen),
    media_type="text/event-stream",
    headers={"Content-Type": "text/event-stream"},
)
```

Mirror on messages / responses **only** if those routes also return a bare tools generator today. Do not wrap `generate_chunks` (it already heartbeats). Do not change `timeout_duration`.

- [x] **Step 4: Run tests**

Run: `python -m pytest src/routes/normal/tests/test_tools_keepalive_offline.py -v`

Expected: PASS (`5 passed`). Wrapper finally cancels the pending `__anext__` before `aclose()` so a running inner gen does not raise `RuntimeError: aclose(): asynchronous generator is already running`.

- [ ] **Step 5: Commit** (not run — wait for an explicit commit request)

```bash
git add src/routes/normal/tools_keepalive.py src/routes/normal/tests/test_tools_keepalive_offline.py src/routes/normal/chat.py src/routes/normal/messages.py src/routes/normal/responses.py
git commit -m "fix: SSE comment keepalives on native tools streams"
```

---

### Task 9: INDEX regen

**Files:**
- Modify: `src/providers/INDEX.md` and `src/tools/providers/INDEX.md` via the scripts only

- [x] **Step 1: No new test** — run the existing verifier

- [x] **Step 2: Regenerate**

```
python -m src.providers.tests.build_index_inventory
python -m src.providers.tests.generate_indexes
python -m src.providers.tests.verify_indexes
```

Expected: 57 providers, tools count unchanged, `ok`.

Ran: `providers=57 tools=35`, `consistency ... ok`, `verify_indexes` → `ok`. Scripts rewrote the files with no git diff.

- [ ] **Step 3: Commit only if the scripts wrote a diff** (skipped — no diff; wait for an explicit commit request)

```bash
git add src/providers/INDEX.md src/tools/providers/INDEX.md src/providers/tests/results/index_inventory.json
git commit -m "docs: regenerate provider INDEX after remaining runtime rework"
```

---

### Task 10: Explicit no-ops

Do not write code for these. If a later session starts them, it needs a new spec.

- Do not wrap any remaining client in `AttemptRuntime`.
- Do not add `runtime/responses/` (OpenAI / jewproxy / smolproxy Codex).
- Do not add a curl_cffi iterator (chutes / venice).
- Do not extract vermal dual-codec or comparia dual-lane. `cfplayground` is deprecated (keep `cf`).
- Do not set `stream_mode="buffered"` on anyone new.
- Do not delete `ProviderFlags` bools.
- Do not raise free-queue TTL 180, markethub quote 30, e2b/mistral 180, models-cache 300.
- Do not flatten `src/providers/runtime/`.
- Do not edit `models.json` or revert deprecations.

- [x] **Step 1: Confirm no new files under `src/providers/runtime/responses/`**

Run: `python -c "from pathlib import Path; p=Path('src/providers/runtime/responses'); assert not p.exists()"`

Confirmed: path does not exist. `AttemptRuntime(` only in gonka/dahl. `stream_mode="buffered"` only verboo / inferhub / surplusintelligence. Runtime layout still nested (auth / attempt / overflow / profile / progress / sse / usage / kwargs / marketplace).

- [x] **Step 2: No commit**

---

## Self-review

1. **Spec coverage:** Slice A = Tasks 2–6 (after Task 1 primitive). Slice B remaining = Task 8 (Task 0 shipped). Slice C = Task 7. Slice D/E = Task 10. INDEX = Task 9.
2. **Placeholder scan:** no TBD. JewProxy `prompt_thinking` has an explicit fallback (emitter only when False).
3. **Type consistency:** `iter_openai_choice_chunks(data, think, progress, held_usage)` and `flatten_delta_content(content) -> str` are the names Tasks 2–6 import.
