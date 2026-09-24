---
name: meta-campaign-setup
description: Use whenever a Tenjin user wants to set up, launch, or troubleshoot a Facebook/Meta ad campaign alongside Tenjin tracking — using the Meta Ads MCP server and the Tenjin MCP server together. Trigger this for requests like "create a Meta campaign for my app," "set up Facebook ads and track it in Tenjin," "why don't my Tenjin installs match Meta's," "add my Facebook app ID to Tenjin," "set up the Meta callback," or "pull combined reporting from Meta and Tenjin." Also trigger when a user reports a discrepancy between Meta-reported installs and Tenjin tracked installs — this skill covers the AMM (Advanced Mobile Measurement) terms fix, which is the single highest-leverage step for that problem, plus the privacy-threshold behavior that explains most other cases.
---

# Meta + Tenjin Combined Campaign Setup

This skill walks through standing up a Meta (Facebook) app-install campaign end to
end, wiring it to Tenjin for independent attribution, and building combined
reporting that correctly separates Meta's self-reported numbers from Tenjin's own
tracked/confirmed numbers. It also captures the tool quirks and dead ends
discovered building this workflow the first time, so they don't have to be
rediscovered.

Use the Meta Ads MCP server for anything under `ads_*`. Use the Tenjin MCP server
for anything app/campaign/callback/report related on the Tenjin side.

## Before you start: connecting the Meta Ads MCP server to Claude

If `ads_*` tools aren't available yet, the Meta Ads MCP server needs to be
connected first. This is a one-time setup per user, done in Claude's interface,
not something Claude can do on someone's behalf:

1. Open **Customize → Connectors** in Claude
2. Check the **Connectors Directory** first — if Meta Ads is listed there
   directly, click it and authenticate; this is the simplest path when
   available
3. If it isn't listed, add it as a **custom connector**: click the **"+"**
   button next to Connectors, choose custom connector, and enter Meta's remote
   MCP server URL: `https://mcp.facebook.com/ads`
4. Follow the OAuth prompts to authorize the Facebook Business account — this
   confirms which Meta account/business is being connected, so double-check
   it's the right one before approving
5. Once connected, start a new chat and confirm the connector is toggled on
   (via the "+" in the chat box → Connectors)

**On a Team or Enterprise plan**: an Owner or Primary Owner needs to enable
connectors org-wide first, under **Organization settings → Connectors**, before
individual members can connect their own accounts. If `ads_*` tools are missing
and this is a company account, check with an admin before assuming the setup
steps above will work — individual members can't add net-new connectors to the
org catalog themselves.

The same applies to the **Tenjin MCP server** if it isn't connected yet either —
same Connectors flow, Tenjin's own MCP server URL in place of Meta's.

## Overview of the full workflow

1. Connect Meta as a channel in Tenjin (if not already done)
2. Register the app across Meta's three separate systems (App Dashboard,
   Business Manager, Events Manager) — all manual, no API for this part
3. Sign AMM terms — the single highest-leverage step for making Tenjin's
   tracked numbers match Meta's reported numbers; do this before judging
   whether tracking is working
4. Wire up the Tenjin callback for the app
5. Build the Meta campaign → ad set → ad (all paused)
6. Activate
7. Pull combined reporting and interpret it correctly

---

## Step 1 — Connect Meta as a channel in Tenjin

Check first with `list_ad_accounts(channel_id=<meta's id>)` (Tenjin MCP) —
`list_channels` lists every ad network whether or not it's connected, so it
can't tell you this. If Meta has no ad account, it needs to be connected via
OAuth in the Tenjin dashboard —
there's no API for this step, and you cannot create the connection yourself.
This mirrors the `connect-ad-account` skill's guidance — use that skill directly
if it's available, or follow the same steps here:

1. Call `list_channels` with `query="Meta"` (or `"Facebook"`) to resolve Meta's
   numeric channel id. If no match, call `list_channels` with no query to see
   the full list and confirm the right one with the user.
2. Hand the user this exact dashboard link, with the resolved id filled in:
   `https://dashboard.tenjin.com/dashboard/integrations/new?ad_network_id={id}`
   — this is where they authenticate with Meta and enter credentials. Never
   collect, request, or handle the credentials yourself; there is nothing safe
   for the agent to do with them, and the flow is architecturally confined to
   the dashboard.
