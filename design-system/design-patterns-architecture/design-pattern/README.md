# Design Pattern

Link tham khảo design pattern: [https://www.kodeco.com/books/design-patterns-by-tutorials/v3.0](https://www.kodeco.com/books/design-patterns-by-tutorials/v3.0)

Có 3 nhóm Design Patterns chính:

1. Creational Pattern (nhóm khởi tạo): Singleton, …. ⇒ Nhóm này sẽ giúp bạn trong việc khởi tạo đối tượng thông qua một số thủ thuật
   * [Singleton](singleton.md)
2. Structural Pattern (nhóm cấu trúc): Adapter, Facade, Decorator,… ⇒ Dùng để thiết lập, định nghĩa quan hệ giữa các đối tượng
3. Behavioral Pattern (nhóm hành vi): Observer … ⇒ Dùng trong thực hiện các hành vi của đối tượng
   * [Strategy](strategy-pattern.md)
   * [Observer](observer-pattern.md)

## Câu hỏi phỏng vấn tổng quan

**Q: Ba nhóm GoF patterns?** Creational (Factory/Singleton), Structural (Adapter/Facade/Decorator), Behavioral (Observer/Strategy/State); phân loại theo vấn đề giải quyết.

**Q: Adapter vs Facade?** Adapter đổi interface để tương thích; Facade đơn giản hóa một subsystem; có thể dùng để che SDK bên thứ ba.

**Q: Singleton vs Dependency Injection?** Singleton quản lý một instance/global access; DI cung cấp dependency explicit, có thể inject shared instance mà không global accessor.

**Q: Observer vs Strategy?** Observer phân phối event, Strategy chọn algorithm; khác use case, có thể cùng tồn tại.
