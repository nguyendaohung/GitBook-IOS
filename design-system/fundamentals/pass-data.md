# Pass Data

Thường dùng:

* Delegate: Sử dụng khi trong cần truyền nhiều dữ liệu/nhiều action ⇒ giúp code clean
* Closure: Sử dụng khi cần truyền ít dữ liệu/action ⇒ sử dụng nhanh chóng, dễ dàng
* Notification: Sử dụng khi 2 màn hình không liên quan gì nhau hoặc cần truyền dữ liệu sang nhiều nơi trong ứng dụng

Ngoài ra còn có:&#x20;

1. Sử dụng Segue: Sử dụng segue để chuyển từ một màn hình đến một màn hình khác, bạn có thể truyền dữ liệu từ màn hình nguồn đến màn hình đích bằng cách sử dụng thuộc tính `prepare(for:sender:)`.

## Truyền dữ liệu giữa màn hình: cách chọn

| Cách | Khi dùng | Lưu ý |
| --- | --- | --- |
| Constructor injection | Push detail với ID/model | Dependency bắt buộc, testable |
| Delegate | Child trả action về owner | Protocol AnyObject, weak delegate |
| Closure | Một vài callbacks | Capture/lifetime |
| NotificationCenter | Broadcast nhiều consumer | Event schema, observer cleanup |
| Shared ViewModel/store | State giữa màn hình có cùng owner | Không dùng global singleton tùy ý |
| Coordinator/Router | Navigation/deep link | Route typed, auth guard |

```swift
import UIKit

protocol DetailDelegate: AnyObject {
    func didSelect(id: Int)
}

final class DetailController: UIViewController {
    let productID: Int
    weak var delegate: (any DetailDelegate)?

    init(productID: Int) {
        self.productID = productID
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) { nil }

    func confirm() { delegate?.didSelect(id: productID) }
}
```

Forward data qua initializer; backward event qua delegate/closure. SwiftUI dùng value props, Binding để chỉnh parent state, ObservableObject/Observation cho shared state, NavigationStack value route. Không truyền password/token qua notification/deep link; dùng session owner.

**Q: Delegate vs closure?** Delegate rõ nhiều methods/contract, weak ownership; closure nhẹ cho callback đơn lẻ.

**Q: NotificationCenter vs delegate?** Broadcast loose coupling vs one-to-one; không dùng broadcast thay state source.

**Q: Truyền dữ liệu khi pop về màn trước?** Delegate/closure/shared store, update state rồi pop; tránh tìm VC trong navigation stack tùy tiện.
