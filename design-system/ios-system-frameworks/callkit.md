# CallKit và Call Directory Extension

### Định nghĩa

CallKit tích hợp trải nghiệm cuộc gọi với hệ thống. CXProvider/CXCallController dành cho quản lý VoIP call của app; CXCallDirectoryProvider là extension để cung cấp số nhận diện/chặn cho cuộc gọi hệ thống. Không phải quyền đọc call log, ghi âm hay intercept toàn bộ cuộc gọi.

### Cơ chế hoạt động

1. Main app tải danh sách spam, chuẩn hóa số quốc tế thành CXCallDirectoryPhoneNumber (Int64, gồm country code, không ký tự +/space), sort tăng dần và loại trùng.
2. Lưu snapshot cho extension (ví dụ App Groups), ghi atomically, giữ version.
3. Kiểm tra extension enabled bằng CXCallDirectoryManager.getEnabledStatusForExtension; người dùng bật trong Settings; app không tự bật.
4. reloadExtension(withIdentifier:) yêu cầu system load; extension beginRequest(with:) cung cấp entries theo thứ tự tăng dần, gọi completeRequest.
5. System lưu/index và thực hiện matching khi có cuộc gọi. Call Directory truyền thống không gọi REST API mỗi lần ring. Live Caller ID Lookup là cơ chế khác, có yêu cầu hệ thống/backend riêng, không suy ra từ CV.

```swift
import CallKit

final class CallDirectoryHandler: CXCallDirectoryProvider {
    override func beginRequest(with context: CXCallDirectoryExtensionContext) {
        // Ví dụ dữ liệu giả, đã sort và deduplicate.
        let entries: [(CXCallDirectoryPhoneNumber, String)] = [
            (12025550101, "Spam"),
            (12025550102, "Unknown business")
        ]
        if context.isIncremental {
            // Minh họa full refresh trong incremental context.
            context.removeAllIdentificationEntries()
            context.removeAllBlockingEntries()
        }
        for (number, label) in entries {
            context.addIdentificationEntry(
                withNextSequentialPhoneNumber: number, label: label
            )
        }
        context.completeRequest()
    }
}
```

Để block dùng addBlockingEntry(withNextSequentialPhoneNumber:). Production incremental có thể thêm/xóa diff đúng snapshot mà system đã load; nếu không biết version, reset khi incremental được phép. Xử lý error từ reload, extension context failure, không đánh dấu snapshot đã apply khi reload thất bại.

### Lưu ý

Extension là process riêng, giới hạn memory/time; stream đọc file đã sort nếu dataset lớn, không tải toàn bộ JSON nhiều lần. Không làm network chậm trong beginRequest, không đoán hard limit entries. Test trên iPhone thật, bật/tắt extension, số khác format, update lỗi, mất mạng và dữ liệu trùng.

### Q&A

**Q: Tại sao phải sort tăng dần?** API yêu cầu sequential entries; số trùng/sai thứ tự có thể làm reload fail.

**Q: App có chặn mọi cuộc gọi theo logic tự định nghĩa real-time?** Call Directory cung cấp list cho hệ thống; không nhận callback mọi incoming cellular call.

**Q: Khi reload fail?** Giữ snapshot last-known-good, ghi log đã redact, kiểm tra enabled status, sort/duplicate/resource, retry có giới hạn.

**Liên hệ:** Call ID – caller identification, blocking, local spam data.

[CallKit - Apple](https://developer.apple.com/documentation/callkit)
