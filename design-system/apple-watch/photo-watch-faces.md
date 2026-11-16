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

## PhotoKit, image pipeline và watchOS chuyên sâu

### PhotoKit authorization và fetch

```swift
import Photos

func recentPhotos() async -> PHFetchResult<PHAsset>? {
    let status = await PHPhotoLibrary.requestAuthorization(for: .readWrite)
    guard status == .authorized || status == .limited else { return nil }
    let options = PHFetchOptions()
    options.sortDescriptors = [NSSortDescriptor(key: "creationDate", ascending: false)]
    options.fetchLimit = 100
    return PHAsset.fetchAssets(with: .image, options: options)
}
```

PhotoKit cần usage description, limited chỉ cho subset user chọn. Nếu chỉ chọn ảnh, PHPicker/PhotosPicker không cần full library access. PHAsset là metadata/reference, không phải UIImage. Asset iCloud có thể chưa local; isNetworkAccessAllowed và progressHandler tùy nhu cầu.

```swift
import Photos
import UIKit

func requestThumbnail(asset: PHAsset, size: CGSize,
                      completion: @escaping (UIImage?) -> Void) -> PHImageRequestID {
    let options = PHImageRequestOptions()
    options.deliveryMode = .opportunistic
    options.resizeMode = .fast
    options.isNetworkAccessAllowed = false
    return PHImageManager.default().requestImage(
        for: asset, targetSize: size, contentMode: .aspectFill, options: options
    ) { image, info in
        if info?[PHImageCancelledKey] as? Bool == true { return }
        // May receive degraded and final; check PHImageResultIsDegradedKey.
        completion(image)
    }
}
```

size tính pixel, không phải point. Cell giữ request ID, cancelImageRequest khi reuse, check asset.localIdentifier trước apply. PHCachingImageManager preheat vùng gần visible, stopCachingImages khi scroll xa.

### SwiftUI PhotosPicker (iOS 16+)

```swift
import SwiftUI
import PhotosUI

struct PickPhotoView: View {
    @State private var selection: PhotosPickerItem?
    @State private var image: UIImage?

    var body: some View {
        VStack {
            PhotosPicker("Choose photo", selection: $selection, matching: .images)
            if let image {
                Image(uiImage: image).resizable().scaledToFit()
            }
        }
        .task(id: selection) {
            guard let selection else { return }
            guard let data = try? await selection.loadTransferable(type: Data.self),
                  !Task.isCancelled else { return }
            image = UIImage(data: data)
        }
    }
}
```

Ví dụ đơn giản cho ảnh nhỏ; production dùng file representation/custom Transferable và ImageIO downsample, tránh load full-size Data/UIImage. Xử lý loading/error, cancel khi selection đổi. Không upload ảnh nếu không cần.

### Watch face vs complication vs watch app

| Khái niệm | Vai trò |
| --- | --- |
| Watch face | Mặt đồng hồ system cung cấp, tùy chỉnh trong giới hạn Apple |
| .watchface | File chia sẻ mặt đồng hồ hợp lệ, system import |
| Complication | Nội dung nhỏ trong slot hỗ trợ, WidgetKit timeline |
| watchOS app | UI app riêng, không phải native system face |

CLKWatchFaceLibrary import file hợp lệ tùy OS/device, user/system kiểm soát cài đặt; không tự ghi đè active face tùy ý. WidgetKit complication timeline entries và reload budget do system quyết định; không update mỗi giây tùy ý. ClockKit legacy và WidgetKit availability khác nhau.

**Q: Ảnh 12 megapixel RAM?** RGBA decode khoảng 48 MB, chưa tính buffers; downsample trước display.

**Q: Có tạo third-party native face hoàn chỉnh?** Không; chia sẻ/import face hỗ trợ và complication, system giới hạn.

**Bài tập:** Picker + thumbnail, test iCloud/offline/limited/cancel, gallery memory; tạo WidgetKit complication demo trên watchOS hỗ trợ.

## Câu hỏi phỏng vấn tình huống

**Q: PHAsset vs UIImage?** PHAsset metadata/identifier của library, UIImage decoded image object; fetch targetSize theo pixel, iCloud/limited permission.

**Q: PHPicker result có thể dùng file URL lâu dài?** ItemProvider temporary file phải copy khi callback còn hiệu lực, validate type/size, cleanup. Không assume toàn bộ photo library quyền truy cập.

**Q: WidgetKit complication có update mỗi giây?** Timeline/reload budget do system quản lý; không bảo đảm arbitrary refresh. Face installation và complication là API khác nhau.
