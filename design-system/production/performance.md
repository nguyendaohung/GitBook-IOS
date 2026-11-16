# Instruments và Production Debugging

### Định nghĩa

Performance optimization dựa trên đo lường, không chỉ cảm giác. Main thread xử lý UI; long synchronous task, decode ảnh lớn, layout liên tục và I/O gây hitch/hang. Memory leak khác memory spike và cache hợp lệ.

| Công cụ | Mục đích |
| --- | --- |
| Time Profiler | CPU hotspot, call stack |
| Allocations / Leaks | Allocation growth, leak |
| Memory Graph | Retain cycle, object graph |
| Core Animation / Animation Hitches | Frame hitch, rendering |
| Network / URLSessionTaskMetrics | Request timing, transfer |
| Thread Sanitizer | Data race trong môi trường hỗ trợ |
| Main Thread Checker | Vi phạm UIKit main thread |

### Quy trình

1. Reproduce trên build/device tương tự production, ghi OS/version, network, steps.
2. Đo baseline (launch time, scrolling hitch, memory peak, API p95, crash rate).
3. Xác định bottleneck bằng trace; sửa một nguyên nhân và đo lại.
4. Regression test, rollout có kiểm soát, theo dõi metrics. Không đưa số liệu cải thiện vào CV khi chưa đo.

```swift
import os

let logger = Logger(subsystem: "com.example.app", category: "network")

func logRequest(status: Int) {
    logger.info("HTTP status: \(status, privacy: .public)")
}
```

Không log access token, SMS, số điện thoại hay dữ liệu thanh toán. Crash report cần symbolication bằng dSYM đúng build UUID; xem exception type, crashed thread, stack và breadcrumbs đã redact. MetricKit có diagnostic/metric payload tùy OS. Dùng breakpoint symbolic, LLDB po, view debugger, network inspection với dữ liệu test.

### Q&A

**Q: Leak vs memory spike?** Leak object không giải phóng dù không cần; spike do tải/decode/tạm thời; đo lifecycle và allocations.

**Q: News feed lag?** Profile main thread, image downsample/cache, reuse, batch updates; không tối ưu vô căn cứ.

**Q: Crash production không reproduce?** Symbolicate, phân nhóm OS/version/device, tìm pattern, thêm log an toàn, tạo regression test.

**Liên hệ:** BIDU scrolling/Texture, Watch Faces ảnh lớn, Map4D rendering, Call ID extension memory, FastVPN test transfer.
