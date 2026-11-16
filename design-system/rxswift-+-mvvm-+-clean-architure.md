# 😄 RxSwift + MVVM + Clean Architure

### Định nghĩa RxSwift

RxSwift biểu diễn dữ liệu/tương tác theo stream. Observable phát next, error hoặc completed; Observer nhận event; Disposable hủy subscription, DisposeBag dispose khi deinit.

| Loại | Đặc điểm |
| --- | --- |
| Observable | 0...n event, có thể error/completed |
| Single | 1 success hoặc error |
| Completable | completed hoặc error, không có value |
| PublishSubject | Subscriber mới chỉ nhận event sau subscribe |
| BehaviorSubject | Có giá trị hiện tại, phát lại cho subscriber mới |
| ReplaySubject | Replay số event cấu hình |
| PublishRelay / BehaviorRelay | RxRelay, không phát error/completed |
| Driver | RxCocoa, main scheduler, không error, share replay 1 while connected |

### Operator và Scheduler

* map chuyển đổi value; filter lọc; distinctUntilChanged bỏ giá trị liên tiếp giống nhau; debounce chờ user ngừng nhập.
* flatMap subscribe các inner stream đồng thời; flatMapLatest chỉ giữ inner mới nhất (dispose cũ; network chỉ thật sự cancel nếu disposable hỗ trợ); concatMap chạy tuần tự.
* merge gộp stream, zip ghép theo cặp, combineLatest phát khi bất kỳ stream thay đổi sau khi mọi stream đã có giá trị.
* subscribe(on:) ảnh hưởng việc subscribe/unsubscribe upstream; observe(on:) đổi scheduler nhận event downstream.
* catchError/catchAndReturn xử lý lỗi; đặt catch trong inner stream để outer search vẫn sống. share(replay:scope:) tránh gọi API lặp cho nhiều subscriber; chú ý lifetime và cache.

### MVVM + Clean Architecture

Presentation = View + ViewModel, Domain = Entity + UseCase + Repository protocol, Data = Repository implementation + API/CoreData/SDK. ViewModel không trực tiếp phụ thuộc concrete network SDK; composition root inject UseCase. MVVM không bắt buộc two-way binding: một chiều input → state giúp dễ test.

```swift
import RxSwift
import RxCocoa

protocol SearchPlaces {
    func execute(_ query: String) -> Single<[String]>
}

final class SearchViewModel {
    let results: Driver<[String]>

    init(query: Observable<String>, useCase: any SearchPlaces) {
        results = query
            .debounce(.milliseconds(300), scheduler: MainScheduler.instance)
            .distinctUntilChanged()
            .flatMapLatest { text -> Observable<[String]> in
                guard !text.isEmpty else { return .just([]) }
                return useCase.execute(text).asObservable()
                    .catchAndReturn([])
            }
            .asDriver(onErrorJustReturn: [])
    }
}
```

Ví dụ đơn giản hóa error thành empty result; production nên có state idle/loading/content/empty/error, không đánh đồng lỗi với kết quả rỗng. Dùng RxTest TestScheduler để test debounce, cancellation và latest result; inject scheduler thay vì hardcode trong testable ViewModel. Giữ DisposeBag theo lifetime view, tạo bag mới khi reuse cell, tránh retain cycle trong subscribe closure.

### Q&A

**Q: Khi nào dùng flatMapLatest?** Search địa chỉ/di chuyển camera, chỉ hiển thị kết quả query mới nhất.

**Q: Cold vs hot Observable?** Cold thường tạo execution riêng mỗi subscription; hot chia sẻ nguồn event, subscriber có thể bỏ lỡ event cũ.

**Q: Tại sao Driver dùng cho UI?** Không error, observe trên main scheduler, share side effects; Signal không replay latest value.

**Liên hệ:** Map4D có RxSwift + MVVM + Clean Architecture; sơ đồ trên là cách triển khai tham khảo, không khẳng định source dự án.

## RxSwift chuyên sâu từ cơ bản đến thực tế

### Observable cold/hot và disposal

```swift
import RxSwift

let cold = Observable<Int>.create { observer in
    print("Start work")
    observer.onNext(1)
    observer.onNext(2)
    observer.onCompleted()
    return Disposables.create { print("Disposed") }
}
let bag = DisposeBag()
cold.subscribe(onNext: { print("A", $0) }).disposed(by: bag)
cold.subscribe(onNext: { print("B", $0) }).disposed(by: bag)
```

Mỗi subscription cold chạy producer riêng; terminal event kết thúc stream. Hot subject nhận event bên ngoài, subscriber đến sau không nhất thiết nhận dữ liệu cũ. Disposable hủy việc quan sát; nguồn tự viết phải trả cleanup phù hợp để hủy URLSessionTask/observer.

### Operator: phân biệt bằng timeline

```text
Search:      A-----AB------ABC
flatMap:     request A, AB, ABC có thể cùng tồn tại
flatMapLatest: dispose A khi AB đến, dispose AB khi ABC đến
concatMap:   chạy A xong mới AB, rồi ABC
```

