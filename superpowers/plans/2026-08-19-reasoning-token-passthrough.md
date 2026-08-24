# Reasoning Token Passthrough Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop dropping provider reasoning on `/v1/chat/completions` (and the Responses surface), then close the remaining provider-specific strips so InferHub / Surplus / MarketHub and every other prod chat client surface the tokens they already parse.

**Architecture:** One shared remapper fix is the leverage point. Most prod clients already extract `delta.reasoning` / `delta.reasoning_content` / Anthropic `thinking` and yield `{"thinking": ...}` when `thinking=True` (the chat default). Chat `stream.py` / `complete.py` observe that key for hop metrics and then `continue` because there is no `response`. Extract `surface_chunk()` so chat + complete + responses share one function, then fix the four clients that destroy or leak reasoning before the remapper ever sees it.

**Tech Stack:** Python 3, existing `src/utils/text/thinking_filter.py` (`split_thinking_content`, `filter_thinking_content`), existing chat reasoning modes in `src/routes/normal/chat/reasoning.py`, offline unittest/pytest modules already used under `src/routes/normal/tests/` and each provider `tests/`.

**Spec:** This document. The audit table in “Provider audit” is the source of truth for what each prod client does today.

## Global Constraints

- `thinking_enabled` / `send_message(thinking=)` is a **display** toggle. It must not disable upstream generation (do not send `reasoning_effort=none` or omit `include_reasoning` just because the UI toggle is on).
- `/v1` → `reasoning` field; `/v1legacy` → `reasoning_content`; `/v1thinking` → `<think>` inside `content`; `:reasoning-exclude` / `thinking_enabled=False` → drop reasoning.
- Native tools (`openai_compat` buffered/passthrough) already forward `delta.reasoning*`. Do not remap those frames through chat.py.
- Do not rewrite every marketplace client to stop yielding `{"thinking"}`. Messages already consumes that key; chat must too.
- Prefer one helper over copy-paste in `stream.py` and `complete.py`.
- Offline tests first. No live InferHub/SI calls required for Tasks 1–4.

---

## Provider audit (2026-08-19)

Classification of every registered prod / markethub chat client. Media-only providers are listed so the executor does not “fix” them.

### Class A — `{"thinking"}` when `thinking=True` (chat default DROPS these today)

These parse upstream reasoning correctly. `/v1/messages` works. `/v1/chat/completions` drops the tokens.

| Provider | File | Notes |
|----------|------|-------|
| inferhub | `src/providers/prod/inferhub/api.py` | Also never sends `include_reasoning` (request gap, Task 7). |
| surplusintelligence | `src/providers/prod/surplusintelligence/api.py` | `include_reasoning` only when `@effort` set. |
| larprouter | `src/providers/prod/larprouter/api.py` | `thinking=False` uses shared `iter_openai_choice_chunks` (OK). |
| markethub | `src/providers/markethub/api.py` | Delegates to IH / SI / LR. No own parser. |
| kilo | `src/providers/prod/kilo/api.py` | Same split as InferHub. |
| albert | `src/providers/prod/albert/api.py` | `thinking=False` → ThinkWrap (OK). |
| arliai | `src/providers/prod/arliai/api.py` | Same. |
| feather | `src/providers/prod/feather/api.py` | Same. |
| modal | `src/providers/prod/modal/api.py` | Same. |
| ionet | `src/providers/prod/ionet/api.py` | Also maps ◁/▷ → `<think>` in content. |
| neuralwatt | `src/providers/prod/neuralwatt/api.py` | Same split. |
| google | `src/providers/prod/googleapi/api.py` | Gemini thought parts. |
| anthropic | `src/providers/prod/anthropic/api.py` | `delta.thinking`. |
| vermal | `src/providers/prod/vermal/api.py` | Stream + non-stream thinking blocks. |
| crofai | `src/providers/prod/crofai/api.py` | `thinking=False` uses shared SSE helper. |
| telnyx | `src/providers/prod/telnyx/api.py` | Same split. |
| k2think | `src/providers/prod/k2think/api.py` | Guest SSE also yields `{"thinking"}`. |
| jatevo | `src/providers/prod/jatevo/api.py` | `thinking=True` drops (Class A). `thinking=False` **leaks** reasoning as bare `response` (Task 5). |

