# Store notifications, renewals and restore

A subscription is a long-lived thing that changes state mostly while your app is closed. Renewal,
cancellation, billing failure, recovery, refund, price-change consent, expiry - almost none of it
happens with the user staring at your paywall. Verification at purchase time gives you a correct
database for about a month. Webhooks keep it correct after that.

## The two feeds

| Platform | Feed | Configure at |
|---|---|---|
| Apple | App Store Server Notifications **V2** | App Store Connect → App Information → App Store Server Notifications |
| Google | Real-time Developer Notifications (RTDN) | Play Console → Monetize → Monetization setup → Pub/Sub topic |

Apple posts signed JWS to an HTTPS endpoint you own. Google publishes to a Pub/Sub topic; you
either push-subscribe to your endpoint or pull from the topic.

Set the **sandbox** URL as well as production on Apple's side. Otherwise your test renewals go
nowhere and the lifecycle code never runs until real users hit it.

## Normalize before you branch

Both feeds carry the same handful of facts in different shapes. Translating once, at the edge,
keeps the rest of the system platform-agnostic:

```ts
type SubscriptionEvent = {
  platform: "ios" | "android";
  subscriptionId: string;          // original transaction id | purchase token
  productId: string;
  eventType:
    | "purchase" | "renewal" | "cancellation"
    | "expiration" | "refund" | "grace_period" | "recovered";
  expiresAt?: string;
  environment: "sandbox" | "production";
};
```

Apple's `notificationType` + `subtype` pairs and Google's `subscriptionNotification.notificationType`
integers both map cleanly onto this.

## Rules for the handler

- **Write to the same entitlement row** the verify endpoint writes. Never a second table.
- **Look up the user by `subscriptionId`**, because webhooks carry no user identity. This is why
  storing the original transaction ID / purchase token at purchase time matters.
- **Re-query rather than trust the payload.** The safest handler treats a notification as
  "something changed" and calls the store API for current truth. It costs one request and removes
  a whole class of ordering bugs.
- **Acknowledge fast, process async.** Both providers retry on non-2xx. Return 200 quickly and
  queue the work.
- **Expect duplicates and out-of-order delivery.** Make handling idempotent, and ignore an event
  whose `expiresAt` is older than what you have stored.
- **Refund and revoke take effect immediately**, regardless of stored expiry.
- **Route sandbox events to non-production data**, or at minimum tag them.

## Restore

Restore does not mean "trust whatever the client found locally". It means "ask the store again,
then sync the backend".

- **iOS:** iterate `Transaction.currentEntitlements`, send the original transaction ID, let the
  backend refresh status. Do not use `Transaction.all` - it includes expired history.
- **Android:** `queryPurchasesAsync(QueryPurchasesParams…SUBS)`, send the purchase token, same
  refresh path.

Call the restore path automatically on login and on app launch, not just from a "Restore
purchases" button. Most users who lose access never find that button; they leave a one-star
review instead.

Apple requires a visible restore affordance in apps with non-consumable or subscription content,
so keep the button too.

## Reconciliation

Webhooks get missed - endpoints go down, deploys drop requests, Pub/Sub subscriptions expire. A
daily job that re-queries the store for every entitlement expiring in the next 48 hours costs
almost nothing and catches the drift before a user does.
