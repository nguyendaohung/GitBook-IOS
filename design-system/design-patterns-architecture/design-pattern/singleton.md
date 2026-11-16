# SINGLETON

Có 2 cách chính để tạo Singleton:&#x20;

1. Sử dụng thuộc tính tĩnh (static property): Đây là cách đơn giản nhất để tạo một singleton bằng cách sử dụng một thuộc tính tĩnh trong lớp.
2. Sử dụng dispatch queue để tạo một đối tượng singleton (Singleton Dispatch Queue pattern): Đây là một phiên bản nâng cao của cách tạo singleton với mục đích tối ưu hóa tính toàn vẹn và tăng tính an toàn của đối tượng singleton.

Singleton: Tạo ra 1 instance duy nhất và có thể truy cập mọi nơi trong app. Mẫu này đảm bảo rằng một lớp chỉ có một đối tượng duy nhất và cung cấp một điểm truy cập duy nhất đến đối tượng đó.

Vấn đề:

* Có thể gặp vấn đề Read-Write problem. (Dữ liệu chưa write xong, đã có nơi khác read ⇒ sai xót dữ liệu): Vấn đề này có thể xảy ra khi nhiều luồng cùng truy cập vào đối tượng Singleton để đọc và ghi dữ liệu cùng lúc. Điều này có thể dẫn đến việc mất đi dữ liệu hoặc việc thực thi sai của ứng dụng.
* Cách khắc phục:
  * Dùng **thread sync**: đưa write và read vào 1 hàng đồng bộ. 1 lúc chỉ có làm được 1 việc: read hoặc write ⇒ Chậm
  * Dùng **barrier** của GCD: có 1 điểm sync ⇒ Nhanh

Nhược điểm: Vì nó tồn tại từ lúc khởi tạo cho đến cuối cùng. Nên nếu tạo nhiều singleton thì tốn bộ nhớ.

## Singleton và thread safety

```swift
actor SessionStore {
    static let shared = SessionStore()
    private var token: String?
    private init() {}

    func update(_ value: String?) { token = value }
    func current() -> String? { token }
}
```

static let khởi tạo lazy một lần thread-safe; không có nghĩa mutable state của instance thread-safe. Actor bảo vệ state, nhưng qua await vẫn có logical race. GCD concurrent custom queue + barrier cho writes, sync reads phải tránh reentrant deadlock; barrier không có ý nghĩa exclusive trên global concurrent queue. Singleton không tự giải phóng khi screen đóng; logout clear credentials/state và cancel work. Prefer DI protocol, composition root giữ app-scoped instance thay global accessor.

**Q: Singleton thread-safe?** Initialization static let thread-safe, mutable state cần isolation/lock/queue.

**Q: Singleton vs shared dependency?** Shared instance inject được, caller không cần global; singleton pattern giới hạn instance/access.

**Q: Khi nào không dùng?** Screen/session state, cần nhiều instance/test isolation; global mutable state làm coupling.

**Q: Actor singleton có tránh mọi race?** Data race isolated state, logical race qua await vẫn cần invariant/generation.
