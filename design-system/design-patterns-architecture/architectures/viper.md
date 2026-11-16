# VIPER

### Định nghĩa

VIPER = View, Interactor, Presenter, Entity, Router. View hiển thị và forward user actions; Presenter điều phối presentation, gọi Interactor, nhận result và điều hướng qua Router; Interactor xử lý business/use case; Entity là domain data; Router tạo module và navigation.

```text
View <-> Presenter <-> Interactor -> Repository
              |
            Router
Entity đi qua domain/presentation mapping
```

### Ví dụ module

```swift
struct Article {
    let id: Int
    let title: String
}

protocol ArticleInteractor {
    func article(id: Int) -> Article?
}

protocol ArticleView: AnyObject {
    func show(title: String)
    func showError()
}

protocol ArticleRouter: AnyObject {
    func close()
}

final class ArticlePresenter {
    weak var view: (any ArticleView)?
    private let interactor: any ArticleInteractor
    private let router: any ArticleRouter

    init(interactor: any ArticleInteractor, router: any ArticleRouter) {
        self.interactor = interactor
        self.router = router
    }

    func didLoad(id: Int) {
        guard let article = interactor.article(id: id) else {
            view?.showError()
            return
        }
        view?.show(title: article.title)
    }

    func didTapClose() { router.close() }
}
```

Ví dụ synchronous để minh họa responsibility; production Interactor async + loading/error state, Presenter main actor, Router weak VC để tránh Presenter→Router→VC→Presenter cycle. Module builder lắp ghép dependencies. Entity không chứa UIKit. Test Presenter với mock View/Router, fake Interactor; test Interactor riêng.

### Khi dùng VIPER?

Module nhiều nghiệp vụ, team lớn cần boundary rõ, reuse/test; nhược điểm protocol/file nhiều, navigation complexity. MVP nhẹ hơn (Presenter có thể gọi service), MVVM data binding/state thuận tiện cho SwiftUI/UIKit reactive. Clean Architecture là nguyên tắc dependency, VIPER là tổ chức presentation/module, có thể kết hợp nhưng không bắt buộc.

**Q: Presenter vs Interactor?** Presenter format UI state và điều phối action; Interactor business rule/domain.

**Q: Router có business logic?** Không; route/module composition, navigation; auth/business decision ở domain/presenter theo contract.

**Bài tập:** Tạo article module, test success/not found/navigation, thay repository fake.

## Câu hỏi phỏng vấn tình huống

**Q: VIPER có phù hợp mọi screen?** Không, nhiều protocol/module builder/boilerplate; chọn khi boundary/testability mang giá trị, MVVM/MVP nhẹ hơn cho feature nhỏ.

**Q: Test Router và Presenter thế nào?** Mock View/Router ghi calls, fake Interactor trả result; assert render/route. Test business Interactor riêng.

**Q: Router giữ VC mạnh có rủi ro?** Có, VC→Presenter→Router→VC cycle; weak reference khi phù hợp, owner navigation giữ đúng lifetime.
