# RxSwift Operators – Flattening, Combining, Error và Sharing

## 1. Flattening: map, flatMap, flatMapLatest, concatMap, flatMapFirst

map chuyển T sang R; nếu R là Observable<T>, kết quả là stream lồng nhau. Flattening quản lý subscription của inner observable. Chọn theo yêu cầu nghiệp vụ, không theo sở thích.

| Toán tử | Khi outer phát event mới | Use case |
| --- | --- | --- |
| flatMap | Subscribe inner mới, giữ inner cũ | Độc lập, upload nhiều item |
| flatMapLatest | Dispose inner trước, lấy inner mới | Search, filter, map camera |
| concatMap | Xếp hàng, subscribe inner lần lượt | Ordered writes/uploads |
| flatMapFirst | Bỏ event mới nếu inner đang chạy | Login, submit, purchase UI gate |
| switchLatest | Chuyển sang inner mới nhất của stream lồng | Dynamic source |
| merge(maxConcurrent:) | Giới hạn số inner active, queue còn lại | Bounded parallel transfer |

Timeline minh họa cold request: A đến t=0, B t=10, C t=20; mỗi request mất 50 time units. Outer vẫn mở.

```text
outer:         A---B---C------------------------>
flatMap:                     a---b---c-------->
flatMapLatest:                       c-------->
concatMap:                   a----------b----------c
flatMapFirst:                a---------------->
```

flatMap result không bảo đảm thứ tự A,B,C nếu response latency khác. concatMap chờ inner hoàn thành, nếu inner không complete thì queue bị kẹt. flatMapLatest dispose inner cũ; không bảo đảm server rollback side effect. flatMapFirst bỏ event chứ không queue.

```swift
import RxSwift

protocol ProductAPI {
    func search(_ query: String) -> Single<[String]>
    func upload(_ id: String) -> Completable
}

func latestSearch(query: Observable<String>, api: any ProductAPI)
    -> Observable<[String]> {
    query.flatMapLatest { api.search($0).asObservable() }
}

func orderedUploads(ids: Observable<String>, api: any ProductAPI)
    -> Observable<String> {
    ids.concatMap { id in
        api.upload(id).andThen(Observable.just(id))
    }
}

func parallelUploads(ids: [String], api: any ProductAPI)
    -> Observable<String> {
    Observable.from(ids)
        .map { id in api.upload(id).andThen(Observable.just(id)) }
        .merge(maxConcurrent: 3)
}
```

Các inner phải cold hoặc deferred để subscription mới bắt đầu work; tạo eager Future-like work trước merge/concat sẽ phá giới hạn thực tế. merge maxConcurrent giới hạn subscriptions, không giới hạn queue memory nếu outer phát vô hạn. Cần bounded input/backpressure policy. RxSwift Observable không có demand-based backpressure như Combine.

### Thực tế: chọn toán tử

* Gõ search/camera viewport: flatMapLatest, debounce và state error inner.
* Tap login khi request chạy: flatMapFirst, disable button; backend vẫn idempotent nếu cần.
* Sync event cần đúng thứ tự: concatMap; handle error per item để queue không chết nếu policy cho phép.
* Download nhiều thumbnail: bounded merge, cache, cancel theo screen lifetime.
* Mutation như charge/redeem: không dùng flatMapLatest để "hủy giao dịch"; server idempotency và query status.
* Switch user/session: switchLatest vào session stream mới, invalidate old credentials/cache và guard generation.

## 2. Combining: merge, concat, zip, combineLatest, withLatestFrom

Giả sử A phát a1 t10, a2 t30; B phát b1 t20, b2 t40. Hai stream complete sau event cuối.

```text
A:             --a1------a2--------|
B:             -----b1------b2----|
merge:         --a1--b1--a2--b2---|
zip:           ----- (a1,b1) -- (a2,b2) --|
combineLatest: ----- (a1,b1) (a2,b1) (a2,b2) --|
A.withLatestFrom(B): -----------b1--------|
```

