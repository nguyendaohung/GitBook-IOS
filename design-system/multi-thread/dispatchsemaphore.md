# DispatchSemaphore

Sử dụng dispatch semaphore để giới hạn số lượng công việc được thực thi cùng lúc.

Bạn nên sử dụng Dispatch Semaphore trong các trường hợp sau:

1. Khi có nhiều luồng cần truy cập vào tài nguyên chia sẻ như một mảng, một đối tượng, hoặc một biến cùng một lúc.
2. Khi bạn muốn đảm bảo rằng các tác vụ được thực hiện theo thứ tự nhất định.
3. Khi bạn muốn đồng bộ hóa các luồng để tránh race condition và deadlock.
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

Nếu giá trị Semaphore đã đạt tới giới hạn tối đa (2 trong trường hợp này), các luồng khác sẽ bị chặn và phải đợi đến khi một trong các luồng đang chạy đã hoàn thành tải xuống và giảm giá trị Semaphore.

Với ví dụ này, chúng ta sử dụng `URLSession` để tải xuống ảnh từ các URL được cung cấp. Sau khi tải xuống xong, chúng ta có thể cập nhật UI hoặc lưu ảnh xuống đĩa bằng cách sử dụng DispatchQueue.main.async để đảm bảo rằng các tác vụ được thực hiện trên luồng chính (main thread) của ứng dụng.

<br>
