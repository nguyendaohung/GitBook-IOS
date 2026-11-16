# DispatchSemaphore

Sử dụng dispatch semaphore để giới hạn số lượng công việc được thực thi cùng lúc.

Bạn nên sử dụng Dispatch Semaphore trong các trường hợp sau:

1. Khi có nhiều luồng cần truy cập vào tài nguyên chia sẻ như một mảng, một đối tượng, hoặc một biến cùng một lúc.
2. Khi cần giới hạn số tác vụ giữ permit; không bảo đảm thứ tự.
3. Khi cần đồng bộ truy cập; sử dụng sai vẫn có thể deadlock.
4. Khi bạn muốn giảm tải cho các luồng bằng cách hạn chế số lượng luồng được truy cập tài nguyên chia sẻ cùng một lúc.

Ví dụ:&#x20;

```
func downloadImages(_ urls: [URL]) {
let semaphore = DispatchSemaphore(value: 2) // Giới hạn số lượng luồng tối đa đang chạy là 2
let queue = DispatchQueue.global(qos: .userInitiated)
for url in urls {
    queue.async {
        semaphore.wait()
        
        URLSession.shared.dataTask(with: url) { data, response, error in
            defer {
                semaphore.signal()
            }
            
            if let error = error {
                print("Error downloading image from url \(url.absoluteString): \(error)")
                return
            }
            
            guard let data = data, let image = UIImage(data: data) else {
                print("Invalid data or image format downloaded from url \(url.absoluteString)")
                return
            }
            
            // Do something with the downloaded image
            DispatchQueue.main.async {
                // Update UI or save image to disk
            }
        }.resume()
    }
}
}
```

Trong ví dụ này, chúng ta sử dụng Dispatch Semaphore để giới hạn số lượng luồng tối đa đang chạy là 2 bằng cách khởi tạo Semaphore với giá trị ban đầu là 2. Mỗi lần một luồng muốn tải xuống ảnh, nó phải yêu cầu giảm giá trị Semaphore trước khi tải xuống, và khi đã tải xong thì luồng đó sẽ tăng giá trị Semaphore lên.

Nếu giá trị Semaphore đã đạt tới giới hạn tối đa (2 trong trường hợp này), các luồng khác sẽ bị chặn và phải đợi đến khi một trong các luồng đang chạy đã hoàn thành tải xuống và signal trả permit.

Với ví dụ này, chúng ta sử dụng `URLSession` để tải xuống ảnh từ các URL được cung cấp. Sau khi tải xuống xong, chúng ta có thể cập nhật UI bằng DispatchQueue.main.async; lưu ảnh xuống đĩa trên queue phù hợp, không block main để đảm bảo rằng các tác vụ được thực hiện trên luồng chính (main thread) của ứng dụng.

<br>

## Semaphore: permit, giới hạn và rủi ro

DispatchSemaphore(value: N) có N permits; wait lấy permit hoặc block khi không có, signal trả permit. Không bảo đảm FIFO/order, không tự tránh deadlock. Nếu wait timeout thất bại thì không signal tương ứng; nếu thành công luôn trả permit bằng defer.

```swift
let semaphore = DispatchSemaphore(value: 1)
func withPermit<T>(_ work: () throws -> T) rethrows -> T {
    semaphore.wait()
    defer { semaphore.signal() }
    return try work()
}
```

Chỉ minh họa synchronous critical section ngắn; không dùng semaphore.wait trên main/cooperative executor để chờ async network. Ví dụ cũ download 1000 ảnh enqueue 1000 blocks chờ permit có thể gây thread explosion/starvation. Ưu tiên bounded TaskGroup (xem Swift Concurrency) hoặc async OperationQueue đúng lifecycle; RxSwift merge(maxConcurrent:) với cold sources. Không ghi file lớn trên main.

**Q: Semaphore value 2 có giới hạn thread đúng 2?** Giới hạn số holder permits, không giới hạn số threads được scheduler tạo.

**Q: Semaphore có bảo đảm thứ tự?** Không; dùng serial queue/dependencies/concatMap khi cần ordering.

**Q: Semaphore vs lock?** Semaphore permit count có thể >1; mutex bảo vệ critical section độc quyền, ownership rules khác.

**Q: Tại sao wait trên main nguy hiểm?** Block UI, nếu signal cần main callback sẽ deadlock.

**Bài tập:** So sánh bounded TaskGroup với semaphore legacy, test cancel và error không leak permit.
