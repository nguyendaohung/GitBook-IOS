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

## File storage và cache thực tế

### Cache-aside

```swift
import Foundation

actor DataCache {
    private var entries: [String: (data: Data, expiry: Date)] = [:]

    func value(for key: String) -> Data? {
        guard let entry = entries[key], entry.expiry > Date() else {
            entries.removeValue(forKey: key)
            return nil
        }
        return entry.data
    }

    func insert(_ data: Data, for key: String, ttl: TimeInterval) {
        entries[key] = (data, Date().addingTimeInterval(ttl))
    }

    func removeAll() { entries.removeAll() }
}
```

Minh họa cache TTL, production cần count/size bound, LRU/NSCache, disk cache và dedup in-flight requests. Cache miss → fetch remote → validate → write cache → return; stale-while-revalidate có thể show dữ liệu cũ có label rồi refresh. Không cache token/PII vô thời hạn.

### FileManager: tạo thư mục, đọc và dọn dẹp

```swift
func cacheDirectory() throws -> URL {
    let fm = FileManager.default
    let root = try fm.url(for: .cachesDirectory, in: .userDomainMask,
                          appropriateFor: nil, create: true)
    let directory = root.appendingPathComponent("Thumbnails", isDirectory: true)
    try fm.createDirectory(at: directory, withIntermediateDirectories: true)
    return directory
}
```

Dùng hashed key/validated filename, không nối path từ input tùy ý (path traversal). Data.write(.atomic) ghi file tạm rồi replace; nếu ghi nhiều file cần manifest/version để consistency. Tính disk usage và xóa theo TTL/limit; Caches có thể bị purge bất kỳ lúc nào.

### Shared container và extension

App group ID phải đúng ở app và extension. Extension đọc snapshot đã hoàn tất; main app là single writer, atomic replace, validate schema/checksum. Không đồng bộ bằng UserDefaults.synchronize hoặc giả định notifications liên process luôn giao. File coordination/locking cần thiết nếu nhiều process write, không giữ lock qua network await.

**Q: UserDefaults có thread-safe không?** Các API cơ bản thread-safe, nhưng read-modify-write nhiều bước không atomic; dùng actor/lock hoặc store phù hợp.

**Q: Cache khác persistence?** Cache có thể tái tạo, mất không làm mất dữ liệu nghiệp vụ; persistence phải có migration/backup/consistency.

**Bài tập:** Tạo image disk cache, TTL và size cap; test file mất/corrupt và concurrent read/write.
