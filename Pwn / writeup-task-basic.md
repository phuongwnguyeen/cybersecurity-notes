# 1_warmUp

Đầu tiên, sử dụng các lệnh sau để kiểm tra thông tin binary:
```bash
file 1_warmUp
checksec 1_warmUp
```
<img width="806" height="274" alt="image" src="https://github.com/user-attachments/assets/9db12d4a-b62b-4533-9c12-269304f09e27" />

-> Binary là ELF 64-bit Linux có bật NX và PIE

Tiếp theo, ta cấp quyền chạy và cho chạy file

<img width="647" height="138" alt="image" src="https://github.com/user-attachments/assets/b69ba44b-8a18-449c-9da1-cdb730aeefc8" />

Khi nhập tổng thực sự của 2 số, chương trình vẫn báo sai.

→ Điều này cho thấy có hidden logic bên trong binary

Vậy ta mở file bằng IDA để xem cụ thể hơn, mở hàm main phần pseudocode

<img width="1328" height="451" alt="image" src="https://github.com/user-attachments/assets/0f25e2e2-f917-437f-a7df-06f0e6df7cf3" />

Ta thấy được trong hàm main, chương trình cộng thêm giá trị 1337 vào tổng 2 số random

→ Do đó đáp án đúng không phải thật sự như chúng ta nghĩ là num1 + num2 mà là:

 num1 + num2 + 1337

Sau khi ta xác định được hidden logic của bài, ta có hướng exploit như sau:

1. Đọc 2 số random mà chương trình đưa ra
2. Tính: num1 + num2 + 1337
3. Gửi lại kết quả cho chương trình và nhận flag

Ta tạo file script bằng cách nhập : nano solve.py

Sau đó viết script solve pwntools bằng python theo hướng exploit

<img width="462" height="493" alt="image" src="https://github.com/user-attachments/assets/de6734a4-1ddd-4448-beef-35e9c7879719" />

<img width="806" height="257" alt="image" src="https://github.com/user-attachments/assets/67b4aefa-f1f3-434b-9aa2-e19f7d0e5d58" />

Ra flag: ISP{b4bY_p4rs1ng_w1th_pwnt00ls}

# 2_SpeedRunTime

Đầu tiên, ta sử dụng

 ``` bash
file 2_SpeedRunTime
checksec 2_SpeedRunTime
```

→ Để kiểm tra thông tin binary

<img width="808" height="264" alt="image" src="https://github.com/user-attachments/assets/8af608fb-beac-4784-803e-f2e02640759a" />

-> Binary là ELF 64-bit Linux có bật NX và PIE

Tiếp theo ta cấp quyền và cho chạy file:

<img width="582" height="488" alt="image" src="https://github.com/user-attachments/assets/3746ec53-2153-4132-a0f0-7e14c914eeb6" />

→ Chương trình yêu cầu giải 50 phép toán trong vòng 5 giây nên không thể nhập tay.

Vậy ta mở file bằng IDA dể xem cụ thể hơn, mở hàm main phần pseudocode

<img width="1142" height="1092" alt="image" src="https://github.com/user-attachments/assets/d1c870c6-4199-43db-9766-7a5740df405b" />

Ta thấy chương trình sử dụng:

- rand( ) để random số
- rand( ) % 3 để chọn phép toán
- vòng lặp 50 lần để kiểm tra đáp án

Các phép toán gồm: cộng, trừ, nhân

Sau khi xác định được vấn đề bài toán, ta có hướng exploit như sau:

1. Đọc phép toán từ chương trình
2. Tách lấy số trong output
3. Tính toán đáp án chính xác
4. Gửi lại tự động và lấy flag

Ta tạo file script cho bài thứ 2 theo đúng hướng exploit:

<img width="385" height="682" alt="image" src="https://github.com/user-attachments/assets/919e90f1-3b70-4134-a8da-48c69b94aa00" />

<img width="754" height="169" alt="image" src="https://github.com/user-attachments/assets/516146d0-91c4-4fbb-b788-4c870a9ff0de" />

Ra flag: ISP{sp33d_m4th_l00p_m4st3r}

# 3_PIN

Đầu tiên, ta kiểm tra thông tin binary:

<img width="1020" height="266" alt="image" src="https://github.com/user-attachments/assets/124cfd65-6746-47b8-b9af-2a87857ecc03" />

→  Binary là ELF 64-bit Linux có bật NX và PIE

Tiếp theo ta cấp quyền và cho chạy file:

<img width="462" height="143" alt="image" src="https://github.com/user-attachments/assets/c0863f10-5c2b-4a99-92b8-078917fc4e9f" />

→ Khi chạy file, chương trình đưa ra một giá trị Target và yêu cầu nhập PIN tương ứng. Nhưng ta không biết phải nhập PIN là gì? 

Vậy ta mở file bằng IDA dể xem cụ thể hơn, mở hàm main phần pseudocode

<img width="1030" height="469" alt="image" src="https://github.com/user-attachments/assets/d3fe383f-3d96-4e5a-8475-099b22da667e" />

Chương trình tạo một số ngẫu nhiên v8 và biến đổi nó theo công thức:

- v7 = 3 * (v8 ^ 0xCAFE) - 100

