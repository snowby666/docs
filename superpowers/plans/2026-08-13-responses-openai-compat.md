# /v1/responses OpenAI Compatibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make Electron Hub `/v1/responses` accept the OpenAI Responses request shapes that agents actually send (SDK `input_text` parts, flat function tools, `function_call` / `function_call_output` round-trips) and forward Chat Completions-shaped payloads to every upstream, instead of dropping the conversation or sending Responses part types that Z.ai / Gonka / Venice reject.

**Architecture:** One dict-shaped internal representation at the route boundary. `ResponseRequest` must stop coercing `input` into `List[ResponseMessage]` (that is what makes the converter skip every item). `responses_input_to_chat_messages` becomes the single Responses→Chat adapter used by **both** the native-tools path and the non-tools `send_message` path. Helpers that already flatten `input_text` for logging (`__split_responses_content`) are no longer the source of truth for what is sent upstream.

**Tech Stack:** FastAPI / Pydantic v2, existing `src/tools/_converters.py`, `src/utils/openai_type.py`, `src/routes/normal/responses.py`, pytest offline tests.

## Global Constraints

- Do not change `/v1/chat/completions` behavior; chat already works (live control: glm-5.2 and gpt-oss-120b returned `tool_calls` for `calc`).
- Do not invent hosted-tool execution (`web_search`, `file_search`, computer use). Reject or drop with a clear 4xx when the request is hosted-tools-only on a non-OpenAI-fallback provider.
- Do not implement `previous_response_id` / `item_reference` store. Return 400 if those are the only input.
- Never leak upstream provider names/URLs in client error bodies. Mapping a known empty-input 400 to a 4xx is allowed; raw Z.ai JSON is not.
- Keep Responses **output** shape (`output` items, `function_call`, SSE `response.*` events). This plan is request-normalization plus empty-conversation guards.
- TDD: add failing offline tests first. Do not “fix while looking at live logs.”
- Commits: only when the user asks. Do not commit as part of a task unless they say so.

---

## Evidence (already reproduced — do not re-litigate)

Offline matrix (`tmp/deep_responses_0377_offline.py`): **26/33 shapes non-OK** after `ResponseRequest.model_validate` then `responses_input_to_chat_messages`.

Live (`127.0.0.1:7000`, `ek-proxy-` key, `glm-5.2` / `gpt-oss-120b`; `:dev` models 403 for non-DevPass keys):

| Shape | Result |
|---|---|
| `"input": "…"` no tools | 200, text |
| `[{role, content: "…"}]` no tools | 200, text |
| SDK `input_text` parts, no tools | 200, empty text; Z.ai `content[0].type type error` |
| Two `input_text` parts | same type error |
| Assistant history `output_text` | Z.ai `messages[1].content[0].type type error` |
| `item_reference` only | becomes empty user; Z.ai `prompt parameter was not received normally` |
| List + tools (Bug 1 curl) | 200 `status=failed` “The upstream request failed”; Z.ai `Input cannot be empty`; Venice `At least one message must have a role of "user"` |
| Same, **streaming** | SSE `response.error` with the same generic message |
| Nested Chat-format tools + list input | still empty (Pydantic skip, not tool schema) |
| `"input": "…"` + tools | 200, `function_call` `calc` `{"a":17,"b":23}` |
| Same, **streaming** | SSE `function_call` |
| `/v1/chat/completions` + tools | 200, `tool_calls` |
| Round-trip `function_call` + `function_call_output` | empty messages / generic fail |
| `tool_choice: "required"` | HTTP **400** (not 422): `Input should be 'auto' or 'none' (parameter: tool_choice.literal['auto','none'])` — `src/api.py` `RequestValidationError` handler always maps schema errors to 400 |
| `role: developer` / `type: "text"` / object `image_url` | HTTP 400 with a **misleading** union error: `Input should be a valid string (parameter: input.str)` because `Union[str, List[ResponseMessage]]` failed the list arm then reported the `str` arm |
| Stream `input_text` | SSE empty (`STREAM_EMPTY_TEXT`) |
| Stream `{role, content: string}` | live classifier also said empty — **do not trust** a naive `"text": ""` substring; Responses SSE always emits empty placeholders plus `response.completed`. Non-stream of the same body returned text. Verify streams by scanning `output_text.done` / `function_call` payloads |

`:dev` catalog (`glm-5.2:dev`, `kimi-k2.7-code:dev`, `gpt-oss-120b:dev`) is the same conversion code with `source` `zai` / `gonka` / `venice`. Fix is provider-agnostic.

### Exact Bug 2 mechanism (do not “fix” `__split_responses_content` and stop)

`__split_responses_content` **does** map `input_text` → `text`, but only on a **local copy** after `model_dump()`. Tokenization uses that copy (`text_messages`). The send path then does:

```python
messages = await helpers.__add_responses_tools_prompt(messages)  # original Pydantic list
response = {"bot": baseModel, "message": messages}
```

`__add_responses_tools_prompt` dumps the **original** `ResponseMessage` objects (still `type: input_text`) and also **strips** `role=tool` / `tool_calls`. Upstream Z.ai/Gonka therefore see Responses part types; Venice `_extract_text_and_images` ignores them. Patching Venice or the splitter alone will not fix the default provider branch.

