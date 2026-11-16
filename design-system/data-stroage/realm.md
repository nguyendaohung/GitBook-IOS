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

## Realm CRUD và reactive query

```swift
import RealmSwift

func favoriteTitles(search: String) throws -> [String] {
    let realm = try Realm()
    let results = realm.objects(Favorite.self)
        .filter("title CONTAINS[c] %@", search)
        .sorted(byKeyPath: "title")
    return results.map(\.title)
}

func removeFavorite(id: String) throws {
    let realm = try Realm()
    guard let item = realm.object(ofType: Favorite.self, forPrimaryKey: id) else {
        return
    }
    try realm.write { realm.delete(item) }
}
```

Results là live collection, không phải array snapshot. Observe collection notification để nhận initial/update/error, giữ NotificationToken; invalidate khi màn hình không còn quan sát. Không giữ object đã bị delete; check isInvalidated khi cần. Transaction write atomic trong Realm, không mutate managed object ngoài write transaction.

### Migration và thread safety

```swift
var configuration = Realm.Configuration.defaultConfiguration
configuration.schemaVersion = 2
configuration.migrationBlock = { migration, oldVersion in
    if oldVersion < 2 {
        // Minh họa rename một field khi schema v2 đã đổi tên.
        migration.renameProperty(onType: "Favorite", from: "name", to: "title")
    }
}
Realm.Configuration.defaultConfiguration = configuration
```

Chỉ dùng rename nếu schema v1 thật sự có name và v2 có title; test file cũ trước release. ThreadSafeReference resolve một lần tại Realm đích; frozen object đọc snapshot, không mutate; thaw nếu phù hợp. Prefer chuyển primary key/value DTO qua concurrency boundary.

**Q: Realm nhanh hơn Core Data?** Không mặc định. Benchmark workload, query/index, object graph, writes, migration, memory và dependency maintenance.

**Bài tập:** CRUD favorites, observe thay đổi, migration v1→v2, test duplicate primary key và background query.

## Câu hỏi phỏng vấn tình huống

**Q: Realm Results là snapshot?** Live Results cập nhật theo Realm, map sang value array khi cần snapshot/cross-actor; giữ notification token đúng lifetime.

**Q: Write transaction có mutate ngoài transaction?** Managed object cần write transaction, atomic commit; batch và error handling, tránh blocking main.

**Q: Frozen object vs ThreadSafeReference?** Frozen read-only snapshot, ThreadSafeReference resolve một lần tại Realm đích; cân nhắc primary key/DTO đơn giản hơn.
