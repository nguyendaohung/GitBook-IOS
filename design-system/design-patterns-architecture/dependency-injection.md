# Dependency Injection (DI)

### Định nghĩa

DI là cung cấp dependency từ bên ngoài thay vì để object tự tạo dependency. DIP là nguyên lý phụ thuộc abstraction; DI là một kỹ thuật để thực hiện DIP, không đồng nghĩa với framework DI.

```swift
protocol VoucherRepository {
    func vouchers() async throws -> [String]
}

final class VoucherViewModel {
    private let repository: any VoucherRepository

    init(repository: any VoucherRepository) {
        self.repository = repository
    }

    func load() async throws -> [String] {
        try await repository.vouchers()
    }
}

struct StubVoucherRepository: VoucherRepository {
    func vouchers() async throws -> [String] {
        ["Coffee voucher"]
    }
}
```

### Cách áp dụng

* Constructor injection: dependency bắt buộc, immutable; ưu tiên cho ViewModel/UseCase.
* Property injection: phù hợp dependency tùy chọn, dễ quên khởi tạo.
* Method injection: dependency chỉ cần cho một hành động.
* Composition root (App/Coordinator) tạo service, repository, use case, view model; tránh Service Locator/global singleton ẩn dependency.
* Scope: app-wide cho service có state chung; screen/session cho state theo màn hình/user. Không giữ ViewController trong singleton.

### Clean Architecture

Domain chứa entity, use case, repository protocol; Data implement repository, mapping DTO, remote/local data source. Presentation gọi use case và biến kết quả thành UI state. Dependency source code hướng vào Domain; runtime có thể gọi Data qua protocol. Coordinator quản lý navigation, không chứa nghiệp vụ.

### Q&A

**Q: Tại sao không tạo API service trực tiếp trong ViewModel?** Tăng coupling, khó test lỗi mạng/offline và thay implementation.

**Q: Có cần UseCase cho từng CRUD?** Không nhất thiết. Cân nhắc độ phức tạp và khả năng tái sử dụng; tránh layer chỉ forward call.

**Liên hệ:** Map4D MVVM + Clean Architecture; dùng fake repository để kiểm thử tìm địa chỉ không cần Map4D SDK/backend.

## Thiết kế module thực tế

### Dependency direction và composition root

```text
Presentation -> Domain (UseCase -> Repository protocol)
Data -> Domain (Repository implementation)
App/Composition root -> tất cả các module để lắp ghép
```

Ví dụ luồng danh sách sản phẩm: DTO từ server có snake_case; repository chuyển DTO sang Product domain; use case lọc sản phẩm còn hàng; ViewModel chuyển kết quả thành loading/content/error. ViewModel không cần biết URLSession hay Core Data. Nếu đổi API client, chỉ Data thay đổi.

```swift
struct Item: Sendable {
    let id: Int
    let inStock: Bool
}

protocol ItemRepository: Sendable {
    func allItems() async throws -> [Item]
}

struct LoadAvailableItems: Sendable {
    let repository: any ItemRepository

    func execute() async throws -> [Item] {
        try await repository.allItems().filter(\.inStock)
    }
}

struct FakeItems: ItemRepository {
    func allItems() async throws -> [Item] {
        [Item(id: 1, inStock: true), Item(id: 2, inStock: false)]
    }
}
```

Composition root tạo RemoteItemRepository(client:...), rồi LoadAvailableItems(repository:...), ViewModel(useCase:...). Test thay FakeItems mà không sửa production code.

### So sánh MVC, MVP, MVVM và Clean

| Mô hình | Trách nhiệm UI | Business logic | Hạn chế |
| --- | --- | --- | --- |
| MVC | Controller điều phối View | Model/service | Massive ViewController nếu thiếu boundary |
| MVP | View passive, Presenter điều phối | Presenter/use case | Nhiều protocol, callback |
| MVVM | View render state | ViewModel/use case | Massive ViewModel, binding phức tạp |
| Clean | Presentation là một layer | Domain độc lập | Nhiều mapping/module nếu áp dụng quá mức |

Clean Architecture không phải đối thủ MVVM; MVVM là cách tổ chức Presentation. Không bắt buộc mỗi button có UseCase. Tách khi có nghiệp vụ/reuse/testability rõ ràng.

### Dependency lifetime và Coordinator

App scope (API client, database stack), session scope (token cache, authenticated services), screen scope (ViewModel, task). Logout phải hủy task, clear session/cache phù hợp; tránh global singleton giữ user cũ. Coordinator nhận route và tạo màn hình; child coordinator cần remove khi flow kết thúc để tránh leak.

```swift
protocol Routing: AnyObject {
    func showDetail(id: Int)
}

final class ListPresenter {
    weak var router: (any Routing)?

    func didSelect(id: Int) {
        router?.showDetail(id: id)
    }
}
```

Weak router tránh vòng giữ giữa coordinator → screen/presenter → coordinator; bảo đảm owner thật vẫn giữ router.

### Q&A bổ sung

**Q: DI framework bắt buộc không?** Không. Constructor injection dễ trace và test; container hữu ích graph lớn nhưng có thể che dependency và lỗi resolution runtime.

**Q: Repository có phải chỉ bọc API?** Không, điều phối local/remote/cache, mapping và consistency; có thể bỏ abstraction thừa nếu app nhỏ.

**Bài tập:** Tách màn hình API đang gọi trực tiếp URLSession khỏi VC thành Repository, UseCase, ViewModel; test success/empty/failure với fake.

## Câu hỏi phỏng vấn tình huống

**Q: DI vs Service Locator?** DI dependency explicit trong constructor/property; Service Locator tự resolve hidden dependencies, khó test/trace.

**Q: Session-scoped dependency logout xử lý gì?** Cancel work, invalidate credential/cache theo user, tạo session scope mới, tránh old response ghi vào new session.

**Q: Mock vs fake repository?** Mock verify interactions; fake có implementation đơn giản, stub trả kết quả cố định. Test behavior.
