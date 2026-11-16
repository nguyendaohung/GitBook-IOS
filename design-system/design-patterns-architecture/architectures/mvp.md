# MVP

**Định Nghĩa:**&#x20;

Đối với những người vừa tiếp cận lập trình iOS, chúng ta thường sẽ xây dựng dự án dưới dạng mô hình MVC (Model View Controller). Tuy nhiên khi dự án đã lớn dần, nhiều tính năng hơn, sẽ có lúc bạn thấy được Controller của chúng ta có thể lên đến 1000 dòng code. Nên người ta hay gọi vui MVC là Massive View Controller. Đó là vì Controller (trong MVC) nhận quá nhiều trách nhiệm: xử lý business logic, view logic, navigator, ... Vậy có cách nào tránh được trường hợp này hay không?

\-------------------------------------------------------------------------

Tất nhiên rồi, đó là lý do chúng ta sẽ tìm hiểu một mô hình mới giải quyết vấn đề trong mô hình MVC. Đó là mô hình MV

#### Thành phần chính của MVP <a href="#id-31-thanh-phan-chinh-cua-mvp-3" id="id-31-thanh-phan-chinh-cua-mvp-3"></a>

![](https://images.viblo.asia/317b4cdb-2da9-441c-9ff8-5ee2837d2a53.png)

Trên đây là sơ đồ luồng hoạt động của các thành phần chính trong MVP. MVP bao gồm 3 thành phần chính: Model, View và Presenter. Mình sẽ giới thiệu từng thành phần nhé:

* `Model`: Là nơi giữ các struct/class mà ứng dụng chúng ta sử dụng. Thường thì mình sẽ tách **Model** ra khỏi mô hình để sử dụng chung cho toàn bộ project (tham khảo ở example code ở phần tiếp theo nhé)
* `View`: bao gồm UIViewController và storyboard (hoặc xibs nếu bạn không dùng storyboard). Bây giờ ở **View** chỉ cần quan tâm đến view logic: setup UI, nhận các event/action từ user,.. Sau đó **View** sẽ truyền các event/action đó sang cho **Presenter** xử lý. Tất nhiên nó cũng phải chứa một reference đến **Presenter** nhé
*   `Presenter`:

    * Đây chính là trái tim của mô hình MVP. Là nơi chúng ta sẽ nhận các event/action từ **View**, là nơi chúng ta thực hiện restful API (gọi API đến server để lấy dữ liệu), validate input, ... Nhìn chung mọi business logic chúng ta đều xử lý ở **Presenter**
    * Sau khi có data (data đã được xử lý), **Presenter** sẽ truyền data đó về lại cho **View** và nhờ **View** hiển thị ra màn hình
    * **Presenter** cần chứa 1 liên kết yếu đến **View** để tránh tình trạng retain cycle nhé

    **Một lưu ý quan trọng**: là **Presenter** không nên import UIKit nhé (vì có thể gây ra code smell)

\=> Note: Các thành phần này sẽ giao tiếp với nhau thông qua Protocol nhé mọi người. Điều này khá là quan trọng trong mô hình này và cũng là ưu điểm so với MVC.&#x20;

\=> Qua đây chúng ta đã hiểu sơ được các thành phần chính và công dụng của từng thành phần trong mô hình MVP.

#### Ưu điểm <a href="#id-51-uu-diem-7" id="id-51-uu-diem-7"></a>

Vì Presenter đã tách code business logic ra khỏi View nên sẽ giúp chúng ta:

* Dễ dàng đọc hiểu code.
* Dễ dàng viết Unit test cho logic.
* Dễ maintain, bảo trì.

#### &#x20;Nhược điểm <a href="#id-52-nhuoc-diem-8" id="id-52-nhuoc-diem-8"></a>

* Số lượng code và file code tăng lên nhiều so với MVC
* Vì mô hình chỉ xử lý việc tách business logic ra khỏi View, nên việc call API, lưu data xuống local,... đều được xử lý ở Presenter. Sẽ gây ra hiện tượng Massive Presenter (Presenter nhận nhiều trách nhiệm nên bị phình to ra)

### &#x20;Kết luận 📔 <a href="#id-6-ket-luan--9" id="id-6-ket-luan--9"></a>

* MVP bao gồm 3 phần: Model, View và Presenter
* View và Presenter phụ thuộc contract của nhau qua protocol, không cần phụ thuộc concrete implementation
* Presenter là nơi xử lý business logic và truyền lại cho View để hiển thị data

## MVP thực tế: passive view và Presenter

```swift
protocol ProfileView: AnyObject {
    func showLoading(_ value: Bool)
    func showName(_ name: String)
    func showError(_ message: String)
}

protocol ProfileService {
    func loadName() async throws -> String
}

@MainActor
final class ProfilePresenter {
    weak var view: (any ProfileView)?
    private let service: any ProfileService

    init(service: any ProfileService) { self.service = service }

    func load() async {
        view?.showLoading(true)
        defer { view?.showLoading(false) }
        do {
            let name = try await service.loadName()
            try Task.checkCancellation()
            view?.showName(name)
        } catch is CancellationError {
            return
        } catch {
            view?.showError("Could not load")
        }
    }
}
```

VC conform ProfileView, giữ presenter, gọi load từ lifecycle/action; presenter weak view, service inject. Presenter biết View protocol, View biết Presenter contract, không biết concrete implementation tùy thiết kế. Business rules phức tạp tách UseCase/domain; presenter điều phối presentation. Production cần generation/loading ownership khi load overlap.

**Q: MVP vs MVVM?** MVP gọi view methods; MVVM expose state/binding để view observe.

**Q: Vì sao weak View?** VC giữ presenter, presenter strong VC tạo retain cycle.

**Q: Test Presenter như thế nào?** Mock View ghi calls, fake service trả success/error, assert loading và render.

**Q: Massive Presenter xử lý?** Tách use case/repository/mapper/coordinator, presenter giữ presentation logic.