### Exact Bug 1 mechanism

`normalize_input` keeps `{role, content}` dicts, then the field type `List[ResponseMessage]` rehydrates them into models. `responses_input_to_chat_messages` does `if not isinstance(item, dict): continue` → `chat_messages == []` while `tools` are still converted → Z.ai `Input cannot be empty`.

Independently, `normalize_input` drops any item with `type` other than `"message"` (`function_call`, `function_call_output`, `reasoning`, `item_reference`). If everything is dropped it returns `""`, which becomes a user message with empty content (live `item_reference` → Z.ai `The prompt parameter was not received normally.`).

---

## File map

| File | Responsibility |
|---|---|
| `src/tools/_converters.py` | Coerce Pydantic→dict; convert all Responses input items and content parts to Chat Completions messages; stringify tool outputs; do not flatten a lone image part to `""`. |
| `src/utils/openai_type.py` | Widen `input` and content-part types; stop dropping `function_call*`; accept `developer`, `tool_choice: required`, `type: text`, object `image_url`. |
| `src/routes/normal/responses.py` | Convert once; send **chat** messages to `send_message`; 400 if conversion yields no user/tool/assistant content; pass dumped dicts into native tools. |
| `src/utils/helpers.py` | Align `__validate_responses_format` / `__split_responses_content` with the same part aliases; do not use `__add_responses_tools_prompt` on the main send path (it strips `tool_calls`). |
| `src/tools/responses_handler.py` | Guard empty `chat_messages` before upstream; map empty-input upstream 400 to 4xx-shaped Responses error (still HTTP 200 only if we keep today’s envelope — prefer `status=failed` with a **specific** message, not “upstream failed”). |
| `src/tools/tests/test_responses_input_offline.py` | Parametrized offline contract tests (HTTP-path = `model_validate` then convert). |
| `src/utils/tests/test_response_request_schema_offline.py` | Schema accept/reject cases. |
| `src/providers/prod/venice/api.py` | Optional defense: treat `input_text` / `output_text` as text in `_extract_text_and_images`. Only if a test still fails after the route fix. |

---

## Target conversion contract

After `ResponseRequest` parsing, `responses_input_to_chat_messages(req.input, req.instructions)` MUST produce Chat Completions messages.

### Input items (keep)

- `{role, content}` or `{type:"message", role, content}` (optional `id`/`status` ignored)
- `{type:"function_call", call_id, name, arguments}`
- `{type:"function_call_output", call_id, output}` where `output` is a string **or** an array of `{type:input_text,text}` parts
- Consecutive `function_call` items → one assistant message with `tool_calls`
- Bare strings in the list → user messages
- `instructions` → leading `role: system`

### Content parts → Chat

| Responses | Chat Completions |
|---|---|
| `input_text` / `output_text` / `text` / `summary_text` | `{type:"text", text}` |
| `input_image` (`image_url` string **or** `{url}`) / `computer_screenshot` | `{type:"image_url", image_url:{url}}` |
| `input_file` | skip (no chat equivalent); if that was the only part, 400 |
| unknown | skip, do not `str(part)` into fake text |

Single text part may flatten to a string. A single **image** part must remain a one-element list (today it becomes `""` via `parts[0].get("text")`).

### Roles

- `developer` → `system` for Chat Completions (compat providers reject `developer`)
- `user` / `assistant` / `system` unchanged
- `function_call_output` → `role: tool`, `tool_call_id=call_id`

### Tools / tool_choice

- Flat Responses `{type:function, name, parameters}` → nested Chat `{type:function, function:{…}}` (already implemented)
- Nested Chat tools passthrough (already implemented)
- `tool_choice`: `"auto"` \| `"none"` \| `"required"` \| `{type:function, name}` \| `{type:function, function:{name}}` (last already implemented)
- Hosted `{type:web_search}` etc.: drop from native Chat payload; if **no** function tools remain, HTTP 400 `param=tools`

### Reject (400), do not coerce to empty user

- `item_reference` as the only remaining item
- `previous_response_id` set with empty `input` (unsupported store)
- Conversion result empty (no messages, or only empty-string user)
- `normalize_input` today returns `""` when everything is stripped — **delete that fallback**

---

### Task 1: Failing offline contract tests

**Files:**
- Create: `src/tools/tests/test_responses_input_offline.py`
- Create: `src/utils/tests/test_response_request_schema_offline.py`

**Interfaces:**
- Consumes: `ResponseRequest`, `responses_input_to_chat_messages`, `responses_tools_to_chat`, `responses_tool_choice_to_chat`
- Produces: pytest module that the later tasks must turn green

- [ ] **Step 1: Write schema tests**

