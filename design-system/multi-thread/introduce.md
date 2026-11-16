# INTRODUCE

### Giới thiệu

* Task: Hiểu đơn giản đó là 1 phần của công việc và cần phải thực hiện.​
* Processes: Là một đoạn mã code có thể thực hiện nhiều thread.​
* Thread: Một tập hợp các câu lệnh được thực hiện
* Multi Thread: Khi ứng dụng có nhiều hơn 1 thread
* Công dụng của đa luồng: Tăng hiệu năng của ứng dụng, giúp quản lý những công việc được thực hiện đồng thời

Có 2 loại thread chính: main thread và nhiều background thread. Main thread dùng để khởi tạo và quản lý UI. Background thread dùng để xử lý những task nặng hoặc lâu.

⇒ Khi call API async: không block main trong lúc chờ network; xử lý CPU-heavy phù hợp executor ⇒ update UI ở main actor

Có 3 cách để tạo đa luồng:

1. Tự tạo và quản lý thread manually
2. Sử dụng GCD
3. Sử dụng Operation Queue

⇒ Thường sử dụng cách 2, một số trường hợp cần sẽ sử dụng cách 3

## Concurrency vs Parallelism

Process có address space/resources; thread là execution context; queue là abstraction lập lịch work, không phải thread cố định. Concurrency là nhiều task tiến triển xen kẽ; parallelism là thực thi đồng thời trên nhiều core. Async có thể chạy trên main actor, không tự tạo thread. URLSession async/callback có thể được gọi từ main mà không block khi chờ network; CPU-heavy decode/mapping phải có isolation/executor phù hợp, UI update MainActor.

**Q: Thread vs queue?** Thread thực thi instructions; queue sắp xếp work, scheduler chọn thread; main queue gắn main thread.

**Q: Concurrency vs parallelism?** Concurrency nhiều work in progress; parallelism thực thi cùng lúc, phụ thuộc hardware/scheduler.

**Q: Race condition là gì?** Kết quả phụ thuộc timing/interleaving; data race là truy cập memory đồng thời không đồng bộ có write. Dùng actor/lock/serial isolation.

**Q: GCD vs OperationQueue vs async/await?** GCD queue low-level, Operation dependency/cancel/limit, Swift Concurrency structured tasks/actor isolation.
