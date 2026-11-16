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

## CoreLocation chuyên sâu

### Permission, accuracy và lifecycle

Authorization WhenInUse/Always khác accuracy full/reduced. Không xin Always ngay khi app chỉ cần nearby. Info.plist purpose string rõ; denied hướng dẫn Settings và fallback manual search. iOS background location có điều kiện capability/authorization và system policy; stop khi không cần. CLLocation có timestamp, horizontalAccuracy; accuracy âm là invalid.

```swift
import CoreLocation

func isUsable(_ location: CLLocation,
              maxAge: TimeInterval = 30,
              maxAccuracy: CLLocationAccuracy = 100) -> Bool {
    location.horizontalAccuracy >= 0 &&
    location.horizontalAccuracy <= maxAccuracy &&
    abs(location.timestamp.timeIntervalSinceNow) <= maxAge
}

func distance(from a: CLLocationCoordinate2D,
              to b: CLLocationCoordinate2D) -> CLLocationDistance {
    CLLocation(latitude: a.latitude, longitude: a.longitude)
        .distance(from: CLLocation(latitude: b.latitude, longitude: b.longitude))
}
```

distance là đường thẳng địa lý, không phải road distance/ETA. desiredAccuracy và distanceFilter là hint, không bảo đảm callback đúng interval. requestLocation một lần; startUpdatingLocation liên tục; significant-change/region monitoring phù hợp nhu cầu tiết kiệm pin tùy OS và permission. CLGeocoder reverseGeocodeLocation có async/callback; tránh gọi mỗi location update, cache/debounce.

### Map rendering và viewport

```text
Camera gesture -> camera idle/debounce -> viewport bounds
-> search API -> map DTO -> diff marker IDs -> render
```

Camera center/zoom/bearing/tilt; marker là point, polyline route là chuỗi tọa độ, polygon area. Clustering theo zoom/viewport giảm marker. Không render toàn bộ dataset nếu chỉ thấy một vùng. Map4D SDK API phụ thuộc version, dùng adapter để domain không phụ thuộc SDK. Route API có start/end/waypoints, travel mode, traffic; ETA là estimate. Cache theo tile/viewport/version, tôn trọng license.

### Q&A bổ sung

**Q: Accuracy tốt nhất có nên dùng luôn?** Không, tốn pin; chọn theo feature, stopUpdates khi không cần.

**Q: Geofence và push notification khác nhau?** Region monitoring phát event vị trí theo hệ thống; push là server gửi APNs. Có thể kết hợp nhưng không đảm bảo thời điểm chính xác.

**Q: Map search sai sau pan nhanh?** Cancel/ignore stale result theo query generation; debounce camera idle, diff marker ID.

**Bài tập:** Mock location provider, test denied/reduced/stale, tính khoảng cách 2 điểm, debounce viewport search.

## Câu hỏi phỏng vấn tình huống

**Q: CLLocation timestamp cũ và accuracy âm?** Reject invalid accuracy, kiểm tra age/max accuracy theo feature; không update camera từ stale sample.

**Q: distance(from:) có là khoảng cách đường đi?** Không, geodesic straight-line; routing server/SDK tính road network, ETA tùy traffic.

**Q: Background location có luôn chạy nếu Always?** Không, system policy/capability/configuration/permission và feature-specific behavior; tiết kiệm pin, stop khi không cần.