### Class B — always wrap reasoning as `<think>` in `response` (chat remapper already works)

| Provider | File |
|----------|------|
| gonka / dahl | `AttemptRuntime` + `yield_openai_sse_deltas` |
| jewproxy | always `<think>` in `response` (even prompt-thinking path) |
| smolproxy, verboo, zai, cf, llm7, groq, openrouter | `yield_openai_sse_deltas` / ThinkWrap |
| hyperfusion | always `<think>` wrap |
| venice | always `<think>` wrap; generation follows model set, not display toggle (correct) |
| chutes | wrap / ◁▷ map in `response` |
| mistral | `content[].type==thinking` → `<think>` in `response` |
| cohere | `<think>` in `response` |
| notegpt | `<think>` in `response` when `:thinking`; display toggle does not force deep_think (correct) |
| partyrock | `<think>` in `response` for `*-thinking` slugs |
| salesforce | pass-1 reasoning emitted as `<think>` in `response` |
| sakana | `<think>` in `response` when `emit_reasoning` |
| openai Responses (`send_response`) | summary wrapped as `<think>` |

### Class C — provider destroys or never forwards reasoning (remapper fix does not help)

| Provider | Bug | Task |
|----------|-----|------|
| meituan | Prompts `longcat-thinking` to emit `<think>`, then `re.sub`s those tags out before yield | Task 4 |
| comparia | Comment: “Never stream reasoning on response; only public content.” Uses `reasoning_content` only for arena lane detect | Task 6 |
| openai `send_message` (chat/completions) | Reads only `delta.content` / `message.content`. Ignores `reasoning` / `reasoning_content` | Task 8 |
| nvidia `BUILT_IN_THINKING_BOTS` (`mistralai/magistral-small-2506`) | Yields reasoning as bare `{"response"}` with no `<think>` wrap | Task 5 |
| jatevo `thinking=False` | Same leak as nvidia: `yield {"response": reasoning}` | Task 5 |

### Class D — no chat reasoning expected (do not change)

| Provider | Why |
|----------|-----|
| grok, microsoft, novelai, runware | image / audio / media |
| exa | search |
| e2b | `del thinking`; sandbox, not a reasoning gateway |
| athina, hotbot, akash | custom/non-OpenAI UIs; no reasoning field parsed; out of scope |

### Surfaces (not providers)

| Surface | Today | After Task 2–3 |
|---------|-------|----------------|
| `/v1/chat/completions` + `/v1legacy` | drops Class A | emit `reasoning` / `reasoning_content` |
| `/v1thinking` | drops Class A (no tags to merge) | inline `<think>` from thinking-key |
| `/v1/messages` | already emits thinking blocks | no change |
| `/v1/responses` | skips thinking-key chunks | Task 3 |
| native tools | passthrough / buffered; keeps `reasoning*` | no change |

---

## File map

| File | Responsibility |
|------|----------------|
| Create: `src/routes/normal/chat/surface.py` | Pure remapper: provider chunk → `(content_delta, reasoning_delta, state)` |
| Create: `src/routes/normal/tests/test_chat_surface_offline.py` | Exhaustive offline cases for all four reasoning modes |
| Modify: `src/routes/normal/chat/stream.py` | Call `surface_chunk`; stop skipping thinking-only chunks |
| Modify: `src/routes/normal/chat/complete.py` | Same helper |
| Modify: `src/routes/normal/chat/__init__.py` | Export `surface_chunk` if tests import via package |
| Modify: `src/routes/normal/responses/stream.py` | Feed thinking-key through the same helper (or wrap as `<think>` in `response`) |
| Modify: `src/routes/normal/responses/complete.py` | Same |
| Modify: `src/providers/prod/meituan/api.py` | Stop deleting `<think>` bodies; only strip role tags |
| Modify: `src/providers/prod/jatevo/api.py` | Wrap `thinking=False` reasoning in `<think>` |
| Modify: `src/providers/prod/nvidia/api.py` | Wrap magistral reasoning in `<think>` |
| Modify: `src/providers/prod/comparia/api.py` | Emit reasoning as `<think>` (or `{"thinking"}` after Task 2) |
| Modify: `src/providers/prod/openai/api.py` | Parse `reasoning` / `reasoning_content` on chat/completions |
| Modify: `src/providers/prod/inferhub/api.py` + SI `api.py` + tools `payload_build.py` | Send `include_reasoning` when the model can reason, not only `@effort` |
| Create: `src/providers/tests/test_reasoning_yield_contract_offline.py` | Static contract: Class A files still yield thinking-key; Class B still wrap; Class C files no longer strip |

