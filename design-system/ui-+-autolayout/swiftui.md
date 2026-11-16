# SwiftUI

## 1. Định nghĩa

SwiftUI là framework declarative UI của Apple (iOS 13+). Thay vì trực tiếp thay đổi từng UIView, ta mô tả UI tương ứng với state hiện tại. Khi dependency thay đổi, SwiftUI tính lại body cần thiết, đối chiếu identity và cập nhật UI.

UIKit là imperative UI: tạo UIView/UIViewController, cấu hình constraint, quản lý lifecycle, nhận event rồi gọi code thay đổi view (set text, reloadData...). UIKit vẫn là framework native hiện đại; SwiftUI không thay thế mọi API UIKit.

```text
UIKit:   Event → Controller → mutate model → update từng view
SwiftUI: Event → update state → body(state) → framework update UI
```

| Tiêu chí | UIKit | SwiftUI |
| --- | --- | --- |
| UI | UIView/UIViewController (reference type) | View thường là struct, mô tả UI |
| Update | Gọi update/reload/snapshot, binding tự thiết kế | State thay đổi kích hoạt cập nhật dependency |
| Layout | Auto Layout, frame, UIStackView | HStack/VStack/ZStack, modifier, size proposal |
| Lifecycle | viewDidLoad, viewWillAppear, viewDidDisappear... | Identity, onAppear/onDisappear, task |
| List | UITableView/UICollectionView, reusable cells | List, ForEach, LazyVStack/LazyVGrid |
| Navigation | UINavigationController, present/dismiss | NavigationStack, path, sheet/alert theo state |
| Data binding | Delegate, closure, Combine, RxSwift | @State, @Binding, ObservableObject/Observation |
| Custom UI | UIView subclass, Core Animation, drawing | View composition, Shape, Canvas, representable |
| OS | Hỗ trợ app có deployment target cũ | API mới khác nhau theo iOS version |
| Testing | XCTest/UI test, VC/service test | Test state/model, preview, XCUITest |

### Khi nào sử dụng?

* App UIKit lớn, UI tùy chỉnh phức tạp, SDK phụ thuộc UIKit hoặc hỗ trợ OS cũ: giữ UIKit, migrate từng màn hình.
* Feature mới có UI theo state (settings, paywall, form, gallery): SwiftUI giảm boilerplate, dễ composition.
* Không kết luận framework nào luôn nhanh hơn. Profile launch, scrolling, memory, layout trên device thật. SwiftUI có thể tốn chi phí nếu identity không ổn định hoặc body làm việc nặng; UIKit có thể lag khi main thread xử lý layout/image decode.

## 2. Cùng một màn hình Counter bằng hai framework

### UIKit

```swift
import UIKit

final class CounterViewController: UIViewController {
    private var count = 0
    private let countLabel = UILabel()
    private let incrementButton = UIButton(type: .system)

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground
        incrementButton.setTitle("Increment", for: .normal)
        incrementButton.addTarget(self, action: #selector(increment),
                                  for: .touchUpInside)

        let stack = UIStackView(arrangedSubviews: [countLabel, incrementButton])
        stack.axis = .vertical
        stack.alignment = .center
        stack.spacing = 16
        stack.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(stack)
        NSLayoutConstraint.activate([
            stack.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            stack.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])
        render()
    }

    @objc private func increment() {
        count += 1
        render()
    }

    private func render() {
        countLabel.text = "Count: \(count)"
    }
}
```

Controller sở hữu count và label. Nếu đổi count mà quên gọi render(), UI không tự đổi. UIKit có thể dùng binding/Combine/RxSwift để tự động hóa render; imperative không nghĩa là không có data binding.

### SwiftUI

```swift
import SwiftUI

struct CounterView: View {
    @State private var count = 0

    var body: some View {
        VStack(spacing: 16) {
            Text("Count: \(count)")
            Button("Increment") {
                count += 1
            }
        }
    }
}
```

@State là storage do SwiftUI quản lý theo identity của CounterView. Khi count đổi, body được đánh giá lại; không cần giữ UILabel hay gọi render(). View struct có thể được tạo lại nhiều lần mà state vẫn còn nếu identity không đổi.

## 3. State, Binding và data flow

| Wrapper | Ownership | Sử dụng |
| --- | --- | --- |
| @State | View sở hữu storage | Giá trị local, ví dụ showSheet, selectedTab; iOS 17 có thể giữ @Observable reference |
| @Binding | Không sở hữu, đọc/ghi storage của owner | Child chỉnh state của parent |
| @StateObject | View sở hữu ObservableObject | Model tạo một lần theo identity, iOS 14+ |
| @ObservedObject | Không sở hữu lifetime | Model được parent/DI truyền vào |
| @EnvironmentObject | Không sở hữu | ObservableObject được inject vào environment |
| @Environment | Đọc environment value | dismiss, colorScheme, locale, dependency |
| @Observable / @Bindable | Observation iOS 17+ | Theo dõi property được đọc; tạo binding tới model |

