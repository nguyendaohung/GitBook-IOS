# Access Levels

### \`open\` và \`public\`

Cả 2 keywords trên đều cho phép chúng ta truy cập nó từ bên trong target nơi nó được define (define module) và cả bên ngoài - những module khác.

* **open**: Có thể truy cập từ bên ngoài, cho sử dụng ngoài _module_ hay _project._ Có thể subclass ngoài module; chỉ member khai báo open mới override ngoài module

```swift
// First.framework – A.swift

open class A {}
```

```swift
// Second.framework – C.swift

import First

internal class C: A {} // ok
```

* **public**:  Tương tự như _open_ nhưng việc kế thừa và _override_ lại bị giới hạn. Chỉ có các thành viên cùng module thì mới có thể kế thừa

```swift
// First.framework – B.swift

public class B: A {} // ok
```

```swift
// Second.framework – D.swift

import First

internal class D: B {} // error: B cannot be subclassed
```

### \`internal\` - default access level

Chỉ cho phép chúng ta truy cập nó từ bên trong target nơi nó đực define (define module). Là default level, nếu không khai báo access level => mặc định là internal

### \`fileprivate\`

Chỉ được truy cập trong cùng 1 file code.

```swift
// First.framework – A.swift

internal struct A {
    fileprivate static let x: Int = 1
}

A.x // ok
```

```swift
// First.framework – B.swift

A.x // error: x is not available
```

### \`private\`

Chỉ được truy cập trong cùng 1 class (bao gồm extensions trong cùng 1 file với class đó)

Extensions khác file cũng không truy cập được

```swift
// Declaring "A" class that has the two types of "private" and "fileprivate":
class A {
    private var aPrivate: String?
    fileprivate var aFileprivate: String?

    func accessMySelf() {
        // this works fine
        self.aPrivate = ""
        self.aFileprivate = ""
    }
}

// Declaring "B" for checking the abiltiy of accessing "A" class:
class B {
    func accessA() {
        // create an instance of "A" class
        let aObject = A()

        // Error! this is NOT accessable...
        aObject.aPrivate = "I CANNOT set a value for it!"

        // this works fine
        aObject.aFileprivate = "I CAN set a value for it!"
    }
}
```

### \`final\`

Ngoài ra, với **final** không phải mức truy cập, mà là modifier ngăn kế thừa/override

* Ngăn chặn việc _override_ các phương thức cũng như kế thừa _class_
* Có thể truy cập các phương thức và thuộc tính với khai báo (_public và open_). Tuy nhiên không thể kế thừa lại chúng
* Khuyến cáo nên sử dụng để tăng tốc độ biên dịch của Swift

## Kiến thức bổ sung và phỏng vấn

Access control hiện có open, public, package (Swift 5.9), internal, fileprivate, private. package cho các modules trong cùng Swift package; internal trong cùng module. Module thường là target build độc lập, không đồng nhất project. open chỉ cho class/class members, cho phép subclass/override từ module khác nếu member được open; public vẫn có thể sử dụng ngoài module nhưng không override ngoài module. final không phải access level, mà ngăn subclass/override.

```swift
public final class Counter {
    public private(set) var value = 0
    public init() {}
    public func increment() { value += 1 }
}
```

public class không tự làm initializer/members public; khai báo public init/method cần thiết. private(set) cho đọc theo access của property nhưng giới hạn ghi. private dùng trong declaration và extension cùng file của type; fileprivate rộng hơn cho mọi code trong file.

**Q: public và open khác gì?** public cho truy cập ngoài module; open thêm subclass/override phù hợp.

**Q: private vs fileprivate?** private giới hạn theo declaration/extension cùng file; fileprivate cho mọi code trong file.

**Q: package vs internal?** package có thể chia sẻ giữa modules cùng package, internal chỉ trong module.

**Q: final có luôn làm app nhanh hơn?** Không phải cam kết performance; compiler có thể devirtualize, mục đích chính là ngăn override/subclass.

**Q: Vì sao dùng private(set)?** Bảo vệ invariant, caller đọc được state nhưng chỉ owner mutate qua method.
