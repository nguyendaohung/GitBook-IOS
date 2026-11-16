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

## Payment và QR: thiết kế state machine

### Payment state

```text
idle -> creatingOrder -> awaitingPayment -> verifying
                                  -> paid / failed / expired / cancelled
App restart -> query backend order -> render server state
```

Client callback chỉ là tín hiệu refresh; server webhook/IPN phải verify chữ ký, amount/currency, order ID, trạng thái và idempotency. Không cộng tiền/cấp voucher nhiều lần khi callback và webhook tới đồng thời. Visa là card network; dùng gateway/SDK hỗ trợ, không lưu PAN/CVV. Test sandbox trước production, không hardcode merchant secret.

```swift
enum PaymentState: Equatable {
    case idle, pending(orderID: String), paid(orderID: String)
    case failed(message: String)
}

struct OrderStatus: Decodable {
    let id: String
    let status: String
}

func paymentState(from status: OrderStatus) -> PaymentState {
    switch status.status {
    case "paid": return .paid(orderID: status.id)
    case "pending": return .pending(orderID: status.id)
    default: return .failed(message: "Payment not completed")
    }
}
```

Production dùng enum server contract và unknown state riêng, xử lý refund/expiry/cancel theo nghiệp vụ. Backend idempotency key ổn định mỗi checkout; app giữ pending order ID (không giữ card secret). Poll bounded, refresh on foreground; không retry charge POST nếu server không hỗ trợ idempotency.

### Social login OAuth/OIDC

Authorization Code + PKCE: app tạo verifier/challenge, mở ASWebAuthenticationSession, provider callback code + state, exchange code theo provider/backend, verify identity token issuer/audience/nonce/expiry. Không gửi password vào WebView tự dựng; tránh custom scheme hijack, ưu tiên provider-approved redirect. Access token ngắn hạn, refresh token bảo vệ Keychain, logout revoke. Sign in with Apple có identity token/authorizationCode, tên/email có thể chỉ cấp lần đầu; backend lưu account ID đúng.

### QR redeem thực tế

```swift
import AVFoundation

final class QRDelegate: NSObject, AVCaptureMetadataOutputObjectsDelegate {
    var onCode: ((String) -> Void)?
    private var processing = false

    func metadataOutput(_ output: AVCaptureMetadataOutput,
                        didOutput metadataObjects: [AVMetadataObject],
                        from connection: AVCaptureConnection) {
        guard !processing,
              let object = metadataObjects.first as? AVMetadataMachineReadableCodeObject,
              object.type == .qr,
              let code = object.stringValue, !code.isEmpty else { return }
        processing = true
        onCode?(code)
    }

    func reset() { processing = false }
}
```

Delegate queue main như ví dụ cấu hình trước, reset sau server response/retry. Capture session chạy serial queue, preview layer resize trên main, permission và interruption xử lý. Validate token format/length, không mở arbitrary URL hoặc thực thi nội dung QR. Backend redeem atomically, expiry, one-time token; client debounce chỉ giảm request lặp.

**Q: Payment callback success có đủ?** Không, server verify là source of truth.

**Q: OAuth vs authentication?** OAuth cấp quyền, OIDC thêm identity; validate token và server session.

**Q: Camera permission denied?** Giải thích và hướng dẫn Settings, fallback manual code nếu nghiệp vụ cho phép.

**Bài tập:** Mock order pending→paid, callback duplicate, app restart, QR repeated frames và redeem already used.

## Câu hỏi phỏng vấn tình huống

**Q: PKCE, state và nonce có vai trò gì?** PKCE chống code interception/replay với verifier; state chống CSRF/correlation, nonce trong OIDC liên kết identity token với auth request. Verify theo provider/backend.

**Q: Timeout payment POST có retry ngay?** Query order status/idempotency key trước; server có thể đã charge. Không tin URL callback là paid.

**Q: Camera session chạy ở thread nào?** Configure/start/stop trên serial session queue, preview/UI main; delegate queue xác định callback, debounce/redeem state và server atomicity.