```python
# src/utils/tests/test_response_request_schema_offline.py
import pytest
from pydantic import ValidationError
from src.utils.openai_type import ResponseRequest

CALC = [{"type": "function", "name": "calc", "parameters": {"type": "object", "properties": {}}}]

def test_accepts_sdk_input_text():
    req = ResponseRequest.model_validate({
        "model": "x",
        "input": [{"type": "message", "role": "user",
                   "content": [{"type": "input_text", "text": "hi"}]}],
    })
    item = req.input[0]
    data = item.model_dump() if hasattr(item, "model_dump") else item
    assert isinstance(data, dict) or hasattr(item, "role")

def test_keeps_function_call_items():
    req = ResponseRequest.model_validate({
        "model": "x",
        "input": [
            {"type": "message", "role": "user", "content": "hi"},
            {"type": "function_call", "call_id": "c1", "name": "calc", "arguments": "{}"},
            {"type": "function_call_output", "call_id": "c1", "output": "1"},
        ],
        "tools": CALC,
    })
    raw = [x if isinstance(x, dict) else x.model_dump() for x in req.input]
    types = [x.get("type") or "message" for x in raw]
    assert "function_call" in types
    assert "function_call_output" in types

def test_tool_choice_required_accepted():
    req = ResponseRequest.model_validate({
        "model": "x", "input": "hi", "tools": CALC, "tool_choice": "required",
    })
    assert req.tool_choice == "required"

def test_developer_role_accepted():
    ResponseRequest.model_validate({
        "model": "x", "input": [{"role": "developer", "content": "sys"}],
    })

def test_type_text_alias_accepted():
    ResponseRequest.model_validate({
        "model": "x",
        "input": [{"role": "user", "content": [{"type": "text", "text": "hi"}]}],
    })

def test_image_url_object_accepted():
    ResponseRequest.model_validate({
        "model": "x",
        "input": [{"role": "user", "content": [
            {"type": "input_image", "image_url": {"url": "https://example.com/a.png"}},
        ]}],
    })

def test_item_reference_only_rejected():
    with pytest.raises(ValidationError):
        ResponseRequest.model_validate({
            "model": "x", "input": [{"type": "item_reference", "id": "msg_x"}],
        })
```

- [ ] **Step 2: Write converter HTTP-path tests**

HTTP-path means: parse with `ResponseRequest`, then convert `req.input` (exactly what `responses_handler` does).

