# Swift Concurrency: Task, Actor, Sendable

### Định nghĩa

async biểu diễn hàm có thể suspend; await là điểm có thể tạm dừng, không đồng nghĩa chạy background thread. Executor quyết định nơi chạy; @MainActor dùng cho state UI. Structured concurrency giữ lifetime của child task trong scope cha.

| API | Ý nghĩa |
| --- | --- |
| async let | Số tác vụ con cố định, chờ kết quả trong scope |
| TaskGroup | Số tác vụ con động, thu kết quả theo thứ tự hoàn thành |
| Task {} | Unstructured task, kế thừa actor context, priority/task-local khi phù hợp |
| Task.detached | Unstructured task không kế thừa actor context/task-local |
| Actor | Cô lập mutable state, serialize truy cập trong các đoạn không suspend |
| Sendable | Kiểu có thể truyền an toàn qua isolation boundary |

```swift
func loadBoth() async throws -> (Data, Data) {
    async let a = URLSession.shared.data(from: URL(string: "https://example.com/a")!).0
    async let b = URLSession.shared.data(from: URL(string: "https://example.com/b")!).0
    return try await (a, b)
}
```

Ví dụ trên dùng URL cố định để minh họa; với input động phải validate URL. TaskGroup hữu ích tải nhiều trang/ảnh, giới hạn concurrency theo batch thay vì tạo hàng nghìn task một lần.

```swift
actor TokenCache {
    private var token: String?

    func update(_ value: String) {
        token = value
    }

    func current() -> String? {
        token
    }
}

@MainActor
final class SearchViewModel {
    private var task: Task<Void, Never>?
    private(set) var text = ""

    func search(_ operation: @escaping @Sendable () async throws -> String) {
        task?.cancel()
        task = Task { [weak self] in
            do {
                let result = try await operation()
                try Task.checkCancellation()
                self?.text = result
            } catch {
                // Phân biệt cancellation với lỗi hiển thị UI.
            }
        }
    }

    func cancel() { task?.cancel() }
}
```

### Cơ chế và lưu ý

* Cancellation là cooperative: Task.cancel() đặt flag; checkCancellation/Task.isCancelled, API hỗ trợ cancellation hoặc hủy request tương ứng. Không đảm bảo dừng ngay.
* Actor reentrancy: qua await, actor có thể xử lý task khác. Không giả định state không đổi; kiểm tra lại invariant/version sau await.
* Swift 6 strict concurrency phát hiện nhiều data race tại compile time. Dùng immutable value types, actor, @MainActor; @unchecked Sendable chỉ khi tự đảm bảo thread safety.
* Không block cooperative executor bằng semaphore.wait hay sleep đồng bộ. Dùng async API, Task.sleep, continuation (resume đúng một lần); CPU-heavy cần thiết kế isolation/executor phù hợp.
* MainActor không phải cơ chế tự động đẩy mọi công việc nặng sang background. Đặc biệt chú ý actor inheritance của Task.
* Swift 5.5 là thời điểm giới thiệu concurrency; kiểm tra deployment target của từng API (URLSession async iOS 15+).

### Q&A

**Q: TaskGroup khác DispatchGroup?** TaskGroup quản lý child task, cancellation và lifetime theo scope; DispatchGroup đếm công việc GCD, không tự quản lý cancellation.

**Q: Actor có tránh mọi race?** Tránh data race trên state isolated; vẫn có logical race qua suspension point.

**Q: Search API trả kết quả cũ sau query mới?** Cancel task cũ, kiểm tra cancellation hoặc request generation trước update state.

**Liên hệ:** FastVPN hủy speed test, Watch Faces tải ảnh, Call ID cập nhật dữ liệu spam.
