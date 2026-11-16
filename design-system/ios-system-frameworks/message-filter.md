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
