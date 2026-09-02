# Android with Google Play Billing

Play Billing Library **8 or later is required** for new apps and app updates as of
August 31, 2026 (extensions to November 1, 2026 on request). Before debugging anything else,
check the dependency version:

```gradle
implementation "com.android.billingclient:billing:9.1.0"
```

Versions 6 and 7 are past end of support. The migration is mostly mechanical, but three signatures
changed and silently break older sample code copied from blog posts:

| Old | Current |
|---|---|
| `enablePendingPurchases()` | `enablePendingPurchases(PendingPurchasesParams…)` |
| `queryProductDetailsAsync { result, list -> }` | `queryProductDetailsAsync { result, queryProductDetailsResult -> }` |
| `queryPurchaseHistoryAsync` | `queryPurchasesAsync` (plus the server-side Voided Purchases API) |

## Contents
- The shape of a correct implementation
- Acknowledgement is a revenue bug waiting to happen
- Choosing the offer
- Things that bite

## The shape of a correct implementation

```kotlin
class BillingManager(
    context: Context,
    private val onEntitlementChanged: (Boolean) -> Unit,
) : PurchasesUpdatedListener {

    private val productIds = listOf("premium_monthly", "premium_yearly")

    private val billingClient = BillingClient.newBuilder(context)
        .setListener(this)
        .enablePendingPurchases(
            PendingPurchasesParams.newBuilder().enableOneTimeProducts().build()
        )
        .enableAutoServiceReconnection()
        .build()

    fun connect(onReady: () -> Unit) {
        billingClient.startConnection(object : BillingClientStateListener {
            override fun onBillingSetupFinished(result: BillingResult) {
                if (result.responseCode == BillingClient.BillingResponseCode.OK) onReady()
            }
            override fun onBillingServiceDisconnected() {
                // With enableAutoServiceReconnection the library retries for you.
            }
        })
    }

    fun queryProducts(onResult: (List<ProductDetails>) -> Unit) {
        val products = productIds.map { id ->
            QueryProductDetailsParams.Product.newBuilder()
                .setProductId(id)
                .setProductType(BillingClient.ProductType.SUBS)
                .build()
        }
        val params = QueryProductDetailsParams.newBuilder()
            .setProductList(products)
            .build()

        billingClient.queryProductDetailsAsync(params) { result, queryResult ->
            if (result.responseCode == BillingClient.BillingResponseCode.OK) {
                // queryResult.unfetchedProductList tells you which IDs Play could not
                // resolve. Log it: it is usually an inactive base plan or a typo.
                onResult(queryResult.productDetailsList)
            } else {
                onResult(emptyList())
            }
        }
    }

    fun launch(activity: Activity, product: ProductDetails) {
        val offer = product.subscriptionOfferDetails?.firstOrNull() ?: return
        val productParams = BillingFlowParams.ProductDetailsParams.newBuilder()
            .setProductDetails(product)
            .setOfferToken(offer.offerToken)
            .build()
        val params = BillingFlowParams.newBuilder()
            .setProductDetailsParamsList(listOf(productParams))
            .build()

        billingClient.launchBillingFlow(activity, params)
    }

    override fun onPurchasesUpdated(result: BillingResult, purchases: List<Purchase>?) {
        if (result.responseCode != BillingClient.BillingResponseCode.OK) return
        purchases.orEmpty().forEach(::handlePurchase)
    }

    private fun handlePurchase(purchase: Purchase) {
        if (purchase.purchaseState != Purchase.PurchaseState.PURCHASED) return

        BackendClient.verifySubscription(
            platform = "android",
            productId = purchase.products.first(),
            purchaseToken = purchase.purchaseToken,
            purchaseData = purchase.originalJson,
            signature = purchase.signature,
        ) { entitled ->
            onEntitlementChanged(entitled)

            if (!purchase.isAcknowledged) {
                val params = AcknowledgePurchaseParams.newBuilder()
                    .setPurchaseToken(purchase.purchaseToken)
                    .build()
                billingClient.acknowledgePurchase(params) { ackResult ->
                    // Log failures loudly. See the warning below.
                }
            }
        }
    }
}
```

## Acknowledgement is a revenue bug waiting to happen

Google Play **automatically refunds and revokes** any subscription purchase that is not
acknowledged within three days. Not "warns" - refunds. Two consequences:

1. Acknowledge on *every* path where you deliver access, including the one where the app was
   killed mid-purchase and the purchase comes back through `queryPurchasesAsync` at next launch.
2. Never gate acknowledgement behind a backend call that can fail permanently. Deliver access,
   acknowledge, then reconcile.

Prefer acknowledging **from your backend** with the Play Developer API if you can, since your
server is more reliable than a phone in a tunnel.

## Choosing the offer

`product.subscriptionOfferDetails` is a *list*: base plan, base plan with free trial, base plan
with intro price, one per eligible offer. `firstOrNull()` is fine for a single-offer product and
wrong the moment marketing adds a trial. Pick deliberately - usually the offer with the longest
free trial the user is eligible for, which Play has already filtered for you.

## Things that bite

- **Pending transactions.** Cash and carrier billing produce `PENDING`. Do not grant access, do
  not acknowledge, and re-check via `queryPurchasesAsync`.
- **Purchases arriving outside your flow.** Always call `queryPurchasesAsync` on resume; that is
  how you catch purchases completed while the app was backgrounded or killed.
- **The app must be published to a track** (internal testing is enough) before billing works.
- **Upgrades and downgrades** need `setSubscriptionUpdateParams` with a replacement mode; without
  it the user ends up with two active subscriptions.
