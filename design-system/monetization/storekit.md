# StoreKit 1/2 và Subscription Lifecycle

### Định nghĩa

StoreKit bán digital content/services. Consumable dùng hết, non-consumable mua một lần, auto-renewable tự gia hạn, non-renewing có thời hạn do app quản lý. StoreKit 1 dùng SKProductsRequest/SKPaymentQueue observer/receipt; StoreKit 2 (iOS 15+) dùng Product, Transaction, async/await, signed transaction verification.

### Purchase flow

```swift
import StoreKit

@MainActor
final class PremiumStore {
    private(set) var activeIDs = Set<String>()

    func refresh() async {
        var ids = Set<String>()
        for await result in Transaction.currentEntitlements {
            if case .verified(let transaction) = result {
                ids.insert(transaction.productID)
            }
        }
        activeIDs = ids
    }

    func purchase(_ product: Product) async throws {
        switch try await product.purchase() {
        case .success(let result):
            guard case .verified(let transaction) = result else {
                throw PurchaseError.unverified
            }
            await refresh()
            await transaction.finish()
        case .pending, .userCancelled:
            break
        @unknown default:
            break
        }
    }

    func restore() async throws {
        try await AppStore.sync()
        await refresh()
    }
}

enum PurchaseError: Error { case unverified }
```

Ví dụ non-consumable/subscription; không xử lý consumable balance. Production lọc product IDs, persist/fulfill idempotent, báo unverified/error/pending. Listener Transaction.updates từ launch, verify/deliver/finish sau delivery; xử lý Ask to Buy, renewal/refund, cancel listener theo lifetime. Không chỉ lưu isPremium=true trong UserDefaults.

### Subscription lifecycle

* Product.products(for:), displayPrice theo locale/kỳ hạn; trial/offer eligibility không đoán.
* Subscription group, upgrade/downgrade, expiration, billing retry, grace period (nếu cấu hình), cancel renew không đồng nghĩa mất quyền ngay, revocation/refund.
* currentEntitlements và subscription status/renewal info; server dùng App Store Server API/Notifications V2, verify JWS, dedup transaction/notification, reconcile out-of-order; không tin client boolean.
* Restore user-initiated AppStore.sync; currentEntitlements cập nhật bình thường, không bắt mua lại khi reinstall.
* StoreKit 1 observer đăng ký sớm, purchased/restored/failed/deferred, finishTransaction sau delivery; receipt verification backend theo flow cũ.
* Test Xcode StoreKit Configuration, sandbox, TestFlight: purchase, cancel, pending, renewal, expiry, refund, offline, reinstall. Không dùng production receipt trong môi trường test.

### Q&A

**Q: Vì sao finish sau delivery?** Đảm bảo không mất quyền nếu app crash; xử lý transaction ID idempotent khi redelivery.

**Q: User tắt auto-renew có mất premium?** Không, còn hiệu lực tới expiration, trừ revocation hoặc điều kiện khác.

**Q: Restore khác purchase?** Restore đồng bộ quyền đã mua, không tạo giao dịch tính tiền mới.

**Liên hệ:** Call ID, FastVPN, Watch Faces; CV chưa xác định StoreKit version.

[StoreKit](https://developer.apple.com/documentation/storekit)
