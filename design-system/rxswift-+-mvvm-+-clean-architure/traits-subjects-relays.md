# RxSwift Traits, Subjects, Relays và RxCocoa Binding

## 1. Trait là gì?

Trait là abstraction có contract cụ thể hơn Observable. API signature cho caller biết sequence có bao nhiêu giá trị, có thể lỗi không, và (với UI traits) scheduler/sharing behavior. Không phải tất cả trait đều có replay, main-thread delivery hoặc tự share. Trait cũng không đảm bảo tác vụ chắc chắn hoàn tất: network có thể pending, subscription có thể bị dispose.

```text
Observable<T> (0...n next, optional terminal)
  ├─ Single<T>       (success T hoặc failure Error)
  ├─ Maybe<T>        (success T, completed rỗng, hoặc error)
  ├─ Completable     (completed hoặc error, không next)
  ├─ Infallible<T>   (0...n next, không error)
  └─ RxCocoa SharedSequence
       ├─ Driver<T>  (main, no error, replay latest)
       └─ Signal<T>  (main, no error, no replay)
```

PrimitiveSequence<Trait, Element> là abstraction chung của Single/Maybe/Completable. Completable có Element = Never. ControlProperty và ControlEvent là UI traits khác, không phải PrimitiveSequence. RxRelay types là subject-like wrappers không có terminal events, không phải SharedSequence.

| Kiểu | Contract khi terminate | Scheduler / sharing mặc định | Use case |
| --- | --- | --- | --- |
| Observable<T> | 0...n next, completed/error | Tùy producer/operator | Stream tổng quát |
| Single<T> | Một success hoặc failure | Không tự main/share | HTTP response |
| Maybe<T> | Một success, empty completion hoặc error | Không tự main/share | Local optional lookup |
| Completable | Completed hoặc error | Không tự main/share | Delete, write, upload không cần payload |
| Infallible<T> | 0...n next, có thể completed, không error | Không tự main/share | Pure/domain state stream không lỗi |
| Driver<T> | Không error; có thể completed | Main, replay 1 while connected | UI state |
| Signal<T> | Không error; có thể completed | Main, không replay, while connected | Toast, navigation |
| ControlProperty<T> | Initial/current + UI changes, completed khi control dealloc | Main, latest behavior | TextField, Switch |
| ControlEvent<T> | UI events, không initial, completed khi control dealloc | Main | Button tap, selection |

Single success khác Observable next: success là terminal event của trait; asObservable() biểu diễn nó thành next rồi completed. Dispose không phát completed cho observer. Không ép mọi Observable thành Single chỉ vì muốn một kết quả: phải có finite contract.

## 2. Single – API request một kết quả

```swift
import Foundation
import RxSwift

struct Product: Codable, Equatable {
    let id: Int
    let name: String
}

enum ProductAPIError: Error {
    case invalidResponse
    case httpStatus(Int)
    case missingData
}

func fetchProduct(url: URL) -> Single<Product> {
    Single.create { observer in
        let task = URLSession.shared.dataTask(with: url) { data, response, error in
            if let error {
                observer(.failure(error))
                return
            }
            guard let http = response as? HTTPURLResponse else {
                observer(.failure(ProductAPIError.invalidResponse))
                return
            }
            guard (200..<300).contains(http.statusCode) else {
                observer(.failure(ProductAPIError.httpStatus(http.statusCode)))
                return
            }
            guard let data else {
                observer(.failure(ProductAPIError.missingData))
                return
            }
            do {
                observer(.success(try JSONDecoder().decode(Product.self, from: data)))
            } catch {
                observer(.failure(error))
            }
        }
        task.resume()
        return Disposables.create { task.cancel() }
    }
}
```

```swift
let bag = DisposeBag()
let url = URL(string: "https://example.com/products/1")!
fetchProduct(url: url)
    .observe(on: MainScheduler.instance)
    .subscribe(onSuccess: { product in
        print(product.name)
    }, onFailure: { error in
        print("Load failed:", error)
    })
    .disposed(by: bag)
```

