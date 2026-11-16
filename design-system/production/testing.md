# Unit Test, UI Test và Swift Testing

### Định nghĩa

Unit test kiểm tra logic nhỏ độc lập; integration test kiểm tra boundary/service/database; UI test kiểm tra user flow qua XCUITest. XCTest hỗ trợ unit/UI/performance; Swift Testing (Xcode 16+, Swift 6 toolchain) cung cấp @Test, #expect, parameterized test, không thay thế XCUITest.

### Ví dụ XCTest

```swift
import XCTest

final class SpeedTests: XCTestCase {
    func testMegabitsPerSecond() {
        let bytes: Double = 10_000_000
        let seconds: Double = 2
        XCTAssertEqual(bytes * 8 / seconds / 1_000_000, 40,
                       accuracy: 0.001)
    }
}
```

Production nên gọi hàm thật từ app module bằng @testable import; ví dụ trên minh họa assertion. Arrange–Act–Assert; test boundary, error, cancellation, timeout và state transition.

```swift
import Testing

@Test(arguments: [0, 1, 100])
func nonNegativeCount(count: Int) {
    #expect(count >= 0)
}
```

### Testability

* DI inject protocol; stub trả kết quả, mock verify interaction, fake có implementation đơn giản. Test behavior thay vì private method.
* Async XCTest dùng async throws, expectation cho callback, tránh sleep cố định; fake clock/scheduler để test debounce và retry.
* URLProtocol stub HTTP, fixture JSON success/malformed/401/500; không phụ thuộc server thật trong unit test.
* Core Data in-memory store cho logic, SQLite temp store cho migration; StoreKit configuration/sandbox cho purchase.
* UI test launch arguments, accessibilityIdentifier ổn định, waitForExistence, mock backend nếu phù hợp. Test purchase/permissions/extension/VPN/watch trên device thật; simulator không đủ.
* CI chạy test, coverage là tín hiệu chứ không phải chất lượng; ưu tiên critical flow và regression.

### Q&A

**Q: Mock vs Stub?** Stub cung cấp data; mock kiểm tra interaction; fake là implementation đơn giản.

**Q: Test API không gọi mạng thật?** Inject client/repository hoặc URLProtocol, fixture và assert error mapping.

**Q: Test race condition?** Controlled scheduler/clock, nhiều task, assertion invariant; chạy Thread Sanitizer khi phù hợp.

**Liên hệ:** Map4D search, FastVPN speed math/state, StoreKit entitlement, Call ID spam normalization, Watch Faces sync dedup.

## Testing chuyên sâu: từ unit test đến integration và UI test

### Test Pyramid và phạm vi

Unit test kiểm tra một đơn vị logic nhanh, isolated; integration test kiểm tra phối hợp nhiều thành phần thật (DB/API client với stub transport); UI test chạy app và thao tác qua accessibility. Không dùng UI test cho mọi edge case vì chậm và flaky. Test behavior, không test private implementation. Coverage là tín hiệu thiếu test, không bảo đảm test chất lượng.

### Fake Repository và async XCTest

```swift
import XCTest

struct Product: Equatable, Sendable {
    let id: Int
    let name: String
}

protocol ProductRepository: Sendable {
    func fetch() async throws -> [Product]
}

struct FakeRepository: ProductRepository {
    let result: Result<[Product], TestError>
    func fetch() async throws -> [Product] { try result.get() }
}

enum TestError: Error { case offline }

@MainActor
final class ProductViewModel {
    enum State: Equatable {
        case idle, loading, loaded([Product]), failed
    }
    private(set) var state: State = .idle
    private let repository: any ProductRepository

    init(repository: any ProductRepository) {
        self.repository = repository
    }

    func load() async {
        state = .loading
        do {
            let items = try await repository.fetch()
            try Task.checkCancellation()
            state = .loaded(items)
        } catch is CancellationError {
            state = .idle
        } catch {
            state = .failed
        }
    }
}

final class ProductViewModelTests: XCTestCase {
    @MainActor
    func testLoadSuccess() async {
        let products = [Product(id: 1, name: "Coffee")]
        let sut = ProductViewModel(repository: FakeRepository(
            result: .success(products)
        ))
        await sut.load()
        XCTAssertEqual(sut.state, .loaded(products))
    }

    @MainActor
    func testLoadFailure() async {
        let sut = ProductViewModel(repository: FakeRepository(
            result: .failure(.offline)
        ))
        await sut.load()
        XCTAssertEqual(sut.state, .failed)
    }
}
```

