# Higher-order Functions trong Swift và RxSwift

Các ví dụ sử dụng Swift Standard Library, Foundation cho xử lý text và RxSwift 6.x. Thêm import Foundation và import RxSwift khi ghép snippets vào playground/target.

## 1. Higher-order function là gì?

Higher-order function (hàm bậc cao) nhận function/closure làm argument, trả function/closure, hoặc cả hai. Swift function là first-class value: có thể gán biến, truyền tham số và trả về. Higher-order không đồng nghĩa async, reactive hoặc functional programming thuần túy. map/filter/reduce là ví dụ quen thuộc; RxSwift dùng closure để compose event streams.

```swift
func apply(_ number: Int, transform: (Int) -> Int) -> Int {
    transform(number)
}
print(apply(3) { $0 * 2 }) // 6

func multiplier(by factor: Int) -> (Int) -> Int {
    { value in value * factor }
}
let double = multiplier(by: 2)
print(double(5)) // 10
```

Closure parameter mặc định nonescaping; nếu lưu hoặc truyền sang escaping API cần @escaping. Return closure capture factor; nếu capture reference type, xem ownership. @Sendable liên quan concurrency safety, không đồng nghĩa @escaping.

## 2. map – biến đổi từng phần tử

```swift
let numbers = [1, 2, 3]
let doubled = numbers.map { $0 * 2 } // [2, 4, 6]
let titles = numbers.map { "Item \($0)" } // [String]

struct ProductDTO {
    let id: Int
    let title: String
    let priceInCents: Int
}
struct ProductRow {
    let id: Int
    let text: String
}

let products = [ProductDTO(id: 1, title: "Coffee", priceInCents: 250)]
let rows = products.map { ProductRow(id: $0.id, text: $0.title) }
```

Array.map trả array mới, giữ số phần tử và thứ tự; thông thường O(n) nếu transform O(1). Không mutate input tự động, nhưng closure có side effects hoặc reference values vẫn có thể mutate shared object. Map DTO sang Domain/Presentation model, không để UI xử lý raw JSON. Use key path products.map(\.title) khi phù hợp.

```swift
import RxSwift
let bag = DisposeBag()
Observable.from(numbers)
    .map { $0 * 2 }
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
// 2, 4, 6 (3 next events)
```

Array map chạy khi gọi; Observable map tạo pipeline, transform khi upstream phát next trong subscription. Observable.just(numbers).map { $0.map { $0 * 2 } } phát một [Int], không phải 3 events.

## 3. filter – chọn phần tử theo điều kiện

```swift
let even = [1, 2, 3, 4, 5].filter { $0.isMultiple(of: 2) }
// [2,4]
let affordable = products.filter { $0.priceInCents < 500 }
```

filter giữ element type và relative order, không transform. Thường O(n); result size <= input. Predicate throwing thì propagate error trong Swift, RxSwift throwing predicate làm stream error.

```swift
let queries = PublishSubject<String>()
queries
    .map { $0.trimmingCharacters(in: .whitespacesAndNewlines) }
    .filter { !$0.isEmpty }
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
```

Filter empty chỉ phù hợp nếu muốn bỏ event. Trong search UI, empty phải clear/cancel old request nên branch trong flatMapLatest thay filter trước. filter { $0 != nil } vẫn giữ Optional type; compactMap unwrap tốt hơn.

## 4. compactMap – biến đổi và bỏ nil

```swift
let raw = ["10", "bad", "20", ""]
let parsed = raw.compactMap { Int($0) } // [10,20]
let optionalNames: [String?] = ["A", nil, "B"]
let names = optionalNames.compactMap { $0 } // [A,B]
let mapped = raw.map { Int($0) } // [Optional(10), nil, Optional(20), nil]
```

compactMap bỏ nil, không tự loại 0/false/empty string. Nếu parse lỗi là data integrity issue, dùng throwing map/Result để không âm thầm bỏ dữ liệu. Swift Array compactMap vs RxSwift compactMap: một cái iterate collection, cái kia transform từng emitted value và bỏ nil.

```swift
Observable.from(raw)
    .compactMap { Int($0) }
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
// 10,20
```

## 5. flatMap – làm phẳng collection hoặc inner stream

### Swift Sequence.flatMap

```swift
let nested = [[1, 2], [3], [4, 5]]
let flattened = nested.flatMap { $0 } // [1,2,3,4,5]
let expanded = [1, 2, 3].flatMap { n in [n, n * 10] }
// [1,10,2,20,3,30]
```

Sequence.flatMap nhận closure T -> Sequence<U>, nối từng sequence thành array U. Flatten một cấp, không recursive flatten mọi nested levels. Old Swift flatMap bỏ optional đã được thay bằng compactMap. Optional.flatMap có contract khác: unwrap optional rồi transform T -> U?, tránh Optional<Optional<U>>.

```swift
let input: String? = "42"
let parsedOptional = input.flatMap { Int($0) } // Int? 42
```

### RxSwift flatMap

```swift
func mockPrices(for id: Int) -> Observable<String> {
    Observable.just("Price for \(id)")
}
Observable.of(1, 2, 3)
    .flatMap { id in mockPrices(for: id) }
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
```

