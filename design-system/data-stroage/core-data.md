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
