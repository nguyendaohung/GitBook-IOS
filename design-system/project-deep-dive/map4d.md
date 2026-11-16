# Map4D – Project Deep Dive

### CV

iOS Developer: phát hành phiên bản App Store, tích hợp Map4D SDK, tối ưu performance, phối hợp backend/UI/UX/PM. RxSwift, SnapKit, CoreData, CoreLocation, Alamofire, Map4D SDK, MVVM + Clean Architecture.

### Architecture tham khảo

View (camera/search events) → ViewModel (Rx input/output) → UseCase → Repository protocol → Data (Alamofire, CoreData, Map SDK adapter). Domain không phụ thuộc SDK; DI ở composition root. Search query debounce/distinct → flatMapLatest → state loading/content/error → render main; cache theo key/version. [RxSwift + Clean Architecture](../rxswift-+-mvvm-+-clean-architure.md).

### Các vấn đề thực tế cần hiểu

* Camera pan tạo nhiều request: debounce/idle viewport, cancel/ignore old request, marker clustering, cache.
* CoreLocation permission denied/reduced accuracy, stale coordinate, pin; [CoreLocation](../ios-system-frameworks/core-location.md).
* CoreData context queue confinement, background import, merge viewContext; [Core Data](../data-stroage/core-data.md).
* SnapKit constraint update, Map SDK camera/marker/polyline/route. Không đoán API Map4D cụ thể.

### Q&A

**Q: flatMapLatest vs flatMap?** Latest bỏ subscription cũ, phù hợp search; flatMap giữ nhiều inner stream.

**Q: Clean Architecture dependency hướng nào?** Source dependency hướng vào domain abstraction, Data implement protocol.

**Q: Tối ưu map thế nào?** Đo renderer/main CPU và memory, giảm cập nhật marker/layout, viewport query, cache; nêu số liệu chỉ nếu đã đo.

**Chuẩn bị:** Vẽ kiến trúc thật, version SDK, API đã tích hợp, schema local, cách test/search và một bug production thật.
