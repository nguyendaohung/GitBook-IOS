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
