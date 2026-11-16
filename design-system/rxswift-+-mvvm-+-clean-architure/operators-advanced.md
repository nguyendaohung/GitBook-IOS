# RxSwift Advanced Operators – Edge Cases, Custom Operators và Testing

## 1. Bản đồ chọn toán tử

Bài này bổ sung các nhóm chưa có ví dụ đầy đủ ở [Foundation](operators-foundation.md) và [Composition](operators-composition.md). Đọc input/output type và contract trước khi chọn.

| Yêu cầu | Operator | Điều cần nhớ |
| --- | --- | --- |
| Tạo dãy số hữu hạn | range, generate | Cold, không tự background |
| Lặp một giá trị | repeatElement + take | Infinite nếu không giới hạn |
| Chọn kết quả đầu tiên từ nhiều nguồn | amb | First event thắng, kể cả error/completed |
| Chờ tới trigger mới nhận event | skip(until:) | Trigger next mở gate |
| Dừng theo trigger | take(until:) | Dispose upstream khi terminate |
| Lấy N event cuối | takeLast | Chờ completion, buffer N |
| Bỏ N event cuối | Custom droppingLast | RxSwift không có skipLast trong API đã đối chiếu; buffer N |
| Gom nhóm theo key | groupBy | Inner stream mỗi key, lifetime/memory |
| Gom batch | buffer | Array mỗi time/count window |
| Chia stream thành windows | window | Inner Observable, cần subscribe |
| Lấy mẫu theo tick | sample | Latest mới kể từ sample trước |
| Đổi nguồn dynamic | switchLatest | Dispose inner cũ |
| Giới hạn active work | merge(maxConcurrent:) | Queue pending vẫn cần bounded input |
| Log lifecycle | do, debug | Không đổi contract, không log secret |
| Fallback nếu empty | defaultIfEmpty, ifEmpty(switchTo:) | Không phải error recovery |
| Dùng resource theo subscription | using | Resource disposed khi lifetime kết thúc |
| Chặn owner bị retain | subscribe(with:), withUnretained | Cẩn thận replay buffer |
| Nối async/await | Single.value, Observable.values | Availability/cancellation/buffering |

## 2. range, generate, repeatElement

```swift
import Foundation
import RxSwift
let bag = DisposeBag()
Observable<Int>.range(start: 3, count: 4)
    .subscribe(onNext: { print($0) }).disposed(by: bag)
// 3,4,5,6

Observable<Int>.generate(initialState: 1,
                         condition: { $0 <= 8 },
                         iterate: { $0 * 2 })
    .subscribe(onNext: { print($0) }).disposed(by: bag)
// 1,2,4,8

Observable<String>.repeatElement("retry marker")
    .take(3)
    .subscribe(onNext: { print($0) }).disposed(by: bag)
// marker x3
```

repeatElement lặp một value, không phải retry side effect hoặc periodic timer. Không subscribe vô hạn trên synchronous scheduler nếu không có take/termination: có thể chiếm thread. Dùng interval/timer cho tick có thời gian, retry(when:) cho resubscription khi lỗi, deferred cho factory mới. generate condition/iterate có thể throw; overflow/loop condition cần kiểm tra.

## 3. takeLast, custom droppingLast, take/skip until, inclusive behavior

```swift
let values = Observable.of(1, 2, 3, 4, 5)
values.takeLast(2).subscribe(onNext: { print($0) }).disposed(by: bag)
// 4,5 chỉ khi source completed
values.take(while: { $0 < 3 }, behavior: .exclusive)
    .subscribe(onNext: { print($0) }).disposed(by: bag)
// 1,2
values.take(while: { $0 < 3 }, behavior: .inclusive)
    .subscribe(onNext: { print($0) }).disposed(by: bag)
// 1,2,3
```

takeLast buffer N; infinite source takeLast không emit nếu không complete. inclusive include boundary element khi predicate false; exclusive mặc định bỏ. filter khác take(while:): filter xét mọi event, take terminate ngay khi điều kiện fail.

