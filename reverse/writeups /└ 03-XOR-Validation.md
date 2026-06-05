## Analysis

Chạy chương trình, ta thấy chương trình yêu cầu nhập một chuỗi và trả về `Correct` hoặc `Wrong`.

Mở file bằng IDA và tìm kiếm chuỗi `Correct`, sau đó trace ngược lại để xác định hàm kiểm tra input.

Trong hàm check, chương trình không so sánh trực tiếp dữ liệu nhập vào mà thực hiện phép XOR với giá trị `0x55` trước khi kiểm tra:

```asm
xor eax, 0x55
cmp eax, edx
```

Điều này cho thấy mỗi ký tự của input sẽ được XOR với `0x55`, sau đó kết quả mới được so sánh với một mảng byte được lưu trong chương trình.

Dump mảng dữ liệu dùng để kiểm tra:

```text
16 3a 38 25 64 27 30 0a 21 3d 30 0a 2d 3a 27
```

Để khôi phục chuỗi gốc, ta XOR lại từng byte với `0x55`:

```text
0x16 ^ 0x55 = 0x43 = 'C'
0x3a ^ 0x55 = 0x6f = 'o'
...
```

Sau khi giải mã toàn bộ mảng dữ liệu, ta thu được password:

```text
Comp4r3_the_x0r
```