Timeline tượng trưng, chú ý complete/error của nguồn. merge hợp nhất events cùng element type theo thứ tự tới; zip ghép event thứ n từng stream, buffer khi lệch tốc độ; combineLatest cần mỗi nguồn emit ít nhất một lần, rồi phát khi bất kỳ nguồn đổi; withLatestFrom chỉ trigger bởi primary stream, bỏ trigger trước secondary có value.

### combineLatest – form validation

```swift
import RxSwift
import RxRelay

let email = BehaviorRelay(value: "")
let password = BehaviorRelay(value: "")
let valid = Observable.combineLatest(email.asObservable(),
                                     password.asObservable()) { email, password in
    email.contains("@") && password.count >= 8
}
.distinctUntilChanged()
```

Ví dụ UI validation đơn giản, backend vẫn kiểm tra. BehaviorRelay initial values giúp emit ngay. Nếu PublishSubject chưa emit, combineLatest chưa có đủ tuple. Không dùng combineLatest để tự submit mỗi lần user gõ.

### withLatestFrom – submit form

```swift
struct Credentials {
    let email: String
    let password: String
}

let submit = PublishSubject<Void>()
let credentials = Observable.combineLatest(email.asObservable(),
                                           password.asObservable()) {
    Credentials(email: $0, password: $1)
}
let submitCredentials = submit.withLatestFrom(credentials)
```

Chỉ tap trigger request; dùng filter/validation và flatMapFirst tiếp theo. Không log password trong debug/operator. withLatestFrom khác combineLatest: thay password không tự submit. Secondary latest state không nhất thiết phải Relay nếu đã có value phù hợp.

### zip – ghép kết quả API

```swift
struct Profile { let name: String }
struct Dashboard { let profile: Profile; let balance: Int }

func dashboard(profile: Single<Profile>, balance: Single<Int>)
    -> Single<Dashboard> {
    Single.zip(profile, balance) { Dashboard(profile: $0, balance: $1) }
}
```

Hai Single được subscribe, có thể chạy concurrent nếu sources async; zip chỉ trả khi cả hai success, một error làm kết quả error. Không dùng zip cho hai event stream lệch tốc độ vô hạn nếu buffer có thể lớn. Nếu cần profile trước để biết account ID, dùng flatMap chứ không zip.

### merge vs concat

```swift
let local = Observable.of("Cached")
let remote = Observable.of("Fresh")
Observable.concat([local, remote])
    .subscribe(onNext: { print($0) })
    .disposed(by: bag)
```

Concat subscribe remote sau local complete, phù hợp cache-then-network khi local source hữu hạn. Nếu local live query không complete, remote không chạy; trigger remote riêng và merge state theo policy. merge không đảm bảo cached trước fresh, cần timestamp/version.

## 3. Error handling và retry

Error là terminal event của source. catch chuyển sang fallback observable; không tiếp tục source cũ sau error. retry resubscribe source; nếu source có side effect, nó có thể chạy lại.

```swift
enum APIError: Error { case offline, unauthorized }

func load(_ api: @escaping () -> Single<[String]>) -> Observable<[String]> {
    Observable.deferred { api().asObservable() }
        .retry(2) // tối đa 2 attempts, tức 1 lần retry
        .catch { _ in .just([]) }
}
```

Ví dụ minh họa retry rồi fallback, không dùng empty thay error state trong production. retry(2) gồm lần đầu + một lần retry; retry() vô hạn nguy hiểm. retry trên eager source không tạo request mới nếu factory không cold. Với HTTP chỉ retry transient error, request idempotent/Idempotency-Key được backend hỗ trợ, bounded delay/backoff/jitter, tôn trọng Retry-After. 401 refresh single-flight, replay tối đa một lần, không retry invalid credential vô hạn.

### catch ở outer vs inner