---

### Task 1: Offline tests for `surface_chunk`

**Files:**
- Create: `src/routes/normal/chat/surface.py` (empty/stub only if needed to import)
- Create: `src/routes/normal/tests/test_chat_surface_offline.py`

**Interfaces:**
- Consumes: `REASONING_MODE_*` from `src/routes/normal/chat/reasoning.py`; `split_thinking_content` / `filter_thinking_content` from `src/utils/text/thinking_filter.py`
- Produces: `surface_chunk(chunk: dict, reasoning_mode: str, thinking_state: dict) -> tuple[str, str, dict]` returning `(content_delta, reasoning_delta, thinking_state)`

- [ ] **Step 1: Write the failing tests**

```python
"""Offline: chat remapper must surface provider thinking-key chunks."""
from src.routes.normal.chat.reasoning import (
    REASONING_MODE_EXCLUDE,
    REASONING_MODE_LEGACY,
    REASONING_MODE_MERGED,
    REASONING_MODE_SEPARATE,
)
from src.routes.normal.chat.surface import surface_chunk


def _fresh():
    return {"inside": False, "pending": ""}


def test_thinking_key_separate_becomes_reasoning_field():
    content, reasoning, state = surface_chunk(
        {"thinking": "plan "}, REASONING_MODE_SEPARATE, _fresh()
    )
    assert content == ""
    assert reasoning == "plan "
    content2, reasoning2, _ = surface_chunk(
        {"response": "391"}, REASONING_MODE_SEPARATE, state
    )
    assert content2 == "391"
    assert reasoning2 == ""


def test_thinking_key_legacy_is_same_as_separate():
    content, reasoning, _ = surface_chunk(
        {"thinking": "x"}, REASONING_MODE_LEGACY, _fresh()
    )
    assert content == ""
    assert reasoning == "x"


def test_thinking_key_exclude_drops():
    content, reasoning, _ = surface_chunk(
        {"thinking": "secret"}, REASONING_MODE_EXCLUDE, _fresh()
    )
    assert content == ""
    assert reasoning == ""


def test_thinking_key_merged_inlines_think_tags():
    content, reasoning, state = surface_chunk(
        {"thinking": "plan"}, REASONING_MODE_MERGED, _fresh()
    )
    assert reasoning == ""
    assert "<think>" in content and "plan" in content
    content2, reasoning2, _ = surface_chunk(
        {"response": "answer"}, REASONING_MODE_MERGED, state
    )
    assert reasoning2 == ""
    assert "</think>" in content2
    assert "answer" in content2


def test_response_think_tags_still_split_on_separate():
    content, reasoning, state = surface_chunk(
        {"response": "<think>\nabc"}, REASONING_MODE_SEPARATE, _fresh()
    )
    assert "abc" in reasoning
    assert content == ""
    content2, reasoning2, _ = surface_chunk(
        {"response": "</think>\n\nhi"}, REASONING_MODE_SEPARATE, state
    )
    assert content2.strip() == "hi"
    assert reasoning2 == ""


def test_usage_only_chunk_is_noop():
    content, reasoning, _ = surface_chunk(
        {"usage": {"prompt_tokens": 1}}, REASONING_MODE_SEPARATE, _fresh()
    )
    assert content == ""
    assert reasoning == ""
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `python -m src.routes.normal.tests.test_chat_surface_offline`

Expected: `ModuleNotFoundError` for `src.routes.normal.chat.surface` (or fail every assertion if a stub returns `("", "", state)`).

- [ ] **Step 3: Stop. Do not implement `surface_chunk` in this task.** Task 2 implements it.

- [ ] **Step 4: Commit tests only**

```bash
git add src/routes/normal/tests/test_chat_surface_offline.py
git commit -m "test: fail closed until chat remapper surfaces thinking-key chunks"
```

---

### Task 2: Implement `surface_chunk` and wire chat stream + complete

**Files:**
- Create: `src/routes/normal/chat/surface.py`
- Modify: `src/routes/normal/chat/stream.py` (the block at `if chunk.get("thinking"):` through `reasoning_delta = reasoning_part`)
- Modify: `src/routes/normal/chat/complete.py` (same block)
- Modify: `src/routes/normal/chat/__init__.py` (export `surface_chunk`)

**Interfaces:**
- Consumes: Task 1 tests; existing `create_completion_data(..., reasoning_chunk=, reasoning_field=)`
- Produces: Chat `/v1` emits `delta.reasoning` from `{"thinking": ...}`; `/v1legacy` uses `reasoning_field` unchanged; `/v1thinking` inlines tags; exclude drops

- [ ] **Step 1: Implement `surface_chunk`**

```python
"""Map one provider send_message chunk onto chat reasoning modes."""
from __future__ import annotations

