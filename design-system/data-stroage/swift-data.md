# SwiftData

## 1. Định nghĩa

SwiftData là framework persistence của Apple, giới thiệu tại WWDC 2023 (iOS 17+, macOS 14+, watchOS 10+). SwiftData xây dựng trên nền tảng công nghệ Core Data, kết hợp Swift macro và Observation để mô tả model bằng Swift code. Một app SwiftData mới không cần tạo file .xcdatamodeld như cách dùng Core Data truyền thống.

SwiftData quản lý object graph, persistent storage, query, relationships và migration. Framework dùng được với SwiftUI, UIKit hoặc repository không phụ thuộc UI. Không phải database server, không thay REST API, không dùng thay Keychain để lưu secret.

### Khi nào sử dụng?

* App mới iOS 17+, local notes, favorites, offline catalog, dữ liệu nghiệp vụ có relationship.
* SwiftUI cần query tự cập nhật giao diện với ít boilerplate.
* UIKit iOS 17+ có thể dùng ModelContext trực tiếp, không cần chuyển UI sang SwiftUI.
* App Core Data production hoặc hỗ trợ iOS dưới 17: cân nhắc compatibility, migration, tooling và tính năng trước khi chuyển. Không migrate chỉ vì framework mới.
* Settings nhỏ: UserDefaults; token/password: Keychain; file/ảnh lớn: FileManager/cache, lưu metadata trong SwiftData khi phù hợp.

## 2. So sánh SwiftData, Core Data và Realm

| Tiêu chí | SwiftData | Core Data | Realm |
| --- | --- | --- | --- |
| Model | Swift class + @Model | NSManagedObject | Object + @Persisted |
| Schema | Swift code, VersionedSchema | .xcdatamodeld hoặc programmatic model | Swift model, schemaVersion |
| Stack | ModelContainer | NSPersistentContainer | Realm configuration/instance |
| Unit of work | ModelContext | NSManagedObjectContext | Realm write transaction |
| Query | FetchDescriptor, #Predicate, @Query | NSFetchRequest, NSPredicate | Results, filter/query |
| Observation | @Query, Observation | NSFetchedResultsController, @FetchRequest | Live Results, notification token |
| Write | insert/delete/mutate + save/autosave | insert/delete/mutate + save | write transaction |
| Concurrency | ModelActor, context isolation | Context queue, perform | Thread/actor confinement tùy SDK |
| Migration | VersionedSchema, SchemaMigrationPlan | Versioned model, lightweight/custom | migrationBlock |
| Dependency | Apple framework | Apple framework | SDK bên thứ ba |
| iOS | 17+ | Hỗ trợ OS cũ hơn | Tùy phiên bản SDK |

Core Data có tooling, batch/merge/history APIs trưởng thành. SwiftData giảm boilerplate và tích hợp SwiftUI tốt nhưng không có đầy đủ mọi API tương đương Core Data trên iOS 17. Realm dùng object DB với live query/write transaction riêng. Không khẳng định framework nào luôn nhanh hơn: benchmark workload, dataset, memory và migration.

## 3. Cơ chế hoạt động

```text
@Model class -> Schema -> ModelContainer -> persistent store
                              |
                         ModelContext
                   insert / fetch / delete / save
                              |
                   @Query / DTO / repository
                              |
                         SwiftUI / UIKit
```

* @Model: macro biến Swift class thành persistent model, tích hợp Observation; model là reference type, không phải struct.
* Schema: mô tả model, attributes, relationships.
* ModelContainer: quản lý schema, storage configuration và migration; thường giữ theo app/store scope, không tạo lại mỗi màn hình.
* ModelConfiguration: in-memory/on-disk, store URL, app group/CloudKit phù hợp. In-memory mất dữ liệu khi container kết thúc.
* ModelContext: fetch, insert, theo dõi mutation, delete, save, rollback. mainContext thuộc MainActor.
* FetchDescriptor<T>: predicate, sort, fetchLimit/fetchOffset. context.fetch trả kết quả để caller quản lý.
* @Query: property wrapper SwiftUI, fetch và phản ánh thay đổi của model; cần container/context trong environment.
* PersistentIdentifier: identity record trong store, khác business ID/remote ID.

