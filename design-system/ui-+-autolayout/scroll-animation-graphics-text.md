# UIScrollView, Core Animation, Core Graphics và Attributed String

## 1. UIScrollView: contentSize, contentInset, contentOffset

UIScrollView hiển thị một viewport nhỏ hơn content. bounds.size là kích thước viewport; contentSize là kích thước vùng nội dung; contentOffset là vị trí cuộn trong hệ tọa độ content; contentInset là khoảng đệm bổ sung quanh content. adjustedContentInset gồm contentInset và system adjustment (safe area/bar) theo configuration.

```swift
import UIKit

final class ScrollExample: UIViewController {
    private let scrollView = UIScrollView()
    private let content = UIStackView()

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground
        scrollView.translatesAutoresizingMaskIntoConstraints = false
        content.translatesAutoresizingMaskIntoConstraints = false
        content.axis = .vertical
        content.spacing = 12
        view.addSubview(scrollView)
        scrollView.addSubview(content)

        NSLayoutConstraint.activate([
            scrollView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            scrollView.bottomAnchor.constraint(equalTo: view.bottomAnchor),
            scrollView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            scrollView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            content.topAnchor.constraint(equalTo: scrollView.contentLayoutGuide.topAnchor),
            content.bottomAnchor.constraint(equalTo: scrollView.contentLayoutGuide.bottomAnchor),
            content.leadingAnchor.constraint(equalTo: scrollView.contentLayoutGuide.leadingAnchor),
            content.trailingAnchor.constraint(equalTo: scrollView.contentLayoutGuide.trailingAnchor),
            content.widthAnchor.constraint(equalTo: scrollView.frameLayoutGuide.widthAnchor)
        ])

        for index in 1...30 {
            let label = UILabel()
            label.text = "Row \(index)"
            content.addArrangedSubview(label)
        }
    }

    func scrollToTop() {
        scrollView.setContentOffset(
            CGPoint(x: -scrollView.adjustedContentInset.left,
                    y: -scrollView.adjustedContentInset.top),
            animated: true
        )
    }
}
```

contentLayoutGuide định nghĩa content geometry; frameLayoutGuide định nghĩa viewport. Pin content width bằng frame width để vertical scroll. Content height cần đủ constraints từ top tới bottom; stack intrinsic size từ arrangedSubviews. Không vừa set contentSize thủ công vừa dùng Auto Layout không nhất quán. Table/Collection đã kế thừa scroll view, có data source/reuse/prefetch; không tạo hàng nghìn UIView trong stack nếu list lớn.

### Thực tế: keyboard, pagination và scroll indicator

Keyboard avoidance: keyboardLayoutGuide (iOS 15+) hoặc keyboard notification + coordinate conversion; điều chỉnh bottom constraint/inset và scroll active field, tránh cộng safe area hai lần. contentInsetAdjustmentBehavior quyết định automatic adjustment. scrollIndicatorInsets điều chỉnh indicator, không thay content position. scrollViewDidScroll có tần suất cao; throttle/sample metrics, không call API mỗi pixel. Pagination theo stable cursor/isLoading/hasMore và threshold, tránh duplicate trigger.

**Q: contentOffset khi ở đầu scroll có luôn 0?** Không, y thường là -adjustedContentInset.top khi có inset.

**Q: contentInset và adjustedContentInset khác gì?** contentInset là developer-set, adjusted thêm system adjustment theo behavior.

**Q: ScrollView content không cuộn dù nhiều subviews?** Thiếu content constraints/contentSize hoặc content không vượt viewport; kiểm tra contentLayoutGuide/frameLayoutGuide.

**Q: Pagination gọi lặp nhiều lần?** Gate isLoading/hasMore, stable cursor, dedup và generation khi refresh, không chỉ dựa didScroll.

## 2. CGRect, CGPoint, CGSize và CGAffineTransform

CGRect gồm origin CGPoint và size CGSize, đơn vị point với UIKit. frame trong superview, bounds local, center trong superview. CGAffineTransform là affine transform 2D (translation/rotation/scale/shear); thứ tự concatenation quan trọng. Khi transform không identity, tránh đặt frame trực tiếp vì frame là bounding rectangle.

