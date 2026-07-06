---
name: tenjin-callbacks
description: Create, update, and inspect Tenjin S2S callbacks and callback groups (postbacks) for an app via the Tenjin MCP. Use when a user wants to set up, enable, configure, or modify callbacks or postbacks for a Tenjin app — including custom server-to-server callbacks (ping a URL on an event) or channel callback groups (template-based, tied to a marketing integration). Also covers finding callback macros for building postback URLs.
---

# Tenjin Callbacks

Manage Tenjin callbacks/postbacks through the Tenjin MCP tools. There are TWO distinct types — identify which the user needs before doing anything.

## The two callback types (explain this when the user is unsure)
- **Custom S2S callback** (`create_s2s_callbacks`): a standalone callback that pings a URL you specify when an event happens. Use when the user wants to send data to their OWN server/endpoint on an event. Simple: event + URL + method. No template, no channel required.
- **Callback group** (`create_callback_groups`): a template-based callback tied to a specific marketing channel/integration (e.g. sending postbacks to an ad network). Use when the user wants to send postbacks to a CHANNEL/ad network using Tenjin's prebuilt template. Requires a strict setup order (below).

If unsure which the user wants, ask: "Are you sending data to your own server (custom callback), or sending postbacks to an ad network/channel (callback group)?"

**Key signals for callback groups:** phrases like "install callback to [network]", "postback to [network]", "enable [network] callback", or any mention of a specific ad network as the destination → use Path B (callback group), NOT S2S.

## Core principle: resolve names to IDs first
These tools take a UUID `app_id`, not a name. Resolve the app name to its UUID via `list_apps` before anything. For callback groups you also need an `integration_id` (the marketing channel) — resolve via `list_channels` / `inspect_app_integration`. Never guess ids.

## Path A — Custom S2S callback
1. Resolve the app to its UUID via `list_apps`.
2. Collect per callback: `event` (what triggers it, e.g. `purchase`), `url` (endpoint to hit), `http_method` (`GET`/`POST`). Optional: `name`, `action` (`ping`/`ping_on_first`), `active`, `advertising_id_filter`, `user_filter` (array of ad-network ids; empty = all networks).
3. If the user needs macros in their URL (e.g. device id, country), use `search_callback_macros` to find them. Macro names come back bare — wrap them in `{{ }}` to use (e.g. `country` → `{{country}}`).
4. Confirm the event + URL + method back to the user.
5. Call `create_s2s_callbacks` with `app_id` and an `items` array. Use `dry_run: true` to preview first.
6. Report what was created.

## Path B — Callback group (STRICT ORDER — do not reorder)
Callback groups MUST be set up in this exact order, or the group comes back `active: false` (silently broken):

1. Resolve the app to its UUID (`list_apps`) and the channel to its `integration_id`.
2. **Inspect first.** Call `inspect_app_integration` with `app_id` + `integration_id` to get the available `callback_group_template_id`(s) and events. You CANNOT create a group without a template id from here.
3. **Set channel settings BEFORE creating the group — ALWAYS, no exceptions.** Call `update_callback_settings` with `app_id`, `integration_id`, and any `settings` you have (credentials/macros; merged, not replaced). Call this step **even if the user gave no credentials and even if you think there is nothing to set** — skipping it makes the group come back `active: false` (silently broken). Do NOT call `create_callback_groups` unless `update_callback_settings` was already called first in this same flow.
4. **Then create the group.** Call `create_callback_groups` with `app_id`, `integration_id`, and an `items` array. Each item needs `callback_group_template_id`. If the template is a custom-event template (its event is null), also provide `event` or `event_definition_id` (from `list_events`; `event_definition_id` takes priority). Optional: `name`, `active`, `user_filter` (`all`/`channel`/`channel_and_organic`).
5. **Verify.** Call `inspect_app_integration` again to confirm the group is active. If it's `active: false`, the settings step was likely missed — fix settings and retry.

## Updating callbacks
- Custom S2S: `update_s2s_callbacks` with the callback's `id` (get it from `list_s2s_callbacks`) and changed fields.
- Channel settings: `update_callback_settings` (merges into existing settings).

## Macros reference
`search_callback_macros` finds macros for building URLs. Filter by `category` (App, Attribution, Device Identifier, Device Properties, Event, Ad Impressions, Meta Attributions) or `provider` (AppLovin, Unity LevelPlay, AdMob, Topon — for Ad Impressions). No args = full catalog. Always wrap names in `{{ }}` when using.

## After callbacks are verified — offer tracking links (onboarding flow: App → Channels → Callbacks → Tracking Links)
This is step 3 of the Tenjin onboarding flow. Once callbacks are confirmed active, offer the next step:

Ask: "Callbacks are live for [channel name]. Want to set up tracking links now? I'd suggest starting with [channel name] since that's what we just configured — but you can create tracking links for any channel. Which would you like?"

- If they choose the same channel, hand off to the **campaigns skill** with that channel already identified.
- If they choose a different channel, hand off to the **campaigns skill** and let it resolve the new channel fresh.
- If they want to skip for now, confirm onboarding is complete and they can create tracking links anytime.

## Safety
- Always identify the callback type before acting.
- For callback groups: NEVER skip update_callback_settings before create_callback_groups, and always verify active status after.
- Resolve all ids via list_/inspect tools — never guess.
- Confirm event + URL (custom) or channel + template (group) before creating.
- Offer dry_run: true to preview.
