# RxSwift thực tế, MVVM và RxTest

## 1. Quy trình thiết kế một reactive feature

Xác định input (query, tap, lifecycle), state (idle/loading/content/empty/error), event một lần (navigation/toast), nguồn dữ liệu (repository), ownership (screen/session/app) và cancellation. Chọn operator theo business rule: latest, first, sequential, parallel. Tách UI khỏi network và domain, inject scheduler/repository để test.

```text
View Input -> ViewModel transform -> UseCase -> Repository -> API / DB
View <- Driver<State> + Signal<Event> <- ViewModel
```

Driver: không error, main scheduler, share replay 1 while connected. Signal: không error, main scheduler, share không replay. Không dùng Relay như global store không có owner. State mới subscriber cần latest, event một lần không replay.

## 2. Search thực tế: debounce, cancellation, empty/error và retry

Ví dụ search catalog. Empty query phải clear UI và dispose request cũ; nếu debounce cả empty query thì sẽ chờ 300ms mới clear. Ví dụ bên dưới dùng flatMapLatest outer để cancel ngay khi query đổi, delay request mới bằng timer. Không debounce cả pipeline error; retry user-triggered dùng latest query và xử lý trong inner.

```swift
import RxSwift
import RxCocoa
import Foundation

struct SearchItem: Decodable, Equatable {
    let id: Int
    let title: String
}

protocol SearchRepository {
    func search(_ query: String) -> Single<[SearchItem]>
}

enum CatalogSearchState: Equatable {
    case idle, loading, content([SearchItem]), empty, failure(String)
}

final class CatalogSearchViewModel {
    struct Input {
        let query: Observable<String>
        let retry: Observable<Void>
    }

    struct Output {
        let state: Driver<CatalogSearchState>
    }

    private let repository: any SearchRepository
    private let scheduler: SchedulerType

    init(repository: any SearchRepository, scheduler: SchedulerType) {
        self.repository = repository
        self.scheduler = scheduler
    }

    func transform(_ input: Input) -> Output {
        let query = input.query
            .map { $0.trimmingCharacters(in: .whitespacesAndNewlines) }
            .distinctUntilChanged()
            .share(replay: 1, scope: .whileConnected)

        let requests = Observable.merge(
            query,
            input.retry.withLatestFrom(query)
        )

        let state = requests.flatMapLatest {
            [repository, scheduler] term -> Observable<CatalogSearchState> in
            guard !term.isEmpty else { return .just(.idle) }

            return Observable<Int>.timer(.milliseconds(300), scheduler: scheduler)
                .flatMap { _ in
                    Observable.deferred { repository.search(term).asObservable() }
                        .map { items -> CatalogSearchState in
                            items.isEmpty ? .empty : .content(items)
                        }
                        .catchAndReturn(.failure("Could not search"))
                        .startWith(.loading)
                }
        }
        .startWith(.idle)
        .asDriver(onErrorJustReturn: .failure("Search input failed"))

        return Output(state: state)
    }
}
```

* Query shared để retry lấy latest, không tạo hai source side effects. UI input thường ControlProperty không error; boundary xử lý nếu source custom fail.
* flatMapLatest dispose timer/request cũ ngay khi query mới emit. Empty -> idle ngay. Retry cũng delay 300ms để giữ ví dụ đơn giản; nếu muốn retry ngay, gắn trigger enum + delay riêng.
* Repository cold Single; producer disposable cancel task. Network error thành state trong inner, outer query sống.
* Loading phát khi request thật bắt đầu, không ngay khi gõ; nếu UX cần clear old content ngay, thêm .startWith(.idle) ở outer inner hoặc .searching state. Không đánh đồng error với empty.

### UIKit binding

```swift
import UIKit
import RxSwift
import RxCocoa

final class CatalogSearchController: UIViewController {
    private let searchBar = UISearchBar()
    private let retryButton = UIButton(type: .system)
    private let statusLabel = UILabel()
    private let bag = DisposeBag()
    private let model: CatalogSearchViewModel

    init(model: CatalogSearchViewModel) {
        self.model = model
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) { nil }

    override func viewDidLoad() {
        super.viewDidLoad()
        view.backgroundColor = .systemBackground
        retryButton.setTitle("Retry", for: .normal)
        let stack = UIStackView(arrangedSubviews: [
            searchBar, statusLabel, retryButton
        ])
        stack.axis = .vertical
        stack.spacing = 12
        stack.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(stack)
        NSLayoutConstraint.activate([
            stack.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            stack.leadingAnchor.constraint(equalTo: view.layoutMarginsGuide.leadingAnchor),
            stack.trailingAnchor.constraint(equalTo: view.layoutMarginsGuide.trailingAnchor)
        ])

        let output = model.transform(.init(
            query: searchBar.rx.text.orEmpty.asObservable(),
            retry: retryButton.rx.tap.asObservable()
        ))
        output.state.drive(onNext: { [weak self] in self?.render($0) })
            .disposed(by: bag)
    }

    private func render(_ state: CatalogSearchState) {
        switch state {
        case .idle: statusLabel.text = "Enter a query"
        case .loading: statusLabel.text = "Loading..."
        case .content(let items): statusLabel.text = items.map(\.title).joined(separator: ", ")
        case .empty: statusLabel.text = "No results"
        case .failure(let message): statusLabel.text = message
        }
    }
}
```