```swift
import UIKit

let tile = UIView(frame: CGRect(x: 20, y: 30, width: 100, height: 60))
tile.backgroundColor = .systemBlue
tile.transform = CGAffineTransform(rotationAngle: .pi / 4)
    .scaledBy(x: 1.2, y: 1.2)
// Reset
tile.transform = .identity
```

Point conversion dùng convert(_:to:) / convert(_:from:) giữa coordinate spaces; không cộng frame.origin qua ancestors tùy ý vì transform/scrolling. Core Animation layer có anchorPoint/position/bounds và CATransform3D cho 3D. Thay anchorPoint có thể làm view/layer position nhìn dịch nếu không bù position.

## 3. Core Animation: CALayer, animation và rendering

UIView có backing CALayer. UIKit quản lý event/layout; Core Animation compositing/rendering và animatable properties (opacity, position, transform, bounds...). Model layer chứa giá trị đích, presentation layer phản ánh giá trị đang animate, render server thực thi compositing. Không mutate presentation layer. UIView.animate thay model properties trong transaction, animation interpolation do system.

```swift
import UIKit

func showPanel(_ panel: UIView) {
    panel.alpha = 0
    panel.transform = CGAffineTransform(translationX: 0, y: 20)
    UIView.animate(withDuration: 0.25, delay: 0,
                   options: [.curveEaseOut]) {
        panel.alpha = 1
        panel.transform = .identity
    }
}
```

Animate constraint: set constant mới rồi UIView.animate { ancestor.layoutIfNeeded() }; layoutIfNeeded trước khi đổi nếu cần baseline. UIViewPropertyAnimator cho interactive/interruptible animation (fractionComplete, pause/continue). Core Animation explicit CABasicAnimation:

```swift
import QuartzCore

func pulse(_ layer: CALayer) {
    let animation = CABasicAnimation(keyPath: "opacity")
    animation.fromValue = 1.0
    animation.toValue = 0.4
    animation.duration = 0.2
    animation.autoreverses = true
    layer.add(animation, forKey: "pulse")
}
```

Ví dụ pulse quay về model opacity mặc định 1; explicit animation không tự thay model value. Nếu animation cần giữ final value, set model layer property và add animation phù hợp, không chỉ fillMode/isRemovedOnCompletion để giả vờ state đã đổi. Transaction/CATransaction dùng implicit action management; UIView-backed layer thường được UIKit quản lý.

### Performance

Offscreen rendering có thể xảy ra với masking/shadow/compositing tùy layer configuration; không mặc định cornerRadius luôn offscreen. Dùng Instruments Animation Hitches/Core Animation để đo. shadowPath có thể giúp shadow geometry ổn định, tránh rasterize mọi view (cache invalidation/scale/memory). Không làm CPU-heavy layout/drawing mỗi frame. Tôn trọng Reduce Motion và accessibility.

**Q: Model layer vs presentation layer?** Model đích, presentation giá trị đang interpolate; dùng presentation để đọc current visual state khi interrupt, không mutate trực tiếp.

**Q: UIView.animate vs CABasicAnimation?** UIKit API cập nhật model properties và animate; explicit CA animation cần tự đảm bảo model final state.

**Q: Animation constraint không chạy?** Cần layout baseline, update constraint constant và layoutIfNeeded trong animation trên ancestor đúng.

**Q: 60Hz vs 120Hz?** Tổng frame budget ~16.7ms/~8.3ms, app chỉ một phần; profile hitches và main/GPU work.

## 4. Core Graphics và custom drawing

Core Graphics (Quartz 2D) cung cấp CGContext, CGPath, color, transforms, drawing bitmap/PDF. UIKit draw(_:) dùng CGContext hiện tại; CALayer/CAShapeLayer phù hợp vector shape/animation. Không gọi draw trực tiếp; setNeedsDisplay yêu cầu redraw theo system. Core Graphics coordinate systems tùy context; UIKit view drawing origin trên-trái, cần chú ý flip với raw CGContext.