3. This also connects the underlying **ad account** — confirm afterward with
   `list_ad_accounts(channel_id=<meta's id>)`.

Meta's channel `id` from `list_channels` is the `integration_id` (commonly `3`,
but confirm — don't assume). Every later Tenjin callback call needs this
`integration_id`.

## Step 2 — Register the app across Meta's systems

This trips people up because it's **three separate systems**, each needing its
own step, none of which are reachable via the Meta Ads MCP tools:

| System | What happens there | URL |
|---|---|---|
| App Dashboard | Create the App ID (My Apps → Create App, or select an existing app — the App ID shows at the top next to the app name), set Bundle ID + Store ID under the iOS/Android platform section | https://developers.facebook.com/apps |
| Business Manager | Link the App ID as a business asset, assign it to the ad account | business.facebook.com/settings/apps |
| Events Manager | Register the app as a data source (dataset) for event/SKAN/AEM tracking | business.facebook.com/events_manager2 |

Common failure modes here:
- **"Application/Object Store URL Mismatch"** — the app's Store ID/Bundle ID in
  the App Dashboard doesn't match what's passed as `object_store_url` in the
  campaign's `promoted_object`. Check Settings → Basic → the platform section for
  the exact values (field is often labeled **"iPhone Store ID"**, a bare numeric
  ID, not a full URL).
- **Wrong ad account context** — Events Manager is scoped per Business Manager
  account. If the account selector top-right doesn't match the ad account
  actually running the campaign, the app's dataset will look invisible even
  though it exists. Always check that selector first.
