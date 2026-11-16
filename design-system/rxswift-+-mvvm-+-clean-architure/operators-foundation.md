# RxSwift Operators – Creation, Transformation và Filtering

## 1. Cách đọc một Rx pipeline

Observable phát 0...n next, sau đó có thể completed hoặc error. Một stream có thể không bao giờ terminate. Operator nhận stream và trả stream mới, thường không thực hiện work cho tới khi subscribe. Mỗi subscription có lifetime riêng; dispose ngừng quan sát, không tự đảm bảo hủy side effect ở producer nếu producer không hỗ trợ.

```swift
import RxSwift

let bag = DisposeBag()
Observable.from([1, 2, 3, 4, 5])
    .filter { $0.isMultiple(of: 2) }
    .map { $0 * 10 }
    .subscribe(onNext: { print($0) },
               onCompleted: { print("Done") })
    .disposed(by: bag)
// 20, 40, Done
```

Timeline dùng dấu - biểu diễn thời gian, | là completed, X là error. Ví dụ minh họa độc lập, dùng RxSwift 6.x; RxCocoa/RxRelay/RxTest chỉ import khi cần. Code học tập không phụ thuộc source dự án.

## 2. Creation operators – tạo nguồn

| Toán tử | Ý nghĩa | Ví dụ ứng dụng |
| --- | --- | --- |
| just(value) | Một value rồi completed | Default state, cache hit |
| of(a,b,c) | Các value riêng rồi completed | Fixture nhỏ |
| from(array) | Phát từng element | Xử lý list từng item |
| empty() | Completed không value | Bỏ một event/nhánh |
| never() | Không event/terminal | Test pending, không dùng làm fallback tùy tiện |
| error(error) | Error ngay | Mock failure, propagate |
| deferred | Factory chạy khi subscribe | Đọc token/cache mới, retry |
| create | Tự quản lý observer và disposable | Bridge callback, URLSession |
| timer | Delay, một event hoặc periodic tùy overload | Timeout/retry delay |
| interval | Counter theo thời gian | Polling, countdown (không dùng làm wall clock) |

```swift
Observable.just([1, 2, 3]) // 1 event kiểu [Int]
Observable.from([1, 2, 3]) // 3 events kiểu Int
Observable<Int>.empty()    // |
Observable<Int>.never()    // không phát gì
```

### deferred: tại sao cần?

```swift
var attempts = 0
let source = Observable<Int>.deferred {
    attempts += 1
    return .just(attempts)
}
source.subscribe(onNext: { print($0) }).disposed(by: bag) // 1
source.subscribe(onNext: { print($0) }).disposed(by: bag) // 2
```

Ví dụ chạy trên một thread; production shared mutable state cần isolation. deferred không tự chuyển background, chỉ trì hoãn tạo source tới subscription. just(expensiveWork()) thực hiện expensiveWork khi tạo biểu thức, không đợi subscribe. Cold network source phải tạo/resume request mỗi subscribe để retry đúng.

### create và cleanup

```swift
import Foundation
import RxSwift

func requestData(_ url: URL) -> Observable<Data> {
    Observable.create { observer in
        let task = URLSession.shared.dataTask(with: url) { data, response, error in
            if let error {
                observer.onError(error)
                return
            }
            guard let http = response as? HTTPURLResponse,
                  (200..<300).contains(http.statusCode),
                  let data else {
                observer.onError(URLError(.badServerResponse))
                return
            }
            observer.onNext(data)
            observer.onCompleted()
        }
        task.resume()
        return Disposables.create { task.cancel() }
    }
}
```

Mỗi subscription tạo task riêng; dispose cancel task. Server có thể vẫn hoàn tất operation đã nhận; không dùng cancellation như bảo đảm rollback. Producer callback phải serialize event và không phát sau terminal; Rx disposal ngừng nhận event theo contract. Với một HTTP response, Single phù hợp hơn; ví dụ Observable giúp thấy onNext/onCompleted. Swift 6 strict concurrency cần kiểm tra annotations/bridge isolation của RxSwift và URLSession theo phiên bản SDK.

### timer / interval

```swift
Observable<Int>.timer(.seconds(1), scheduler: MainScheduler.instance)
    .subscribe(onNext: { _ in print("One second later") })
    .disposed(by: bag)

Observable<Int>.interval(.seconds(1), scheduler: MainScheduler.instance)
    .take(3)
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
// 0,1,2 rồi completed
```

