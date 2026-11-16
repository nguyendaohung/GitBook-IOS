# Keychain Access

### Định nghĩa

Keychain Services lưu credential/token dưới cơ chế bảo vệ hệ thống. UserDefaults không dùng cho password/access token. Keychain access group cần entitlement để chia sẻ giữa app/extension; khác App Groups dùng shared container.

```swift
import Foundation
import Security

enum KeychainError: Error {
    case status(OSStatus)
}

func saveToken(_ token: Data, account: String) throws {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrService as String: "com.example.auth",
        kSecAttrAccount as String: account
    ]
    let attributes = [kSecValueData as String: token]
    let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)
    if status == errSecItemNotFound {
        var item = query
        item[kSecValueData as String] = token
        item[kSecAttrAccessible as String] = kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        let result = SecItemAdd(item as CFDictionary, nil)
        guard result == errSecSuccess else { throw KeychainError.status(result) }
    } else if status != errSecSuccess {
        throw KeychainError.status(status)
    }
}
```

Đọc bằng SecItemCopyMatching với kSecReturnData, xóa SecItemDelete khi logout. Xử lý errSecItemNotFound, errSecInteractionNotAllowed và duplicate do concurrent writes; serialize nếu cần. Chọn accessibility theo yêu cầu background/locked device, không tự giảm bảo mật.

### Face ID

LocalAuthentication (LAContext) xác thực người dùng cục bộ, không gửi dữ liệu Face ID lên server. Có thể bảo vệ keychain item bằng SecAccessControl (userPresence/biometryCurrentSet tùy yêu cầu). Server vẫn kiểm tra token/session; không lưu mật khẩu chỉ để đăng nhập lại.

### Q&A

**Q: Keychain có bị xóa khi uninstall?** Không nên dựa vào hành vi giữ/xóa sau reinstall như contract; thiết kế logout/revoke rõ ràng.

**Q: App Groups có thay thế Keychain?** Không; shared file container và credential storage khác mục đích.

**Liên hệ:** DaPass social login, REST API authentication.

## Đọc, xóa token và chọn accessibility

```swift
import Foundation
import Security

func readToken(account: String) throws -> Data? {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrService as String: "com.example.auth",
        kSecAttrAccount as String: account,
        kSecReturnData as String: true,
        kSecMatchLimit as String: kSecMatchLimitOne
    ]
    var result: CFTypeRef?
    let status = SecItemCopyMatching(query as CFDictionary, &result)
    if status == errSecItemNotFound { return nil }
    guard status == errSecSuccess else { throw KeychainError.status(status) }
    return result as? Data
}

func deleteToken(account: String) throws {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrService as String: "com.example.auth",
        kSecAttrAccount as String: account
    ]
    let status = SecItemDelete(query as CFDictionary)
    guard status == errSecSuccess || status == errSecItemNotFound else {
        throw KeychainError.status(status)
    }
}
```

Service/account cùng định danh với saveToken ở trên. Không dùng query quá rộng khi delete. Khi logout, revoke session server, clear in-memory credential và keychain; hủy refresh đang chạy để token cũ không xuất hiện lại.

| Accessibility | Ý nghĩa |
| --- | --- |
| WhenUnlocked | Đọc khi thiết bị unlocked |
| AfterFirstUnlock | Sau lần unlock đầu tiên sau boot, phù hợp background khi cần |
| ThisDeviceOnly | Không migrate sang thiết bị khác qua backup |
| WhenPasscodeSetThisDeviceOnly | Yêu cầu passcode; item mất khả dụng khi passcode bị gỡ |

### Face ID vs Keychain

```swift
import LocalAuthentication

func authenticate() async throws -> Bool {
    let context = LAContext()
    var error: NSError?
    guard context.canEvaluatePolicy(.deviceOwnerAuthentication, error: &error) else {
        return false
    }
    return try await context.evaluatePolicy(
        .deviceOwnerAuthentication,
        localizedReason: "Unlock your account"
    )
}
```

deviceOwnerAuthentication cho phép passcode fallback; deviceOwnerAuthenticationWithBiometrics chỉ biometric. Cần NSFaceIDUsageDescription khi dùng Face ID. Để bảo vệ secret thật sự, dùng SecAccessControl với flags userPresence/biometryCurrentSet khi tạo item, không chỉ set boolean sau evaluatePolicy. Keychain item có access control có thể prompt khi đọc. BiometryCurrentSet thay đổi enrollment có thể làm item không còn truy cập, thiết kế fallback login.

### Q&A bổ sung

**Q: Có nên lưu password để Face ID login?** Không ưu tiên; lưu refresh token có thể revoke, server validate. Local biometric không chứng minh session còn hiệu lực.

**Q: Keychain vs UserDefaults?** Keychain cho secret, UserDefaults cho setting; không thay thế DB/cache.

**Bài tập:** Tạo token store protocol, in-memory fake, test login/read/logout và refresh race. Test device locked/Face ID denied trên thiết bị thật.

## Câu hỏi phỏng vấn tình huống

**Q: AfterFirstUnlock vs WhenUnlocked?** AfterFirstUnlock khả dụng sau unlock đầu tiên, phù hợp background cần secret; WhenUnlocked chỉ khi unlocked. Chọn least privilege.

**Q: Face ID evaluatePolicy true có chứng minh backend login?** Không, local authentication; server validate token/session. Dùng access control bảo vệ Keychain item thật sự.

**Q: Logout và refresh race xử lý?** Invalidate session generation, cancel in-flight refresh, revoke server và delete keychain; không để old token ghi lại.
