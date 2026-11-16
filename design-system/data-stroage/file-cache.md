# FileManager, UserDefaults, App Groups và Cache

### Định nghĩa

* UserDefaults: setting nhỏ, không lưu secret hoặc file lớn.
* Documents: dữ liệu người dùng cần giữ; Application Support: dữ liệu app; Caches: dữ liệu tái tạo được, system có thể purge; temporaryDirectory: dữ liệu ngắn hạn.
* App Groups: shared container giữa targets có cùng entitlement; UserDefaults(suiteName:) cho setting nhỏ, FileManager.containerURL cho file.
* NSCache: memory cache có thể tự evict; disk cache cần TTL, size limit, invalidation và xử lý I/O ngoài main actor.

```swift
import Foundation

func writeSnapshot(_ data: Data, group: String) throws {
    guard let directory = FileManager.default.containerURL(
        forSecurityApplicationGroupIdentifier: group
    ) else {
        throw CocoaError(.fileNoSuchFile)
    }
    try data.write(to: directory.appendingPathComponent("snapshot.json"),
                   options: .atomic)
}
```

### Lưu ý

Atomic write tránh đọc file đang ghi dở, nhưng không tự serialize nhiều writer/transaction nhiều file. Dùng một writer, versioned snapshot/checksum và coordination khi cần. Không hardcode group ID khác entitlement. Chọn file protection phù hợp lock screen và extension; hạn chế retention dữ liệu nhạy cảm.

### Q&A

**Q: Cache mất khi nào?** NSCache có thể evict khi pressure; Caches có thể bị system xóa; app phải tải lại được.

**Q: App Groups và Keychain Access Groups khác nhau?** File/settings chia sẻ qua App Groups; secret chia sẻ qua keychain entitlement riêng.

**Liên hệ:** Call ID chia sẻ spam snapshot, Watch Faces cache ảnh; cơ chế thực tế cần xác nhận với source dự án.
