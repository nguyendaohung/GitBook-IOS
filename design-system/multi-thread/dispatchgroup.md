# DispatchGroup

Đôi khi chúng ta có nhiều tác vụ trên nhiều luồng cần chờ chúng hoàn thành một lúc trước khi thực thi việc khác, khi đó chúng ta sử dụng DispatchGroup

Một ví dụ dễ thấy nhất là khi chúng ta lấy dữ liệu từ 2 API khác nhau về để update cho cùng một UI, khi đó chúng ta cần phải chờ data load xong từ cả 2 API trước khi update UI, trong trường hợp này sử dụng Dispatch Group là cực kì phù hợp.

```Swift
DispatchQueue.global(qos: .userInitiated).async {
            //1
            let dispatchGroup = DispatchGroup()
            
            //2
            dispatchGroup.enter()
            //3
            self.taskOne {
                print("ONE -> DONE")
                //4
                dispatchGroup.leave()
            }
            
            dispatchGroup.enter()
            self.taskTwo {
                print("TWO -> DONE")
                dispatchGroup.leave()
            }
            
            dispatchGroup.enter()
            self.taskThree {
                print("THREE -> DONE")
                dispatchGroup.leave()
            }
            
            //5
            dispatchGroup.notify(queue: .main) {
                print("ALL DONE")
            }
        }
```

Một kết quả có thể xảy ra (ONE/TWO/THREE không bảo đảm thứ tự):

```none
ONE -> DONE
TWO -> DONE
THREE -> DONE
ALL DONE
```

1. Tạo ra một đối tượng DispatchGroup ; group không tự schedule work
2. Đưa 1 task vào group để thực thi, sử dụng `.enter().`&#x20;
3. Thực thi task đó
4. Thoát task ra khỏi DispatchGroup, sử dụng `.leave()`
5. Sau khi hoàn thành tất cả các task thì Group sẽ báo 1 notify về main queue (hoặc có thể bất kỳ queue nào). Cần chú ý về notify tới thread nào.

\=> Thường sử dụng để call nhiều API một lần => Ví dụ một màn cần lấy dữ liệu từ nhiều API để update UI

## DispatchGroup: completion và error

DispatchGroup đếm work, không schedule task, không giữ result/error, không đảm bảo order hoàn thành. enter trước start async, leave đúng một lần mỗi enter, kể cả error/cancel. notify khi count về 0; wait block thread, tránh main. Nếu callback có thể không gọi, thiết kế timeout/lifetime, không leave nhiều lần.

```swift
let group = DispatchGroup()
for id in [1, 2, 3] {
    group.enter()
    DispatchQueue.global().async {
        defer { group.leave() }
        print("Work", id)
    }
}
group.notify(queue: .main) { print("All done") }
```

Work 1/2/3 completion order bất kỳ; All done cuối. Shared result dictionary cần lock/actor/serial queue, group không bảo vệ memory. Swift Concurrency async let/TaskGroup có result/error/cancellation và structured scope; dùng khi API async.

**Q: enter/leave lệch?** leave nhiều hơn enter có thể crash; thiếu leave notify không chạy.

**Q: notify vs wait?** notify async callback, wait block caller; ưu tiên notify cho UI.

**Q: DispatchGroup vs DispatchSemaphore?** Group chờ tập work, Semaphore giới hạn permit/đồng bộ blocking.

**Q: Group có cancel task?** Không, quản lý task handles riêng.