```swift
let events = PublishSubject<Int>()
let start = PublishSubject<Void>()
let stop = PublishSubject<Void>()
events.skip(until: start)
    .take(until: stop)
    .subscribe(onNext: { print($0) }, onCompleted: { print("Done") })
    .disposed(by: bag)
events.onNext(1) // bỏ
start.onNext(())
events.onNext(2) // 2
events.onNext(3) // 3
stop.onNext(()) // Done
events.onNext(4) // bỏ
```

Use case: chỉ thu metrics khi test session bắt đầu, dừng khi user cancel. Notifier completion không tương đương next; notifier error có thể terminate với error. Lifecycle take(until: rx.deallocated) hữu ích nhưng không tự sửa strong capture cycle khiến owner không dealloc.

### Custom droppingLast: bỏ N event cuối

RxSwift ObservableType không cung cấp skipLast trong API được đối chiếu; có thể compose bằng scan và flatMap. Ví dụ đơn giản cho N nhỏ, giữ buffer N elements.

```swift
extension ObservableType {
    func droppingLast(_ count: Int) -> Observable<Element> {
        precondition(count >= 0)
        if count == 0 { return asObservable() }
        return scan((buffer: [Element](), output: [Element]())) { state, value in
            var buffer = state.buffer
            buffer.append(value)
            let output = buffer.count > count ? [buffer.removeFirst()] : []
            return (buffer: buffer, output: output)
        }
        .flatMap { Observable.from($0.output) }
    }
}

values.droppingLast(2)
    .subscribe(onNext: { print($0) }).disposed(by: bag)
// 1,2,3; mỗi output trễ 2 events, bỏ 4,5 khi complete
```

Với buffer lớn dùng ring buffer/deque thay Array.removeFirst O(N); test count=0, empty, ít hơn count, error/cancel. Đây là custom operator, không phải API chuẩn RxSwift.

## 4. amb – nguồn phản hồi đầu tiên

```swift
let fast = Observable.just("fast")
    .delay(.milliseconds(100), scheduler: MainScheduler.instance)
let slow = Observable.just("slow")
    .delay(.milliseconds(300), scheduler: MainScheduler.instance)
fast.amb(slow)
    .subscribe(onNext: { print($0) }).disposed(by: bag)
// fast
```

amb chọn source có event đầu tiên, dispose nguồn thua; event đầu có thể error hoặc completed, không phải first successful response. Không dùng amb cho duplicate payment/request mutation. Với redundant read muốn first success, thiết kế error handling, timeout và cancellation riêng; race nhiều endpoint tăng data usage/server load, cần policy. merge + take(1) cũng có thể fail nếu một inner error sớm; không mặc định an toàn hơn.

## 5. groupBy – phân luồng theo key

```swift
struct Metric {
    let category: String
    let value: Int
}

let metrics = Observable.of(
    Metric(category: "download", value: 10),
    Metric(category: "upload", value: 3),
    Metric(category: "download", value: 20)
)

metrics.groupBy(keySelector: { $0.category })
    .flatMap { group in
        group.scan(0) { total, metric in total + metric.value }
            .map { total in (group.key, total) }
    }
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
// (download,10), (upload,3), (download,30)
```

groupBy trả Observable<GroupedObservable<Key, Element>>; group.key là identity, group là stream theo key. Subscribe group ngay (flatMap) để không bỏ các event đã phát; nhóm thường sống theo source, không tự expire theo key inactivity. Với key vô hạn (user IDs, URLs), memory tăng; dùng bounded categories, explicit window/session reset hoặc dictionary reducer có eviction policy.

**Thực tế:** Tổng bytes theo direction của speed test, analytics category counters. Nếu chỉ cần dictionary state toàn bộ, scan dictionary có thể đơn giản hơn groupBy; nếu mỗi key cần pipeline riêng, groupBy hữu ích.

## 6. buffer, window và sample – batching/metrics

```swift
let source = PublishSubject<Int>()
source.buffer(timeSpan: .seconds(1), count: 3,
              scheduler: MainScheduler.instance)
    .filter { !$0.isEmpty }
    .subscribe(onNext: { print("Batch", $0) })
    .disposed(by: bag)

source.onNext(1)
source.onNext(2)
source.onNext(3) // [1,2,3]
source.onNext(4) // batch tiếp theo khi time/count đủ
```

