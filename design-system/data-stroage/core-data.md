# Core Data

### Định nghĩa

Core Data là framework quản lý object graph và persistence, không chỉ là SQLite wrapper. NSPersistentContainer dựng stack gồm managed object model, persistent store coordinator và context.

* NSManagedObject: object thuộc context, faulting tải thuộc tính khi cần.
* NSManagedObjectContext: unit of work, track change, save/rollback.
* NSFetchRequest: predicate, sort descriptor, fetchBatchSize.
* NSFetchedResultsController: theo dõi thay đổi để cập nhật UITableView/UICollectionView.
* Migration: lightweight nếu model thay đổi tương thích; mapping/custom migration cho thay đổi phức tạp. Test migration từ dữ liệu phiên bản cũ.

### Ví dụ concurrency

```swift
import CoreData

func importPlaces(container: NSPersistentContainer, names: [String]) {
    container.performBackgroundTask { context in
        do {
            for name in names {
                let place = NSEntityDescription.insertNewObject(
                    forEntityName: "Place", into: context
                )
                place.setValue(name, forKey: "name")
            }
            try context.save()
        } catch {
            context.rollback()
            // Báo lỗi cho caller / logging.
        }
    }
}
```

Ví dụ cần model Place và attribute name trong .xcdatamodeld. Sau loadPersistentStores, cấu hình viewContext.automaticallyMergesChangesFromParent = true nếu phù hợp; đặt mergePolicy theo nghiệp vụ.

### Lưu ý

* Main queue context chỉ trên queue của nó; private queue context dùng perform/performAndWait. Không truyền NSManagedObject giữa queue; truyền objectID (permanent ID nếu cần), DTO/value rồi fetch lại.
* Batch insert/delete nhanh nhưng thay đổi có thể không tự merge vào context đang giữ object; merge object IDs hoặc refresh.
* Fetch batch, predicate/index hợp lý, tránh N+1 relationship fault; đo bằng Instruments/SQL debug khi phát triển.
* Dữ liệu cache có version, TTL, migration và chính sách dọn dẹp. Save fail phải rollback/retry phù hợp; không xóa store production để chữa lỗi migration.

### Q&A

**Q: CoreData vs SQLite?** Core Data quản lý graph, change tracking, relationships; SQLite là engine lưu trữ/query.

**Q: Vì sao crash khi dùng NSManagedObject qua thread khác?** Object gắn context/queue; dùng objectID để resolve ở context khác.

**Liên hệ:** Map4D cache địa chỉ/dữ liệu địa lý; ví dụ là kiến trúc tham khảo, không xác nhận schema dự án.

## Core Data chuyên sâu

### Khởi tạo stack và CRUD

```swift
import CoreData

final class Persistence {
    let container: NSPersistentContainer

    init(inMemory: Bool = false) {
        container = NSPersistentContainer(name: "AppModel")
        if inMemory {
            container.persistentStoreDescriptions.first?.type = NSInMemoryStoreType
        }
    }

    func load(completion: @escaping (Error?) -> Void) {
        container.loadPersistentStores { _, error in
            if error == nil {
                self.container.viewContext.automaticallyMergesChangesFromParent = true
            }
            completion(error)
        }
    }
}
```

AppModel là .xcdatamodeld chứa entity Place với name/id. Chỉ dùng context sau khi loadPersistentStores thành công; lỗi load/migration phải báo recovery, không tự xóa DB. Ví dụ đọc trên queue context:

```swift
func placeNames(context: NSManagedObjectContext, search: String)
    async throws -> [String] {
    try await context.perform {
        let request = NSFetchRequest<NSManagedObject>(entityName: "Place")
        request.predicate = NSPredicate(format: "name CONTAINS[cd] %@", search)
        request.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]
        request.fetchBatchSize = 50
        return try context.fetch(request).compactMap {
            $0.value(forKey: "name") as? String
        }
    }
}
```

Predicate parameterized tránh format injection; DTO String trả qua queue boundary an toàn hơn managed object. Với update/delete, fetch theo id, mutate context, save; rollback khi lỗi. Fetch batch size giảm materialization, không đồng nghĩa pagination; dùng fetchLimit/cursor phù hợp.

### Relationships, faulting và migration

* To-one/to-many relationship, inverse giúp graph nhất quán; delete rule nullify/cascade/deny/no action. Cascade dùng cẩn thận nếu object được share.
* Fault lazy load thuộc tính/relationship; truy cập hàng nghìn relationship có thể N+1. Prefetch relationshipKeyPaths khi phù hợp.
* Unique constraint + merge policy để dedup import; merge policy không thay nghiệp vụ conflict resolution.
* Lightweight migration cho thay đổi model tương thích (thêm optional/default, rename với renamingIdentifier); test store từ version cũ. Thay data type/relationship phức tạp có thể cần mapping/custom.
* Parent-child context save child chỉ đẩy thay đổi lên parent; parent phải save tới persistent store. Background context từ container thường save tới store; viewContext merge.

### So sánh Core Data, Realm, SQLite, UserDefaults

| Công nghệ | Dữ liệu | Chọn khi |
| --- | --- | --- |
| Core Data | Object graph, relations | Native iOS, persistence phức tạp, context |
| Realm | Object DB, live query | Đội dùng Realm, transaction API |
| SQLite | SQL rows/query | Cần kiểm soát SQL/index trực tiếp |
| UserDefaults | Key/value nhỏ | Settings, không phải database |

### Q&A bổ sung

**Q: Tại sao objectID temporary?** Object mới chưa có permanent identity; obtainPermanentIDs hoặc save trước khi truyền sang context khác.

**Q: Batch delete không cập nhật UI?** Nó thao tác store, context có object cũ; merge object IDs hoặc refresh/reset phù hợp.

**Bài tập:** Tạo Place model, import background 100 records, fetch sorted/search, test in-memory và migration từ model v1.
