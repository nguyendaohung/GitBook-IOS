# Frame VS Bounds

* Frame: Là kích thước và vị trí của view trong hệ tọa độ cha của nó. Tức là, frame được đo bằng đơn vị point trong hệ tọa độ của superview (view cha). Ví dụ, nếu một view có frame (x: 10, y: 20, width: 100, height: 50) thì nó sẽ có vị trí x = 10, y = 20 so với góc trên bên trái của superview, và có kích thước 100x50.
* Bounds: Là kích thước và vị trí của view trong hệ tọa độ của chính nó. Tức là, bounds được đo bằng đơn vị point trong hệ tọa độ của view đó. Ví dụ, nếu một view có bounds (x: 0, y: 0, width: 100, height: 50) thì nó sẽ có vị trí x = 0, y = 0 so với góc trên bên trái của nó, và có kích thước 100x50.

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

```
Frame
   origin = (20, 52)
   width = 118
   height = 187
 
Bounds 
   origin = (0, 0)
   width = 80
   height = 130
```

Vì frame và bounds được đo bằng các hệ tọa độ khác nhau, nên chúng ta cần chú ý khi sử dụng chúng trong mã của mình. Ví dụ, nếu ta muốn lấy vị trí của view trong superview, dùng frame; muốn tọa độ window cần convert. Tuy nhiên, nếu ta muốn tính toán kích thước thực tế của nội dung trong một view, ta nên sử dụng thuộc tính bounds.

Một số điểm khác biệt khác giữa frame và bounds:

* Thay đổi giá trị của frame sẽ ảnh hưởng đến vị trí và kích thước của view trong hệ tọa độ cha của nó, trong khi thay đổi giá trị của bounds chỉ ảnh hưởng đến vị trí và kích thước của nội dung trong view.
* Bounds thường được sử dụng để thực hiện các phép biến đổi hình học, ví dụ như xoay, thu phóng, hoặc dịch chuyển nội dung của view.
* Frame thường được sử dụng để xác định vị trí và kích thước của một view trong quá trình định vị.

## Frame, Bounds, Center và Transform

UIKit layout dùng point, không phải pixel. Pixel = point × scale của render target. frame là rectangle trong coordinate space của superview; bounds là rectangle trong local coordinate space, origin thường (0,0) nhưng có thể khác (scrolling). center là vị trí trung tâm trong superview. Khi transform không identity, không dùng frame để set layout; dùng bounds/center/constraints.

```swift
import UIKit

let parent = UIView(frame: CGRect(x: 0, y: 0, width: 300, height: 300))
let child = UIView(frame: CGRect(x: 20, y: 30, width: 100, height: 50))
parent.addSubview(child)
print(child.frame.origin)  // 20,30
print(child.bounds.origin) // 0,0
child.bounds.origin.x = 10
let localPoint = CGPoint(x: 10, y: 0)
let inParent = child.convert(localPoint, to: parent)
print(inParent)
```

Thay bounds.origin đổi local coordinate mapping/content, không phải vị trí view trong parent. frame không phải tọa độ màn hình nếu nhiều ancestor; dùng convert(_:to:) / convert(_:from:) và window coordinate space. Auto Layout tính frame sau layout pass; đọc bounds trong viewDidLayoutSubviews khi cần size cuối.

**Q: Vì sao frame.size có thể khác bounds.size?** Transform rotation/scale; frame là bounding rectangle trong parent, bounds local.

**Q: contentOffset liên quan bounds?** UIScrollView cuộn bằng thay đổi coordinate origin/contentOffset, adjustedContentInset tính safe area.

**Q: Lấy vị trí view trên màn hình?** convert rectangle/point sang window hoặc coordinate space cần, không chỉ đọc frame.
