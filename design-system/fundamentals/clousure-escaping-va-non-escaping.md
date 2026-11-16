# Clousure: Escaping và Non-escaping

### Mục đích sử dụng Closure

* Sử dụng chúng như là biến/hằng
* Sử dụng như các tham số trong khai báo 1 function.
* Sử dụng làm đối số để truyền vào các lời gọi hàm.
* Sử dụng như là các property trong class, struct, enum.
* Tái sử dụng hay kế thừa lại.

&#x20;**@nonescaping:** Một closure (hoặc hàm) là non-escaping khi nó được gọi và hoàn thành trong cùng một scope (phạm vi), và không được truyền ra ngoài scope đó hoặc lưu trữ trong một biến để sử dụng sau này.

* Life cycle của một **@nonescaping closure** rất đơn giản:
  * Closure được pass vào function argument.
  * Function excute closure. Closure được giải phóng khỏi bộ nhớ.
  * Function return.
* Khi sử dụng **@nonescaping closure,** compiler cũng không cần reference tới **self**, không xảy ra trường hợp reference cycle. Vì vậy không cần sử dụng `[weak self]` hoặc `[unowned self]`.

&#x20;**@escapting:** Một closure (hoặc hàm) là escaping khi nó được truyền ra ngoài scope hiện tại hoặc lưu trữ trong một biến để sử dụng sau này. Vì vậy, nó có thể được gọi hoặc sử dụng sau khi scope ban đầu đã kết thúc.

* Sử dụng **@escapting** khi cần giữ lại closure thành property của class để chờ excute sau.
  * Để closure có thể được giữ lại trong bộ nhớ sau khi function excute xong và return compiler

