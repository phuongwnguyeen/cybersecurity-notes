## Analysis

Chạy chương trình, ta thấy chương trình yêu cầu nhập một chuỗi và trả về `Correct` hoặc `Wrong` tùy theo kết quả kiểm tra.

Mở file bằng IDA và tìm kiếm chuỗi `Correct`. Từ vị trí tham chiếu đến chuỗi này, trace ngược lại để xác định hàm kiểm tra input.

Trong hàm check, chương trình thực hiện so sánh trực tiếp từng ký tự của input với các giá trị hằng được hardcode trong mã lệnh:

```asm
cmp eax, 0x43
cmp eax, 0x6f
cmp eax, 0x6d
...
```

Các giá trị này chính là mã ASCII của chuỗi cần nhập:

```text
0x43 = 'C'
0x6f = 'o'
0x6d = 'm'
...
```

Chuyển đổi toàn bộ các giá trị hex sang ký tự ASCII và ghép lại theo đúng thứ tự, ta thu được password:

```text
Compar3_the_ch4ract3r
```

