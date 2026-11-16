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

## Networking chuyên sâu

### HTTP pipeline, error mapping và request lifecycle

URLRequest (method, headers, body, cache policy) → URLSessionConfiguration (timeout, cookies, cache, delegate) → task → DNS/TCP/TLS/HTTP → status/body → decode → domain mapping. URLSession không coi HTTP 404/500 là transport error, phải validate status.

```swift
import Foundation

struct CreateOrder: Encodable { let productID: Int }
struct OrderDTO: Decodable { let id: String }

func createOrder(baseURL: URL, body: CreateOrder, token: String,
                 idempotencyKey: String) async throws -> OrderDTO {
    var request = URLRequest(url: baseURL.appendingPathComponent("orders"))
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.setValue("Bearer " + token, forHTTPHeaderField: "Authorization")
    request.setValue(idempotencyKey, forHTTPHeaderField: "Idempotency-Key")
    request.httpBody = try JSONEncoder().encode(body)
    let (data, response) = try await URLSession.shared.data(for: request)
    guard let http = response as? HTTPURLResponse else {
        throw NetworkError.invalidResponse
    }
    guard (200..<300).contains(http.statusCode) else {
        throw NetworkError.status(http.statusCode)
    }
    return try JSONDecoder().decode(OrderDTO.self, from: data)
}
```

Idempotency-Key chỉ có ý nghĩa nếu backend hỗ trợ; giữ cùng key khi retry cùng operation. GET thường safe/idempotent, PUT/DELETE idempotent theo HTTP semantics, POST không mặc định. 204 không có body, không decode JSON rỗng. 429 dùng Retry-After, 401 refresh, 403 không retry auth vô hạn, 5xx có bounded retry cho operation an toàn.

### Alamofire: request, validation và interceptor

```swift
import Alamofire

func loadProducts(url: URL) async throws -> [Product] {
    try await AF.request(url, method: .get)
        .validate(statusCode: 200..<300)
        .serializingDecodable([Product].self)
        .value
}
```

Product là Decodable model trong ví dụ. Session tùy chỉnh giữ configuration/interceptor; RequestInterceptor.adapt gắn header, retry nhận response/error và gọi completion với retry/delay/doNotRetry. Authenticator/AuthenticationInterceptor có thể quản lý refresh theo SDK; xác minh version. Không gọi refresh thông qua cùng interceptor bị 401 lặp vô hạn. Dùng single-flight gate ở bài Swift Concurrency.

### Moya: Provider, plugin và stub

```swift
import Moya

let provider = MoyaProvider<ProductAPI>()
provider.request(.list(page: 1)) { result in
    switch result {
    case .success(let response):
        do {
            let products = try response
                .filterSuccessfulStatusCodes()
                .map([Product].self)
            print(products.count)
        } catch {
            // Map HTTP/decode error to UI state.
        }
    case .failure(let error):
        // Transport/provider error, redact logs.
        print(error.localizedDescription)
    }
}
```

Test dùng MoyaProvider<ProductAPI>(stubClosure: MoyaProvider.immediatelyStub), sampleData phù hợp schema, custom endpoint closure để stub status code. PluginType prepare/willSend/didReceive/process phù hợp logging/header/metrics, không log authorization. Moya không tự phân biệt business error và HTTP error nếu không validate.

### Pagination và caching

```text
Idle -> loading(cursor) -> append unique IDs -> nextCursor / exhausted
Error -> retry same cursor
New query -> reset items + generation, cancel old request
```

Cursor ổn định hơn offset khi data thay đổi, nhưng backend quyết định contract. Tránh load next nhiều lần khi cell onAppear lặp; giữ isLoading/hasMore. Cache-Control/ETag/If-None-Match và 304 khác cache local nghiệp vụ; URLCache tuân theo HTTP policy. Offline-first cần version/TTL/refresh, phân biệt stale content và fresh.

### Q&A bổ sung

**Q: Request timeout và resource timeout?** Request timeout liên quan thời gian chờ thêm dữ liệu; resource timeout giới hạn toàn bộ transfer.

**Q: Retry 401 thế nào?** Single refresh, replay một lần bằng token mới, logout khi refresh invalid; không retry vô hạn.

**Q: SSL pinning có thay ATS không?** Không; thêm ràng buộc identity, cần rotation/backup và vẫn validate trust.

**Bài tập:** Stub 200, 204, 401, 429, malformed JSON bằng URLProtocol; test mapping và retry count.
