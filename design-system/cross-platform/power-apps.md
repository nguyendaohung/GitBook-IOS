# Microsoft Power Apps và Power Fx

### Định nghĩa

Power Apps là low-code platform. Canvas App thiết kế giao diện/control và formula linh hoạt; Model-driven App dựa trên Dataverse data model/process. Data source có thể Dataverse, SharePoint, SQL hoặc connector khác; CV không xác định loại app/source.

### Power Fx

```text
Filter(Materials, Quantity < MinimumQuantity)
LookUp(Materials, ID = selectedID)
Patch(Materials, Defaults(Materials), {Name: inputName.Text, Quantity: 10})
```

Ví dụ minh họa tên table/control; kiểm tra schema và locale formula separator thực tế.

* Filter trả table, LookUp trả record/giá trị đầu tiên, Patch tạo/sửa record, Collect/ClearCollect quản lý collection, Set global variable, UpdateContext screen context.
* Gallery, Form, validation, SubmitForm/OnSuccess/OnFailure; navigation và loading/error.
* Delegation: connector chỉ thực thi server-side một số operation; non-delegable formula có thể chỉ xử lý tập record giới hạn → kết quả thiếu. Chọn delegable filter/index, không tăng limit để che lỗi.
* Database: primary key, foreign key, relationship, quantity validation, concurrency update; backend/Dataverse security role và connector permission, không dựa vào hidden control.
* Development/test/prod environment, solution, connection reference, documentation, versioning và UAT.

### Q&A

**Q: Canvas vs Model-driven?** Canvas UI linh hoạt; model-driven theo Dataverse model/process.

**Q: Filter vs LookUp?** Filter trả nhiều record, LookUp trả record đầu tiên phù hợp.

**Q: Vì sao dữ liệu filter thiếu?** Delegation limit/connector support; chuyển query delegable/server-side.

**Liên hệ:** Power Apps Planning – supply planning, raw materials, quantity/storage, database/UI design và tài liệu. Cần xác nhận data source, loại app, formula thực tế trước phỏng vấn.
