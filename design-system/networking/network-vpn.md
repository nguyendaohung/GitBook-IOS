# Network Framework và NetworkExtension (VPN)

### Định nghĩa

Network.framework cung cấp NWPathMonitor, NWConnection (TCP/UDP/TLS), NWListener. URLSession làm việc ở tầng HTTP; NetworkExtension cho system-managed VPN và network extension. Network.framework không tự tạo VPN.

```swift
import Network

final class NetworkMonitor {
    private let monitor = NWPathMonitor()
    private let queue = DispatchQueue(label: "network.monitor")

    func start(onChange: @escaping @Sendable (Bool, Bool) -> Void) {
        monitor.pathUpdateHandler = { path in
            onChange(path.status == .satisfied, path.isExpensive)
        }
        monitor.start(queue: queue)
    }

    func stop() { monitor.cancel() }
}
```

NWPath có status, usesInterfaceType(.wifi/.cellular), isExpensive, isConstrained; satisfied không chứng minh endpoint truy cập được. Update UI trên main actor. Monitor đã cancel không restart. NWConnection dùng start(queue:), stateUpdateHandler, send/receive, cancel; TCP stream không bảo toàn message boundary, UDP không đảm bảo delivery/order.

### VPN flow

| API | Vai trò |
| --- | --- |
| NEVPNManager | Personal VPN dùng IKEv2/IPsec |
| NETunnelProviderManager | Cấu hình custom packet tunnel |
| NEPacketTunnelProvider | Extension xử lý packet, thiết lập network settings |
| NEVPNConnection | startVPNTunnel, stopVPNTunnel, status |

1. Bật capability/entitlement theo target và protocol thực tế.
2. loadFromPreferences (loadAllFromPreferences với tunnel provider), cấu hình server, credential reference, enabled.
3. saveToPreferences, load lại nếu cần, user đồng ý thêm VPN configuration.
4. startVPNTunnel; theo dõi NEVPNStatusDidChange: invalid/disconnected/connecting/connected/reasserting/disconnecting.
5. stopVPNTunnel khi user yêu cầu; xử lý permission denied, stale config, network handover, provider fail. Start thành công không đảm bảo connected.

```swift
import NetworkExtension

func connectConfiguredVPN() async throws {
    let manager = NEVPNManager.shared()
    try await manager.loadFromPreferences()
    guard manager.isEnabled else { throw VPNError.notConfigured }
    try manager.connection.startVPNTunnel()
}

enum VPNError: Error { case notConfigured }
```

Ví dụ Personal VPN đã cấu hình. Với custom tunnel, provider startTunnel/stopTunnel, setTunnelNetworkSettings và packetFlow; engine đảm nhiệm encryption/authentication, DNS/route/transport. Không tự viết crypto. WireGuard/OpenVPN thường cần engine/SDK + packet tunnel. Giao thức FastVPN chưa được ghi trong CV.

### CoreTelephony / Cellular

CTTelephonyNetworkInfo.serviceCurrentRadioAccessTechnology có thể trả radio technology (LTE, NR...), tùy OS/hardware; không đảm bảo carrier data, signal strength hay tốc độ. Dùng NWPath để biết interface; tránh private API, hiển thị Unknown khi không có dữ liệu.

### Q&A

**Q: VPN khác HTTPS?** VPN tunnel bảo vệ traffic trong phạm vi route đến VPN endpoint; HTTPS mã hóa ứng dụng đến origin. VPN không thay thế HTTPS.

**Q: VPN switch Wi-Fi → 5G?** Theo dõi path/status, engine reconnect/reasserting, backoff, không start song song.

**Q: App bị kill thì VPN sao?** Provider do system quản lý, tùy config/on-demand/OS; không giả định app process luôn chạy.

**Liên hệ:** FastVPN – kết nối một chạm, permission/state, cellular information. Xác nhận engine/protocol thực tế trước khi ghi vào CV.

[Network](https://developer.apple.com/documentation/network) · [Network Extension](https://developer.apple.com/documentation/networkextension)