```swift
enum SearchResult {
    case loading, content([String]), failure
}

func resilientSearch(_ query: Observable<String>, api: any ProductAPI)
    -> Observable<SearchResult> {
    query.flatMapLatest { term in
        api.search(term).asObservable()
            .map { SearchResult.content($0) }
            .catchAndReturn(.failure)
            .startWith(.loading)
    }
}
```

Nếu catchAndReturn ở ngoài flatMapLatest, inner error terminate outer subscription, fallback phát một value rồi complete; search sau không còn chạy. Catch bên trong giúp query stream tiếp tục. Error của outer query vẫn cần xử lý tại boundary. Không nuốt lỗi auth/payment; phân loại domain error.

### retry(when:) bounded backoff

```swift
func retryTransient<T>(_ source: Observable<T>,
                       scheduler: SchedulerType,
                       canRetry: @escaping (Error) -> Bool) -> Observable<T> {
    source.retry(when: { errors in
        errors.enumerated().flatMap { index, error -> Observable<Int> in
            guard index < 2, canRetry(error) else {
                return .error(error)
            }
            return Observable<Int>.timer(.seconds(1 << index),
                                         scheduler: scheduler)
        }
    })
}
```

Tối đa 3 attempts, delay 1s/2s, propagate error cuối. Production thêm jitter/Retry-After, cancellation và classification. Không dùng retryWhen tên cũ khi API hiện tại có retry(when:). Source phải cold; dispose hủy timer/subscription.

### materialize / dematerialize

materialize chuyển next/error/completed thành Event values, stream bên ngoài complete khi source terminal. Dùng inspect event/branch success-error khi cần nhiều output, không làm nguồn cũ sống lại. dematerialize chuyển Event stream về semantics terminal.

```swift
Observable<Int>.error(APIError.offline)
    .materialize()
    .subscribe(onNext: { event in
        if case .error(let error) = event { print(error) }
    })
    .disposed(by: bag)
```

do(onNext:onError:onCompleted:onDispose:) dùng side effects/debug/metrics, không transform. Không gọi nghiệp vụ write trong do nhiều nơi nếu stream có nhiều subscriptions; không log token/password.

## 4. Sharing, replay và connectable

Cold API: mỗi subscribe có thể tạo request. share chia sẻ một upstream subscription giữa subscribers overlap; replay N giữ N values theo scope. Không phải permanent disk cache.

```swift
let shared = requestData(url)
    .share(replay: 1, scope: .whileConnected)

shared.subscribe(onNext: { data in
    print("UI", data.count)
}).disposed(by: bag)

shared.subscribe(onNext: { data in
    print("Metrics", data.count)
}).disposed(by: bag)
```

requestData là cold URLSession wrapper ở bài Foundation; url và bag do caller cung cấp. Nếu hai subscribers overlap, một upstream request. Nếu source synchronous/complete trước subscriber thứ hai, .whileConnected có thể tạo connection mới. Không hứa "chỉ gọi API một lần mãi mãi".

| API / scope | Ý nghĩa |
| --- | --- |
| share(replay:0) | Không replay, share connection |
| .whileConnected | Subject mới theo connection; release buffer khi disconnect/terminal |
| .forever | Subject/replay buffer dùng qua connections, có thể giữ terminal event |
| publish() | ConnectableObservable không replay; connect thủ công |
| replay(n) | ConnectableObservable replay n values |
| refCount() | Connect khi subscriber đầu, disconnect khi subscriber cuối rời |

```swift
let connectable = Observable<Int>.interval(.seconds(1),
                                          scheduler: MainScheduler.instance)
    .publish()
connectable.subscribe(onNext: { print("A", $0) }).disposed(by: bag)
connectable.subscribe(onNext: { print("B", $0) }).disposed(by: bag)
let connection = connectable.connect()
connection.disposed(by: bag)
```