Main context SwiftUI thường bật autosave. Autosave không cam kết mutation đã ghi disk ngay khi gán property; gọi try context.save() khi cần biết commit thành công. Context tạo thủ công phải quản lý save theo nhu cầu. Save local không đồng nghĩa server sync thành công.

## 4. Ví dụ Model và CRUD

Ví dụ app Favorites, target iOS 17+. Các snippet mục 4–8 sử dụng model Favorite này; không cần dependency bên thứ ba.

```swift
import Foundation
import SwiftData

@Model
final class Favorite {
    @Attribute(.unique) var remoteID: String
    var title: String
    var isPinned: Bool
    var createdAt: Date

    init(remoteID: String, title: String,
         isPinned: Bool = false, createdAt: Date = .now) {
        self.remoteID = remoteID
        self.title = title
        self.isPinned = isPinned
        self.createdAt = createdAt
    }
}
```

@Attribute(.unique) yêu cầu uniqueness trên attribute trong schema hỗ trợ; insert trùng có thể dẫn đến upsert. Không thay business merge/conflict policy. Model này dành cho local-only store; CloudKit-backed schema không hỗ trợ unique constraints như trên.

### Khởi tạo container

```swift
import SwiftData

@MainActor
func makeContainer(inMemory: Bool = false) throws -> ModelContainer {
    let schema = Schema([Favorite.self])
    let configuration = ModelConfiguration(
        schema: schema,
        isStoredInMemoryOnly: inMemory,
        cloudKitDatabase: .none
    )
    return try ModelContainer(for: schema, configurations: [configuration])
}
```

App root tạo và giữ container, inject vào repository/UI. Nếu lỗi mở store hoặc migration, hiển thị recovery/error, log đã redact; không tự xóa database người dùng. Preview/test dùng inMemory: true để không ghi store thật.

### CRUD qua ModelContext

```swift
import Foundation
import SwiftData

@MainActor
final class FavoritesStore {
    private let context: ModelContext

    init(context: ModelContext) {
        self.context = context
    }

    func all() throws -> [Favorite] {
        try context.fetch(FetchDescriptor<Favorite>(
            sortBy: [SortDescriptor(\.createdAt, order: .reverse)]
        ))
    }

    func find(id: String) throws -> Favorite? {
        var descriptor = FetchDescriptor<Favorite>(
            predicate: #Predicate { $0.remoteID == id }
        )
        descriptor.fetchLimit = 1
        return try context.fetch(descriptor).first
    }

    func upsert(id: String, title: String) throws {
        if let item = try find(id: id) {
            item.title = title
        } else {
            context.insert(Favorite(remoteID: id, title: title))
        }
        try context.save()
    }

    func togglePinned(id: String) throws {
        guard let item = try find(id: id) else { return }
        item.isPinned.toggle()
        try context.save()
    }

    func delete(id: String) throws {
        guard let item = try find(id: id) else { return }
        context.delete(item)
        try context.save()
    }
}
```

SwiftData theo dõi property mutation trên model thuộc context, không cần update() riêng. Upsert tìm theo remoteID để refresh title mà không ghi đè isPinned local. Khi save fail, xử lý error/retry/rollback theo nghiệp vụ; context.rollback() hoàn tác tất cả unsaved changes trong context, không chỉ một operation. Nếu cần isolation cho transaction, dùng context riêng và không rollback main context bừa bãi.

## 5. SwiftUI + SwiftData: Favorites List