buffer(timeSpan,count) emit array khi đầy hoặc hết time. Có thể emit empty batch theo timer/completion behavior, filter empty nếu không cần. Window cùng boundary nhưng trả stream con, xử lý streaming từng window thay allocate array toàn bộ ngay. Window không phải groupBy: windows theo time/count, group theo key.

```swift
source.window(timeSpan: .seconds(1), count: 3,
              scheduler: MainScheduler.instance)
    .flatMap { window in window.reduce(0, accumulator: +) }
    .subscribe(onNext: { print("Window sum", $0) })
    .disposed(by: bag)
```

```swift
let tick = Observable<Int>.interval(.milliseconds(500),
                                    scheduler: MainScheduler.instance)
source.sample(tick)
    .subscribe(onNext: { print("Latest metric", $0) })
    .disposed(by: bag)
```

sample lấy latest mới khi sampler emit, không phải timer tự tạo value và không nhất thiết repeat last value nếu không có dữ liệu mới. Dùng cho tốc độ tải, scroll progress, map camera updates; không drop business-critical events. buffer/window giới hạn batch riêng nhưng không tự backpressure toàn hệ thống: downstream chậm hoặc pending inner queue vẫn tăng. Store event critical trong durable queue, bounded retry, dedup.

## 7. switchLatest, concat và merge – outer/inner lifetime

```swift
let a = PublishSubject<String>()
let b = PublishSubject<String>()
let sources = PublishSubject<Observable<String>>()
sources.switchLatest()
    .subscribe(onNext: { print($0) }).disposed(by: bag)
sources.onNext(a.asObservable())
a.onNext("A1")
sources.onNext(b.asObservable())
a.onNext("A2") // ignored
b.onNext("B1") // B1
```

flatMapLatest = map selector + switchLatest. Outer complete không nhất thiết complete result ngay nếu current inner còn active; result complete khi outer và latest inner hoàn thành. Inner error terminate result trừ catch. concat on stream-of-streams subscribe inner tuần tự; hot inner có thể emit trước khi được subscribe, mất events. Cold deferred phù hợp ordered network writes.

```swift
func upload(id: Int) -> Completable {
    Completable.deferred {
        // Replace with cancellable HTTP upload.
        print("Uploading", id)
        return .empty()
    }
}
Observable.from([1,2,3,4,5])
    .map { id in upload(id).andThen(Observable.just(id)) }
    .merge(maxConcurrent: 2)
    .subscribe(onNext: { print("Done", $0) })
    .disposed(by: bag)
```

Ví dụ synchronous chỉ minh họa lazy work, không chứng minh concurrency thực. Trong async upload, tối đa 2 active inner subscriptions; queued IDs có thể nhiều, giới hạn input pages/batches. concatMap cho thứ tự, merge(maxConcurrent:1) serialize subscription; nếu selector có eager work, việc map trước vẫn thực thi trước subscription. Không giữ unbounded pending queue. Nếu một inner error, merge mặc định terminate và dispose active others; nếu cần best-effort batch, map mỗi inner thành Result/id state và catch trong inner.

## 8. Empty, error, completion và disposal

```swift
enum Failure: Error { case offline }

Observable<Int>.empty().defaultIfEmpty(0) // next 0, completed
Observable<Int>.empty().ifEmpty(switchTo: Observable.just(42)) // 42
Observable<Int>.error(Failure.offline).defaultIfEmpty(0) // error
Observable<Int>.never().defaultIfEmpty(0) // không emit
```

defaultIfEmpty chỉ fallback khi successful completion không có next; error cần catch. ifEmpty(switchTo:) subscribe fallback khi empty. Không dùng empty để biểu diễn API error nếu cần hiển thị error.

```swift
let sequence = Observable.of(1,2,3)
    .do(onSubscribe: { print("Start") },
        onDispose: { print("Cleanup") })
sequence.ignoreElements().asCompletable()
    .subscribe(onCompleted: { print("Completed") })
    .disposed(by: bag)
```

