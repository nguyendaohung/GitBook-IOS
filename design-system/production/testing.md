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
