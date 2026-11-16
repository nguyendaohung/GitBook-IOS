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
