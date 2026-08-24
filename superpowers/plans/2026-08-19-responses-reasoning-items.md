# Responses Reasoning Items Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. Do **not** commit unless the user explicitly asks.

**Goal:** `/v1/responses` emits OpenAI-shaped `type=reasoning` items and `response.reasoning_text.*` events instead of stuffing `<think>` tags into `output_text`.

**Architecture:** Switch the Responses remapper from `MERGED` to `SEPARATE` (same as `/v1/messages`). A shared `ResponsesReasoningSink` lazy-opens a reasoning item, then a message item. Input conversion replays `type=reasoning` as assistant `reasoning` plus `<think>` in history so tool loops keep continuity. Tools `/v1/responses` uses the same sink. Usage reports `output_tokens_details.reasoning_tokens`. Do not mint `encrypted_content` or honor `previous_response_id` store.

**Tech Stack:** Python 3, existing `surface_chunk` / `display_reasoning_mode`, OpenAI Responses item + gpt-oss `reasoning_text` events.

**Spec:** This plan. Official contracts: OpenAI Reasoning models + Responses reference; OpenRouter Reasoning Tokens (fetched 2026-08-19). Prior passthrough plan: `docs/superpowers/plans/2026-08-19-reasoning-token-passthrough.md`.

## Global Constraints

- `thinking_enabled` is a **display** toggle. Do not send `reasoning_effort=none` or omit `include_reasoning` because the UI toggle is on.
- When display is on: emit a `type=reasoning` item with `content: [{type: reasoning_text}]`. When off: `EXCLUDE` remapper — no reasoning item, no `<think>` in `output_text`.
- Never put `<think>` / `</think>` in Responses `output_text`.
- Do not invent `encrypted_content`. Do not implement stored `previous_response_id`.
- Deep-research raw proxy (`send_response_raw` for o3/o4-mini-deep-research) is unchanged.
- Do not rewrite Class D / deprecated providers. Do not add LarpRouter `include_reasoning`.
- `output_tokens` still includes reasoning tokens (OpenAI/OR). `reasoning_tokens` is the split, not extra on top of `output_tokens`.
- A reasoning-only hop is **not** empty (`flushed=True`).
- Offline tests first. Do not commit unless asked.

## Why not MERGED

`merged_reasoning_mode` inlines `<think>` into `content_delta`. Official Responses clients look for `output[].type == "reasoning"` and `response.reasoning_text.delta`. Putting tags in `output_text` is the wrong item type. `surface_chunk(..., SEPARATE)` already splits thinking-key and response `<think>` into `reasoning_delta` vs `content_delta`. Responses must use that split.

## File map

- Create: `src/routes/normal/responses/emit.py` — sink + item builders + SSE tuples
- Create: `src/routes/normal/tests/test_responses_reasoning_items_offline.py`
- Modify: `src/routes/normal/responses/stream.py` — SEPARATE, sink, no eager message item, flushed/usage
- Modify: `src/routes/normal/responses/complete.py` — same; do not `continue` on reasoning-only chunks
- Modify: `src/routes/normal/responses/helpers.py` — pass `reasoning_tokens` into usage
- Modify: `src/tools/_converters.py` — replay `type=reasoning`
- Modify: `src/tools/tests/test_responses_input_offline.py`
- Modify: `src/tools/responses_handler.py` — sink + `thinking_enabled`
- Modify: `src/routes/normal/responses/dispatch.py` — pass `thinking_enabled`
- Modify: `src/routes/normal/tests/test_responses_thinking_key_offline.py`
- Modify: `src/providers/tests/live_reasoning_surfaces.py` — Responses check = SEPARATE reasoning, not `<think>` in content

---

### Task 1: Sink + item builders (TDD)

**Files:**
- Create: `src/routes/normal/responses/emit.py`
- Test: `src/routes/normal/tests/test_responses_reasoning_items_offline.py`

**Interfaces:**
- Produces:
  - `reasoning_item(item_id: str, text: str) -> dict`
  - `message_item(item_id: str, text: str) -> dict`
  - `responses_output_items(*, reasoning_text: str, message_text: str, reasoning_id: str, message_id: str) -> list[dict]`
  - `extract_reasoning_item_text(item: dict) -> str`
  - `replay_think_wrap(text: str) -> str` → `"<think>\n{text}\n</think>\n\n"` if text else `""`
  - `class ResponsesReasoningSink` with `feed(content: str, reasoning: str) -> list[tuple[str, dict]]`, `finish() -> list[tuple[str, dict]]`, `output_list() -> list[dict]`, fields `reasoning_text`, `message_text`, `reasoning_id`, `message_id`
  - `sse_bytes(event: str, data: dict) -> bytes`

