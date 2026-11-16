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
