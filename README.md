# Tenjin Agent Plugins

Plugins that bring Tenjin into AI coding agents. Two plugins ship from this
repo, installable on **Claude Code** and **Codex**:

| Plugin | What it gives the agent |
|--------|-------------------------|
| `tenjin-core` | Connects to Tenjin's hosted MCP server (`https://mcp.tenjin.com`, OAuth) and a skill that orients the agent to its tools. |
| `tenjin-sdk` | Per-platform Tenjin SDK integration guides (iOS, Android, Flutter, Ionic, React Native, Unity). |

## Install

### Codex

```bash
codex plugin marketplace add tenjin/agent-plugins
codex plugin install tenjin-core
```

### Claude Code

```
/plugin marketplace add tenjin/agent-plugins
/plugin install tenjin-core@tenjin-agent-plugins
```

Install `tenjin-sdk` the same way.

## Layout

Each plugin carries both a Claude manifest (`.claude-plugin/plugin.json`) and a
Codex manifest (`.codex-plugin/plugin.json`) over one shared `skills/` body.
Two marketplace catalogs (`.claude-plugin/marketplace.json` and
`.agents/plugins/marketplace.json`) list the same two plugins.

Licensed under Apache-2.0.
