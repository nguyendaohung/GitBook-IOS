# Autolayout Priority

Link tham khảo: [https://thanhvu.dev/2019/09/16/autolayout-priority/](https://thanhvu.dev/2019/09/16/autolayout-priority/)

**Content hugging priority** là độ ưu tiên quyết định view không bị dãn ra.&#x20;

Ví dụ: có 2 label trong 1 hàng.&#x20;

* Label 1 nếu có hugging lớn hơn Label 2 thì Label 1 sẽ có width bằng với title của nó, còn Label 2 thì lấy đầy phần width còn lại (bị kéo dài ra so với title của nó).
* Label 1 nếu có hugging nhỏ hơn Label 2 thì Label 1 sẽ bị kéo dài ra, còn Label 2 thì có width bằng với title của nó.



**Content compression resistance priority** là độ ưu tiên quyết định view không bị co vào.&#x20;

Ví dụ: có 2 button trong 1 hàng. 2 button này có title dài hơn chiều dài màn hình.

* Button 1 nếu có priority lớn hơn Button 2 thì Button 1 ưu tiên giữ intrinsic size còn Button 2 bị co trước

## Auto Layout: priority, CHCR và lỗi thực tế

```swift
import UIKit

let title = UILabel()
let value = UILabel()
title.setContentHuggingPriority(.defaultHigh, for: .horizontal)
value.setContentHuggingPriority(.defaultLow, for: .horizontal)
title.setContentCompressionResistancePriority(.required, for: .horizontal)
value.setContentCompressionResistancePriority(.defaultHigh, for: .horizontal)
```

Trong horizontal stack, title chống giãn tốt hơn và chống co tốt hơn value. Hugging: chống lớn hơn intrinsic size; compression resistance: chống nhỏ hơn intrinsic. Priority 1000 required, 1...999 optional. Nếu required conflict, system break constraint và log; nếu thiếu constraints, ambiguous layout. Dùng symbolic breakpoint UIViewAlertForUnsatisfiableConstraints, hasAmbiguousLayout, debug view hierarchy. Không set mọi CHCR required nếu Dynamic Type text dài có thể conflict.

**Q: Hugging vs compression?** Hugging chống stretch, compression chống shrink.

**Q: Vì sao priority 999?** Optional gần required tạo đường thoát khi conflict; dùng có lý do, không che layout bug.

**Q: translatesAutoresizingMaskIntoConstraints?** Programmatic constraints thường false để không sinh mask constraints xung đột; storyboard system quản lý.

**Q: Auto Layout vs SwiftUI layoutPriority?** Auto Layout constraint solver/CHCR, SwiftUI proposal/size selection; layoutPriority không tương đương 1000 required.

**Bài tập:** Hai labels trong stack, test title dài/RTL/Dynamic Type và thay CHCR.
