# CoreLocation và Map SDK

### Định nghĩa

CoreLocation lấy vị trí, heading, region monitoring. CLLocationManager delegate nhận location, authorization và error. GPS chỉ là một trong các nguồn; accuracy phụ thuộc môi trường, quyền và cấu hình.

```swift
import CoreLocation

final class LocationService: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyHundredMeters
    }

    func start() {
        manager.requestWhenInUseAuthorization()
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        switch manager.authorizationStatus {
        case .authorizedAlways, .authorizedWhenInUse:
            manager.requestLocation()
        case .denied, .restricted:
            // Hiển thị hướng dẫn hoặc cho phép tìm địa chỉ thủ công.
            break
        default:
            break
        }
    }

    func locationManager(_ manager: CLLocationManager,
                         didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last,
              location.horizontalAccuracy >= 0 else { return }
        // Kiểm tra timestamp/accuracy trước khi cập nhật camera.
    }

    func locationManager(_ manager: CLLocationManager,
                         didFailWithError error: Error) {
        // Phân biệt locationUnknown tạm thời và denied.
    }
}
```

Info.plist cần NSLocationWhenInUseUsageDescription; Always chỉ khi feature thực sự cần, có usage description và background capability phù hợp. Reduced accuracy (iOS 14+) có thể đủ nearby; request temporary full accuracy với purpose key nếu cần. Không bật background chỉ để gọi API liên tục.

### Map4D SDK / Map SDK

* Coordinate latitude/longitude (WGS84 thông thường, kiểm tra SDK contract), camera center/zoom/bearing/tilt; marker, polyline route, polygon, overlay.
* Geocoding: address → coordinate; reverse geocoding: coordinate → address; routing dùng road network, ETA tùy traffic/server, không chỉ khoảng cách đường chim bay.
* Camera idle/debounce/throttle search; cancel/ignore stale result, query theo viewport bounding box, marker clustering/reuse, cache tile/data theo SDK licensing.
* 2D/3D/4D là khả năng của platform, không suy ra cách renderer nội bộ từ CV. Xác minh tên API/version Map4D SDK đã dùng trước phỏng vấn.

### Q&A

**Q: Khi nào dùng requestLocation thay startUpdatingLocation?** Một lần lấy vị trí vs theo dõi liên tục; stop khi không cần để giảm pin.

**Q: User tắt precise location?** Chấp nhận approximate cho nearby hoặc giải thích lý do xin full accuracy đúng lúc.

**Q: Camera pan tạo nhiều request?** Debounce camera idle, distinct viewport, cancel request cũ, cache/dedup marker.

**Liên hệ:** Map4D, Karaoke nearby notification, DaPass gợi ý cafe.