```swift
import SwiftUI

struct SettingsView: View {
    @State private var notificationsEnabled = false

    var body: some View {
        VStack {
            NotificationToggle(isEnabled: $notificationsEnabled)
            Text(notificationsEnabled ? "Enabled" : "Disabled")
        }
    }
}

struct NotificationToggle: View {
    @Binding var isEnabled: Bool

    var body: some View {
        Toggle("Notifications", isOn: $isEnabled)
    }
}
```

$notificationsEnabled là projected value kiểu Binding<Bool>. Child không copy giá trị sang @State vì sẽ tạo hai nguồn sự thật khác nhau. Binding không tự lưu vào server/UserDefaults; persistence là trách nhiệm model/service.

### ObservableObject (iOS 14+) và Observation (iOS 17+)

```swift
import SwiftUI

@MainActor
final class LegacyProfileModel: ObservableObject {
    @Published var name = ""
    @Published private(set) var isSaving = false

    func save() async {
        guard !isSaving else { return }
        isSaving = true
        defer { isSaving = false }
        // Gọi repository thực tế tại đây.
    }
}

struct LegacyProfileView: View {
    @StateObject private var model = LegacyProfileModel()

    var body: some View {
        Form {
            TextField("Name", text: $model.name)
            Button("Save") {
                Task { await model.save() }
            }
            .disabled(model.isSaving)
        }
    }
}
```

Nếu parent đã sở hữu model, child dùng @ObservedObject var model: LegacyProfileModel. Không tạo @ObservedObject inline rồi kỳ vọng lifetime ổn định.

```swift
import SwiftUI
import Observation

@Observable
@MainActor
final class ProfileModel {
    var name = ""
    var email = ""
}

struct ModernProfileView: View {
    @State private var model = ProfileModel()

    var body: some View {
        ProfileForm(model: model)
    }
}

struct ProfileForm: View {
    @Bindable var model: ProfileModel

    var body: some View {
        Form {
            TextField("Name", text: $model.name)
            TextField("Email", text: $model.email)
        }
    }
}
```

Observation theo dõi property được đọc trong body; không cần @Published. @Bindable tạo binding, không tạo ownership. Cả hai mô hình cần main actor cho UI state; Observation không tự đảm bảo thread safety.

## 4. Lifecycle và identity

UIKit: loadView/viewDidLoad thường một lần mỗi lần view được load; viewWillAppear/viewDidAppear có thể lặp; viewWillDisappear/viewDidDisappear theo transition. viewDidLayoutSubviews có thể chạy nhiều lần.

SwiftUI: init/body có thể chạy nhiều lần. onAppear không tương đương viewDidLoad một lần, vì list row, navigation, conditional branch có thể xuất hiện lại. State sống theo structural identity hoặc explicit ID.

```swift
struct DetailView: View {
    let productID: String
    @State private var title = ""

    var body: some View {
        Text(title)
            .task(id: productID) {
                // Minh họa: thay bằng repository.fetch(productID).
                await Task.yield()
                guard !Task.isCancelled else { return }
                title = "Product " + productID
            }
    }
}
```

.task(id:) hủy task cũ khi id đổi/view biến mất (cooperative cancellation). Không gọi API trong body. .id(productID) thay identity và reset state của subtree; chỉ dùng khi muốn reset, không dùng UUID() mỗi render.

**Lỗi thường gặp:** ForEach(items.indices, id: \.self) với insert/delete/reorder làm state/cell identity sai. Dùng ID nghiệp vụ ổn định.

## 5. Layout: Auto Layout vs SwiftUI

UIKit giải hệ constraint theo priority/intrinsic size. SwiftUI dùng size proposal: parent đề xuất kích thước, child chọn size, parent đặt child. Modifier tạo view mới bao quanh view cũ, thứ tự quan trọng.

```swift
Text("Hello")
    .padding(16)
    .background(Color.blue) // background bao cả padding

Text("Hello")
    .background(Color.blue)
    .padding(16) // padding nằm ngoài background
```

* VStack/HStack tương tự stack view nhưng không đồng nhất constraint. Spacer() chiếm không gian linh hoạt; layoutPriority hỗ trợ phân bổ size, không phải Auto Layout priority 1000.
* .frame(maxWidth: .infinity, alignment: .leading) nhận width đề xuất; không đặt width cứng theo UIScreen để hỗ trợ split view.
* GeometryReader nhận proposal và thường chiếm không gian có sẵn; dùng khi thực sự cần geometry, không lồng tùy ý.
* safeAreaInset, safe area, ScrollView, Dynamic Type; fixedSize chỉ khi cần intrinsic size, tránh text overflow.
* LazyVStack/LazyVGrid tạo content khi cần; List có behavior row/selection/swipe, không mặc định giống UITableView cell lifecycle.

