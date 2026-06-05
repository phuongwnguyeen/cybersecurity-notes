## Phân tích

Chạy chương trình, ta thấy chương trình yêu cầu nhập một chuỗi và trả về:

- Correct nếu input đúng.
- Wrong nếu input sai.

Mở file bằng IDA và tìm kiếm chuỗi "Correct". Từ vị trí tham chiếu đến chuỗi này, trace ngược lên hàm kiểm tra input.

Trong hàm check, chương trình so sánh trực tiếp từng ký tự của input với các hằng số:

- cmp eax, 0x43
- cmp eax, 0x6f
- cmp eax, 0x6d
- ...

Các giá trị trên tương ứng với mã ASCII:

- 0x43 = 'C'
- 0x6f = 'o'
- 0x6d = 'm'

Tiếp tục chuyển đổi toàn bộ các giá trị hex sang ký tự ASCII và ghép lại, ta thu được password:

-> Compar3_the_ch4ract3r