- [ ] **Step 1: Write failing tests**

```python
# test_responses_reasoning_items_offline.py
from src.routes.normal.responses.emit import (
    ResponsesReasoningSink,
    extract_reasoning_item_text,
    reasoning_item,
    responses_output_items,
)

def test_output_items_reasoning_then_message():
    items = responses_output_items(
        reasoning_text="why",
        message_text="hi",
        reasoning_id="rs_1",
        message_id="msg_1",
    )
    assert items[0]["type"] == "reasoning"
    assert items[0]["content"][0] == {"type": "reasoning_text", "text": "why"}
    assert items[0]["summary"] == []
    assert "encrypted_content" not in items[0]
    assert items[1]["type"] == "message"
    assert items[1]["content"][0]["text"] == "hi"
    assert "<think>" not in items[1]["content"][0]["text"]

def test_sink_emits_reasoning_text_events_before_output_text():
    sink = ResponsesReasoningSink()
    ev = sink.feed("", "plan")
    kinds = [e[0] for e in ev]
    assert "response.output_item.added" in kinds
    assert "response.reasoning_text.delta" in kinds
    assert all(e[1].get("delta") != "plan" or e[0] == "response.reasoning_text.delta" for e in ev)
    ev2 = sink.feed("hi", "")
    kinds2 = [e[0] for e in ev2]
    assert "response.reasoning_text.done" in kinds2
    assert "response.output_text.delta" in kinds2
    assert sink.output_list()[0]["type"] == "reasoning"
    assert sink.output_list()[1]["content"][0]["text"] == "hi"

def test_sink_exclude_has_no_reasoning_item():
    sink = ResponsesReasoningSink()
    sink.feed("vis", "")
    sink.finish()
    assert [i["type"] for i in sink.output_list()] == ["message"]

def test_extract_reasoning_item_text():
    assert "why" in extract_reasoning_item_text({
        "type": "reasoning",
        "content": [{"type": "reasoning_text", "text": "why"}],
        "summary": [{"type": "summary_text", "text": "sum"}],
    })
```

- [ ] **Step 2: Run to verify fail**

Run: `python -m src.routes.normal.tests.test_responses_reasoning_items_offline`

Expected: `ImportError` for `src.routes.normal.responses.emit`

- [ ] **Step 3: Implement `emit.py`**

Sink rules:
- Do not pre-open a message item.
- On first non-empty `reasoning`: open `output_item.added` (`type=reasoning`, `id=rs_…`), `content_part.added` (`part.type=reasoning_text`), then `response.reasoning_text.delta`.
- On first non-empty `content` after reasoning: close reasoning (`reasoning_text.done`, `content_part.done`, `output_item.done`), then open message + `output_text.delta`.
- If content arrives first: open message only (no empty reasoning item).
- If reasoning resumes after message: close message, open a **new** reasoning item (`rs_…` new id), then later a new message item if more content arrives. Append extra items onto `output_list()` in order.
- `finish()` closes whichever item is open. Reasoning-only → output is just reasoning item(s).
- Empty feed → `output_list() == []`.
- `reasoning_item` must not include `encrypted_content`.

Event payloads (gpt-oss / OpenAI cookbook):

```python
{"type": "response.reasoning_text.delta", "item_id": rs_id, "output_index": n, "content_index": 0, "delta": text}
{"type": "response.reasoning_text.done", "item_id": rs_id, "output_index": n, "content_index": 0, "text": full}
```

- [ ] **Step 4: Re-run tests — expect PASS**

---

### Task 2: Non-stream + stream routes

**Files:**
- Modify: `src/routes/normal/responses/complete.py`
- Modify: `src/routes/normal/responses/stream.py`
- Modify: `src/routes/normal/responses/helpers.py`
- Modify: `src/routes/normal/tests/test_responses_thinking_key_offline.py`

