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

## StoreKit chuyên sâu

### StoreKit 1 vs StoreKit 2

| Chức năng | StoreKit 1 | StoreKit 2 |
| --- | --- | --- |
| Load product | SKProductsRequest delegate | Product.products(for:) async |
| Buy | SKPaymentQueue.add | Product.purchase() |
| Observe | SKPaymentTransactionObserver | Transaction.updates |
| Verification | Receipt/backend | VerificationResult signed transaction |
| Restore | restoreCompletedTransactions | AppStore.sync + currentEntitlements |
| Finish | finishTransaction | Transaction.finish() |

StoreKit 2 iOS 15+. Entitlement là quyền sử dụng hiện tại, transaction là giao dịch, renewal status là thông tin kỳ gia hạn. Auto-renew off không đồng nghĩa expired.

### Transaction listener

```swift
import StoreKit

@MainActor
final class PurchaseObserver {
    private var task: Task<Void, Never>?

    func start(deliver: @escaping @MainActor (Transaction) async -> Bool) {
        guard task == nil else { return }
        task = Task {
            for await result in Transaction.updates {
                guard !Task.isCancelled else { break }
                guard case .verified(let transaction) = result else { continue }
                if await deliver(transaction) {
                    await transaction.finish()
                }
            }
        }
    }

    func stop() {
        task?.cancel()
        task = nil
    }

    func products(ids: Set<String>) async throws -> [Product] {
        try await Product.products(for: ids)
    }
}
```

Start listener sớm theo app lifetime. deliver idempotent theo transaction ID + persisted ledger/server, verify product ID/revocation/expiry/upgrade, cập nhật entitlement; chỉ finish sau delivery. Pending/Ask to Buy không cấp quyền. Consumable cần balance ledger, currentEntitlements không thay consumable tracking.

### Subscription state

```text
purchase -> active -> renewal -> active
                  -> cancel auto-renew -> active đến expiry
                  -> billing retry / grace -> recover / expire
                  -> refund/revocation -> mất quyền theo verified state
```

Upgrade/downgrade/crossgrade theo subscription group/store rules. Không tự tính expiration bằng ngày mua + 30 ngày. Trial/intro eligibility theo StoreKit; Product.displayPrice theo locale. currentEntitlements + subscription status/renewal info; server authoritative với backend service, verify JWS từ App Store Server Notifications V2, dedup notification UUID/transaction ID, reconcile out-of-order và query Server API.

### Restore, offline, lỗi

Restore button user chủ động gọi AppStore.sync, refresh entitlement; không sync mỗi launch gây prompt. Offline dùng verified local state theo policy/expiry, reconnect reconcile. UserCancelled là expected outcome, pending show processing, unverified không cấp quyền. Paywall ghi kỳ hạn/giá/renewal/terms/privacy/restore.

### Test matrix và Q&A

Xcode StoreKit config local, sandbox end-to-end, TestFlight sandbox. Test success/cancel/pending, crash trước finish, duplicate, renewal/expiry/refund, grace/billing retry, reinstall, account switch, network fail, server out-of-order.

**Q: Crash sau deliver trước finish?** Transaction có thể redeliver, idempotent ledger tránh cấp hai lần.

**Q: Tắt auto-renew có mất premium?** Không ngay, vẫn active đến expiry trừ revocation/điều kiện khác.

**Q: StoreKit vs VNPay?** Digital goods theo App Store rules; hàng hóa/dịch vụ ngoài app dùng gateway phù hợp, kiểm tra guideline/region.

**Bài tập:** StoreKit config subscription + non-consumable, paywall/listener, test restore/refund/expiry và duplicate delivery.
