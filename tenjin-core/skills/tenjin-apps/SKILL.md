---
name: tenjin-apps
description: Create, update, list, and inspect apps in Tenjin via the Tenjin MCP. Use when a user wants to add a new app to Tenjin, onboard an app, edit an existing app's name or credentials (iOS shared secret, Android public key, deeplink protocol), look up an app's ID, or check an app's integration status.
---

# Tenjin Apps

Manage Tenjin apps through the Tenjin MCP tools: create, update, list, and inspect.

## Core principle: resolve names to IDs first
The write tools (`create_apps`, `update_apps`) and inspect tools take UUIDs, not names. When a user refers to an app by name, ALWAYS run `list_apps` with a `query` to resolve the name to its UUID `id` before updating or inspecting. If the query returns more than one match, show the matches and ask which one — never guess.

## Adding a new app
Required fields: `name`, `bundle_id`, `platform` (`ios`/`android`/`amazon`/`android_other`).

1. Collect `name`, `bundle_id`, and `platform`. `name` must be <= 255 characters.
2. **CONFIRM before creating.** `bundle_id` and `platform` are IMMUTABLE — they cannot be changed after creation (`update_apps` only edits name/credentials). Restate the exact `bundle_id` and `platform` back to the user and get explicit confirmation before calling `create_apps`. If anything is ambiguous, ask.
3. Optionally collect credential fields if the user mentions them: `ios_shared_secret` (iOS subscription revenue), `public_key` (Android receipt validation), `protocol` (Facebook deeplink, <= 255 chars), `destination_url` (required only for `android_other`), `match_strings` (<= 255 chars).
4. Call `create_apps` with an `items` array. Use `dry_run: true` first if the user wants to preview without writing.
5. Report back the new app's `id` and confirm what was created.

## After creating an app — channel selection (onboarding flow: App → Channels → Callbacks → Tracking Links)
This is step 1 of the Tenjin onboarding flow. Once an app is created, continue immediately:

1. Call `list_channels` to fetch available channels and display them to the user.
2. Ask: "Which channel would you like to configure callbacks for?" Let the user pick from the list.
3. Note the chosen channel name and its id — pass this context into the next step.
4. Hand off to the **callbacks skill**: "Great, let's set up callbacks for [channel name]. I'll configure the postbacks for this app + channel now."

Do not skip this step or jump straight to campaigns. The correct onboarding order is: app → channel selection → callbacks → tracking links.

## Editing an existing app
Editable: `name`, `ios_shared_secret`, `public_key`, `protocol`, `destination_url`, `match_strings`. NOT editable: `bundle_id`, `platform`.

1. Resolve the app name to its UUID via `list_apps`.
2. If the user asks to change `bundle_id` or `platform`, STOP and tell them these are immutable — the app must be recreated. Do not attempt a workaround.
3. Call `update_apps` with the app's `id` and only the changed fields.
4. Confirm what changed.

## Looking up an app / checking status
- To find an app or its ID: `list_apps` with a `query` (preferred over paging).
- To check an app's callback/integration config for a channel: `inspect_app_integration` with `app_id` and `integration_id`.

## Safety
- Never call `create_apps` without confirming `bundle_id` + `platform` first.
- Never guess a UUID. Always resolve via a `list_` tool.
- Offer `dry_run: true` when the user seems unsure.
