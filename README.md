# Tenjin Agent Plugins

Plugins that bring Tenjin into AI coding agents. Three plugins ship from this
repo, installable on **Claude Code** and **Codex**:

| Plugin                 | What it gives the agent                                                                                                       |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `tenjin-core`          | Connects to Tenjin's hosted MCP server (`https://mcp.tenjin.com`, OAuth) and a skill that orients the agent to its tools.     |
| `tenjin-sdk`           | Per-platform Tenjin SDK integration guides (iOS, Android, Flutter, Ionic, React Native, Unity).                               |
| `in-app-subscriptions` | Implementing native in-app subscriptions end to end: store setup, purchase flow, server-side verification, webhooks, restore. |

## Install

### Claude Code

From your shell:

```bash
claude plugin marketplace add tenjin/agent-plugins
claude plugin install tenjin-core@tenjin-agent-plugins
claude plugin install tenjin-sdk@tenjin-agent-plugins   # optional
claude plugin install in-app-subscriptions@tenjin-agent-plugins   # optional
claude mcp login plugin:tenjin-core:tenjin   # complete authentication in browser
```

Or from inside a Claude Code session:

```
/plugin marketplace add tenjin/agent-plugins
/plugin install tenjin-core@tenjin-agent-plugins
/plugin install tenjin-sdk@tenjin-agent-plugins   # optional
/plugin install in-app-subscriptions@tenjin-agent-plugins   # optional
/reload-plugins
/mcp   # choose plugin:tenjin-core:tenjin, then Authenticate, then complete authentication in browser
```

On Claude Code Web / Desktop, use the GUI instead:

1. Open **Customize → Plugins**.
2. Click **Add → Add marketplace**.
3. Choose **Add from a repository** and enter `tenjin/agent-plugins`.
4. Install `tenjin-core` — and optionally `tenjin-sdk` or `in-app-subscriptions` — from the marketplace's plugin list.
5. Complete Tenjin's OAuth flow when prompted — it opens automatically once `tenjin-core` installs.

### Codex

From your shell:

```bash
codex plugin marketplace add tenjin/agent-plugins
codex plugin add tenjin-core@tenjin-agent-plugins
codex plugin add tenjin-sdk@tenjin-agent-plugins   # optional
codex plugin add in-app-subscriptions@tenjin-agent-plugins   # optional
codex mcp login tenjin   # authenticates tenjin mcp
```

On Codex Desktop, use the GUI instead:

1. Click **Plugins** in the sidebar.
2. Expand the `+` dropdown menu, then select **Add marketplace**.
3. For Source, enter `tenjin/agent-plugins`. Click **Add Marketplace**.
4. Go to the **Personal** tab on the plugins page. You should see "Tenjin Agent Plugins" listed.
5. Click **Install** next to the `tenjin-core` plugin. It will install the plugin and open the Tenjin OAuth flow.
6. Optional: Click **Install** next to the `tenjin-sdk` plugin to install the SDK skills, or next to `in-app-subscriptions` for the subscriptions skill.

## Layout

Each plugin carries both a Claude manifest (`.claude-plugin/plugin.json`) and a
Codex manifest (`.codex-plugin/plugin.json`) over one shared `skills/` body.
Two marketplace catalogs (`.claude-plugin/marketplace.json` and
`.agents/plugins/marketplace.json`) list the same three plugins.

## MCP Only

The tenjin-core plugin already provides MCP functionality. If you cannot install the plugin,
or if you prefer not to have Tenjin's provided skills, you may use the Tenjin MCP server directly
at `https://mcp.tenjin.com`.

## See also

- Claude apps (Web / Desktop / Cowork) docs: <https://support.claude.com>
- Claude Code CLI docs: <https://code.claude.com>
- Codex docs: <https://developers.openai.com/codex>

Licensed under MIT.
