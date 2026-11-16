# Async/Await

**ĐỊNH NGHĨA:**&#x20;

Async/Await là một tính năng mới được giới thiệu trong Swift 5.5. Nó là một cách để thực hiện các tác vụ không đồng bộ một cách dễ dàng và thân thiện với người dùng.

Trong Swift, bạn có thể sử dụng từ khóa `async` để chỉ định rằng một hàm sẽ trả về một kết quả không đồng bộ. Sau đó, bạn có thể sử dụng từ khóa `await` để đợi cho kết quả trả về từ hàm đó trước khi tiếp tục thực thi code.

**Có thể sử dụng Async/Await** trong các tình huống mà các tác vụ cần phải thực hiện bất đồng bộ, nhưng vẫn cần phải đảm bảo rằng các tác vụ này hoàn thành trước khi tiếp tục thực hiện các tác vụ khác. Các tình huống này bao gồm:

1. **Thực hiện các yêu cầu mạng**: Khi thực hiện các yêu cầu mạng, việc phản hồi chậm và không đồng bộ là điều phổ biến. Sử dụng Async/Await giúp đảm bảo rằng các yêu cầu mạng hoàn thành trước khi xử lý các phản hồi.

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 11.13.30.png" alt=""><figcaption></figcaption></figure>

Trong ví dụ này, chúng ta sử dụng hàm `URLSession.shared.data(from:completionHandler:)` để thực hiện yêu cầu mạng bất đồng bộ và sử dụng từ khóa `await` để đợi cho yêu cầu hoàn thành trước khi trả về kết quả.

2. **Xử lý các tác vụ nặng bất đồng bộ**: Khi thực hiện các tác vụ nặng như đọc và ghi tệp hoặc xử lý hình ảnh, việc thực hiện chúng trên luồng chính sẽ gây tắc nghẽn và làm giảm hiệu suất. Sử dụng Async/Await giúp đảm bảo rằng các tác vụ nặng được thực hiện trên luồng nền và không làm giảm hiệu suất của ứng dụng

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 11.15.21.png" alt=""><figcaption></figcaption></figure>

Trong ví dụ này, chúng ta sử dụng hàm `fetchData(from:)` để tải dữ liệu từ một URL và sử dụng từ khóa `await` để đợi cho tải xuống hoàn thành trước khi tiếp tục thực hiện các tác vụ khác. Sau đó, chúng ta sử dụng hàm `processImage(_:)` để xử lý ảnh một cách bất đồng bộ và đợi cho xử lý hoàn thành trước khi trả về kết quả.

3. **Xử lý các tác vụ đa luồng**: Khi bạn cần thực hiện các tác vụ đa luồng, việc quản lý và đồng bộ hóa chúng có thể là một thách thức. Sử dụng Async/Await giúp đơn giản hóa quản lý các tác vụ đa luồng bằng cách đợi các tác vụ hoàn thành trước khi tiếp tục thực hiện các tác vụ khác.

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 11.17.15.png" alt=""><figcaption></figcaption></figure>

Trong ví dụ này, chúng ta sử dụng các hàm `fetchDataTask1()`, `fetchDataTask2()` và `fetchDataTask3()` để thực hiện các tác vụ bất đồng bộ song song. Sau đó, chúng ta sử dụng từ khóa `await` để đợi cho các tác vụ hoàn thành trước khi trả về kết quả dưới dạng một mảng các chuỗi.

4. **Xử lý các tác vụ theo thứ tự đúng:**

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-22 at 11.18.26.png" alt=""><figcaption></figcaption></figure>

Trong ví dụ này, chúng ta sử dụng các hàm `fetchDataTask1()`, `fetchDataTask2()` và `fetchDataTask3()` để thực hiện các tác vụ bất đồng bộ theo thứ tự. Chúng ta sử dụng từ khóa `await` để đợi cho mỗi tác vụ hoàn thành trước khi tiếp tục thực hiện tác vụ tiếp theo. Cuối cùng, chúng ta sử dụng hàm `processResults(_:_:_:)` để xử lý kết quả trả về từ các tác vụ.
