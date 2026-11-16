# SnapKit, Kingfisher và UI Performance

### SnapKit

DSL cho Auto Layout. makeConstraints tạo constraint; updateConstraints thay constant constraint phù hợp; remakeConstraints gỡ constraint do SnapKit quản lý trên view và tạo lại.

```swift
import UIKit
import SnapKit

final class ProductView: UIView {
    private let titleLabel = UILabel()

    override init(frame: CGRect) {
        super.init(frame: frame)
        addSubview(titleLabel)
        titleLabel.numberOfLines = 0
        titleLabel.snp.makeConstraints { make in
            make.edges.equalToSuperview().inset(16)
        }
    }

    required init?(coder: NSCoder) { nil }
}
```

Auto Layout: intrinsicContentSize, content hugging (chống giãn), compression resistance (chống co), priority 1000 required; debug unsatisfiable constraints với symbolic breakpoint UIViewAlertForUnsatisfiableConstraints. contentInset là vùng đệm scroll, contentOffset là vị trí scroll, adjustedContentInset tính cả safe area. CGAffineTransform thay đổi transform; frame không đáng tin khi transform không identity, bounds/center dùng layout.

### Kingfisher và Image Cache

```swift
import UIKit
import Kingfisher

final class ProductCell: UICollectionViewCell {
    private let photoView = UIImageView()

    func configure(url: URL, size: CGSize) {
        photoView.kf.cancelDownloadTask()
        photoView.kf.setImage(
            with: url,
            placeholder: UIImage(systemName: "photo"),
            options: [.processor(DownsamplingImageProcessor(size: size)),
                      .scaleFactor(UIScreen.main.scale)]
        )
    }

    override func prepareForReuse() {
        super.prepareForReuse()
        photoView.kf.cancelDownloadTask()
        photoView.image = nil
    }
}
```

Ví dụ tập trung vào lifecycle tải ảnh, chưa có layout photoView. Memory cache giảm decode/I/O; disk cache giảm download. Downsample theo pixel size màn hình; ảnh 4000x3000 RGBA có thể chiếm khoảng 48 MB sau decode dù JPEG nhỏ. Key cache cần gồm URL/processor và version nội dung. Prefetch có giới hạn, cancel khi item không còn cần; không tải toàn bộ gallery.

### Texture (AsyncDisplayKit)

ASDisplayNode, ASCellNode, ASTableNode/ASCollectionNode; layoutSpecThatFits xây ASLayoutSpec, node block có thể được gọi background, chỉ thao tác UIKit/view/layer trên main thread. Range controller chuẩn bị display/preload trước khi cell xuất hiện. Tránh capture indexPath biến đổi; snapshot model theo ID. So với UICollectionView + prefetch/diffable, Texture thêm dependency và learning curve; chỉ chọn khi profiling cho thấy cần.

### Q&A

**Q: Cell hiện sai ảnh?** Cancel download khi reuse, reset placeholder, kiểm tra ID/URL trước apply callback.

**Q: updateConstraints vs remakeConstraints?** Update constant trên constraint hiện có; remake khi đổi quan hệ/anchor, tránh tạo constraint trùng.

**Q: News feed lag?** Dùng Time Profiler/Core Animation đo main-thread layout, image decode, offscreen rendering; pagination, downsample và async preparation.

**Liên hệ:** BIDU Texture/Kingfisher; Map4D SnapKit; Watch Faces cache bộ sưu tập ảnh.
