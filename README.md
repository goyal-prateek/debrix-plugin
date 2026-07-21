# Debrix IDE plugin

Cursor · Claude Code · Codex — local MCP tools plus `debrix-debug` / `debrix-instrument` skills.

**Prerequisite:** Debrix desktop running with MCP at `http://127.0.0.1:17419/mcp`.

Source: [goyal-prateek/debrix-plugin](https://github.com/goyal-prateek/debrix-plugin)

## Layout

```text
.cursor-plugin/plugin.json
.claude-plugin/plugin.json + marketplace.json
.codex-plugin/plugin.json
mcp.json                 # Cursor
.mcp.json                # Claude / Codex (http type)
skills/
├── debrix-debug/        # inspect traces + mocks / replay via MCP
└── debrix-instrument/   # pip install + SDK instrumentation
```

MCP covers Observe (traces, spans, SQL), Tool Mocks, mark failed,
Deterministic Replay, and status — see `skills/debrix-debug`.

## Cursor

### One-click MCP (tools only)

Start Debrix desktop, then:

[![Add Debrix MCP to Cursor](https://cursor.com/deeplink/mcp-install-dark.svg)](cursor://anysphere.cursor-deeplink/mcp/install?name=debrix&config=eyJ1cmwiOiJodHRwOi8vMTI3LjAuMC4xOjE3NDE5L21jcCJ9)

### Plugin (MCP + skills)

```bash
git clone https://github.com/goyal-prateek/debrix-plugin.git
mkdir -p ~/.cursor/plugins/local
ln -sf "$(pwd)/debrix-plugin" ~/.cursor/plugins/local/debrix
# Reload Window in Cursor
```

Submit this repository to the [Cursor Marketplace](https://cursor.com/marketplace/publish) when ready.

## Claude Code

```text
/plugin marketplace add https://github.com/goyal-prateek/debrix-plugin
/plugin install debrix@debrix-plugins
```

```bash
claude plugin marketplace add https://github.com/goyal-prateek/debrix-plugin
claude plugin install debrix@debrix-plugins
```

## Codex

```bash
codex plugin marketplace add https://github.com/goyal-prateek/debrix-plugin
```

Then **Plugins** → install **Debrix**.

## Portable skills

```bash
git clone https://github.com/goyal-prateek/debrix-plugin.git
cp -R debrix-plugin/skills/debrix-debug ~/.claude/skills/debrix-debug
cp -R debrix-plugin/skills/debrix-instrument ~/.claude/skills/debrix-instrument
```

## Related

- Python SDK: [`pip install debrix`](https://pypi.org/project/debrix/) · [goyal-prateek/debrix-sdk](https://github.com/goyal-prateek/debrix-sdk)

## License

MIT
