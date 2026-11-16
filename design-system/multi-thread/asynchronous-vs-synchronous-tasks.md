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
