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

## Network và VPN chuyên sâu

### TCP, UDP, TLS và DNS

TCP là byte stream tin cậy, có connection, ordering, retransmission và congestion control; không bảo toàn message boundary. UDP gửi datagram, không bảo đảm giao/thứ tự; ứng dụng hoặc protocol bên trên tự xử lý loss. TLS mã hóa và xác thực peer trên transport phù hợp. DNS phân giải hostname; NWPath satisfied không chứng minh DNS hoặc server hoạt động. VPN định tuyến traffic qua tunnel, có thể full/split tunnel tùy policy.

```swift
import Network

final class TCPProbe {
    private let queue = DispatchQueue(label: "tcp.probe")
    private var connection: NWConnection?

    func start(host: String, port: UInt16) {
        guard let port = NWEndpoint.Port(rawValue: port) else { return }
        let connection = NWConnection(host: NWEndpoint.Host(host),
                                      port: port, using: .tcp)
        self.connection = connection
        connection.stateUpdateHandler = { state in
            switch state {
            case .ready: print("TCP connected")
            case .waiting(let error): print(error)
            case .failed(let error): print(error)
            default: break
            }
        }
        connection.start(queue: queue)
    }

    func stop() {
        connection?.cancel()
        connection = nil
    }
}
```

Đây là TCP connect probe, không phải HTTP latency hay ICMP ping. Khi gửi/nhận, dùng send/receive; TCP framing cần length prefix/delimiter. Callback không mặc định main thread. .ready chỉ chứng minh connection tương ứng ready.

### Personal VPN vs Packet Tunnel

| Thành phần | Personal VPN | Custom Packet Tunnel |
| --- | --- | --- |
| Manager | NEVPNManager | NETunnelProviderManager |
| Protocol | IKEv2/IPsec | NETunnelProviderProtocol |
| Engine | System | Provider + engine/SDK |
| Extension | Không cần custom packet provider | NEPacketTunnelProvider |
| Packet | System xử lý | packetFlow đọc/ghi IP packets |

Flow: load preferences → cấu hình protocol/credential → save (system approval) → load lại → start → observe NEVPNStatusDidChange → stop. start thành công không có nghĩa connected. Không start song song khi connecting; dùng state machine và timeout. Credential lưu Keychain/reference theo API, không nhúng key/password vào UserDefaults.

```swift
import NetworkExtension

final class PacketTunnelProvider: NEPacketTunnelProvider {
    override func startTunnel(options: [String: NSObject]?,
                              completionHandler: @escaping (Error?) -> Void) {
        // Production: negotiate remote engine, then set network settings
        // using assigned address, routes, DNS and MTU; start packet loop.
        completionHandler(NSError(domain: "ExampleTunnel", code: 1))
    }

    override func stopTunnel(with reason: NEProviderStopReason,
                             completionHandler: @escaping () -> Void) {
        // Stop engine, timers, packet loop and release sockets.
        completionHandler()
    }
}
```

Ví dụ khung provider cố ý trả lỗi vì không có engine/server/key negotiation. Không phải VPN chạy được. Provider dùng setTunnelNetworkSettings, NEIPv4Settings/NEIPv6Settings, DNS, MTU và included/excluded routes; packetFlow đọc/ghi IP packets với engine. WireGuard/OpenVPN cần engine phù hợp; không tự viết crypto. Kiểm tra IPv6, DNS leak và route loop theo threat model.

### Test và Q&A

Test permission cancel, config invalid, server unreachable, Wi-Fi→cellular, airplane mode, app kill/relaunch, stop lúc connecting, reconnect, IPv6/DNS. Test device thật và entitlement. Không log nội dung traffic.

**Q: NWPathMonitor vs VPN status?** Path phản ánh network route khả dụng; NEVPNStatus phản ánh VPN session. Cả hai không chứng minh endpoint hoạt động.

**Q: VPN app bị kill thì tunnel có tắt?** Provider do system quản lý, tùy configuration/OS; không giả định main app process luôn sống.

**Q: HTTPS có cần khi dùng VPN?** Có. VPN bảo vệ segment tới tunnel endpoint; HTTPS bảo vệ traffic ứng dụng tới origin.

**Bài tập:** Tạo fake VPN state reducer, test duplicate tap, permission denial, timeout, handover và reconnect.

## Câu hỏi phỏng vấn tình huống

**Q: VPN connected nhưng API timeout?** Kiểm tra actual request, DNS/routes/MTU/TLS/server, path/NEVPNStatus không đảm bảo endpoint reachable.

**Q: Packet Tunnel provider startTunnel completion success lúc nào?** Sau khi engine/settings/tunnel thật sự sẵn sàng theo contract; skeleton không thay VPN engine.

**Q: TCP vs UDP framing?** TCP byte stream cần framing/length prefix, UDP datagram giữ boundary nhưng có loss/reorder; protocol VPN quyết định.