```swift
import SwiftUI
import SwiftData

@main
struct FavoritesApp: App {
    var body: some Scene {
        WindowGroup {
            FavoritesView()
        }
        .modelContainer(for: Favorite.self)
    }
}

struct FavoritesView: View {
    @Environment(\.modelContext) private var context
    @Query(sort: \Favorite.createdAt, order: .reverse)
    private var favorites: [Favorite]

    @State private var title = ""
    @State private var errorMessage: String?

    var body: some View {
        NavigationStack {
            List {
                Section("Add favorite") {
                    TextField("Title", text: $title)
                    Button("Add", action: add)
                        .disabled(title.trimmingCharacters(
                            in: .whitespacesAndNewlines).isEmpty)
                }

                Section("Favorites") {
                    ForEach(favorites) { item in
                        NavigationLink {
                            FavoriteEditor(item: item)
                        } label: {
                            Label(item.title,
                                  systemImage: item.isPinned ? "pin.fill" : "star")
                        }
                    }
                    .onDelete(perform: remove)
                }
            }
            .navigationTitle("Favorites")
            .alert("Save failed", isPresented: Binding(
                get: { errorMessage != nil },
                set: { if !$0 { errorMessage = nil } }
            )) {
                Button("OK", role: .cancel) { errorMessage = nil }
            } message: {
                Text(errorMessage ?? "")
            }
        }
    }

    private func add() {
        let name = title.trimmingCharacters(in: .whitespacesAndNewlines)
        guard !name.isEmpty else { return }
        context.insert(Favorite(remoteID: UUID().uuidString, title: name))
        do {
            try context.save()
            title = ""
        } catch {
            errorMessage = "Could not save favorite"
        }
    }

    private func remove(at offsets: IndexSet) {
        let items = offsets.map { favorites[$0] }
        items.forEach { context.delete($0) }
        do {
            try context.save()
        } catch {
            errorMessage = "Could not delete favorite"
        }
    }
}

struct FavoriteEditor: View {
    @Bindable var item: Favorite

    var body: some View {
        Form {
            TextField("Title", text: $item.title)
            Toggle("Pinned", isOn: $item.isPinned)
        }
    }
}
```

Giải thích:

1. .modelContainer(for:) cung cấp container/context cho view tree. Production có thể inject container tạo bằng makeContainer để kiểm soát configuration và migration.
2. @Query theo dõi collection; insert/delete/edit cập nhật SwiftUI theo dependency, không gọi reloadData().
3. @Bindable tạo binding tới property @Model để TextField/Toggle chỉnh trực tiếp.
4. Editor là live editing, dùng autosave. Nếu cần Save/Cancel, dùng @State draft rồi validate và ghi khi Save; không bind trực tiếp model rồi kỳ vọng Cancel tự hoàn tác.
5. Add/Delete hiển thị lỗi save; production phải quyết định giữ pending changes để retry, undo/rollback riêng operation hoặc reload theo nghiệp vụ.
6. ForEach dùng model identity; remoteID dùng mapping backend. Không dùng index khi list reorder.

### @Query search/filter động

```swift
import SwiftUI
import SwiftData

struct FilteredFavorites: View {
    @Query private var items: [Favorite]

    init(search: String, pinnedOnly: Bool) {
        let term = search.trimmingCharacters(in: .whitespacesAndNewlines)
        _items = Query(
            filter: #Predicate<Favorite> { item in
                (term.isEmpty || item.title.localizedStandardContains(term))
                    && (!pinnedOnly || item.isPinned)
            },
            sort: [SortDescriptor(\.createdAt, order: .reverse)]
        )
    }

    var body: some View {
        List(items) { item in Text(item.title) }
    }
}

struct FavoritesSearchView: View {
    @State private var search = ""
    @State private var pinnedOnly = false

    var body: some View {
        NavigationStack {
            VStack {
                Toggle("Pinned only", isOn: $pinnedOnly)
                FilteredFavorites(search: search, pinnedOnly: pinnedOnly)
            }
            .searchable(text: $search)
        }
    }
}
```

Parent state đổi tạo query configuration mới. #Predicate tạo expression để store translate; không phải mọi Swift method/custom function đều translate được. Nếu predicate phức tạp fail trên OS target, đơn giản hóa, kiểm tra availability và test disk store thật. Debounce search khi dataset lớn, tránh fetch toàn bộ rồi filter trong body.

## 6. SwiftData với UIKit

UIKit không có @Query tự render UITableView. Inject container/context/repository, fetch và render snapshot; refresh theo completion hoặc domain event khi dữ liệu thay đổi.