**Interfaces:**
- Consumes: `display_reasoning_mode`, `surface_chunk`, `ResponsesReasoningSink`, `sse_bytes`
- Stop using `merged_reasoning_mode` / `close_merged_think` on this surface

- [ ] **Step 1: Update static test**

Assert responses files contain `display_reasoning_mode(`, `ResponsesReasoningSink`, `reasoning_text`, and do **not** contain `merged_reasoning_mode(` or `close_merged_think(`.

- [ ] **Step 2: complete.py**

```python
reasoning_mode = display_reasoning_mode(thinking_enabled)
sink = ResponsesReasoningSink()
# per chunk:
content_delta, reasoning_delta, thinking_state = surface_chunk(chunk, reasoning_mode, thinking_state)
if content_delta or reasoning_delta:
    observe_surfaced(hop, reasoning_mode, content_delta, reasoning_delta)
    sink.feed(content_delta, reasoning_delta)
# do NOT continue when only reasoning_delta
# flushed:
flushed = bool(sink.message_text or sink.reasoning_text)
# empty-hop exception:
if sink.message_text or sink.reasoning_text: raise
# output:
output = sink.output_list() or [message_item(sink.message_id, "")]
# usage:
reasoning_tokens = provider_reasoning_tokens(usage_acc) or await helpers.__tokenize(sink.reasoning_text)
output_tokens = provider or await helpers.__tokenize(sink.reasoning_text + sink.message_text)
# set usage_block output_tokens_details.reasoning_tokens before _apply_usage_block
# max_output_tokens counts sink.reasoning_text + sink.message_text
```

- [ ] **Step 3: stream.py**

- Keep `response.created` / `response.in_progress`.
- **Delete** the eager `output_item.added` + `content_part.added` before the hop loop (those assumed a message item at index 0).
- Inside the winning hop, `sink.feed` → `yield sse_bytes(event, data)` for each tuple.
- After hop success: `sink.finish()` then `response.completed` with `output=sink.output_list()`.
- `if last_text:` empty-fail guards become `if sink.message_text or sink.reasoning_text`.
- On hop walk `continue`, construct a **new** sink (do not leak items from a discarded hop). Because created/in_progress already fired with `output: []`, the client has not seen item-added yet if the failed hop produced no yields… **If a hop yielded events then failed**, current code already re-raises when text flushed. Keep that: if sink has text, raise; else new sink and walk.
- Deep-research branch unchanged.

- [ ] **Step 4: helpers.py**

```python
def provider_reasoning_tokens(usage_acc) -> int:
    # read last/merged completion_tokens_details.reasoning_tokens or output_tokens_details.reasoning_tokens
```

If `UsageAccumulator` has no such field, scan `usage_acc` raw merges or accept `0` and use local tokenize. Prefer reading from the last added usage dict if stored; otherwise tokenize.

`_apply_usage_block` already copies `usage_block["output_tokens_details"]["reasoning_tokens"]`. Set that key before calling it.

- [ ] **Step 5: Run**

`python -m src.routes.normal.tests.test_responses_reasoning_items_offline`
`python -m src.routes.normal.tests.test_responses_thinking_key_offline`

---

### Task 3: Replay `type=reasoning` on input

**Files:**
- Modify: `src/tools/_converters.py` `responses_input_to_chat_messages`
- Modify: `src/tools/tests/test_responses_input_offline.py`

**Interfaces:**
- Consumes: `extract_reasoning_item_text`, `replay_think_wrap` from `emit.py` (converters can import from responses.emit — avoid a cycle: `_converters` is imported by the route. `emit.py` must **not** import `_converters` or `responses/__init__`).

- [ ] **Step 1: Replace `test_reasoning_item_skipped_keeps_user`**

```python
def test_reasoning_item_replays_onto_following_assistant():
    msgs = _http_convert({"model": "x", "input": [
        {"role": "user", "content": "q"},
        {"type": "reasoning", "id": "rs_1",
         "content": [{"type": "reasoning_text", "text": "why"}],
         "summary": []},
        {"type": "message", "role": "assistant",
         "content": [{"type": "output_text", "text": "hi"}]},
    ]})
    asst = [m for m in msgs if m["role"] == "assistant"][0]
    assert asst.get("reasoning") == "why"
    assert "<think>" in (asst["content"] or "")
    assert "hi" in (asst["content"] or "")

def test_reasoning_item_replays_onto_function_call():
    msgs = _http_convert({"model": "x", "input": [
        {"type": "reasoning", "id": "rs_1",
         "summary": [{"type": "summary_text", "text": "need calc"}]},
        {"type": "function_call", "call_id": "c1", "name": "calc", "arguments": "{}"},
    ]})
    asst = msgs[-1]
    assert asst["role"] == "assistant"
    assert asst.get("reasoning") == "need calc"
    assert asst["tool_calls"][0]["id"] == "c1"
```

