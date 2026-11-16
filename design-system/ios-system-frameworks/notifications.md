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
