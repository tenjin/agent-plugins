---
name: using-tenjin-mcp
description: Use when working with a user's Tenjin account through the Tenjin MCP server — listing or editing apps, campaigns, channels, ad accounts, events, S2S callbacks, or site-id filters, and pulling user-acquisition, ad-monetization, or SKAN reports. Covers which tool to reach for and how to authenticate.
---

# Using the Tenjin MCP Server

The `tenjin-core` plugin connects the agent to Tenjin's hosted MCP server at
`https://mcp.tenjin.com`. The connection uses OAuth: on first use the
agent is prompted to authorize in the browser; tokens are managed by the host.
If tools return auth errors, tell the user to re-run the plugin's authorization.

## Tool map

**Discover the account**
- `list_apps` — apps in the account
- `list_channels` — ad networks / channels
- `list_campaigns` — campaigns (optionally filtered by app or channel)
- `list_ad_accounts` — connected ad-network accounts
- `list_events` — tracked in-app events
- `inspect_app_integration` — check an app's SDK/integration status

**Manage apps & campaigns (writes)**
- `create_apps`, `update_apps`, `delete_apps`
- `create_campaigns`, `update_campaigns`, `delete_campaigns`

**Server-to-server callbacks**
- `list_s2s_callbacks`, `create_s2s_callbacks`, `update_s2s_callbacks`
- `create_callback_groups`, `update_callback_groups`, `update_callback_settings`
- `search_callback_macros` — find the macro tokens a callback URL can use

**Site-id filtering**
- `get_site_id_filters`, `block_site_ids`, `reinstate_site_ids`, `clear_site_id_filters`

**Reporting**
- `get_user_acquisition_report` — UA cohort data by channel, campaign, country, creative, site id
- `get_ad_monetization_report` — ad revenue / eCPM by app, country, ad network
- `get_skan_report` — SKAdNetwork attribution data
- `search_metrics` — discover available metrics and dimensions before building a report

## How to choose

1. **Reading vs. writing:** confirm intent before any `create_*`, `update_*`,
   `delete_*`, `block_*`, or `clear_*` call — these mutate the user's account.
2. **Reporting questions** (ROAS, LTV, retention, fraud, cohort health, etc.)
   are deeper than raw report fetches. If the `tenjin-advanced-analytics`
   plugin is installed, prefer its playbooks; otherwise start from
   `search_metrics` to find valid metrics/dimensions, then the matching report tool.
3. **Unknown metric or dimension:** call `search_metrics` first rather than
   guessing field names.

## Output style

Lead with the answer, use tables for comparative data, and call out any metric
that is far below benchmark or trending the wrong way.