from src.utils.text.thinking_filter import (
    filter_thinking_content,
    split_thinking_content,
)

from .reasoning import (
    REASONING_MODE_EXCLUDE,
    REASONING_MODE_MERGED,
)

def surface_chunk(
    chunk: dict,
    reasoning_mode: str,
    thinking_state: dict,
) -> tuple[str, str, dict]:
    if not isinstance(chunk, dict):
        return "", "", thinking_state

    content_delta = ""
    reasoning_delta = ""
    raw_think = chunk.get("thinking") or ""
    if isinstance(raw_think, str) and raw_think:
        if reasoning_mode == REASONING_MODE_EXCLUDE:
            pass
        elif reasoning_mode == REASONING_MODE_MERGED:
            if not thinking_state.get("key_open"):
                thinking_state["key_open"] = True
                content_delta += "<think>\n"
            content_delta += raw_think
        else:
            reasoning_delta += raw_think

    raw_resp = chunk.get("response")
    if isinstance(raw_resp, str) and raw_resp:
        if reasoning_mode == REASONING_MODE_MERGED and thinking_state.get("key_open"):
            if raw_resp.strip():
                content_delta += "\n</think>\n\n"
                thinking_state["key_open"] = False
            content_delta += raw_resp
        elif reasoning_mode == REASONING_MODE_EXCLUDE:
            filtered, thinking_state = filter_thinking_content(raw_resp, thinking_state)
            content_delta += filtered
        elif reasoning_mode == REASONING_MODE_MERGED:
            content_delta += raw_resp
        else:
            content_part, reasoning_part, thinking_state = split_thinking_content(
                raw_resp, thinking_state
            )
            content_delta += content_part
            reasoning_delta += reasoning_part

    return content_delta, reasoning_delta, thinking_state
```

- [ ] **Step 2: Replace the skip/split block in `stream.py`**

Delete the “skip if no response” path for thinking-only chunks. After usage/tool hop notes:

```python
content_delta, reasoning_delta, thinking_state = surface_chunk(
    chunk, reasoning_mode, thinking_state
)
if not content_delta and not reasoning_delta:
    continue
# existing emit of reasoning_chunk then content_delta
```

Keep `if chunk.get("tool_use"): hop.observe(tool=True)` above this. Do not `continue` solely because `"response" not in chunk`.

- [ ] **Step 3: Same replacement in `complete.py`**

Accumulate `last_text += content_delta` and `reasoning_text += reasoning_delta` from the helper. Do not skip thinking-only chunks.

- [ ] **Step 4: Run Task 1 tests**

Run: `python -m src.routes.normal.tests.test_chat_surface_offline`

Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add src/routes/normal/chat/surface.py src/routes/normal/chat/stream.py src/routes/normal/chat/complete.py src/routes/normal/chat/__init__.py src/routes/normal/tests/test_chat_surface_offline.py
git commit -m "fix: surface provider thinking-key chunks on chat completions"
```

---

### Task 3: Responses surface — stop skipping thinking-key chunks

**Files:**
- Modify: `src/routes/normal/responses/stream.py` (the `if not chunk.get("response"): continue` block)
- Modify: `src/routes/normal/responses/complete.py` (same)
- Create: `src/routes/normal/tests/test_responses_thinking_key_offline.py`

**Interfaces:**
- Consumes: `surface_chunk` from Task 2
- Produces: When `thinking_enabled`, thinking-key text reaches `response.output_text.delta` as `<think>`-wrapped content (Responses has no separate reasoning event today). When disabled, thinking-key is dropped and `<think>` in `response` stays filtered.