do hook theo lifecycle, không thay source event type; onDispose chạy khi terminal hoặc dispose, không dùng như success signal. ignoreElements trả Observable<Never>, bỏ next giữ error/completed; chuyển asCompletable để diễn tả completion-only contract. onCompleted không được gọi chỉ vì dispose. Cancellation producer phải cleanup handle.

### materialize để giữ lỗi trong data

```swift
let events = Observable<Int>.error(Failure.offline).materialize()
events.subscribe(onNext: { event in
    switch event {
    case .next(let value): print(value)
    case .error: print("Failure event")
    case .completed: print("Completion event")
    }
}).disposed(by: bag)
```

materialize biến terminal thành value rồi complete outer stream; dematerialize khôi phục terminal semantics. Không làm source tiếp tục phát sau error; muốn outer input sống phải materialize/catch mỗi inner. Có thể split success/failure từ shared materialized stream, nhưng dùng enum UI state thường dễ hiểu hơn.

## 9. Resource lifetime: using, subscribe(with:), withUnretained

```swift
final class DemoResource: Disposable {
    func dispose() { print("Resource closed") }
}
let work = Observable<Int>.using({
    DemoResource()
}, observableFactory: { _ in
    Observable.of(1,2)
})
work.subscribe(onNext: { print($0) }).disposed(by: bag)
```

using tạo resource theo subscription, dispose cùng sequence lifetime; phù hợp resource cần close/unregister. Với URLSession create trả disposable cancel task thường đủ. Không reuse disposed resource hoặc giữ reference global ngoài lifetime.

```swift
final class Screen {
    let bag = DisposeBag()
    func bind(_ source: Observable<Int>) {
        source.subscribe(with: self, onNext: { owner, value in
            owner.render(value)
        }).disposed(by: bag)
    }
    func render(_ value: Int) { print(value) }
}
```

subscribe(with:) weakly associates object; callback chỉ chạy nếu owner còn tồn tại, không thay lifecycle disposal. withUnretained(owner) tạo tuple owner,value; đặt trước share(replay:) có thể khiến buffer giữ owner mạnh. Không dùng unowned nếu callback có thể sau dealloc. DisposeBag deinit, CompositeDisposable/SerialDisposable tùy owner lifetime.

## 10. Custom operator bằng composition

```swift
import Foundation
import RxSwift

extension ObservableType where Element == String {
    func normalizedQueries() -> Observable<String> {
        map { $0.trimmingCharacters(in: .whitespacesAndNewlines) }
            .distinctUntilChanged()
    }
}
```

Ưu tiên compose operators thay tự viết Observable.create: dễ đảm bảo terminal/disposal/scheduler contract, test dễ. Operator không subscribe bên trong trừ khi implement lifecycle đúng; không return stream đã subscribe. Không hardcode scheduler khi operator cần test virtual time.

```swift
extension ObservableType {
    func traced(_ name: String) -> Observable<Element> {
        self.do(onError: { _ in print(name, "error") },
                onCompleted: { print(name, "complete") },
                onSubscribe: { print(name, "subscribe") },
                onDispose: { print(name, "dispose") })
    }
}
```

Ví dụ log metadata, không log element/error payload nhạy cảm. Production dùng structured logging, disable debug, correlation IDs.

## 11. RxSwift ↔ Swift Concurrency

```swift
// Requires compatible RxSwift 6.x and Swift concurrency deployment.
func loadTitle(repository: any ProductRepository) async throws -> String {
    let product = try await repository.product(id: 1).value
    return product.name
}
```

Single.value await one success/error; Maybe.value trả optional, Completable.value Void. RxSwift cung cấp Observable.values AsyncSequence bridge trên phiên bản hỗ trợ. Không tự giả định no buffering/backpressure; consumer chậm, cancellation/lifetime và Sendable cần kiểm tra. Swift Concurrency không thay tất cả reactive operators: dùng async/await cho one-shot structured tasks, Rx cho continuous UI streams, bridge tại boundary.

```swift
func observeUpdates(_ updates: Observable<String>) async throws {
    for try await value in updates.values {
        try Task.checkCancellation()
        print(value)
    }
}
```