Endpoint minh họa, không phải API thực. Mỗi subscription tạo URLSessionTask; dispose cancel underlying task. Không log response chứa PII/token. Single không tự share hoặc main; hai subscribe có thể tạo hai requests. HTTP transport success không đồng nghĩa 2xx/business success. Nếu server trả 204, chọn Completable hoặc response contract phù hợp thay decode Product.

### Single operators

```swift
protocol ProductRepository {
    func product(id: Int) -> Single<Product>
    func price(id: Int) -> Single<Decimal>
}

func productTitle(repo: any ProductRepository, id: Int) -> Single<String> {
    repo.product(id: id).map { $0.name }
}

func priceForProduct(repo: any ProductRepository, id: Int) -> Single<Decimal> {
    repo.product(id: id).flatMap { product in repo.price(id: product.id) }
}
```

map vẫn giữ Single vì mỗi success chuyển thành đúng một success. flatMap nối Single phụ thuộc (product ID trước, price sau). Single.zip cho nhiều request độc lập; nếu một lỗi, result lỗi. retry tạo subscription mới, chỉ retry transient/idempotent theo policy. Single.catch có thể fallback sang Single khác.

## 3. Maybe – Có hoặc không có kết quả

Local cache lookup có thể miss bình thường, không phải error.

```swift
import RxSwift

func cachedProduct(id: Int, cache: [Int: Product]) -> Maybe<Product> {
    Maybe.deferred {
        if let product = cache[id] { return .just(product) }
        return .empty()
    }
}

cachedProduct(id: 1, cache: [:])
    .subscribe(onSuccess: { print($0.name) },
               onError: { print("Read failed:", $0) },
               onCompleted: { print("Cache miss") })
    .disposed(by: bag)
```

Dictionary snapshot immutable trong ví dụ; real disk/DB lookup có thể throw. Maybe.empty() là complete không value; Maybe.error(error) là lỗi thực sự. Single<Product?> có một success kể cả nil; Maybe<Product> có thể không next. Hai contract khác nhau khi kết hợp operators.

### Cache-first với Maybe

```swift
func loadProduct(id: Int, cache: [Int: Product],
                 repo: any ProductRepository) -> Single<Product> {
    cachedProduct(id: id, cache: cache)
        .ifEmpty(switchTo: Single.deferred { repo.product(id: id) })
}
```

Chỉ gọi network khi cache miss; cached error không tự biến thành miss, hãy phân loại lỗi trước fallback. Cache-first khác stale-while-revalidate: cache hit ở ví dụ không refresh remote. ifEmpty(switchTo:) overload của Maybe nhận Single và trả Single; deferred giữ network lazy.

## 4. Completable – Chỉ quan tâm thành công hay lỗi

```swift
import Foundation
import RxSwift

func saveProduct(_ product: Product, to url: URL) -> Completable {
    Completable.deferred {
        do {
            try JSONEncoder().encode(product).write(to: url, options: .atomic)
            return .empty()
        } catch {
            return .error(error)
        }
    }
}

func saveThenReload(_ product: Product, url: URL,
                    repo: any ProductRepository) -> Single<Product> {
    let worker = ConcurrentDispatchQueueScheduler(qos: .utility)
    return saveProduct(product, to: url)
        .subscribe(on: worker)
        .andThen(Single.deferred { repo.product(id: product.id) })
}
```

File write synchronous, subscribe(on:) tránh block main; atomic file write không tự giải quyết conflict nhiều writers. Completable.andThen subscribe next sequence chỉ sau complete thành công; khi save error thì reload không chạy. Completable.empty() complete ngay, không có payload. Observable<Void>.just(()) có một next Void rồi complete, khác Completable.

**Ví dụ thực tế:** xóa favorite local -> reload list; update profile -> fetch latest profile; upload file nếu không cần response body. Với network delete validate status trước khi complete; HTTP timeout không có nghĩa server rollback.

## 5. Infallible – Stream không phát error

```swift
import RxSwift

let selectedIDs = Infallible<Int>.from([1, 2, 3])
let titles = selectedIDs.map { "Item \($0)" }
titles.subscribe(onNext: { print($0) }).disposed(by: bag)
```

