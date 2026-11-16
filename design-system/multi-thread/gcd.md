# GCD

Là 1 API cấp thấp giúp chúng ta quản lý và thực hiện đa luồng. Đây là một low-level API chịu trách nhiệm quản lý các queue trong Swift. Chức năng chính của GCD là:

* Tạo queue
* Đưa các block code vào queue

Nhắc đến **GCD** ta có 1 số khái niệm:

* **The main queue**: Là 1 serial queue. Các task trong queue này sẽ được thực thi ở main thread. Các task liên quan tới giao diện thường sẽ được đưa vào đây để thực thi. Truy cập bằng DispatchQueue.main ⇒ update UI
*   **Global queue**: Là Concurrent queue. Các task trong queue này sẽ được thực thi ở background thread.Thường dùng để xử lý các tác vụ tốn thời gian, không tác động đến UI. Truy cập bằng DispatchQueue.global  ⇒ Nó chia ra làm 4 loại chính và 1 loại default theo QoS (Quality of Service), theo mức độ ưu tiên từ cao nhất xuống thấp nhất:

    * **userInteractive** (high priority): Used for animations, or updating UI. Dành cho các công việc cần phải hoàn thành nhanh chóng để trải nghiệm người dùng tốt nhất, chẳng hạn như các tác vụ trực quan hoặc tương tác trực tiếp.
    * **userInitiated** (high priority): Used for tasks like loading data from API, preventing the user from making interactions. Dành cho các công việc do người dùng khởi động, nhưng cần phải hoàn thành trong thời gian ngắn.
    * **default:** Ưu tiên trung bình, dành cho các công việc mặc định.
    * **utility** (low priority): Used for tasks that do not need to be tracked by the user. Thường dành cho các tác vụ chạy lâu, có hiển thị chỉ số tiến trình. Ví dụ cho việc tính toán, I/O, networking, sự liên tục của dữ liệu và các tác vụ tương tự. Cái này dùng để tiết kiệm năng lượng. Nó sẽ được mapped với low priority của global queue.
    * **background** (low priority): Used for tasks like saving data in the local database or any maintenance code which is not on high priority. Dành cho những tasks mà user không cần phải biết trước. Sử dụng cho prefetching data, maintenance, hoặc những task mà không bắt buộc có sự tương tác của user

    ⇒ 2 mức độ utility và background thường được sử dụng cho những task nặng (lưu dữ liệu xuống local) để tránh việc block UI
* **Custom Queues** (hay còn gọi là private queues): chúng ta tạo ra nó, có thể là serial hoặc concurrent queues

**Một số API thường dùng của GCD trong Swift bao gồm:**

1. **dispatchQueue.async**: Thực thi một block hoặc closure nền tảng.
2. **dispatchQueue.sync**: Thực thi một block hoặc closure nền tảng theo chế độ đồng bộ.
3. **dispatchQueue.afte**r: Chờ một khoảng thời gian nhất định rồi mới thực thi một block hoặc closure.
4. **dispatchQueue.barrierAsync**: Thực thi một block hoặc closure nền tảng với một barrier. Barriers giúp đảm bảo rằng tất cả các công việc được thực thi trước barrier phải hoàn thành trước khi thực thi công việc sau barrier.
5. **dispatchGroup.enter / dispatchGroup.leave**: Sử dụng dispatch group để quản lý các công việc đang chạy và phải đợi cho đến khi tất cả các công việc đã hoàn thành.
6. **dispatchSemaphore.wait / dispatchSemaphore.signal**: Sử dụng dispatch semaphore để giới hạn số lượng công việc được thực thi cùng lúc.

**Ưu điểm:** Dễ sử dụng (GCD đã lo việc tạo thread, quản lý task)

**Nhược điểm:**

1. Không thể cancel hoặc tạm dừng 1 task khi task đó đang chạy
2. Không thể kiểm tra xem còn bao nhiêu tác vụ đang đợi trong 1 queue
3. Không thể cấu hình có tối đa bao nhiêu tác vụ chạy cùng lúc trong queue (can not define the maximum number of concurrent operations)

⇒ Trong 1 số trường hợp nhất định, chúng ta cần tracking 1 task vụ trong queue nhưng GCD không làm được

⇒ Sử dụng Operation Queue

WARNING: Khi có nhiều luồng truy cập cũng xảy ra vấn để read write  tương tự như vấn đề của [singleton](../design-patterns-architecture/design-pattern/singleton.md)

## GCD thực hành: QoS, barrier, cancellation

```swift
import Foundation

final class Cache {
    private let queue = DispatchQueue(label: "cache", attributes: .concurrent)
    private var storage: [String: Data] = [:]

    func read(_ key: String) -> Data? {
        queue.sync { storage[key] }
    }

    func write(_ data: Data, key: String) {
        queue.async(flags: .barrier) {
            self.storage[key] = data
        }
    }
}
```

Mọi access qua cùng custom concurrent queue. barrier exclusive chỉ trong queue này; không áp dụng global queue để bảo vệ arbitrary state. Write async trả trước commit; nếu caller cần completion, cung cấp callback/async API. Không gọi queue.sync từ block đang chạy trên cùng serial queue hoặc barrier gây deadlock. Swift 6 strict concurrency cần isolation/Sendable phù hợp; actor thường đơn giản hơn.

DispatchWorkItem.cancel đánh dấu cancellation, work đang chạy phải check flag/cooperate. asyncAfter(deadline:) schedule delayed work, không đảm bảo chính xác khi system bận. QoS mô tả importance/latency/energy, không phải chỉ định thread; userInitiated cho user chờ, utility dài có progress, background maintenance. Không dùng QoS để thực hiện UIKit off main.

**Q: Barrier hoạt động trên global concurrent queue?** Không dùng global để đảm bảo exclusive như custom concurrent queue.

**Q: GCD cancel task đang chạy?** Cooperative; không forcibly kill block. URLSession task cần cancel riêng.

**Q: Deadlock vs race?** Deadlock chờ vòng tròn không tiến triển; race phụ thuộc interleaving. Lock/queue sai có thể gây cả hai.

**Q: GCD vs Actor?** GCD manual synchronization, Actor compiler isolation/structured concurrency integration, vẫn logical race qua await.

**Bài tập:** Cache concurrent read/write, test state consistency; mô phỏng deadlock bằng lý thuyết, không chạy main.sync trên main.
