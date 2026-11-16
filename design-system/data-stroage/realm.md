# Realm

### Định nghĩa

Realm là object database bên thứ ba, hỗ trợ query, relationship và reactive change notification. Khác Core Data, dùng model Object và transaction write của Realm.

```swift
import RealmSwift

final class Favorite: Object {
    @Persisted(primaryKey: true) var id: String
    @Persisted var title: String
}

func saveFavorite(id: String, title: String) throws {
    let realm = try Realm()
    try realm.write {
        let item = Favorite()
        item.id = id
        item.title = title
        realm.add(item, update: .modified)
    }
}
```

### Lưu ý

* Managed object và Realm instance có thread/actor confinement; truyền primary key, unmanaged value, frozen object hoặc ThreadSafeReference phù hợp version SDK.
* Write transaction có thể block nếu nhiều writer; batch hợp lý. Live Results tự cập nhật, giữ notification token và invalidate khi không cần.
* Schema version và migrationBlock; test upgrade với store cũ. Encryption key lưu Keychain, không hardcode.
* Realm local database không đồng nghĩa cloud sync; chọn giải pháp sync riêng và kiểm tra phiên bản SDK.

### Q&A

**Q: Realm vs CoreData?** Realm API object database trực tiếp; Core Data tích hợp hệ sinh thái Apple, context, object graph. Chọn theo migration, dependency, team và profiling.

**Q: Có truyền object Realm qua background thread không?** Không truyền managed live object tùy ý; resolve tại thread/actor đích.

Realm là kiến thức nền trong GitBook; CV không ghi dự án nào dùng Realm.
