# HTTP METHOD

* **GET**: được sử dụng để lấy thông tin từ sever theo url đã cung cấp.&#x20;
* **HEAD**: giống với GET nhưng response trả về không có body, chỉ có header&#x20;
* **POST**: gửi thông tin tới sever thông qua các biểu mẫu http( đăng kí chả hạn..)&#x20;
* **PUT:** được sử dụng để cập nhật toàn bộ tài nguyên trên server.
* **PATCH**: chỉ cập nhật (thay đổi) một phần thông tin của tài nguyên đã có trên server.DELETE: xóa tài nguyên trên server. **CONNECT**: thiết lập một kết nối tới server theo URI.&#x20;
* **OPTIONS**: mô tả các tùy chọn giao tiếp cho resource.&#x20;
* **TRACE**: thực hiện một bài test loop - back theo đường dẫn đến resource.



* Phương thức `GET` được sử dụng khi bạn muốn lấy dữ liệu từ server. Yêu cầu `GET` gửi thông tin từ client tới server qua URL của request. Thông thường, các tham số truyền vào `GET` được thêm vào URL dưới dạng query string(dấu "?" trong url). Phương thức này thường được sử dụng để lấy dữ liệu một cách nhanh chóng và dễ dàng, tuy nhiên, nó có một giới hạn về kích thước của URL, do đó nó không thích hợp để gửi dữ liệu lớn.
* Phương thức `POST`, ngược lại, được sử dụng khi bạn muốn gửi dữ liệu từ client tới server để thực hiện một hành động cụ thể như tạo mới, cập nhật hoặc xóa dữ liệu. Yêu cầu `POST` gửi thông tin từ client tới server qua phần thân (body) của request. Phương thức này cho phép gửi dữ liệu lớn hơn và bảo mật hơn so với `GET`. Tuy nhiên, yêu cầu `POST` sẽ tốn nhiều thời gian hơn để thực hiện do phải gửi dữ liệu tới server thông qua phần thân của request.
*   Trong Swift, `PATCH` và `PUT` là hai phương thức khác nhau được sử dụng để cập nhật tài nguyên trên server. Dưới đây là sự khác biệt giữa `PATCH` và `PUT`:

    1. Mục đích sử dụng:
    2. `PUT` được sử dụng để cập nhật toàn bộ tài nguyên trên server.
    3. `PATCH` được sử dụng để cập nhật một phần thông tin của tài nguyên trên server.
    4. Nội dung của yêu cầu:
    5. Trong yêu cầu `PUT`, toàn bộ nội dung (data) của tài nguyên mới cần được gửi đến server để thay thế toàn bộ thông tin của tài nguyên đã có trên server.
    6. Trong yêu cầu `PATCH`, chỉ những thông tin cần được cập nhật (thay đổi) trên tài nguyên đã có trên server mới được gửi đi.
    7. Tác động lên tài nguyên trên server:
    8. Yêu cầu `PUT` thay thế toàn bộ thông tin của tài nguyên đã có trên server bằng thông tin mới.
    9. Yêu cầu `PATCH` chỉ cập nhật (thay đổi) một phần thông tin của tài nguyên đã có trên server.
    10. Thứ tự thực hiện yêu cầu:
    11. Yêu cầu `PUT` được thực hiện theo thứ tự xóa (delete) và tạo mới (create) tài nguyên mới trên server.
    12. Yêu cầu `PATCH` được thực hiện theo thứ tự cập nhật (update) thông tin của tài nguyên trên server.

    Vì vậy, khi sử dụng các phương thức `PUT` và `PATCH`, cần phải xác định rõ mục đích và nội dung của yêu cầu, để đảm bảo gửi đúng yêu cầu tới server và thực hiện chức năng mong muốn. Nếu muốn cập nhật toàn bộ thông tin của tài nguyên, thì nên sử dụng `PUT`, còn nếu chỉ muốn cập nhật một phần thông tin của tài nguyên thì nên sử dụng `PATCH`.
