# Async/Await

**ĐỊNH NGHĨA:**&#x20;

Async/Await là một tính năng mới được giới thiệu trong Swift 5.5. Nó là một cách để thực hiện các tác vụ không đồng bộ một cách dễ dàng và thân thiện với người dùng.

Trong Swift, bạn có thể sử dụng từ khóa `async` để chỉ định rằng một hàm sẽ trả về một kết quả không đồng bộ. Sau đó, bạn có thể sử dụng từ khóa `await` để đợi cho kết quả trả về từ hàm đó trước khi tiếp tục thực thi code.

**Có thể sử dụng Async/Await** trong các tình huống mà các tác vụ cần phải thực hiện bất đồng bộ, nhưng vẫn cần phải đảm bảo rằng các tác vụ này hoàn thành trước khi tiếp tục thực hiện các tác vụ khác. Các tình huống này bao gồm:

1. **Thực hiện các yêu cầu mạng**: Khi thực hiện các yêu cầu mạng, việc phản hồi chậm và không đồng bộ là điều phổ biến. Sử dụng Async/Await giúp đảm bảo rằng các yêu cầu mạng hoàn thành trước khi xử lý các phản hồi.

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 11.13.30.png" alt=""><figcaption></figcaption></figure>

Trong ví dụ này, chúng ta sử dụng hàm `URLSession.shared.data(from:)` để thực hiện yêu cầu mạng bất đồng bộ và sử dụng từ khóa `await` để đợi cho yêu cầu hoàn thành trước khi trả về kết quả.

2. **Xử lý các tác vụ nặng bất đồng bộ**: Khi thực hiện các tác vụ nặng như đọc và ghi tệp hoặc xử lý hình ảnh, việc thực hiện chúng trên luồng chính sẽ gây tắc nghẽn và làm giảm hiệu suất. Async/Await cho phép suspend mà không block thread khi API hỗ trợ; không tự chuyển tác vụ CPU-bound ra background. Cần chọn isolation/executor phù hợp để không block main actor.

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 11.15.21.png" alt=""><figcaption></figcaption></figure>

Trong ví dụ này, chúng ta sử dụng hàm `fetchData(from:)` để tải dữ liệu từ một URL và sử dụng từ khóa `await` để đợi cho tải xuống hoàn thành trước khi tiếp tục thực hiện các tác vụ khác. Sau đó, chúng ta sử dụng hàm `processImage(_:)` để xử lý ảnh một cách bất đồng bộ và đợi cho xử lý hoàn thành trước khi trả về kết quả.

3. **Xử lý các tác vụ đa luồng**: Khi bạn cần thực hiện các tác vụ đa luồng, việc quản lý và đồng bộ hóa chúng có thể là một thách thức. Sử dụng Async/Await giúp đơn giản hóa quản lý các tác vụ đa luồng bằng cách đợi các tác vụ hoàn thành trước khi tiếp tục thực hiện các tác vụ khác.

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 11.17.15.png" alt=""><figcaption></figcaption></figure>

Trong ví dụ này, chúng ta sử dụng các hàm `fetchDataTask1()`, `fetchDataTask2()` và `fetchDataTask3()` để thực hiện các tác vụ bất đồng bộ song song. Sau đó, chúng ta sử dụng từ khóa `await` để đợi cho các tác vụ hoàn thành trước khi trả về kết quả dưới dạng một mảng các chuỗi.

4. **Xử lý các tác vụ theo thứ tự đúng:**

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 11.18.26.png" alt=""><figcaption></figcaption></figure>

Trong ví dụ này, chúng ta sử dụng các hàm `fetchDataTask1()`, `fetchDataTask2()` và `fetchDataTask3()` để thực hiện các tác vụ bất đồng bộ theo thứ tự. Chúng ta sử dụng từ khóa `await` để đợi cho mỗi tác vụ hoàn thành trước khi tiếp tục thực hiện tác vụ tiếp theo. Cuối cùng, chúng ta sử dụng hàm `processResults(_:_:_:)` để xử lý kết quả trả về từ các tác vụ.