## 6. MVVM + REST API thực tế: danh sách sản phẩm

Ví dụ chạy từ iOS 15+, endpoint minh họa; truyền URL backend thật khi tích hợp. Repository độc lập UI, ViewModel chịu trách nhiệm state, View chỉ render.

```swift
import Foundation

struct Product: Decodable, Identifiable, Hashable, Sendable {
    let id: Int
    let title: String
}

protocol ProductRepository: Sendable {
    func fetchProducts() async throws -> [Product]
}

struct RemoteProductRepository: ProductRepository {
    let url: URL

    func fetchProducts() async throws -> [Product] {
        let (data, response) = try await URLSession.shared.data(from: url)
        guard let response = response as? HTTPURLResponse,
              (200..<300).contains(response.statusCode) else {
            throw URLError(.badServerResponse)
        }
        return try JSONDecoder().decode([Product].self, from: data)
    }
}
```

```swift
import SwiftUI

@MainActor
final class ProductsModel: ObservableObject {
    enum State {
        case idle, loading, loaded([Product]), failed(String)
    }

    @Published private(set) var state: State = .idle
    private let repository: any ProductRepository
    private var generation = 0

    init(repository: any ProductRepository) {
        self.repository = repository
    }

    func load() async {
        generation += 1
        let requestID = generation
        state = .loading
        do {
            let products = try await repository.fetchProducts()
            try Task.checkCancellation()
            guard requestID == generation else { return }
            state = .loaded(products)
        } catch is CancellationError {
            guard requestID == generation else { return }
            state = .idle
        } catch {
            guard requestID == generation else { return }
            state = Task.isCancelled ? .idle : .failed("Could not load products")
        }
    }
}

struct ProductsView: View {
    @StateObject private var model: ProductsModel

    init(repository: any ProductRepository) {
        _model = StateObject(wrappedValue: ProductsModel(repository: repository))
    }

    var body: some View {
        Group {
            switch model.state {
            case .idle, .loading:
                ProgressView()
            case .loaded(let products):
                if products.isEmpty {
                    Text("No products")
                } else {
                    List(products) { product in
                        Text(product.title)
                    }
                }
            case .failed(let message):
                VStack {
                    Text(message)
                    Button("Retry") { Task { await model.load() } }
                }
            }
        }
        .task { await model.load() }
        .refreshable { await model.load() }
    }
}
```

Generation chống stale response khi refresh và retry overlap. Cancellation không đảm bảo server dừng xử lý, chỉ không dùng kết quả cũ. Production có thể giữ content khi refresh, thêm pagination và offline cache. Unit test inject fake repository trả success/failure, không cần render UI.

## 7. Navigation

UIKit pushViewController/popViewController, present/dismiss; Coordinator giữ navigation. SwiftUI NavigationStack (iOS 16+) quản lý stack dựa trên value/path.

```swift
import SwiftUI

struct CatalogView: View {
    @State private var path: [Int] = []

    var body: some View {
        NavigationStack(path: $path) {
            List([1, 2, 3], id: \.self) { id in
                NavigationLink("Product \(id)", value: id)
            }
            .navigationTitle("Catalog")
            .navigationDestination(for: Int.self) { id in
                Text("Detail \(id)")
            }
        }
    }
}
```

path.append(id) push bằng code; path.removeAll() pop root. Với app lớn dùng enum Route: Hashable, Coordinator/router quản lý deep link và auth guard. sheet(item:) cho optional identifiable item, tránh hai boolean bất đồng bộ. NavigationView cho OS cũ; kiểm tra availability.

## 8. UIKit ↔ SwiftUI

### Nhúng SwiftUI vào UIKit (UIHostingController)

```swift
import UIKit
import SwiftUI

final class LegacyCatalogController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        let host = UIHostingController(rootView: CounterView())
        addChild(host)
        host.view.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(host.view)
        NSLayoutConstraint.activate([
            host.view.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            host.view.bottomAnchor.constraint(equalTo: view.safeAreaLayoutGuide.bottomAnchor),
            host.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            host.view.trailingAnchor.constraint(equalTo: view.trailingAnchor)
        ])
        host.didMove(toParent: self)
    }
}
```

Có thể push/present hosting controller thay vì containment. Khi remove child: willMove(toParent:nil), remove view, removeFromParent. Dùng chung ViewModel/service thay vì copy state qua hai UI.