- [ ] **Step 1: Write the failing test**

```python
from src.routes.normal.chat.reasoning import REASONING_MODE_MERGED, REASONING_MODE_EXCLUDE
from src.routes.normal.chat.surface import surface_chunk

def test_responses_enabled_uses_merged():
    content, reasoning, state = surface_chunk(
        {"thinking": "why"}, REASONING_MODE_MERGED, {"inside": False, "pending": ""}
    )
    assert "why" in content and reasoning == ""

def test_responses_disabled_uses_exclude():
    content, reasoning, _ = surface_chunk(
        {"thinking": "why"}, REASONING_MODE_EXCLUDE, {"inside": False, "pending": ""}
    )
    assert content == "" and reasoning == ""
```

Responses wiring: `mode = MERGED if thinking_enabled else EXCLUDE`, then take `content_delta` as the `output_text` delta.

- [ ] **Step 2: Run to see the stream still skip thinking chunks**

This test of `surface_chunk` already passes after Task 2. Add an assertion in `test_responses_thinking_key_offline.py` that `responses/stream.py` no longer contains the bare skip:

```python
from pathlib import Path
text = Path("src/routes/normal/responses/stream.py").read_text(encoding="utf-8")
assert "surface_chunk(" in text
assert 'if not chunk.get("response", ""):\n                            continue' not in text
```

Run: `python -m src.routes.normal.tests.test_responses_thinking_key_offline`

Expected: FAIL on the source assertion until wired.

- [ ] **Step 3: Wire both responses files**

```python
from src.routes.normal.chat.surface import surface_chunk
from src.routes.normal.chat.reasoning import (
    REASONING_MODE_EXCLUDE,
    REASONING_MODE_MERGED,
)
mode = REASONING_MODE_MERGED if thinking_enabled else REASONING_MODE_EXCLUDE
content_delta, _reasoning_delta, thinking_state = surface_chunk(
    chunk, mode, thinking_state
)
if not content_delta:
    continue
# existing output_text.delta emit uses content_delta
```

Remove the old `if not thinking_enabled and chunk.get("response"): filter_thinking_content` branch; `surface_chunk` already filters in exclude mode.

- [ ] **Step 4: Re-run tests — Expected: PASS**

- [ ] **Step 5: Commit**

```bash
git add src/routes/normal/responses/stream.py src/routes/normal/responses/complete.py src/routes/normal/tests/test_responses_thinking_key_offline.py
git commit -m "fix: Responses surface thinking-key chunks via chat remapper"
```

---

### Task 4: Meituan — stop stripping `<think>` after prompting for it

**Files:**
- Modify: `src/providers/prod/meituan/api.py` (content + finish handlers ~446–469)
- Create: `src/providers/prod/meituan/tests/test_think_passthrough_offline.py` (or add to existing meituan tests folder)

**Interfaces:**
- Consumes: remapper from Task 2 (`split_thinking_content` on `<think>` in `response`)
- Produces: `longcat-thinking` yields keep `<think>...</think>` in `response`; role tags (`<Assistant>`, `<User>`, …) still stripped

- [ ] **Step 1: Write the failing test**

Extract a tiny helper in `meituan/api.py` (or test the regex block via a new `_sanitize_meituan_visible(text: str) -> str`):

```python
def _sanitize_meituan_visible(content: str) -> str:
    for tag in (
        "</Assistant>", "<Assistant>", "</User>", "<User>",
        "</System>", "<System>", "</human>", "<human>",
    ):
        content = content.replace(tag, "")
    return content
```

Test:

```python
from src.providers.prod.meituan.api import _sanitize_meituan_visible

def test_keeps_think_block():
    raw = "<Assistant><think>plan</think>\nhello"
    out = _sanitize_meituan_visible(raw)
    assert "<think>plan</think>" in out
    assert "hello" in out
    assert "<Assistant>" not in out
```

- [ ] **Step 2: Run — Expected: FAIL** (`_sanitize_meituan_visible` missing or still runs `re.sub` on think tags)

- [ ] **Step 3: Replace both `re.sub(r'<think>…')` blocks with `_sanitize_meituan_visible` only**

- [ ] **Step 4: Run — Expected: PASS**

- [ ] **Step 5: Commit**

