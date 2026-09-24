# Known tool quirks and bugs

Things discovered the hard way building this workflow the first time. Check here
before assuming a new error is a dead end or spending a long time re-diagnosing
something already understood.

## Meta Ads MCP

### `ads_create_ad`'s `adset_spec` parameter
Documented as a way to create an ad and its ad set inline in one call. In
practice, it has been observed to fail and demand a real, pre-existing
`ad_set_id` regardless of whether `adset_spec` is populated. If this happens,
fall back to creating the ad set separately (see below), then pass its real ID
to `ads_create_ad`.

### `ads_create_ad_set` visibility
This tool has been observed to be **undiscoverable via `tool_search`** in some
sessions — searching directly for its name returned nothing, even though other
sibling tools (`ads_create_campaign`, `ads_create_ad`) loaded fine on the first
try. In other sessions it has loaded correctly and worked as documented,
including enforcing real business rules (e.g. rejecting an ad set under an
archived campaign, requiring an `end_time` for lifetime-budget campaigns,
requiring enough total campaign budget to cover all its ad sets). Retry
`tool_search` with a couple of different phrasings before concluding it's
unavailable. If it's genuinely not loading, fall back to:
1. Give the user the Ads Manager link for the campaign
2. Have them create the ad set there manually (targeting, optimization goal, no
   budget if CBO), and **save/publish it** — a draft-only ad set in Ads Manager
   does not get a real, API-queryable ID until it's actually saved
3. Get the resulting ad set ID from the URL (`selected_adset_ids=`) and pass it
   to `ads_create_ad` directly

### `ads_update_entity` and deletion
Setting `{"status": "DELETED"}` on a campaign does not delete it — the tool has
a built-in safety guard that silently forces the status to `PAUSED` instead
(response includes `status_forced_to_paused: true`). There is no working
deletion path via these tools. To actually delete a campaign, the user needs to
do it manually in Ads Manager. A paused-but-not-deleted campaign is harmless to
leave around, but note it can become "archived" over time, which will then
reject any attempt to add new ad sets to it (`"Campaign Archived"`).

### `ads_activate_entity` vs. Ads Manager UI
Observed case: the API rejected activation of a campaign with an AEM
ineligibility error, while the user was able to successfully activate the exact
same campaign, ad set, and ad directly in the Ads Manager UI moments later
(confirmed via `ads_get_ad_entities` showing genuine `ACTIVE`/`ACTIVE` status,
not just a UI-side illusion). Takeaway: if API activation fails and you believe
eligibility/configuration is otherwise correct, don't assume the campaign is
permanently blocked — have the user try activating in the UI and verify the
real resulting status via a read call rather than trusting the API's rejection
as final.

### AEM/SKAN eligibility errors on edit, not just creation
A campaign created successfully with `is_skadnetwork_attribution: false` (AEM)
can still fail on *any subsequent edit* — even a budget-only change — with the
same ineligibility error, despite Events Manager's "Check app eligibility" tool
showing the app as eligible. This looks like the campaign carries a stale
eligibility snapshot from its original creation time rather than Meta
re-checking live status on edit. No field combination has been found to fix an
already-affected campaign. The reliable fix is to create a **new** campaign
(new `campaign_id`) after eligibility is confirmed, rather than continuing to
edit the old one. Reuse the existing creative — that doesn't need to be rebuilt.

### Object Store URL Mismatch (error #1885093)
Caused by a mismatch between the app's registered Store ID (set only in the App
Dashboard, under Settings → Basic → the platform section, commonly labeled
**"iPhone Store ID"** — a bare numeric ID) and the `object_store_url` passed in
`promoted_object`. This field cannot be set or read via the Graph API at all —
it's UI-only. If Bundle ID and Store ID both check out in the App Dashboard and
the error persists, check whether the app's **App Mode** (Development vs. Live)
matters for the specific case, and whether the person editing has access to the
correct Business Manager context — Events Manager and the App Dashboard can
each show different app states depending on which ad account/business is
currently selected in the account switcher (top right). Always confirm that
selector matches the ad account actually running the campaign before debugging
further.

### SKAdNetwork Not Ready (error #2446981)
Usually means the app was never registered as a data source in **Events
Manager** at all (distinct from being linked in Business Manager). Add it via
Events Manager → Connect Data Sources → App. For a brand-new app with zero
event volume, also check the **"Meta's attribution for iOS 14+"** section on
the app's Settings tab for an eligibility acknowledgment/override — a new app
with no events yet often needs this explicit override before SKAN/AEM will
accept it at all.

### Page permission errors on creative creation
`"No permission to access this profile"` when calling `ads_create_creative`
means the Page is *linked to the ad account* but the calling user/token doesn't
have Advertiser-level access to that specific Page. Check with
`ads_get_user_pages` (empty result = no usable Pages) rather than
`ads_get_ad_account_pages` (which will still show the Page as linked,
misleadingly suggesting it should work). Fix in Business Settings → Accounts →
Pages → assign the right role to the relevant person/system user, then retry.

## Tenjin MCP

### `search_metrics` result
`installs` in Tenjin's own metric catalog is explicitly labeled **"Reported
Installs"** — worth checking `search_metrics` directly if a metric name's
actual meaning is ever unclear, rather than assuming a plain-sounding name means
what it sounds like.

### Transient "No approval received" errors
Some Tenjin MCP write calls (`update_callback_settings`,
`ads_account_get_activity_logs` equivalents) have intermittently failed with a
bare `"No approval received"` error, then succeeded on an identical retry
moments later with no changes to the request. Worth one immediate retry before
concluding a call is broken — but if it fails consistently across multiple
retries, treat it as a real, non-transient failure and say so rather than
continuing to retry blindly.
