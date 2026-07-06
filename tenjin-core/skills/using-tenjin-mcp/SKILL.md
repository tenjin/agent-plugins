---
name: using-tenjin-mcp
description: Use when working with a user's Tenjin account through the Tenjin MCP server — listing or editing apps, campaigns, channels, ad accounts, events, S2S callbacks, or site-id filters, pulling user-acquisition, ad-monetization, or SKAN reports, or searching Tenjin's product docs. Covers which tool to reach for and how to authenticate.
---

# Using the Tenjin MCP Server

The `tenjin-core` plugin connects the agent to Tenjin's hosted MCP server at
`https://mcp.tenjin.com`. The connection uses OAuth: on first use the
agent is prompted to authorize in the browser; tokens are managed by the host.
If tools return auth errors, tell the user to re-run the plugin's authorization.

## Tool map

**Discover the account**
- `list_apps` — apps in the account (slim fields by default; pass `all_fields=true` for the full record)
- `list_channels` — ad networks / channels
- `list_campaigns` — campaigns (optionally filtered by app or channel; slim by default — pass `all_fields=true` for every field, or `show_tracking_links=true` to add the tracking-link URLs)
- `list_ad_accounts` — connected ad-network accounts
- `list_events` — tracked in-app events
- `get_apps` / `get_campaigns` — full records for specific ids (`ids`, max 50)
- `inspect_app_integration` — check an app's SDK/integration status

**Manage apps & campaigns (writes)**
- `create_apps`, `update_apps`, `delete_apps`
- `create_campaigns`, `update_campaigns`, `delete_campaigns`

**Server-to-server callbacks**
- `list_s2s_callbacks`, `create_s2s_callbacks`, `update_s2s_callbacks`
- `create_callback_groups`, `update_callback_groups`, `update_callback_settings`
- `search_callback_macros` — find the macro tokens a callback URL can use

**Site-id filtering**
- `list_site_id_filters`, `block_site_ids`, `reinstate_site_ids`, `clear_site_id_filters`

**Reporting**
- `get_user_acquisition_report` — UA cohort data by channel, campaign, country, creative, site id
- `get_ad_monetization_report` — ad revenue / eCPM by app, country, ad network
- `get_skan_report` — SKAdNetwork attribution data
- `search_metrics` — discover available metrics and dimensions before building a report

**Documentation**
- `search_docs` — search Tenjin's official product docs (SDK integration, attribution,
  callbacks, SKAN, account setup); ground answers in the returned `tenjin.com/docs/...`
  URLs rather than prior knowledge, and cite them

## How to choose

1. **Reading vs. writing:** confirm intent before any `create_*`, `update_*`,
   `delete_*`, `block_*`, or `clear_*` call — these mutate the user's account.
   Write tools take a batch and return a per-item `{ok, id, error}`; pass
   `dry_run=true` to preview a write without applying it.
2. **Reporting:** start from `search_metrics` to find valid metrics and
   dimensions, then call the matching report tool rather than guessing field names.
3. **Callbacks activate in order:** `inspect_app_integration` first, then
   `update_callback_settings` for channel credentials, then `create_callback_groups`.
   A group can return `ok:true` but `active:false` until its settings are filled —
   re-check `active`. Use `create_s2s_callbacks` only for custom non-channel endpoints.

## Conventions

Most read tools default to TSV (pass `format=json` for JSON); `search_docs` is
JSON-only. The `list_*` tools are cursor-paginated — prefer each tool's
`query`/filter parameters over paging.

## Output style

Lead with the answer, use tables for comparative data, and call out any metric
that is far below benchmark or trending the wrong way.
