# OPERATION QUEUE

## Operation Queue

Cũng là một cách để thực hiện đa luồng trong iOS. Giúp quản lý/tracking sâu hơn vào 1 tác vụ (operation) trong queue.Operation: Sẽ đóng gói và thực hiện 1 task.

OperationQueue: Là nơi chứa nhiều operation. Operation có cooperative cancel; queue có thể suspend việc bắt đầu operation mới, không pause operation đang chạy. Cũng có thể kiểm tra trạng thái của task đó như thế nào (Đang làm, làm xong rồi).An operation can be in several states, depending on its current execution status.

<figure><img src="https://2560383644-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FJJeHgPhibgG2MUnhhB9K%2Fuploads%2F63BstCXvYQxZQZHerPhq%2Fmultiple-threads-operation-queue.png?alt=media&#x26;token=8db66fda-e895-48a7-a4d8-5a6609062f95" alt=""><figcaption></figcaption></figure>

* **Ready:** It’s prepared to start
* **Executing:** The task is currently running
* **Finished:** Once the process is completed
* **Cancelled:** The task cancelled

Chúng ta thêm task vào hàng đợi bằng cách sử dụng BlockOperation. Có thể sử dụng method start() để bắt đầu task hoặc thêm BlockOperation đó vào OperationQueue (khuyến khích cách này)

**Ưu điểm:** là 1 số ý đã nêu ở nhược điểm GCD

## OperationQueue chuyên sâu

```swift
import Foundation

let queue = OperationQueue()
queue.maxConcurrentOperationCount = 2
let decode = BlockOperation { print("Decode") }
let save = BlockOperation { print("Save") }
save.addDependency(decode)
queue.addOperations([decode, save], waitUntilFinished: false)
```

Dependency chờ operation finished, không tự yêu cầu success; nếu decode failed/cancelled, save cần kiểm tra result/cancel policy. OperationQueue tự start, không gọi start() thêm sau enqueue. isSuspended ngăn operation mới bắt đầu, không pause work đang chạy. cancel cooperative, isCancelled flag, custom async Operation phải quản lý isExecuting/isFinished KVO đúng, kết thúc khi URLSession callback xong. BlockOperation chỉ start async URLSession rồi return sẽ đánh dấu finished sớm, maxConcurrentOperationCount không giới hạn request thật.

**Q: OperationQueue vs DispatchQueue?** Operation object dependency/state/cancel/limit; GCD lightweight queue.

**Q: cancelAllOperations có dừng ngay?** Không, đánh dấu cancel; running work cần cooperate/cancel underlying task.

**Q: Operation dependency có propagate error?** Không tự động, dependency chỉ finished; thiết kế result/error propagation.

**Q: Download 1000 ảnh dùng BlockOperation start task?** Không đủ, phải async Operation lifecycle đúng hoặc TaskGroup bounded concurrency.

**Bài tập:** 3 operations fetch/decode/save, dependency và cancellation; kiểm tra max 2 active, test lỗi dependency.
