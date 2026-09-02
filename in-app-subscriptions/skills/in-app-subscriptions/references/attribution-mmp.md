# Attribution with an MMP

Billing answers one question: *does this user have access?* Attribution answers a different one:
*which campaign brought this subscriber in?*

The stores tell you a subscription happened. They do not tell you whether the user arrived from a
Google campaign, a TikTok ad, an influencer link, or organic search. A mobile measurement partner
(AppsFlyer, Adjust, Tenjin, Branch, Singular) closes that gap, and it starts to matter the moment
there is paid acquisition spend.

This is optional. An app with no ad spend does not need it, and adding it before entitlement is
solid is a way of building two broken things instead of one working one.

## Contents
- The ordering that keeps this clean
- What to send
- Worked example: Tenjin
- Mistakes worth catching in review

## The ordering that keeps this clean

**Verify, entitle, then track.**

Fire the MMP event only after your backend has confirmed the purchase. Two reasons, and the second
is the expensive one:

1. The event is for reporting. It must never be in the path that decides access.
2. Ad networks optimize toward the events you send them. Fire on the client callback and you are
   teaching the network to find users who *start* purchases, including cancelled, failed, and
   fraudulent ones. The bid model degrades quietly and the CAC numbers look fine until they don't.

## What to send

You already collected everything the MMP needs during verification:

| Field | Source |
|---|---|
| Product ID | `productId` / `product.id` |
| Price and currency | The `ProductDetails` / `Product` you queried, not a hardcoded value |
| Transaction ID (iOS) | `transaction.id` and `transaction.originalID` |
| Purchase token (Android) | `purchase.purchaseToken` |
| Store payload | iOS receipt or signed JWS; Android `originalJson` + `signature` |

The store payload matters: most MMPs re-validate the purchase themselves before counting revenue,
so sending it is what turns a raw event into verified revenue in their dashboard.

Use the **real** price from the product object. Hardcoded prices break the moment a user buys in
another currency or on an intro offer, and the error shows up as inflated LTV rather than a crash.

## Worked example: Tenjin

The API shape below is Tenjin's. Every other MMP takes the same facts under different names, so
the structure transfers; check your vendor's subscription/purchase validation docs for the exact
parameter list.

### Flutter

Initialize once at startup. On iOS, call `connect()` after the ATT prompt if your app shows one,
so the SDK knows the tracking authorization state.

```dart
TenjinSDK.instance.init(apiKey: tenjinApiKey);
TenjinSDK.instance.connect();
```

After your backend returns a successful verification, forward the purchase fields. Both
branches read everything off `purchase`, so cast it to the platform type first:
`AppStorePurchaseDetails` from `in_app_purchase_storekit`, `GooglePlayPurchaseDetails`
from `in_app_purchase_android`.

```dart
Future<void> trackVerifiedSubscription(
  PurchaseDetails purchase,
  ProductDetails product,
) async {
  if (Platform.isIOS && purchase is AppStorePurchaseDetails) {
    final txn = purchase.skPaymentTransaction;

    // A first purchase has no originalTransaction; it is its own original.
    final originalId = txn.originalTransaction?.transactionIdentifier ??
        txn.transactionIdentifier;

    await TenjinSDK.instance.subscription(
      productId: product.id,
      currencyCode: product.currencyCode,
      unitPrice: product.rawPrice,
      iosTransactionId: txn.transactionIdentifier,
      iosOriginalTransactionId: originalId,
      iosReceipt: purchase.verificationData.serverVerificationData,
    );
  }

  if (Platform.isAndroid && purchase is GooglePlayPurchaseDetails) {
    final billing = purchase.billingClientPurchase;
    await TenjinSDK.instance.subscription(
      productId: billing.products.first,
      currencyCode: product.currencyCode,
      unitPrice: product.rawPrice,
      androidPurchaseToken: billing.purchaseToken,
      androidPurchaseData: billing.originalJson,
      androidDataSignature: billing.signature,
    );
  }
}
```

`iosSKTransaction` is for the StoreKit 2 signed transaction and has no StoreKit 1 equivalent,
so it is omitted above. If your app has opted into StoreKit 2, note that the plugin's
`SK2PurchaseDetails` does not currently expose the transaction identifiers at all
([flutter/flutter#116383](https://github.com/flutter/flutter/issues/116383)) - take them from
your own verification response instead, which is where the authoritative values already are.

### Native iOS

```objectivec
[TenjinSDK initialize:@"<API_KEY>"];

// On iOS, call connect after the ATT prompt if you show one.
[TenjinSDK connect];

NSDecimalNumber *price =
    [NSDecimalNumber decimalNumberWithString:@"9.99"];

// transaction is the SKPaymentTransaction you just verified.
NSString *originalId = transaction.originalTransaction.transactionIdentifier
    ?: transaction.transactionIdentifier;

[TenjinSDK subscriptionWithProductName:@"com.yourapp.premium.monthly"
                       andCurrencyCode:@"USD"
                          andUnitPrice:price
                      andTransactionId:transaction.transactionIdentifier
              andOriginalTransactionId:originalId
                      andBase64Receipt:base64Receipt
                      andSKTransaction:signedTransactionJws];
```

### Native Android

```java
TenjinSDK tenjin = TenjinSDK.getInstance(this, "<API_KEY>");
tenjin.setAppStore(TenjinSDK.AppStoreType.googleplay);
tenjin.connect();

tenjin.transaction(
    purchase.getProducts().get(0),
    currencyCode,
    1,
    unitPrice,
    purchase.getOriginalJson(),
    purchase.getSignature()
);
```

## Mistakes worth catching in review

- **Firing before verification.** Covered above; it is the one that costs money.
- **Firing on every renewal from the client.** Renewals happen while the app is closed. If you
  also fire on the next launch when the transaction replays, you double-count. Send renewal events
  from your webhook handler, where you see each renewal exactly once, or not at all.
- **Firing on restore.** A restore is not a new purchase. De-duplicate by transaction ID or
  purchase token before sending.
- **Sandbox events reaching production dashboards.** Gate on your `environment` column.
- **Treating the MMP as the entitlement store.** It is a reporting sink. If your app ever asks the
  MMP whether a user is premium, something has gone wrong.
