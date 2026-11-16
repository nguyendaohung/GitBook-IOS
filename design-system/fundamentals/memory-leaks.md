---
description: >-
  Link tham khảo:
  https://ali-akhtar.medium.com/all-about-memory-leaks-in-ios-cdd450d0cc34
---

# Memory leaks



> Swift sử dụng Automatic Reference Counting(ARC) để theo dõi và quản lý bộ nhớ ứng dụng. Trong đa số trường hợp, điều này nghĩa là sự quản lý chỉ làm việc trong Swift framework 1 cách tự động, và bạn không cần phải hiểu về quản lý bộ nhớ như nào. ARC tự động giải phóng bộ nhớ bởi việc khi 1 instances của class sẽ giải phóng khi không cần thiết nữa
>
> ARC (Automatic Reference Counting) là kỹ thuật quản lý bộ nhớ tự động được sử dụng trong Swift để quản lý việc giải phóng bộ nhớ của các đối tượng. Cơ chế hoạt động của ARC có thể được mô tả như sau:
>
> * Mỗi khi một đối tượng được khởi tạo, một tham chiếu đến đối tượng đó được tạo ra.
> * Khi một tham chiếu mới được tạo đến đối tượng, số lượng tham chiếu đến đối tượng đó tăng lên.
> * Khi một tham chiếu không còn được sử dụng đến đối tượng, số lượng tham chiếu đến đối tượng đó giảm đi.
> * Khi số lượng tham chiếu đến một đối tượng đạt đến 0, đối tượng đó sẽ được giải phóng và trả lại bộ nhớ.
>
> ARC tự động quản lý số lượng tham chiếu đến mỗi đối tượng trong quá trình thực thi ứng dụng. Nếu một đối tượng không còn được tham chiếu bởi bất kỳ biến hay hằng số nào, ARC sẽ giải phóng bộ nhớ của đối tượng đó.

### Định nghĩa

Memory leak là hiện tượng bộ nhớ không được giải phóng bởi ARC bởi khi không thể biết được cái không gian bộ nhớ này có đang còn được sử dụng hay không.

### Các rule để tránh memory leak

* Tại sao lại bị memory leak:
  * Khi 2 đối tượng giữ strong reference đến nhau. Gây ra hiện tượng không bên nào có thể được giải phóng cả. ⇒ Rule 1: Khi 2 đối tượng có liên kết với nhau, xác định ownership và dùng weak/unowned cho back-reference nếu phù hợp
  * Memory leak ở closure. Closure là 1 khối lệnh, nếu khối lệnh này giữ strong reference đến lớp khác thì có thể dẫn đến không bên nào được giải phóng ⇒ Sử dụng weak và unowned.
  * Sử dụng các công cụ debug memory: Bạn có thể sử dụng các công cụ như Instruments để debug memory và tìm ra các memory leaks.
  * &#x20;Sử dụng singleton trong iOS có thể dẫn đến memory leak nếu bạn không chú ý đến việc hủy nó. => quản lý ownership và cleanup; lazy không tự tránh leak
  * Sử dụng bộ nhớ một cách có hiệu quả: Để tránh memory leak trong trường hợp này, bạn nên sử dụng các kỹ thuật tối ưu để xử lý và tải dữ liệu. Ví dụ, bạn nên sử dụng các thư viện xử lý ảnh để giảm dung lượng của ảnh, hoặc sử dụng các kỹ thuật phân trang để tải dữ liệu từng phần
* Khác nhau giữa weak và unowned:
  * weak: không phải là 1 strong reference, có thể bằng nil. Sử dụng weak khi không chắc khi closure thực hiện thì reference có tồn tại hay không
  * unowned: không phải là 1 strong reference, không thể bằng nil. Sử dụng unowned khi chắc chắn closure luôn có thể thực hiện

⇒ Rule 2: Khi bạn có 1 lớp và có 1 closure trong lớp đó. Nếu ở closure truy cập vào bất cứ cái gì ở lớp đó thì cân nhắc weak/unowned khi có cycle hoặc lifetime không bảo đảm

## ARC, retain cycle và thực hành

Không phải mọi reference hai chiều đều cần weak cả hai phía. Thiết kế owner mạnh, back-reference/delegate weak; unowned chỉ khi lifetime invariant chắc chắn. Lazy singleton không tự giải quyết leak; singleton thường sống app lifetime, không giữ VC/subscription user cũ.

```swift
protocol ScreenDelegate: AnyObject {
    func didFinish()
}

final class Child {
    weak var delegate: (any ScreenDelegate)?
    var completion: (() -> Void)?
    deinit { print("Child released") }
}

final class Parent: ScreenDelegate {
    let child = Child()
    init() {
        child.delegate = self
        child.completion = { [weak self] in self?.didFinish() }
    }
    func didFinish() {}
    deinit { print("Parent released") }
}
```

Parent strong child; child weak parent. Closure capture self mạnh nếu child giữ completion sẽ tạo cycle. Weak reference tự nil khi dealloc; unowned trap khi truy cập object đã dealloc. Closure one-shot có thể strong capture và clear callback sau completion nếu lifetime đúng. Task giữ captured values qua await; cancellation không tự giải phóng nếu task không kết thúc. Timer/CADisplayLink/NotificationCenter block observer/Combine/Rx có lifetime cần cleanup.

**Q: ARC vs garbage collection?** ARC reference count, giải phóng khi strong references hết; không tự collect strong cycle.

**Q: weak vs unowned?** weak optional tự nil, unowned giả định lifetime và trap nếu sai.

**Q: Memory leak vs memory spike?** Leak object còn giữ khi không cần; spike allocation tạm, cache growth không nhất thiết leak. Dùng Memory Graph/Allocations/Leaks.

**Q: DisposeBag có ngăn mọi cycle?** Không, nếu self giữ bag và closure giữ self thì bag không deinit.

**Q: Task [weak self] có luôn tránh giữ self?** Nếu guard let self trước await, strong self có thể được giữ xuyên await; thiết kế lifetime và cancel.
