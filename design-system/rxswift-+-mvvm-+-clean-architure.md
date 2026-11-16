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
