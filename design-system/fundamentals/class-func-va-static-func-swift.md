# class func và static func swift

Cả `class func` và `static func` trong Swift đều là các phương thức tĩnh (static method) mà không cần tạo ra một thể hiện đối tượng (instance) của lớp để gọi.

Tuy nhiên, hai loại phương thức này có một số điểm khác biệt như sau:

1. `class func` có thể được ghi đè bởi các lớp con (subclass) và được kế thừa, trong khi `static func` không thể.
2. `class func` được sử dụng khi bạn muốn xử lý trường hợp đa hình (polymorphism) và muốn các lớp con có thể ghi đè phương thức đó. Trong khi đó, `static func` được sử dụng khi bạn muốn xác định một phương thức độc lập với bất kỳ lớp nào và không muốn nó bị ghi đè.

\
Ví dụ, giả sử bạn có một lớp `Shape` đại diện cho các hình học và bạn muốn có một phương thức tính diện tích chung cho tất cả các hình học. Bạn có thể khai báo phương thức `calculateArea()` như sau:

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-23 at 15.01.48.png" alt=""><figcaption></figcaption></figure>

Trong trường hợp này, phương thức `calculateArea()` được khai báo là một `class func` để cho phép các lớp con (ví dụ như `Rectangle`, `Circle`,...) có thể ghi đè phương thức này nếu cần thiết.

Tuy nhiên, nếu bạn muốn có một phương thức tính chu vi cố định cho một hình tròn với bán kính được cung cấp, bạn có thể khai báo phương thức như sau:

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-23 at 15.02.52.png" alt=""><figcaption></figcaption></figure>

Trong trường hợp này, phương thức `calculateCircumference()` được khai báo là một `static func` để xác định một phương thức độc lập với bất kỳ lớp nào, và không muốn nó bị ghi đè.

## Ví dụ chạy được và Q&A

```swift
class Formatter {
    class func name() -> String { "Base" }
    static func version() -> Int { 1 }
}
final class JSONFormatter: Formatter {
    override class func name() -> String { "JSON" }
}
let type: Formatter.Type = JSONFormatter.self
print(type.name()) // JSON
print(JSONFormatter.version()) // 1, inherited
```

static method của class có thể được kế thừa nhưng không override; class func có dynamic dispatch. struct/enum dùng static, không class func. Type method không cần instance, có thể truy cập static/class members phù hợp; không truy cập instance properties nếu không có instance.

**Q: static vs class?** static không override; class func override trong subclass.

**Q: final class func có ý nghĩa?** Ngăn override như static đối với method class; chọn API rõ ràng.

**Q: Protocol static requirement dùng class func implement được?** Class có thể đáp ứng bằng static/class method theo requirement; dynamic behavior tùy implementation.