RxSwift flatMap nhận T -> ObservableConvertibleType<U>, subscribe inner sources và merge next events. Không đảm bảo order nếu inner async; outer complete còn phải chờ active inner hoàn thành, error mặc định terminate result. Nó không tự tạo threads. Single.flatMap nối Single -> Single theo one-result contract; Observable.flatMap có nhiều inner tùy outer events.

| Kiểu input | Closure | Kết quả |
| --- | --- | --- |
| Array<T>.map | T -> U | [U] |
| Array<T>.compactMap | T -> U? | [U] |
| Array<T>.flatMap | T -> Sequence<U> | [U] |
| Optional<T>.flatMap | T -> U? | U? |
| Observable<T>.map | T -> U | Observable<U> |
| Observable<T>.compactMap | T -> U? | Observable<U> |
| Observable<T>.flatMap | T -> Observable<U> hoặc trait convertible | Observable<U> |
| Single<T>.flatMap | T -> Single<U> | Single<U> |

**Bẫy phỏng vấn:** Observable<[Product]>.flatMap { products in products.map(...) } không phải cách flatten array trực tiếp nếu closure trả Array không conform ObservableConvertibleType. Dùng .map { $0.map(...) } giữ list, hoặc .flatMap { Observable.from($0) } phát từng product. Nếu mỗi product gọi API, chọn flatMap/concatMap/merge(maxConcurrent:) theo lifetime và ordering.

## 6. reduce, reduce(into:) và scan

```swift
let sum = [1, 2, 3, 4].reduce(0, +) // 10

let counts = ["A", "B", "A"].reduce(into: [String: Int]()) { result, item in
    result[item, default: 0] += 1
}
// ["A":2, "B":1]
```

reduce(initial, combine) tích lũy từng bước và trả final accumulator. Empty trả initial. reduce(into:) mutate inout accumulator, thường giảm copy khi build Array/Dictionary. Không đảm bảo mọi reduce(into:) nhanh hơn: đo workload. Tránh dùng reduce với side effects khó đọc khi for loop rõ hơn.

```swift
Observable.from([1,2,3])
    .scan(0, accumulator: +) // 1,3,6
    .subscribe(onNext: { print($0) }).disposed(by: bag)

Observable.from([1,2,3])
    .reduce(0, accumulator: +) // 6 khi completed
    .subscribe(onNext: { print($0) }).disposed(by: bag)
```

Rx scan emit intermediate state, seed không tự emit; startWith(seed) nếu cần. Rx reduce chờ source complete, infinite input không emit final. toArray cũng chờ complete và có thể tăng RAM; giới hạn finite source.

### Thực tế: giỏ hàng và state reducer

```swift
enum CartAction {
    case add(id: Int)
    case remove(id: Int)
    case clear
}

let cartActions = PublishSubject<CartAction>()
let cart = cartActions.scan(Set<Int>()) { current, action in
    var next = current
    switch action {
    case .add(let id): next.insert(id)
    case .remove(let id): next.remove(id)
    case .clear: next.removeAll()
    }
    return next
}
.startWith(Set<Int>())
.distinctUntilChanged()
```

Pure reducer dễ test, event source serialize; Set không đảm bảo order, nếu UI order cần array + ID dedup. Không giữ mutation side effect trong reducer. Share state nếu nhiều binding, state owner cần tồn tại đúng lifetime.

## 7. sorted, sort, forEach, contains, allSatisfy, first(where:)

```swift
let values = [3, 1, 2]
let ascending = values.sorted() // [1,2,3], original unchanged
var mutable = values
mutable.sort(by: >) // [3,2,1], in-place

let byPrice = products.sorted { lhs, rhs in
    if lhs.priceInCents == rhs.priceInCents { return lhs.id < rhs.id }
    return lhs.priceInCents < rhs.priceInCents
}

values.forEach { print($0) }
let hasEven = values.contains { $0.isMultiple(of: 2) } // true
let allPositive = values.allSatisfy { $0 > 0 } // true
let firstEven = values.first { $0.isMultiple(of: 2) } // 2
```

sorted O(n log n) thông thường, comparator strict weak ordering (không dùng <=), Swift sort stable. sort mutate array, sorted trả mới. forEach không dùng break/continue như for loop; return chỉ thoát closure iteration. contains/first/allSatisfy short-circuit; allSatisfy trên empty true (vacuous truth). Nếu muốn yêu cầu nonempty phải thêm !isEmpty. max/min phù hợp tìm extrema O(n), không sort toàn bộ.

```swift
let unique = [1, 2, 1, 3, 2].reduce(into: (seen: Set<Int>(), items: [Int]())) {
    if $0.seen.insert($1).inserted { $0.items.append($1) }
}.items // [1,2,3]
```

Dedup giữ order expected O(n) với Set membership, dùng stable ID cho model. Rx distinctUntilChanged chỉ bỏ adjacent duplicate, không giống Set dedup toàn lịch sử.

## 8. Lazy evaluation, escaping và performance

