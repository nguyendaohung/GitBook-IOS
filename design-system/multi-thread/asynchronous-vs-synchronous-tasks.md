# Asynchronous vs synchronous tasks

**Synchronous**: sẽ trả về chỉ sau khi task được hoàn thành. Có nghĩa là khả năng block thread hiện tại nếu sử dụng để thực thi một task tốn nhiều thời gian (request APIs, download media content,…)

**Asynchronous**: sẽ trả về ngay lập tức mà không cần phải chờ. Dễ hiểu hơn là nó sẽ add task vào queue vào tiếp tục thực hiện các công việc khác mà không gây block

#### Ví dụ: <a href="#vi-du-2" id="vi-du-2"></a>

```none
print("Exe 1") 
synchronous {
   wait(10)
   print("Exe 2")
}
print("Exe 3")
```

Với đoạn code trên thì output sẽ là:

```none
Exe 1
Exe 2
Exe 3
```

Trường hợp asynchronous

```none
print("Exe 1") 
asynchronous {
   wait(10)
   print("Exe 2")
}
print("Exe 3")
```

Output sẽ là:

```none
Exe 1
Exe 3
Exe 2
```

## Sync/async thực tế

```swift
let queue = DispatchQueue(label: "work")
print("1")
queue.sync { print("2") }
print("3") // 1,2,3

print("A")
queue.async { print("B") }
print("C") // A trước; B/C có thể theo bất kỳ thứ tự tương đối
```

async enqueue không bảo đảm B sau C; ví dụ cũ có wait nên thường C trước B, không phải contract. sync block caller đến khi work hoàn thành, không đồng nghĩa main thread. async/await suspend task khi API hỗ trợ, khác GCD async fire-and-forget; await không tự chạy CPU-heavy background.

**Q: async vs concurrent?** Async liên quan caller có đợi hay không; concurrent liên quan nhiều work overlap. Serial async vẫn serial.

**Q: async/await có block thread?** Await có thể suspend; synchronous work bên trong async vẫn block executor/thread.

**Q: Làm sao đợi hai request?** async let/TaskGroup, DispatchGroup notify cho callback; không semaphore.wait trên main.

**Q: Khi nào sync hợp lý?** Work ngắn cần kết quả ngay, hoặc synchronous access qua isolation queue không reentrant; tránh UI/I/O dài.
