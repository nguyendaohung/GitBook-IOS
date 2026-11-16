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
