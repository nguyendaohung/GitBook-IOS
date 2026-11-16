# Localization, Security và AI-assisted Development

### Localization

String Catalog (.xcstrings) hoặc Localizable.strings/.stringsdict theo target/OS; NSLocalizedString/String(localized:), pluralization, format Date/Number/Currency theo locale, RTL và Dynamic Type. Không nối chuỗi bằng từ rời khi thứ tự ngữ pháp khác nhau. Test pseudo-localization, tiếng dài, fallback, font và accessibility.

```swift
import Foundation

let title = String(localized: "premium.title")
let price = 99000.formatted(.currency(code: "VND"))
```

Giá StoreKit dùng Product.displayPrice thay vì format tự tính. R.Swift có generated resource accessors theo setup.

### Security và Privacy

* HTTPS/ATS, certificate validation mặc định; pinning chỉ khi có threat model/rotation; không bypass trust. OAuth PKCE, Keychain, least privilege, revoke/logout.
* Permission xin theo context, Info.plist usage description, entitlement chỉ đúng feature. App Groups không phải encryption; file protection, không giữ SMS/phone data quá nhu cầu.
* App privacy labels, privacy manifest, required-reason API, SDK data collection, consent/ATT. Validate deep link, backend authorization, idempotency payment/entitlement; không tin client state.
* Không nhúng secret, không log token/PII; code signing và dependency audit. LocalAuthentication xác minh local user, server vẫn xác minh session.

### AI-assisted Development

GitHub Copilot, ChatGPT, Claude, Cursor hỗ trợ draft code, test, refactor, docs. Không gửi secret, code khách hàng hay user data vào công cụ không được phép. Cung cấp context giới hạn, OS target, architecture, coding convention; review API/version, edge cases, concurrency, security; chạy test/lint và đo hiệu năng. Không dùng AI output làm bằng chứng kinh nghiệm thực tế.

### Q&A

**Q: Localization plural xử lý thế nào?** String Catalog/stringsdict theo locale, không hardcode singular/plural.

**Q: AI sinh code có thể merge ngay?** Không; review correctness, license/privacy, test và verify API.

**Liên hệ:** FoxCode localization, AI-assisted workflow, production maintenance; tất cả dự án có privacy/security boundary.

## Production practices chuyên sâu

### Localization và accessibility

String Catalog (.xcstrings) quản lý translation/pluralization; legacy .strings/.stringsdict nếu target cần. Không nối câu từ nhiều literal vì ngữ pháp khác nhau. Dùng format localized number/date/currency, locale/time zone rõ ràng. Key có context, translator comment; fallback language. Test text dài, RTL, Dynamic Type, VoiceOver, contrast.

```swift
import Foundation

func remainingItems(_ count: Int) -> String {
    String(localized: "You have \(count) items",
           comment: "Number of items remaining")
}
```

Configure plural variants trong String Catalog; ví dụ không tự xử lý plural bằng if count == 1 cho mọi ngôn ngữ. Giá StoreKit dùng Product.displayPrice, không format currency bằng hardcoded USD.

```swift
import SwiftUI

struct StatusRow: View {
    let connected: Bool
    var body: some View {
        Label(connected ? "Connected" : "Disconnected",
              systemImage: connected ? "checkmark.circle" : "xmark.circle")
            .accessibilityLabel(connected ? "VPN connected" : "VPN disconnected")
    }
}
```

Không dựa chỉ màu xanh/đỏ cho state. UIKit dùng accessibilityLabel/value/traits, preferredFont(forTextStyle:), adjustsFontForContentSizeCategory.

### Security boundary

Client không đáng tin: backend authorize từng resource, validate order/amount/entitlement, idempotency, rate limit. HTTPS/ATS, trust validation mặc định; pinning thêm ràng buộc khi threat model cần và rotation/backup pins. Không disable certificate check để fix dev. Keychain cho token, App Groups file protection cho shared data, không lưu SMS/phone/photo không cần thiết. Secure logging redact secrets; avoid clipboard/token deep link. Dependency audit, privacy manifest, ATT/consent và App Store labels khác nhau.

### AI-assisted workflow có kiểm soát

Đưa context tối thiểu: deployment target, architecture, rule style, acceptance criteria, test cases. Yêu cầu AI giải thích trade-off, không gửi secret/client data/code không được phép. Review generated code, xác minh Apple API/version, compile, run tests, profile. Kiểm tra hallucinated API, race, retain cycle, license/security. Không dùng AI output thay bằng chứng kinh nghiệm.

**Q: Tại sao không hardcode API secret?** App binary có thể inspect; secret ở backend/CI secret store, client dùng short-lived credentials.

**Q: ATT denied có nghĩa không analytics?** Không đồng nhất; tracking theo Apple definition cần permission, first-party analytics vẫn phải tuân privacy/consent.

**Bài tập:** Review một API client có log token/skip trust và sửa; localize screen với plural/RTL, test VoiceOver.

## Câu hỏi phỏng vấn tình huống

**Q: String(localized:) có tự xử lý plural mọi ngôn ngữ?** Cần String Catalog/stringsdict plural variants và context; format locale, test translations/RTL.

**Q: Privacy manifest vs ATT?** Manifest mô tả data/required-reason APIs theo requirements; ATT permission cho tracking theo Apple definition; labels/consent là layer riêng.

**Q: AI tạo API không tồn tại xử lý?** Đối chiếu docs/SDK version, compile/test, review concurrency/security/license, không gửi secret/user data ngoài policy.
