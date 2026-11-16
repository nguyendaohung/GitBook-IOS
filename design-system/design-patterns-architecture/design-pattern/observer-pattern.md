# OBSERVER PATTERN

### Định nghĩa

Observer là behavioral pattern: subject phát thay đổi tới một hoặc nhiều observer mà không cần biết concrete implementation. Ví dụ notification, KVO, Combine publisher, Rx observable. One-to-many khác Delegate thường one-to-one; callback đơn giản phù hợp một consumer. NotificationCenter broadcast trong process, không đồng nghĩa APNs.

### Ví dụ NotificationCenter

```swift
import Foundation

extension Notification.Name {
    static let sessionDidChange = Notification.Name("sessionDidChange")
}

final class SessionObserver {
    private var token: NSObjectProtocol?

    func start() {
        guard token == nil else { return }
        token = NotificationCenter.default.addObserver(
            forName: .sessionDidChange, object: nil, queue: .main
        ) { [weak self] notification in
            self?.handle(notification)
        }
    }

    func stop() {
        if let token { NotificationCenter.default.removeObserver(token) }
        token = nil
    }

    private func handle(_ notification: Notification) {
        // Refresh state from authoritative source.
    }

    deinit { stop() }
}
```

Post: NotificationCenter.default.post(name: .sessionDidChange, object: nil). Không gửi token/PII trong userInfo. Block observer giữ token và remove khi lifetime kết thúc; weak capture tránh cycle. queue nil callback trên posting thread; queue main cho UI nhưng vẫn chú ý actor isolation. NotificationCenter không guarantee replay/latest state; khi screen mở cần đọc source hiện tại.

### So sánh

| Pattern | Use case | Lưu ý |
| --- | --- | --- |
| Delegate | Một owner nhận action/callback | weak delegate, protocol |
| NotificationCenter | Broadcast loose coupling | String/name schema, lifetime, khó trace |
| KVO | Quan sát property ObjC-compatible | @objc dynamic, observation token |
| Combine/Rx | Stream có operators/lifecycle | cancel/dispose, scheduler |
| SwiftUI Observation | UI property dependencies | identity và main actor |

**Q: Observer có tránh coupling hoàn toàn?** Không; phụ thuộc event schema/contract, khó debug event chain. Dùng typed state khi phù hợp.

**Bài tập:** Post logout event, observer refresh UI, remove observer; so sánh với Combine subject.
