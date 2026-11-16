# FastVPN – Project Deep Dive

### CV

Speed test download/upload/ping, one-tap VPN, permission/state, cellular information, premium subscription, local notification, responsive UI/localization, production fixes. Swift, UIKit, URLSession, Network Framework, StoreKit, UserNotifications, REST, Auto Layout, ads/analytics.

### Flow tham khảo

User start VPN → load config → permission/save → start → observe status → connected/error → stop. Speed test state idle → preparing → latency → download → upload → completed/cancelled/error. Run ID và cancellation tránh stale update. [VPN](../networking/network-vpn.md) · [Speed Test](../networking/speed-test.md).

### Trade-off

* HTTP latency vs ICMP, endpoint placement, warm-up, bytes/time monotonic, bounded streams, data cap.
* NWPathMonitor là hint, request thực tế mới xác nhận connectivity; Wi-Fi/cellular handover invalidate test hoặc báo rõ.
* VPN engine/protocol không có trong CV: xác nhận SDK, Personal VPN/custom tunnel, entitlement, reconnect/on-demand.
* Premium verified entitlement, ads state, permission/local notification, cellular API limitations.

### Q&A

**Q: Speed 10 MB/2s?** 40 Mbps; phân biệt MB/MiB và wire/application bytes.

**Q: Start VPN thành công nhưng chưa connected?** start chỉ yêu cầu; observe NEVPNStatus, timeout/error và user permission.

**Q: Stop speed test?** cancel URLSession tasks, check cancellation/run ID, release delegate/session, reset UI.

**Chuẩn bị:** Endpoint/method đo thực tế, protocol/SDK, lifecycle, state diagram, số liệu đo và production bug thật. Không khẳng định NetworkExtension implementation cụ thể khi chưa kiểm tra source.
