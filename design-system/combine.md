# 📨 Combine

### Định nghĩa

Combine là reactive framework của Apple (iOS 13+). Publisher phát Output/Failure; Subscriber nhận subscription/value/completion và thể hiện demand; Cancellable hủy luồng. AnyCancellable tự cancel khi deinit.

* PassthroughSubject không lưu giá trị trước đó.
* CurrentValueSubject lưu giá trị hiện tại.
* @Published phát thay đổi property; publisher phát trong willSet nên đọc value từ closure thay vì đọc property cũ.
* map/filter/debounce/removeDuplicates/flatMap/switchToLatest; receive(on:) đổi scheduler downstream, subscribe(on:) tác động upstream subscribe/request/cancel.
* Failure là typed Error; Never nếu không lỗi. catch/replaceError xử lý lỗi; eraseToAnyPublisher che implementation.

```swift
import Combine
import Foundation

final class SearchModel: ObservableObject {
    @Published var query = ""
    @Published private(set) var results: [String] = []
    private var cancellables = Set<AnyCancellable>()

    init(search: @escaping (String) -> AnyPublisher<[String], Error>) {
        $query
            .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
            .removeDuplicates()
            .map { text -> AnyPublisher<[String], Never> in
                guard !text.isEmpty else {
                    return Just([]).eraseToAnyPublisher()
                }
                return search(text).replaceError(with: []).eraseToAnyPublisher()
            }
            .switchToLatest()
            .receive(on: RunLoop.main)
            .sink { [weak self] in self?.results = $0 }
            .store(in: &cancellables)
    }
}
```

### Q&A

**Q: RxSwift vs Combine?** RxSwift hỗ trợ nhiều deployment target, operator/traits RxCocoa; Combine native, typed failure và demand. Chọn theo codebase và OS support.

**Q: async/await vs Combine?** Async phù hợp một lần request; reactive phù hợp nhiều event liên tục, composition. AsyncSequence có thể biểu diễn stream.

**Q: Vì sao không giữ AnyCancellable?** Subscription bị cancel sớm; nếu giữ trong self và closure capture self mạnh có thể tạo cycle.

Combine là kiến thức bổ sung, CV ghi RxSwift tại Map4D.

### Tham khảo

[Combine - Apple](https://developer.apple.com/documentation/combine)

[Combine trong Swift](https://viblo.asia/p/combine-trong-swift-WAyK86nmlxX)

## Combine chuyên sâu

### Pipeline và demand

```text
Publisher -> Subscription -> Subscriber
Subscriber request demand -> Publisher emit values -> completion/cancel
```

sink thường yêu cầu unlimited demand; custom Subscriber có thể request .max(n) và trả additional demand từ receive. Demand không tự giới hạn memory nếu operator/source buffer; chọn strategy phù hợp. Just phát một value, Future phát một kết quả (Future bắt đầu work khi khởi tạo), Deferred tạo publisher lúc subscribe.

```swift
import Combine
import Foundation

func fetchNames(url: URL) -> AnyPublisher<[String], Error> {
    URLSession.shared.dataTaskPublisher(for: url)
        .tryMap { data, response in
            guard let http = response as? HTTPURLResponse,
                  (200..<300).contains(http.statusCode) else {
                throw URLError(.badServerResponse)
            }
            return data
        }
        .decode(type: [String].self, decoder: JSONDecoder())
        .eraseToAnyPublisher()
}
```

dataTaskPublisher là publisher của một request; nhiều subscription có thể tạo nhiều request. share() chia sẻ upstream giữa subscriber đang kết nối, không replay giá trị cũ. .retry(n) retry failure không phân biệt status; không dùng mù quáng cho payment.

### Operator và RxSwift đối chiếu

| RxSwift | Combine | Mục đích |
| --- | --- | --- |
| Observable | AnyPublisher | Stream |
| PublishSubject | PassthroughSubject | Event không lưu |
| BehaviorSubject | CurrentValueSubject | Latest value |
| flatMapLatest | map + switchToLatest | Search latest |
| observe(on:) | receive(on:) | Scheduler downstream |
| DisposeBag | Set<AnyCancellable> | Lifetime |
| combineLatest | combineLatest | Form state |

```swift
let query = PassthroughSubject<String, Never>()
let results = query
    .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
    .removeDuplicates()
    .map { term -> AnyPublisher<[String], Never> in
        guard var components = URLComponents(string: "https://example.com/search")
        else { return Just([]).eraseToAnyPublisher() }
        components.queryItems = [URLQueryItem(name: "q", value: term)]
        guard let url = components.url else {
            return Just([]).eraseToAnyPublisher()
        }
        return fetchNames(url: url)
            .replaceError(with: [])
            .eraseToAnyPublisher()
    }
    .switchToLatest()
    .receive(on: RunLoop.main)
```

Giữ cancellable từ sink; switchToLatest cancel inner cũ. Production dùng state enum và handle error ở inner thay vì biến mọi lỗi thành empty. receive(on:) scheduler main không thay compiler actor isolation; khi gọi @MainActor API từ closure, thiết kế bridge phù hợp Swift concurrency.

### Lỗi thường gặp

* Future eager: dùng Deferred { Future { ... } } nếu cần mỗi subscription tạo work.
* assign(to:on:) có thể retain object; self giữ cancellable gây cycle. sink weak hoặc ownership khác.
* @Published phát willSet: closure nhận newValue, đọc object.property trong callback có thể là oldValue.
* Sử dụng combineLatest cho submit gây call mỗi lần input đổi; dùng button event với latest state.

**Q: Combine vs async/await?** Combine compose nhiều event, typed failure/demand; async/await biểu diễn flow tuần tự hoặc structured concurrency. AsyncSequence phù hợp stream, publisher.values là bridge nhưng cần hiểu buffering/lifetime.

**Bài tập:** Tạo search publisher và validation form, test nhiều subscriber, cancellation và error recovery.

## Câu hỏi phỏng vấn tình huống

**Q: Future vs Deferred?** Future thực hiện closure lúc tạo, lưu kết quả; Deferred tạo publisher khi subscribe. Dùng deferred cho cold retry work.

**Q: switchToLatest vs flatMap?** Switch hủy inner cũ, latest only; flatMap merge inner, có maxPublishers để giới hạn active subscriptions.

**Q: Combine share() replay latest không?** Không, share live upstream; CurrentValueSubject giữ latest, thiết kế cache/state owner riêng.

**Q: receive(on:) có thay MainActor?** Runtime scheduler delivery khác compiler actor isolation; UI mutation phải tuân actor contract.