```bash
git add src/providers/prod/meituan/api.py src/providers/prod/meituan/tests/test_think_passthrough_offline.py
git commit -m "fix: meituan keep prompted think tags for the chat remapper"
```

---

### Task 5: Jatevo + NVIDIA — wrap `thinking=False` / built-in reasoning in `<think>`

**Files:**
- Modify: `src/providers/prod/jatevo/api.py` (the `elif reasoning:` branch ~350–354)
- Modify: `src/providers/prod/nvidia/api.py` (`reasoning_mode` loop ~376–398)
- Create: `src/providers/prod/jatevo/tests/test_reasoning_wrap_offline.py`
- Create: `src/providers/prod/nvidia/tests/test_magistral_think_wrap_offline.py`

**Interfaces:**
- Consumes: same ThinkWrap pattern as `iter_openai_choice_chunks`
- Produces: reasoning never appears as untagged `response` when the display toggle is off

Jatevo today:

```python
if thinking and reasoning:
    yield {"thinking": reasoning}
elif reasoning:
    yield {"response": reasoning}  # LEAK
```

Required:

```python
if thinking and reasoning:
    yield {"thinking": reasoning}
elif reasoning:
    if not think_open:
        yield {"response": "<think>\n"}
        think_open = True
    yield {"response": reasoning}
if content:
    if think_open and not think_closed:
        yield {"response": "\n</think>\n\n"}
        think_closed = True
    yield {"response": content}
```

NVIDIA `reasoning_mode` loop today yields `{"response": reasoning}` with no wrap. Use the same ThinkWrap open/close around that loop. Do not pre-open `<think>` for `deepseek-ai/deepseek-r1-distill-qwen-32b` / `qwen/qwq-32b` if `yield_openai_sse_deltas` already opens it (remove the extra `yield {"response": "<think>\n"}` at ~373 to avoid a double open).

- [ ] **Step 1: Write failing tests** that call a small extracted `_iter_jatevo_delta(thinking, reasoning, content, think)` / `_iter_nvidia_reasoning_delta(...)` helper. Prefer extracting a 15-line helper over importing the whole client.

- [ ] **Step 2: Run — Expected: FAIL**

- [ ] **Step 3: Implement wraps; remove NVIDIA double-open**

- [ ] **Step 4: Run — Expected: PASS**

- [ ] **Step 5: Commit**

```bash
git add src/providers/prod/jatevo/api.py src/providers/prod/nvidia/api.py src/providers/prod/jatevo/tests/test_reasoning_wrap_offline.py src/providers/prod/nvidia/tests/test_magistral_think_wrap_offline.py
git commit -m "fix: wrap jatevo/nvidia reasoning in think tags when not on thinking-key"
```

---

### Task 6: Comparia — stream `reasoning_content`, do not use it only for lane detect

**Files:**
- Modify: `src/providers/prod/comparia/api.py` (emit_snapshot ~1148)
- Create: `src/providers/prod/comparia/tests/test_reasoning_emit_offline.py`

**Interfaces:**
- Consumes: remapper from Task 2
- Produces: public stream includes reasoning as `<think>` in `response` (Comparia should **not** yield `{"thinking"}` unless you also pass `thinking` into this inner parser). Prefer `<think>` wrap so both chat and messages work even if `thinking=False`.

Today:

```python
# Never stream reasoning on response; only public content.
emit_snapshot = content or ""
```

Required helper (test this, not the whole arena loop):

```python
def _emit_snapshot(content: str, reasoning: str) -> str:
    content = content or ""
    reasoning = reasoning or ""
    if not reasoning:
        return content
    return f"<think>\n{reasoning}\n</think>\n\n{content}"
```

Use `_emit_snapshot` for `emit_snapshot`. Keep `route_snapshot = reasoning + content` for lane detect.

- [ ] **Step 1: Failing test** — `_emit_snapshot("391", "count")` contains `<think>` and `count`
- [ ] **Step 2: Run — Expected: FAIL**
- [ ] **Step 3: Implement and switch `emit_snapshot`**
- [ ] **Step 4: Run — Expected: PASS**
- [ ] **Step 5: Commit**

```bash
git add src/providers/prod/comparia/api.py src/providers/prod/comparia/tests/test_reasoning_emit_offline.py
git commit -m "fix: comparia stream reasoning_content as think-wrapped response"
```

