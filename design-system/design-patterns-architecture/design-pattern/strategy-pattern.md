# Strategy Pattern

### Định nghĩa

Strategy là behavioral pattern: đóng gói các thuật toán có cùng contract, thay đổi implementation khi runtime/composition mà caller không cần biết chi tiết. Khác State pattern: State thay behavior theo trạng thái nội bộ; Strategy chọn phương án xử lý. Không cần inheritance, dùng protocol + composition.

### Ví dụ image processing

```swift
import UIKit

protocol ImageResizeStrategy {
    func resize(_ image: UIImage, to size: CGSize) -> UIImage
}

struct FitResize: ImageResizeStrategy {
    func resize(_ image: UIImage, to size: CGSize) -> UIImage {
        let ratio = min(size.width / image.size.width,
                        size.height / image.size.height)
        let target = CGSize(width: image.size.width * ratio,
                            height: image.size.height * ratio)
        return UIGraphicsImageRenderer(size: target).image { _ in
            image.draw(in: CGRect(origin: .zero, size: target))
        }
    }
}

struct StretchResize: ImageResizeStrategy {
    func resize(_ image: UIImage, to size: CGSize) -> UIImage {
        UIGraphicsImageRenderer(size: size).image { _ in
            image.draw(in: CGRect(origin: .zero, size: size))
        }
    }
}

struct ImageProcessor {
    let strategy: any ImageResizeStrategy
    func process(_ image: UIImage, size: CGSize) -> UIImage {
        strategy.resize(image, to: size)
    }
}
```

Fit giữ aspect ratio; Stretch thay đổi ratio. Validate size > 0 trước call; production ảnh lớn dùng ImageIO downsample thay decode/draw full UIImage. Caller inject strategy theo feature, test bằng fake. Strategy cũng dùng cho cache policy, retry policy, sort, payment provider adapter (không tự xử lý card secret).

### Ưu và nhược điểm

Ưu: OCP, tách algorithm khỏi caller, test và thay implementation dễ. Nhược: nhiều types/indirection, caller/composition root phải chọn strategy đúng; không over-engineer nếu chỉ một if đơn giản. DI cung cấp strategy; Factory tạo strategy, không đồng nghĩa Strategy.

**Q: Strategy vs Factory?** Strategy chọn behavior; Factory tạo object.

**Q: Strategy vs State?** Strategy là thuật toán thay thế; State phản ánh lifecycle và transition nội bộ.

**Bài tập:** Tạo retry strategies exponential backoff/no retry, test chọn theo HTTP method/status.

## Câu hỏi phỏng vấn tình huống

**Q: Khi nào Strategy tốt hơn switch?** Nhiều thuật toán thay đổi độc lập, cần test/extension; switch nhỏ exhaustive enum có thể đơn giản hơn.

**Q: Strategy có state không?** Có thể, nhưng contract rõ và ownership/thread safety đúng; immutable strategy dễ reuse.

**Q: Ví dụ retry policy?** Strategy chọn retry/backoff theo method/status/idempotency, API client inject policy, test 401/429/5xx.
