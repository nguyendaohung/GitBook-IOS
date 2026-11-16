# CocoaPods, SPM, CI/CD và App Store Release

### Dependency Management

CocoaPods: Podfile mô tả dependency, Podfile.lock khóa resolution, pod install giữ lock, pod update cập nhật theo yêu cầu; mở xcworkspace. SPM tích hợp Xcode, Package.swift/Package.resolved, pin version, review transitive dependencies. Không commit secret. SwiftLint dùng .swiftlint.yml, lint trong CI; R.Swift sinh accessor type-safe resource (image/string/font tùy cấu hình), đảm bảo build phase/plugin chạy và resource thuộc target đúng.

```ruby
platform :ios, '15.0'
target 'ExampleApp' do
  use_frameworks!
  pod 'Kingfisher'
end
```

Ví dụ Podfile, pin version theo dự án và commit lockfile của app.

### Signing và Release

Certificate xác minh identity ký; provisioning profile liên kết App ID, team, entitlements, certificate và device tùy loại. Bundle ID riêng cho app/extension, App Groups/Network Extension đúng entitlement. Debug/Release configuration, scheme, xcconfig và dev/staging/prod endpoint; không nhúng server secret trong app.

1. CI checkout → resolve dependencies → lint → build/test → archive → export/upload (credentials từ secret store).
2. TestFlight internal/external, beta review nếu cần; QA trên device/OS, extension, purchase, localization, accessibility.
3. App Store Connect metadata, privacy labels, privacy manifest/SDK requirements, screenshots, review notes/account test.
4. Submit review, phased release nếu phù hợp, monitor crash/analytics. Rollback mobile thường cần hotfix build hoặc server feature flag, không giả định downgrade tự động.

Fastlane có scan/run_tests, gym/build_app, pilot/upload_to_testflight, deliver; match quản lý signing theo team policy. CI dùng macOS/Xcode version pinned, cache dependency, không log API key/cert; Git feature branch, PR, review, rebase/merge theo team, revert commit thay rewrite history shared branch.

### Q&A

**Q: Certificate vs provisioning?** Identity ký vs profile cấp quyền chạy/phân phối với App ID/entitlements.

**Q: pod install vs pod update?** install tôn trọng lock; update resolve lại dependency được chọn.

**Q: App review fail extension/VPN?** Kiểm tra entitlement, privacy disclosure, review instructions, device behavior và guideline; sửa đúng nguyên nhân.

**Liên hệ:** DaPass/BIDU CocoaPods/SwiftLint/R.Swift; FoxCode end-to-end release. CI/CD/Fastlane là kiến thức bổ sung, CV chưa ghi công cụ cụ thể.

## Release chuyên sâu: từ build đến App Store

### Target, scheme, configuration, signing

Target là product/build settings/sources (app, test, extension); scheme chọn target/action/configuration cho Run/Test/Archive; build configuration (Debug/Release/custom) chứa flags/xcconfig. Bundle identifier, Team, entitlements và signing phải khớp từng target. Dev/Staging/Prod có base URL/bundle ID khác nếu cần; .xcconfig chứa public config, không chứa secret. Info.plist build settings expansion cho API base URL.

```text
// Staging.xcconfig (ví dụ, không chứa secret)
PRODUCT_BUNDLE_IDENTIFIER = com.example.app.staging
API_HOST = staging.example.com
SWIFT_ACTIVE_COMPILATION_CONDITIONS = $(inherited) STAGING
```

Không hardcode token/API secret trong app binary. Signing certificate private key được bảo vệ; provisioning profile chứa app ID, team, entitlements và device list tùy distribution. Development để debug device; App Store distribution để upload. Automatic signing đơn giản, manual signing cần quản lý cert/profile rotation. App extension/watch app có bundle ID/signing/entitlement riêng, cùng App Group nếu cần.

### SPM vs CocoaPods

| Tiêu chí | Swift Package Manager | CocoaPods |
| --- | --- | --- |
| Tool | Native Swift/Xcode | Ruby + Pods |
| Dependency file | Package.swift / project | Podfile |
| Lock | Package.resolved | Podfile.lock |
| Build integration | Package targets/products | Workspace/Pods project |
| Cập nhật | Resolve/update version rules | pod install vs pod update |

pod install tôn trọng lock, pod update thay version theo constraints. Commit lock file theo app policy; không update toàn bộ vô tình. SPM version ranges, exact/branch/revision trade-off; kiểm tra transitive dependencies, binary checksum, license/privacy manifest. SwiftLint rule config, R.Swift generated resources; ensure build phase/CI chạy đúng và không commit generated tùy rule.

### CI/CD và Fastlane

```ruby
# Fastfile minh họa; cấu hình signing và credentials từ CI secret store.
default_platform(:ios)

platform :ios do
  lane :beta do
    run_tests(scheme: "App")
    build_app(scheme: "App", export_method: "app-store")
    upload_to_testflight
  end
end
```

CI macOS runner/Xcode pinned, resolve dependencies, lint, unit/UI tests, archive, export, upload. App Store Connect API key trong secret store, không in logs. Fastlane match quản lý signing theo team policy; restrict access/private key, read-only CI nếu phù hợp. Cache dependencies theo lock hash, không cache signing credentials tùy ý. Build number unique cho upload; version marketing khác build number. TestFlight internal/external, external review có thể cần.

### Release checklist và incident

Check permission usage descriptions, app groups, VPN/CallKit/watch entitlements, privacy labels/manifest, SDK privacy requirements, ATT, localized screenshots/metadata, StoreKit product/paywall/restore. Test iOS min/latest supported, fresh install/upgrade, migration, offline, dark mode, Dynamic Type, accessibility. Archive validates signatures; export/upload không đồng nghĩa App Review approval. Phased release giúp giảm blast radius; theo dõi crash rate/metrics, pause rollout nếu cần. iOS không rollback binary tức thì; hotfix, feature flag/server rollback theo risk, không tắt security validation.

**Q: TestFlight build có giống App Store?** Cùng release artifact/signing flow nhưng environment/receipt và distribution behavior có thể khác; test production config.

**Q: Certificate vs profile?** Certificate xác định signing identity; profile cấp quyền cho app ID/team/entitlements/device theo loại.

**Bài tập:** Tạo dev/staging schemes + xcconfig, CI test/archive pipeline, mô phỏng signing mismatch và checklist release.