```python
# src/tools/tests/test_responses_input_offline.py
import pytest
from src.utils.openai_type import ResponseRequest
from src.tools._converters import (
    responses_input_to_chat_messages,
    responses_tools_to_chat,
    responses_tool_choice_to_chat,
)

def _http_convert(payload):
    req = ResponseRequest.model_validate(payload)
    return responses_input_to_chat_messages(req.input, req.instructions)

def test_role_string_survives_pydantic():
    msgs = _http_convert({"model": "x", "input": [{"role": "user", "content": "hello"}]})
    assert msgs == [{"role": "user", "content": "hello"}]

def test_sdk_input_text_becomes_chat_text():
    msgs = _http_convert({"model": "x", "input": [{
        "type": "message", "role": "user",
        "content": [{"type": "input_text", "text": "hello"}],
    }]})
    assert msgs[0]["role"] == "user"
    assert msgs[0]["content"] == "hello" or msgs[0]["content"] == [{"type": "text", "text": "hello"}]

def test_two_text_parts_keep_both():
    msgs = _http_convert({"model": "x", "input": [{"role": "user", "content": [
        {"type": "input_text", "text": "hello "},
        {"type": "input_text", "text": "world"},
    ]}]})
    c = msgs[0]["content"]
    if isinstance(c, str):
        assert "hello" in c and "world" in c
    else:
        texts = [p["text"] for p in c]
        assert texts == ["hello ", "world"]

def test_roundtrip_function_call_and_output():
    msgs = _http_convert({
        "model": "x",
        "input": [
            {"type": "message", "role": "user",
             "content": [{"type": "input_text", "text": "17*23"}]},
            {"type": "function_call", "call_id": "call_1", "name": "calc",
             "arguments": '{"a":17,"b":23}'},
            {"type": "function_call_output", "call_id": "call_1", "output": "391"},
        ],
    })
    assert msgs[0]["role"] == "user"
    assert msgs[1]["role"] == "assistant"
    assert msgs[1]["tool_calls"][0]["id"] == "call_1"
    assert msgs[1]["tool_calls"][0]["function"]["name"] == "calc"
    assert msgs[2] == {"role": "tool", "tool_call_id": "call_1", "content": "391"}

def test_function_call_output_parts_stringify():
    msgs = _http_convert({
        "model": "x",
        "input": [
            {"role": "user", "content": "q"},
            {"type": "function_call", "call_id": "c1", "name": "calc", "arguments": "{}"},
            {"type": "function_call_output", "call_id": "c1",
             "output": [{"type": "input_text", "text": "391"}]},
        ],
    })
    assert msgs[-1]["content"] == "391"

def test_parallel_function_calls_one_assistant():
    msgs = _http_convert({
        "model": "x",
        "input": [
            {"role": "user", "content": "both"},
            {"type": "function_call", "call_id": "c1", "name": "calc", "arguments": '{"a":1}'},
            {"type": "function_call", "call_id": "c2", "name": "calc", "arguments": '{"a":2}'},
            {"type": "function_call_output", "call_id": "c1", "output": "1"},
            {"type": "function_call_output", "call_id": "c2", "output": "2"},
        ],
    })
    assert len(msgs[1]["tool_calls"]) == 2
    assert [m["role"] for m in msgs] == ["user", "assistant", "tool", "tool"]

def test_developer_maps_to_system():
    msgs = _http_convert({"model": "x", "input": [
        {"role": "developer", "content": "sys"},
        {"role": "user", "content": "hi"},
    ]})
    assert msgs[0] == {"role": "system", "content": "sys"}

def test_image_only_not_empty_string():
    msgs = _http_convert({"model": "x", "input": [{"role": "user", "content": [
        {"type": "input_image", "image_url": "https://example.com/a.png"},
    ]}]})
    c = msgs[0]["content"]
    assert c != ""
    assert isinstance(c, list)
    assert c[0]["type"] == "image_url"
    assert c[0]["image_url"]["url"] == "https://example.com/a.png"

def test_image_url_object():
    msgs = _http_convert({"model": "x", "input": [{"role": "user", "content": [
        {"type": "input_text", "text": "what"},
        {"type": "input_image", "image_url": {"url": "https://example.com/a.png"}},
    ]}]})
    imgs = [p for p in msgs[0]["content"] if isinstance(p, dict) and p.get("type") == "image_url"]
    assert imgs[0]["image_url"]["url"] == "https://example.com/a.png"

def test_instructions_prepend_system():
    msgs = _http_convert({"model": "x", "input": "hi", "instructions": "be brief"})
    assert msgs[0] == {"role": "system", "content": "be brief"}
    assert msgs[1]["content"] == "hi"

def test_tools_flat_and_nested():
    flat = responses_tools_to_chat([{"type": "function", "name": "calc", "parameters": {"type": "object"}}])
    nested = responses_tools_to_chat([{
        "type": "function", "function": {"name": "calc", "parameters": {"type": "object"}},
    }])
    assert flat[0]["function"]["name"] == "calc"
    assert nested[0]["function"]["name"] == "calc"

def test_tool_choice_required_and_forced():
    assert responses_tool_choice_to_chat("required") == "required"
    assert responses_tool_choice_to_chat({"type": "function", "name": "calc"}) == {
        "type": "function", "function": {"name": "calc"},
    }

def test_list_of_raw_strings():
    msgs = _http_convert({"model": "x", "input": ["hello", "there"]})
    assert [m["content"] for m in msgs] == ["hello", "there"]

def test_item_reference_mixed_keeps_user():
    msgs = _http_convert({"model": "x", "input": [
        {"type": "item_reference", "id": "msg_x"},
        {"role": "user", "content": "hello"},
    ]})
    assert msgs == [{"role": "user", "content": "hello"}]

def test_reasoning_item_skipped_keeps_user():
    msgs = _http_convert({"model": "x", "input": [
        {"type": "reasoning", "id": "rs_1", "summary": []},
        {"role": "user", "content": "hello"},
    ]})
    assert msgs[-1] == {"role": "user", "content": "hello"}

def test_instructions_not_duplicated():
    msgs = _http_convert({"model": "x", "input": "hi", "instructions": "be brief"})
    systems = [m for m in msgs if m["role"] == "system"]
    assert len(systems) == 1
    assert systems[0]["content"] == "be brief"

def test_assistant_output_text_history():
    msgs = _http_convert({"model": "x", "input": [
        {"role": "user", "content": "Say HI"},
        {"type": "message", "role": "assistant",
         "content": [{"type": "output_text", "text": "HI", "annotations": []}]},
        {"role": "user", "content": "Now PONG"},
    ]})
    assert [m["role"] for m in msgs] == ["user", "assistant", "user"]
    a = msgs[1]["content"]
    text = a if isinstance(a, str) else a[0]["text"]
    assert text == "HI"

def test_previous_response_id_does_not_strip_input():
    msgs = _http_convert({
        "model": "x", "previous_response_id": "resp_abc",
        "input": [{"role": "user", "content": "hello"}],
    })
    assert msgs == [{"role": "user", "content": "hello"}]

def test_hosted_tools_dropped_functions_kept():
    chat = responses_tools_to_chat([
        {"type": "web_search"},
        {"type": "function", "name": "calc", "parameters": {"type": "object"}},
    ])
    assert len(chat) == 1
    assert chat[0]["function"]["name"] == "calc"

def test_hosted_tools_only_empty():
    assert responses_tools_to_chat([{"type": "web_search"}]) == []
```

- [ ] **Step 3: Run tests — they must fail**

```bash
python -m pytest src/utils/tests/test_response_request_schema_offline.py src/tools/tests/test_responses_input_offline.py -v
```

Expected: FAIL on `test_keeps_function_call_items`, `test_role_string_survives_pydantic`, `test_sdk_input_text_becomes_chat_text`, `test_tool_choice_required_accepted`, `test_developer_role_accepted`, `test_type_text_alias_accepted`, `test_image_url_object_accepted`, `test_item_reference_only_rejected`, `test_image_only_not_empty_string`, `test_roundtrip_function_call_and_output`.

---

