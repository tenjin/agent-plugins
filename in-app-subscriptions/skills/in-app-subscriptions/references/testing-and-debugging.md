# Testing and debugging

## Nothing comes back from the product query

In rough order of likelihood:

1. The product is not **active** (Play base plan not activated; Apple product in "Missing Metadata").
2. The build's bundle ID / package name does not match the store listing.
3. On Android, the app has never been published to a track - internal testing counts, nothing does not.
4. On Android, the installed build is not signed with the same key as the uploaded one.
5. Product IDs differ by a character between code and console.
6. The paid applications agreement is not active on the account.
7. Play permission changes were made minutes ago; they can take hours to propagate.
8. iOS: you are on a simulator. Use a device, or a local StoreKit configuration file.

On Android, log `queryProductDetailsResult.unfetchedProductList` - it names the IDs Play could not
resolve, which turns most of the above into a one-minute fix.

## Sandbox behaviors that look like bugs

| Symptom | Cause |
|---|---|
| Subscription renews every 5 minutes | Expected. iOS sandbox compresses a month into minutes and auto-renews ~6 times before stopping. |
| Restore delivers dozens of transactions, most expired | Expected. Sandbox replays history. De-duplicate, and still `finish()` each one. |
| Purchases work in TestFlight but server verification 404s | TestFlight is sandbox. Your server must fall back to the sandbox host. |
| Android purchase disappears after a few days | It was never acknowledged, so Play refunded and revoked it. |
| Purchase sheet asks to sign in repeatedly | Sandbox tester signed in via Settings instead of at the purchase prompt. |
| Everything works, then stops for one tester | Sandbox account consumed its renewal allotment. Create a new tester. |

## A test matrix worth actually running

Not exhaustive, but these are the cases that break in production:

- Buy → verify → premium unlocked
- Buy → kill the app before the callback → relaunch → access restored without a second charge
- Buy → airplane mode during verification → recovers on reconnect, no double charge
- Restore on a second device with the same store account
- Reinstall → restore
- Cancel → access persists until the period ends → expires on time
- Refund (Apple: request via support; Google: refund in Play Console) → access revoked
- Upgrade monthly → yearly → one active subscription, not two
- Ask to Buy (iOS) → pending → approved later
- Pending payment (Android cash/carrier) → not granted until purchased

## Making the loop fast

- **iOS:** Xcode StoreKit configuration file. Renewals, failures, refunds and Ask to Buy can all
  be simulated locally, in seconds, without App Store Connect. It does not exercise server
  verification, so pair it with sandbox for the end-to-end path.
- **Android:** license testers plus the internal testing track. Static response test product IDs
  (`android.test.purchased`) still exist but do not produce a real purchase token, so they cannot
  test verification.
- **Both:** a `/verify-subscription` integration test that replays a captured store response is
  worth more than any amount of manual tapping, and it runs in CI.

## Logging that pays for itself

Log, for every verification: platform, product ID, subscription ID, store response status,
computed `is_active`, computed `expires_at`, and environment. When a user says "I paid and it's
gone", that one line answers it. Redact the purchase token and signed payload.
