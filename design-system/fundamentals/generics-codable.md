# Generics, Optional và Codable

### Định nghĩa

Generic giúp viết code dùng lại cho nhiều kiểu mà vẫn kiểm tra type tại compile time. Protocol constraint giới hạn hành vi cần thiết; associatedtype biểu diễn kiểu phụ thuộc vào conforming type.

```swift
func first<T: Equatable>(_ items: [T], matching value: T) -> T? {
    items.first { $0 == value }
}

struct APIResponse<T: Decodable>: Decodable {
    let data: T
}

struct Product: Codable, Equatable, Hashable {
    let id: Int
    let name: String
}
```

* `some Protocol`: opaque type, một concrete type cố định do hàm chọn.
* `any Protocol`: existential chứa một giá trị conform protocol; có thể đổi concrete type, nhưng có giới hạn với associated type.
* Generic `T: Protocol`: caller chọn concrete type, compiler giữ được quan hệ giữa các kiểu.
* Optional là enum có .some/.none; dùng guard let, if let, nil-coalescing thay vì force unwrap. Optional chaining trả về optional.
* Equatable so sánh giá trị; Hashable dùng với Set/Dictionary, giá trị bằng nhau phải có hash nhất quán trong cùng execution. Comparable thêm thứ tự.
* Array/Dictionary/String có value semantics và thường dùng copy-on-write: chỉ copy storage khi mutate và storage không còn unique. Struct chứa class vẫn có thể chia sẻ reference.

### Parse JSON và Error Handling

```swift
enum APIError: Error {
    case invalidResponse
    case status(Int)
}

func decodeProduct(_ data: Data) throws -> Product {
    try JSONDecoder().decode(APIResponse<Product>.self, from: data).data
}
```

* Codable = Encodable + Decodable; CodingKeys đổi tên field; decoder có date/key decoding strategy.
* Field bắt buộc thiếu hoặc sai type gây DecodingError; optional xử lý missing/null, không che lỗi type mismatch.
* SwiftyJSON thuận tiện khi JSON động; Codable tốt hơn cho contract ổn định và compile-time type safety.
* throws/try/catch truyền lỗi; Result phù hợp với callback, async throws phù hợp Swift Concurrency.
* Log codingPath khi decode fail, không log token hay dữ liệu riêng tư.

### Q&A

**Q: Generic khác Any?** Generic giữ thông tin type để compiler kiểm tra; Any cần cast runtime.

**Q: Struct có luôn nằm trên stack?** Không. Vị trí allocation phụ thuộc compiler, capture, boxing, storage; value semantics mới là đặc tính cần nhớ.

**Q: API thiếu field có nên dùng optional toàn bộ?** Không. Phân biệt field bắt buộc và field thực sự nullable; validate dữ liệu tại boundary.

**Liên hệ:** DaPass/BIDU parse JSON; Map4D, Call ID và Watch Faces dùng model rõ ràng để mapping API sang domain.

## Kiến thức chuyên sâu

### Generic constraint, associatedtype và type erasure

```swift
protocol Repository {
    associatedtype Entity: Identifiable
    func fetch() async throws -> [Entity]
}

struct LoadItems<R: Repository> {
    let repository: R

    func execute() async throws -> [R.Entity] {
        try await repository.fetch()
    }
}
```

Generic LoadItems giữ được quan hệ giữa Repository và Entity; compiler không cần cast. Khi cần lưu nhiều implementation khác concrete type, dùng protocol có primary associated type (Swift 5.7+) hoặc type erasure. Ví dụ:

```swift
struct AnyLoader<Item> {
    private let load: () async throws -> [Item]

    init(_ load: @escaping () async throws -> [Item]) {
        self.load = load
    }

    func fetch() async throws -> [Item] { try await load() }
}
```

Type erasure che concrete type nhưng vẫn giữ Item. Đổi lại có thêm abstraction, có thể có dynamic dispatch. Không cần ép mọi service thành generic nếu protocol thường đủ.

### Codable: DTO khác Domain Model

```swift
import Foundation

struct UserDTO: Decodable {
    let userID: Int
    let displayName: String?
    let createdAt: Date

    enum CodingKeys: String, CodingKey {
        case userID = "user_id"
        case displayName = "display_name"
        case createdAt = "created_at"
    }
}

struct User {
    let id: Int
    let name: String

    init(dto: UserDTO) {
        id = dto.userID
        name = dto.displayName ?? "Unknown"
    }
}

func parseUser(_ data: Data) throws -> User {
    let decoder = JSONDecoder()
    decoder.dateDecodingStrategy = .iso8601
    return User(dto: try decoder.decode(UserDTO.self, from: data))
}
```

DTO phản ánh server contract; Domain dùng kiểu và invariant của nghiệp vụ. Không đưa default 0/empty vào mọi field để che API lỗi. Khi API đổi kiểu Int thành String, xác nhận contract hoặc custom decoder có test rõ ràng. Khi decode lỗi, đọc DecodingError.keyNotFound/typeMismatch/valueNotFound/dataCorrupted và codingPath.

### Copy-on-write và ARC

```swift
var original = [1, 2, 3]
var copy = original
copy.append(4) // original vẫn [1, 2, 3]

final class Box { var value = 0 }
struct Container { var box = Box() }

let a = Container()
let b = a
b.box.value = 10
print(a.box.value) // 10, reference được chia sẻ
```

Value semantics không đồng nghĩa deep copy reference bên trong. ARC quản lý class instance bằng reference count, không phải garbage collector. Closure escaping capture self có thể cycle nếu self giữ closure; weak optional không kéo lifetime, unowned giả định object còn sống và có thể crash nếu sai.

### Q&A bổ sung

**Q: any và some có thay thế nhau?** Không. some che một concrete type ổn định; any cho phép nhiều concrete type nhưng mất một số quan hệ kiểu.

**Q: Codable và SwiftyJSON chọn gì?** API contract ổn định ưu tiên Codable; JSON động có thể dùng SwiftyJSON/JSONSerialization và validate tại boundary.

**Bài tập:** Decode JSON có field optional, date sai format, field bắt buộc thiếu; assert lỗi. Viết generic repository và fake repository để test.

## Câu hỏi phỏng vấn tình huống

**Q: Generic constraint có thể dùng where?** Có, where biểu diễn quan hệ nhiều generic/associatedtype, ví dụ T.Element == U.Element; compiler kiểm tra compile time.

**Q: Hashable có dùng hashValue làm persistent ID?** Không, hash có thể thay giữa process/run; dùng domain ID ổn định, equality/hash nhất quán.

**Q: Missing field và null decode khác gì?** decodeIfPresent xử lý cả missing/null thành nil; nếu cần phân biệt, kiểm tra contains/decodeNil; type mismatch vẫn lỗi.