```swift
import UIKit
import SwiftData

@MainActor
final class FavoritesViewController: UITableViewController {
    private let store: FavoritesStore
    private var items: [Favorite] = []

    init(container: ModelContainer) {
        store = FavoritesStore(context: container.mainContext)
        super.init(style: .insetGrouped)
    }

    required init?(coder: NSCoder) { nil }

    override func viewDidLoad() {
        super.viewDidLoad()
        title = "Favorites"
        tableView.register(UITableViewCell.self,
                           forCellReuseIdentifier: "Favorite")
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        reloadFavorites()
    }

    private func reloadFavorites() {
        do {
            items = try store.all()
            tableView.reloadData()
        } catch {
            showError()
        }
    }

    private func showError() {
        let alert = UIAlertController(title: "Data error",
                                      message: "Please try again.",
                                      preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "OK", style: .default))
        present(alert, animated: true)
    }

    override func tableView(_ tableView: UITableView,
                            numberOfRowsInSection section: Int) -> Int {
        items.count
    }

    override func tableView(_ tableView: UITableView,
                            cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(
            withIdentifier: "Favorite", for: indexPath
        )
        var content = cell.defaultContentConfiguration()
        content.text = items[indexPath.row].title
        cell.contentConfiguration = content
        return cell
    }

    override func tableView(_ tableView: UITableView,
                            commit editingStyle: UITableViewCell.EditingStyle,
                            forRowAt indexPath: IndexPath) {
        guard editingStyle == .delete else { return }
        do {
            try store.delete(id: items[indexPath.row].remoteID)
            reloadFavorites()
        } catch {
            showError()
            // Reconcile pending changes according to application policy.
        }
    }
}
```

| SwiftUI | UIKit |
| --- | --- |
| @Environment modelContext | Inject container/context/repository |
| @Query | context.fetch + state/snapshot |
| @Bindable | TextField delegate/target-action → update |
| List phản ánh thay đổi | reloadData/diffable snapshot |
| View identity | VC/cell lifecycle |

Production UIKit có thể dùng value DTO + UITableViewDiffableDataSource với stable ID. Không giả định background save tự reload table; refresh sau import và reconcile theo OS/context. Có thể dùng UIHostingController nhúng FavoritesView vào UIKit, truyền cùng container bằng .modelContainer(container). Không giữ model reference qua context/actor khác.

## 7. Relationships, delete rule và attribute

### Category – FavoriteItem (một-nhiều)

Ví dụ model độc lập với Favorite phía trên; đăng ký cả hai model trong container.

```swift
import SwiftData

@Model
final class Category {
    var name: String

    @Relationship(deleteRule: .cascade, inverse: \FavoriteItem.category)
    var items: [FavoriteItem] = []

    init(name: String) { self.name = name }
}

@Model
final class FavoriteItem {
    var title: String
    var category: Category?

    init(title: String, category: Category? = nil) {
        self.title = title
        self.category = category
    }
}

@MainActor
func addCategory(context: ModelContext) throws {
    let category = Category(name: "Coffee")
    category.items.append(FavoriteItem(title: "Latte"))
    category.items.append(FavoriteItem(title: "Espresso"))
    context.insert(category)
    try context.save()
}
```

SwiftData quản lý graph và inverse relationship; insert root có thể insert related models mới. Giữ graph cùng context; không gán object từ context khác. Inverse duy trì quan hệ hai chiều. Many-to-many có metadata như createdAt/order có thể dùng model trung gian.

| Delete rule | Khi xóa owner |
| --- | --- |
| .nullify | Bỏ liên kết, giữ related model nếu hợp lệ |
| .cascade | Xóa related model theo relationship |
| .deny | Không cho xóa khi còn related objects |
| .noAction | Không tự xử lý liên kết; developer chịu trách nhiệm |

Cascade phù hợp child thuộc ownership của parent. Nếu xóa category chỉ muốn bỏ nhóm chứ không mất favorites, dùng nullify và optional relationship phù hợp. Test xóa parent/child/inverse/orphan.

### Attribute options

* @Attribute(.unique): uniqueness local store; không thay authorization/business conflict policy.
* @Attribute(.externalStorage): binary data có thể được framework lưu ngoài store; không thay image cache.
* @Transient: property không persist, ví dụ UI state tạm thời; có default phù hợp.
* @Attribute(originalName: "oldName"): hỗ trợ rename attribute khi migration tương thích.
* #Index và #Unique (compound uniqueness) từ iOS 18; kiểm tra deployment target. Index giảm query cost phù hợp nhưng tăng write/storage cost.

