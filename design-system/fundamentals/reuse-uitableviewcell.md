# Reuse UITableViewCell

The problem: a table view can potentially have thousands of rows (or millions, whatever). It would be tedious and wasteful to create a separate cell for each data row. Instead, the table view only asks for as many rows as it displays on the screen at the same time (this is typically no more than 10-15-20 cells). This is manageable, doesn't consume a lot of memory and plays nice with the fact that not all of the rows are visible on the display anyway.

So, when the table view needs a new cell to be displayed (because the user has scrolled the view), it takes a cell that went out of the visible area, and queues it back to the end, reusing it

Vấn đề: Một tableview có thể có hàng nghìn hàng (hoặc hàng triệu, bất kỳ). Sẽ là mệt mỏi và lãng phí để tạo một ô riêng cho mỗi hàng dữ liệu. Thay vào đó, tableview chỉ yêu cầu bấy nhiêu hàng như nó hiển thị trên màn hình cùng lúc (điều này thông thường là không nhiều hơn 10-15-20 ô). Điều này có thể quản lý, không tiêu tốn nhiều bộ nhớ và hoạt đọng tốt với sự thật rằng không phải tất cả các hàng đều hiển thị trên màn hình.&#x20;

Vì vậy, khi tableview cần một ô mới được hiển thị (bởi vì người dùng đã cuộn xem), nó lấy một ô đã rời khỏi khu vực hiển thị, và xếp hàng nó vào cuối, tái sử dụng nó.

Cơ chế reuse của table view là quá trình sử dụng lại các đối tượng (thường là các ô trong table view) đã được tạo trước đó. Khi một ô trong table view không còn được hiển thị trên màn hình, nó sẽ được gán cho một mục dữ liệu khác trong danh sách. Điều này giúp giảm số lượng đối tượng mới tạo ra và tăng tốc độ hiển thị của table view.\
Bạn có thể tạo các ô trong table view bằng cách sử dụng lớp UITableViewCell và xác định một reuse identifier cho chúng. Khi table view cần hiển thị một ô, nó sẽ tìm kiếm một đối tượng đã được tạo với reuse identifier tương tự và sử dụng nó. Nếu không tìm thấy, table view sẽ tạo một đối tượng mới.

nếu muốn download 1000 ảnh dưới background thread thì xử lý ntn => Xử dụng operation queue để giới hạn số ảnh download 1 lần để hiện ra sớm cho user. =>Tối ưu UX, Hoặc dùng Dispatch Semaphore&#x20;

## Cell reuse thực tế: async image và stable ID

```swift
import UIKit

final class ProductCell: UITableViewCell {
    private var representedID: String?
    private var imageTask: Task<Void, Never>?

    func configure(id: String, title: String, imageURL: URL) {
        representedID = id
        imageTask?.cancel()
        var content = defaultContentConfiguration()
        content.text = title
        content.image = UIImage(systemName: "photo")
        contentConfiguration = content

        imageTask = Task { [weak self] in
            guard let (data, response) = try? await URLSession.shared.data(from: imageURL),
                  let http = response as? HTTPURLResponse,
                  (200..<300).contains(http.statusCode),
                  !Task.isCancelled, self?.representedID == id else { return }
            // Demo nhỏ; production downsample/cache off main.
            guard let image = UIImage(data: data), let self else { return }
            var content = self.defaultContentConfiguration()
            content.text = title
            content.image = image
            self.contentConfiguration = content
        }
    }

    override func prepareForReuse() {
        super.prepareForReuse()
        representedID = nil
        imageTask?.cancel()
        imageTask = nil
        contentConfiguration = nil
    }
}
```

Register/dequeue theo reuseIdentifier, configure đầy đủ mọi state (text, image, selected, hidden). Cell không tương đương record; async callback phải verify stable ID, cancel cũ. prepareForReuse không gọi khi reconfigureItems của diffable data source, nên configure tự reset. Prefetch có thể chạy trước visible; không giả định số cells cố định. Giới hạn concurrent transfer bằng TaskGroup/OperationQueue đúng async operation lifecycle, không block main bằng semaphore.

**Q: Tại sao cell hiện ảnh sai?** Callback request cũ sau reuse; cancel, reset placeholder, verify representedID.

**Q: reloadData vs diffable snapshot?** reloadData refresh rộng; diffable dựa stable unique IDs, apply/reconfigure phù hợp, không tự sửa model.

**Q: Prefetch có cần cancel?** Có khi item không còn cần; shared request cần ownership/ref count để không hủy của cell khác.

**Q: prepareForReuse nên reset gì?** Transient state/callback/request; configure đầy đủ content mỗi lần.

**Bài tập:** List 200 ảnh, scroll nhanh, test reorder và async response cũ; profile memory/downsample.
