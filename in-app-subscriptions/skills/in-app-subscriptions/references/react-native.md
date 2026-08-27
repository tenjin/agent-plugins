# React Native and Expo

Two mainstream options, and the choice matters less than getting the server side right.

| | `react-native-iap` | `expo-in-app-purchases` |
|---|---|---|
| Maintenance | Actively maintained, largest install base | Deprecated by Expo; do not start new work on it |
| Expo Go | No (needs a dev build) | No |
| Recommendation | Use this, with an Expo dev build or bare RN | Migrate away |

Neither works in Expo Go. Subscriptions need native modules, so a development build
(`expo prebuild` / EAS Build) is required. This surprises Expo teams late, so raise it early.

## Shape with react-native-iap

```ts
import {
  initConnection,
  getSubscriptions,
  requestSubscription,
  finishTransaction,
  purchaseUpdatedListener,
  purchaseErrorListener,
  getAvailablePurchases,
} from 'react-native-iap';

const SKUS = Platform.select({
  ios: ['com.yourapp.premium.monthly'],
  android: ['premium_monthly'],
})!;

// Register listeners once, at app root - not inside the paywall screen.
export function initIAP() {
  await initConnection();

  const updateSub = purchaseUpdatedListener(async (purchase) => {
    const proof =
      Platform.OS === 'ios'
        ? { transactionId: purchase.transactionId,
            originalTransactionId: purchase.originalTransactionIdentifierIOS,
            jws: purchase.transactionReceipt }
        : { purchaseToken: purchase.purchaseToken,
            productId: purchase.productId };

    const { isActive } = await api.verifySubscription(Platform.OS, proof);
    setPremium(isActive);

    // Required. iOS replays unfinished transactions; Play refunds
    // unacknowledged Android purchases after three days.
    await finishTransaction({ purchase, isConsumable: false });
  });

  const errorSub = purchaseErrorListener((e) => log.warn('iap error', e));

  return () => { updateSub.remove(); errorSub.remove(); };
}

export async function buy(sku: string) {
  const subs = await getSubscriptions({ skus: [sku] });
  const sub = subs[0];

  await requestSubscription({
    sku,
    // Android requires an explicit offer token; without it the call fails.
    ...(Platform.OS === 'android' && {
      subscriptionOffers: sub.subscriptionOfferDetails.map((o) => ({
        sku,
        offerToken: o.offerToken,
      })),
    }),
  });
}
```

## Notes

- **Register listeners at the app root.** Purchases that complete after a screen unmounts are
  exactly the ones you must not lose.
- **`finishTransaction` is mandatory** on both platforms, and `isConsumable: false` for subscriptions.
- **Android needs `subscriptionOffers`.** The API version differences here are the single most
  common source of "it works on iOS" bug reports.
- **Restore** is `getAvailablePurchases()`, then send each to your verify endpoint. Run it on
  login and launch, not only behind a button.
- **The API surface has changed across major versions** of `react-native-iap`. Check the installed
  version's docs before copying any snippet, including this one.
- Everything in `references/backend-verification.md` applies unchanged. The React Native layer is
  only a transport for the proof.
