# Social Login, Payment và QR Code

### Social Login

OAuth 2.0 là authorization, OpenID Connect thêm identity. Native app dùng Authorization Code + PKCE, ASWebAuthenticationSession, redirect URI, state và nonce theo flow. Backend xác minh token/issuer/audience/expiry; không coi email client gửi là chứng minh identity. Lưu token bằng Keychain, revoke/logout phù hợp, không nhúng secret.

### VNPay / Visa Payment Flow

1. App yêu cầu backend tạo order (amount/currency/order ID từ server), backend ký request và trả payment URL.
2. Mở SDK hoặc browser/WebView theo hướng dẫn provider; Visa là card network, tích hợp thực tế qua payment gateway/SDK, không suy ra SDK từ CV.
3. User trả về qua callback/deep link. App chỉ hiển thị pending và query backend; backend xác thực chữ ký/webhook/IPN, amount/order/currency, xử lý idempotency.
4. Backend cập nhật paid/failed/expired/refunded, app refresh khi cold start, mất mạng hoặc user cancel. Không cấp voucher chỉ dựa vào URL success.

Không lưu PAN/CVV, không log payment URL chứa secret, không tự implement card handling khi gateway hosted flow đủ dùng. IAP dành cho digital goods theo App Store rules; hàng hóa/dịch vụ sử dụng ngoài app là trường hợp khác, kiểm tra guideline theo feature.

### QR Code với AVFoundation

```swift
import AVFoundation

func configureQR(session: AVCaptureSession,
                 delegate: AVCaptureMetadataOutputObjectsDelegate) throws {
    guard let camera = AVCaptureDevice.default(for: .video) else {
        throw CocoaError(.featureUnsupported)
    }
    let input = try AVCaptureDeviceInput(device: camera)
    let output = AVCaptureMetadataOutput()
    session.beginConfiguration()
    defer { session.commitConfiguration() }
    guard session.canAddInput(input), session.canAddOutput(output) else {
        throw CocoaError(.featureUnsupported)
    }
    session.addInput(input)
    session.addOutput(output)
    output.setMetadataObjectsDelegate(delegate, queue: .main)
    if output.availableMetadataObjectTypes.contains(.qr) {
        output.metadataObjectTypes = [.qr]
    }
}
```

Cần NSCameraUsageDescription, request permission; cấu hình/startRunning/stopRunning trên serial session queue, preview layer/UI main thread. Delegate parse stringValue, validate format, debounce repeated frames, backend redeem atomic one-time voucher. QR không chứa secret có thể replay; dùng opaque token/expiry và verify server.

### Q&A

**Q: App tắt trước payment callback?** Server webhook là source of truth, query order lúc app mở lại.

**Q: QR được scan 2 lần?** Client debounce không đủ; backend transaction/idempotency bảo đảm voucher chỉ redeem một lần.

**Liên hệ:** DaPass – social login, VNPay/Visa, QR voucher.
