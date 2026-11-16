# UserNotifications, APNs và Deep Link

### Định nghĩa

NotificationCenter là in-process event broadcast, khác UserNotifications (system local/remote notification). Local schedule trên thiết bị; remote gửi qua APNs từ server. FCM có thể làm trung gian APNs nhưng CV không xác nhận SDK.

```swift
import UserNotifications

func scheduleReminder() async throws {
    let center = UNUserNotificationCenter.current()
    guard try await center.requestAuthorization(options: [.alert, .sound]) else {
        return
    }
    let content = UNMutableNotificationContent()
    content.title = "Reminder"
    content.body = "Check your favorites"
    content.userInfo = ["route": "favorites"]
    let request = UNNotificationRequest(
        identifier: "favorites-reminder",
        content: content,
        trigger: UNTimeIntervalNotificationTrigger(timeInterval: 3600, repeats: false)
    )
    try await center.add(request)
}
```

### Push flow

1. App đăng ký remote notifications, nhận APNs device token trong AppDelegate, gửi server theo user/device/environment; token có thể đổi.
2. Server gửi payload APNs với topic/token và authentication; alert push vs background push (content-available), đúng push type/priority.
3. Foreground xử lý UNUserNotificationCenterDelegate.willPresent; tap dùng didReceive response. Cold start route khi app/session/navigation đã sẵn sàng.
4. Silent push không đảm bảo giao/thời điểm chạy; system throttle, không dùng thay job scheduler. Notification Service Extension có giới hạn thời gian để sửa nội dung remote mutable-content.

### Deep Link và Location

Custom scheme có thể bị app khác claim; Universal Links dùng Associated Domains + apple-app-site-association. Validate scheme/host/path, auth guard, route whitelist; không coi link là proof purchase. Location-based notifications có thể dùng UNLocationNotificationTrigger hoặc backend geofencing tùy feature/permission; không giả định cách dự án đã dùng. Xử lý permission denied, timezone, duplicate identifier, notification settings.

### Q&A

**Q: Có permission alert mới nhận device token không?** Đăng ký APNs và quyền hiển thị alert là hai bước khác nhau.

**Q: Push nhận foreground nhưng không hiện banner?** Kiểm tra delegate willPresent, authorization/settings và presentation options.

**Q: Tap notification lúc app chưa mở?** Parse route, chờ navigation/auth sẵn sàng, xử lý idempotent.

**Liên hệ:** DaPass/BIDU, Karaoke, FastVPN, Watch Faces.

## Push notification từ server đến app

### APNs pipeline

```text
App -> registerForRemoteNotifications -> device token -> backend
Backend -> APNs (token, topic, push type, auth) -> iOS
-> notification center / app delegate / service extension
```

APNs token là opaque Data, có thể đổi, gửi backend khi callback; không hardcode length. Sandbox/production APNs environment phải khớp signing. Backend lưu token theo device/user, remove invalid token khi APNs báo invalid/unregistered; logout unlink user. Alert permission khác registration. Foreground decide presentation bằng delegate; tap xử lý route; silent push không bảo đảm chạy.

```swift
import UIKit
import UserNotifications

final class AppDelegate: UIResponder, UIApplicationDelegate,
                         UNUserNotificationCenterDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions:
                     [UIApplication.LaunchOptionsKey: Any]? = nil) -> Bool {
        UNUserNotificationCenter.current().delegate = self
        application.registerForRemoteNotifications()
        return true
    }

    func application(_ application: UIApplication,
                     didRegisterForRemoteNotificationsWithDeviceToken token: Data) {
        let hex = token.map { String(format: "%02x", $0) }.joined()
        // Send token to authenticated backend; do not log it.
        _ = hex
    }

    func userNotificationCenter(_ center: UNUserNotificationCenter,
                                willPresent notification: UNNotification,
                                withCompletionHandler completionHandler:
                                @escaping (UNNotificationPresentationOptions) -> Void) {
        completionHandler([.banner, .sound])
    }
}
```

Ví dụ iOS 14+ cho .banner; SwiftUI App dùng UIApplicationDelegateAdaptor nếu cần. Authorization request riêng theo context. Server payload ví dụ:

```json
{"aps":{"alert":{"title":"New voucher","body":"View details"},"sound":"default"},"route":"voucher","id":"123"}
```

Không đưa secret/PII vào payload. APNs headers apns-topic, apns-push-type alert/background, priority phù hợp; collapse-id coalesce thông báo thay thế. Background content-available:1 không phải scheduler, system throttle. Service extension mutable-content dùng để sửa alert/media, có thời hạn; luôn có fallback.

### Deep link router

```swift
enum NotificationRoute {
    case voucher(id: String)
    case favorites
}

func route(from userInfo: [AnyHashable: Any]) -> NotificationRoute? {
    switch userInfo["route"] as? String {
    case "voucher":
        guard let id = userInfo["id"] as? String, !id.isEmpty else { return nil }
        return .voucher(id: id)
    case "favorites": return .favorites
    default: return nil
    }
}
```

Parse không đồng nghĩa authorize. Router chờ session/navigation ready, backend check voucher ownership; idempotent tap và cold launch. Universal Link AASA host/path, Associated Domains, test install/redirect; custom scheme tránh chứa token. Local trigger calendar/time interval/location, repeating interval tối thiểu theo API; identifier dùng cancel/reschedule.

**Q: Push token có thay đổi?** Có, luôn cập nhật từ callback, không coi token là user ID.

**Q: App foreground không thấy banner?** Kiểm tra willPresent, notification settings và OS focus.

**Bài tập:** Schedule reminder, cancel theo ID, test foreground/cold launch route, payload thiếu field và permission denied.

## Câu hỏi phỏng vấn tình huống

**Q: APNs sandbox vs production token?** Token/environment/topic cần đúng signing/backend, token opaque có thể đổi; unlink khi logout và handle invalid token.

**Q: Silent push có dùng làm guaranteed scheduler?** Không, delivery/execution best effort, throttle/system state; BGTaskScheduler cũng không đảm bảo thời điểm.

**Q: Universal Link có bypass login không?** Không, validate route/host/path và auth/resource ownership; chờ navigation ready, idempotent tap.

**Q: Notification Service Extension timeout?** Phải cung cấp fallback content, gọi contentHandler đúng contract, không network dài; mutable-content phù hợp.