- [ ] **Step 2: Implement converter**

Hold `pending_reason` string. On `type=="reasoning"`: append `extract_reasoning_item_text(item)`. On following assistant message or `function_call` group: set `reasoning=pending` and prefix `replay_think_wrap(pending)` onto string content (if content is `None` and there are tool_calls, set `content` to the wrap or keep `None` and only set `reasoning`). Clear pending. Trailing pending with no follower → append `{"role":"assistant","content": wrap, "reasoning": pending}`.

If assistant content already contains `<think>`, do not double-wrap; still set `reasoning` if missing.

- [ ] **Step 3: Run** `python -m src.tools.tests.test_responses_input_offline`

---

### Task 4: Tools `/v1/responses`

**Files:**
- Modify: `src/tools/responses_handler.py`
- Modify: `src/routes/normal/responses/dispatch.py`
- Test: `src/tools/tests/test_responses_handler_offline.py`

- [ ] **Step 1: Tests**

Stream: fake chat SSE with `delta.reasoning_content` then `delta.content` → events include `response.reasoning_text.delta` and `output_text` without `<think>`.

Non-stream: `message.reasoning` + `content` → `output[0].type=="reasoning"`, `output[1]` message text has no think tags.

Disabled: `thinking_enabled=False` and content `"<think>hid</think>vis"` → no reasoning item, message is `vis`.

- [ ] **Step 2: Handler**

Add `thinking_enabled: bool = True`. Remap each delta through `surface_chunk` + `display_reasoning_mode`. Feed the sink. When opening function_call, `sink.finish()` first so reasoning/message close before `fc_` items (existing text-close logic). Non-stream: remap `message.reasoning` / `reasoning_content` / content, prepend reasoning item onto `output` before message/function_call.

- [ ] **Step 3: dispatch.py** pass `thinking_enabled=auth_data.get("thinking_enabled", True) if auth_data else True` into both tool handlers.

- [ ] **Step 4: Run** `python -m src.tools.tests.test_responses_handler_offline`

---

### Task 5: Live harness + verification

**Files:**
- Modify: `src/providers/tests/live_reasoning_surfaces.py`

Responses surface check today is `"<think>" in mer_c`. That is `/v1thinking` remapper, not HTTP Responses. Change the `responses` row to SEPARATE reasoning length (same as chat) **or** add `responses_http` that asserts `sep_r` and `"<think>" not in sep_c`. Keep MERGED as `v1thinking` if the JSON still wants a merged row.

```python
"responses": {"ok": (not upstream) or bool(sep_r), "reasoning_len": len(sep_r), "leaked_think": "<think>" in sep_c.lower()},
```

`merged_ok` stays for `/v1thinking` under key `v1thinking`.

- [ ] **Step 1: Run offline suite**

```
python -m src.routes.normal.tests.test_responses_reasoning_items_offline
python -m src.routes.normal.tests.test_responses_thinking_key_offline
python -m src.routes.normal.tests.test_chat_surface_offline
python -m src.tools.tests.test_responses_input_offline
python -m src.tools.tests.test_responses_handler_offline
python -m src.tools.tests.test_messages_thinking_offline
```

Expected: all PASS. Do not claim live-all-providers-pass.

## Out of scope

- Chat `reasoning_details[]` (OpenRouter structured blocks)
- `previous_response_id` persistence
- Fake `encrypted_content`
- Changing `/v1thinking` (still MERGED `<think>` in chat content)
- Deep-research raw SSE proxy

## Self-review

- Spec: emit reasoning items, no think tags in output_text, replay, tools, usage split, display toggle — each has a task.
- No encrypted_content / store.
- `display_reasoning_mode` vs `merged_reasoning_mode` names stay consistent: Responses uses SEPARATE/EXCLUDE.
- Converter imports `emit.py` only; `emit.py` has no converter import.
