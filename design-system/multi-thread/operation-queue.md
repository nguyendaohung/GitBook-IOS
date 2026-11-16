# OPERATION QUEUE

## Operation Queue

Cũng là một cách để thực hiện đa luồng trong iOS. Giúp quản lý/tracking sâu hơn vào 1 tác vụ (operation) trong queue.Operation: Sẽ đóng gói và thực hiện 1 task.

OperationQueue: Là nơi chứa nhiều operation. Operation có thể huỷ (cancel), tạm dừng (pause). Cũng có thể kiểm tra trạng thái của task đó như thế nào (Đang làm, làm xong rồi).An operation can be in several states, depending on its current execution status.

<figure><img src="https://2560383644-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FJJeHgPhibgG2MUnhhB9K%2Fuploads%2F63BstCXvYQxZQZHerPhq%2Fmultiple-threads-operation-queue.png?alt=media&#x26;token=8db66fda-e895-48a7-a4d8-5a6609062f95" alt=""><figcaption></figcaption></figure>

* **Ready:** It’s prepared to start
* **Executing:** The task is currently running
* **Finished:** Once the process is completed
* **Cancelled:** The task cancelled

Chúng ta thêm task vào hàng đợi bằng cách sử dụng BlockOperation. Có thể sử dụng method start() để bắt đầu task hoặc thêm BlockOperation đó vào OperationQueue (khuyến khích cách này)

**Ưu điểm:** là 1 số ý đã nêu ở nhược điểm GCD