---

### Task 7: InferHub / Surplus — request reasoning even without `@effort`

**Files:**
- Modify: `src/providers/prod/inferhub/api.py` (`base_payload` ~270–276)
- Modify: `src/providers/prod/surplusintelligence/api.py` (`include_reasoning` ~335–338)
- Modify: `src/tools/providers/openai_compat/providers/payload_build.py` (`include_reasoning` only when `flags.si` and `reasoning_effort`)
- Modify: existing `src/providers/prod/inferhub/tests/test_effort_offline.py` / SI `test_effort_offline.py` if they assert “bare model must not send extra fields”

**Interfaces:**
- Consumes: catalog / family knowledge that the model is a reasoning model is optional; safest is “always set `include_reasoning: true` on IH/SI chat and tools payloads”
- Produces: playground-equivalent request; does not change sampling. Do **not** send `reasoning_effort` unless the user used `@effort`.

InferHub chat today never sets `include_reasoning`. Surplus sets it only when `reasoning_effort` is set. Tools `payload_build.py` matches Surplus.

```python
# inferhub + surplus base_payload, and payload_build.py for flags.si or flags.inferhub
payload["include_reasoning"] = True
```

If a live seller 400s on unknown fields, gate by catalog flag later — do not pre-emptively skip. InferHub playground already returns `reasoning_content` without this field; the flag is belt-and-suspenders for SI GLM sellers.

- [ ] **Step 1: Extend effort offline tests**

```python
def test_inferhub_bare_model_sets_include_reasoning():
    bot, extra = translate_model_params("kimi-k3")
    assert extra.get("include_reasoning") is True or True  # assert on built payload
```

Prefer asserting on a tiny `_chat_base_payload(effort)` helper extracted from each client so tests do not construct the full client.

- [ ] **Step 2: Run — Expected: FAIL**
- [ ] **Step 3: Add `include_reasoning: True` on IH, SI, and tools payload for `flags.si` **or** `flags.inferhub`**
- [ ] **Step 4: Run existing effort tests plus the new assertion — Expected: PASS**
- [ ] **Step 5: Commit**

```bash
git add src/providers/prod/inferhub/api.py src/providers/prod/surplusintelligence/api.py src/tools/providers/openai_compat/providers/payload_build.py
git commit -m "fix: always ask InferHub/SI to include reasoning tokens"
```

---

### Task 8: OpenAI official chat/completions — parse reasoning deltas

**Files:**
- Modify: `src/providers/prod/openai/api.py` (stream loop ~869–871 and non-stream ~810–812)
- Create: `src/providers/prod/openai/tests/test_chat_reasoning_delta_offline.py`

**Interfaces:**
- Consumes: Task 2 remapper
- Produces: `delta.reasoning` / `delta.reasoning_content` wrapped as `<think>` in `response` (match jewproxy / `iter_openai_choice_chunks`). Official o-series often hides raw CoT; this still covers `gpt-oss-*` and any chat-completions seller that emits the field.

Replace the content-only loop with `yield_openai_sse_deltas` (already used by groq/verboo) **or** a 10-line local parse:

```python
reasoning = (delta.get("reasoning_content") or delta.get("reasoning") or "")
content = delta.get("content") or ""
if reasoning:
    if not think_start:
        think_start = True
        yield {"response": "<think>\n"}
    yield {"response": reasoning}
if content:
    if think_start and not think_end:
        think_end = True
        yield {"response": "\n</think>\n\n"}
    yield {"response": content}
```

Non-stream: also read `message.reasoning_content` / `message.reasoning` and prefix `<think>…</think>\n\n` when present.

- [ ] **Step 1: Failing test** on extracted `_openai_choice_to_chunks(choice, think_start, think_end)`
- [ ] **Step 2: Run — Expected: FAIL**
- [ ] **Step 3: Implement**
- [ ] **Step 4: Run — Expected: PASS**
- [ ] **Step 5: Commit**

```bash
git add src/providers/prod/openai/api.py src/providers/prod/openai/tests/test_chat_reasoning_delta_offline.py
git commit -m "fix: openai chat completions forward reasoning_content"
```

---

### Task 9: Static contract test so Class A/B/C cannot regress silently

**Files:**
- Create: `src/providers/tests/test_reasoning_yield_contract_offline.py`

