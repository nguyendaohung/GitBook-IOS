# WatchConnectivity và WCSession

### Định nghĩa

WatchConnectivity chuyển dữ liệu giữa companion iOS/watchOS app. WCSession.default, delegate, activate(); isSupported kiểm tra support. Activation asynchronous; reachable không đồng nghĩa paired/installed.

| API | Mục đích |
| --- | --- |
| sendMessage | Interactive khi counterpart reachable, reply/error |
| updateApplicationContext | Latest state, ghi đè context chưa giao |
| transferUserInfo | Queue dictionary, giao bất đồng bộ |
| transferFile | Queue file + metadata, giao bất đồng bộ |

```swift
import WatchConnectivity

final class WatchSync: NSObject, WCSessionDelegate {
    func activate() {
        guard WCSession.isSupported() else { return }
        let session = WCSession.default
        session.delegate = self
        session.activate()
    }

    func syncFavorites(_ ids: [String]) throws {
        let session = WCSession.default
        guard session.activationState == .activated else { return }
        try session.updateApplicationContext(["favorites": ids])
    }

    // MARK: - WCSessionDelegate
    func session(_ session: WCSession,
                 activationDidCompleteWith activationState: WCSessionActivationState,
                 error: Error?) {
        // Publish activation state / retry queued state.
    }

    func session(_ session: WCSession,
                 didReceiveApplicationContext applicationContext: [String: Any]) {
        let ids = applicationContext["favorites"] as? [String] ?? []
        // Apply snapshot; UI update on main actor.
        print(ids.count)
    }

    #if os(iOS)
    func sessionDidBecomeInactive(_ session: WCSession) {}
    func sessionDidDeactivate(_ session: WCSession) {
        session.activate()
    }
    #endif
}
```

Giữ instance sống theo lifecycle; delegate callback không mặc định main. iOS kiểm tra isPaired/isWatchAppInstalled, xử lý đổi watch.

### Đồng bộ thực tế

* Context cho latest favorites/settings; userInfo cho event có ID/dedup; file cho asset lớn. Không gửi ảnh lớn qua sendMessage.
* File nhận ở URL tạm: move/copy trong didReceive file trước system cleanup; validate size/type, không tin path metadata.
* Không giả định transfer giao ngay hoặc đúng thứ tự giữa các loại; version/schema/timestamp, conflict policy (server-authoritative hoặc local merge).
* Subscription entitlement xác minh StoreKit/server; sync state chỉ cho UI, không tin message làm proof purchase.
* Test iPhone + watch thật, airplane mode, paired nhưng app không cài, reachable false, đổi watch, duplicate transfer.

### Q&A

**Q: sendMessage vs transferUserInfo?** sendMessage cần reachable, phản hồi tức thời; transferUserInfo queue giao sau.

**Q: Context có mất update không?** Chỉ giữ latest state, không dùng cho event log.

**Q: Vì sao UI không update ngay?** Delegate có thể background, chuyển main actor.

**Liên hệ:** Watch Faces – content sync.

[WatchConnectivity](https://developer.apple.com/documentation/watchconnectivity)
