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

## Layout và image pipeline chuyên sâu

### UIKit Auto Layout thuần vs SnapKit

```swift
// UIKit native
label.translatesAutoresizingMaskIntoConstraints = false
NSLayoutConstraint.activate([
    label.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
    label.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16),
    label.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor,
                               constant: 12)
])
// SnapKit tương đương (sau addSubview)
label.snp.makeConstraints { make in
    make.leading.trailing.equalToSuperview().inset(16)
    make.top.equalTo(view.safeAreaLayoutGuide.snp.top).offset(12)
}
```

Không chạy cả hai block cho cùng constraint. SnapKit giảm boilerplate, vẫn là NSLayoutConstraint, không nhanh hơn mặc định. Auto Layout giải hệ phương trình, required priority 1000; conflicting required constraints bị break, ambiguous layout thiếu ràng buộc. UIStackView giảm constraint tay, intrinsic content size của UILabel theo text/font. Hugging giữ không giãn, compression resistance giữ không co.

```swift
import SnapKit

private var heightConstraint: Constraint?
// Khi tạo layout:
panel.snp.makeConstraints { make in
    heightConstraint = make.height.equalTo(120).constraint
}
// Khi state đổi:
heightConstraint?.update(offset: 240)
UIView.animate(withDuration: 0.25) {
    view.layoutIfNeeded()
}
```

Đoạn minh họa nằm trong UIViewController với panel đã add vào hierarchy. Không gọi makeConstraints lặp trong layoutSubviews. remake khi đổi anchor quan hệ, update khi đổi constant. Animate layoutIfNeeded trên ancestor phù hợp.

### Image pipeline

```text
URL + size/version -> memory cache -> disk cache -> network
-> downsample/decode -> cache processed -> display nếu ID còn đúng
```

Ảnh 4000 x 3000 x 4 byte ~ 48 MB decoded; JPEG 2 MB không nói lên RAM. Downsample bằng ImageIO/Kingfisher processor trước display, size tính theo point x scale. Cache original và processed là trade-off disk/reprocessing; hạn mức memory/disk, TTL, clear khi logout nếu ảnh riêng tư. Dedup in-flight theo cache key. Placeholder/failure image, request cancel khi reuse/offscreen.

### Collection view pagination và prefetch

```swift
import UIKit

final class GalleryPrefetch: NSObject, UICollectionViewDataSourcePrefetching {
    let start: (IndexPath) -> Void
    let cancel: (IndexPath) -> Void

    init(start: @escaping (IndexPath) -> Void,
         cancel: @escaping (IndexPath) -> Void) {
        self.start = start
        self.cancel = cancel
    }

    func collectionView(_ collectionView: UICollectionView,
                        prefetchItemsAt indexPaths: [IndexPath]) {
        indexPaths.forEach(start)
    }

    func collectionView(_ collectionView: UICollectionView,
                        cancelPrefetchingForItemsAt indexPaths: [IndexPath]) {
        indexPaths.forEach(cancel)
    }
}
```

Controller map indexPath -> stable item ID từ snapshot hiện tại, giữ request theo ID, không assume indexPath ổn định khi insert/delete. Không cancel shared download nếu cell khác còn cần; reference count hoặc library quản lý. Diffable data source snapshot dùng ID unique, reconfigureItems để update content khi phù hợp. Pagination guard isLoading/hasMore, cursor và generation.

### Q&A bổ sung

**Q: Auto Layout priority 999 để làm gì?** Cho constraint gần required nhưng có đường thoát khi xung đột, ví dụ layout text Dynamic Type; không dùng để che lỗi layout.

**Q: Prefetch và lazy loading khác nhau?** Prefetch chuẩn bị trước khi visible, lazy chỉ khi cần; giới hạn prefetch để không tăng memory/network.

**Bài tập:** Gallery 200 ảnh, đo memory trước/sau downsample; tạo layout dynamic text bằng native và SnapKit, test landscape/Dynamic Type.

## Câu hỏi phỏng vấn tình huống

**Q: Tại sao ảnh JPEG 2MB decode tốn 48MB?** 4000×3000×4 bytes RGBA ~48MB, compressed file size không phản ánh decoded memory. Downsample theo target pixel.

**Q: Cache key ảnh gồm gì?** URL/asset ID + version + processor/size/scale; tránh stale content, riêng tư cần scope/clear khi logout.

**Q: Diffable snapshot dùng indexPath làm ID?** Không, stable unique domain ID; indexPath đổi khi insert/delete/reorder. reconfigure content theo ID.
