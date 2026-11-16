# VIP (Clean Swift)

### Định nghĩa

VIP = ViewController, Interactor, Presenter. Flow unidirectional: ViewController gửi Request → Interactor xử lý business → Presenter format Response thành ViewModel → ViewController display. Router điều hướng/data passing; Worker hỗ trợ service/API/DB. Clean Swift áp dụng boundary theo scene, không phải tất cả Clean Architecture đều VIP.

```text
ViewController -> Interactor -> Presenter -> ViewController
      |                |
    Router           Worker/UseCase
```

### Ví dụ scene

```swift
enum ProfileScene {
    struct Request { let name: String }
    struct Response { let normalizedName: String }
    struct ViewModel { let title: String }
}

protocol ProfileDisplay: AnyObject {
    func display(_ viewModel: ProfileScene.ViewModel)
}

final class ProfilePresenter {
    weak var view: (any ProfileDisplay)?

    func present(_ response: ProfileScene.Response) {
        view?.display(.init(title: "Hello, " + response.normalizedName))
    }
}

final class ProfileInteractor {
    let presenter: ProfilePresenter

    init(presenter: ProfilePresenter) { self.presenter = presenter }

    func load(_ request: ProfileScene.Request) {
        let name = request.name.trimmingCharacters(in: .whitespacesAndNewlines)
        presenter.present(.init(normalizedName: name))
    }
}
```

Ví dụ cần Foundation; VC conform ProfileDisplay, inject interactor/presenter, gọi load Request. Presenter weak view tránh cycle. Với async API, Interactor gọi Worker/UseCase, Presenter format loading/error/content, UI update main actor. Router quản lý navigation. Không đưa logic UI vào Interactor.

### Ưu/nhược điểm

Ưu: separation rõ, test Interactor/Presenter, flow predictable; phù hợp module lớn. Nhược: nhiều file/protocol/model Request/Response/ViewModel, boilerplate cho màn hình nhỏ. MVVM thường nhẹ hơn, ViewModel trực tiếp expose state. VIP Presenter không phải MVP Presenter hoàn toàn; Interactor giữ business.

**Q: VIP vs VIPER?** VIP tập trung unidirectional scene cycle; VIPER có View/Interactor/Presenter/Entity/Router, Presenter điều phối View và Interactor theo module.

**Bài tập:** Scene load profile success/error, test Interactor bằng fake worker và Presenter format.

## Câu hỏi phỏng vấn tình huống

**Q: Request, Response, ViewModel khác nhau?** Request input của scene, Response kết quả business, ViewModel format display; boundary giúp test và tránh UIKit dependency.

**Q: Worker và Interactor khác gì?** Worker thao tác API/DB/service; Interactor điều phối use case/business, không format UI.

**Q: VIP có tránh retain cycle tự động?** Không, Presenter weak view, Router/VC ownership rõ, cancel async task và observer.
