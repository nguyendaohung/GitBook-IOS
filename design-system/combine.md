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
