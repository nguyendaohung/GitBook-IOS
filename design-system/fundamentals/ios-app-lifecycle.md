# iOS app lifecycle

Khi app khởi động, sẽ nhảy vào hàm willFinishLaunchingWithOptions và didFinishLaunchingWithOptions (hàm này chúng ta thường dùng để enable 1 số service như: IQKeyboard, Firebase, …

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

Ngoài ra, ứng dụng iOS sẽ nằm 1 trong 5 trạng thái sau đây:

* Active (Foreground): Khi ứng dụng đang hoạt động bình thường, chúng ta thao tác trên đó
* Inactive (Foreground): nếu có sự kiện từ hệ thống (như có tin nhắn mới, có điện thoại) thì ứng dụng chuyển về **Inactive**
* Background: Khi người dùng bấm phím Home (ẩn ứng dụng xuống nền), ứng dụng sẽ chuyển sang trạng thái background. Ở trạng thái này, ứng dụng không hiển thị với người dùng nhưng vẫn thực hiện các dòng lệnh. Trong trạng thái này, app sẽ tiếp tục chạy nhưng sẽ bị giới hạn về tài nguyên và thời gian chạy
* Suspended: Ứng dụng ở trạng thái background nhưng không còn thực hiện dòng lệnh. Không có thời hạn cố định. Process có thể còn trong bộ nhớ nhưng không thực thi; hệ thống có thể terminate mà không tự lưu toàn bộ trạng thái ứng dụng.
* Terminated: Khi người dùng đóng app hoặc hệ thống bị đột ngột tắt, app sẽ bị giải phóng khỏi bộ nhớ và chuyển sang trạng thái Terminated. Điện thoại cần memory để chạy 1 ứng dụng khác, thì ứng dụng ở trạng thái suspended sẽ bị chấm dứt và chuyển xuống trạng thái Terminate&#x64;**.** Khi app chuyển sang trạng thái Terminated, tất cả các tài nguyên và bộ nhớ đang được sử dụng bởi app đều được giải phóng

## App lifecycle hiện đại: AppDelegate, SceneDelegate và SwiftUI

Từ iOS 13, app có thể có nhiều UIScene. UIApplicationDelegate xử lý process-level launch/registration, SceneDelegate xử lý từng window scene. SwiftUI dùng App/Scene và scenePhase (.active/.inactive/.background). Suspended là process còn trong memory nhưng không chạy code; không có thời hạn 5–10 giây cố định, có thể bị system terminate. System không đảm bảo callback applicationWillTerminate hoặc tự lưu mọi state.

```swift
import SwiftUI

struct LifecycleView: View {
    @Environment(\.scenePhase) private var phase
    var body: some View {
        Text("Home")
            .onChange(of: phase) { newPhase in
                if newPhase == .background {
                    // Persist small state, stop UI-only work.
                }
            }
    }
}
```

Ví dụ onChange overload iOS 14+, API mới iOS 17 có old/new value overload. Background execution giới hạn theo system: beginBackgroundTask để hoàn tất work ngắn với expiration handler, BGTaskScheduler cho deferred tasks, background URLSession cho transfer, capability theo feature. Không đảm bảo thời điểm chạy. Save quan trọng ngay khi state đổi, không chờ terminate.

**Q: App inactive vs background?** Inactive có thể foreground nhưng không nhận interaction; background không hiển thị, execution giới hạn.

**Q: SceneDelegate vs AppDelegate?** Scene xử lý window/session lifecycle, AppDelegate process-level.

**Q: App bị kill có callback luôn không?** Không; persist dữ liệu sớm và restore khi launch.

**Q: async/await có giữ app chạy background?** Không, cần system-approved background mode/task phù hợp.

**Q: SwiftUI scenePhase và view onDisappear khác gì?** Scene state toàn window/app context, onDisappear là view visibility/identity.
