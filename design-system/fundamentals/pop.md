# POP

Link tham khảo :  [https://viblo.asia/p/gioi-thieu-protocol-oriented-programming-trong-swift-phan-1-naQZR1vqKvx](https://viblo.asia/p/gioi-thieu-protocol-oriented-programming-trong-swift-phan-1-naQZR1vqKvx)

Protocol-Oriented Programming (POP) là một phong cách lập trình trong Swift mà nổi bật với việc ưu tiên sử dụng protocol để thể hiện các tính năng và chức năng của một đối tượng. Trong POP, chúng ta sử dụng protocol để xác định các chức năng của một đối tượng và việc sử dụng protocol như một "template" để tạo ra các đối tượng cụ thể.

Bời vì POP được kế thừa từ OOP nên POP có rất nhiều ưu diểm vượt trội hơn OOP. Ngày nay, trong Swift, Mình thấy rất nhiều người ưa thích sử dụng Struct kết hợp Protocol (Thường được sử dụng trong POP) thay vì sử dụng class (OOP). Mọi thứ Class có thể làm, Struct cũng có thể. Struct dựa trên Value Type thay vì Reference Type như Class -> Chúng ta không cần phải lo nghĩ nhiều tới việc Memory Leak, deadlock (multithread), implicit sharing data...

### Ví dụ về POP <a href="#id-2-vi-du-ve-pop-1" id="id-2-vi-du-ve-pop-1"></a>

Bây giờ chúng ta sẽ đi tới ví dụ cụ thể

Giả sử chúng ta có class là Bird, Bird có 2 thuộc tính: name và feather.

```none

class Bird {
    
    var name:String
    var feathers:String
    
    func fly(){
        //TODO: nothing
    }
    
    init(name:String, feathers:String){
        self.name = name
        self.feathers = feathers
    }
    
}
```

Tiếp theo, Chúng ta có 2 subclass của Bird, là Parrot và Eagle.

```none
class Parrot:Bird {
    
    override init(name: String, feathers: String) {
        super.init(name: name, feathers: feathers)
    }
    
    override func fly() {
        //TODO: nothing
    }
}

class Eagle:Bird{
    
    override init(name: String, feathers: String) {
        super.init(name: name, feathers: feathers)
    }

    override func fly() {
        //TODO: nothing
    }
}
```

Và giờ mình muốn theo Penguin. Oops...

```none
class Penguin:Bird { 
    
    //Oops, i do not have feather
    override init(name: String, feathers: String) {
        super.init(name: name, feathers: feathers)
    }

    // i'm too fatty, how can i fly :(
    override func fly() {
    }
}
```

Penguin là chim, nhưng nó không hề có lông và không thể bay&#x20;

Vậy giờ phải sửa thế nào? Trong OOP chúng ta sẽ làm cách sau:

Chia tách ra thành 2 class khác nhau : Bird và Penguin Class → Oops, ý tưởng tồi. Tạo 1 protocol Flyable → Chúng ta sẽ phải viết lại khá nhiều code Đổi thành static func fly() → Sẽ thế nào nếu ta có 10 - 100 subclass???

#### 3. Welcome to POP: <a href="#id-3-welcome-to-pop-2" id="id-3-welcome-to-pop-2"></a>

Chúng ta có thể thiết kế lại base code, thay đổi từ Class sang Protocol và Struct để giải quyết vấn đề này.

```none
protocol Bird {
    var name:String? {get set}
}

protocol Flyable {
    func fly()
}
extension Flyable {
    func fly() {
        print("I believe i can fly")
    }
}

protocol Featherable {
    var feather:String?{get set}
}
extension Featherable {
    var feather:String? {
        get {
            return "I have feather"
        }
        set{
            
        }
    }
}
```

Với Extension, chúng ta không cần viết lại quá nhiều code, chỉ cần sửa hoặc trong một vài hành vi đặc biệt của việc bay

```none
struct Parrot:Bird,Flyable,Featherable{
    var name: String?
    
    init(name: String){
        self.name = name
    }
}

struct Eagle:Bird,Flyable,Featherable {
    var name: String?
    
    init(name:String){
        self.name = name
    }
}

struct Penguin:Bird{
    var name: String?
    
    init(name:String){
        self.name = name
    }
}
```

Như bạn có thể thấy, Penguin không cần phải implement Flyable và Featherable và không phá vỡ thiết kế và cấu trúc.

Bây giờ bạn có thể thêm loài mới như Ostrich(Không thể bay nhưng có lông) ) ![😄](https://twemoji.maxcdn.com/2/72x72/1f604.png)

## 4. Kết luận <a href="#id-4-ket-luan-3" id="id-4-ket-luan-3"></a>

Với ví dụ này, mình hi vọng bạn sẽ có cái nhìn tốt về Protocol-Oriented Programming. Hiển nhiên, POP chỉ là khái niệm, bạn có thể áp dụng nếu bạn muôn hoặc không nhất thiết phải dùng, tuỳ thuộc vào quyết định và vấn đề của bạn.
