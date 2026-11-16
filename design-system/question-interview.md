# 😚 QUESTION INTERVIEW

**Link Question tham khảo:** [**https://fxstudio.dev/ios-swift-tuyen-tap-cac-cau-hoi-cho-phong-van/**](https://fxstudio.dev/ios-swift-tuyen-tap-cac-cau-hoi-cho-phong-van/)



Q: Xử lý đăng nhập FaceID với IOS?\
\- Không ưu tiên lưu password; lưu token có thể revoke trong Keychain và dùng LocalAuthentication/access control.\
\- S2: khi đăng nhập sever cấp cho 1 token sau đó lưu xuống keychain Access lần sau vào lại lấy token đó gửi lên sever -> match thì cho login

**Q1: Xử lý khi thực hiện 1 task nặng dưới background thread Swift ?**

\=>  Trong lập trình iOS, việc thực hiện các task nặng trong background thread là rất quan trọng để tránh gây tràn bộ nhớ hoặc gián đoạn các task khác trong app. Để thực hiện điều này, bạn có thể sử dụng Grand Central Dispatch (GCD) hoặc Operation Queue trong Swift.

1. Sử dụng Grand Central Dispatch (GCD) :

* Tạo một queue background với DispatchQueue.global().async
* Thực hiện task trong closure của async
* Sử dụng DispatchQueue.main.async để trả kết quả về main thread.

<figure><img src=".gitbook/assets/Screen Shot 2023-02-10 at 09.55.05.png" alt=""><figcaption><p>USE GCD</p></figcaption></figure>

2. Sử dụng Operation Queue :

* Tạo một OperationQueue với qualityOfService cài đặt là .background
* Tạo một Operation với task cần thực hiện
* Thêm Operation vào OperationQueue; queue tự start, không gọi start() thêm.

<figure><img src=".gitbook/assets/Screen Shot 2023-02-10 at 09.56.04.png" alt=""><figcaption><p>USE OPERATION </p></figcaption></figure>

**Q2: Nếu muốn download 1000 ảnh dưới background thread trong swift ios thì xử lý như thế nào ?**

* Để download 1000 ảnh dưới background thread trong Swift iOS, bạn có thể sử dụng Operation Queue để quản lý việc download. Bạn có thể tạo một Operation cho mỗi hình ảnh cần tải xuống và thêm chúng vào Operation Queue. Bạn có thể xác định số lượng Operation có thể chạy song song và số lượng Operation đang chờ trong hàng đợi để đảm bảo tốc độ tải xuống hợp lý và tránh quá tải cho máy.

<figure><img src=".gitbook/assets/Screen Shot 2023-02-10 at 09.58.10.png" alt=""><figcaption></figcaption></figure>

Trong ví dụ trên, chúng ta tạo một OperationQueue và xác định số lượng operation có thể chạy song song là 10. Sau đó, chúng ta tạo một BlockOperation cho mỗi hình ảnh cần tải xuống và thêm chúng vào OperationQueue. Các Operation sẽ được thực hiện song song như đã xác định và khi một operation hoàn thành, operation khác sẽ được thực hiện tiếp theo trong hàng đợi.

* Cũng có thể sử dụng [Dispatch Semaphore ](multi-thread/dispatchsemaphore.md)để giới hạn số lượng tải xuống đồng thời trong trường hợp này. Dispatch Semaphore là một công cụ để giới hạn số lượng tài nguyên đang được sử dụng đồng thời.

<figure><img src=".gitbook/assets/Screen Shot 2023-02-10 at 10.04.57.png" alt=""><figcaption></figcaption></figure>

Trong ví dụ trên, chúng ta tạo một Dispatch Semaphore với giá trị ban đầu là 5. Điều này có nghĩa là chỉ có tối đa 5 hình ảnh được tải xuống đồng thời. Khi block code gọi semaphore.wait(), nó sẽ đợi cho đến khi một tài nguyên trở lại và gọi semaphore.signal(). Block code sẽ tiếp tục thực hiện khi semaphore.wait() trả về. Điều này giúp giới hạn số lượng tải xuống đồng thời và tránh việc tải quá nhiều dữ liệu tại một thời điểm dẫn đến việc giảm tốc độ hoặc gặp vấn đề bộ nhớ.

**Q3: Call Api ở luồng nào? Cách call nhiều API?**

* Thông thường, gọi **API** được thực hiện trong **background thread** (luồng nền), để tránh block main thread và giữ độ trơn tru của UI. Cập nhật UI thì thực hiện trong main thread.
* Khi gọi nhiều API cùng lúc, có thể sử dụng các kỹ thuật như **Dispatch Group**, **OperationQueue** hoặc **Promise**. Chọn phương pháp phù hợp với yêu cầu và mục tiêu của bạn.

**Q4: Cách tương tác API ?**&#x20;

Để gọi **API** trong Swift, có nhiều cách như sử dụng thư viện third-party như **Alamofire**, **AFNetworking** hoặc dùng các **API** của **Foundation** như **URLSession**.

Các bước call API:  Xem tại [call API](networking/)

**Q5: Vì sao main thread là serial queue ?**&#x20;

* **Main thread** (luồng chính) trong iOS là một serial queue (hàng đợi tuần tự), bởi vì nó cần phải tuần tự xử lý các tác vụ liên quan đến UI. Việc tuần tự hóa này đảm bảo rằng tác vụ nào được thực hiện trước và sau đó sẽ được xử lý một cách chính xác, giúp tránh xảy ra những vấn đề về đồng bộ hoá và tràn đầy bộ nhớ.

\=> Vì vậy, tất cả các tác vụ liên quan đến cập nhật UI như vẽ đồ họa, cập nhật giao diện, tương tác với người dùng, v.v. đều phải được thực hiện trên main thread. Các tác vụ khác như gọi API, xử lý dữ liệu, v.v. thì có thể được thực hiện trên **background thread.**

**Q4: Loại bỏ phần tử trùng lặp trong mảng dùng swift ?**

Ví dụ, giả sử bạn có một mảng chứa các phần tử có thể trùng lặp như sau:

<figure><img src=".gitbook/assets/Screen Shot 2023-02-23 at 15.05.15.png" alt=""><figcaption></figcaption></figure>

* Để loại bỏ các phần tử trùng lặp, bạn có thể sử dụng method `removeDuplicates()` như sau:

<figure><img src=".gitbook/assets/Screen Shot 2023-02-23 at 15.08.50.png" alt=""><figcaption></figcaption></figure>

Trong đó, `Set(numbers)` sẽ tạo ra một tập hợp chứa các phần tử duy nhất của mảng `numbers`, sau đó `Array(...)` sẽ chuyển đổi tập hợp này thành một mảng mới.

Sau khi thực hiện, mảng `numbers` sẽ chỉ chứa các phần tử duy nhất:

<figure><img src=".gitbook/assets/Screen Shot 2023-02-23 at 15.09.35.png" alt=""><figcaption></figcaption></figure>

Lưu ý rằng, khi sử dụng cách này để loại bỏ các phần tử trùng lặp, thứ tự của các phần tử trong mảng có thể bị thay đổi.

* Ngoài cách sử dụng `Set` và `Array` như đã giới thiệu ở trên, bạn có thể sử dụng một phương pháp khác để loại bỏ các phần tử trùng lặp trong một mảng bằng cách sử dụng vòng lặp `for` và kiểm tra xem một phần tử đã xuất hiện trong mảng chưa.

<figure><img src=".gitbook/assets/Screen Shot 2023-02-23 at 15.10.44.png" alt=""><figcaption></figcaption></figure>

Trong đoạn mã này, một mảng mới được khởi tạo với tên là `uniqueNumbers` để chứa các phần tử duy nhất. Trong vòng lặp `for`, mỗi phần tử trong mảng `numbers` được kiểm tra để xem nó đã xuất hiện trong mảng `uniqueNumbers` chưa. Nếu chưa xuất hiện, phần tử đó sẽ được thêm vào mảng `uniqueNumbers`.

Lưu ý rằng cách này có thể hiệu quả hơn khi xử lý các mảng nhỏ hoặc khi bạn cần duy trì thứ tự của các phần tử. Tuy nhiên, với các mảng lớn, cách sử dụng `Set` có thể nhanh hơn.

**Q6:  Tìm phần tử lớn thứ 2 trong mảng**

Để tìm phần tử lớn thứ hai trong một mảng Swift, có thể làm theo các bước sau:

1. Sắp xếp mảng theo thứ tự giảm dần sử dụng hàm `sorted()`.
2. Lấy phần tử ở vị trí thứ hai của mảng sử dụng toán tử subscript `[]`.

<figure><img src=".gitbook/assets/Screen Shot 2023-02-27 at 16.04.17.png" alt=""><figcaption></figcaption></figure>

Nếu mảng chứa ít hơn 2 phần tử, hoặc tất cả các phần tử giống nhau, thì phần tử lớn thứ hai không tồn tại và kết quả sẽ là nil.

## Bộ câu hỏi tổng hợp Middle/Senior – tình huống

**Q: Face ID login có lưu password?** Ưu tiên refresh token trong Keychain với access control phù hợp, LocalAuthentication xác minh local user, server kiểm tra session/revocation. Không lưu password chỉ để login lại; không gửi biometric data lên server.

**Q: Download 1000 ảnh giới hạn concurrent?** Bounded TaskGroup, async Operation lifecycle hoặc Rx merge(maxConcurrent:) cold source; downsample/cache, cancel reuse, không enqueue hàng nghìn blocking semaphore waits.

**Q: App API lỗi 401 nhiều request?** Single-flight refresh, replay mỗi request tối đa một lần, invalidate session khi refresh fail, không refresh loop.

**Q: SwiftUI @StateObject vs @ObservedObject?** Ownership/lifetime theo identity vs quan sát object được inject; @Observable iOS17 dùng Observation.

**Q: SwiftData vs Core Data?** SwiftData Swift macros/ModelContext/@Query iOS17+, Core Data context/model/tooling trưởng thành; chọn target và migration.

**Q: RxSwift flatMapLatest vs flatMapFirst?** Latest dispose inner cũ phù hợp search; first bỏ triggers khi inner active phù hợp submit; server side effect vẫn cần idempotency.

**Q: Rx retry(2) là hai lần retry?** Không, tối đa 2 attempts, một retry.

**Q: StoreKit transaction success có cấp premium ngay?** Verify, deliver idempotent, cập nhật entitlement, finish sau delivery; pending/unverified không grant.

**Q: VPN start call thành công là connected?** Không, observe NEVPNStatus, permission/server/timeout và provider readiness.

**Q: Call Directory reload fail vì sao?** Disabled, list trùng/không tăng dần, resource, version mismatch; validate/last-known-good.

**Q: WatchConnectivity sendMessage vs transferFile?** Interactive reachable vs queued file transfer; receiver copy temporary URL, không assume immediate.

**Q: Core Data/SwiftData model qua thread?** Không truyền managed object tùy ý, DTO/objectID/PersistentIdentifier và resolve context/actor đích.

**Q: Tìm second distinct maximum O(n)?** Duyệt một lần giữ largest và second, bỏ duplicate largest; nil nếu không đủ hai giá trị khác nhau.

```swift
func secondLargest(_ values: [Int]) -> Int? {
    var largest: Int?
    var second: Int?
    for value in values {
        if let max = largest {
            if value > max {
                second = max
                largest = value
            } else if value < max && (second.map { value > $0 } ?? true) {
                second = value
            }
        } else {
            largest = value
        }
    }
    return second
}
```

**Q: Dedup array giữ thứ tự O(n) expected?** Set seen, filter khi insert(...).inserted, không dùng Array(Set(array)) nếu cần order.

Đọc Q&A theo từng chương để có giải thích và ví dụ đầy đủ. Khi phỏng vấn trả lời định nghĩa → cơ chế → trade-off → ví dụ thực tế đã xác nhận, không tự gán số liệu/SDK chưa sử dụng.