Timer scheduler không đảm bảo đúng giờ khi app suspend. Countdown production tính remaining từ deadline/monotonic time, không chỉ trừ 1 mỗi tick.

## 3. Transform operators – biến đổi giá trị

| Operator | Input → Output | Khi dùng |
| --- | --- | --- |
| map | T → R | DTO → domain, format giá |
| compactMap | T → R? rồi bỏ nil | parse Int, optional input |
| scan | accumulator mỗi event | Running total, state reducer |
| reduce | accumulator cuối khi completed | Tổng file đã xử lý |
| toArray | gom toàn bộ khi completed (Single) | Batch hữu hạn |
| startWith | thêm value trước source | Loading/initial state |
| defaultIfEmpty | default khi source complete rỗng | Empty fallback, không xử lý error |
| enumerated | index + value | Debug sequence/order |

```swift
Observable.from(["1", "bad", "3"])
    .compactMap { Int($0) }
    .map { $0 * 2 }
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
// 2, 6
```

map không flatten Observable trả về từ closure; map { api($0) } tạo Observable<Observable<Result>>. Cần flatMap/flatMapLatest/concatMap tùy semantics (bài tiếp theo). compactMap loại nil, không phải catch error; parse lỗi quan trọng cần state/error thay vì âm thầm bỏ.

### scan vs reduce

```swift
Observable.from([1, 2, 3])
    .scan(0, accumulator: +)
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
// 1, 3, 6

Observable.from([1, 2, 3])
    .reduce(0, accumulator: +)
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
// 6 sau completed
```

scan không tự phát seed; startWith(seed) nếu cần initial UI state. reduce/toArray chỉ emit khi complete thành công; với button/search stream dài vô hạn sẽ không emit cuối, toArray có thể tăng RAM vô hạn.

### Thực tế: state reducer giỏ hàng

```swift
enum CartAction { case add(Int), remove(Int), reset }

let actions = PublishSubject<CartAction>()
let count = actions.scan(0) { current, action in
    switch action {
    case .add(let n): return current + n
    case .remove(let n): return max(0, current - n)
    case .reset: return 0
    }
}
.startWith(0)
.distinctUntilChanged()
```

State reducer pure dễ test. Serialize action source, không read-modify-write BehaviorRelay.value từ nhiều threads. UI subscribe count trên main; business state phải có owner rõ ràng.

## 4. Filtering operators – chọn hoặc bỏ event

| Operator | Ý nghĩa | Use case |
| --- | --- | --- |
| filter | Giữ event thỏa predicate | Input hợp lệ |
| distinctUntilChanged | Bỏ value giống value liền trước | Search, render state |
| take(n) | Lấy n event đầu rồi completed | Một snapshot |
| skip(n) | Bỏ n event đầu | Bỏ initial relay nếu thật sự cần |
| take(while:) | Giữ tới khi predicate false | Dừng stream theo điều kiện |
| skip(while:) | Bỏ tới khi predicate false | Chờ bắt đầu |
| take(until:) | Dừng khi notifier phát next | Lifetime/cancel event |
| first() | Single<Element?> đầu hoặc nil khi empty | Optional first snapshot |
| single() | Yêu cầu đúng một element | Contract một result |
| ignoreElements() | Bỏ next, giữ terminal | Completion của operation |

```swift
Observable.from([1, 1, 2, 2, 1, 3])
    .distinctUntilChanged()
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
// 1, 2, 1, 3 (không phải distinct toàn lịch sử)
```

take(1) trả Observable và completed sau event đầu; first() trả Single optional, empty -> nil; single() lỗi nếu empty hoặc >1. take(until:) dừng khi notifier emit, không nhất thiết khi notifier chỉ completed. skip(1) với BehaviorRelay có thể bỏ initial state đúng chủ ý, nhưng không dùng mù quáng cho cold API vì bỏ response đầu.

### Thực tế: chỉ submit input hợp lệ

```swift
let names = PublishSubject<String>()
let validNames = names
    .map { $0.trimmingCharacters(in: .whitespacesAndNewlines) }
    .filter { !$0.isEmpty }
    .distinctUntilChanged()
```

