# Pass Data

Thường dùng:

* Delegate: Sử dụng khi trong cần truyền nhiều dữ liệu/nhiều action ⇒ giúp code clean
* Closure: Sử dụng khi cần truyền ít dữ liệu/action ⇒ sử dụng nhanh chóng, dễ dàng
* Notification: Sử dụng khi 2 màn hình không liên quan gì nhau hoặc cần truyền dữ liệu sang nhiều nơi trong ứng dụng

Ngoài ra còn có:&#x20;

1. Sử dụng Segue: Sử dụng segue để chuyển từ một màn hình đến một màn hình khác, bạn có thể truyền dữ liệu từ màn hình nguồn đến màn hình đích bằng cách sử dụng thuộc tính `prepare(for:sender:)`.
