# Store setup

Nothing else can be tested until products exist and are active. If `queryProductDetails` comes
back empty, the cause is almost always in this file, not in the client code.

## App Store Connect

1. **Your app → Monetization → Subscriptions.**
2. **Create a subscription group.** Everything a subscriber can switch between (monthly, yearly)
   belongs in one group. A user can hold one active subscription per group, and moving between
   plans inside a group is an upgrade/downgrade rather than a second subscription. Getting this
   wrong later is painful, because subscriptions cannot move between groups.
3. **Create each subscription** with a stable product ID (`com.yourapp.premium.monthly`).
4. Set price, duration, display name, description, and at least one localization. Missing
   localization is a common reason a product does not return.
5. **Sandbox testers:** Users and Access → Sandbox → Test Accounts. These are separate Apple
   Accounts. Sign in to one when the purchase sheet asks, not in system Settings.
6. **API key:** Users and Access → Integrations → App Store Connect API. Create an In-App
   Purchase key. The backend needs the key ID, the issuer ID, and the downloadable `.p8`.
   The `.p8` downloads exactly once.

### Gotchas

- The **first** subscription group must be submitted alongside a new app version. It cannot be
  approved on its own. Plan for that in the release schedule.
- Products in "Missing Metadata" or "Developer Action Needed" will not return from a query.
- A paid-apps agreement must be active on the account or nothing returns anywhere.

## Google Play Console

1. **Your app → Monetize → Products → Subscriptions.**
2. **Create a subscription** with a stable product ID (`premium_monthly`).
3. **Add base plans.** This is the part people miss: on Play, price and billing period live on the
   *base plan*, not on the subscription. One subscription can carry several base plans.
4. **Add offers** for free trials, intro pricing, and promos. Offers hang off a base plan.
5. **Activate the base plans.** Inactive base plans return nothing from a product query, with no
   error to tell you why.
6. **Service account:** Play Console → Setup → API access. Link a Google Cloud project, create a
   service account, and grant it *View financial data* plus permission on the app. Permission
   changes can take several hours to propagate, so grant them before you need them.
7. **License testers:** Play Console → Settings → License testing. Testers buy without being
   charged and get accelerated renewals.

### Gotchas

- The app must have been published to at least a closed/internal track before billing works on a
  device, even for testers.
- The tester's device account must match a license-tester email.
- Play Billing Library **8 or later is required** for new apps and updates as of August 31, 2026.
  Check the dependency version before debugging anything else.

## Naming products

The product ID is the join key between store, client, and your database, and it is immutable on
both platforms. Keep them stable and descriptive, never encode price, and map both platforms'
IDs to one internal plan identifier server-side so entitlement logic never branches on platform.
