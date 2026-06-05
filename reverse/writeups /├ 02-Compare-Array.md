## Analysis

Chạy chương trình và thử nhập dữ liệu bất kỳ, chương trình sẽ trả về `Correct` hoặc `Wrong`.

Mở file bằng IDA và tìm kiếm chuỗi `Correct`, sau đó trace ngược lại để xác định hàm kiểm tra input.

Trong hàm check, chương trình sử dụng vòng lặp để duyệt qua từng ký tự của input và so sánh với dữ liệu được lưu trong vùng `.data`:

```asm
cmp DWORD PTR [rcx+rax*4], edx
```

Điều này cho thấy giá trị đúng đã được lưu sẵn trong chương trình thay vì được tính toán trong lúc chạy.

Tiếp tục theo dõi địa chỉ được tham chiếu và dump dữ liệu tại `0x140003000`. Chuyển các giá trị thu được sang ký tự ASCII rồi ghép lại theo đúng thứ tự.

Kết quả thu được:

```text
Comp4re_the_arr4y
```
