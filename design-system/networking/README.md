# 😝 NETWORKING

1. **CALL API:**&#x20;

* Tạo URL: Xác định URL API mà chúng ta muốn gọi.
* Tạo một request từ **URL** được xác định tới **API** bằng cách sử dụng **URLRequest** (hoặc các thư viện như **Alamofire** hoặc **Moya),** cung cấp các thông tin yêu cầu như HTTP method, headers, parameters,...
* Sử dụng một **session** (hoặc một thư viện như **Alamofire** hoặc Moya) để quản lý các tác vụ liên quan đến network, bao gồm việc tạo và thực thi các request.
* Gửi request: Sử dụng đối tượng URLSession (hoặc một thư viện như **Alamofire)** hoặc Moya để gửi request với completion handler để xử lý kết quả nhận được.
*   Xử lý kết quả: Trong completion handler, chúng ta cần xử lý kết quả nhận được từ API. Nếu thành công, dữ liệu trả về sẽ được truy cập thông qua các thuộc tính của đối tượng Data hoặc JSONSerialization. Nếu có lỗi, chúng ta cần xử lý lỗi và hiển thị thông báo cho người dùng.

    &#x20;\=>Trong completion handler của đối tượng URLSession, chúng ta sẽ nhận được các tham số:

    * data: Đây là dữ liệu trả về từ API, dưới dạng đối tượng Data hoặc JSONSerialization. Nếu không có dữ liệu trả về, giá trị của biến này sẽ là nil.
    * response: Đối tượng HTTPURLResponse, đại diện cho phản hồi HTTP từ server. Nó chứa các thông tin như mã trạng thái HTTP, headers,...(chứa status code success, failure)
    * error: Đối tượng Error, đại diện cho lỗi (nếu có) xảy ra trong quá trình gọi API.

**Ví dụ**:&#x20;

*   Call API bằng Swift



    <figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 12.29.05.png" alt=""><figcaption></figcaption></figure>
* Call Api bằng Alamofire:&#x20;

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 12.36.28.png" alt=""><figcaption></figcaption></figure>

2. **PARSE JSON:**

Trong Swift, để parse dữ liệu JSON, chúng ta có thể sử dụng các cách sau:

* Sử dụng **JSONSerialization**: đây là một cách đơn giản để parse JSON thành đối tượng Foundation object như NSArray, NSDictionary, hoặc NSNumber. Ví dụ:

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 12.39.12.png" alt=""><figcaption></figcaption></figure>

* Sử dụng **Codable**: đây là một cách đơn giản hơn để parse JSON thành các đối tượng Swift, bằng cách tạo một struct hoặc class tương ứng với đối tượng JSON. Ví dụ:

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 12.40.00.png" alt=""><figcaption></figcaption></figure>

* Sử dụng **SwiftyJSON**: đây là một thư viện Swift dễ sử dụng để parse và truy cập dữ liệu JSON. Ví dụ:

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 12.40.43.png" alt=""><figcaption></figcaption></figure>

### Kiến thức mở rộng

* [URLSession, Alamofire, Moya](api-client.md)
* [Network Framework và VPN](network-vpn.md)
* [Speed Test](speed-test.md)

## Q&A tổng quan Networking

**Q: URLSession có cần background queue để gọi API?** Không bắt buộc; async/callback không block khi chờ network, CPU-heavy parsing cần executor phù hợp, UI MainActor.

**Q: Transport error vs HTTP error?** Network/TLS/timeout là transport; HTTP status 4xx/5xx vẫn có response, cần validate; business/decode errors riêng.

**Q: Codable vs SwiftyJSON?** Codable type-safe schema, SwiftyJSON thuận tiện dynamic JSON; validate data và mapping DTO-domain.

**Q: Khi nào chọn Moya/Alamofire?** Alamofire request/interceptor, Moya typed endpoint/provider/stub; URLSession native đủ nhiều use cases.

**Q: API retry và cache thế nào?** Bounded retry theo idempotency/error, URLCache HTTP semantics và local cache nghiệp vụ, không cache token/PII tùy ý.
