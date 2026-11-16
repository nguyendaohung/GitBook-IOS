# 😁 UI + Autolayout

* [Storyboard vs Xib](storyboard-vs-xib.md)
* [Autolayout Priority](autolayout-priority.md)
* [SwiftUI](swiftui.md)
* [Texture](texture-asyncdisplaykit.md)
* [SnapKit, Kingfisher và UI Performance](snapkit-image-performance.md)
* [ScrollView, Core Animation, Core Graphics và Attributed String](scroll-animation-graphics-text.md)

## Câu hỏi phỏng vấn tổng quan

**Q: UIKit vs SwiftUI?** UIKit imperative VC/view + constraints; SwiftUI declarative state/identity/layout proposal. Chọn theo OS/codebase/SDK, interop khi cần.

**Q: Layout bug và scroll lag debug?** Inspect constraints/CHCR, stable ID/reuse, Instruments Time Profiler/Animation Hitches, downsample/cache, tránh main-thread heavy work.

**Q: Texture và SnapKit có thay UIKit?** Texture node abstraction trên UIKit, SnapKit DSL Auto Layout, không thay mọi UIKit API.