Khi tạo Task từ VC, giữ handle/cancel theo lifetime; UI update MainActor. Bridge custom Task -> Single cần disposable cancel Task và terminal exactly-once, isolation/Sendable phù hợp Swift 6; không dùng blocking wait để convert async thành sync.

## 12. RxTest: đọc output và edge cases

```swift
import XCTest
import RxSwift
import RxTest

final class AdvancedOperatorsTests: XCTestCase {
    func testTakeUntilStops() {
        let scheduler = TestScheduler(initialClock: 0)
        let source = scheduler.createHotObservable([
            .next(10, 1), .next(20, 2), .next(40, 3), .completed(60)
        ])
        let stop = scheduler.createHotObservable([.next(30, true)])
        let observer = scheduler.createObserver(Int.self)
        let bag = DisposeBag()
        source.take(until: stop).subscribe(observer).disposed(by: bag)
        scheduler.start()
        XCTAssertEqual(observer.events, [.next(10,1), .next(20,2), .completed(30)])
        XCTAssertEqual(source.subscriptions, [Subscription(0,30)])
    }

    func testTakeLastWaitsForCompletion() {
        let scheduler = TestScheduler(initialClock: 0)
        let source = scheduler.createHotObservable([
            .next(10, 1), .next(20, 2), .next(30, 3), .completed(40)
        ])
        let observer = scheduler.createObserver(Int.self)
        let bag = DisposeBag()
        source.takeLast(2).subscribe(observer).disposed(by: bag)
        scheduler.start()
        XCTAssertEqual(observer.events, [.next(40,2), .next(40,3), .completed(40)])
    }
}
```

Test virtual time không sleep; assert events và subscriptions. Thêm tests amb first error, groupBy per-key, buffer count/time, skip(until:) before/after trigger, retry max attempts, Single empty/multiple, share reconnect. Test same-timestamp scheduling nếu business phụ thuộc boundary ordering. Không dùng real URLSession trong operator unit test.

### Q&A phỏng vấn

**Q: amb có chọn request success đầu tiên?** Không, source có first event thắng, kể cả error/completed.

**Q: takeLast(2) trên PublishRelay không complete?** Không emit vì chờ completion; dùng scan/replay nếu cần latest history live.

**Q: filter và take(while:) khác gì?** filter tiếp tục xét event sau false, take terminate tại boundary.

**Q: groupBy có bao nhiêu stream con?** Một theo key xuất hiện trong source lifetime; bounded key space/cleanup cần thiết.

**Q: buffer vs window?** buffer array batch, window inner observable; cả hai không tự tạo demand-based backpressure.

**Q: sample vs throttle?** sample trigger bởi sampler lấy latest mới, throttle giới hạn tần suất theo thời gian nguồn; semantics khác.

**Q: switchLatest khi outer complete nhưng inner chưa complete?** Chờ latest inner hoàn tất; error terminate theo contract.

**Q: repeatElement vs retry?** repeatElement phát cùng value lặp, retry resubscribe source khi error; không phải periodic timer.

**Q: defaultIfEmpty xử lý lỗi không?** Không, chỉ successful completion không value. catch cho error.

**Q: onDispose có đồng nghĩa thành công?** Không, terminal success/error hoặc cancel đều có disposal.

**Q: Custom operator có nên subscribe bên trong?** Ưu tiên compose; nếu tự subscribe cần cleanup/terminal/scheduler/serialization đúng.

**Q: Single.value có block thread?** Async await có thể suspend, không dùng semaphore; cancellation/underlying cleanup phụ thuộc producer.

**Bài tập:** Build metrics pipeline groupBy direction + buffer/sample, test bounded uploads, implement normalizedQueries, test first error amb và takeLast completion; giải thích từng operator lựa chọn.

### Tham khảo

* [ObservableType API](https://docs.rxswift.org/protocols/observabletype)
* [PrimitiveSequenceType API](https://docs.rxswift.org/protocols/primitivesequencetype)
* [RxSwift Getting Started](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/GettingStarted.md)
* [Traits](traits-subjects-relays.md) · [Higher-order functions](higher-order-functions.md) · [RxTest thực tế](practical-patterns.md)