### Nhúng UIKit vào SwiftUI (UIViewRepresentable)

```swift
import SwiftUI
import UIKit

struct LegacyTextField: UIViewRepresentable {
    @Binding var text: String

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeUIView(context: Context) -> UITextField {
        let field = UITextField()
        field.borderStyle = .roundedRect
        field.addTarget(context.coordinator,
                        action: #selector(Coordinator.changed(_:)),
                        for: .editingChanged)
        return field
    }

    func updateUIView(_ uiView: UITextField, context: Context) {
        context.coordinator.parent = self
        if uiView.text != text { uiView.text = text }
    }

    final class Coordinator: NSObject {
        var parent: LegacyTextField

        init(_ parent: LegacyTextField) { self.parent = parent }

        @objc func changed(_ field: UITextField) {
            parent.text = field.text ?? ""
        }
    }
}
```

makeUIView tạo view, updateUIView đồng bộ khi state đổi; Coordinator chuyển delegate/event về Binding. Không set state vô điều kiện trong updateUIView, dễ vòng lặp. UIViewControllerRepresentable cho SDK có UIViewController. dismantleUIView để cleanup observer/resource.

## 9. Performance, animation, accessibility và lỗi thường gặp

* body phải nhẹ, không decode ảnh, query DB hoặc tạo network request. Dùng async service, downsample, cache, lazy container.
* Stable ID và state scope nhỏ; không đưa mọi app state vào một object lớn. Profile bằng Instruments, đừng dùng EquatableView/memoization theo thói quen.
* withAnimation thay đổi state trong transaction; .animation(_:value:) theo value cụ thể. Không animate mọi update lớn.
* accessibilityLabel, accessibilityValue, Dynamic Type, VoiceOver, contrast, Reduce Motion. UIKit accessibility tương tự nhưng gắn lên UIView.
* Preview dùng fake data/repository, không gọi production API. XCUITest chọn accessibilityIdentifier.
* @EnvironmentObject thiếu injection gây crash; @StateObject có thể giữ model cũ khi parameter đổi; dùng identity hoặc phương án ownership phù hợp.
* List nested scroll/GeometryReader dễ gây layout bất ngờ; thử size class/iPad, landscape, keyboard.

## 10. Q&A phỏng vấn

**Q: SwiftUI là MVC hay MVVM?** Framework UI không bắt buộc architecture. View state local phù hợp feature đơn giản, ViewModel tách nghiệp vụ/API cho feature phức tạp. Domain/data độc lập UI.

**Q: body chạy lại có nghĩa render toàn bộ màn hình?** Không. Body mô tả mới, SwiftUI reconcile theo dependency/identity. Không dựa vào số lần body chạy như lifecycle contract.

**Q: StateObject vs ObservedObject?** Ownership/lifetime. Parent tạo một model, child observe; nếu child tự sở hữu thì StateObject.

**Q: Khi nào @State vs @Binding?** State là source of truth local; Binding đọc/ghi source của owner.

**Q: @Observable khác ObservableObject?** Observation theo property access, iOS 17; ObservableObject phát objectWillChange/@Published, hỗ trợ OS cũ hơn.

**Q: onAppear vs task?** onAppear callback đồng bộ có thể lặp; task hỗ trợ async và cancellation theo view lifetime.

**Q: Có nên migrate cả app UIKit sang SwiftUI?** Không mặc định. Chọn theo deployment target, SDK, chi phí test, complexity. Migrate screen-by-screen qua UIHostingController, chia sẻ domain/repository.

**Q: SwiftUI list hiện sai state sau reorder?** Kiểm tra stable ID, ownership state và ForEach identity.

**Q: Làm sao test ViewModel SwiftUI?** Inject fake repository, test state transitions/async cancellation, UI test luồng người dùng; không cần phụ thuộc body.

### Bài tập thực hành

1. Viết Counter UIKit và SwiftUI, thêm reset và disable khi count = 0.
2. Tạo Product list có loading/empty/error/retry, fake repository trả lỗi sau delay.
3. Thêm search debounce/cancellation, kiểm tra response cũ không ghi đè kết quả mới.
4. Nhúng SwiftUI Product detail vào UIKit navigation, giữ cùng repository.
5. Thử đổi ID trong ForEach, quan sát state; sửa bằng stable ID.

### Tham khảo

* [SwiftUI - Apple](https://developer.apple.com/documentation/swiftui)
* [UIKit - Apple](https://developer.apple.com/documentation/uikit)
* [Observation - Apple](https://developer.apple.com/documentation/observation)
* [SwiftUI - FX Studio](https://fxstudio.dev/category/code/swiftui/page/6/)