## 8. Concurrency và ModelActor

ModelContext có isolation; mainContext thuộc MainActor. Không truyền trực tiếp @Model instance qua actor/context rồi mutate background. Truyền Sendable DTO, business ID hoặc PersistentIdentifier hợp lệ, fetch lại record ở context đích.

```swift
import Foundation
import SwiftData

struct FavoriteDTO: Sendable {
    let id: String
    let title: String
}

@ModelActor
actor FavoritesImporter {
    func importItems(_ incoming: [FavoriteDTO]) throws {
        do {
            for dto in incoming {
                let id = dto.id
                var descriptor = FetchDescriptor<Favorite>(
                    predicate: #Predicate { $0.remoteID == id }
                )
                descriptor.fetchLimit = 1

                if let existing = try modelContext.fetch(descriptor).first {
                    existing.title = dto.title
                } else {
                    modelContext.insert(Favorite(remoteID: id, title: dto.title))
                }
            }
            try modelContext.save()
        } catch {
            modelContext.rollback()
            throw error
        }
    }
}
```

@ModelActor tạo modelContainer, modelExecutor và modelContext cho actor. Actor serialize access qua executor, không cam kết luôn chạy trên background thread. Khởi tạo importer ở isolation phù hợp, profile main thread; giữ một instance cho writer scope.

```swift
func importFromAPI(container: ModelContainer,
                   items: [FavoriteDTO]) async throws {
    let importer = FavoritesImporter(modelContainer: container)
    try await importer.importItems(items)
}
```

Ví dụ caller async; production giữ importer lâu dài và tránh nhiều writer song song. ModelActor giải quyết isolation, không tự giải quyết logical race giữa nhiều contexts ghi cùng record. Khi background save, test SwiftUI/UIKit refresh trên OS target; fetch lại nếu cần.

### Import dữ liệu lớn

Fetch existing IDs theo batch, dictionary lookup thay N+1 fetch; validate/dedup DTO, save theo batch vừa phải, handle cancellation giữa batches. Sync revision/cursor chỉ cập nhật sau commit. Không ghi đè local-only fields khi refresh API. Tách network fetch (DTO) khỏi persistence write, không giữ transaction logic qua await tùy ý.

## 9. Migration: VersionedSchema và SchemaMigrationPlan

Thêm optional/default attribute có thể automatic/lightweight khi tương thích; rename dùng originalName. Type/constraint/relationship thay đổi phức tạp có thể cần custom migration.

Ví dụ độc lập Note V1 → V2 thêm isArchived có default:

```swift
import Foundation
import SwiftData

enum NotesV1: VersionedSchema {
    static var versionIdentifier: Schema.Version { .init(1, 0, 0) }
    static var models: [any PersistentModel.Type] { [Note.self] }

    @Model
    final class Note {
        var title: String
        init(title: String) { self.title = title }
    }
}

enum NotesV2: VersionedSchema {
    static var versionIdentifier: Schema.Version { .init(2, 0, 0) }
    static var models: [any PersistentModel.Type] { [Note.self] }

    @Model
    final class Note {
        var title: String
        var isArchived: Bool = false

        init(title: String, isArchived: Bool = false) {
            self.title = title
            self.isArchived = isArchived
        }
    }
}

enum NotesMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [NotesV1.self, NotesV2.self]
    }
    static var stages: [MigrationStage] {
        [.lightweight(fromVersion: NotesV1.self, toVersion: NotesV2.self)]
    }
}

func makeNotesContainer() throws -> ModelContainer {
    let schema = Schema(versionedSchema: NotesV2.self)
    return try ModelContainer(for: schema, migrationPlan: NotesMigrationPlan.self)
}
```

Giữ model definition từng version đã ship, không chỉnh V1 tùy ý sau release. App mới dùng NotesV2.Note. MigrationStage.custom có willMigrate/didMigrate để cleanup/transform theo schema available ở từng phase; không giả định cả model cũ và mới cùng fetch được trong một phase. Migration phức tạp có thể cần intermediate schema.

### Core Data → SwiftData

