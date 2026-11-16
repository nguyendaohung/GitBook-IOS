# MVC

### Định nghĩa

&#x20;Là 1 mẫu kiến trúc Model-View-Controller, trong đó:

* Model: Là nơi chứa những đối tượng, dữ liệu mình tự định nghĩa, đóng gói các đối tượng và logic của dữ liệu đó.
* View: Là đối tượng mà user có thể nhìn thấy, có thể thao tác lên đó.
* Controller: Là đối tượng nằm giữa Model và View. Nhận trách nhiệm quản lý điều khiển mọi logic giữa View và Model. Ví dụ như là nhận action từ View rồi xử lý dữ liệu ở Model

### Ưu nhược điểm

* Có 2 luồn chính trong MVC:
  * View ⇒ Controller ⇒ Model: Khi user thao tác trên View dẫn đến data thay đổi ở Model.
  * Model ⇒ Controller ⇒ View: Khi data thay đổi ở Model và View được thông báo để thay đổi theo cho phù hợp.

#### Ưu điểm:

* Là kiến trúc phổ biến và dễ sử dụng nhất trong lập trình iOS
* Nó giúp mình thiết kế hệ thống 1 cách rõ ràng hơn. Giúp mình định nghĩa luồng dữ liệu 1 cách rõ ràng

#### Nhược điểm

* Controller nhận quá nhiều trách nhiệm, dẫn đến phình to controller. Hầu như mọi thứ trong app đều được xử lý ở controller ⇒ Cần tách responsibility; MVC không giới hạn dự án nhỏ.
* Khó kiểm thử. Vì View và Controller trong iOS dính quá chặt với nhau. Nguyên nhân: Controller (UIViewController) quản lý life cycle của View.

## MVC thực tế

```swift
import UIKit

struct Item { let title: String }
final class ItemsController: UITableViewController {
    private var items = [Item(title: "Coffee")]

    override func tableView(_ tableView: UITableView,
                            numberOfRowsInSection section: Int) -> Int {
        items.count
    }

    override func tableView(_ tableView: UITableView,
                            cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = UITableViewCell(style: .default, reuseIdentifier: nil)
        cell.textLabel?.text = items[indexPath.row].title
        return cell
    }
}
```

Ví dụ tối giản để thấy controller kết nối Model và View; production register/dequeue cell, repository fetch, state loading/error, cancellation. MVC không buộc controller chứa network/business logic; tách service/domain, child VC, data source, coordinator để tránh massive VC. Dùng MVC cho feature đơn giản hoặc app lớn có boundary rõ, không giới hạn project nhỏ.

**Q: Massive ViewController là gì?** VC làm quá nhiều network/business/layout/navigation; tách responsibilities và dependency.

**Q: MVC có test được?** Domain/service thuần test dễ, controller phụ thuộc UI test phức tạp hơn; inject dependencies.

**Q: MVC vs MVP?** MVP presenter điều phối passive view qua protocol, tách presentation logic khỏi VC.

**Q: MVC vs MVVM?** MVVM expose UI state/binding; MVC controller tự điều phối update. Chọn theo complexity/testability.