### Task 2: Widen `ResponseRequest` so items stay dicts and typed items survive

**Files:**
- Modify: `src/utils/openai_type.py` (`InputTextContentPart`, `InputImageContentPart`, `ResponseMessage`, `ResponseRequest`)

**Interfaces:**
- Consumes: Task 1 tests
- Produces: `req.input` is `str` or `list[dict]` (never a list of `ResponseMessage` objects). Typed items `function_call` / `function_call_output` remain in the list.

- [ ] **Step 1: Content-part aliases**

Replace `InputTextContentPart` / `InputImageContentPart` / `ResponseMessage` with extra-ignore models that accept SDK extras (`id`, `status`, `annotations`):

```python
class InputTextContentPart(BaseModel):
    model_config = ConfigDict(extra="ignore")
    type: Literal["input_text", "text", "output_text", "summary_text"] = "input_text"
    text: str

class InputImageContentPart(BaseModel):
    model_config = ConfigDict(extra="ignore")
    type: Literal["input_image", "computer_screenshot"] = "input_image"
    image_url: Union[str, dict]

class ResponseMessage(BaseModel):
    model_config = ConfigDict(extra="ignore")
    type: Optional[Literal["message"]] = None
    role: Literal["user", "assistant", "system", "developer"]
    content: Union[str, List[Union[InputContentPart, OutputContentPart]]]
```

Add:

```python
class FunctionCallInputItem(BaseModel):
    model_config = ConfigDict(extra="ignore")
    type: Literal["function_call"]
    call_id: str
    name: str
    arguments: str = "{}"
    id: Optional[str] = None
    status: Optional[str] = None

class FunctionCallOutputInputItem(BaseModel):
    model_config = ConfigDict(extra="ignore")
    type: Literal["function_call_output"]
    call_id: str
    output: Union[str, List[Any]]
    id: Optional[str] = None
    status: Optional[str] = None
```

- [ ] **Step 2: Change `input` type and `normalize_input`**

**Do not** type `input` as `List[ResponseMessage]`. That is Bug 1. Use:

```python
input: Union[str, List[Any]]
```

Rewrite `normalize_input`:

- If `v` is not a list, return `v`.
- For each element: `model_dump` if needed.
- If `str` → `{ "role": "user", "content": v }`.
- If `type` in `("function_call", "function_call_output")` → **keep** (do not `continue`).
- If `type` in `("item_reference", "reasoning", "computer_call", "computer_call_output")` → skip (counted). Mixed with a real user/assistant/tool item is fine.
- If `type` in `(None, "message")` or `role` in user/assistant/system/developer → keep as message dict. Preserve `content` (including part arrays). Do **not** require dropping `type` — the converter accepts both. Ignore extra keys (`id`, `status`).
- If `content is None` and there are no `tool_calls`, skip that item.
- If the kept list is empty → **raise `ValueError("Invalid input format for responses API.")`**. FastAPI/`RequestValidationError` will surface this as HTTP **400** (see `src/api.py`), not 422. Never return `""`.

- [ ] **Step 3: `tool_choice`**

```python
tool_choice: Optional[Union[Literal["auto", "none", "required"], Dict[str, Any]]] = "auto"
```

- [ ] **Step 4: Re-run schema tests**

```bash
python -m pytest src/utils/tests/test_response_request_schema_offline.py -v
```

Expected: schema tests PASS. Converter tests still FAIL until Task 3 if any leftover Pydantic objects exist — they should not, because `List[Any]` keeps dicts.

---

### Task 3: Harden `responses_input_to_chat_messages`

**Files:**
- Modify: `src/tools/_converters.py`

**Interfaces:**
- Consumes: dict-or-model items from Task 2
- Produces: Chat Completions `messages` matching Task 1 converter tests

- [ ] **Step 1: Add helpers at the top of the Responses→Chat section**

```python
def _dump(item: Any) -> Any:
    if hasattr(item, "model_dump"):
        return item.model_dump()
    return item

def _content_parts_to_chat(content: Any) -> Any:
    if isinstance(content, str) or content is None:
        return content if content is not None else ""
    if not isinstance(content, list):
        return str(content)
    parts = []
    for part in content:
        part = _dump(part)
        if isinstance(part, str):
            parts.append({"type": "text", "text": part})
            continue
        if not isinstance(part, dict):
            continue
        ptype = part.get("type") or ""
        if ptype in ("input_text", "output_text", "text", "summary_text"):
            parts.append({"type": "text", "text": part.get("text") or ""})
        elif ptype in ("input_image", "computer_screenshot", "image_url"):
            url = part.get("image_url") or part.get("image") or ""
            if isinstance(url, dict):
                url = url.get("url") or ""
            if url:
                parts.append({"type": "image_url", "image_url": {"url": url}})
        # input_file / unknown: skip
    if not parts:
        return ""
    if len(parts) == 1 and parts[0].get("type") == "text":
        return parts[0].get("text", "")
    return parts

def _tool_output_to_text(output: Any) -> str:
    if output is None:
        return ""
    if isinstance(output, str):
        return output
    if isinstance(output, list):
        bits = []
        for p in output:
            p = _dump(p)
            if isinstance(p, str):
                bits.append(p)
            elif isinstance(p, dict) and p.get("text"):
                bits.append(str(p["text"]))
        return "\n".join(bits)
    return str(output)
```