Infallible giữ contract no-error nhưng không tự chuyển main, không tự share/replay. Một số operators nhận throwing closure sẽ có overload/return type khác; kiểm tra signature, không force-cast. Dùng cho stream pure value hoặc đã map lỗi thành typed state. Không thay Single ở API có thể fail.

```swift
enum CatalogState {
    case loading
    case content([Product])
    case failed
}

func safeState(_ source: Observable<[Product]>) -> Infallible<CatalogState> {
    source.map { CatalogState.content($0) }
        .startWith(.loading)
        .asInfallible(onErrorJustReturn: .failed)
}
```

Fallback phát failed rồi completed; no-error không đồng nghĩa stream sống mãi. Để search tiếp sau failure, xử lý error bên trong mỗi request và giữ outer trigger sống.

## 6. Conversion giữa các kiểu

| Conversion | Ý nghĩa / rủi ro |
| --- | --- |
| trait.asObservable() | Bỏ type-level contract cụ thể, dùng Observable operators |
| observable.asSingle() | Yêu cầu đúng một next rồi complete; empty hoặc >1 lỗi |
| observable.asMaybe() | 0 hoặc 1 value; >1 lỗi |
| observable.asCompletable() | Chỉ hợp với Observable<Never>; bỏ next bằng ignoreElements() trước nếu cần |
| observable.first() | Single<Element?>, nil nếu empty, emit đầu là đủ |
| observable.take(1) | Observable, 0 hoặc 1 next rồi complete |
| observable.single() | Observable, yêu cầu đúng một matching element; không phải Single trait |
| single.asMaybe() | Widen cardinality từ one sang 0/1 |
| single.asCompletable() | Bỏ success payload, giữ completion/error |
| observable.asDriver(onErrorJustReturn:) | Main + error fallback + replay/share |
| observable.asSignal(onErrorJustReturn:) | Main + error fallback + share không replay |
| observable.asInfallible(onErrorJustReturn:) | No-error fallback, không tự main/share |

```swift
Observable.of(1, 2).asSingle() // lỗi moreThanOneElement
Observable<Int>.empty().asSingle() // lỗi noElements
Observable.of(1, 2).first() // Single<Int?> success(1)
Observable<Int>.empty().first() // Single<Int?> success(nil)
Observable.of(1, 2).take(1).asSingle() // success(1)
Observable<Int>.empty().asMaybe() // completed
```

Không nhầm asSingle với take(1): asSingle cần biết stream có phát thêm hay không nên thường chờ completion nếu có một value; take(1) complete sau next đầu. Một hot stream có thể không complete, asSingle sẽ pending. Dùng timeout hoặc snapshot operator đúng contract. Không dùng asDriver(onErrorJustReturn: []) để che lỗi nghiệp vụ thành empty UI.

## 7. Driver vs Signal – UIKit ViewModel

Driver render state, Signal emit event một lần. Driver replay latest trong connection còn active, không phải persistent cache; Signal không replay nên subscriber đến muộn có thể bỏ lỡ event. Cả hai không tự bảo đảm event durable/exactly-once.

```swift
import UIKit
import RxSwift
import RxCocoa
import RxRelay

enum LoginUIState {
    case idle, loading, success, failure
}

protocol LoginUseCase {
    func execute(email: String) -> Single<Bool>
}

final class LoginViewModel {
    struct Input {
        let email: Driver<String>
        let tap: Signal<Void>
    }

    struct Output {
        let state: Driver<LoginUIState>
        let enabled: Driver<Bool>
        let route: Signal<Void>
    }

    private let useCase: any LoginUseCase
    init(useCase: any LoginUseCase) { self.useCase = useCase }

    func transform(_ input: Input) -> Output {
        let email = input.email.asObservable()
        let result = input.tap.asObservable()
            .withLatestFrom(email)
            .filter { $0.contains("@") }
            .flatMapFirst { [useCase] email -> Observable<LoginUIState> in
                useCase.execute(email: email).asObservable()
                    .map { $0 ? LoginUIState.success : .failure }
                    .catchAndReturn(.failure)
                    .startWith(.loading)
            }
            .share(replay: 0, scope: .whileConnected)

        let state = result.startWith(.idle)
            .asDriver(onErrorJustReturn: .failure)
        let enabled = Driver.combineLatest(input.email, state) { email, state in
            let loading: Bool
            if case .loading = state { loading = true } else { loading = false }
            return email.contains("@") && !loading
        }
        let route = result.compactMap { state -> Void? in
            if case .success = state { return () }
            return nil
        }.asSignal(onErrorSignalWith: .empty())
        return Output(state: state, enabled: enabled, route: route)
    }
}
```

