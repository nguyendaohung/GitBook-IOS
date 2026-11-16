# UIViewController lifecycle

<figure><img src="../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

* Hàm **init** sẽ được gọi đầu tiên. init(coder:) nếu ViewController được tạo từ storyboard và init(nibName:bundle:) nếu được tạo từ nib file (thay vì storyboard).

&#x20;Ví dụ: `let myViewController = MyViewController(nibName: "MyView", bundle: nil)`

* **loadView**(): chỉ override method này khi muốn build lại toàn bộ UI cho ViewController bằng code. Nếu dùng storyboard hoặc nib thì không nên đụng vào hàm này. Hàm này được implement trong lúc UIViewController load interface và connect các outlet/action

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 22.28.54.png" alt=""><figcaption><p>Phương thức này được gọi khi UIViewController cần hiển thị một view. Trong phương thức này, ta có thể tạo view bằng code hoặc load view từ file nib</p></figcaption></figure>

* **viewDidLoad**(): Method chỉ được gọi duy nhất 1 lần trong lifetime của ViewController. Lúc này UIViewController đã được tạo và outlet/action đã sẵn sàng. Thường chúng ta sẽ thao tác với hàm này rất nhiều

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 22.29.25.png" alt=""><figcaption><p>Trong phương thức này, ta có thể cập nhật giao diện, khởi tạo các thuộc tính, hoặc load dữ liệu từ server</p></figcaption></figure>

* **viewWillAppear**(): Method được gọi nhiều lần mỗi khi ViewController xuất hiện lên màn hình. Thông báo cho ViewController rằng view này chuẩn bị được thêm vào view hierarchy

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 22.30.58.png" alt=""><figcaption><p>Trong phương thức này, ta có thể cập nhật dữ liệu hoặc thực hiện các thao tác liên quan đến giao diện trước khi view xuất hiện</p></figcaption></figure>

* **viewDidAppear**(): method được gọi khi ViewController đã hiện thị trên màn hình

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 22.31.39.png" alt=""><figcaption><p>Trong phương thức này, ta có thể thực hiện các thao tác liên quan đến giao diện sau khi view đã xuất hiện</p></figcaption></figure>

* **ViewWillDisappear**: Trong giai đoạn này, view của UIViewController sẽ sắp bị ẩn khỏi màn hình. Phương thức viewWillDisappear() được gọi, cho phép UIViewController thực hiện các công việc chuẩn bị trước khi view bị ẩn.

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 22.32.16.png" alt=""><figcaption><p>Trong phương thức này, ta có thể thực hiện các thao tác liên quan đến giao diện trước khi view biến mất</p></figcaption></figure>

* **ViewDidDisappear**: Trong giai đoạn này, view của UIViewController đã bị ẩn khỏi màn hình. Phương thức viewDidDisappear() được gọi, cho phép UIViewController thực hiện các công việc liên quan đến việc ẩn view trên màn hình.

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 22.32.46.png" alt=""><figcaption><p>Trong phương thức này, ta có thể thực hiện các thao tác liên quan đến giao diện sau khi view đã biến mất</p></figcaption></figure>

* **viewWillLayoutSubviews**: được gọi để thông báo cho ViewController rằng view chuẩn bị layout các subview. Method này được gọi mỗi khi frame có sự thay đổi (ví dụ khi quay hoặc subview cần được layout)
* **viewDidLayoutSubviews**: được gọi để thông báo cho ViewController rằng view vừa layout lại các subview. Ta thường dùng hàm này để thực hiện thêm các thay đổi sau khi layout subview
* **deinit**(): trước khi ViewController bị remove khỏi memory
* **didReceiveMemoryWarning**()

```
Show screen A:

DEBUG: A viewDidLoad
DEBUG: A viewWillAppear
DEBUG: A viewWillLayoutSubviews
DEBUG: A viewDidLayoutSubviews
DEBUG: A viewDidAppear

=======================
Screen A => Screen B:

DEBUG: B viewDidLoad
DEBUG: A viewWillDisappear
DEBUG: B viewWillAppear
DEBUG: B viewWillLayoutSubviews
DEBUG: B viewDidLayoutSubviews
DEBUG: A viewDidDisappear
DEBUG: B viewDidAppear
```