Ví dụ standalone để học; production đưa Product/ViewModel vào app module, test target dùng @testable import. Arrange/Act/Assert rõ ràng. Fake trả data/error deterministic; mock verify interaction (ví dụ repository gọi đúng một lần); spy ghi lại calls. Stub cung cấp response cố định. Không mock mọi type: domain logic thuần test trực tiếp.

### XCTest vs Swift Testing

| Tiêu chí | XCTest | Swift Testing |
| --- | --- | --- |
| Khai báo | XCTestCase/test method | @Suite, @Test |
| Assert | XCTAssertEqual | #expect |
| Async | async test/expectation | async test, confirmation |
| Parameterized | Loop/helper | @Test(arguments:) |
| UI test | XCUITest | Tiếp tục dùng XCUITest |

```swift
import Testing

@Test(arguments: [0, 1, 10])
func speedIsNonNegative(bytes: Int) {
    let mbps = Double(bytes) * 8 / 1_000_000
    #expect(mbps >= 0)
}
```

Swift Testing từ Xcode 16/Swift 6 toolchain; kiểm tra target support. Không dùng sleep ngẫu nhiên để chờ event. Inject clock/scheduler; XCTest expectation.fulfillment(of:timeout:) cho callback async. Test cancellation bằng task và controllable fake, assert state cuối.

### Integration test API và persistence

URLProtocol subclass có thể intercept URLSessionConfiguration.ephemeral.protocolClasses để trả HTTPURLResponse/data/error fixture. Test 200/204/401/429/500, malformed JSON, retry cap, cancellation. Không global stub mutable không đồng bộ khi test parallel; dùng per-test session/handler có isolation. Core Data dùng in-memory store cho CRUD, temporary SQLite để test migration thật. StoreKit Configuration/StoreKitTest cho purchase/renewal; simulator không thay device test extension/VPN/watch.

### XCUITest

```swift
import XCTest

final class LoginUITests: XCTestCase {
    func testLoginForm() {
        let app = XCUIApplication()
        app.launchArguments = ["-uiTesting", "-useFakeBackend"]
        app.launch()
        let email = app.textFields["login.email"]
        XCTAssertTrue(email.waitForExistence(timeout: 5))
        email.tap()
        email.typeText("test@example.com")
        XCTAssertTrue(app.buttons["login.submit"].exists)
    }
}
```

App test build phải hỗ trợ launch arguments và accessibilityIdentifier; fake backend chỉ test configuration, không vô hiệu security production. waitForExistence thay sleep, test accessibility IDs thay text localization. Reset state giữa tests. Test matrix iOS/device/language/permission/offline.

**Q: Test async API không chờ mạng thật?** Inject repository/transport; URLProtocol fixture, fake clock.

**Q: Unit test vs UI test?** Unit nhanh và isolated, UI test end-to-end user flow; bổ sung integration ở boundary.

**Bài tập:** Viết test success/empty/failure/cancel cho ViewModel, test pagination duplicate và 401 refresh single-flight.

## Câu hỏi phỏng vấn tình huống

**Q: Test async cancellation không sleep?** Inject controllable fake/clock, signal start, cancel task và resolve fake theo contract, assert state và stale result; RxTest virtual time cho reactive.

**Q: Test migration SwiftData/Core Data dùng in-memory đủ?** Không, disk store schema cũ mở schema mới, verify data/relations; in-memory tốt cho CRUD logic.

**Q: Coverage 100% có đủ?** Không, branch/edge cases/assertion quality và critical flow quan trọng; test behavior/error/cancellation, integration/device.

**Q: UI test flaky xử lý?** Stable accessibilityIdentifier, deterministic backend/fixtures, waitForExistence/conditions, reset state; không tăng sleep tùy ý.
