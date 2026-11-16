# Clousure: Escaping và Non-escaping

### Mục đích sử dụng Closure

* Sử dụng chúng như là biến/hằng
* Sử dụng như các tham số trong khai báo 1 function.
* Sử dụng làm đối số để truyền vào các lời gọi hàm.
* Sử dụng như là các property trong class, struct, enum.
* Tái sử dụng hay kế thừa lại.

&#x20;**nonescaping:** Một closure (hoặc hàm) là non-escaping khi nó được gọi và hoàn thành trong cùng một scope (phạm vi), và không được truyền ra ngoài scope đó hoặc lưu trữ trong một biến để sử dụng sau này.

* Life cycle của một **nonescaping closure** rất đơn giản:
  * Closure được pass vào function argument.
  * Function excute closure. Closure được giải phóng khỏi bộ nhớ.
  * Function return.
* Khi sử dụng **nonescaping closure,** compiler cũng không cần reference tới **self**, không tạo cycle do chính việc lưu parameter ra ngoài scope; vẫn cần xem ownership của code bên trong. Không mặc định cần `[weak self]` hoặc `[unowned self]`.

&#x20;**@escaping:** Một closure (hoặc hàm) là escaping khi nó được truyền ra ngoài scope hiện tại hoặc lưu trữ trong một biến để sử dụng sau này. Vì vậy, nó có thể được gọi hoặc sử dụng sau khi scope ban đầu đã kết thúc.

* Sử dụng **@escaping** khi cần giữ lại closure thành property của class để chờ excute sau.
  * Để closure có thể được giữ lại trong bộ nhớ sau khi function excute xong và return compiler

## Ví dụ và ownership

Closure là function value có thể capture biến/context. Tham số closure mặc định nonescaping; @escaping cho phép lưu hoặc gọi sau khi hàm trả về. Nonescaping không bắt buộc phải được gọi; có thể gọi 0 hoặc nhiều lần trong lifetime cho phép. Escaping không đồng nghĩa async: closure có thể được gọi đồng bộ dù khai báo escaping.

```swift
func transform(_ value: Int, using work: (Int) -> Int) -> Int {
    work(value)
}

final class Worker {
    var completion: (() -> Void)?
    func setCompletion(_ completion: @escaping () -> Void) {
        self.completion = completion
    }
}

let worker = Worker()
worker.setCompletion { [weak worker] in
    print(worker != nil)
}
```

Capture list [weak self] không giữ owner; unowned không giữ nhưng trap nếu owner deallocated. Strong capture có thể đúng nếu work cần owner sống và không tạo cycle. Nonescaping parameter tự nó không sống quá function call; code bên trong vẫn có thể tạo một escaping closure khác/cycle. @Sendable khác @escaping, liên quan an toàn truyền closure qua concurrency boundary.

**Q: Closure escaping khi nào?** Khi có thể sống ngoài call, lưu property hoặc truyền vào API escaping.

**Q: Capture biến var như nào?** Closure thường capture storage; capture list [value] lấy snapshot tại thời điểm tạo closure.

**Q: weak self luôn cần?** Không, phụ thuộc ownership/lifetime. Cycle khi owner giữ closure và closure giữ owner.

**Q: @escaping có đồng nghĩa background?** Không. Scheduler/executor do API quyết định.
