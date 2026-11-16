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
