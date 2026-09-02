# Backend verification

This is the part that turns "the client says they paid" into an entitlement you can trust.
The language does not matter; the shape does.

## Contents
- One endpoint
- Apple
- Google
- Storing entitlement
- Idempotency and failure

## One endpoint

`POST /verify-subscription` should:

1. Read the authenticated user from the request token. **Never** take a user ID from the body.
2. Read the platform and the purchase proof from the body.
3. Call Apple or Google's server API with that proof.
4. Decide active/inactive from the store's answer plus the expiry date.
5. Write **one** entitlement record for that user.
6. Return the entitlement to the client.

```ts
serve(async (req) => {
  if (req.method !== "POST") return json({ error: "Method not allowed" }, 405);

  const userId = await readUserIdFromAuthToken(req.headers.get("authorization"));
  if (!userId) return json({ error: "Invalid auth token" }, 401);

  const body = await req.json();

  let result: VerificationResult | null = null;
  if (body.platform === "ios") {
    result = await verifyAppleSubscription(body.original_transaction_id);
  } else if (body.platform === "android") {
    result = await verifyGoogleSubscription(body.purchase_token);
  } else {
    return json({ error: "Unsupported platform" }, 400);
  }

  if (!result) return json({ error: "Verification failed" }, 400);

  const isActive = result.isActive && result.expiresAt > new Date();

  await db.users.update(userId, {
    is_premium: isActive,
    subscription_platform: body.platform,
    subscription_product_id: result.productId,
    subscription_id: result.subscriptionId,
    premium_expires_at: result.expiresAt.toISOString(),
  });

  return json({ is_active: isActive, expires_at: result.expiresAt.toISOString() });
});
```

`json`, `db`, and `readUserIdFromAuthToken` are your own helpers; the parts worth copying are
the ordering and the single write at the end.

## Apple

Use the **App Store Server API**, endpoint *Get All Subscription Statuses*, keyed by the
**original transaction ID**. It returns the latest transaction and renewal info for the whole
subscription group, which is what you want: it answers "what is true now", not "what happened once".

Authentication is a short-lived ES256 JWT signed with your `.p8` key:

```ts
const jwt = await new SignJWT({
  iss: APPLE_ISSUER_ID,
  iat: now,
  exp: now + 20 * 60,          // Apple rejects tokens longer than 60 minutes
  aud: "appstoreconnect-v1",
  bid: APPLE_BUNDLE_ID,
})
  .setProtectedHeader({ alg: "ES256", kid: APPLE_KEY_ID, typ: "JWT" })
  .sign(key);
```

Two hosts, and **you must try both**:

```
https://api.storekit.itunes.apple.com          (production)
https://api.storekit-sandbox.itunes.apple.com  (sandbox and TestFlight)
```

Call production first; on `404` fall back to sandbox. TestFlight builds produce sandbox
transactions even when your app is live, so a production-only implementation appears to work
until your own QA team tests it.

The response fields are JWS-encoded (`signedTransactionInfo`, `signedRenewalInfo`). Decoding the
payload is enough to read it; verifying the x5c certificate chain is what makes it trustworthy,
and Apple's server libraries do that for you. Rolling your own decode is fine for reading, but
because you fetched the data from Apple over TLS, the chain check is belt-and-braces rather than
the security boundary.

`status` is numeric:

| status | Meaning | Grant access? |
|---|---|---|
| 1 | Active | yes |
| 2 | Expired | no |
| 3 | In billing retry | usually yes |
| 4 | In billing grace period | yes |
| 5 | Revoked | no, immediately |

## Google

Use `purchases.subscriptionsv2.get` with the purchase token, authenticated with a service account
(scope `https://www.googleapis.com/auth/androidpublisher`).

The single most common mistake: **the expiry is on the line item, not the top level.**

```ts
const lineItem = data.lineItems?.[0];
const expiresAt = new Date(lineItem?.expiryTime);
```

`subscriptionState` values you care about:

| State | Grant access? |
|---|---|
| `SUBSCRIPTION_STATE_ACTIVE` | yes |
| `SUBSCRIPTION_STATE_IN_GRACE_PERIOD` | yes |
| `SUBSCRIPTION_STATE_ON_HOLD` | no (payment failed, still recoverable) |
| `SUBSCRIPTION_STATE_PAUSED` | no |
| `SUBSCRIPTION_STATE_CANCELED` | yes until `expiryTime` - cancelled is not expired |
| `SUBSCRIPTION_STATE_EXPIRED` | no |

`SUBSCRIPTION_STATE_CANCELED` catches people out: the user turned off auto-renew but has paid
through the end of the period. Cutting them off early is both wrong and a support ticket.

## Storing entitlement

One row, one source of truth. Both the verify endpoint and the webhook handler write to the
*same* record - the moment there are two places, they disagree.

Minimum useful columns:

```
user_id, platform, product_id, subscription_id (original txn id / purchase token),
is_active, expires_at, environment (sandbox|production), updated_at
```

Store `environment`. Sandbox purchases leaking into production entitlements is a classic
free-premium exploit, and without the column you cannot even audit for it.

Check `expires_at > now()` on **read**, not only on write. If a webhook is missed, an expiry
check on read fails closed instead of granting access forever.

## Idempotency and failure

- Key writes by `(user_id, subscription_id)` so a replayed request is harmless.
- If the store API times out, return an error the client can retry. Do **not** grant access
  optimistically.
- Log every verification failure with the product ID and platform. A spike is usually either an
  expired API key or someone probing your endpoint.
- Rate-limit the endpoint. It takes an unauthenticated-ish payload and calls out to a third party.
