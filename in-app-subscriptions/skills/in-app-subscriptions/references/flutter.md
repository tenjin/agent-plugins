# Flutter

The first-party `in_app_purchase` plugin wraps StoreKit and Play Billing behind one API. The
platform-specific packages (`in_app_purchase_storekit`, `in_app_purchase_android`) are where you
reach when you need something the common API does not expose.

## The mistake almost everyone makes first

Listening to `purchaseStream` inside a paywall widget. The stream delivers purchases that complete
after that widget is disposed - interrupted purchases, Ask to Buy approvals, renewals at launch -
and those updates go nowhere. Listen **once**, app-wide, and keep the subscription alive for the
process lifetime.

```dart
final iap = InAppPurchase.instance;

Future<List<ProductDetails>> loadProducts() async {
  if (!await iap.isAvailable()) {
    throw Exception('In-app purchases are not available');
  }

  const ids = {'premium_monthly', 'premium_yearly'};
  final response = await iap.queryProductDetails(ids);

  if (response.notFoundIDs.isNotEmpty) {
    debugPrint('Products not found: ${response.notFoundIDs}');
  }
  return response.productDetails;
}
```

```dart
late final StreamSubscription<List<PurchaseDetails>> _purchaseSub;
final _processed = <String>{};

void startPurchaseListener() {
  _purchaseSub = InAppPurchase.instance.purchaseStream.listen(
    _handlePurchaseUpdates,
    onError: (error) => debugPrint('Purchase stream error: $error'),
  );
}

Future<void> buy(ProductDetails product) async {
  final param = PurchaseParam(productDetails: product);
  await InAppPurchase.instance.buyNonConsumable(purchaseParam: param);
}

Future<void> _handlePurchaseUpdates(List<PurchaseDetails> purchases) async {
  for (final purchase in purchases) {
    switch (purchase.status) {
      case PurchaseStatus.pending:
        // Show pending UI. Do not unlock yet.
        break;

      case PurchaseStatus.error:
      case PurchaseStatus.canceled:
        if (purchase.pendingCompletePurchase) {
          await InAppPurchase.instance.completePurchase(purchase);
        }
        break;

      case PurchaseStatus.purchased:
      case PurchaseStatus.restored:
        final key = purchase.purchaseID ??
            '${purchase.productID}:${purchase.transactionDate}';

        if (_processed.contains(key)) {
          if (purchase.pendingCompletePurchase) {
            await InAppPurchase.instance.completePurchase(purchase);
          }
          break;
        }

        final result = await backend.verifySubscription(purchase);
        _processed.add(key);

        if (purchase.pendingCompletePurchase) {
          await InAppPurchase.instance.completePurchase(purchase);
        }
        setPremium(result.isActive);
        break;
    }
  }
}
```

## Notes

- **`buyNonConsumable` is correct for subscriptions.** `buyConsumable` is for consumables only;
  using it on a subscription lets the same purchase be bought repeatedly.
- **`completePurchase` maps to `finish()` / `acknowledge()`.** Skipping it means iOS replays
  forever and Google Play refunds the purchase after three days. Call it on the error and
  already-processed paths too, or the queue never drains.
- **De-duplicate.** iOS sandbox restores replay a long history, and `purchaseStream` re-emits at
  launch. Without the `_processed` set you will hammer your verify endpoint.
- **`restorePurchases()`** re-emits everything through the same stream with status `restored`.
  Because the same handler runs, restore is nearly free once the purchase path is right.
- **Getting the raw proof for the backend:** cast to the platform type.
  `purchase as AppStorePurchaseDetails` exposes `skPaymentTransaction` / the StoreKit 2
  transaction, and `purchase as GooglePlayPurchaseDetails` exposes `billingClientPurchase`
  (`purchaseToken`, `originalJson`, `signature`).
- **`purchase.verificationData.serverVerificationData`** is the convenient cross-platform handle:
  the receipt/JWS on iOS, the purchase token on Android. It is enough for most backends.
