# Call ID – Project Deep Dive

### CV

Caller ID, call blocking, SMS filtering, number lookup, local spam data; Swift, UIKit, SwiftUI, CallKit, Message Filter Extension, StoreKit, REST API, ads/analytics/attribution, localization, App Store releases.

### Flow tham khảo

API spam dataset → normalize/sort/dedup → versioned atomic snapshot → shared container → Call Directory reload → system matching. SMS query → Message Filter → local classification hoặc system network deferral → action. App search riêng gọi REST API, không đồng nghĩa call extension lookup real-time.

[CallKit](../ios-system-frameworks/callkit.md) · [Message Filter](../ios-system-frameworks/message-filter.md) · [App Groups](../data-stroage/file-cache.md)

### Edge cases và trade-off

* Extension disabled: hướng dẫn Settings, check enabled, UI không báo active sai.
* Reload fail/duplicate/out-of-order/memory: validate snapshot, last-known-good, streaming, incremental nếu biết version.
* SMS false positive: .none khi không chắc, test nhiều ngôn ngữ, không log body/number.
* Subscription quyền premium qua verified entitlement; ads không present khi premium, tracking consent.

### Q&A

**Q: CallKit có intercept cuộc gọi real-time?** Call Directory cung cấp entries cho system; khác VoIP CallKit và Live Caller ID Lookup.

**Q: Main app chia sẻ list thế nào?** App Groups là giải pháp tham khảo, file snapshot atomic; xác nhận cơ chế thực tế.

**Q: 100k số spam update?** Sort/dedup trước, memory profile, stream, version/diff và reload error recovery.

**Chuẩn bị:** Vẽ flow thật, kích thước data thực tế, extension target/entitlement, StoreKit version/SDK, một incident và giải pháp thực tế; không tự gán số liệu.
