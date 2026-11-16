# Speed Test: Download, Upload và Latency

### Định nghĩa

Throughput là lượng dữ liệu thực truyền mỗi giây; bandwidth là năng lực đường truyền. Mbps = bytes * 8 / elapsedSeconds / 1_000_000. MB/s khác Mbps (8 lần). Latency là thời gian khứ hồi; ICMP RTT khác TCP connect time/HTTP latency/TTFB. Jitter là biến thiên latency; packet loss là tỷ lệ packet mất.

### Cơ chế đo

1. Chọn endpoint đủ năng lực, khu vực phù hợp, ghi interface/path, warm-up và cache policy; không đo dữ liệu cache.
2. Download: URLSession data delegate đếm bytes và monotonic time, đo riêng warm-up; có thể nhiều stream giới hạn để saturate link; không giữ cả payload trong RAM.
3. Upload: uploadTask gửi payload không dễ nén, đếm progress và xác nhận server thực nhận.
4. Latency: nhiều request nhỏ, median/p95; ghi rõ HTTP RTT hay ICMP. URLSessionTaskMetrics tách DNS/TCP/TLS, chú ý reuse connection.
5. Kết thúc theo duration/byte cap, cancel task/session, báo error; VPN trước/sau cùng endpoint nhưng không khẳng định chỉ VPN gây chênh lệch.

```swift
import Foundation

func megabitsPerSecond(bytes: Int64, elapsed: TimeInterval) -> Double? {
    guard bytes >= 0, elapsed > 0 else { return nil }
    return Double(bytes) * 8 / elapsed / 1_000_000
}

final class DownloadCounter: NSObject, URLSessionDataDelegate {
    private var bytes: Int64 = 0
    private let started = ProcessInfo.processInfo.systemUptime

    func urlSession(_ session: URLSession, dataTask: URLSessionDataTask,
                    didReceive data: Data) {
        bytes += Int64(data.count)
        let elapsed = ProcessInfo.processInfo.systemUptime - started
        if let speed = megabitsPerSecond(bytes: bytes, elapsed: elapsed) {
            print(speed) // Minh họa; throttle UI sample trong production.
        }
    }
}
```

Một counter cho một lần đo, tạo ngay trước resume, delegate queue serial, kiểm tra status/didCompleteWithError. data.count là bytes delegate nhận, không nhất thiết wire bytes. Monotonic clock tránh wall-clock thay đổi.

### Edge cases

* Wi-Fi/cellular handover hoặc VPN đổi trạng thái: đánh dấu kết quả không đồng nhất, cancel/restart theo lựa chọn user.
* Run ID tránh kết quả cũ ghi đè test mới; timeout, captive portal, no network, low data mode.
* Speed test tốn data/pin, thông báo trước cellular, giới hạn duration/bytes.
* Test math với fake clock/transport, 0 duration, interrupted transfer; đo device thật.

### Q&A

**Q: Download 10 MB trong 2 giây?** 40 Mbps (MB thập phân).

**Q: HTTP ping giống ICMP?** Không, HTTP gồm overhead protocol/server tùy phương pháp.

**Q: Tại sao speed không ổn định?** Endpoint, radio, congestion, TCP ramp-up, TLS, VPN route và sampling. Đo nhiều lần, báo phương pháp.

**Liên hệ:** FastVPN download/upload/ping.

## Speed Test: triển khai và kiểm tra chi tiết

### Sampling và warm-up

Instant speed = bytes tăng trong sample / duration sample. Average = tổng measured bytes / measured duration. Không lấy trung bình cộng speed sample có duration khác nhau. Dùng monotonic clock, loại warm-up theo phương pháp công bố. Nếu nhiều stream đồng thời, cộng bytes rồi chia cùng wall-clock elapsed, không cộng duration mỗi stream.

```swift
import Foundation

struct TransferSample {
    let bytes: Int64
    let seconds: TimeInterval
}

func averageMbps(_ samples: [TransferSample]) -> Double? {
    guard samples.allSatisfy({ $0.bytes >= 0 && $0.seconds > 0 }) else {
        return nil
    }
    return megabitsPerSecond(bytes: samples.reduce(0) { $0 + $1.bytes },
                             elapsed: samples.reduce(0) { $0 + $1.seconds })
}
```

Ví dụ samples nối tiếp; production giới hạn byte cap tránh overflow. Server đủ bandwidth, cache policy phù hợp, tránh response nén làm sai interpretation; application bytes khác wire bytes. Endpoint và protocol phải ghi rõ.

### Upload measurement

```swift
final class UploadProgress: NSObject, URLSessionTaskDelegate {
    private let started = ProcessInfo.processInfo.systemUptime

    func urlSession(_ session: URLSession, task: URLSessionTask,
                    didSendBodyData bytesSent: Int64,
                    totalBytesSent: Int64,
                    totalBytesExpectedToSend: Int64) {
        let elapsed = ProcessInfo.processInfo.systemUptime - started
        if let mbps = megabitsPerSecond(bytes: totalBytesSent, elapsed: elapsed) {
            print(mbps)
        }
    }
}
```

Upload progress đo bytes client gửi, không chắc server nhận/commit; validate HTTP status và server response. Dùng file/stream thay vì allocate hàng trăm MB RAM, payload fixture không chứa dữ liệu cá nhân. Download đếm chunks bằng delegate hoặc async bytes, không giữ toàn bộ payload. Hủy task/session và dọn temporary file khi cancel.

### HTTP latency

```swift
func httpLatency(url: URL) async throws -> TimeInterval {
    var request = URLRequest(url: url)
    request.httpMethod = "HEAD"
    request.cachePolicy = .reloadIgnoringLocalCacheData
    let start = ProcessInfo.processInfo.systemUptime
    let (_, response) = try await URLSession.shared.data(for: request)
    guard let http = response as? HTTPURLResponse,
          (200..<300).contains(http.statusCode) else {
        throw URLError(.badServerResponse)
    }
    return ProcessInfo.processInfo.systemUptime - start
}
```

Endpoint phải hỗ trợ HEAD. HTTP elapsed có thể gồm DNS/TCP/TLS/server overhead, không phải ICMP RTT. URLSessionTaskMetrics tách phase DNS/connect/TLS/request/response, phase có thể nil khi reuse connection. Đo nhiều samples, median/p95; jitter là variation theo phương pháp, HTTP failure không đồng nghĩa packet loss.

### State machine và UX

Idle → latency → download → upload → result; error/cancel bất kỳ bước. Run ID chặn callback cũ ghi đè run mới; throttle progress, UI main actor. Path/VPN đổi giữa test: báo mixed hoặc restart theo policy. Cảnh báo data usage, giới hạn duration/bytes, xử lý Low Data Mode. Test fake clock/transport, interrupted connection, zero duration, stale callback.

**Q: 5 stream truyền tổng 100 MB trong 10s?** 80 Mbps nếu MB thập phân, không chia cho 50s.

**Q: Tại sao kết quả đo không bằng tốc độ gói mạng?** Radio, congestion, TCP ramp-up, endpoint capacity, protocol overhead, VPN route và sampling.

**Bài tập:** Tạo state reducer và mock delegate, test 10 MB/2s, samples unequal duration, cancel và path change.
