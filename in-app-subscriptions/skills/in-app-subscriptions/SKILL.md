---
name: in-app-subscriptions
description: Use when adding, debugging, or reviewing paid subscriptions or in-app purchases in a mobile or desktop app — StoreKit / StoreKit 2, Google Play Billing, in_app_purchase, react-native-iap, expo-in-app-purchases, or a vendor SDK such as RevenueCat, Adapty, or Qonversion. Covers store product setup, the client purchase flow, server-side receipt verification, entitlement storage, store webhooks, restore, and sandbox testing. Applies even when the request sounds narrow — "why is my subscription not renewing in sandbox", "add a paywall", "verify a receipt on my server", "handle refunds", "premium keeps unlocking for free users", "acknowledge purchase", "restore purchases" — because those symptoms almost always come from a missing piece of the wider flow.
---

# In-app subscriptions

Subscriptions look like one button and are actually a distributed system: a store the user pays,
a device that learns about it, a server that has to be convinced, and a database that answers
"is this person paid?" long after the purchase. Most subscription bugs are not billing-API bugs.
They are bugs in how those four parts were wired together.

## The one rule everything else follows from

**The device asks. The server decides.**

A client can tell you a purchase happened. It cannot be the thing that decides what the user
paid for, because a client can be replayed, patched, or simply wrong (a stale cache, a failed
network call, a reinstall). Every design choice below falls out of that split.

## The flow

```
1. App queries products from the store by product ID
2. User buys through the native purchase sheet
3. Store returns purchase proof (signed transaction / purchase token)
4. App sends that proof to your backend
5. Backend verifies the proof with Apple or Google's server API
6. Backend writes one entitlement record
7. App reads entitlement from the backend, not from local state
8. Store webhooks keep that record correct while the app is closed
```

Steps 5, 6, and 8 are the ones people skip, and they are the ones that cause the expensive bugs.

## Four failure modes worth naming up front

Bring these up when reviewing someone's implementation, because they are rarely visible in
a happy-path demo:

- **Unlocking from the client callback alone.** Works in testing, gets bypassed in production.
- **Not finishing / acknowledging the transaction.** iOS replays the transaction forever;
  Google Play auto-refunds and revokes Android purchases that go unacknowledged
  (3 days for subscriptions).
- **Entitlement living only in app state.** The user reinstalls, or opens on a second device,
  and their subscription is gone.
- **No webhook handling.** Renewals, cancellations, refunds, grace periods, and billing retries
  all happen while the app is closed. Without webhooks, your database drifts from reality
  and you either bill-block paying users or serve free access to churned ones.

## How to use this skill

Start by working out where the person actually is, since the answer changes which reference
file matters:

| Situation | Read |
|---|---|
| Products don't exist yet, or don't come back from a query | [references/store-setup.md](references/store-setup.md) |
| Writing the purchase flow on iOS | [references/ios-storekit2.md](references/ios-storekit2.md) |
| Writing the purchase flow on Android | [references/android-play-billing.md](references/android-play-billing.md) |
| Flutter app | [references/flutter.md](references/flutter.md) |
| React Native or Expo app | [references/react-native.md](references/react-native.md) |
| Any backend, any language | [references/backend-verification.md](references/backend-verification.md) |
| Renewals, refunds, cancellations, grace periods | [references/store-notifications.md](references/store-notifications.md) |
| Considering RevenueCat / Adapty / a billing vendor | [references/vendor-vs-diy.md](references/vendor-vs-diy.md) |
| Connecting subscription revenue to ad campaigns | [references/attribution-mmp.md](references/attribution-mmp.md) |
| Nothing works in sandbox and nobody knows why | [references/testing-and-debugging.md](references/testing-and-debugging.md) |

Read only what the task needs. These files are self-contained and assume the mental model above.

## Working with someone's existing code

When you are dropped into a codebase that already has purchases, resist rewriting it. Read the
existing flow first and answer these five questions - the answers tell you what is actually broken
far faster than reading every file:

1. Where does `isPremium` (or its equivalent) get its value? If the answer is anywhere on the
   device, that is the bug.
2. Is every purchase finished / completed / acknowledged, including the error and already-owned paths?
3. Is there a server endpoint that calls Apple's or Google's API, or does the server just trust
   the body it was posted?
4. Is there a webhook endpoint, and does it write to the *same* row the purchase path writes to?
5. What happens on reinstall?

## Product IDs

Product IDs are the join key between the store, your client, and your database. They cannot be
changed after creation on either platform, so they are worth five minutes of thought.

- Keep them stable and boring: `com.yourapp.premium.monthly`, `premium_monthly`.
- They do not have to be identical across iOS and Android, but your backend should map both to
  the same internal plan, so entitlement logic never branches on platform.
- Never encode price in the ID. Prices change; IDs cannot.

## Deciding what "has access" means

Both stores expose more states than "active" and "expired", and the states you honor are a
product decision, not a technical one. A reasonable default, and the one to suggest unless the
person says otherwise:

**Grant access** while the subscription is active, in a billing-retry period, or in a grace
period, as long as the stored expiry is still in the future.
**Revoke immediately** on refund or revocation, regardless of expiry.

Store the expiry date, and check it on read. A subscription that "should" be active but expired
three days ago means something upstream failed, and failing closed is the safer default.

## Definition of done

Use this as the review checklist before anyone ships:

- [ ] Products created and **activated** in both stores (inactive Play base plans return nothing)
- [ ] Product IDs mapped to internal plans server-side
- [ ] Sandbox testers (iOS) and license testers (Android) created and tested on real devices
- [ ] Client handles success, cancel, error, pending, deferred, restore, and replayed purchases
- [ ] Every purchase finished / completed / acknowledged after access is delivered
- [ ] Server verifies with the App Store Server API and the Play Developer API
- [ ] Entitlement stored in exactly one place
- [ ] Webhooks configured and writing to that same place
- [ ] Restore tested on a second device with the same store account
- [ ] Behavior defined for verification failure (fail closed, log, retry)
- [ ] Upgrade / downgrade / crossgrade paths tested if there is more than one plan

## Things that are worth saying out loud to the user

- **Sandbox is not production.** iOS sandbox subscriptions renew on an accelerated clock
  (a month becomes minutes) and can replay a long history of past renewals on restore. Code that
  does not de-duplicate looks broken in sandbox and fine in production, or the reverse.
- **The first subscription needs an app version.** On iOS, the first subscription group must be
  submitted along with a new app version, which surprises people mid-sprint.
- **Test on device.** Neither store's purchase flow behaves correctly in a simulator.
