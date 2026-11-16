# Storyboard vs Xib

**Storyboard và XIB** là hai cách để thiết kế giao diện người dùng trong Swift. Dưới đây là so sánh giữa hai cách này:

1. **Tổ chức dự án:**

Storyboard cho phép bạn tạo nhiều màn hình giao diện người dùng trong cùng một file, trong khi XIB chỉ cho phép tạo một màn hình giao diện người dùng trong một file. Vì vậy, nếu dự án của bạn có nhiều màn hình, bạn có thể dễ dàng quản lý chúng bằng cách sử dụng storyboard.

2. **Hiệu suất:**

Storyboard không mặc định instantiate mọi scene cùng lúc; performance phụ thuộc instantiation/layout/assets, cần đo.

3. **Tính linh hoạt:**

Storyboard có tính linh hoạt cao hơn so với XIB. Bạn có thể thực hiện nhiều chức năng bằng cách sử dụng Segues và các kết nối giữa các màn hình khác nhau. Tuy nhiên, XIB cho phép bạn tùy chỉnh nhiều hơn vì bạn có thể chỉnh sửa từng màn hình một cách độc lập.

4. **Khả năng tái sử dụng:**

XIB cho phép bạn tái sử dụng các thành phần giao diện người dùng. Bạn có thể sử dụng một XIB chung để tạo các màn hình giao diện người dùng khác nhau. Trong khi đó, với Storyboard, bạn cần phải tạo các màn hình giao diện người dùng riêng biệt cho từng màn hình trong ứng dụng của mình.

\=> _Về cơ bản, chúng ta có thể sử dụng **Storyboard** cho các dự án có nhiều màn hình và có tính tương tác phức tạp giữa chúng, đồng thời tận dụng tính linh hoạt của **Segues** để chuyển đổi giữa các màn hình. Các dự án có một màn hình hoặc một vài màn hình đơn giản hơn thì có thể sử dụng **XIB**, đồng thời tận dụng khả năng **tái sử dụng** của **XIB** để tạo các thành phần giao diện người dùng chung cho các màn hình khác nhau._

## Storyboard, XIB và programmatic UI

Storyboard mô tả nhiều scene/segue; XIB mô tả object graph (UIView/VC/custom cell), không bắt buộc một màn hình. Storyboard không tải mọi scene cùng lúc mặc định; performance phụ thuộc instantiation/layout/assets, cần profile. Programmatic UI dễ review diff/DI/dynamic layout, nhiều setup code; storyboard/XIB trực quan nhưng merge conflicts/outlet/runtime configuration.

```swift
import UIKit

let storyboard = UIStoryboard(name: "Main", bundle: nil)
let detail = storyboard.instantiateViewController(withIdentifier: "Detail")
let nib = UINib(nibName: "ProductCell", bundle: nil)
// tableView.register(nib, forCellReuseIdentifier: "ProductCell")
```

Identifier/module/class/outlet phải đúng; XIB custom view init(coder:), awakeFromNib; programmatic init(frame:) hoặc init(nibName:bundle:). Storyboard segue prepare(for:sender:) truyền data, initializer injection có creator API khi phù hợp. Không mặc định storyboard cho app lớn, XIB cho nhỏ; chia feature scene và team workflow.

**Q: XIB vs storyboard?** XIB reusable object graph; storyboard scene/navigation graph, có thể kết hợp.

**Q: Storyboard load tất cả VC?** Không, instantiate scenes theo navigation/identifier.

**Q: Programmatic UI có ưu điểm?** DI rõ, diff/merge dễ, dynamic composition; đổi lại boilerplate/layout code.

**Q: IBOutlet nil vì sao?** Sai connection/class, truy cập trước view load, nib/identifier/target membership sai.