connect() là disposable riêng; cần giữ lifetime. replay có thể giữ object lớn hoặc self trong buffer, chú ý memory. withUnretained trước replay/share buffer có thể giữ tuple chứa owner; đặt weak capture ở subscriber hoặc thiết kế ownership phù hợp.

## 5. Scheduler và lifetime operators

subscribe(on:) điều khiển subscription/unsubscription side effects, không cam kết callback producer chạy ở scheduler đó. observe(on:) điều khiển downstream event delivery. URLSession tự quản lý network, parse CPU-heavy có thể chuyển scheduler trước map.

```swift
let worker = ConcurrentDispatchQueueScheduler(qos: .userInitiated)
requestData(url)
    .observe(on: worker)
    .map { try JSONDecoder().decode([String].self, from: $0) }
    .observe(on: MainScheduler.instance)
    .subscribe(onNext: { print($0) },
               onError: { print("Request failed") })
    .disposed(by: bag)
```

observe(on: MainScheduler.instance) không tự làm mọi @MainActor compiler checks biến mất; bridge UIKit/Swift Concurrency theo SDK annotations. MainScheduler.instance có thể thực thi ngay trên main; asyncInstance luôn enqueue async. SerialDispatchQueueScheduler cho serialized work; ConcurrentDispatchQueueScheduler có thể concurrent source work nhưng Observable event contract phải serialize. CurrentThreadScheduler/trampoline hữu ích nội bộ; không block scheduler với semaphore đợi network.

* DisposeBag: dispose khi deinit; replace bag cho cell/screen binding mới.
* CompositeDisposable: thêm/xóa disposable động, dispose toàn bộ.
* SerialDisposable: thay disposable sẽ dispose cái cũ; nếu đã disposed, cái mới cũng dispose.
* take(until: lifetime): kết thúc theo notifier; không thay producer cleanup.
* debug: log event khi dev, tuyệt đối redact secret.

### Q&A phỏng vấn

**Q: flatMapLatest có cancel HTTP thật không?** Dispose subscription cũ; producer phải cancel URLSessionTask. Server có thể vẫn xử lý side effect.

**Q: flatMapFirst vs throttle?** First bỏ event khi inner active bất kể duration; throttle chỉ giới hạn thời gian.

**Q: concatMap và merge(maxConcurrent:1) khác gì?** Đều serialize subscription inner trong use case cold; concatMap trực tiếp map+queue; cần chú ý selector eager work.

**Q: zip vs combineLatest?** zip ghép theo index, combineLatest dùng latest của mỗi nguồn khi một nguồn đổi sau initial values.

**Q: withLatestFrom khi secondary chưa emit?** Trigger bị bỏ, không đợi để replay. Dùng initial state nếu phù hợp.

**Q: retry(3) bao nhiêu lần?** Tối đa 3 attempts gồm lần đầu, tức tối đa 2 retry.

**Q: catch ngoài flatMapLatest vì sao search chết?** Inner error terminate flattened stream, fallback complete; catch inner giữ outer sống.

**Q: share replay 1 có là cache vĩnh viễn?** Không. Scope/connection/terminal quyết định lifetime, cần repository cache riêng.

**Q: observe(on:) vs subscribe(on:)?** Downstream delivery vs subscription/unsubscription; callback của producer có scheduler riêng.

**Bài tập:** Search có lỗi rồi query tiếp, upload 10 file max 3 concurrent, login double tap, hai subscribers cùng API và test share reconnect.

### Tham khảo

[RxSwift ObservableType](https://docs.rxswift.org/protocols/observabletype) · [RxSwift SubjectLifetimeScope](https://docs.rxswift.org/enums/subjectlifetimescope) · [Ví dụ thực tế và RxTest](practical-patterns.md)

### Đọc tiếp

* [Advanced Operators](operators-advanced.md): edge cases, batching, groupBy, amb, custom operators, async/await bridge.
* [Traits và Subjects](traits-subjects-relays.md): chọn Single/Maybe/Completable/Driver/Signal, conversion và binding.