## UIKit thực hành: xây màn hình theo lifecycle

### Khi nào dùng từng lifecycle method?

| Method | Khi sử dụng |
| --- | --- |
| loadView | Tạo root view thủ công nếu không dùng storyboard/XIB |
| viewDidLoad | Setup views, constraints, data source, binding một lần mỗi load |
| viewWillAppear | Refresh dữ liệu cần cập nhật mỗi khi quay lại màn hình |
| viewDidAppear | Bắt đầu tác vụ cần view đã hiển thị, analytics screen event |
| viewDidLayoutSubviews | Điều chỉnh theo bounds/layout thực tế, có thể chạy nhiều lần |
| viewWillDisappear | Chuẩn bị transition, commit editing khi phù hợp |
| viewDidDisappear | Dừng animation/timer/observer theo screen visibility |
| deinit | Kiểm tra giải phóng object và cleanup lifetime còn lại |

Không dùng viewDidLoad để đảm bảo API chỉ gọi một lần toàn app; VC mới tạo sẽ có lifecycle mới. viewWillAppear có thể gọi lại khi dismiss/push/pop. Không đặt network request/constraint creation không guard trong viewDidLayoutSubviews. Khi app foreground/background, dùng Scene/App lifecycle thay vì suy ra chỉ từ VC appearance.

### Ví dụ UIKit programmatic

```swift
import UIKit

final class DetailsViewController: UIViewController {
    private let titleLabel = UILabel()
    private var loadTask: Task<Void, Never>?

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground
        titleLabel.translatesAutoresizingMaskIntoConstraints = false
        titleLabel.numberOfLines = 0
        titleLabel.font = .preferredFont(forTextStyle: .headline)
        titleLabel.adjustsFontForContentSizeCategory = true
        view.addSubview(titleLabel)
        NSLayoutConstraint.activate([
            titleLabel.leadingAnchor.constraint(equalTo: view.layoutMarginsGuide.leadingAnchor),
            titleLabel.trailingAnchor.constraint(equalTo: view.layoutMarginsGuide.trailingAnchor),
            titleLabel.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor, constant: 16)
        ])
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        loadTask?.cancel()
        loadTask = Task { [weak self] in
            // Replace with a repository async API.
            await Task.yield()
            guard !Task.isCancelled else { return }
            self?.titleLabel.text = "Details loaded"
        }
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        loadTask?.cancel()
        loadTask = nil
    }
}
```

UIViewController hoạt động trên main actor trong SDK concurrency annotations; CPU-heavy work cần service/executor phù hợp. Task trên VC có thể giữ self qua await nếu capture mạnh; weak capture trước await và tránh giữ self lâu khi không cần. Cancellation không tự chấm dứt work không hỗ trợ cancel. Nếu data phải load khi screen bị cover/present, lifetime chọn theo nghiệp vụ thay vì luôn cancel viewDidDisappear.

### UIKit và SwiftUI lifecycle

viewDidLoad không có tương đương hoàn toàn trong SwiftUI. View struct init/body có thể chạy nhiều lần, state sống theo identity; .task(id:) async theo identity, onAppear có thể lặp. UIKit quản lý UIViewController instance, SwiftUI quản lý UI declaration và state. Cùng một ViewModel/repository có thể chia sẻ giữa hai framework; xem bài [SwiftUI](../ui-+-autolayout/swiftui.md) để so sánh Counter, layout, navigation, UIHostingController và representable.

**Q: viewDidLayoutSubviews có gọi một lần?** Không, rotation, constraints, safe area, dynamic content đều có thể trigger.

**Q: Khi nào deinit không chạy?** Retain cycle, coordinator/parent giữ VC, observer/timer/subscription còn giữ object; dùng Memory Graph.

**Bài tập:** Push/pop VC, log lifecycle, rotate device; thêm task/cancellation và kiểm tra deinit.