- [ ] **Step 2: Coerce every list item with `_dump` before `isinstance(..., dict)`**

Map `role == "developer"` to `"system"`.

Use `_content_parts_to_chat` instead of the inline part loop.

Use `_tool_output_to_text` for `function_call_output`.

Keep grouping consecutive `function_call` items. `tool_calls[].id` must be `call_id` (not item `id`) so round-trips match `tool_call_id`.

- [ ] **Step 3: Run converter tests**

```bash
python -m pytest src/tools/tests/test_responses_input_offline.py -v
```

Expected: PASS.

---

### Task 4: Route always sends Chat Completions messages

**Files:**
- Modify: `src/routes/normal/responses.py` (input handling ~275–390 and tools branch ~419–529)
- Modify: `src/utils/helpers.py` (`__validate_responses_format` roles + part types; do not call `__add_responses_tools_prompt` on the payload sent to `send_message`)

**Interfaces:**
- Consumes: converter from Task 3
- Produces: `response["message"]` is Chat Completions messages (`type: text` / string), including when `tools` is absent

- [ ] **Step 1: Convert immediately after reading `data.input`**

Keep a copy of the original Responses-shaped list **before** converting (needed for the OpenAI fallback):

```python
from src.tools._converters import (
    responses_input_to_chat_messages,
    responses_tools_to_chat,
    responses_tool_choice_to_chat,
)

responses_input = input_data  # str or list[dict], already dumped by Task 2
chat_messages = responses_input_to_chat_messages(responses_input, instructions)
if not chat_messages or all(
    (not m.get("content") and not m.get("tool_calls") and m.get("role") != "tool")
    for m in chat_messages
):
    raise HTTPException(
        detail={"error": {
            "message": "Invalid input format for responses API.",
            "type": "invalid_request_error", "param": "input", "code": 400,
        }},
        status_code=400,
    )
```

**Do not** run the existing `if instructions: text_messages.insert(0, …)` afterwards. The converter already prepends `instructions`. Double-insert would send two system messages.

**Do not** call `__validate_responses_format` on the raw Responses list after this change — that helper rejects `type: "text"` and `role: "developer"`. Either delete the call (converter + empty-output 400 is enough) or rewrite it to accept Chat Completions messages (`role` in user/assistant/system/tool, content string or `text`/`image_url` parts). Prefer delete-the-call.

Use `chat_messages` for tokenization and as `response["message"]` for **every** provider branch (`TAGS_PROVIDERS`, `hotbot`/`together`, and the default `else`).

Extract image URLs from converted chat `image_url` parts. A cheap helper:

```python
def _chat_image_urls(messages: list) -> list[str]:
    urls = []
    for m in messages:
        c = m.get("content")
        if isinstance(c, list):
            for p in c:
                if isinstance(p, dict) and p.get("type") == "image_url":
                    u = p.get("image_url")
                    if isinstance(u, dict):
                        u = u.get("url")
                    if u:
                        urls.append(u)
    return urls
```

- [ ] **Step 2: Native tools branch**

Today both `native_tool_responses_stream` and `native_tool_responses_non_stream` call `responses_input_to_chat_messages(input_data, instructions)` internally (lines 74 and 536). After Task 4 the route already converted.

Do **not** pass Chat messages back through that converter: a Chat `role: tool` item has no Responses `type` and would be skipped; `instructions` would be prepended a second time.

Add a `messages: list` parameter and drop the internal convert:

```python
async def native_tool_responses_stream(
    display_model: str,
    base_model: str,
    messages: list,          # Chat Completions messages; already converted
    tools: list,
    ...
    # delete input_data and do not call responses_input_to_chat_messages
):
    chat_messages = messages
    chat_tools = responses_tools_to_chat(tools)
    mapped_tool_choice = responses_tool_choice_to_chat(tool_choice)
```

Same for `_non_stream`. Call sites in `responses.py`:

```python
native_tool_responses_stream(..., messages=chat_messages, tools=tools, ...)
```

Keep `instructions` only for the Responses **output** envelope (the `instructions` field on the returned `response` object), not for a second convert.

- [ ] **Step 3: Stop `__add_responses_tools_prompt` on the send path**

That helper (1) `model_dump()`s original Pydantic (Bug 2) and (2) **removes** `role=tool` and `tool_calls`. After Task 2, a no-`tools` replay of a prior function call would be destroyed.

Replace the three send branches with one:

```python
response = {"bot": baseModel, "message": chat_messages}
```

`hotbot`/`together` still receive `chat_messages`. They never get native tools; string content is enough. Do not run the strip helper on them either.

Leave `__add_responses_tools_prompt` in helpers.py unused rather than deleting it in this plan (call-site grep first; if nothing else calls it, deleting is fine).

- [ ] **Step 4: Validator / splitter**

If the `__validate_responses_format` call was deleted in Step 1, skip this. If kept, allow `developer` (or expect it already mapped to `system`), part types `text` / `image_url`, and `role: tool`.

