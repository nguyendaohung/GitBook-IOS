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

## Call Directory thực hành chi tiết

### Chuẩn hóa dữ liệu và kiểm tra thứ tự

Phone number cho Call Directory là Int64 digits gồm country code. Không chỉ xóa ký tự '+' rồi chặn mọi số: local number cần region và quy tắc numbering plan, dùng parser đáng tin cậy hoặc backend normalized E.164. Từ chối extension, malformed, overflow và country code không hợp lệ. Ví dụ input đã chuẩn hóa:

```swift
import CallKit

struct SpamEntry {
    let number: CXCallDirectoryPhoneNumber
    let label: String
}

func prepareEntries(_ entries: [SpamEntry]) -> [SpamEntry] {
    var unique: [CXCallDirectoryPhoneNumber: SpamEntry] = [:]
    for entry in entries where entry.number > 0 {
        unique[entry.number] = entry // policy: latest entry wins
    }
    return unique.values.sorted { $0.number < $1.number }
}
```

Dữ liệu cần validate trước khi extension load; duplicate label conflict có policy rõ, danh sách block và identification có thể độc lập. Nếu full dataset rất lớn, sort/dedup tại backend/main app hoặc external merge, extension đọc sequential record thay vì JSON decode toàn bộ. Snapshot manifest version/count/checksum, atomic replace, last-known-good.

### Main app reload flow

```swift
import CallKit

func reloadDirectory(identifier: String) {
    let manager = CXCallDirectoryManager.sharedInstance
    manager.getEnabledStatusForExtension(withIdentifier: identifier) { status, error in
        if let error {
            // Update UI/log sanitized error.
            print(error.localizedDescription)
            return
        }
        guard status == .enabled else {
            // Guide user to enable extension in Settings.
            return
        }
        manager.reloadExtension(withIdentifier: identifier) { error in
            // Handle error; only mark applied version after success.
            if let error { print(error.localizedDescription) }
        }
    }
}
```

Callback UI chuyển main actor. Identifier là bundle identifier của extension, không phải main app. Reload có thể fail khi extension disabled, loading interrupted, entries sai thứ tự/trùng, resource limit. Không reload liên tục cho mỗi số; batch update, backoff. Khi isIncremental false, system yêu cầu full list; true có thể remove/add diff, hoặc removeAll và rebuild. Không ghi applied version trước complete/reload success.

### So sánh CallKit, Call Directory, Live Caller ID

| API | Use case | Giới hạn |
| --- | --- | --- |
| CXProvider/CXCallController | VoIP call của app | Không đọc mọi cellular call |
| Call Directory | Offline/system lookup danh sách | Không callback app mỗi incoming call |
| Live Caller ID Lookup | Lookup caller identity theo cơ chế riêng | OS/backend/extension requirements, không giống REST tự do |
| IdentityLookup | SMS filtering | Không đọc toàn bộ Messages |

### Test và edge cases

Test device thật, enable/disable, country code, same number different format, empty list, duplicate/sort, reload khi app background, corrupt snapshot, low memory. Log version/count/error code, không log toàn bộ số người dùng. Test pure normalization/sort/diff bằng unit test. App extension có process/lifetime riêng; không dùng shared singleton in-memory.

**Q: Chặn số mới có hiệu lực ngay?** Sau khi system load/reload list thành công, không bảo đảm tức thì khi app chỉ tải API.

**Q: Vì sao incremental update nguy hiểm khi version lệch?** System đang giữ list version khác, diff có thể remove/add sai; full reset khi phù hợp.

**Bài tập:** Tạo app + Call Directory target, App Group, fixture 3 số giả; reload và test lỗi duplicate. Không dùng số điện thoại cá nhân trong test fixture.

## Câu hỏi phỏng vấn tình huống

**Q: Call Directory extension có thể nhận incoming phone number từng call để gọi API?** Không với traditional directory; system match list. Live Caller ID Lookup khác cơ chế và requirements.

**Q: Số local 090... chuyển thành Int64 thế nào?** Cần region/numbering plan normalize E.164, bỏ prefix quốc tế theo parser đáng tin, sort/dedup; không chỉ strip ký tự.

**Q: Main app process còn sống mới extension chạy?** Không, extension process/lifetime riêng do system quản lý; shared container snapshot, không singleton in-memory.
