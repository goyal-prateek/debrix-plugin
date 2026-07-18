---
name: debrix-debug
description: >-
  Debug Python AI agents with Debrix local traces via MCP. Use when an agent
  failed, a tool misbehaved, or the user asks what the last Debrix run did.
---

# Debrix debug workflow

Debrix is a **local-first AI agent debugger**. The desktop app must be running
(defaults: OTLP `:17418`, MCP `http://127.0.0.1:17419/mcp` — use the URLs
shown in Debrix if ports changed).

## When to use

- Agent run failed or behaved incorrectly
- Need prompts / tool I/O from the last local run
- User mentions Debrix, traces, waterfall, or "what just broke?"

## Preferred tool order

1. `get_latest_failed_trace` (optional `agent_name`) **or** filtered `list_traces`
2. `get_trace` with the `trace_id`
3. `get_span_messages` for the failing / LLM span
4. Only then `query_schema` → `query` for custom SQL

Use `find_spans` when hunting by span name, kind (`agent`/`llm`/`tool`/…), or errors across runs.
Use `get_trace_link` to focus the Debrix UI on a trace.

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

## Diagnosis → patch

Ground every claim in tool output (trace id, span name, message excerpt, error).
Suggest a concrete prompt or code change the user can apply in the IDE.
Do not claim you changed Debrix data — you only read it via MCP.
