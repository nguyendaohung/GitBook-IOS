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

## Structured concurrency thực hành

### Chạy song song và giới hạn concurrency

```swift
import Foundation

func fetchAll(_ urls: [URL]) async throws -> [Data] {
    try await withThrowingTaskGroup(of: (Int, Data).self) { group in
        let limit = 4
        var results = Array<Data?>(repeating: nil, count: urls.count)

        for index in 0..<min(limit, urls.count) {
            group.addTask {
                let (data, response) = try await URLSession.shared.data(from: urls[index])
                guard let http = response as? HTTPURLResponse,
                      (200..<300).contains(http.statusCode) else {
                    throw URLError(.badServerResponse)
                }
                return (index, data)
            }
        }

        var next = min(limit, urls.count)
        while let (index, data) = try await group.next() {
            results[index] = data
            if next < urls.count {
                let index = next
                next += 1
                group.addTask {
                    let (data, response) = try await URLSession.shared.data(from: urls[index])
                    guard let http = response as? HTTPURLResponse,
                          (200..<300).contains(http.statusCode) else {
                        throw URLError(.badServerResponse)
                    }
                    return (index, data)
                }
            }
        }
        return results.compactMap { $0 }
    }
}
```

TaskGroup tối đa bốn request đang thực thi, giữ kết quả theo index gốc. Throw thoát scope sẽ yêu cầu cancel child task và đợi child kết thúc; không dùng cho toàn bộ ảnh lớn nếu cần streaming/file cache. TaskGroup không tự giới hạn concurrency khi addTask hàng loạt.

### Actor reentrancy và single-flight refresh

Actor serialize các đoạn không suspend, nhưng qua await task khác có thể vào actor. Ví dụ token hết hạn: hai request cùng gọi refresh, nếu chỉ kiểm tra token trước await có thể tạo hai network calls.

```swift
actor RefreshGate {
    private var inFlight: Task<String, Error>?

    func refresh(using work: @escaping @Sendable () async throws -> String)
        async throws -> String {
        if let existing = inFlight { return try await existing.value }
        let task = Task { try await work() }
        inFlight = task
        defer { inFlight = nil }
        return try await task.value
    }
}
```

Shared refresh task có lifetime do gate sở hữu; một caller cancel không nên hủy refresh cho tất cả callers. Logout cần thiết kế session generation, hủy in-flight và chặn token cũ ghi vào session mới. Actor không đảm bảo fairness hay tránh mọi logical race.

### Continuation và callback cũ

```swift
func bridge(_ start: (@escaping (Result<Data, Error>) -> Void) -> Void)
    async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        start { result in continuation.resume(with: result) }
    }
}
```

Callback phải gọi đúng một lần. Checked continuation hỗ trợ phát hiện misuse, không tự timeout/cancel callback API; khi cần cancellation dùng withTaskCancellationHandler và đồng bộ handle/token để tránh race giữa start và cancel. Không resume hai lần từ timeout và callback.

### GCD vs OperationQueue vs Swift Concurrency

| Công cụ | Phù hợp | Cancellation |
| --- | --- | --- |
| GCD | Queue thấp tầng, callback legacy | DispatchWorkItem cooperative |
| OperationQueue | Dependency, maxConcurrentOperationCount | Operation.cancel, tự check flag |
| async/await | Async API, flow dễ đọc | Task cancellation |
| Actor | Shared mutable state | Không thay task cancellation |
| TaskGroup | Child tasks có scope/lifetime | Propagate cooperative |

Không gọi DispatchQueue.main.sync từ main queue (deadlock). Không dùng semaphore để chờ async network trên main/cooperative executor. CPU-bound lớn cần chia nhỏ hoặc executor phù hợp, đo UI responsiveness.

### Q&A bổ sung

**Q: async let tạo task lúc nào?** Khi khai báo, child thực thi đồng thời theo scheduler; scope chờ/cancel khi thoát.

**Q: Task.detached có kế thừa MainActor?** Không. Chỉ dùng khi hiểu isolation/lifetime; vẫn phải chuyển UI update về MainActor.

**Q: Sendable class được không?** Có nếu immutable/thread-safe hoặc actor; compiler kiểm tra theo rule. @unchecked Sendable là cam kết của developer.

**Bài tập:** Viết search cancel cũ, task group tối đa 4 request, actor token refresh. Test cancellation và concurrent callers.