```swift
import UIKit

final class ProgressRingView: UIView {
    var progress: CGFloat = 0.5 {
        didSet { setNeedsDisplay() }
    }

    override func draw(_ rect: CGRect) {
        guard let context = UIGraphicsGetCurrentContext() else { return }
        let center = CGPoint(x: bounds.midX, y: bounds.midY)
        let radius = max(0, min(bounds.width, bounds.height) / 2 - 8)
        let start = -CGFloat.pi / 2
        let end = start + 2 * .pi * min(max(progress, 0), 1)

        context.setLineWidth(8)
        context.setLineCap(.round)
        context.setStrokeColor(UIColor.systemBlue.cgColor)
        context.addArc(center: center, radius: radius,
                       startAngle: start, endAngle: end, clockwise: false)
        context.strokePath()
    }
}
```

Ví dụ custom ring nhỏ; animation progress dùng CAShapeLayer.strokeEnd hoặc SwiftUI Shape/trim phù hợp hơn redraw CPU mỗi frame. UIGraphicsImageRenderer tạo bitmap theo point/scale; ImageIO thumbnail downsample trước decode ảnh lớn. CGContext saveGState/restoreGState khi đổi transform/clip; không chia sẻ mutable context qua threads tùy ý.

**Q: Core Graphics vs Core Animation?** Core Graphics vẽ 2D pixels/vector vào context; Core Animation compositing/animate layer. Có thể phối hợp.

**Q: setNeedsDisplay vs setNeedsLayout?** Redraw content vs layout subviews. layoutIfNeeded force pending layout, không trực tiếp thay drawing.

**Q: CAShapeLayer vs draw(_:) chọn gì?** Shape layer tốt cho vector path/animatable stroke; draw custom khi cần raster/drawing phức tạp, profile.

## 5. Attributed String, String và text rendering

Swift String là Unicode text, Character là extended grapheme cluster; không dùng integer subscript tùy ý. String.Index, unicodeScalars/utf8/utf16 khác nhau. NSString/NSAttributedString dùng UTF-16 ranges (NSRange); chuyển Range<String.Index> bằng NSRange(range,in: text), không lấy character count làm NSRange length cho emoji.

```swift
import UIKit

let text = "Premium ⭐ yearly"
let attributed = NSMutableAttributedString(string: text)
if let range = text.range(of: "Premium") {
    attributed.addAttributes([
        .font: UIFont.boldSystemFont(ofSize: 17),
        .foregroundColor: UIColor.systemBlue
    ], range: NSRange(range, in: text))
}
let label = UILabel()
label.attributedText = attributed
label.numberOfLines = 0
```

NSAttributedString immutable, NSMutableAttributedString mutate attributes/ranges. Swift AttributedString (iOS 15+) value type, SwiftUI Text có thể render attributes được hỗ trợ; bridge theo attribute scope, không giả định mọi UIKit attribute render giống SwiftUI. Dùng localized string catalog và semantic text, không hardcode substring tiếng Anh trong mọi locale. Dynamic Type, line break/paragraph style và accessibility.

```swift
import SwiftUI

struct PremiumText: View {
    var body: some View {
        var title = AttributedString("Premium")
        title.font = .headline
        title.foregroundColor = .blue
        return Text(title)
    }
}
```

String operations: isEmpty, trimmingCharacters(in:), split, contains/range(of:), replacingOccurrences, map/filter theo Character. count tính grapheme clusters, utf8.count byte count, utf16.count NSRange length khi cần. Tránh O(n²) repeated index(offsetBy:) trong loop dài; dùng iteration/slicing. String interpolation không thay locale-aware number/date formatting.

**Q: String.count vs NSString.length?** Character grapheme count vs UTF-16 code units; emoji có thể khác, dùng NSRange(range,in:) cho attributed text.

**Q: AttributedString vs NSAttributedString?** Swift value-type API vs Foundation object/UTF-16 ranges; UIKit thường dùng NSAttributedString, SwiftUI hỗ trợ AttributedString.

**Q: Vì sao label text dài bị cắt?** numberOfLines, constraints/CHCR, lineBreakMode, intrinsic size/Dynamic Type; test layout locale dài.

### Bài tập

1. Tạo vertical ScrollView với contentLayoutGuide, keyboard avoidance và scroll-to-top; kiểm tra inset/safe area.
2. Animate panel constraint, interactive property animator và CAShapeLayer ring.
3. Custom drawing icon bằng Core Graphics, profile redraw so với shape layer.
4. Attributed text có emoji/tiếng Việt, test NSRange đúng và Dynamic Type/RTL.
5. List pagination: scroll threshold + state gate, test duplicate trigger và refresh.