Nếu search text rỗng phải clear list, không filter empty trước flatMapLatest: sẽ không cancel request cũ/clear UI. Thay bằng branch trả .just(.idle) bên trong flattening.

## 5. Time operators – debounce, throttle, delay, timeout, sample

Ví dụ event A tại 0ms, B tại 100ms, C tại 220ms, D tại 600ms; stream vẫn mở; khoảng 300ms.

```text
source:         A--B---C-------D--------->
debounce(300):           C--------D----->
                    t=520       t=900
throttle(latest:false): A----------D---->
                         t=0      t=600
throttle(latest:true):  A------C----D--->
                         t=0  t=300 t=600 (ranh giới phụ thuộc scheduling)
```

debounce chờ khoảng im lặng; nếu source completed có pending value, RxSwift có thể flush pending ngay. throttle đảm bảo khoảng thời gian tối thiểu giữa emissions; latest:true mặc định giữ latest trong cửa sổ, latest:false chỉ emit first khi được phép. Timeline không dùng để cam kết thứ tự event trùng timestamp.

```swift
let search = PublishSubject<String>()
let queries = search
    .debounce(.milliseconds(300), scheduler: MainScheduler.instance)
    .distinctUntilChanged()

let taps = PublishSubject<Void>()
let limited = taps.throttle(.milliseconds(500), latest: false,
                            scheduler: MainScheduler.instance)
```

Search dùng debounce để giảm request. Button throttle chỉ giới hạn thời gian, không chống request đang chạy lâu hơn cửa sổ; dùng flatMapFirst hoặc state gate. Với payment cần server idempotency, không dựa throttle.

* delay: trì hoãn emissions, không phải trì hoãn tạo subscription; delaySubscription trì hoãn subscribe. Error có semantics riêng, không dùng delay làm retry policy.
* timeout: nếu không nhận event trong thời gian quy định, error hoặc chuyển fallback source theo overload; với Single HTTP có thể giới hạn response wait, vẫn cấu hình URLSession timeout.
* sample: lấy latest value khi sampler phát event (nếu có value mới theo semantics), không tự gọi API; phù hợp render metrics theo tick.
* buffer(timeSpan:count:scheduler:): gom array theo time/count, giới hạn memory, xử lý batch rỗng nếu cần.
* window(timeSpan:count:scheduler:): tạo Observable con theo window; cần flatten/subscribe phù hợp.

### Q&A phỏng vấn

**Q: just và from khác gì?** just(array) phát một array, from(array) phát từng element.

**Q: map vs compactMap?** map giữ mỗi input thành output; compactMap bỏ kết quả nil. Không tự catch error.

**Q: scan vs reduce?** scan emit mỗi bước, reduce emit cuối khi complete; state UI dài hạn dùng scan.

**Q: distinctUntilChanged có bỏ mọi duplicate?** Chỉ bỏ duplicate liên tiếp theo equality/comparer.

**Q: Debounce hay throttle cho search?** Debounce chờ user dừng nhập; throttle giới hạn tần suất, phù hợp progress/gesture. Search vẫn cần latest cancellation.

**Q: take(1) có nghĩa timeout không?** Không; nếu source không emit/complete, vẫn chờ. Dùng timeout hoặc lifetime cancellation.

**Q: toArray trên NotificationCenter stream?** Không emit nếu stream không complete, có thể tích tụ memory; giới hạn take/buffer.

**Bài tập:** Dùng RxTest mô phỏng search "a", "ab", "abc"; so sánh debounce/throttle; viết reducer count với scan và test duplicate/empty.

### Tham khảo

[RxSwift ObservableType API](https://docs.rxswift.org/protocols/observabletype) · [ReactiveX operators](https://reactivex.io/documentation/operators.html) · [Flattening, Combining và Error Handling](operators-composition.md)

### Đọc tiếp

* [Higher-order Functions trong Swift và RxSwift](higher-order-functions.md): đối chiếu Array/Optional/Observable, map/flatMap/compactMap, reduce/scan.
* [Advanced Operators](operators-advanced.md): groupBy, amb, takeLast, custom operators, resource lifetime.
* [Traits](traits-subjects-relays.md): first(), take(1), single() và asSingle() khác kiểu kết quả/contract.
