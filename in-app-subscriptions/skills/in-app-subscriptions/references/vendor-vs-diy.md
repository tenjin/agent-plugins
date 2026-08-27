# Vendor or do it yourself

RevenueCat, Adapty, Qonversion and similar services sit between your app and the stores. They run
the verification, store the entitlement, handle both webhook feeds, and give you a dashboard.

This is a real decision with a real answer, and the answer is usually "use a vendor". Say so
plainly rather than defaulting to whichever the person hinted at.

## Use a vendor when

- Nobody on the team has shipped subscriptions before. The vendor encodes months of edge cases.
- You need a paywall or pricing experiments without an app release.
- You want subscription analytics (cohort retention, churn, trial conversion) and are not going to
  build them.
- You have no backend, or no appetite to run one for billing.
- Time to first revenue matters more than per-transaction cost.

## Do it yourself when

- Revenue is large enough that the percentage fee outweighs the engineering cost.
- Entitlement must live inside your own system for regulatory or architectural reasons.
- Your subscription model is unusual enough that vendor abstractions fight you.
- You already have a mature backend and webhook infrastructure.

## What does not change either way

Even with a vendor, keep these in your own head and your own code review:

- **The client still must finish/acknowledge transactions.** Vendors mostly do this for you;
  verify that yours does.
- **Your server should still be the thing your app asks "is this user paid?"** - either your own
  entitlement row synced from vendor webhooks, or the vendor SDK's cached entitlement, but one of
  them, consistently.
- **You still need to handle refunds and revocations.** Configure the vendor's webhook to your
  backend so a revoked subscription actually revokes something in your system.
- **Product setup in both consoles is still yours.** See `references/store-setup.md`.
- **Migration cost is real.** Leaving a vendor means re-verifying an existing subscriber base
  against the store APIs. Consider storing the raw purchase tokens and original transaction IDs
  yourself from day one, even with a vendor. It is cheap insurance and keeps the exit open.

## Attribution is a separate question

Billing answers "does this user have access". Attribution answers "which campaign brought this
subscriber in". An MMP (AppsFlyer, Adjust, Tenjin, Branch) covers the second, and it does not
replace entitlement logic.

The ordering that keeps this clean: **verify, entitle, then track.** See
`references/attribution-mmp.md` for the fields to send, a worked example, and the mistakes that
quietly corrupt ad-network bidding.
