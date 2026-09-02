# iOS with StoreKit 2

StoreKit 2 (iOS 15+) gives you async APIs and cryptographically signed transactions. Its local
verification (`VerificationResult`) proves the payload came from Apple and was not tampered with,
which is genuinely useful, but it does not tell you whether the subscription is still active
*right now*. That question belongs to your server.

## Contents
- The shape of a correct implementation
- Fields your backend needs
- Things that bite
- StoreKit Testing in Xcode

## The shape of a correct implementation

```swift
import StoreKit

struct VerifySubscriptionRequest: Encodable {
    let platform: String
    let productId: String
    let transactionId: String
    let originalTransactionId: String
    let signedTransaction: String
}

@MainActor
final class SubscriptionStore: ObservableObject {
    @Published private(set) var products: [Product] = []
    @Published private(set) var isPremium = false

    private let productIds = [
        "com.yourapp.premium.monthly",
        "com.yourapp.premium.yearly"
    ]
    private var updatesTask: Task<Void, Never>?

    init() {
        // Start this at launch, before any purchase. Interrupted purchases,
        // Ask to Buy approvals, and renewals arrive here.
        updatesTask = Task { [weak self] in
            for await update in Transaction.updates {
                await self?.handle(update)
            }
        }
    }

    deinit { updatesTask?.cancel() }

    func loadProducts() async throws {
        products = try await Product.products(for: productIds)
    }

    func purchase(_ product: Product) async throws {
        let result = try await product.purchase()
        switch result {
        case .success(let verification):
            await handle(verification)
        case .pending:
            // Ask to Buy or SCA. The real result arrives via Transaction.updates.
            break
        case .userCancelled:
            break
        @unknown default:
            break
        }
    }

    func restore() async {
        for await entitlement in Transaction.currentEntitlements {
            await handle(entitlement)
        }
    }

    private func handle(_ result: VerificationResult<Transaction>) async {
        guard case .verified(let transaction) = result else { return }

        let request = VerifySubscriptionRequest(
            platform: "ios",
            productId: transaction.productID,
            transactionId: String(transaction.id),
            originalTransactionId: String(transaction.originalID),
            signedTransaction: transaction.jwsRepresentation
        )

        do {
            let response = try await BackendClient.verifySubscription(request)
            isPremium = response.isActive
        } catch {
            // Fail closed. Do not unlock on a failed verification.
            isPremium = false
        }

        // Only finish after access has been delivered. Unfinished transactions
        // are replayed on every launch, forever.
        await transaction.finish()
    }
}
```

## Fields your backend needs

| Field | Why |
|---|---|
| `transaction.originalID` | The stable identifier for the *subscription*, not this renewal. Use it to query the App Store Server API and as your subscription key. |
| `transaction.id` | This specific transaction. Useful for idempotency and for MMP events. |
| `transaction.productID` | Maps to your internal plan. |
| `transaction.jwsRepresentation` | The signed payload. Send it so the server can verify independently. |

## Things that bite

- **`Transaction.updates` must be listening before the purchase completes.** Start it in `init`
  or at app launch, not inside a paywall view that may not be on screen when the transaction lands.
- **`finish()` is not optional.** Unfinished transactions replay on every launch.
- **`currentEntitlements` is the restore path**, not `Transaction.all`. `all` includes expired
  history and will happily "restore" a subscription that ended a year ago.
- **Sandbox renews fast and replays history.** A restore in sandbox can deliver dozens of past
  renewals, many expired. De-duplicate by transaction ID and finish each one anyway.
- **Ask to Buy (Family Sharing)** produces `.pending`. The purchase can complete hours later.
  If your UI treats `.pending` as failure, those users are stuck.
- **Offer codes and promotional offers** arrive through the same `Transaction.updates` stream, so
  a correct implementation gets them for free.

## StoreKit Testing in Xcode

A local `.storekit` configuration file lets you test without App Store Connect and without
sandbox accounts, including subscription renewal simulation and failure injection. It is by far
the fastest loop for client-side work. It does **not** exercise your server verification path,
because there is no real Apple transaction to verify, so use it for UI and state-machine work and
sandbox for the end-to-end path.
