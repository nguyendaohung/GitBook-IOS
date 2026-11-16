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