```swift
// Inside UIViewController, on main, after controls are created.
let output = viewModel.transform(.init(
    email: emailField.rx.text.orEmpty.asDriver(),
    tap: loginButton.rx.tap.asSignal()
))
output.state.drive(onNext: { [weak self] in self?.render($0) }).disposed(by: bag)
output.enabled.drive(loginButton.rx.isEnabled).disposed(by: bag)
output.route.emit(onNext: { [weak self] in self?.showHome() }).disposed(by: bag)
```

Ví dụ contract minh họa email-only login; auth thật cần credential/challenge/token/session. Attach bindings một lần trước user input; result share replay 0 tránh navigation replay khi rebind. State Driver replay 1 cho UI trong active connection; route derive từ non-replay result để không navigate từ old state khi subscriber đến muộn. Nếu navigation phải survive view absence, dùng durable state/coordinator acknowledgment thay Signal đơn thuần. Runtime main scheduler khác Swift 6 compiler actor isolation, kiểm tra SDK annotations.

## 8. ControlProperty, ControlEvent và Binder

```swift
let text: ControlProperty<String?> = emailField.rx.text
let tap: ControlEvent<Void> = loginButton.rx.tap
let enabled: Binder<Bool> = loginButton.rx.isEnabled

text.orEmpty
    .map { $0.contains("@") }
    .bind(to: enabled)
    .disposed(by: bag)

tap.subscribe(onNext: { print("Tapped") }).disposed(by: bag)
```

ControlProperty biểu diễn UI property initial/current + user changes, có value sink để bind vào control. Programmatic assignment (textField.text = ...) không mặc định phát control event; nếu cần nguồn state chung, update owner rồi bind UI, không dựa event giả. ControlEvent không initial value, completed khi control dealloc. Binder là observer hướng binding, giữ target weak, mặc định main scheduler, không nhận error hợp lệ; không phải Observable/trait stream, không subscribe Binder.

### Custom Binder

```swift
extension Reactive where Base: UILabel {
    var errorMessage: Binder<String?> {
        Binder(base) { label, message in
            label.text = message
            label.isHidden = message == nil
        }
    }
}
```

Sử dụng observable state map sang optional message rồi bind(to: label.rx.errorMessage). Không tạo custom ControlProperty nếu không đảm bảo contract UI/main/no-error/dealloc; Binder đủ cho write-only rendering.

## 9. Subjects & Relays

| Kiểu | Giá trị subscriber mới | Terminal | Use case |
| --- | --- | --- | --- |
| PublishSubject | Chỉ next tương lai | error/completed | Event stream |
| BehaviorSubject | Initial/latest khi còn active | error/completed | State có terminal |
| ReplaySubject | N values theo buffer | error/completed | Replay bounded |
| AsyncSubject | Last value khi completed thành công; empty nếu không có | error/completed | Final result phụ thuộc completion |
| PublishRelay | Next tương lai | Không | UI action |
| BehaviorRelay | Initial/latest | Không | State owner |
| ReplayRelay | Buffer N values | Không | Replay event có chủ đích |