Sau khi ta nhập PIN, chương trình tiếp tục biến đổi PIN bằng công thức tương tự:

- v6 = 3 * (v5 ^ 0xCAFE) - 100

Nếu v6 == v7 thì mới lấy được flag

→ Do đó, ta cần đảo ngược công thức để tìm lại giá trị ban đầu của v8

Sau khi xác định được vấn đề bài toán, ta đảo ngược công thức theo thứ tự ngược lại:

1. +100
2. /3  
3. XOR lại vs 0xCAFE

→ Công thức cuối cùng: **pin = ( (target + 100) // 3) ^ 0xCAFE**

Ta tạo file script cho bài 3 theo hướng đã tư duy:

<img width="409" height="357" alt="image" src="https://github.com/user-attachments/assets/15f8f86f-6cbc-4882-ba2c-9f8c109f1afe" />

<img width="768" height="199" alt="image" src="https://github.com/user-attachments/assets/6be3a4a2-1642-464e-bcd4-d302398db196" />

Solve ra flag: ISP{r3v3rs3_th3_3qu4t10n_l1k3_4_b0ss}

# 4_prophet

Đầu tiên, ta kiểm tra thông tin binary:

<img width="1015" height="271" alt="image" src="https://github.com/user-attachments/assets/e9ab39a4-a950-45fa-953e-dc9b9103b331" />

→  Binary là ELF 64-bit Linux có bật NX và PIE

Tiếp theo, ta cấp quyền và chạy file:

<img width="612" height="486" alt="image" src="https://github.com/user-attachments/assets/9467de70-cf6c-454b-b0f2-374a2740866b" />

→ Ta thấy rằng có 25 lượt đoán trong khoảng giá trị 1 → 1,000,000.

Ta mở file bằng IDA, mở hàm main phần pseudocode để xem cụ thể cách chương trình hoạt động:

<img width="1165" height="764" alt="image" src="https://github.com/user-attachments/assets/af6e1583-9992-4512-b6fe-9ed46dca7556" />

Mỗi lần nhập số, chương trình sẽ trả về:

- [Signal 1] : số đoán nhỏ hơn secret
- [Signal -1]: số đoán lớn hơn hoặc bằng secret
- [Signal 0]: Thành công và in flag

→ Brute force không khả thi 

→ Ta dùng Binary Search

- Đoán số ở giữa
- Nếu secret lớn hơn hoặc bằng → tìm tiếp nửa bên phải
- Nếu secret nhỏ hơn → tìm tiếp nửa bên trái

→ Mỗi lần tìm sẽ giảm một nửa khoảng tìm kiếm

Ta đã có hướng giải quyết bài toán vậy ta tạo file script python :

<img width="386" height="568" alt="image" src="https://github.com/user-attachments/assets/e64ca1e5-faef-4258-82ea-1d79684183ae" />

<img width="689" height="180" alt="image" src="https://github.com/user-attachments/assets/5e58d577-78fa-4e6f-afb0-4295012f6455" />

Ra flag: ISP{b1n4ry_s34rch_m4st3r_1d4_pr0}

# 5_black_m4rk37

<img width="806" height="274" alt="image" src="https://github.com/user-attachments/assets/c0a578fb-46d7-4765-ae48-7b574ec920d5" />

→  Binary là ELF 64-bit Linux có bật NX và PIE

Tiếp theo, ta cấp quyền và chạy thử file:

<img width="677" height="413" alt="image" src="https://github.com/user-attachments/assets/361a4df4-04ee-497e-983f-3e2edf8e9ada" />

→ Ta thấy, ban đầu số gold rất nhỏ nên không thể mua flag bằng cách bình thường được

Vậy ta mở file trong IDA, mở hàm main trong phần pseudocode để xem kĩ hơn:

<img width="1314" height="1256" alt="image" src="https://github.com/user-attachments/assets/127a5444-d863-4a9f-be7b-5169b471d060" />

Chương trình có:

- 1 : mua flag
- 2 : kiếm 1 gold
- 3 : thoát game
- 1337 : cheat code bí mật

→ Ngoài các choice này nhập đều hiện Invalid choice (không hợp lệ)

→ Điều này cho thấy nếu ta nhập 1337 thì chương trình sẽ nhảy tới hàm bí mật

<img width="876" height="239" alt="image" src="https://github.com/user-attachments/assets/ab1e2268-b864-4ba4-b3d5-fc6fb8726c6a" />

→ Trong hàm này, hàm sẽ cộng cho player rất nhiều tiền

→ Mua được flag

Sau khi xác định vấn đề bài toán, ta có hướng exploit như sau:

1. Nhập 1337 → kích hoạt hàm ẩn
2. Nhập 1 → mua flag

Ta lập file script python theo hướng exploit như trên:

<img width="407" height="283" alt="image" src="https://github.com/user-attachments/assets/b534a6d0-f216-43ce-8e44-9e62d0633ec8" />

<img width="694" height="199" alt="image" src="https://github.com/user-attachments/assets/ab811c2c-56b2-4498-911c-ff42599abc90" />

Ra flag:  ISP{h1dd3n_m3nus_c4nt_h1d3_fr0m_1d4_pr0}

