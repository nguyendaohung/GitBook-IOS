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
