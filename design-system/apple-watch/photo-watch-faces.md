# PhotoKit, PhotosUI và Watch Faces

### PhotoKit / PhotosUI

PhotoKit truy cập PHAsset/PHPhotoLibrary, PHImageManager lấy ảnh theo targetSize; quyền readWrite/limited/denied, addOnly nếu chỉ ghi. PhotosUI PHPickerViewController (iOS 14+) hoặc PhotosPicker (SwiftUI iOS 16+) cho chọn ảnh không cấp quyền toàn bộ library. iCloud asset có thể cần network/progress.

```swift
import PhotosUI

func makePicker(delegate: PHPickerViewControllerDelegate) -> PHPickerViewController {
    var configuration = PHPickerConfiguration()
    configuration.filter = .images
    configuration.selectionLimit = 1
    let picker = PHPickerViewController(configuration: configuration)
    picker.delegate = delegate
    return picker
}
```

Delegate nhận PHPickerResult.itemProvider, loadObject/loadFileRepresentation; copy temporary file trong callback, UI main actor. PhotoKit cần usage description, request permission đúng lúc; limited không phải denied. PHImageManager có thể callback degraded + final, cancel request khi reuse.

### Xử lý ảnh

```swift
import ImageIO
import UIKit

func thumbnail(url: URL, maxPixelSize: Int) -> UIImage? {
    guard let source = CGImageSourceCreateWithURL(url as CFURL, nil) else {
        return nil
    }
    let options: [CFString: Any] = [
        kCGImageSourceCreateThumbnailFromImageAlways: true,
        kCGImageSourceCreateThumbnailWithTransform: true,
        kCGImageSourceThumbnailMaxPixelSize: maxPixelSize,
        kCGImageSourceShouldCacheImmediately: true
    ]
    guard let image = CGImageSourceCreateThumbnailAtIndex(
        source, 0, options as CFDictionary
    ) else { return nil }
    return UIImage(cgImage: image)
}
```

Downsample theo pixel size, orientation/crop phù hợp watch model, tránh decode full-size. Background processing, cache theo ID/version/size, cancel reuse, xóa temp file; không upload ảnh khi không cần.

### Watch Face Installation

* ClockKit CLKWatchFaceLibrary.addWatchFace(at:completionHandler:) thêm .watchface hợp lệ theo system flow/compatibility; không tự ghi đè active face tùy ý.
* Photos-based face: hướng dẫn system Photos/watch face sharing flow; không có public API tự tạo third-party native face engine.
* WidgetKit complications cung cấp timeline/content trong slot hỗ trợ; ClockKit complications là API cũ. Widget khác watch face hoàn chỉnh.
* Validate file, model/watchOS support, user confirmation, install failure. Cơ chế Watch Faces trong CV cần đối chiếu source/version.

### Q&A

**Q: PhotosUI vs PhotoKit?** Picker quyền tối thiểu, user chọn; PhotoKit quản lý/query library với authorization.

**Q: 1000 ảnh?** Pagination, downsampling, cache limit, bounded prefetch, cancel, stable ID.

**Q: Có tạo mặt đồng hồ tùy ý?** Public API chỉ hỗ trợ sharing/install/customization trong giới hạn system.

**Liên hệ:** Watch Faces – photo customization, gallery, preview/install.

[PhotoKit](https://developer.apple.com/documentation/photokit) · [ClockKit](https://developer.apple.com/documentation/clockkit) · [WidgetKit](https://developer.apple.com/documentation/widgetkit)
