# Architecture vs Design Pattern

* Architecture trong iOS sử dụng Swift: Trong iOS, có nhiều architecture khác nhau mà các nhà phát triển có thể sử dụng, bao gồm MVC (Model-View-Controller), MVVM (Model-View-ViewModel), VIPER, vv. Chúng ta cần chọn một architecture phù hợp với yêu cầu của dự án và tạo ra một cấu trúc dễ quản lý, dễ mở rộng và dễ bảo trì.
* Design Pattern trong iOS sử dụng Swift: Cũng giống như trong lập trình chung, trong iOS cũng có nhiều design pattern khác nhau mà các nhà phát triển có thể sử dụng, bao gồm Singleton, Factory, Decorator, vv. Chúng ta cần chọn design pattern phù hợp với yêu cầu của dự án và tạo ra một giải pháp tổng quan cho vấn đề đang gặp phải.

Giống nhau: chúng đều là pattern. Mục đích là giải quyết 1 cấu trúc phức tạp thành những phần nhỏ đơn giản hơn

Khác nhau: ở một mức cao, chúng ta sử dụng architect pattern để giải quyết. Ở một mức độ thấp hơn, chúng ta sử dụng design pattern. Mức độ cơ bản nhất (implememtation level), chúng ta sử dụng programming paradigms

## So sánh qua một feature

Architecture quyết định dependency/layer và flow toàn module/app; design pattern giải quyết vấn đề thiết kế lặp lại trong phạm vi nhỏ. Programming paradigm như OOP/POP/reactive là cách biểu diễn code. Ví dụ catalog: MVVM + Clean tổ chức presentation/domain/data; Repository abstract data source; Strategy chọn cache policy; Observer nhận state; Coordinator navigation. Không bắt buộc chọn chỉ một pattern.

**Q: MVVM và Singleton có đối lập?** Không, MVVM là presentation architecture, Singleton là creational pattern; service singleton có thể được inject nhưng tránh global state.

**Q: Clean Architecture vs MVVM?** Clean là dependency/boundary, MVVM là Presentation; kết hợp được.

**Q: Khi nào thêm layer?** Khi có lý do business rule, testability, reuse, independent change; tránh layer chỉ forward call.

**Q: Design pattern có phải framework?** Không, là giải pháp thiết kế; framework có thể hỗ trợ implementation.
