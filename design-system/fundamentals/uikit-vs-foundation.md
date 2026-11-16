# UIKit vs Foundation

1. **UIKit** là một framework để xây dựng giao diện người dùng trên iOS và tvOS (macOS qua Mac Catalyst với UIKit). Nó cung cấp các lớp cho các thành phần giao diện người dùng chính như button, label, textfield, vv. Nó cũng cung cấp các lớp để quản lý và xử lý sự kiện, các animation, và các hiệu ứng giao diện người dùng.
2. **Foundation** là một framework cơ bản của iOS và tvOS (macOS qua Mac Catalyst với UIKit). Nó cung cấp các lớp cơ bản cho lập trình, tích hợp các kiểu Swift Standard Library như String, Array, Dictionary, các lớp để làm việc với dữ liệu và thời gian, các lớp để xử lý mạng và file, và rất nhiều lớp hỗ trợ khác.

Có thể nói rằng, UIKit là một framework để xây dựng giao diện người dùng và quản lý sự kiện trong iOS và tvOS (macOS qua Mac Catalyst với UIKit), trong khi Foundation cung cấp các kiểu dữ liệu cơ bản và các lớp hỗ trợ cho lập trình.

3. Trong iOS, một **Bundle** là một đối tượng dùng để quản lý tập hợp các tài nguyên liên quan đến một ứng dụng, chẳng hạn như các tập tin .storyboard, .xib, hình ảnh, âm thanh và các tập tin tài nguyên khác. Bundle cung cấp một cách tiện lợi để truy cập và sử dụng các tài nguyên trong ứng dụng của bạn.<br>

## Foundation, Swift Standard Library và UIKit

String, Array, Dictionary, Optional là Swift Standard Library, không phải types do Foundation định nghĩa; Foundation cung cấp Data, Date, URL, URLSession, JSONDecoder, FileManager, NotificationCenter, Bundle, NSError và bridging. UIKit cung cấp UIView/UIViewController, gestures, layout, rendering/event; macOS native dùng AppKit, UIKit trên Mac qua Mac Catalyst/iOS apps trên thiết bị hỗ trợ, không phải AppKit. SwiftUI dùng declarative UI và có thể interop UIKit.

```swift
import Foundation
struct UserDTO: Codable { let id: Int; let name: String }
func decode(_ data: Data) throws -> UserDTO {
    try JSONDecoder().decode(UserDTO.self, from: data)
}
```

Domain/network code không cần import UIKit; UI layer dùng UIKit. Bundle.main.url(forResource:withExtension:) tìm resource theo target, Bundle.module cho SPM resources; không hardcode absolute sandbox path.

**Q: Foundation vs standard library?** Foundation APIs hệ thống/data/network; standard library primitive/collection/protocol Swift.

**Q: UIKit có re-export Foundation không?** Thường import UIKit cho phép dùng Foundation types; import Foundation trực tiếp ở module không cần UI để thể hiện dependency.

**Q: Bundle.main vs Bundle.module?** main bundle app executable, module SPM resource bundle; framework có bundle riêng.

**Q: UIKit API gọi background được không?** UI creation/mutation theo MainActor/main-thread contract, Foundation không đồng nghĩa mọi API thread-safe.
