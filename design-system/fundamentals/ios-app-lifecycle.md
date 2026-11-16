# iOS app lifecycle

Khi app khởi động, sẽ nhảy vào hàm willFinishLaunchingWithOptions và didFinishLaunchingWithOptions (hàm này chúng ta thường dùng để enable 1 số service như: IQKeyboard, Firebase, …

<figure><img src="../.gitbook/assets/image (3).png" alt=""><figcaption></figcaption></figure>

Ngoài ra, ứng dụng iOS sẽ nằm 1 trong 5 trạng thái sau đây:

* Active (Foreground): Khi ứng dụng đang hoạt động bình thường, chúng ta thao tác trên đó
* Inactive (Foreground): nếu có sự kiện từ hệ thống (như có tin nhắn mới, có điện thoại) thì ứng dụng chuyển về **Inactive**
* Background: Khi người dùng bấm phím Home (ẩn ứng dụng xuống nền), ứng dụng sẽ chuyển sang trạng thái background. Ở trạng thái này, ứng dụng không hiển thị với người dùng nhưng vẫn thực hiện các dòng lệnh. Trong trạng thái này, app sẽ tiếp tục chạy nhưng sẽ bị giới hạn về tài nguyên và thời gian chạy
* Suspended: Ứng dụng ở trạng thái background nhưng không còn thực hiện dòng lệnh. Sau 5-10 giây thì bị đóng băng tạm thời. Trong trạng thái này, app sẽ bị đóng và không thể tiếp tục chạy nền. Tuy nhiên, hệ thống sẽ lưu trạng thái của app để khôi phục khi app được mở lên lại.
* Terminated: Khi người dùng đóng app hoặc hệ thống bị đột ngột tắt, app sẽ bị giải phóng khỏi bộ nhớ và chuyển sang trạng thái Terminated. Điện thoại cần memory để chạy 1 ứng dụng khác, thì ứng dụng ở trạng thái suspended sẽ bị chấm dứt và chuyển xuống trạng thái Terminate&#x64;**.** Khi app chuyển sang trạng thái Terminated, tất cả các tài nguyên và bộ nhớ đang được sử dụng bởi app đều được giải phóng
