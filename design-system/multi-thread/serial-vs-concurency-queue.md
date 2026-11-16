# Serial vs Concurency Queue

* Serial Queue (tuần tự): Các task được thực thi lần lượt, kết thúc task này mới bắt đầu task khác

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-13 at 09.37.17.png" alt=""><figcaption></figcaption></figure>

* Concurrent Queue (đồng thời): Các task có thể được lấy ra và thực thi mà không cần đợi task trước đó kết thúc

<figure><img src="../.gitbook/assets/Screen Shot 2023-02-13 at 09.38.12.png" alt=""><figcaption></figcaption></figure>

## Queue ordering và synchronization

```swift
let serial = DispatchQueue(label: "serial")
serial.async { print("A") }
serial.async { print("B") } // A hoàn thành block trước B
let concurrent = DispatchQueue(label: "concurrent", attributes: .concurrent)
concurrent.async { print("A") }
concurrent.async { print("B") } // completion order không đảm bảo
```

Serial queue không đồng nghĩa một thread cố định. Concurrent queue lấy work theo thứ tự enqueue nhưng execution/completion có thể overlap. Nếu block chỉ start URLSession rồi return, serial queue không serialize lifetime request; cần async completion/TaskGroup/concatMap. Queue khác sync/async: sync chờ block hoàn thành, async enqueue rồi return. barrier trên custom concurrent queue chờ work trước và exclusive trong queue.

**Q: Serial queue có thể chạy background?** Có, custom serial queue không phải main.

**Q: concurrent queue + sync có concurrent với caller không?** Caller đợi block; work khác trong queue vẫn có thể concurrent.

**Q: Vì sao main.sync từ main deadlock?** Main đang chờ block được queue main chạy nhưng main không thể xử lý khi đang chờ.

**Q: Serial queue có tránh mọi race?** Chỉ nếu mọi truy cập state được cô lập qua queue; async work ngoài queue có thể race.
