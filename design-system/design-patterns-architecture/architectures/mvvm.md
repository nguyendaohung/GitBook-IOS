# MVVM

### Định nghĩa

MVVM là viết tắt của Model - View - ViewModel, đây là mô hình hỗ trợ two-way data binding giữa View và View Model​. Là 1 phiên bản nâng cấp của MVC.  Mục đích là giúp khả năng kiểm thử và giảm công việc ở Controller.&#x20;

*   Controller: Controller được giảm công việc và xem như là View. Chỉ thực hiện những thứ liên quan đến UI — Show/get thông tin, một phần của view layer, hiển thị dữ liệu lên giao diện,truyền sự kiện người dùng về cho ViewModel xử lí, lấy dữ liệu từ ViewModel và update lại giao diện​

    ​
* View: Nhận sự kiện của người dùng tác động, rồi truyền về controller (không phải truyền trực tiếp cho view model)​
* Model: Nó giống với model trong mô hình MVC, Model đại diện cho dữ liệu đơn giản, Nó đơn giản nắm giữ dữ liệu và không làm bất cứ thứ gì liên quan tới logic,nó được sử dụng bởi ViewModel và update bất cứ khi nào ViewModel gửi update mới.​
* ViewModel: Nó là ánh xạ của View về mặt dữ liệu. Là nơi xử lý logic, những business trong app. Nhận thông tin từ VC, xử lý tất cả những thông tin này và gửi lại nó cho VC, ViewModel là thành phần chính của mô hình này. ViewModel không biết View là gì và những gì View làm. Điều này làm cho mô hình này dễ kiểm tra hơn và loại bỏ sự phức tạp từ view.​

Một số nguyên tắc phụ trợ khi áp dụng mô hình MVVM:​

* View nhận sự kiện của người dùng tác động, rồi truyền về controller (không phải truyền trực tiếp cho view model)​
* View model không chứa các xử lý về UI (không import UIKit)​
* Controller chịu trách nhiệm update dữ liệu lên view (không phải view model)​
* Controller nhận các tín hiệu update dữ liệu từ view model bằng 2 cách:​
  * Callback (closure & protocol)​
  * Observer (notification, KOV)​
* Một view model có thể được dùng cho nhiều controller (view model là sự ánh xạ của 1 đối tượng của model lên view, trường hợp nhiều controller dùng chung 1 loại đối tượng model và tương đồng về mặc hiển thị, thì có thể dùng chung view model)​

### Ưu điểm

* Là phiên bản nâng cấp của MVC, nên vẫn duy trì cấu trúc của MVC. Có nghĩa là ta có thể migrate 1 cái app có sẵn từ mô hình MVC sang mô hình MVVM
* Giảm khối lượng code chứa trong Controller
* Đưa các xử lý logic + business vào ViewModel ⇒ Code trở nên dễ hiểu, dễ maintain hơn
* Có khả năng kiểm thử.

### Nhược điểm

* Là mô hình khá phức tạp đối với người bắt đầu. Không phù hợp với những dự án nhỏ.
* Vì tương tự với MVC, nên không thể giải quyết triệt để vấn đề phình code của MVC. Giờ đây code có thể phình to ra ở ViewModel

## MVVM chuyên sâu: State, Binding và testability

MVVM không bắt buộc two-way binding. UIKit có thể dùng closure, delegate, RxSwift/Combine; SwiftUI có state observation. ViewModel nhận user action, gọi UseCase/Repository, trả UI state; View render. Business rule phức tạp nằm ở Domain, không dồn toàn bộ vào ViewModel. ViewModel không phụ thuộc UIViewController/UIView.

```text
View action -> ViewModel -> UseCase/Repository -> Result
View <- UI state (idle/loading/content/empty/error) <- ViewModel
```

### Ví dụ ViewModel dùng Swift Concurrency

```swift
import Foundation

struct CatalogItem: Identifiable, Sendable {
    let id: Int
    let title: String
}

protocol CatalogRepository: Sendable {
    func load() async throws -> [CatalogItem]
}

@MainActor
final class CatalogViewModel {
    enum State {
        case idle, loading, content([CatalogItem]), empty, error(String)
    }

    private let repository: any CatalogRepository
    private(set) var state: State = .idle {
        didSet { onState?(state) }
    }
    var onState: ((State) -> Void)?
    private var generation = 0

    init(repository: any CatalogRepository) {
        self.repository = repository
    }

    func load() async {
        generation += 1
        let id = generation
        state = .loading
        do {
            let items = try await repository.load()
            try Task.checkCancellation()
            guard id == generation else { return }
            state = items.isEmpty ? .empty : .content(items)
        } catch {
            guard id == generation else { return }
            state = Task.isCancelled ? .idle : .error("Loading failed")
        }
    }
}
```

UIKit VC giữ VM, bind onState = { [weak self] state in self?.render(state) }, gọi load qua Task, update table snapshot/loading/error trong render. Callback đơn giản chỉ một subscriber; nếu nhiều observers dùng Combine/RxSwift hoặc observation. SwiftUI có thể dùng ObservableObject/@Published hoặc @Observable, không cần callback manual. Không giữ closure capture VC mạnh trong VM.

### MVC/MVP/MVVM/Clean và state ownership

MVC controller xử lý event/render và điều phối; MVP presenter gọi passive view protocol; MVVM exposes state để view bind; Clean tách domain dependency. Với feature nhỏ, MVC hoặc MVVM nhẹ đủ; khi có nghiệp vụ, offline cache, nhiều data source, test, thêm repository/use case. Coordinator/router quản lý navigation, ViewModel phát route intent, không trực tiếp push VC. Không dùng Singleton cho mọi service.

### UI state vs event

State có latest value (loading/content/error), event một lần (navigate/toast). Không replay navigation khi view subscribe lại. Rx Driver/Signal, Combine subject hoặc event handler tùy architecture. Pagination giữ cursor/isLoading/hasMore và generation; refresh giữ content nếu UX cần. Error state phân biệt empty/no connection/permission.

**Q: Massive ViewModel giải quyết thế nào?** Tách use cases, validation, mapper, pagination/session state, coordinator; không tạo layer chỉ forward.

**Q: ViewModel nên import UIKit?** Ưu tiên không phụ thuộc UIKit nếu muốn reuse/test; UI-specific presentation model có thể giữ types hợp lý, không cần dogmatic.

**Q: MVVM + Clean có trùng nhau?** Không, MVVM là presentation pattern; Clean là boundary/dependency architecture.

**Bài tập:** Implement CatalogViewModel, bind cùng state vào UIKit và SwiftUI (thay observation adapter), fake success/empty/error/cancel. Xem [DI](../dependency-injection.md).
