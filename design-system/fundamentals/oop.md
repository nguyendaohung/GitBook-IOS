# OOP

### Tính kế thừa (Inheritance)

Sub-class có thể kế thừa toàn bộ các property, method của base-class. Trong swift không ủng hộ việc đa kế thừa bằng class

### Tính đóng gói (Encapsulation)

Các dữ liệu liên quan sẽ được đóng gói thành một lớp. Giúp che giấu dữ liệu với các đối tượng bên ngoài

Trong 1 class có thể hạn chế phạm vi truy xuất đến các property/method thuộc class ra bên ngoài. Trong swift thường sử dụng access modifier như là keyword để hạn chế điều này

### Tính đa hình (Polymorphism)

Tính đa hình thể hiện ở việc 1 object có thể đóng vai trò ở nhiều hình dạng. Một hành động, hay phương thức (method) sẽ được thực hiện bằng nhiều cách khác nhau.

&#x20;Trong swift, có thể override method như là 1 cách thể hiện tính đa hình

### Tính trừu tượng (Abstraction)

Giúp cho bạn khái quát hóa một lớp. Mang tính chất thiết kế hơn là triển khai các lớp. Vì chỉ có khai báo các thuộc tính & phương thức. Chứ không định nghĩa cụ thể từng cái ra.&#x20;

Trong Swift, chúng ta có thể thể hiện tính trừu tượng bằng cách sử dụng protocol. Một protocol là một tập hợp các quy tắc hoặc chuẩn cho các thuộc tính và hành vi của một đối tượng.

BONUS: 1 lớp có thể là sự trừu tượng hoá, mang những cốt lõi cần thiết về property/method cho các lớp con. Phục vụ cho việc kế thừa đúng bản chất. POP(Protocol-Oriented Programming) về bản chất cũng được xem như 1 cách trừu tượng hoá, vì POP sinh ra để phục vụ cho việc tách rời các thành phần khỏi sự thực thi ( tức là ko bao gồm implement, trừ khi extension protocol bên trong Swift chúng ta có cung cấp default implementation)

[https://fxstudio.dev/lap-trinh-huong-doi-tuong-oop-voi-swift/](https://fxstudio.dev/lap-trinh-huong-doi-tuong-oop-voi-swift/)

## Ví dụ 4 tính chất OOP trong Swift

```swift
class PaymentMethod {
    private(set) var total = 0 // encapsulation
    func pay(amount: Int) {
        guard amount > 0 else { return }
        total += amount
    }
}
final class CardPayment: PaymentMethod {
    override func pay(amount: Int) {
        // Illustrative: real payment belongs to verified backend flow.
        super.pay(amount: amount)
    }
}
let method: PaymentMethod = CardPayment()
method.pay(amount: 100) // polymorphism
```

Abstraction: protocol PaymentService mô tả contract, caller không biết implementation. Inheritance chia sẻ behavior theo is-a; composition dùng has-a, linh hoạt hơn khi ghép services. Encapsulation bảo vệ invariant, không chỉ private mọi property. Swift class single inheritance, nhiều protocol conformances; struct/enum có thể conform protocol và value semantics.

**Q: OOP vs POP?** OOP dùng object/inheritance/polymorphism; POP dùng protocol/composition/default implementation, áp dụng cả value types. Có thể kết hợp.

**Q: Overload vs override?** Overload nhiều signature cùng tên; override thay implementation inherited theo dynamic dispatch.

**Q: Composition vs inheritance?** Composition ghép behavior, giảm coupling; inheritance khi is-a và substitutability đúng.

**Q: Encapsulation khác abstraction?** Encapsulation giới hạn truy cập/invariant; abstraction mô tả contract, che implementation.
