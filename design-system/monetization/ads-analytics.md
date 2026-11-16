# Ads, Analytics và Attribution

### Định nghĩa

Ads: banner, interstitial (điểm chuyển tự nhiên), rewarded (user opt-in, reward sau callback hợp lệ), native. Analytics ghi event để phân tích hành vi; attribution nối install/conversion với campaign trong giới hạn privacy. CV không ghi vendor SDK, ví dụ AdMob/Firebase/AppsFlyer chỉ là công nghệ tham khảo.

### Flow và state

* SDK init sau config/consent phù hợp; load → ready → present → impression/click → dismiss/fail. Reload sau full-screen ad, không giữ controller cũ, chỉ present trên main actor khi view visible.
* Frequency cap, no-fill là bình thường, timeout/fallback, offline; premium entitlement quyết định ads, không spam lúc app launch.
* Rewarded: reward callback khác dismiss; cấp reward idempotent, server-side verification nếu cần, không cấp từ click.
* Analytics event schema: name, properties, timestamp, session/user ID theo consent; tránh PII, số điện thoại/SMS, token, ảnh. Dedup event purchase theo transaction ID. Funnel onboarding → paywall → purchase → retention; phân biệt revenue và conversion.
* Attribution: campaign/source, install vs re-engagement, ATT nếu tracking theo định nghĩa Apple; không fingerprint để bypass. SKAdNetwork/AdAttributionKit hỗ trợ attribution privacy-preserving, dữ liệu aggregated/delayed, không đảm bảo user-level mapping.
* Consent, ATT, privacy manifest và App Store privacy labels là các yêu cầu khác nhau; kiểm tra SDK/version, data collection và vùng pháp lý.

```swift
protocol AnalyticsTracking {
    func track(_ name: String, properties: [String: String])
}

struct PaywallTracker {
    let analytics: any AnalyticsTracking

    func shown(source: String) {
        analytics.track("paywall_view", properties: ["source": source])
    }
}
```

Inject adapter thay vì gọi vendor SDK khắp codebase, test event name/properties và purchase dedup.

### Q&A

**Q: ATT denied có chạy ads được không?** Có thể contextual/non-tracking ads tùy SDK/consent; không truy cập IDFA/tracking khi chưa được phép.

**Q: Interstitial không load?** Kiểm tra no-fill/network/consent/test ad unit, fallback UI, retry bounded.

**Q: Analytics khác attribution?** Hành vi trong app vs nguồn acquisition/conversion, dữ liệu privacy khác nhau.

**Liên hệ:** Call ID/FastVPN ads, analytics, attribution; Watch Faces subscription.

## Ads và Analytics chuyên sâu

### Ad format và state

| Format | Khi dùng | Lỗi thường gặp |
| --- | --- | --- |
| Banner | Vùng ổn định không che content | Layout nhảy, refresh quá nhiều |
| Interstitial | Điểm chuyển tự nhiên | Present khi screen chưa visible, spam |
| Rewarded | User opt-in đổi reward | Grant khi dismiss thay vì earned |
| Native | Content tích hợp layout | Thiếu ad label/privacy |

```text
idle -> loading -> ready -> presenting -> dismissed -> idle
             -> failed/no-fill -> bounded retry/fallback
```

Check entitlement/consent lại trước present vì premium có thể đổi trong lúc load. Chỉ một full-screen ad tại một thời điểm, main actor, visible VC. No-fill bình thường, không retry loop; frequency cap và test ad unit. Rewarded server verification nếu reward có giá trị, dedup event ID.

```swift
enum AdState { case idle, loading, ready, presenting }

@MainActor
final class AdGate {
    private(set) var state: AdState = .idle
    var isPremium = false

    func beginLoad() -> Bool {
        guard !isPremium, state == .idle else { return false }
        state = .loading
        return true
    }

    func loaded() { state = isPremium ? .idle : .ready }

    func beginPresentation() -> Bool {
        guard !isPremium, state == .ready else { return false }
        state = .presenting
        return true
    }

    func finished() { state = .idle }
}
```

Vendor-neutral gate; SDK adapter thực hiện load/present/callback. Khi premium/logout đổi, invalidate loaded ad, handle stale callbacks bằng generation.

### Analytics schema và funnel

onboarding_view → onboarding_complete → paywall_view → purchase_start → purchase_success/cancel/error → renewal/retention. Event name/properties có schema/version, source/placement/productID hợp lệ; purchase_success sau verified transaction, dedup transaction ID (không gửi receipt/token). Impression theo SDK callback, không đếm load là impression. Offline queue bounded size, retry/consent/timestamp; không log PII/SMS/ảnh/phone.

### Attribution và privacy

Attribution đo install/re-engagement nguồn campaign. Last-touch, window, aggregation và delay khiến provider chênh lệch. ATT tracking khác first-party analytics; không fingerprint/bypass denied. SKAdNetwork/AdAttributionKit privacy-preserving, không guarantee user-level mapping. Privacy labels, manifest, consent/ATT và region policy là các layer riêng; audit SDK version/behavior.

**Q: Analytics revenue khác store report?** Duplicate/refund/tax/currency/delay/window khác nhau; reconcile server/store.

**Q: Ad load success nhưng không show?** Expired, screen not active, concurrent presentation, premium/consent hoặc SDK error.

**Bài tập:** Test AdGate premium đổi khi load, duplicate tap, reward dedup; fake analytics assert schema/event.

## Câu hỏi phỏng vấn tình huống

**Q: Rewarded ad đóng màn hình có cấp reward?** Không, earned reward callback/server verification theo SDK; dedup event để không cấp hai lần.

**Q: User mua premium khi ad đang loading?** Recheck entitlement/consent trước present, invalidate loaded ad và guard stale callback; không show full-screen ad sau premium.

**Q: Analytics event nào không nên ghi?** Token/receipt, SMS/phone/photo/PII không cần thiết. Dùng schema, sanitized IDs, consent, bounded queue và transaction dedup.
