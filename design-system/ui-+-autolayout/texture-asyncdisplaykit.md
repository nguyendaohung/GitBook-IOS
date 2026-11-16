# Texture(AsyncDisplayKit)

Texture được build bởi Facebook, là 1 framework của iOS giúp ứng dụng có UI phức tạp trở nên mượt mà và phản hồi tốt. Đơn vị của Texture là **node**. Không giống như **views** (chỉ được dùng trên main thread), node thể khởi tạo vào cấu hình toàn bộ hierarchies (nôm na là cây thư mục cấu tạo nên node ???) song song trong background threads

Texture cho phép đưa image decoding, text sizing, rendering hoặc nhiều hoạt động UI phức tạp khác ra khỏi main thread

⇒ Giúp main thread ít trách nhiệm nhất ⇒ Sẵn sàng hoạt động khi user interaction ⇒ Rất mượt

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption><p>How Texture layout in table view</p></figcaption></figure>

## Texture chuyên sâu

### Node vs UIView

Texture là abstraction trên UIKit. ASDisplayNode có thể chuẩn bị nhiều phần layout/render ngoài main thread, view/layer backing được load khi cần; UIKit object vẫn tuân thủ main thread. Không phải mọi code trong node tự chạy background. ASCellNode quản lý cell content; ASTableNode/ASCollectionNode bọc table/collection, data source dùng node block.

| UIKit | Texture |
| --- | --- |
| UIView, UILabel, UIImageView | ASDisplayNode, ASTextNode, ASImageNode |
| UITableViewCell | ASCellNode |
| UITableView/UICollectionView | ASTableNode/ASCollectionNode |
| Auto Layout | ASLayoutSpec (flexbox-like) |
| Cell reuse/prefetch | Node block, range controller |

```swift
import AsyncDisplayKit
import UIKit

final class ArticleNode: ASCellNode {
    private let titleNode = ASTextNode()
    private let imageNode = ASNetworkImageNode()

    init(title: String, imageURL: URL) {
        super.init()
        automaticallyManagesSubnodes = true
        titleNode.attributedText = NSAttributedString(
            string: title, attributes: [.font: UIFont.systemFont(ofSize: 17)]
        )
        imageNode.url = imageURL
        imageNode.style.preferredSize = CGSize(width: 80, height: 80)
    }

    override func layoutSpecThatFits(_ constrainedSize: ASSizeRange) -> ASLayoutSpec {
        titleNode.style.flexShrink = 1
        let stack = ASStackLayoutSpec(
            direction: .horizontal, spacing: 12,
            justifyContent: .start, alignItems: .center,
            children: [imageNode, titleNode]
        )
        return ASInsetLayoutSpec(
            insets: UIEdgeInsets(top: 12, left: 16, bottom: 12, right: 16),
            child: stack
        )
    }
}
```

Ví dụ cần dependency Texture. automaticallyManagesSubnodes tự thêm/bỏ subnode theo layout spec; không trộn manual addSubnode tùy ý. ASNetworkImageNode dùng pipeline riêng; không mặc định Kingfisher nếu chưa tích hợp adapter. Node block nên capture model value, không capture indexPath rồi đọc data source mutable ở background.

```swift
// Trong ASTableDataSource:
func tableNode(_ tableNode: ASTableNode,
               nodeBlockForRowAt indexPath: IndexPath) -> ASCellNodeBlock {
    let article = articles[indexPath.row] // snapshot trên callback
    return {
        ArticleNode(title: article.title, imageURL: article.imageURL)
    }
}
```

Đoạn minh họa nằm trong controller/data source có articles là value models. Range controller điều chỉnh preload/display/visible ranges; dùng interface state để bắt đầu/dừng work. didLoad truy cập backing view trên main, tránh giữ UI object ngoài main.

### Trade-off và profiling

Texture phù hợp feed cell phức tạp, layout/text/image preparation nặng. UIKit hiện đại có diffable data source, UICollectionViewCompositionalLayout, prefetch, async image decode; SwiftUI LazyVStack/List có declarative state. Không chọn Texture chỉ vì benchmark chung; đo frame hitches, CPU, peak memory và chi phí maintain.

**Q: Tại sao không dùng UIView trong node block background?** UIKit UI creation/update có main-thread constraints, node abstraction hỗ trợ async preparation có giới hạn.

**Q: Node block capture indexPath có rủi ro gì?** Data snapshot đổi trước khi block chạy; capture immutable model/stable ID.

**Bài tập:** Feed article bằng Texture và UICollectionView, đo scrolling/peak memory với cùng ảnh, ghi trade-off.

## Câu hỏi phỏng vấn tình huống

**Q: ASTableNode node block được gọi thread nào?** Có thể được thực thi ngoài main; capture immutable model, không truy cập UIKit/view/layer hoặc mutable indexPath state.

**Q: ASLayoutSpec khác Auto Layout?** Layout spec composition theo size range/flexbox-like, hỗ trợ async measurement; Auto Layout constraint solver UIKit.

**Q: Texture có luôn tốt hơn UICollectionView?** Không, profile same workload/frame hitches/memory, cân nhắc dependency và UIKit diffable/prefetch.

**Q: ASNetworkImageNode dùng Kingfisher mặc định?** Không, có pipeline riêng; cần adapter/configuration nếu muốn dùng cache/downloader khác.
