# 💰 Monetization

* [StoreKit và Subscription](storekit.md)
* [Ads, Analytics và Attribution](ads-analytics.md)

Call ID, FastVPN và Watch Faces có premium/subscription; SDK quảng cáo/analytics cụ thể chưa được ghi trong CV.

## Câu hỏi phỏng vấn tổng quan

**Q: StoreKit entitlement vs purchase transaction?** Transaction record signed/verified, entitlement quyền hiện tại theo expiry/revocation; listener và idempotent delivery.

**Q: Ads khi premium?** Check verified entitlement trước load/present, cancel/invalidate loaded ad, consent/ATT và frequency cap.

**Q: Analytics vs attribution?** Event behavior/funnel vs acquisition/campaign; privacy, dedup và source of truth store/server.