Không tự ý mở mọi SQLite store Core Data bằng SwiftData dù có chung nền công nghệ. Kiểm tra entity/attribute/relationship mapping, model name, store configuration, version compatibility. Có thể thiết kế compatible migration theo tài liệu Apple hoặc export DTO từ Core Data rồi import vào store SwiftData mới. Backup, test upgrade và rollback trước release; tránh hai stack cùng ghi store tùy ý. Với target dưới iOS 17, giữ Core Data hoặc abstraction phù hợp.

### Test migration

Tạo disk store V1, insert data và save; đóng container V1, mở cùng store URL với schema V2 + plan. Assert count/IDs/title/relationship/default mới. Test mọi version được hỗ trợ, data duplicate/large dataset và failure. In-memory test không thay disk migration test. Không xóa store production để chữa migration fail.

## 10. CloudKit, offline và performance

### CloudKit sync

SwiftData hỗ trợ iCloud/CloudKit khi container, entitlement/capability và schema tương thích. Sync eventual consistency, không bảo đảm tức thì. CloudKit-backed schema không dùng unique constraint như Favorite local-only; cần attributes default/optional và relationship tương thích. Test account, development/production schema, conflict/delete propagation nhiều thiết bị. Không coi CloudKit là backend REST hoặc authorization nghiệp vụ.

### Offline-first repository

```text
UI -> Repository -> local fetch -> render cached content
                   -> remote fetch DTO -> validate -> upsert -> render fresh
```

Local store có lastUpdated/version/cursor. Offline show cached content + stale status. Online merge theo remoteID và field ownership (giữ isPinned local). Offline mutation cần outbox event/idempotency key, retry theo server contract, conflict policy/revision. Không để UI gọi API và ghi model tại nhiều nơi.

### Performance

* Predicate/sort/limit, fetchCount khi chỉ cần count; tránh materialize cả graph.
* Pagination theo stable sort; offset có thể không ổn định nếu data đổi.
* Index cho query thường dùng trên OS hỗ trợ, benchmark write/storage trade-off.
* Tránh N+1, ảnh lớn trong UI/model, cache file hợp lý; không giữ toàn bộ binary trong RAM.
* Không tạo container/context trong body/cell, batch mutations/save hợp lý.
* Profile Instruments và disk/device thật; không khẳng định SwiftData nhanh hơn Core Data mặc định.

## 11. Unit test với in-memory container

```swift
import XCTest
import SwiftData

final class FavoritesStoreTests: XCTestCase {
    @MainActor
    func testUpsertKeepsPinnedState() throws {
        let container = try makeContainer(inMemory: true)
        let store = FavoritesStore(context: container.mainContext)
        try store.upsert(id: "1", title: "Old")
        try store.togglePinned(id: "1")
        try store.upsert(id: "1", title: "New")

        let items = try store.all()
        XCTAssertEqual(items.count, 1)
        XCTAssertEqual(items.first?.title, "New")
        XCTAssertEqual(items.first?.isPinned, true)
    }

    @MainActor
    func testDelete() throws {
        let container = try makeContainer(inMemory: true)
        let store = FavoritesStore(context: container.mainContext)
        try store.upsert(id: "1", title: "Coffee")
        try store.delete(id: "1")
        XCTAssertTrue(try store.all().isEmpty)
    }
}
```

Test target dùng @testable import app module thực tế. Mỗi test container riêng, không dùng production store. Test cascade/nullify, validation, duplicate, save failure, DTO import và migration disk. In-memory không đại diện mọi behavior/performance disk store/CloudKit.

## 12. Lỗi thường gặp

| Lỗi | Nguyên nhân / giải pháp |
| --- | --- |
| Không có dữ liệu khi mở app | Thiếu container, in-memory hoặc store URL khác; kiểm tra configuration |
| UI không cập nhật | UIKit chưa refresh, context khác, stale snapshot; kiểm tra ownership/query |
| Dữ liệu mất khi restart | Chưa save/save fail/in-memory; explicit save và handle error |
| Crash hoặc sai dữ liệu khi đổi thread | Model/context qua isolation boundary; DTO/ID + ModelActor |
| Duplicate server item | Thiếu stable ID/upsert policy; dedup trước import |
| Cancel editor vẫn thay đổi | @Bindable live edit/autosave; dùng draft state |
| Migration fail | Schema không tương thích; versioned schema, disk test |
| Query chậm/fail | Predicate không translate, fetch quá nhiều; test OS/profile |
| Xóa parent mất children | Cascade sai ownership; chọn delete rule đúng |
| CloudKit không sync | Entitlement/schema/account/config sai; kiểm tra requirements |

