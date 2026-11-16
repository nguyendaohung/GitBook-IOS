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
