# 😁 GIT

**Merge và rebase** là hai phương pháp khác nhau để tích hợp các thay đổi từ các nhánh khác nhau trong quản lý mã nguồn.

**Merge (kết hợp)** là quá trình thêm các thay đổi từ một nhánh vào nhánh khác. Với merge, các thay đổi từ nhánh nguồn được giữ nguyên và thêm vào nhánh đích dưới dạng một commit mới. Điều này có nghĩa là lịch sử của các nhánh được giữ nguyên và hiển thị rõ ràng trên đồ thị git.

**Rebase (sắp xếp lại)** là quá trình di chuyển các thay đổi từ một nhánh và đặt chúng lên đỉnh của một nhánh khác. Với rebase, các thay đổi từ nhánh nguồn được đặt lên đỉnh của nhánh đích dưới dạng một chuỗi commit mới. Điều này có nghĩa là lịch sử của các nhánh được sắp xếp lại một cách tuần tự và như vậy, sẽ không có commit gộp riêng biệt như trong trường hợp của merge. Rebase sẽ move base lên commit mới nhất của nhánh cha

Vì các thay đổi được sắp xếp lại trong rebase, việc giải quyết xung đột có thể khó hơn so với merge, vì các thay đổi được áp dụng theo thứ tự và không được tách ra như trong trường hợp của merge.

**Sử dụng Merge khi:**

* Cần tích hợp các thay đổi từ nhiều người dùng hoặc nhánh trong một thời điểm cụ thể.
* Các nhánh cần được giữ nguyên lịch sử riêng biệt.
* Có xung đột giữa các commit và cần giải quyết thủ công hoặc sử dụng công cụ hỗ trợ để giải quyết.
* Không cần xóa các commit không cần thiết hoặc lịch sử không cần thiết.

**Sử dụng Rebase khi:**

* Cần tạo một lịch sử commit liền mạch và không bị gián đoạn.
* Muốn giữ các commit của nhánh nguồn nhưng muốn chúng được đặt lên đỉnh của nhánh đích.
* Không có xung đột hoặc ít xung đột giữa các commit.
* Cần xóa các commit không cần thiết hoặc lịch sử không cần thiết.
