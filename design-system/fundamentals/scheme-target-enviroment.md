# Scheme, Target, Enviroment

**Trong iOS development, "Scheme," "Target," và "Environment" là các khái niệm quan trọng liên quan đến việc xây dựng ứng dụng. Dưới đây là mô tả về mỗi khái niệm này:**

1. **Target (Mục tiêu):**
   * Target là một phần của dự án (project) trong Xcode. Một dự án có thể chứa nhiều target khác nhau. Mỗi target thường tượng trưng cho một ứng dụng, extension, hoặc framework riêng biệt mà bạn muốn xây dựng.
   * Ví dụ, một ứng dụng iOS có thể có nhiều target cho phiên bản cơ bản và phiên bản Pro của ứng dụng, mỗi target có cài đặt và tùy chỉnh riêng biệt.
   * Mỗi target có một cấu hình (build settings) riêng, cho phép bạn cấu hình các thông số như thư viện sử dụng, biến tiền trình (preprocessor macros), các đối tượng và các tệp tham chiếu.
2. **Scheme (Kế hoạch):**
   * Scheme là cách bạn thiết lập cách bạn muốn xây dựng và chạy ứng dụng trong Xcode. Một dự án có thể có nhiều Scheme.
   * Một Scheme định nghĩa cách bạn muốn xây dựng ứng dụng (chẳng hạn là chế độ Debug hoặc Release), thiết bị hoặc mô phỏng mà bạn muốn sử dụng, target mà bạn muốn xây dựng, và nhiều cài đặt khác.
   * Khi bạn chọn một Scheme cụ thể và bấm nút "Run" hoặc "Build" trong Xcode, nó sẽ thực hiện xây dựng và chạy ứng dụng dựa trên cài đặt của Scheme đó.
3. **Environment (Môi trường):**
   * Môi trường đề cập đến các biến môi trường và cài đặt cụ thể của ứng dụng khi nó đang chạy. Trong iOS development, bạn thường thay đổi môi trường để kiểm tra và xử lý các tình huống khác nhau như "Development," "Staging," và "Production."
   * Môi trường bao gồm các thông số như URLs của máy chủ, API keys, môi trường kết nối và cài đặt khác mà bạn muốn cấu hình khác nhau cho các phiên bản khác nhau của ứng dụng (thường là để phục vụ mục đích kiểm tra và triển khai).

Khi phát triển ứng dụng iOS, bạn sẽ cần làm việc với các Target để định rõ các thành phần của dự án, Scheme để thiết lập quy trình xây dựng và chạy, và Environment để quản lý các cài đặt môi trường cho ứng dụng của bạn. Điều này giúp bạn kiểm soát cách ứng dụng của bạn hoạt động trong môi trường phát triển, kiểm tra và triển khai.

**Hãy xem xét một ví dụ cụ thể về cách bạn có thể sử dụng "Target," "Scheme," và "Environment" trong iOS development:**

1.  **Target:**

    Giả sử bạn đang phát triển một ứng dụng tên là "MyApp." Ứng dụng này có phiên bản cơ bản và phiên bản Pro với các tính năng bổ sung. Bạn có thể tạo hai target riêng biệt cho các phiên bản này trong cùng một dự án. Một target là "MyApp" và một target khác là "MyAppPro." Mỗi target có thể chia sẻ mã nguồn chung nhưng có cài đặt riêng biệt cho phiên bản cơ bản và phiên bản Pro.
2.  **Scheme:**

    Bây giờ, bạn muốn xác định cách xây dựng và chạy ứng dụng của mình trong các tình huống khác nhau. Bạn có thể tạo các Scheme riêng biệt để xác định các cài đặt khác nhau:

    * `MyApp Debug`: Scheme này được sử dụng để xây dựng và chạy ứng dụng trong chế độ Debug.
    * `MyApp Release`: Scheme này được sử dụng để xây dựng và chạy ứng dụng trong chế độ Release để triển khai ứng dụng lên App Store.
    * `MyAppPro Debug`: Scheme này được sử dụng để xây dựng và chạy phiên bản Pro của ứng dụng trong chế độ Debug để kiểm tra tính năng của phiên bản Pro.
    * `MyAppPro Release`: Scheme này được sử dụng để xây dựng và chạy phiên bản Pro của ứng dụng trong chế độ Release.

    Mỗi Scheme đều thiết lập cụ thể cho target tương ứng (ví dụ: `MyApp Debug` sẽ chạy target "MyApp" trong chế độ Debug).
3.  **Environment:**

    Bạn có ba môi trường chính trong ứng dụng của mình: "Development," "Staging," và "Production." Mỗi môi trường có một URL máy chủ khác nhau và các cài đặt khác nhau, ví dụ:

    * Môi trường "Development" sử dụng URL máy chủ phát triển và chứa các thông tin giả mạo để phục vụ mục đích phát triển và kiểm tra.
    * Môi trường "Staging" sử dụng URL máy chủ dựng để kiểm tra cuối cùng trước khi triển khai lên môi trường Production.
    * Môi trường "Production" sử dụng URL máy chủ thực tế và cài đặt cho ứng dụng hoạt động trong môi trường thực tế.

    Bằng cách sử dụng các cài đặt môi trường khác nhau cho các Scheme, bạn có thể chuyển đổi giữa các môi trường một cách dễ dàng và quản lý cài đặt cho ứng dụng của bạn.

## Build configuration và Q&A

Target định nghĩa sản phẩm/sources/resources/signing. Scheme chọn Build/Run/Test/Profile/Analyze/Archive action, target và configuration. Destination chọn device/simulator. Development/Staging/Production thường map vào configurations + xcconfig; launch arguments/environment variables khi Run không phải secret store, không tự tồn tại trong App Store build.

```text
// Staging.xcconfig
PRODUCT_BUNDLE_IDENTIFIER = com.example.app.staging
API_HOST = staging.example.com
SWIFT_ACTIVE_COMPILATION_CONDITIONS = $(inherited) STAGING
```

Info.plist có thể expand build settings. Không nhúng API secret/token vào binary. Extension/watch target có bundle ID/entitlements/signing riêng; target membership đúng. Shared scheme để CI dùng, version Xcode pinned.

**Q: Scheme vs target?** Target sản phẩm cần build; scheme điều phối actions/target/configuration.

**Q: Debug/Release vs staging/prod?** Debug/Release là build configuration; staging/prod là environment nghiệp vụ, có thể tạo StagingDebug/StagingRelease hoặc xcconfig phù hợp.

**Q: Tại sao CI không thấy scheme?** Scheme chưa shared/commit hoặc sai workspace/project; kiểm tra xcodebuild -list.

**Q: API key trong xcconfig có an toàn?** Không nếu nhúng binary; secret backend/CI, client dùng public config và short-lived credential.
