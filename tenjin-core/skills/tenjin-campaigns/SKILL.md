---
name: tenjin-campaigns
description: Create, update, and list campaigns and tracking links in Tenjin via the Tenjin MCP. Use when a user wants to create a campaign, generate or copy a tracking link, set up a campaign for a channel or ad account, look up existing campaigns, or change a campaign's attribution window.
---

# Tenjin Campaigns

Manage Tenjin campaigns (tracking links) through the Tenjin MCP tools: create, update, and list. In Tenjin's model, a campaign IS the tracking link — creating a campaign generates its tracking link.

This is step 4 (the final step) of the Tenjin onboarding flow: App → Channels → Callbacks → **Tracking Links**. When arriving from the callbacks skill, the app and channel context may already be known — use it, but always confirm before creating.

## How the hierarchy works (explain this to users who are unsure)
Channel → Ad Accounts → Campaigns.
- A **channel** is the ad network itself (e.g. Meta, Unity, TikTok). Attach a campaign at the channel level with `channel_id`. Note: `channel_id: 0` is **Organic**, which appears in listings but cannot have campaigns created against it via this skill.
- An **ad account** is a specific advertiser account *within* a channel. Attach a campaign to that specific account with `ad_account_id`.
- **Channels and ad accounts cannot be created via the API** — they already exist (channels are Tenjin's supported networks; ad accounts are connected via OAuth in the dashboard). If a user asks to add a channel or ad account, tell them that's a dashboard action, not something this skill can do.

## Core principle: resolve names to IDs first
`create_campaigns` takes UUIDs and integer ids, not names. Before creating:
- Resolve the app name to its UUID via `list_apps` (query by name).
- Resolve the channel or ad account to its id via `list_channels` or `list_ad_accounts`.
If any lookup returns multiple matches, show them and ask which — never guess an id.

## Creating a campaign
Required per campaign: `name`, `app_id`, and **exactly one** of `channel_id` OR `ad_account_id` (never both, never neither). Do NOT use `channel_id: 0` (Organic) when creating — campaigns cannot be created against Organic. If the user wants an Organic campaign, tell them this skill can't create one.

1. Collect the campaign `name` and which app it's for. Resolve the app to its UUID.
2. Determine the attach point. If arriving from the onboarding flow, suggest the channel just configured for callbacks but ask: "Want to create the tracking link for [channel name], or a different channel?" If the user is unsure, explain channel vs. ad account (see hierarchy above) and help them pick:
   - Use `channel_id` for a specific paid network (NOT 0 / Organic).
   - Use `ad_account_id` to attach to a specific connected account.
3. Resolve that channel or ad account to its id via the appropriate `list_` tool. Confirm the chosen channel/ad account name back to the user so they know exactly where the campaign will live.
4. Call `create_campaigns` with an `items` array, including `name`, `app_id`, and exactly one of `channel_id`/`ad_account_id`. Use `dry_run: true` first if the user wants to preview.
5. Report back the new campaign and its tracking link/URL from the response.

## Getting / copying tracking links
The tracking link comes back in the `create_campaigns` and `list_campaigns` responses. To retrieve an existing link, use `list_campaigns` (filter by `app_id`, `channel_id`, or `query` — matches name, short_id, or remote_campaign_id) and hand the user the link value.

## Updating a campaign
Only `attribution_window` is mutable — name, app, channel, and ad account are fixed at creation.
1. Resolve the campaign to its UUID via `list_campaigns`.
2. Call `update_campaigns` with the `id` and the new `attribution_window` (in seconds; default 604800 = 7 days).
3. If the user asks to change anything other than attribution_window, tell them it's immutable — the campaign must be recreated.

## Safety
- Never send both `channel_id` and `ad_account_id`; never send neither.
- Never create a campaign with `channel_id: 0` (Organic) — block it and tell the user.
- Never guess a UUID or id — always resolve via a `list_` tool.
- Confirm the attach point (channel/ad account name) before creating.
- Offer `dry_run: true` when the user seems unsure.
