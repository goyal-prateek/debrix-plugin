---
name: debrix-instrument
description: >-
  Instrument Python AI agents with the Debrix SDK (pip install debrix,
  configure, trace_agent, trace_tool, trace_span, record_messages). Use when
  adding Debrix tracing, wiring OTLP to localhost:17418, or when the user asks
  how to instrument, observe, or capture prompts/tool I/O with Debrix.
---

# Instrument with Debrix

Debrix is a **local-first AI agent debugger**. Spans go OTLP/HTTP →
`http://127.0.0.1:17418` by default (use the URL shown in Debrix if ports
changed). **Debrix desktop must be running** or nothing appears in the UI.

After instrumenting, use the `debrix-debug` skill + MCP tools to inspect runs.
For full API details see the [debrix SDK README](https://github.com/goyal-prateek/debrix-sdk).

## 1. Install

```bash
pip install debrix
```

Requires Python `>=3.11`. Package is alpha; APIs may change.

## 2. Configure (once per process)

```python
from debrix import configure, force_flush

configure(
    endpoint=None,           # default http://127.0.0.1:17418
    service_name="my-agent", # OTel resource service.name
    batch=True,              # BatchSpanProcessor; False for short scripts
)
# ... run agent ...
force_flush()  # always call before short scripts exit
```

| Detail | Behavior |
| ------ | -------- |
| Default endpoint | `http://127.0.0.1:17418` → `{endpoint}/v1/traces` |
| Env override | `DEBRIX_OTLP_ENDPOINT` when `endpoint=` omitted (not the shared `OTEL_EXPORTER_OTLP_*` vars) |
| Idempotent | Existing `TracerProvider` is reused (second `configure` won't rebind) |
| Lazy | First `trace_*` / `get_tracer()` calls `configure()` if needed |

**Short demos:** `configure(batch=False, service_name="basic-agent")` then
`force_flush()`. Prefer `batch=True` for long-running agents.

## 3. Instrumentation APIs

Public imports: `configure`, `force_flush`, `trace_agent`, `trace_tool`,
`trace_span`, `DebrixSpan`, `SpanKind`, `Attr`, `get_tracer`.

### Agent boundary — `trace_agent`

Sets span kind `agent` and `debrix.agent.name`. Decorators automatically record
bound function arguments, including defaults, on `debrix.agent.arguments`.
Agent return values are not captured.

```python
from debrix import trace_agent

@trace_agent
def run(query: str) -> str:
    ...

@trace_agent(name="research_agent")
def run_agent(query: str) -> str:
    ...

with trace_agent("planner", arguments={"query": "inspect"}) as span:
    ...
```

Sync and async decorators are supported. Context-managed agents cannot infer
function arguments, so pass a mapping explicitly when needed.

### Tool calls — `trace_tool`

Sets kind `tool` and `debrix.tool.name`. The **decorator** form records tool
I/O for Debrix Tool Mocker and Deterministic Replay. When those features are
armed in Debrix, stubbed spans set `debrix.stub` to `mock` or `replay`
(omit when live). Context-manager form does **not** auto-capture I/O.

For Deterministic Replay of LLM calls, wrap chat completions with
`debrix.llm.complete` instead of calling the provider directly.

```python
from debrix import trace_tool

@trace_tool(name="lookup")
def lookup(topic: str) -> str:
    return f"facts about {topic}"

with trace_tool("search") as span:
    ...
```

### LLM / custom spans — `trace_span`

```python
from debrix import SpanKind, trace_span

with trace_span("complete", kind=SpanKind.LLM) as span:
    span.record_messages([
        {"role": "system", "content": "You are helpful."},
        {"role": "user", "content": query},
        {"role": "tool", "name": "lookup", "content": "..."},  # name optional
    ])
    answer = call_model(...)
    span.record_response({
        "content": answer,
        "model": "…",
        "usage": {"input_tokens": 10, "output_tokens": 5},
    })
```

**`SpanKind`:** `agent`, `llm`, `tool`, `mcp`, `memory`, `evaluation`, `human`,
`custom` (default for `trace_span`).

On exception, instrumented spans get OTel `ERROR` + `debrix.error.summary`
(≤200 chars).

### Full local message / response capture

Decorators alone do **not** capture prompts. Once these methods are called,
Debrix stores the complete payload locally and keeps a bounded preview on the
span for fast inspection:

| Method | Notes |
| ------ | ----- |
| `record_messages([{role, content, name?}])` | Roles: `system` \| `user` \| `assistant` \| `tool` |
| `record_response({content, model?, usage?, …})` | Loose shape; extras allowed |
| `record_exception(exc)` | Manual; also auto on context exit |
| `set_attribute(key, value)` | Extra span attrs |

Bad `role` → `ValueError`. Nested `trace_*` calls propagate via OpenTelemetry
context.

### Capture limits (env)

Not kwargs on `configure()` — set via env:

| Env | Values | Effect |
| --- | ------ | ------ |
| `DEBRIX_MAX_PAYLOAD_BYTES` | default `209715200` | Over-cap → capture skipped |
| `DEBRIX_PREVIEW_CHARS` | default `4096` | Preview size budget |

Without `force_flush()`, short scripts can lose message bodies even if the
span landed.

## 4. Recommended pattern

One agent root → tools as `@trace_tool` → shared `complete()` that wraps the
model call in `trace_span(..., kind=SpanKind.LLM)` with `record_messages` /
`record_response`. Do **not** wrap vendor SDKs or add LangChain adapters yet
(not shipped).

```python
from debrix import configure, force_flush, trace_agent, trace_tool, trace_span, SpanKind

configure(batch=False, service_name="my-agent")

@trace_tool(name="search")
def search(q: str) -> str:
    return "…"

@trace_agent(name="research_agent")
def run(query: str) -> str:
    facts = search(query)
    with trace_span("complete", kind=SpanKind.LLM) as span:
        messages = [
            {"role": "system", "content": "Answer from facts."},
            {"role": "user", "content": query},
            {"role": "tool", "name": "search", "content": facts},
        ]
        span.record_messages(messages)
        answer = "…"  # your LLM call
        span.record_response({"content": answer, "usage": {"input_tokens": 0, "output_tokens": 0}})
        return answer

if __name__ == "__main__":
    print(run("hello"))
    force_flush()
```

## 5. Gotchas

1. **Desktop must be open** — nothing is received otherwise.
2. **`force_flush()` before exit** in scripts — message uploads can lag behind spans.
3. **Messages are opt-in** — use `record_messages` / `record_response`.
4. **`trace_tool` decorator ≠ context manager** — only decorator auto-records I/O.
5. **Idempotent `configure`** — change endpoint/service_name before first span, or restart the process.
6. No framework adapters or `from debrix import Anthropic` wrappers yet — hand-instrument with the APIs above.

## 6. Verify

1. Run the instrumented script with Debrix desktop open.
2. Open Debrix → Observe waterfall, or use MCP (`get_status`, `list_traces` /
   `get_trace`, `get_span_messages`).
3. Confirm LLM spans show messages via `get_span_messages` (MCP) or the UI.
