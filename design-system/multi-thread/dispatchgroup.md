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

Đây là kết quả:

```none
ONE -> DONE
TWO -> DONE
THREE -> DONE
ALL DONE
```

1. Tạo ra một đối tượng DispatchGroup và cần có 1 queue để thực thi các task của DispatchGroup
2. Đưa 1 task vào group để thực thi, sử dụng `.enter().`&#x20;
3. Thực thi task đó
4. Thoát task ra khỏi DispatchGroup, sử dụng `.leave()`
5. Sau khi hoàn thành tất cả các task thì Group sẽ báo 1 notify về main queue (hoặc có thể bất kỳ queue nào). Cần chú ý về notify tới thread nào.

\=> Thường sử dụng để call nhiều API một lần => Ví dụ một màn cần lấy dữ liệu từ nhiều API để update UI
