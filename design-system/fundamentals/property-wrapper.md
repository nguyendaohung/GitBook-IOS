# Property Wrapper

* **Property Wrappers** là một tính năng mới được giới thiệu từ Swift 5.1. Nó cung cấp cách để quản lý các thuộc tính trong các lớp và cấu trúc của chúng ta một cách dễ dàng và linh hoạt hơn. Property Wrappers cho phép chúng ta đóng gói logic quản lý thuộc tính vào một đối tượng riêng biệt, được gọi là "Wrapper", mà có thể được sử dụng bên trong các thuộc tính để cung cấp các tính năng bổ sung.

\
Dưới đây là một ví dụ đơn giản về cách sử dụng Property Wrappers trong Swift:

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 13.01.38.png" alt=""><figcaption></figcaption></figure>

Ở đây, chúng ta định nghĩa một Property Wrapper mới được gọi là `Trimmed`, nó sẽ loại bỏ các khoảng trắng và dấu xuống dòng không cần thiết của chuỗi được gán cho thuộc tính đó. Sau đó, chúng ta sử dụng Property Wrapper này trong một cấu trúc `User` để đảm bảo rằng các trường `name`, `email`, được làm sạch trước khi lưu; không tự động trim password.

## wrappedValue, projectedValue và storage

```swift
@propertyWrapper
struct Clamped {
    private var value: Int
    private let range: ClosedRange<Int>

    init(wrappedValue: Int, _ range: ClosedRange<Int>) {
        self.range = range
        value = min(max(wrappedValue, range.lowerBound), range.upperBound)
    }

    var wrappedValue: Int {
        get { value }
        set { value = min(max(newValue, range.lowerBound), range.upperBound) }
    }
}

struct Settings {
    @Clamped(0...100) var volume = 50
}
var settings = Settings()
settings.volume = 150 // 100
```

Compiler sinh backing storage _volume; wrappedValue là property interface; projectedValue nếu khai báo tạo $volume. SwiftUI @State có projected Binding, @Published có projected Publisher, @AppStorage liên kết UserDefaults. Wrapper không tự thread-safe, không dùng trim password vì có thể thay credential người dùng nhập. SwiftData @Model và Observation @Observable là macro, không phải property wrapper.

**Q: @State vs @Binding?** State sở hữu storage; Binding đọc/ghi storage của owner.

**Q: Property wrapper vs macro?** Wrapper đóng gói access/storage behavior một property; macro sinh/biến đổi code theo vai trò, ví dụ @Model.

**Q: Wrapper có thay computed property?** Wrapper phù hợp logic tái sử dụng nhiều property; computed property đơn giản khi chỉ một chỗ.

**Bài tập:** Viết wrapper range validation, test initial value và mutation; so sánh @Published với @State.
