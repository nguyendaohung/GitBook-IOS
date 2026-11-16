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

## WatchConnectivity chuyên sâu

### Session lifecycle và interactive message

Activation asynchronous; iOS có thể inactive/deactivated khi đổi watch, phải activate lại. isPaired/isWatchAppInstalled chỉ iOS; reachable có thể đổi bất kỳ lúc nào. Background transfer không cần reachable nhưng không đảm bảo deadline.

```swift
import WatchConnectivity

func sendSelection(id: String,
                   onReply: @escaping ([String: Any]) -> Void,
                   onError: @escaping (Error) -> Void) {
    let session = WCSession.default
    guard session.activationState == .activated, session.isReachable else {
        onError(NSError(domain: "WatchSync", code: 1))
        return
    }
    session.sendMessage(["type": "select", "id": id],
                        replyHandler: onReply, errorHandler: onError)
}
```

Reachability có thể đổi sau guard; luôn xử lý errorHandler. Receiver implement didReceiveMessage với replyHandler, reply đúng một lần. Nếu action cần giao sau, queue userInfo với event ID; không gửi cả hai đường mà thiếu dedup.

### Latest snapshot vs event queue

```swift
import Foundation

struct FavoritesSnapshot: Codable {
    let version: Int
    let ids: [String]
}

func updateFavorites(_ snapshot: FavoritesSnapshot) throws {
    let data = try JSONEncoder().encode(snapshot)
    try WCSession.default.updateApplicationContext(["snapshot": data])
}
```

Dictionary dùng property-list compatible types, Data Codable có schema rõ. Receiver decode/validate và ignore stale version. Version chỉ đủ nếu có authority thống nhất; multi-writer cần conflict policy, ví dụ server revision/merge. Application context latest state, không dùng cho event increment. transferUserInfo phù hợp event có UUID, persist dedup/idempotent apply.

### File transfer

```swift
func queueImageFile(url: URL, assetID: String) {
    WCSession.default.transferFile(url, metadata: ["assetID": assetID])
}
```

Receiver didReceive file phải copy/move khỏi temporary URL trước callback return, validate file size/type, map metadata ID sang safe path. Không dùng path do metadata cung cấp tùy ý. didFinish fileTransfer:error theo dõi lỗi/retry; outstanding transfers/cancel theo lifetime. Downsample trước khi transfer để giảm memory/storage/pin. Delegate callback không mặc định main, chuyển UI update về main actor.

### So sánh REST và WCSession

WCSession sync giữa companion devices, không thay backend cho nhiều user/device. REST giữ catalog/entitlement authoritative; local sync tối ưu UX/offline. Watch app independent có thể có network riêng. Không gửi receipt/token nếu không cần.

**Q: transferUserInfo có giao ngay khi offline?** Không, queue và system giao khi có điều kiện, không guarantee thời điểm.

**Q: Có gửi UIImage trong dictionary?** Không trực tiếp; property-list types, Data nhỏ hoặc transferFile.

**Q: Duplicate khi retry?** Event ID, persist dedup, idempotent apply và version.

**Bài tập:** Sync favorites context, event userInfo và thumbnail file; test offline, đổi watch, duplicate, stale version.

## Câu hỏi phỏng vấn tình huống

**Q: isPaired, isWatchAppInstalled và isReachable khác nhau?** Paired/installed mô tả companion presence trên iOS; reachable cho interactive messaging hiện tại, có thể đổi ngay.

**Q: Watch đổi pairing thì session thế nào?** iOS inactive/deactivate callbacks, reactivate session và reconcile watch-specific state; không dùng stale queued state tùy ý.

**Q: File transfer receiver cần làm gì?** Copy/move temporary URL trong callback trước system cleanup, validate metadata/file, dedup/version; không giữ URL tạm.

**Q: Multi-device conflict favorites?** Stable item IDs + authority/version/merge policy, event dedup; applicationContext chỉ latest snapshot, không event log.
