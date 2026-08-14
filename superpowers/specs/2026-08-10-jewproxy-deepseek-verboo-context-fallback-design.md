# JewProxy DeepSeek Context-Limit → Verboo Fallback

**Date:** 2026-08-10  
**Status:** Approved  
**Repo:** uvicorn-electron

## Goal

When jewproxy rejects any `deepseek/*` request with its `max_proxy_tokens` / context-size `proxy_validation_error` (observed live as **271000**), immediately fall back to Verboo `pro/deepseek-v4-flash` instead of surfacing the proxy error.

## Trigger

- Service is `deepseek` (baseModel prefix `deepseek/…`, including flash and pro).
- Soft/hard proxy body matches context-limit validation, e.g. contains `context size limit` or `proxy_validation_error` + tokens wording.
- No content has been streamed yet (safe to switch upstream).

## Behavior

| Path | Fallback |
|------|----------|
| Chat (`JewProxyClient.send_message`) | `VerbooClient.send_message("pro/deepseek-v4-flash", …)` |
| Native tools (`jewproxy_tool_chunks`) | `openai_compat_tool_chunks(..., provider="verboo", base_model="pro/deepseek-v4-flash")` |
| Responses paths (if ever hit for deepseek) | Same Verboo chat/tools remap |

Do **not** burn jewproxy retries on this deterministic 400. Prefer Verboo over smolproxy for this specific error.

## Non-goals

- Do not change catalog `tokens` in this change.
- Do not fallback unrelated deepseek 400s (bad params, tool_choice, etc.).
- Do not remap non-`deepseek` services (e.g. `openrouter/deepseek/…`).

## Tests

Offline unit tests for detection helpers + hook that non-retryable context-limit on deepseek selects Verboo model id.

## Audit follow-ups (2026-08-10)

- Context-limit errors are **never retryable** (even when the body omits `HTTP 400`), so we do not burn key rotation or fall through to smolproxy.
- Detector requires context wording (`context size limit` / `At 'tokens'…exceed` / `proxy_validation_error`+`context`), not bare `token`.
- Hard HTTP non-200 bodies with the same wording also hop to Verboo (chat + tools + responses).
- Shared `_maybe_*` helpers collapse duplicated call sites.