Do **not** rely on `__split_responses_content` to fix part types for upstream. It only mutates a copy. After this task it is only useful for collecting `image_urls` for the vision-provider swap + token estimate — and `_chat_image_urls(chat_messages)` replaces that. Prefer stop calling `__split_responses_content` from this route.

- [ ] **Step 5: Fallback OpenAI Responses proxy** (`supports_native_tools` false)

`payload["input"]` must be **original dumped dicts** (Responses shape, including `function_call*` and `input_text`), not Chat messages. OpenAI’s Responses API wants Responses items.

```python
payload["input"] = responses_input  # the copy from Step 1
```

Do not pass `chat_messages` here. `instructions` stay on `payload["instructions"]` as today.

- [ ] **Step 6: Hosted-tools-only 400**

`tools: []` is falsy — leave that on the non-tools path (live: `n.empty_tools_array` already works for string content).

```python
if tools:
    chat_tools = responses_tools_to_chat(tools)
    if not chat_tools:
        raise HTTPException(
            detail={"error": {
                "message": "Hosted tools (web_search, file_search, computer use) are not supported on this model. Pass function tools with type=function.",
                "type": "invalid_request_error", "param": "tools", "code": 400,
            }},
            status_code=400,
        )
```

Mixed function + hosted: `responses_tools_to_chat` already drops non-function entries; keep going with the function tools. `tool_choice: {type: web_search}` already maps to `"auto"` in `responses_tool_choice_to_chat` — leave that.

- [ ] **Step 7: Run all new offline tests plus existing vermal/responses tests that still apply**

```bash
python -m pytest src/tools/tests/test_responses_input_offline.py src/utils/tests/test_response_request_schema_offline.py -v
```

Expected: PASS.

---

### Task 5: Empty-conversation errors must not look like success

**Files:**
- Modify: `src/tools/responses_handler.py` (non-stream `error` envelope ~568–585; stream `saw_upstream_error` ~174–183 and ~445–455)
- Modify: `src/routes/normal/responses.py` (Task 4 already 400s on empty convert)

**Interfaces:**
- Consumes: empty `chat_messages` never reaches this if Task 4 is correct
- Produces: if upstream still returns empty-input 400, client sees `invalid_request_error` / a specific message, not “The upstream request failed. Please try again.”

- [ ] **Step 1: Classify empty-input upstream errors**

In `native_tool_responses_non_stream`, if `openai_response` has `error` and the logged upstream text contains `Input cannot be empty` or `At least one message must have a role of "user"` or `messages parameter is illegal`, return:

```python
{
    "id": response_id,
    "object": "response",
    "status": "failed",
    "error": {
        "type": "invalid_request_error",
        "message": "The request input was empty after conversion. Check that messages use Responses content parts (input_text) or plain strings, and that function_call items are included in input.",
    },
    "output": [],
}
```

Do not copy the Z.ai/Venice sentence into the body.

Stream path: same message on `response.error`.

- [ ] **Step 2: Z.ai `type type error` on leftover Responses parts**

If that string appears, treat as server_error with message: `Message content parts must use Chat Completions types (text, image_url).` That should be unreachable after Task 4; keep as belt-and-suspenders.

---

### Task 6: Defense in depth for Venice part types (only if Task 4 tests need it)

**Files:**
- Modify: `src/providers/prod/venice/api.py` `_extract_text_and_images` (~323)

**Interfaces:**
- Consumes: content parts that still have `input_text` / `output_text`
- Produces: those parts contribute text

- [ ] **Step 1: Widen the text-type check**

```python
if itype in ("text", "input_text", "output_text", "summary_text") or (
    itype is None and "text" in item
):
    text_parts.append(item.get("text", ""))
```

Skip this task if Task 4’s route conversion is proven by a unit test that `response["message"]` never contains `input_text`. Prefer proving that in Task 4 rather than patching every provider.

---

### Task 7: Live verification (after offline green)

**Files:** none (use `tmp/deep_responses_0377_live.py` against `python main.py -host 127.0.0.1:7000 -dev`)

Use a **non-`:dev`** model the `ek-proxy-` key can call (`glm-5.2` or `gpt-oss-120b`). `:dev` is DevPass-only.

- [ ] **Step 1: Space requests** (Z.ai 429’d the dense matrix). One at a time, or sleep 2s.

Must pass (set `PYTHONIOENCODING=utf-8` — the previous live run crashed on a `👋` in a model string under cp1252):

1. List + `input_text` + no tools → non-empty `output_text` (not 0 tokens)
2. List + string content + tools → `function_call` `calc` or text (not `status=failed`)
3. SDK `input_text` + tools → same
4. Top-level string + tools → still works (regression)
5. Stream list + tools → look for `function_call` in SSE, **not** a `"text": ""` substring
6. Stream `input_text` no tools → `output_text.done` with non-empty `text`
7. Round-trip `function_call` + `function_call_output` → model answers 391, not empty
8. `tool_choice: "required"` → HTTP 200 (not 400 schema error)
9. `role: developer` → 200 text (not `parameter: input.str`)
10. Assistant `output_text` history + follow-up user string → 200 text (not type error)
11. Hosted `web_search` only → 400 `param=tools`
12. `/v1/chat/completions` tools still works (regression)
13. `item_reference` only → 400, not empty 200