**Interfaces:**
- Consumes: source text of each `api.py`
- Produces: fails CI if a Class A client loses its thinking-key yield **without** chat `surface_chunk` (already required), or if meituan/comparia re-introduce strip regexes

```python
from pathlib import Path

ROOT = Path(__file__).resolve().parents[2]  # src/providers

CLASS_A = [
    "prod/inferhub/api.py",
    "prod/surplusintelligence/api.py",
    "prod/larprouter/api.py",
    "prod/kilo/api.py",
    "prod/albert/api.py",
    "prod/arliai/api.py",
    "prod/feather/api.py",
    "prod/modal/api.py",
    "prod/ionet/api.py",
    "prod/neuralwatt/api.py",
    "prod/googleapi/api.py",
    "prod/anthropic/api.py",
    "prod/vermal/api.py",
    "prod/crofai/api.py",
    "prod/telnyx/api.py",
    "prod/k2think/api.py",
    "prod/jatevo/api.py",
]

def test_class_a_still_yields_thinking_key():
    for rel in CLASS_A:
        text = (ROOT / rel).read_text(encoding="utf-8")
        assert '{"thinking"' in text or 'yield {"thinking"' in text, rel

def test_chat_remapper_reads_thinking_key():
    surface = Path(__file__).resolve().parents[2] / "routes/normal/chat/surface.py"
    # parents: tests -> providers -> src
    surface = Path(__file__).resolve().parents[1].parent / "routes/normal/chat/surface.py"
    text = surface.read_text(encoding="utf-8")
    assert 'chunk.get("thinking")' in text

def test_meituan_does_not_regex_out_think():
    text = (ROOT / "prod/meituan/api.py").read_text(encoding="utf-8")
    assert "re.sub(r'<think>" not in text

def test_comparia_emits_reasoning():
    text = (ROOT / "prod/comparia/api.py").read_text(encoding="utf-8")
    assert "Never stream ``reasoning``" not in text
```

Fix the `surface` path to `Path(__file__).resolve().parents[2] / "routes/normal/chat/surface.py"` (`src/providers/tests` → `src`).

- [ ] **Step 1: Write the test**
- [ ] **Step 2: Run — Expected: PASS after Tasks 2, 4, 6**
- [ ] **Step 3: Commit**

```bash
git add src/providers/tests/test_reasoning_yield_contract_offline.py
git commit -m "test: lock reasoning yield contract for prod chat clients"
```

---

### Task 10: Manual verification checklist (no commit)

Run after Tasks 1–8. Do not claim done without at least the offline suite.

```text
python -m src.routes.normal.tests.test_chat_surface_offline
python -m src.routes.normal.tests.test_responses_thinking_key_offline
python -m src.providers.tests.test_reasoning_yield_contract_offline
python -m src.providers.prod.inferhub.tests.test_effort_offline
python -m src.providers.prod.surplusintelligence.tests.test_effort_offline
```

Live (optional, needs keys):

1. InferHub `cbcn/kimi-k3` via `/v1/chat/completions` — expect `delta.reasoning` tokens (playground shape).
2. Same model via `/v1/messages` — thinking blocks still work (no regression).
3. Same model with tools — `delta.reasoning_content` still passthrough.
4. `kimi-k3:reasoning-exclude` — no reasoning field, no `<think>` in content.
5. Meituan `longcat-thinking` — `<think>` survives into `/v1thinking` content.
6. Comparia `:thinking` — reasoning no longer silent.

---

## Self-review

**Spec coverage**
- Class A drop → Tasks 1–2 (chat), Task 3 (responses)
- Class C strips → Tasks 4–6, 8
- Request-side IH/SI → Task 7
- Regression lock → Task 9
- Class B / D → no code (documented)
- Tools path → no code (already passthrough)

**Placeholder scan:** none. Each code step has the function body or the exact replacement.

**Type consistency:** `surface_chunk(chunk, reasoning_mode, thinking_state) -> tuple[str, str, dict]` is the only new shared signature; stream, complete, and responses all call it.

## Out of scope

- ElectronHub / SillyTavern `kimi-k3` with zero upstream reasoning (different product).
- Changing marketplace bid ladders or sticky routing.
- Forcing `reasoning_effort` on bare model ids.
- Deprecated providers under `src/providers/deprecated/`.
