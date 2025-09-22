# Vấn đề Deadlock trong Kotlin WebFlux với Coroutine và Transaction

## Bối cảnh
Trong một dự án **Kotlin WebFlux**, mình gặp tình huống liên quan đến **coroutine cha - con** và **transaction** như sau:

- Có một hàm thực hiện **insert** một row mới vào table (chạy trong **coroutine cha**).
- Trong hàm đó, đồng thời có một **coroutine con** chạy song song, thực hiện **update** lại row vừa được insert ở coroutine cha.
- Mỗi coroutine lại chạy trong **transaction riêng biệt**.

## Vấn đề
- Coroutine cha giữ transaction `A` cho thao tác insert.
- Coroutine con mở transaction `B` cho thao tác update.
- Transaction `B` cần row do `A` insert, nhưng `A` chưa commit vì vẫn còn chạy.
- Trong khi đó, `A` chờ `B` hoàn tất một số điều kiện.
- Kết quả: **deadlock** xảy ra vì cả hai transaction chờ nhau.

## Nguyên nhân
- Do coroutine cha và coroutine con không chia sẻ cùng một transaction.
- Trong môi trường **R2DBC + WebFlux**, mỗi coroutine nếu không kiểm soát, có thể tạo transaction riêng, dẫn đến khóa chéo (deadlock).

## Giải pháp
- **Không tạo transaction riêng trong coroutine con.**
- Thay vào đó, coroutine con **kế thừa transaction của coroutine cha**, đảm bảo tất cả thao tác insert và update nằm trong **cùng một transaction**.
- Điều này giúp:
  - Tránh deadlock do chờ commit giữa các transaction.
  - Đảm bảo tính toàn vẹn dữ liệu (atomicity).

## Bài học rút ra
- Khi dùng **coroutine + transaction trong WebFlux**, cần quản lý transaction theo **structured concurrency** thay vì để mỗi coroutine tự tạo transaction riêng.
- Các coroutine con nên dùng lại transaction cha khi thao tác trên cùng dữ liệu.
- Nếu cần thực hiện song song, nên cân nhắc mức độ tách biệt dữ liệu và transaction.

---
