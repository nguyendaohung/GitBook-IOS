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
