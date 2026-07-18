---
name: debrix-debug
description: >-
  Debug Python AI agents with Debrix local traces via MCP. Use when an agent
  failed, a tool misbehaved, or the user asks what the last Debrix run did —
  including marking failures, configuring capture, Tool Mocks, and Deterministic
  Replay from the IDE.
---

# Debrix debug workflow

Debrix is a **local-first AI agent debugger**. The desktop app must be running
(defaults: OTLP `:17418`, MCP `http://127.0.0.1:17419/mcp` — use the URLs
shown in Debrix if ports changed).

## When to use

- Agent run failed or behaved incorrectly
- Need prompts / tool I/O from the last local run
- Set up Tool Mocks or Deterministic Replay for the next local run
- Adjust capture policy so the next run records messages
- User mentions Debrix, traces, waterfall, mocks, replay, or "what just broke?"

## Preferred tool order

### Observe (read)

1. `get_status` if Debrix connectivity is unclear
2. `get_latest_failed_trace` (optional `agent_name`) **or** filtered `list_traces`
3. `get_trace` with the `trace_id`
4. `get_span_messages` for the failing / LLM span
5. Only then `query_schema` → `query` for custom SQL

Use `find_spans` when hunting by span name, kind (`agent`/`llm`/`tool`/…), or errors across runs.
Use `get_trace_link` to focus the Debrix UI on a trace.
Use `mark_trace_failed` when OTel status is OK but the outcome is wrong (so later
`get_latest_failed_trace` finds it).

### Capture (before re-running)

- `get_settings` / `set_capture_policy` with `full` | `preview` | `off`
- Applies to **next** agent runs. Env `DEBRIX_CAPTURE_MESSAGES` overrides when set.

### Tool Mocks (chaos / edge cases)

1. `discover_tools` (optional) to learn tool names from recent spans
2. `list_mocks` → `upsert_mock` / `set_mock_enabled` / `delete_mock`
3. `set_mocks_enabled` for the master switch (preserves per-rule flags when off)
4. Re-run the agent, then Observe again (`debrix.stub=mock` on stubbed spans)

`upsert_mock` needs `name` + `mode` (`fixed` | `error` | `timeout` | `partial`).
Optional: `kind` (`tool`|`mcp`), `match_args` (object; `{}` = wildcard), `delay_ms`,
`result`, `error_kind`, `error_message`.

### Deterministic Replay

1. `preview_replay` with `trace_id` (mode: `tools` / `tools_only`, or `tools_and_llm`)
2. `start_replay` — then re-run the instrumented agent locally
3. `get_replay_status` for the active session / `result_trace_id`
4. `cancel_replay` (optional `id`; omits → cancel active)

Replay beats mocks for matching tools while a session is active.
For `tools_and_llm`, optional `pinned_llm_span_ids` selects which LLM calls to stub.

### Memory (read-only)

- `memory_summary` — payload / disk usage by agent

## Filter conventions

- Time: ISO 8601 `start_timestamp` / `end_timestamp` (default ~ last 60 minutes)
- Prefer `trace_id`, `agent_name`, `span_name`, status/error — avoid unbounded scans
- Results are capped (~50 traces / 100 query rows)

## Semantic cheat sheet

| Concept | Field |
| ------- | ----- |
| Agent name | `debrix.agent.name` / `agent_name` on traces |
| Span kind | `debrix.span.kind`: agent, llm, tool, mcp, memory, evaluation, human, custom |
| Errors | OTel status ERROR and/or Debrix "Mark failed" |
| Stubbed calls | `debrix.stub=mock` (Tool Mocker) or `debrix.stub=replay` (Deterministic Replay) |
| Messages | Prefer `get_span_messages`; don't invent prompts |

## Diagnosis → patch → experiment

Ground every claim in tool output (trace id, span name, message excerpt, error).
Suggest a concrete prompt or code change the user can apply in the IDE.
You **can** change local Debrix control state via MCP (mocks, capture, mark failed,
replay). Say what you changed. Do not claim you edited the user's agent source
unless you did so in the IDE.