* combineLatest: sau khi mỗi nguồn phát ít nhất một lần, bất kỳ nguồn thay đổi đều phát tuple mới. Phù hợp form validation.
* zip: ghép event thứ n của từng nguồn, có thể chờ lâu nếu tốc độ khác nhau.
* withLatestFrom: event submit lấy giá trị form mới nhất, thay đổi form không tự submit.
* throttle khác debounce: throttle giới hạn tần suất, debounce đợi khoảng yên lặng.
* share(replay: 1, scope: .whileConnected) chia sẻ side effect trong lifetime subscriber; .forever có thể giữ event/terminal lâu hơn, cân nhắc cache.
* Single cho API một kết quả; Driver cho UI, Signal cho event không replay (navigation/toast); Relay không phát error/completed.

```swift
import RxSwift
import RxRelay

let email = BehaviorRelay<String>(value: "")
let password = BehaviorRelay<String>(value: "")
let valid = Observable.combineLatest(email.asObservable(),
                                     password.asObservable()) {
    $0.contains("@") && $1.count >= 8
}
.distinctUntilChanged()
```

Form validation trên chỉ minh họa UI, backend vẫn xác thực. Để submit, button tap.withLatestFrom(combineLatest(...)), dùng flatMapFirst nếu muốn bỏ lần tap khi request đang chạy.

### MVVM Input/Output và state

```swift
import RxSwift
import RxCocoa

enum SearchState {
    case idle, loading, results([String]), failure(String)
}

protocol PlaceSearch {
    func search(_ term: String) -> Single<[String]>
}

final class PlaceViewModel {
    struct Input {
        let query: Observable<String>
    }

    struct Output {
        let state: Driver<SearchState>
    }

    private let repository: any PlaceSearch

    init(repository: any PlaceSearch) { self.repository = repository }

    func transform(_ input: Input) -> Output {
        let state = input.query
            .debounce(.milliseconds(300), scheduler: MainScheduler.instance)
            .distinctUntilChanged()
            .flatMapLatest { [repository] query -> Observable<SearchState> in
                guard !query.isEmpty else { return .just(.idle) }
                return repository.search(query)
                    .asObservable()
                    .map { SearchState.results($0) }
                    .catchAndReturn(.failure("Search failed"))
                    .startWith(.loading)
            }
            .asDriver(onErrorJustReturn: .failure("Search failed"))
        return Output(state: state)
    }
}
```

ViewController bind searchBar.rx.text.orEmpty → Input; output.state.drive(onNext:) để render. Không subscribe API riêng từ nhiều binding; Driver chia sẻ. Error được catch trong inner để outer query không terminate. UseCase có thể thay repository nếu business rule phức tạp.

### Scheduler và memory

subscribe(on:) đổi nơi subscribe/upstream side effects, không buộc mọi callback chạy ở scheduler đó; observe(on:) chuyển event downstream. UIKit render MainScheduler.instance. Nếu CPU mapping nặng, observe(on: background scheduler) trước map rồi về main trước render; không nhầm network callback thread. DisposeBag gắn vòng đời VC; cell reuse reset bag. self → bag → subscription → closure → self có thể cycle, dùng weak capture khi cần.

### Test bằng RxTest

Dùng TestScheduler tạo hot query events và cold fake response, inject scheduler/debounce duration. Assert timeline loading/results, chỉ latest response, error rồi query mới vẫn hoạt động. Test dispose hủy fake request. Không dùng sleep thực trong unit test.

**Q: Driver vs Observable?** Driver không error, main scheduler, share replay 1 while connected; Observable đa dụng, scheduler tùy cấu hình.

**Q: Subscribe nhiều lần tại sao gọi API nhiều lần?** Cold stream tạo work mỗi subscription; share khi cần chung execution.

**Bài tập:** Search có debounce, retry, empty/loading/error; đổi flatMapLatest thành flatMap và quan sát stale result.

## Học sâu RxSwift theo từng nhóm

* [Creation, Transformation, Filtering và Time](rxswift/operators-foundation.md): input/output, timeline, code, use case và lỗi thường gặp.
* [Flattening, Combining, Error và Sharing](rxswift/operators-composition.md): chọn flatMap/flatMapLatest/concatMap/flatMapFirst, zip/combineLatest/withLatestFrom, retry/catch và scheduler.
* [RxSwift thực tế, MVVM và RxTest](rxswift/practical-patterns.md): search, login, pagination, API state, UIKit binding và virtual-time tests.

### Q&A tổng hợp

**Q: Observable vs Single vs Maybe vs Completable?** Observable 0..n values; Single đúng một success/error; Maybe một success hoặc complete rỗng/error; Completable chỉ completed/error. Infallible không phát error. Chọn trait theo contract, không ép mọi stream thành Single.

**Q: Subject vs Relay?** Subject có terminal error/completed; Relay không terminal, phù hợp UI state/events. BehaviorRelay giữ latest value, PublishRelay không replay; Relay không tự thread-safe cho compound mutation.

**Q: Driver vs Signal?** Cả hai UI traits không error, main scheduler và share; Driver replay latest 1 while connected, Signal không replay, phù hợp event navigation/toast. Không xem Signal là durable event queue.

**Q: Vì sao cần phân biệt state và event?** State mới subscriber cần giá trị hiện tại; event một lần không nên replay để tránh navigate/show toast lại.

**Q: RxSwift có backpressure không?** Observable không có demand protocol như Combine; giới hạn concurrency, buffer/window, throttle/sample và bounded queue theo nghiệp vụ.