Ví dụ dùng label để tập trung reactive pipeline. Production table/collection diffable snapshot theo ID, progress/error UI, accessibility, localization. Controller giữ bag, weak capture; nếu binding lại cần reset bag hoặc tránh tạo subscriptions trùng. Driver chỉ đảm bảo UI scheduler ở runtime, chú ý compiler actor checks với Swift 6.

## 3. Login / submit: combineLatest, withLatestFrom, flatMapFirst

```swift
import RxSwift
import RxCocoa

struct LoginCredentials {
    let email: String
    let password: String
    var isValid: Bool { email.contains("@") && password.count >= 8 }
}

protocol LoginRepository {
    func login(_ credentials: LoginCredentials) -> Single<String>
}

enum LoginState {
    case idle, loading, success, failure(String)
}

func loginState(email: Observable<String>,
                password: Observable<String>,
                submit: Observable<Void>,
                repository: any LoginRepository) -> Driver<LoginState> {
    let credentials = Observable.combineLatest(email, password) {
        LoginCredentials(email: $0, password: $1)
    }

    return submit.withLatestFrom(credentials)
        .filter(\.isValid)
        .flatMapFirst { credentials -> Observable<LoginState> in
            repository.login(credentials).asObservable()
                .map { _ in LoginState.success }
                .catchAndReturn(.failure("Login failed"))
                .startWith(.loading)
        }
        .startWith(.idle)
        .asDriver(onErrorJustReturn: .failure("Input failed"))
}
```

Password không log; repository lưu token an toàn qua Keychain/session owner, ví dụ chỉ emit success UI. Button enable theo valid + notLoading (derive từ cùng shared state). flatMapFirst ignore tap khi request active, khác throttle. Khi logout/session đổi, cancel/invalidate request generation; auth server là source of truth. Với purchase, server idempotency, transaction verification/restore; UI gating không đủ chống duplicate charge.

## 4. Pagination: state và serialization

```text
trigger next -> guard !loading && hasMore -> fetch(cursor)
-> append unique IDs -> update cursor -> content / exhausted
error -> keep existing content + retry same cursor
new query/refresh -> new generation -> dispose old pagination
```

Dùng flatMapFirst cho load-more event khi request active, nhưng nếu UI cần queue chính xác event thì concatMap; duplicate triggers không nên queue vô hạn. Cursor cập nhật sau response success; reset khi refresh/search đổi; old response không ghi đè new query. Không gọi page 2 dựa trên indexPath cuối nếu cursor đã hết.

```swift
struct Page {
    let items: [SearchItem]
    let nextCursor: String?
}

protocol PageRepository {
    func page(cursor: String?) -> Single<Page>
}

struct PageState {
    var items: [SearchItem] = []
    var cursor: String?
    var hasMore = true
    var loading = false
}

func appendPage(_ page: Page, to state: PageState) -> PageState {
    var result = state
    var ids = Set(result.items.map(\.id))
    for item in page.items where ids.insert(item.id).inserted {
        result.items.append(item)
    }
    result.cursor = page.nextCursor
    result.hasMore = page.nextCursor != nil
    result.loading = false
    return result
}
```

Pure reducer để test dedup/hasMore. ViewModel serialize input/state transitions trên scheduler cố định hoặc actor; snapshot state khi nhận trigger, flatMapFirst request, map success/error thành action và reduce. Refresh/new query tạo generation mới hoặc flatMapLatest outer paging session. Không gọi relay.accept từ nhiều threads dựa trên relay.value vì compound read-modify-write không atomic.

## 5. Cache-then-network và offline state