### Kiến thức mở rộng

Xem [Swift Concurrency: Task, Actor, Sendable](structured-concurrency.md) để hiểu structured concurrency, cancellation và data-race safety. Dùng async let/TaskGroup để chạy child tasks đồng thời; await từng hàm theo thứ tự không tự tạo parallelism.

## Async/Await chuyên sâu: hiểu đúng suspension và concurrency

async biểu diễn hàm có thể suspend; await đánh dấu điểm có thể tạm dừng. Không có nghĩa tạo thread mới hoặc luôn background. Executor/actor isolation quyết định nơi chạy. Một async function CPU-bound vẫn có thể block actor hiện tại. URLSession async API suspend trong khi chờ network; khi có kết quả, task resume theo isolation.

### Sequential vs parallel

```swift
import Foundation

func sequential(a: URL, b: URL) async throws -> (Data, Data) {
    let first = try await URLSession.shared.data(from: a).0
    let second = try await URLSession.shared.data(from: b).0
    return (first, second)
}

func parallel(a: URL, b: URL) async throws -> (Data, Data) {
    async let first = URLSession.shared.data(from: a).0
    async let second = URLSession.shared.data(from: b).0
    return try await (first, second)
}
```

Sequential cần kết quả A trước B hoặc dependency. async let tạo child tasks có structured lifetime, cho phép hai request đồng thời. Không bảo đảm thời gian bằng đúng max(A,B), còn scheduling/network limits. Ví dụ chỉ lấy Data; production validate HTTP status/decode. URLSession async API iOS 15+.

### Error, cancellation và main actor

```swift
@MainActor
final class ScreenModel {
    private(set) var title = ""
    private var task: Task<Void, Never>?

    func load(fetch: @escaping @Sendable () async throws -> String) {
        task?.cancel()
        task = Task { [weak self] in
            do {
                let value = try await fetch()
                try Task.checkCancellation()
                self?.title = value
            } catch is CancellationError {
                // Expected when screen/query changes.
            } catch {
                guard !Task.isCancelled else { return }
                self?.title = "Failed"
            }
        }
    }

    func stop() {
        task?.cancel()
        task = nil
    }
}
```

Task {} kế thừa actor context khi tạo trong MainActor; fetch nonisolated async/actor service theo isolation rules, không giả định Task tự chạy CPU-heavy background. Cancellation cooperative, check trước side effect/update UI; Task.sleep/URLSession thường hỗ trợ cancellation. Task.detached không kế thừa actor context/lifetime, không dùng chỉ để "đẩy background" tùy ý.

### Callback → async

```swift
func legacyBridge<T>(
    _ start: (@escaping (Result<T, Error>) -> Void) -> Void
) async throws -> T {
    try await withCheckedThrowingContinuation { continuation in
        start { continuation.resume(with: $0) }
    }
}
```

Callback phải hoàn tất đúng một lần; continuation không tự cancel/timeout legacy API. Dùng withTaskCancellationHandler và synchronized cancellation handle nếu cần. Xem [Swift Concurrency](structured-concurrency.md) về TaskGroup, Actor, Sendable và reentrancy.

**Q: await có block thread không?** Suspension có thể nhường executor/thread; synchronous work bên trong vẫn block.

**Q: async let và Task {} khác?** async let child task có structured scope; Task {} unstructured, owner phải quản lý handle/lifetime.

**Bài tập:** So sánh sequential/parallel fetch, cancel khi query thay đổi, test stale result và error.

## Câu hỏi phỏng vấn tình huống

**Q: Task cancellation có tự hủy callback legacy?** Không. Cần cancellation handler và underlying handle, resume continuation đúng một lần.

**Q: MainActor.run khác DispatchQueue.main.async?** MainActor.run execute synchronous closure trên main actor và await completion; GCD enqueue closure, không biểu diễn compiler isolation.

**Q: async function gọi sync expensive work có block?** Có, async không tự offload CPU. Chọn executor/isolation, đo main thread.