## 13. Q&A phỏng vấn

**Q: SwiftData khác Core Data ở bản chất?** SwiftData dùng Swift macros, container/context và Observation trên nền persistence Apple; Core Data dùng NSManagedObject/schema/context queue, tooling/API trưởng thành. Chọn theo target, schema, migration.

**Q: @Model là struct được không?** Không, persistent model dùng class. DTO/domain value dùng struct Sendable.

**Q: ModelContainer vs ModelContext?** Container quản lý schema/store/config/migration; context fetch, track unsaved changes và save.

**Q: @Query vs FetchDescriptor?** @Query tích hợp SwiftUI và phản ánh thay đổi; FetchDescriptor mô tả query để context.fetch, caller quản lý kết quả/refresh.

**Q: Có cần save sau mọi mutation?** Main context có autosave nhưng explicit save khi cần durability/error handling, batch hợp lý.

**Q: ModelActor có phải background thread?** Không; isolation/executor, không cam kết thread. Dùng actor-owned context và DTO/ID.

**Q: SwiftData dùng UIKit được không?** Có. Inject context/container, fetch và render table/diffable snapshot; không cần SwiftUI.

**Q: Refresh server không mất pinned local?** Upsert theo remoteID, merge field ownership/revision, giữ local-only fields.

**Q: SwiftData vs UserDefaults?** SwiftData cho records/query/relationship/migration, UserDefaults settings nhỏ, Keychain secret.

**Q: App Core Data có nên chuyển ngay?** Không mặc định, xem deployment target, migration risk, feature/tooling và cost.

**Q: Lightweight vs custom migration?** Lightweight thay đổi tương thích; custom transform/cleanup phức tạp, versioned schema và migration tests.

**Q: Unique attribute có đủ tránh conflict cloud/context?** Không, uniqueness không thay business policy, CloudKit có schema constraints.

### Bài tập thực hành

1. Favorites SwiftUI add/edit/delete/search/pinned, kiểm tra persistence qua restart.
2. Cùng store xây UIKit UITableView, so sánh @Query và snapshot.
3. Category relationship, cascade/nullify và draft Save/Cancel.
4. Import 1.000 DTO qua ModelActor, dedup và giữ pinned; đo main-thread time/memory.
5. Note V1→V2, assert dữ liệu cũ; viết in-memory CRUD test và disk migration test.
6. Offline-first repository có local edits, server revision, retry/idempotency.

### Tham khảo

* [SwiftData - Apple](https://developer.apple.com/documentation/swiftdata)
* [ModelContainer](https://developer.apple.com/documentation/swiftdata/modelcontainer)
* [ModelContext](https://developer.apple.com/documentation/swiftdata/modelcontext)
* [ModelActor](https://developer.apple.com/documentation/swiftdata/modelactor)
* [SchemaMigrationPlan](https://developer.apple.com/documentation/swiftdata/schemamigrationplan)
* [Preserving your app’s model data across launches](https://developer.apple.com/documentation/swiftdata/preserving-your-apps-model-data-across-launches)
* [Core Data](core-data.md) · [Realm](realm.md) · [SwiftUI](../ui-+-autolayout/swiftui.md)

## Câu hỏi phỏng vấn tình huống

**Q: Save/Cancel editor dùng @Bindable trực tiếp có an toàn?** Live edit thay model và autosave có thể commit trước Cancel; dùng @State draft, validate rồi apply/save, rollback đúng scope.

**Q: PersistentIdentifier có dùng thay remoteID?** Không, identifier theo store, business ID dùng sync/backend và stable domain identity; temporary ID chưa persisted cần cẩn thận.

**Q: Background import làm @Query không refresh như kỳ vọng?** Test OS/context behavior, save writer, refresh/reconcile qua repository/UI state; không truyền @Model instance giữa actor.

**Q: Test migration in-memory đủ không?** Không, cần disk store version cũ, mở bằng schema mới, assert records/relationships/defaults; test failure/rollback.
