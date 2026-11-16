# Any Object

Trong Swift, `AnyObject` được sử dụng để đại diện cho bất kỳ đối tượng class nào. Nó cho phép ta lưu trữ và thao tác với các đối tượng class mà không cần biết chính xác loại đối tượng đó là gì.

\
Ví dụ, giả sử bạn có một hàm nhận đối tượng class làm tham số đầu vào:

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-27 at 16.30.17.png" alt=""><figcaption></figcaption></figure>

Trong trường hợp này, hàm `processObject` chỉ chấp nhận đối tượng của lớp `MyClass` và sẽ bị lỗi nếu đối tượng khác lớp `MyClass` được truyền vào. Tuy nhiên, nếu bạn muốn truyền vào một đối tượng class bất kỳ, bạn có thể sử dụng `AnyObject` thay vì `MyClass`:

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-27 at 16.31.00.png" alt=""><figcaption></figcaption></figure>

Trong trường hợp này, hàm `processObject` có thể chấp nhận bất kỳ đối tượng class nào và sẽ không bị lỗi khi đối tượng khác lớp `MyClass` được truyền vào.

Tương tự, nếu bạn muốn truyền một đối tượng class không biết trước kiểu dữ liệu của nó, bạn cũng có thể sử dụng `AnyObject`:

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-27 at 16.32.15.png" alt=""><figcaption></figcaption></figure>

## Any, AnyObject, any Protocol và some Protocol

Any chứa giá trị bất kỳ (struct, class, enum, function); AnyObject là class-bound existential, có Objective-C bridging khi tương tác Foundation. any Protocol chứa conforming value, có thể là struct/class nếu protocol không class-bound. some Protocol che một concrete type cố định; generic giữ type relationship.

```swift
protocol EventHandler: AnyObject {
    func handle()
}

final class Owner {
    weak var delegate: (any EventHandler)?
}

let values: [Any] = [1, "Swift", true]
for value in values {
    if let number = value as? Int { print(number) }
}
```

**Q: Tại sao delegate thường AnyObject?** Cho phép weak reference để tránh retain cycle, chỉ class instance conform.

**Q: as? vs as!?** as? trả optional khi cast fail; as! trap nếu type sai. Ưu tiên type-safe generic/protocol.

**Q: AnyObject có phải mọi Swift type?** Không, chỉ class reference trong type system; một số value có thể bridge sang Foundation object.

**Q: Any vs any Protocol?** Any không đảm bảo method; existential protocol giữ contract có thể gọi, tùy associated type constraints.