```swift
let result = (1...10_000).lazy
    .filter { $0.isMultiple(of: 2) }
    .map { $0 * 10 }
    .prefix(3)
print(Array(result)) // [20,40,60]
```

Lazy Sequence trì hoãn iteration và giảm intermediate arrays khi consumer chỉ cần prefix. Không tự async/background. Lazy map closure được evaluate khi iterate, có thể chạy lại nhiều lần. Array.map/filter eager tạo arrays trung gian; compiler có thể optimize nhưng không giả định. Rx pipeline cũng lazy theo subscription/event, nhưng time/error/disposal/multiple subscribers khác Swift lazy sequence.

```swift
func makeLogger(prefix: String) -> (String) -> String {
    { message in "\(prefix): \(message)" }
}
let networkLog = makeLogger(prefix: "Network")
print(networkLog("request failed"))
```

Closure composition giúp reuse; tránh capture self mạnh trong stored operator pipeline có lifetime dài. SwiftUI View body không nên sort/filter dataset lớn mỗi recomputation; move derived data vào ViewModel/repository theo nhu cầu.

## 9. Thực tế: DTO → Filter → UI, hai cách triển khai

```swift
struct CatalogDTO {
    let id: Int
    let name: String?
    let available: Bool
    let price: Int
}
struct CatalogRow: Equatable {
    let id: Int
    let title: String
    let price: Int
}

func makeRows(_ items: [CatalogDTO]) -> [CatalogRow] {
    items.filter(\.available)
        .compactMap { dto -> CatalogRow? in
            guard let name = dto.name, !name.isEmpty else { return nil }
            return CatalogRow(id: dto.id, title: name, price: dto.price)
        }
        .sorted { lhs, rhs in
            lhs.price == rhs.price ? lhs.id < rhs.id : lhs.price < rhs.price
        }
}

func reactiveRows(_ items: Observable<[CatalogDTO]>) -> Observable<[CatalogRow]> {
    items.map(makeRows).distinctUntilChanged()
}
```

Một pure function xử lý business mapping, Rx compose quanh function để mỗi API response tạo list mới. Không flatMap từng row nếu UI cần snapshot cả array. distinctUntilChanged dựa Equatable row/list; expensive equality/sort trên dataset lớn cần profile, background scheduler trước map và main trước render. Nếu name bắt buộc theo backend contract, đừng compactMap bỏ lỗi âm thầm: validate DTO và emit failure/partial data theo policy.

### Q&A phỏng vấn

**Q: Higher-order function là gì?** Nhận hoặc trả function/closure, ví dụ map và multiplier factory; không đồng nghĩa async.

**Q: map và forEach khác gì?** map trả collection transformed; forEach chủ yếu side effect, không return collection.

**Q: map vs compactMap vs flatMap?** map một input một output, compactMap bỏ nil, Sequence.flatMap nối nhiều sequence con; Rx flatMap merge inner observable.

**Q: Tại sao map API trả Observable<Observable<T>>?** Closure trả stream nên map giữ nested type; flatten theo use case (latest/first/parallel/sequential).

**Q: reduce vs scan?** Swift reduce final value; Rx reduce final khi completed; Rx scan intermediate values cho state.

**Q: reduce(into:) tốt hơn khi nào?** Accumulator collection mutable, giảm copy trong nhiều trường hợp; benchmark, không tuyệt đối.

**Q: sorted và sort khác nhau?** sorted trả array mới, sort mutate; comparator đúng strict ordering và giữ stable order khi equal.

**Q: allSatisfy([]) trả gì?** true; không có element vi phạm predicate. Validate nonempty riêng.

**Q: Lazy Sequence có giống Observable?** Cùng trì hoãn evaluation theo mô hình riêng, nhưng Sequence pull/iterate, Rx push events/time/error/dispose. Lazy không tự async.

**Q: Có nên dùng compactMap bỏ DTO không hợp lệ?** Tùy contract; missing optional có thể bỏ, invalid required data cần error/telemetry/partial policy.

**Q: Array(Set(values)) giữ thứ tự?** Không cam kết; dùng seen Set + filter/loop để dedup giữ order.

**Q: Rx map chạy thread nào?** Theo upstream event delivery hoặc scheduler đã đặt trước operator; không tự background.

### Bài tập

1. Dự đoán output của map/compactMap/flatMap trên [String?], [[Int]] và Observable<[Int]>.
2. Tạo catalog mapping giữ stable IDs, filter availability, sort price; test empty/duplicate/invalid DTO.
3. Viết reducer giỏ hàng bằng Swift reduce và Rx scan; test reset, repeated action, seed.
4. So sánh Array eager vs lazy prefix trên dataset lớn bằng Instruments; không khẳng định performance nếu chưa đo.
5. Tạo Single product -> Single price bằng flatMap, rồi dùng Observable search flatMapLatest; test cancellation khác nhau.

### Tham khảo

* [Swift Standard Library – Sequence](https://developer.apple.com/documentation/swift/sequence)
* [RxSwift ObservableType](https://docs.rxswift.org/protocols/observabletype)
* [Operators cơ bản](operators-foundation.md) · [Traits](traits-subjects-relays.md)
