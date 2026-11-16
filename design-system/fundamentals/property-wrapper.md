# Property Wrapper

* **Property Wrappers** là một tính năng mới được giới thiệu từ Swift 5.1. Nó cung cấp cách để quản lý các thuộc tính trong các lớp và cấu trúc của chúng ta một cách dễ dàng và linh hoạt hơn. Property Wrappers cho phép chúng ta đóng gói logic quản lý thuộc tính vào một đối tượng riêng biệt, được gọi là "Wrapper", mà có thể được sử dụng bên trong các thuộc tính để cung cấp các tính năng bổ sung.

\
Dưới đây là một ví dụ đơn giản về cách sử dụng Property Wrappers trong Swift:

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 13.01.38.png" alt=""><figcaption></figcaption></figure>

Ở đây, chúng ta định nghĩa một Property Wrapper mới được gọi là `Trimmed`, nó sẽ loại bỏ các khoảng trắng và dấu xuống dòng không cần thiết của chuỗi được gán cho thuộc tính đó. Sau đó, chúng ta sử dụng Property Wrapper này trong một cấu trúc `User` để đảm bảo rằng các trường `name`, `email`, và `password` được làm sạch trước khi được lưu trữ.