```swift
enum CatalogState {
    case cached([SearchItem])
    case fresh([SearchItem])
    case failure(String)
}

protocol CatalogRepository {
    func cached() -> Single<[SearchItem]>
    func remote() -> Single<[SearchItem]>
}

func catalog(_ repository: any CatalogRepository) -> Observable<CatalogState> {
    let local = Observable.deferred {
        repository.cached().asObservable()
            .map { CatalogState.cached($0) }
            .catch { _ in .empty() }
    }
    let remote = Observable.deferred {
        repository.remote().asObservable()
            .map { CatalogState.fresh($0) }
            .catchAndReturn(.failure("Refresh failed"))
    }
    return Observable.concat([local, remote])
}
```

Local Single complete/error -> remote subscribe. Cache failure không chặn remote theo policy. Production state giữ cached items khi remote fail, hiển thị stale/refresh error; repository remote upsert local theo revision/field ownership. Không concat live local query vô hạn với remote vì remote không chạy. Share kết quả để UI/metrics không duplicate request.

## 6. Traits, Subjects và ownership

| API | Phát gì | Chọn khi |
| --- | --- | --- |
| PublishSubject | Next mới + terminal | Event bus có terminal |
| BehaviorSubject | Latest/initial + terminal | State stream có thể error |
| ReplaySubject | N values + terminal | Bounded replay có chủ đích |
| PublishRelay | Next mới, không terminal | UI event |
| BehaviorRelay | Latest/initial, không terminal | State owner |
| Single | 1 success hoặc error | HTTP response |
| Maybe | 0 hoặc 1 success/error | Optional lookup |
| Completable | Complete hoặc error | Delete/upload không cần value |
| Driver | Main, no error, replay 1 while connected | Render state |
| Signal | Main, no error, no replay | Navigate/toast |

DisposeBag hủy subscriptions lúc deinit. Nếu self giữ bag và subscription capture self mạnh, bag không deinit. Weak capture hoặc thay ownership; withUnretained trước replay có thể giữ self qua buffer. Cell reuse tạo bag mới, cancel image request; app/session scope không giữ VC. CompositeDisposable cho nhóm động, SerialDisposable thay subscription cũ.

## 7. RxTest: virtual time, hot/cold và assert timeline

RxTest TestScheduler dùng virtual ticks, không sleep. Hot observable event theo absolute virtual time, cold theo offset từ subscription. TestScheduler.createObserver ghi Recorded events; scheduler.start() chạy virtual queue. Inject scheduler thay hardcode MainScheduler trong domain pipeline.

### Test flatMapLatest hủy request cũ

```swift
import XCTest
import RxSwift
import RxTest

final class SearchOperatorsTests: XCTestCase {
    func testLatestDisposesOldInner() {
        let scheduler = TestScheduler(initialClock: 0)
        let queries = scheduler.createHotObservable([
            .next(10, "a"), .next(20, "ab"), .completed(100)
        ])
        let first = scheduler.createColdObservable([
            .next(30, "old"), .completed(31)
        ])
        let second = scheduler.createColdObservable([
            .next(5, "new"), .completed(6)
        ])
        let observer = scheduler.createObserver(String.self)
        let bag = DisposeBag()

        queries.asObservable()
            .flatMapLatest { $0 == "a" ? first.asObservable() : second.asObservable() }
            .subscribe(observer)
            .disposed(by: bag)
        scheduler.start()

        XCTAssertEqual(observer.events, [.next(25, "new"), .completed(100)])
        XCTAssertEqual(first.subscriptions, [Subscription(10, 20)])
        XCTAssertEqual(second.subscriptions, [Subscription(20, 26)])
    }
}
```

First inner bắt đầu t10, bị dispose t20 trước response t40. Second emit t25, complete t26. Outer complete t100. Assert subscription timeline kiểm tra cancellation, không chỉ output. Producer thật phải implement task.cancel().

### Test debounce

```swift
func testDebounce() {
    let scheduler = TestScheduler(initialClock: 0)
    let source = scheduler.createHotObservable([
        .next(10, "a"), .next(20, "ab"), .next(80, "abc"),
        .completed(150)
    ])
    let observer = scheduler.createObserver(String.self)
    let bag = DisposeBag()

    source.debounce(.seconds(30), scheduler: scheduler)
        .subscribe(observer)
        .disposed(by: bag)
    scheduler.start()

    XCTAssertEqual(observer.events, [
        .next(50, "ab"), .next(110, "abc"), .completed(150)
    ])
}
```

TestScheduler mặc định resolution 1 virtual second per tick, .seconds(30) tương ứng 30 ticks. Không dùng real-world delay trong unit test; test completion flush pending, repeated input, empty query. Đoạn test có thể đặt trong SearchOperatorsTests.

