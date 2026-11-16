# 📱 iOS System Frameworks

Các framework đặc thù trong CV: Call ID (CallKit, IdentityLookup), Map4D (CoreLocation), các ứng dụng có notification và QR Code.

* [CallKit và Call Directory Extension](callkit.md)
* [Message Filter Extension và App Groups](message-filter.md)
* [CoreLocation và Map SDK](core-location.md)
* [UserNotifications và Deep Link](notifications.md)
* [Social Login, Payment và QR Code](payment-qr.md)

## Câu hỏi phỏng vấn tổng quan

**Q: App extension khác main app?** Process/sandbox/lifetime riêng, entitlement và system entry point; shared container nếu phù hợp, không share singleton memory.

**Q: Call Directory vs SMS filter?** Directory cung cấp sorted phone list để system match; IdentityLookup phân loại message query đủ điều kiện.

**Q: Permission và privacy thiết kế thế nào?** Xin quyền theo feature, fallback denied/reduced, usage description/entitlements đúng; không log phone/SMS/PII.