- [ ] **Step 2: Confirm server logs** show no `Input cannot be empty` and no `content[0].type type error` for cases 1–10.

---

## Edge-case checklist (every row must have a task)

| Case | Task | Expected |
|---|---|---|
| Pydantic `ResponseMessage` objects skipped by converter | 2+3 | dicts; messages survive |
| SDK `{type:message, content:[{type:input_text}]}` | 2+3+4 | Chat `text` / string to upstream |
| Two `input_text` parts | 3 | both texts |
| Extra `id`/`status` on message | 2 `extra=ignore` | accepted |
| `type: text` Chat-style parts | 2 alias | accepted + converted |
| `output_text` on assistant history | 3+4 | converted to `text`; Z.ai must not 400 |
| `developer` role | 2+3 | accepted; mapped to `system` |
| `system` + `user` | already OK non-tools; tools must keep both | 3 |
| `instructions` + string + tools | 3 prepends system | OK |
| `content: null` without tool_calls | 2 skip or 400 | not empty user billed as success |
| `item_reference` only | 2 raise | HTTP 400 (empty-normalize `ValueError`) |
| `item_reference` mixed with user | 2 skip ref | user kept |
| `reasoning` item in input | 2 skip | remaining messages kept |
| `computer_call` / `computer_call_output` | 2 skip | same as reasoning; only-those → 400 |
| `function_call` + `function_call_output` | 2 keep + 3 convert | assistant.tool_calls + role tool |
| Parallel consecutive function_calls | 3 | one assistant, two tool_calls |
| `function_call_output.output` as parts array | 3 stringify | string tool content |
| Only `function_call_output` | 2 keep; 3 tool msg; may 400 upstream if no matching call | do not become empty user `""` |
| List of raw strings | 2 coerce | two user messages |
| Nested Chat tools on Responses | already in `responses_tools_to_chat` | keep |
| Flat Responses tools | already | keep |
| `tool_choice: required` | 2 | not 422 |
| Forced `{type:function, name}` | already | keep |
| `tool_choice: none` | already | keep |
| Hosted `web_search` only | 4 | 400 param=tools |
| Mixed function + hosted | 4 drop hosted, keep function | native tools still run |
| `tools: []` | falsy; non-tools path | text completion |
| Image-only user message | 3 | `[{type:image_url,…}]` not `""` |
| `image_url` object | 2+3 | unwrap `url` |
| Stream tools + list input | 4 same convert | not SSE generic error |
| Stream `input_text` no tools | 4 | non-empty `output_text.done` |
| Double `instructions` prepend | 4 convert once, delete route insert | one system message |
| `__validate_responses_format` rejecting `text`/`developer` | 4 delete or rewrite **after** convert | must not 400 converted chat |
| `__split_responses_content` copy vs send original | 4 send `chat_messages` | splitter no longer on the send path |
| `TAGS_PROVIDERS` vs default `else` | 4 both get `chat_messages` | no branch still dumps Pydantic |
| Schema errors | already HTTP 400 via `src/api.py` | live tests expect 400 not 422 |
| `tools: []` | falsy; non-tools path | text completion (already OK for string content) |
| Non-native-tools fallback to OpenAI Responses | 4 pass original Responses dicts | OpenAI still understands `input_text` + `function_call` |
| `__add_responses_tools_prompt` stripping tool history | 4 stop using on send path | round-trip without `tools` still has tool msgs |
| Empty convert → 400 before upstream | 4 | no Z.ai `Input cannot be empty` |
| Generic “upstream failed” masking | 5 | specific invalid_request when empty |
| `/v1/chat/completions` regression | 7 | tools still work |
| `:dev` models | same code path | no special case |
| `previous_response_id` + full `input` | ignore store (out of scope) | still convert `input` |
| `strict: null` on tools | already ignored | OK |

---

## Out of scope

- Implementing OpenAI `store` / `previous_response_id` / `item_reference` fetch.
- Executing hosted tools (web search, file search, computer use) on Z.ai/Gonka/Venice.
- Changing DevPass `:dev` gating (`ek-proxy-` cannot call `:dev`; that is intentional).
- Billing formula changes (except: empty failed conversions must not look like successful 0-output completions — Task 4 400s before `send_message`).

---

## Self-review

**Spec coverage:** Bug 1 (tools + list), Bug 2 (`input_text` dumped via `__add_responses_tools_prompt`), round-trip, streaming, schema 400s, image flatten, tool output parts, hosted tools, fallback path, strip-helper, empty-input 400, double instructions, validator-order, mixed skipped items, chat regression — each has a task.

**Placeholders:** none.

**Type consistency:** `chat_messages` is always Chat Completions `list[dict]`. Original Responses dicts kept as `responses_input` for the OpenAI fallback only. Handler takes `messages: list` and must not reconvert. `instructions` is applied once in the converter and reused only as an output-envelope field.
