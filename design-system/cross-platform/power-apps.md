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

## Power Apps chuyên sâu

### Data model và app type

Canvas app linh hoạt layout/formula, Model-driven app dựa Dataverse tables/forms/views/business process. Chọn theo nghiệp vụ, connector, licensing và security. Ví dụ planning: Material(id, name, unit), Stock(materialID, locationID, quantity), Plan(id, materialID, requiredQty, dueDate). Primary/foreign key, relationship, validation và audit. Không dùng display name làm ID; quantity phải validate server-side.

### CRUD bằng Power Fx (Canvas)

```text
// Gallery.Items
Filter(Materials, StartsWith(Name, SearchBox.Text))

// Save button.OnSelect
If(
    Value(QuantityInput.Text) < 0,
    Notify("Quantity must be non-negative", NotificationType.Error),
    IfError(
        Patch(
            Materials,
            Defaults(Materials),
            {Name: NameInput.Text, Quantity: Value(QuantityInput.Text)}
        ),
        Notify("Save failed", NotificationType.Error),
        Notify("Saved", NotificationType.Success)
    )
)
```

Ví dụ minh họa table/control và locale dấu phân cách; field/schema tùy connector. Production validate empty/numeric/permissions, disable duplicate submit, handle server error. Edit form: EditForm/NewForm, SubmitForm, OnSuccess/OnFailure, ResetForm; Patch cho update subset/multi-record theo connector. Gallery chọn record, LookUp lấy một record, Filter trả table, SortByColumns sắp xếp.

### Delegation và performance

Power Apps có delegation: connector thực thi query server-side nếu hỗ trợ; non-delegable query chỉ xử lý giới hạn local records, có thể trả kết quả sai/thiếu. Không dùng ClearCollect toàn bảng lớn để tránh delegation. Chọn delegable predicates theo connector, index/keys phù hợp, paging/query server. Monitor tool xem network/formula/performance. Concurrent updates stock cần backend concurrency/transaction, không chỉ client tính quantity = old + delta.

### Security và ALM

Dataverse security role/record access hoặc connector permission theo source, không dùng hidden control thay authorization. Connection credentials do platform quản lý, không hardcode secret. Environment dev/test/prod, solution, connection reference/environment variables, export/import, UAT và versioning. Backup/restore và migration schema theo source. Power Automate có thể orchestrate workflow/approval nếu dự án dùng; không mặc định.

**Q: Patch vs SubmitForm?** Patch direct record update, SubmitForm dùng form validation/lifecycle.

**Q: Delegation warning có nghiêm trọng?** Có thể kết quả thiếu ở dataset lớn; kiểm tra connector và chuyển logic server.

**Q: Canvas vs Model-driven?** Canvas custom UI; Model-driven data/process-driven với Dataverse.

**Bài tập:** Material gallery + search delegable, create/edit form, validation, test permission/duplicate/large dataset; viết sơ đồ data model và UAT cases.
