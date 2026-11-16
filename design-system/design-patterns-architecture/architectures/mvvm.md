# MVVM

### Định nghĩa

MVVM là viết tắt của Model - View - ViewModel, đây là mô hình hỗ trợ two-way data binding giữa View và View Model​. Là 1 phiên bản nâng cấp của MVC.  Mục đích là giúp khả năng kiểm thử và giảm công việc ở Controller.&#x20;

*   Controller: Controller được giảm công việc và xem như là View. Chỉ thực hiện những thứ liên quan đến UI — Show/get thông tin, một phần của view layer, hiển thị dữ liệu lên giao diện,truyền sự kiện người dùng về cho ViewModel xử lí, lấy dữ liệu từ ViewModel và update lại giao diện​

    ​
* View: Nhận sự kiện của người dùng tác động, rồi truyền về controller (không phải truyền trực tiếp cho view model)​
* Model: Nó giống với model trong mô hình MVC, Model đại diện cho dữ liệu đơn giản, Nó đơn giản nắm giữ dữ liệu và không làm bất cứ thứ gì liên quan tới logic,nó được sử dụng bởi ViewModel và update bất cứ khi nào ViewModel gửi update mới.​
* ViewModel: Nó là ánh xạ của View về mặt dữ liệu. Là nơi xử lý logic, những business trong app. Nhận thông tin từ VC, xử lý tất cả những thông tin này và gửi lại nó cho VC, ViewModel là thành phần chính của mô hình này. ViewModel không biết View là gì và những gì View làm. Điều này làm cho mô hình này dễ kiểm tra hơn và loại bỏ sự phức tạp từ view.​

Một số nguyên tắc phụ trợ khi áp dụng mô hình MVVM:​

* View nhận sự kiện của người dùng tác động, rồi truyền về controller (không phải truyền trực tiếp cho view model)​
* View model không chứa các xử lý về UI (không import UIKit)​
* Controller chịu trách nhiệm update dữ liệu lên view (không phải view model)​
* Controller nhận các tín hiệu update dữ liệu từ view model bằng 2 cách:​
  * Callback (closure & protocol)​
  * Observer (notification, KOV)​
* Một view model có thể được dùng cho nhiều controller (view model là sự ánh xạ của 1 đối tượng của model lên view, trường hợp nhiều controller dùng chung 1 loại đối tượng model và tương đồng về mặc hiển thị, thì có thể dùng chung view model)​

### Ưu điểm

* Là phiên bản nâng cấp của MVC, nên vẫn duy trì cấu trúc của MVC. Có nghĩa là ta có thể migrate 1 cái app có sẵn từ mô hình MVC sang mô hình MVVM
* Giảm khối lượng code chứa trong Controller
* Đưa các xử lý logic + business vào ViewModel ⇒ Code trở nên dễ hiểu, dễ maintain hơn
* Có khả năng kiểm thử.

### Nhược điểm

* Là mô hình khá phức tạp đối với người bắt đầu. Không phù hợp với những dự án nhỏ.
* Vì tương tự với MVC, nên không thể giải quyết triệt để vấn đề phình code của MVC. Giờ đây code có thể phình to ra ở ViewModel