```swift
import RxRelay

let subject = PublishSubject<Int>()
subject.onNext(1) // chưa subscriber: mất event
subject.subscribe(onNext: { print("P", $0) }).disposed(by: bag)
subject.onNext(2) // P 2

let state = BehaviorRelay<Int>(value: 0)
state.accept(1)
state.subscribe(onNext: { print("B", $0) }).disposed(by: bag) // B 1
state.accept(2) // B 2

let history = ReplaySubject<Int>.create(bufferSize: 2)
history.onNext(1)
history.onNext(2)
history.onNext(3)
history.subscribe(onNext: { print("R", $0) }).disposed(by: bag) // R 2, R 3

let final = AsyncSubject<Int>()
final.subscribe(onNext: { print("A", $0) }).disposed(by: bag)
final.onNext(1)
final.onNext(2)
final.onCompleted() // A 2
```

BehaviorSubject terminal: late subscriber nhận terminal chứ không phải luôn latest. ReplaySubject có thể replay buffered values rồi terminal; AsyncSubject error không emit final value. Relays dùng accept, không onError/onCompleted; có thể dispose subscription riêng. Replay buffer giữ memory, không buffer unbounded event log. BehaviorRelay.value đọc sync nhưng value + accept là compound mutation không atomic; serialize writer bằng main/actor/queue. Không expose Relay mutable ra mọi layer: private owner, expose Observable/Driver readonly, input qua action method.

## 10. Q&A phỏng vấn

**Q: Single có chắc sẽ emit không?** Contract nếu terminate là success hoặc failure; never/pending/dispose vẫn có thể không có terminal cho observer. Timeout/cancellation policy cần riêng.

**Q: Single vs Maybe?** Single success có đúng một payload; Maybe cho phép empty completion. Cache miss là Maybe.empty, DB failure là error.

**Q: Completable vs Single<Void>?** Completable không next, completed/error; Single<Void> success chứa () hoặc failure.

**Q: Trait nào tự main thread?** RxCocoa Driver/Signal/ControlProperty/ControlEvent có main contract; Single/Maybe/Completable/Infallible không.

**Q: Infallible vs Driver?** Cùng không error, nhưng Infallible không mặc định main/share/replay, Driver có UI scheduler và replay while connected.

**Q: Driver replay có vĩnh viễn không?** Không, .whileConnected theo active subscriptions; persistent state/repository cache riêng.

**Q: Navigation dùng Driver hay Signal?** Signal cho transient one-shot event; Driver state replay dễ navigate lại. Durable route cần state/acknowledgment.

**Q: ControlProperty có phản ánh textField.text = value không?** Không mặc định, UI property source phát initial và user-triggered control events, programmatic mutation không nhất thiết emit.

**Q: Binder khác Observer?** Binder là observer chuyên UI binding, weak target, scheduler, không xử lý error như normal observer; không có source stream.

**Q: BehaviorSubject vs BehaviorRelay?** Cả hai latest, Subject terminal error/completed được, Relay không terminal; phù hợp long-lived UI state.

**Q: AsyncSubject vs Single?** AsyncSubject multicast giữ last next đến complete, Single contract one success/error nhưng không tự share; khác ownership/producer.

**Q: Vì sao asSingle() trên BehaviorRelay không trả ngay?** Relay không complete, asSingle phải đảm bảo không có element thứ hai; dùng first/take(1) nếu snapshot.

**Q: asDriver fallback có giữ search chạy sau lỗi?** Fallback ngoài outer pipeline có thể complete; catch inner mỗi request để query sau vẫn xử lý.

**Q: RxRelay có thread-safe cho counter value += 1?** Compound read-modify-write không atomic, serialize mutation.

**Bài tập:** Tạo repository Single fetch, Maybe cache, Completable save; build UI Driver state/Signal event; test late subscriber, error, cache miss, double tap, dispose và replay.

### Tham khảo

* [RxSwift Traits](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/Traits.md)
* [PrimitiveSequence](https://docs.rxswift.org/rxswift/traits/primitivesequence)
* [Infallible](https://docs.rxswift.org/rxswift/traits/infallible)
* [RxSwift Reference](https://docs.rxswift.org/)
* [Higher-order functions](higher-order-functions.md) · [Advanced operators](operators-advanced.md) · [RxSwift thực tế](practical-patterns.md)
