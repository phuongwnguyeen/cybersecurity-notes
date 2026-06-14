# Task 4:
Đăng nhập bằng tài khoản đc cung cấp:
- User: wiener
- Pass: peter

<img width="1009" height="467" alt="image" src="https://github.com/user-attachments/assets/e6c33101-35d0-43d3-93f4-2a04dcb4e280" />

Sử dụng Burp Suite mở proxy, intercept on để chặn request khi upload avatar đuôi png bất kì :

<img width="1015" height="617" alt="image" src="https://github.com/user-attachments/assets/23c68bd3-e6d1-40fb-972b-816dc206c199" />

Send to Repeater, sau đó chỉnh sửa phần dữ liệu upload trong request như sau:

- Đổi tên file từ file ảnh sang .htaccess
- Đổi Content-Type từ “image/png” sang “text/plain”
- Thay đổi nội dung file ảnh bằng “AddType application/x-httpd-php .l33t”

<img width="1025" height="257" alt="image" src="https://github.com/user-attachments/assets/0c9f9c45-63f2-4921-bf2b-81d25c353974" />

Bấm Send, sau khi chỉnh sửa request upload file .htaccess, gửi request bằng Burp Repeater để tải file lên máy chủ. 

Kết quả phản hồi từ server cho thấy request được chấp nhận thành công.

<img width="794" height="474" alt="image" src="https://github.com/user-attachments/assets/8c51c2c0-56f7-447a-9fc1-c9027af87828" />

Nhờ cấu hình trong file .htaccess, Apache sẽ xử lý các file có đuôi .l33t như php, thay vì upload file .php (thường bị blacklist chặn), có thể upload file với đuôi khác như .l33t nhưng vẫn được thực thi như mã php.

Tiếp theo, tiếp tục chỉnh sửa request upload để upload file shell với các thay đổi sau:

- Đổi tên file thành “shell.l33t”
- Giữ Content-Type
- Thay nội dung file bằng đoạn mã php: <?php echo "OK123"; ?>

<img width="1002" height="211" alt="image" src="https://github.com/user-attachments/assets/6f8cc40c-0cf0-4cef-ae01-f545f048d814" />

Bấm Send, sau khi chỉnh sửa request upload file shell.l33t , gửi request bằng Burp Repeater để tải file lên máy chủ. 
Kết quả phản hồi từ server cho thấy request được chấp nhận thành công.

<img width="805" height="432" alt="image" src="https://github.com/user-attachments/assets/21a25359-4f86-4bd5-a0c2-35e2e0a9bbf7" />

Tiếp theo truy cập file này trên trình duyệt tại đường dẫn upload của avatar:
/files/avatars/shell.l33t

<img width="1072" height="169" alt="image" src="https://github.com/user-attachments/assets/7daa6f5b-5e16-4c7a-84a2-995f4a565972" />

Điều này chứng minh rằng:
- File .l33t đã được server xử lý như file php
- Và attacker đã có thể thực thi mã trên server.
- 
Tiếp tục sửa nội dung file shell thành: 
<?php echo file_get_contents('/home/carlos/secret'); ?>

<img width="1032" height="257" alt="image" src="https://github.com/user-attachments/assets/8744cbff-2241-45cf-ba1f-0854e76ede68" />

<img width="787" height="441" alt="image" src="https://github.com/user-attachments/assets/c06a2379-21d7-49a1-976c-95c433ce4385" />

Truy cập lại file shell /files/avatars/shell.l33t  trên trình duyệt để lấy nội dung file /home/carlos/secret. Sau khi lấy được giá trị secret, nhập chuỗi này vào ô submit trên lab để hoàn thành bài.

<img width="1074" height="523" alt="image" src="https://github.com/user-attachments/assets/fff07598-6454-42d6-9109-fd2af1e8e937" />

<img width="1086" height="236" alt="image" src="https://github.com/user-attachments/assets/6dbba5ef-c8b8-4fe2-bada-0c443efc08d6" />

# Task 6:
Đăng nhập bằng tài khoản đc cung cấp:
- User: wiener
- Pass: peter

<img width="960" height="456" alt="image" src="https://github.com/user-attachments/assets/4720f826-d046-43bb-9c55-d53befa41caf" />

Sử dụng Burp Suite mở proxy, intercept on để chặn request khi upload avatar đuôi png bất kì :

<img width="1069" height="617" alt="image" src="https://github.com/user-attachments/assets/8d5cf582-096d-46e9-a117-a6d330ae759e" />

Send to Repeater, sau đó chỉnh sửa phần dữ liệu upload trong request như sau:
- Đổi tên file upload thành shell.php
- Giữ Content-Type 
- Thay nội dung file bằng một file ảnh hợp lệ có chèn mã php, ví dụ <?php echo "OK123"; ?>

<img width="1025" height="572" alt="image" src="https://github.com/user-attachments/assets/c932eced-b7d3-410f-9336-2ced804a2f27" />

Mục đích của bước này là tạo một polyglot file, tức là một file vừa có thể được chấp nhận như ảnh, vừa có thể thực thi như mã php trên server.

<img width="887" height="460" alt="image" src="https://github.com/user-attachments/assets/d376e061-f934-4a14-b9b4-5568e4340a30" />

Bấm Send, sau khi chỉnh sửa request upload file, gửi request bằng Burp Repeater để tải file lên máy chủ. 

Kết quả phản hồi từ server cho thấy request được chấp nhận thành công.

<img width="1069" height="206" alt="image" src="https://github.com/user-attachments/assets/ffd79fd2-4fe3-408c-83f0-f4be14275ad2" />

Sau khi upload thành công file shell.php, truy cập file này trên trình duyệt tại đường dẫn upload của avatar: 

https://0a51005303f95cb980d4a3b0006c00a4.web-security-academy.net/files/avatars/shell.php

<img width="425" height="197" alt="image" src="https://github.com/user-attachments/assets/d9bdcce9-d39c-4854-92f5-92327db0797f" />

Nếu khai thác thành công, trình duyệt sẽ hiển thị kết quả: OK123

Điều này chứng minh rằng:
- File đã vượt qua bước kiểm tra nội dung ảnh,
- Đồng thời mã php bên trong vẫn được server thực thi

<img width="985" height="204" alt="image" src="https://github.com/user-attachments/assets/23851b1b-1c65-4660-83cf-cd3cc44ae72a" />

Sau khi xác nhận có thể thực thi mã PHP, tiếp tục sửa nội dung file shell thành: <?php echo file_get_contents('/home/carlos/secret'); ?>

<img width="892" height="474" alt="image" src="https://github.com/user-attachments/assets/43436e31-8a75-42b2-8aa4-70a4fd398830" />

Bấm Send, sau khi chỉnh sửa request upload file, gửi request bằng Burp Repeater để tải file lên máy chủ. 

Kết quả phản hồi từ server cho thấy request được chấp nhận thành công.

<img width="714" height="162" alt="image" src="https://github.com/user-attachments/assets/b67133ff-3391-45d1-96d4-5e3f616d0b86" />

Truy cập lại file shell trên trình duyệt để lấy nội dung file /home/carlos/secret. 

Sau khi lấy được giá trị secret, nhập chuỗi này vào ô submit trên lab để hoàn thành bài.

<img width="901" height="369" alt="image" src="https://github.com/user-attachments/assets/df46ec89-33c3-4080-af6c-4416fd490ae0" />

<img width="1064" height="197" alt="image" src="https://github.com/user-attachments/assets/8d159c07-980c-4ba6-974d-9a5ef621f83d" />