- **Page permission errors on creative creation** ("No permission to access this
  profile") — being linked to an ad account isn't the same as having
  Advertiser-level access to a specific Page. Check with `ads_get_user_pages`
  (Pages the current user can actually publish creatives under) rather than
  `ads_get_ad_account_pages` (Pages merely linked to the account). Fix in
  Business Settings → Accounts → Pages → assign the right role.

## Step 3 — Sign AMM terms

**Get the user to sign Meta's Advanced Mobile Measurement (AMM) terms now, if
not already done.** This is the single most important step for making Tenjin's
tracked numbers match Meta's reported numbers, and it's easy to miss because
nothing forces it.

Per Tenjin's own documentation, full campaign reporting is only available if
**either** of these conditions is met:
1. The campaign meets the minimum volume threshold: **more than 100 unique
   users or more than 100 tracked installs**, or
2. **AMM terms are signed**

Below that volume threshold, without AMM signed, Tenjin applies
**privacy-threshold logic**: it suppresses or zeroes out tracked installs and
tracked impressions rather than reporting a possibly-unreliable small number.
This is what makes a brand-new or low-volume campaign look like tracking is
"broken" — Meta reports real installs, Tenjin's `tracked_installs` shows 0, and
nothing is actually wrong except being under both thresholds. **Signing AMM is
the only one of the two conditions the user actually controls** — volume is
just a function of budget and time, but AMM removes the dependency on volume
entirely, which is why it's worth doing regardless of campaign size.

- AMM terms page (requires **App Admin** role on the specific Facebook app, not
  just Business Manager access): https://developers.facebook.com/advanced_mobile_measurement/terms
- Tenjin's own docs on this: https://tenjin.com/docs/meta-amm-tenjin/
- Tenjin's privacy threshold docs (source for the numbers above):
  https://tenjin.com/docs/meta-setup/#meta-privacy-threshold

Ask the user whether AMM is signed for the app in question before they run any
real test budget or judge whether tracking is working — it will save a lot of
confusion later comparing Meta vs. Tenjin numbers. See
`references/troubleshooting.md` for the full diagnostic path if discrepancies
persist after AMM is signed.

## Step 4 — Wire up the Tenjin callback

Once the app has a Meta App ID and Meta is a connected channel:

1. `update_callback_settings(app_id, integration_id, {"facebook_app_id": "<App ID>"})`
2. `create_callback_groups(app_id, integration_id, items=[{callback_group_template_id, event, name, active: true}])`
   — for a basic install-only setup, use the **"Meta Install"** template on the
   `open` event. Get the template's id (a UUID) from `available_templates` in
   `inspect_app_integration(app_id, integration_id)`. Always pass `event` on the
   item, even when the template's own `event` field is already set — the tool
   rejects items without `event` or `event_definition_id`.
3. Verify with `inspect_app_integration(app_id, integration_id)` — confirm
   `facebook_app_id` is set and the callback shows `active: true`,
   `missing_settings: []`.

## Step 5 — Build the Meta campaign

Use `ads_create_campaign` → `ads_create_ad_set` → `ads_create_ad`, in that order.
Everything is created **PAUSED** by default — confirm this with the user, don't
skip it.

- **Objective**: `OUTCOME_APP_PROMOTION`
- **`promoted_object`**: `{"application_id": "<App ID>", "object_store_url": "<store URL>"}`
- **Budget**: prefer CBO (campaign-level `campaign_lifetime_budget` or
  `campaign_daily_budget`) unless the user specifically wants ABO
- **`is_skadnetwork_attribution`**: see the eligibility note below before setting
  this — it is not a simple on/off choice
- **Ad set**: needs its own `end_time` if the parent campaign uses a lifetime
  budget, even though the budget itself lives on the campaign
- **Ad**: needs a `creative_id` from `ads_create_creative` first, or an inline
  `object_story_spec`. Creative creation needs Page permission — see Step 2.

### SKAN vs. AEM eligibility — read this before setting `is_skadnetwork_attribution`

For a new or low-volume app, expect **eligibility errors on both settings**
independently:
- `is_skadnetwork_attribution: true` → possible `"App is Ineligible for Apple's
  SKAdNetwork"`
- `is_skadnetwork_attribution: false` → possible `"App is Ineligible for
  Aggregated Event Measurement"`

These are two separate eligibility gates, not one. Fix path:

1. In Events Manager → the app's dataset → Settings → **"Meta's attribution for
   iOS 14+"** section → click **"Check app eligibility"**
2. If eligible, prefer **AEM** (`is_skadnetwork_attribution: false`) over pure
   SKAN — AEM still uses SKAN underneath for opted-out users, it's the more
   inclusive setting, not a lesser one. Pure SKAN-only mode has historically been
   harder to get eligible for a brand-new low-volume app.
3. **Important**: if the campaign was *created* before eligibility was
   confirmed, editing it afterward may keep failing even though eligibility now
   shows as confirmed — the campaign can carry a stale eligibility snapshot from
   creation time. If budget/date edits keep failing with these errors after
   confirming eligibility, don't keep retrying the edit — **rebuild the campaign
   fresh** (new `campaign_id`, same ad set/ad settings, reuse the existing
   creative) rather than continuing to fight the old one.

See `references/troubleshooting.md` for the full list of tool quirks encountered
building this (broken `ads_create_ad_set` in some sessions, `ads_activate_entity`
API/UI inconsistency, etc.) before assuming a new error is a dead end.

## Step 6 — Activate

Use `ads_activate_entity` on campaign → ad set → ad, in that order. All three
need to be Active for delivery — one paused level blocks everything.

**Known inconsistency**: the Meta Ads MCP's `ads_activate_entity` call can reject
activation (e.g. with an AEM/SKAN eligibility error) even when the same action
succeeds through the Ads Manager UI directly. If API activation fails and you've
otherwise confirmed eligibility, ask the user to try activating manually in Ads
Manager, then verify the real status via `ads_get_ad_entities` (fields: `status`,
`effective_status`) rather than assuming the API's rejection is still accurate.

## Step 7 — Combined reporting: Meta vs. Tenjin

This is the part most likely to be misread if you're not careful about which
Tenjin metric you're pulling.

### The critical distinction

Tenjin's `get_user_acquisition_report` metrics come in matched pairs:

| "Reported" (Meta's own number, ingested via API) | "Tracked" (Tenjin's own independent attribution) |
|---|---|
| `installs` | `tracked_installs` |
| `impressions` | `tracked_impressions` |
| `spend` | — (no tracked equivalent; Tenjin can't observe spend independently) |

**`installs` alone is NOT confirmation that Tenjin attributed anything to Meta —
it's just Meta's self-reported number relayed for cost context.** Always pull
`tracked_installs` (and `tracked_impressions`) explicitly if the goal is to
validate that attribution is actually working, not just to relay Meta's own
numbers back to the user.

### Building the report

Pull from both sources for the same date range:

**Meta side** — `ads_get_ad_entities`, `level="adset"` (or `"ad"`/`"campaign"`),
`time_increment=1` for a daily breakdown, fields:
`["amount_spent","impressions","reach","clicks","results","cost_per_result","cpm","ctr"]`

**Tenjin side** — `get_user_acquisition_report`, `group_by="channel,app"`,
`granularity="daily"`, metrics:
`"spend,impressions,tracked_impressions,installs,tracked_installs,clicks"`

Present as one table per date, with columns for Meta-direct, Tenjin-Reported, and
Tenjin-Tracked side by side, e.g.:

| Metric | Meta (direct) | Tenjin (Reported) | Tenjin (Tracked) |
|---|---|---|---|
| Spend | $X | $X | — |
| Impressions | X | X | X |
| Installs | X | X | X |
| Clicks | X | X | — |

Expect Meta-direct and Tenjin-Reported to match closely (small gaps are normal
sync lag, especially on the current/still-accruing day). The number that
actually validates the pipeline is **Tracked** — if it's zero or much lower than
Reported, work through the reasons below before concluding anything is broken.

### Why tracked numbers can legitimately be lower than reported

In rough order of likelihood, check in this order:

1. **AMM not signed** — see Step 3. Combined with being under Tenjin's volume
   threshold (**more than 100 unique users or more than 100 tracked
   installs**), this is the most common cause of `tracked_installs` showing 0
   or near-0 despite real Meta-reported installs. Signing AMM is the fix that
   removes the dependency on volume entirely.
2. **Genuinely below the volume threshold, even with AMM** — a handful of
   installs on a brand-new campaign may simply be under the 100-unique-user/
   100-tracked-install threshold. This resolves itself as volume grows; it
   isn't a configuration problem.
3. **SKAN postback delay** — Apple deliberately randomizes postback timing, up
   to ~24–72 hours. A tracked install can show up dated to when the postback
   arrived, not the original install date — don't conclude non-attribution from
   a single day's snapshot.
4. **Callback misconfigured** — re-check `inspect_app_integration`: is
   `facebook_app_id` actually set, is the Meta Install callback `active: true`?
5. **Even at full reporting, expect some gap** — Tenjin's own docs note Meta
   may use modeled/aggregated data with different attribution windows than
   Tenjin's, so a **30–40% discrepancy between Meta-reported and Tenjin-tracked
   installs is common and expected** even once both thresholds are cleared. Not
   every gap is a problem to chase down.

Attribution happening "behind the scenes" but not being *reported* (thresholds,
delay) is a very different situation from attribution not happening at all
(misconfiguration) — don't conflate the two when explaining a gap to the user.

## Next steps checklist

Give this to the user as a concrete checklist when starting this workflow fresh:

- [ ] Confirm Meta has an ad account in `list_ad_accounts` (Tenjin) — connect via Configure →
      Channels → Add a Channel if not
- [ ] Confirm the app has a Facebook App ID (developers.facebook.com), is linked
      in Business Manager, and shows up in Events Manager as a data source
- [ ] **Sign AMM terms** (https://developers.facebook.com/advanced_mobile_measurement/terms)
      — do this before judging whether tracking works
- [ ] Set `facebook_app_id` + activate the Meta Install callback in Tenjin
- [ ] Confirm Page-level Advertiser permission before creating creatives
      (`ads_get_user_pages`)
- [ ] Check AEM/SKAN eligibility in Events Manager before setting
      `is_skadnetwork_attribution`
- [ ] Build campaign → ad set → ad, all paused; confirm structure with the user
      before activating
- [ ] Activate, verify actual status via `ads_get_ad_entities` if the API
      activation call errors
- [ ] Pull combined reporting daily — always separate Reported vs. Tracked, and
      read `references/troubleshooting.md` before concluding attribution is
      broken

## Reference files

- `references/troubleshooting.md` — known tool bugs/quirks encountered building
  this workflow (broken tools, workarounds, error messages to recognize verbatim)
