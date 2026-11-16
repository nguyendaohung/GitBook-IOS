# DaPass và BIDU E-Commerce

### DaPass (Junior iOS)

**CV:** voucher/cafe, social login, VNPay/Visa, JSON mobile-web integration, QR redeem. Swift, Moya, SwiftyJSON, Kingfisher, SwiftLint, R.Swift, MVP, Notifications, CocoaPods.

**Flow tham khảo:** View → Presenter → API service → backend; order pending → gateway → callback → backend verify → paid → QR redeem. [Payment và QR](../ios-system-frameworks/payment-qr.md), [Moya](../networking/api-client.md), [MVP](../design-patterns-architecture/architectures/mvp.md).

**Trade-off:** MVP tách UI/presentation, tăng protocol/boilerplate; Moya type-safe endpoint nhưng thêm abstraction; SwiftyJSON linh hoạt, Codable kiểm tra type rõ hơn. QR client debounce nhưng backend atomic redeem.

**Q: App bị kill sau thanh toán?** Query order server khi mở lại, không tin callback success; webhook/idempotency.

**Q: Social login token hết hạn?** Refresh một lần, lưu Keychain, revoke/logout nếu refresh fail.

### BIDU

**CV:** fashion social/e-commerce, UI/UX, mobile-web integration, bug fixing; Swift, Moya, Kingfisher, SwiftLint, R.Swift, Notifications, CocoaPods, Texture.

**Flow tham khảo:** feed pagination → DTO/domain → node data source → async layout → thumbnail cache/prefetch → reuse. [Texture và image performance](../ui-+-autolayout/snapkit-image-performance.md).

**Trade-off:** Texture giúp prepare layout/decode ngoài main khi phù hợp, thêm dependency và learning curve; native collection view + prefetch là phương án khác. Dedup cursor/ID, cancel stale request, tránh ảnh sai cell.

**Q: Feed 1000 ảnh lag?** Time Profiler/Allocations, thumbnail downsample, cache limit, bounded prefetch, node reuse, pagination.

**Q: 2 request pagination trùng?** isLoading/cursor guard, dedup ID, reset query generation.

### Chuẩn bị kinh nghiệm thật

Chọn một feature đã trực tiếp làm mỗi dự án: yêu cầu → quyết định kỹ thuật → implementation → bug thật → cách đo/verify. Xác nhận payment SDK, social provider, notification type, feed API/pagination và kết quả đo (nếu có).
