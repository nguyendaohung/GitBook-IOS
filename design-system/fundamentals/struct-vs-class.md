# Struct vs Class

Chúng ta có thể liệt kê một số điểm **giống nhau** giữa Struct và Class trong Swift như sau:

* Công dụng: Giúp cấu hình/định nghĩa 1 object.
* Properties: struct và class đều có property (thuộc tính).
* Method: cả struct và class đều có method (phương thức).
* Initializier: struct và class hỗ trợ các phương thức khởi tạo mặc định, hoặc các phương thức khởi tạo tuỳ chỉnh theo mục đích và logic của lập trình viên.
* Subscript: Struct và class hỗ trợ các subscript syntax (câu lệnh con bên trong một thuộc tính, hoặc một hàm).
* Extension: Extension là khái niệm mới trong Swift, nó giúp lập trình viên có thể mở rộng các struct, hoặc class sẵn có hoặc đã được xây dựng từ trước.

**Khác nhau:**

* Type (kiểu): struct là **value type(tham trị)** còn class là **reference type(tham chiếu)**

**Value Type**: Mỗi instance giữ một bản sao duy nhất dữ liệu của nó. Nó sẽ tạo một instance mới (bản sao) khi được gán cho một biến, hoặc hằng, hoặc khi được truyền cho một hàm. Do đó, khi thay đổi giá trị của 1 instance, những instance khác không bị thay đổi theo. Xem ví dụ để hiểu hơn về khái niệm này

```Swift
struct Employee {
    var code: Int
    var name: String
    var address: String
}

var a = Employee(code: 5, name: "BXH", address: "Sun*")
var b = a
b.name = "MH"
print(a.name) //  In ra "BXH"
print(b.name) // In ra "MH"

// Nhận thấy instance a khi được gắn cho b, nó sẽ copy giá
// trị và tạo ra một instance mới có giá trị giống với a nhưng
// độc lập với a. Do đó khi thay đổi name của b, name của a
// không bị thay đổi theo 
```

**Reference Type**: Mỗi instance chia sẻ một bản sao dữ liệu. Một kiểu mà khi được khởi tạo, khi được gán cho một biến hoặc hằng hoặc khi được truyền cho một hàm, sẽ trả về một tham chiếu đến cùng instance hiện có. Khái niệm thật khó hiểu đúng không. Đọc ví dụ để dễ hiểu hơn nhé

```Swift
class Employee {
    var code: Int
    var name: String
    var address: String
    
    init(code: Int, name: String, address: String) {
        self.code = code
        self.name = name
        self.address = address
    }
}

var a = Employee(code: 5, name: "BXH", address: "Sun*")
var b = a
b.name = "MH"
print(a.name) // In ra  "MH"
print(b.name) // In ra "MH"
// Khi b được gắn bằng a, cả 2 sẽ cùng tham chiếu tới một
// instance. Do đó khi ta thay đổi name của b, name 
// của a cũng bị thay đổi theo
```

* Inheritance (kế thừa): Struct không thể kế thừa, còn class thì có thể ( hiển nhiên – OOP mà).
* Deinitializers: Struct không có hàm huỷ (destructor trong java/C++), chỉ có hàm khởi tạo initializer (constructor trong java/C++), còn Class thì có đầy đủ
* Class là kiểu Reference Type, do đó có thể thay đổi trực tiếp giá trị của các thuộc tính qua phương thức của nó. Còn với Struct ta phải khai báo từ khoá **mutating** trước phương thức để làm việc đó.
* Khi tạo Struct, sẽ mặc định sinh ra một hàm khởi tạo với đầy đủ tham số tương ứng với tất cả Stored Properties (gọi là Memberwise Initializer). Còn Class thì không có. Do đó khi tạo một class, ta sẽ phải khai báo Optional cho các thuộc tính hoặc phải tự định nghĩa một hàm khởi tạo.
* The way they deal with constants:
  * If you have a constant struct with a variable property, that property can’t be changed because the struct itself is constant.
  * If you have a constant class with a variable property, that property can be changed.

```swift
struct StructA {
    var name: String
}

class ClassB {
    var name: String
}

func viewDidLoad() {
    let structA = StructA(name: "Bradley")
    structA.name = "HT Kien" // Error: Cannot assign to property: 'structA' is a 'let' constant
    let classB = ClassB(name: "Hoang Trong Kien")
    classB.name = "Bradley"  // Success
}
```

* Multiple reference (đa tham chiếu): chúng ta có thể có nhiều đối tượng cùng tham chiếu đến 1 class instance, còn ở struct thì không thể (vì nó là value type mà).
* Tham chiếu tới đâu => Tham chiếu tới ô nhớ

**Khi nào sử dụng class or struct:**&#x20;

**Struct:**

*   Nếu dữ liệu của bạn là các dữ liệu đơn giản không cần các phương thức hoặc thuộc tính tính toán hoặc không cần sử dụng tính chất kế thừa, thì sử dụng Struct là lựa chọn tốt hơn. Structs là giá trị dữ liệu đại diện cho một đối tượng và sẽ được sao chép khi gán hoặc truyền vào hàm.



**Class:**

* Trong khi đó, nếu dữ liệu của bạn cần các phương thức hoặc thuộc tính tính toán hoặc cần sử dụng kế thừa thì sử dụng Class là lựa chọn tốt hơn. Classes là các đối tượng đại diện cho một đối tượng và chỉ có một bản sao chung cho tất cả các tham chiếu đến nó.

**Ví dụ:**

* Nếu bạn muốn mô tả một đối tượng như một địa chỉ, bạn có thể sử dụng một Struct để lưu trữ các thuộc tính như tên đường, thành phố, và quốc gia.

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-10 at 15.58.47.png" alt=""><figcaption></figcaption></figure>

* Nếu bạn muốn mô tả một đối tượng như một người dùng, bạn có thể sử dụng một Class để lưu trữ các thuộc tính như tên, tuổi, và email, và các phương thức như đăng nhập và đăng ký.

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-10 at 15.59.51.png" alt=""><figcaption></figcaption></figure>

Như vậy, trong trường hợp này, sử dụng Class cho phù hợp hơn vì nó cần phải có các phương thức để xử lý việc đăng nhập và đăng ký của người dùng. Trong khi đó, sử dụng Struct cho phù hợp hơn cho việc lưu trữ thông tin địa chỉ.