### Test error recovery và sharing

```swift
func testInnerErrorKeepsSearchAlive() {
    let scheduler = TestScheduler(initialClock: 0)
    let source = scheduler.createHotObservable([
        .next(10, "bad"), .next(20, "good"), .completed(50)
    ])
    let observer = scheduler.createObserver(String.self)
    let bag = DisposeBag()
    enum Failure: Error { case offline }

    source.flatMapLatest { query -> Observable<String> in
        let inner: Observable<String> = query == "bad"
            ? .error(Failure.offline) : .just("result")
        return inner.catchAndReturn("error")
    }
    .subscribe(observer)
    .disposed(by: bag)
    scheduler.start()

    XCTAssertEqual(observer.events, [
        .next(10, "error"), .next(20, "result"), .completed(50)
    ])
}
```

Test share bằng cold source + hai observers subscribe overlap, assert upstream subscriptions chỉ một; sau disconnect/reconnect .whileConnected tạo subscription mới. Test flatMapFirst double tap, concatMap order, zip/combineLatest/withLatestFrom initial state, retry attempt count, scheduler injection. Không test Driver bằng cách giả định main scheduler dùng virtual clock; tách core Observable state pipeline, test core rồi kiểm tra UI traits/binding riêng.

## 8. Debugging và performance

* API duplicate: tìm nhiều subscriptions trên cold source, transform gọi nhiều lần, share scope, ViewController binding lặp.
* Search chết sau lỗi: catch đặt sai vị trí outer, terminal error; chuyển error thành state inner.
* Loading không tắt: startWith loading nhưng inner never/timeout/cancel, loading counter shared sai; state machine theo active request.
* UI update background: observe(on:) trước render, Driver; không assume subscribe(on:) đưa UI về main.
* Memory leak: bag/closure/relay/notification/timer, replay buffer giữ object; Instruments Memory Graph, deinit logs.
* Overload: merge maxConcurrent, bounded queue, debounce/throttle/sample; Observable không demand-based backpressure.
* Rx debug event có thể chứa token/password/SMS; chỉ log sanitized metadata trong debug.

### Q&A phỏng vấn tình huống

**Q: Search A chậm, B nhanh, A ghi đè B?** flatMapLatest + producer cancellation, generation nếu side effect không cancel; debounce và distinct phù hợp.

**Q: Tại sao empty query không clear?** filter empty trước flattening làm mất event; branch empty trong flatMapLatest để dispose cũ và emit idle.

**Q: Retry button dùng combineLatest hay withLatestFrom?** withLatestFrom(query), chỉ tap trigger. combineLatest tự phát khi query đổi.

**Q: Double tap login khi request mất 5s?** flatMapFirst/state gate bỏ tap trong suốt inner; throttle 500ms không đủ. Server idempotency nếu operation yêu cầu.

**Q: API response lỗi làm search không chạy nữa?** Catch inner thành error state; outer catch fallback complete làm input stream kết thúc.

**Q: Khi nào Driver không thích hợp?** Domain stream cần error/worker scheduler, dùng Observable/Single; Driver ở UI boundary.

**Q: Chia sẻ loading/data/error thế nào?** Một shared state stream rồi derive outputs, không subscribe API riêng. Scope theo screen/session.

**Q: RxTest kiểm tra cancellation bằng gì?** Recorded subscription start/end của cold inner, output timeline; integration test kiểm tra task.cancel.

**Q: RxSwift vs async/await?** Async/await rõ với one-shot/structured tasks; Rx compose event stream/operators/lifetime. Có thể bridge nhưng cần cancellation và scheduler/isolation đúng. Không migrate chỉ vì công nghệ mới.

**Bài tập:** Build search + retry/empty/error, login double tap, pagination reset, cache-then-network; viết RxTest test latest, error recovery, sharing và bounded upload. Giải thích mỗi operator lựa chọn.

### Tham khảo

[RxSwift](https://github.com/ReactiveX/RxSwift) · [RxTest API](https://docs.rxswift.org/) · [Creation/Transformation/Filtering](operators-foundation.md) · [Composition/Error/Sharing](operators-composition.md)

### Đọc tiếp

* [Traits, Subjects, Relays và RxCocoa](traits-subjects-relays.md): ví dụ Single API, Maybe cache, Completable save, Driver/Signal UI binding và lifecycle.
* [Higher-order Functions](higher-order-functions.md): DTO mapping thuần Swift và Rx.
* [Advanced Operators](operators-advanced.md): thêm RxTest edge cases, groupBy, batching, custom operator.
