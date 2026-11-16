# 🚌 Data Stroage

* [Core Data](core-data.md)
* [SwiftData](swift-data.md)
* [Realm](realm.md)
* [Keychain Access](keychain-access.md)
* [FileManager, UserDefaults và App Groups](file-cache.md)

## Câu hỏi phỏng vấn tổng quan

**Q: Chọn UserDefaults, Keychain, FileManager hay database?** Settings nhỏ → UserDefaults; secret → Keychain; files/cache → FileManager; query/relationships/migration → Core Data/SwiftData/Realm.

**Q: SwiftData vs Core Data vs Realm?** SwiftData iOS17 Swift-native/SwiftUI integration; Core Data native trưởng thành/OS cũ; Realm object DB bên thứ ba. Đo workload/migration.

**Q: Concurrency của persistence?** Context/actor/thread confinement; DTO/stable ID qua boundary, resolve ở context đích, serialize writes và merge conflict.
