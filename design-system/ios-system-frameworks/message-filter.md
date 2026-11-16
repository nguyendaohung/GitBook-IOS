# Message Filter Extension và App Groups

### Định nghĩa

IdentityLookup Message Filter Extension (ILMessageFilterExtension) phân loại tin nhắn từ unknown sender trong phạm vi hệ thống hỗ trợ. Không được đọc toàn bộ Messages database hay mọi iMessage; system quyết định query nào gửi đến extension. User phải bật filter trong Settings.

```swift
import IdentityLookup

final class MessageFilterExtension: ILMessageFilterExtension,
                                    ILMessageFilterQueryHandling {
    func handle(_ queryRequest: ILMessageFilterQueryRequest,
                context: ILMessageFilterExtensionContext,
                completion: @escaping (ILMessageFilterQueryResponse) -> Void) {
        let response = ILMessageFilterQueryResponse()
        let text = queryRequest.messageBody?.lowercased() ?? ""
        // Rule minh họa, không đủ cho spam classifier thực tế.
        response.action = text.contains("claim your prize") ? .junk : .none
        completion(response)
    }
}
```

### Cơ chế và bảo mật

* Query có sender/body (optional); action có none/allow/junk và category/subaction theo OS/API. .none là không phân loại; hạn chế false positive, không tự xóa SMS.
* Nếu cần network classification, dùng deferQueryRequestToNetwork theo cơ chế hệ thống và cấu hình associated domain/server phù hợp; không giả định extension có network tự do.
* Main app và extension có sandbox riêng; App Groups shared container (xem [FileManager và App Groups](../data-stroage/file-cache.md)). Đọc versioned snapshot, atomic write, dữ liệu last-known-good; không chia sẻ object in-memory.
* Không gửi nội dung SMS hoặc số điện thoại vào analytics/log. Giảm dữ liệu gửi backend, giải thích privacy, xử lý fail-open/none khi không chắc.
* Test unknown sender/contact, SMS rỗng, nhiều ngôn ngữ, false positive, extension disabled, offline, snapshot corrupt. Completion đúng một lần.

### Q&A

**Q: Call Directory vs Message Filter?** Một extension cung cấp số nhận diện/chặn call; extension kia phân loại từng message query.

**Q: Có dùng UserDefaults.standard để chia sẻ giữa targets?** Không; dùng suiteName với App Groups hoặc shared file.

**Q: App có thể tự bật SMS filter không?** Không, cần user bật trong Settings.

**Liên hệ:** Call ID – SMS filtering và spam data.

[IdentityLookup - Apple](https://developer.apple.com/documentation/identitylookup)

## Message Filter: classifier và privacy

### Flow

```text
System nhận message đủ điều kiện -> Extension query sender/body
-> rule/model local -> allow/junk/none
-> nếu cần: system-mediated network deferral -> response -> completion
```

Không phải Notification Service Extension. SMS filter nhận message query do hệ thống cung cấp, không sửa/xóa tin nhắn hay xem lịch sử. Action .none khi không đủ bằng chứng. Hệ thống/OS quyết định supported category, kiểm tra API availability trước dùng.

```swift
enum MessageDecision {
    case allow, junk, unknown
}

struct MessageClassifier {
    let trustedSenders: Set<String>
    let blockedSenders: Set<String>

    func classify(sender: String?, body: String?) -> MessageDecision {
        guard let sender, !sender.isEmpty else { return .unknown }
        if trustedSenders.contains(sender) { return .allow }
        if blockedSenders.contains(sender) { return .junk }
        // Không block chỉ vì một từ chung như "payment".
        guard let body, !body.isEmpty else { return .unknown }
        return .unknown
    }
}
```

Pure classifier dễ unit test. Production normalize sender (có thể alpha sender ID), language/Unicode, rule precedence, version, false-positive cost. Không tự thu thập nội dung nhạy cảm; shared snapshot chỉ chứa rule cần thiết.

### App Group và network deferral

Bật App Groups ở hai target, cùng group identifier; data file atomically ghi, extension validate schema/version và fallback nếu thiếu/corrupt. Extension không có quyền đọc UserDefaults.standard của app. Nếu local chưa quyết định, ILMessageFilterExtensionContext.deferQueryRequestToNetwork cho system gửi query tới endpoint đã cấu hình theo Apple, completion parse response/error, fallback .none. Không tự gọi URLSession gửi SMS tùy ý. Kiểm tra privacy disclosure và domain association theo OS.

### Q&A bổ sung

**Q: Filter spam và chặn sender có giống nhau?** Filter phân loại message, không tự chặn mọi message hay xóa lịch sử.

**Q: Offline khi rule không đủ?** Fallback .none, không block nhầm; giữ snapshot last-known-good.

**Q: Completion bị gọi hai lần?** Vi phạm contract; thiết kế một đường hoàn tất duy nhất kể cả network error/timeout.

**Bài tập:** Test classifier trusted/blocked/unknown, sender nil, Unicode, rule conflict; test extension disabled và snapshot corrupt trên device.

## Câu hỏi phỏng vấn tình huống

**Q: Message Filter vs Notification Service Extension?** IdentityLookup phân loại message đủ điều kiện; Notification Service sửa remote notification mutable-content, không đọc SMS.

**Q: Network deferral có gửi SMS trực tiếp URLSession?** Không tự tùy ý; dùng system-mediated deferQueryRequestToNetwork theo domain/configuration contract, fallback khi fail.

**Q: Rule spam false positive xử lý?** Trusted precedence, confidence threshold/none, test Unicode/language, privacy; không log message/phone.
