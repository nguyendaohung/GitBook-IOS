# URLSession, Alamofire, Moya và REST API

### Định nghĩa

URLSession là native API HTTP; Alamofire bổ sung request/validation/interceptor; Moya xây abstraction TargetType trên Alamofire để mô tả endpoint, stub và plugin.

```swift
import Foundation

enum NetworkError: Error {
    case invalidResponse
    case status(Int)
}

struct APIClient {
    let session: URLSession

    func get<T: Decodable>(_ type: T.Type, url: URL) async throws -> T {
        let (data, response) = try await session.data(from: url)
        guard let http = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        guard (200..<300).contains(http.statusCode) else {
            throw NetworkError.status(http.statusCode)
        }
        return try JSONDecoder().decode(type, from: data)
    }
}
```

### Cơ chế hoạt động

* DataTask nhận data; DownloadTask ghi file tạm (move trước khi callback kết thúc); UploadTask gửi body/file. URLSessionConfiguration default/ephemeral/background; background transfer do system quản lý, không phải mọi async task đều chạy khi app suspend.
* URLSessionTaskMetrics tách DNS, TCP, TLS, request/response; timeout request khác resource timeout.
* Alamofire Session, RequestInterceptor (adapt/retry), validate(), responseDecodable. Moya TargetType có baseURL/path/method/task/headers; provider.request, filterSuccessfulStatusCodes, map/decodable; PluginType log đã redact, StubBehavior để test.
* HTTP transport error khác status 4xx/5xx và lỗi decode. 401 refresh token có single-flight để tránh nhiều refresh đồng thời; retry request tối đa một lần sau refresh.
* Retry có exponential backoff + jitter, tôn trọng Retry-After; retry request idempotent hoặc có idempotency key, không retry purchase mù quáng.
* Pagination cursor/page, isLoading, hasMore, deduplicate theo ID, reset khi query thay đổi, ignore stale response.
* HTTPS kiểm tra certificate qua hệ thống; pinning chỉ khi có threat model và rotation/backup pins. Không bỏ trust validation.
* OAuth native dùng Authorization Code + PKCE, ASWebAuthenticationSession, state/nonce theo protocol; lưu token trong Keychain, không nhúng client secret.
* Deep link universal link cần Associated Domains + AASA; validate URL và route, không tin tham số thanh toán từ client.

### Ví dụ Moya (cần dependency Moya)

```swift
import Moya

enum ProductAPI: TargetType {
    case list(page: Int)
    var baseURL: URL { URL(string: "https://example.com")! }
    var path: String { "/products" }
    var method: Moya.Method { .get }
    var task: Moya.Task {
        switch self {
        case .list(let page):
            return .requestParameters(parameters: ["page": page],
                                      encoding: URLEncoding.queryString)
        }
    }
    var headers: [String: String]? { ["Accept": "application/json"] }
    var sampleData: Data { Data("[]".utf8) }
}
```

### Q&A

**Q: Moya khác Alamofire?** Moya gom endpoint thành type, tái sử dụng provider/plugin/stub; Alamofire linh hoạt ở tầng request/session.

**Q: HTTP 200 có chắc nghiệp vụ thành công?** Không, cần kiểm tra business status/schema và server contract.

**Q: NWPathMonitor báo satisfied có chắc API chạy?** Không, DNS/TLS/server vẫn có thể fail; request thực tế là nguồn kết quả.

**Liên hệ:** DaPass/BIDU dùng Moya, Map4D Alamofire, FastVPN URLSession.
