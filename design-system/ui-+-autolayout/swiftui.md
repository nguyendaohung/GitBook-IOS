# SwiftUI

### Định nghĩa

SwiftUI xây UI theo declarative: View là value mô tả giao diện từ state; body được tính lại khi dependency thay đổi. View value ngắn hạn, identity của view quyết định lifetime state. Không thực hiện API/network side effect trực tiếp trong body.

| Property wrapper | Khi sử dụng |
| --- | --- |
| @State | State cục bộ do view sở hữu |
| @Binding | Tham chiếu đọc/ghi state từ owner |
| @StateObject | View sở hữu ObservableObject, lifetime theo identity (iOS 14+) |
| @ObservedObject | Nhận ObservableObject từ bên ngoài |
| @EnvironmentObject | Object chia sẻ qua environment, phải inject |
| @Observable / @Bindable | Observation iOS 17+, theo dõi property được đọc, binding khi cần |

```swift
import SwiftUI

@MainActor
final class FavoritesModel: ObservableObject {
    @Published private(set) var items = ["Photo 1", "Photo 2"]

    func remove(at offsets: IndexSet) {
        items.remove(atOffsets: offsets)
    }
}

struct FavoritesView: View {
    @StateObject private var model = FavoritesModel()
    @State private var showHelp = false

    var body: some View {
        List {
            ForEach(model.items, id: \.self) { item in
                Text(item)
            }
            .onDelete(perform: model.remove)
        }
        .toolbar {
            Button("Help") { showHelp = true }
        }
        .sheet(isPresented: $showHelp) {
            Text("Manage favorites")
        }
    }
}
```

### Cơ chế hoạt động

* ForEach cần ID ổn định, không dùng index cho list có insert/delete; dùng domain ID thay vì tên có thể trùng.
* .task(id:) chạy async theo identity/id, tự yêu cầu cancellation khi view biến mất/id đổi; vẫn check cancellation trước update state.
* NavigationStack + path (iOS 16+) quản lý navigation state; sheet/alert theo state.
* UIKit ↔ SwiftUI: UIHostingController nhúng SwiftUI trong UIKit; UIViewRepresentable/UIViewControllerRepresentable tạo và update native view, Coordinator nối delegate; tránh vòng lặp update.
* Layout: parent đề xuất size, child chọn size, parent đặt vị trí; LazyVGrid/LazyVStack tránh tạo toàn bộ nội dung ngay. Dùng Dynamic Type, accessibility labels, localization.

### Q&A

**Q: StateObject khác ObservedObject?** StateObject quản lý lifetime instance theo identity; ObservedObject quan sát instance được truyền từ owner.

**Q: body chạy lại có tạo lại toàn bộ UI?** Không nhất thiết; SwiftUI đối chiếu identity/dependency để update phần cần thiết.

**Q: Vì sao .task chạy lại?** Identity hoặc id thay đổi; thiết kế operation idempotent và cancellation-aware.

**Liên hệ:** Call ID dùng UIKit + SwiftUI, có thể nhúng màn hình mới bằng UIHostingController.

### Tham khảo

[SwiftUI - Apple](https://developer.apple.com/documentation/swiftui)

[SwiftUI - FX Studio](https://fxstudio.dev/category/code/swiftui/page/6/)
