# Cấu trúc Toàn tập và Cơ chế Nạp Bộ Nhớ File ELF (Executable and Linkable Format)

Các tập tin ELF đóng vai trò trung tâm trong hoạt động của hệ điều hành Linux. Đây chính là "long mạch" để bạn cày Pwnable (Binary Exploitation) và Reverse Engineering trên Linux. Từ việc hiểu cấu trúc ELF, bạn mới có thể viết được script khai thác bộ nhớ, hiểu cơ chế Stack Canary hoạt động ra sao, hay cách mà các hàm trong thư viện `libc` được liên kết động.

---

## 1. ELF File là gì?

**ELF (Executable and Linkable Format)** là định dạng cấu trúc file tiêu chuẩn cho các chương trình chạy trên hệ điều hành dựa trên Unix/Linux. Định dạng này cực kỳ linh hoạt, cho phép một cấu trúc duy nhất phục vụ cho nhiều loại tập tin khác nhau:

* **Tập tin thực thi (Executable File):** Các chương trình chạy trực tiếp (Ví dụ bài Lab `./chall` hoặc các lệnh hệ thống như `/bin/ls`).
* **Thư viện chia sẻ (Shared Library):** Thường có đuôi mở rộng là `.so` (Shared Object), tương đương với file `.dll` trên Windows.
* **Tập tin đối tượng di chuyển được (Relocatable File):** Thường có đuôi `.o`, được tạo ra sau khi bạn gõ lệnh biên dịch thô `gcc -c main.c` trước khi các file này được liên kết lại với nhau.

### 🌟 3 Đặc điểm cốt lõi của định dạng ELF:
* **Tính tương thích:** Chạy mượt mà trên nhiều hệ thống giống Unix (Linux, BSD, Android...).
* **Khả năng mở rộng:** Hỗ trợ liên kết động, giúp nhiều chương trình dùng chung một mã nguồn thư viện để tiết kiệm dung lượng RAM.
* **Hiệu quả:** Được thiết kế tối ưu để hệ điều hành đọc, phân tích và thực thi với tốc độ nhanh nhất.

---

## 2. Kiến trúc hai góc nhìn (Dual-View) của File ELF

Đây là tư duy nền tảng quan trọng nhất của file ELF. Một file ELF trên ổ cứng sẽ được nhìn nhận dưới **hai góc nhìn hoàn toàn khác nhau** tùy thuộc vào việc nó đang **bị biên dịch** hay đang **được nạp vào RAM để chạy**:



* **Góc nhìn liên kết (Linking View - Lúc phân tích tĩnh / Biên dịch):** * File ELF được chia nhỏ thành nhiều ngăn chứa gọi là **Section**. 
    * Toàn bộ việc quản lý các Section này do **Section Header Table** nằm ở cuối file điều hành. Góc nhìn này phục vụ cho Trình liên kết (`ld`) và các công cụ phân tích như **IDA Pro** hoặc lệnh `readelf -S`.
* **Góc nhìn thực thi (Execution View - Lúc nạp vào RAM chạy):** * Hệ điều hành Linux (Kernel) không rảnh để đọc từng Section nhỏ lẻ. Nó sẽ gom các Section có cùng quyền truy cập lại với nhau thành các khối lớn gọi là **Segment**.
    * Việc quản lý các Segment này do **Program Header Table** đảm nhận. Góc nhìn này phục vụ trực tiếp cho Linux Loader khi nạp chương trình.

> 💡 **Ví dụ ví von dễ hiểu:** > Bạn có một tủ sách chứa rất nhiều cuốn sách nhỏ (tương đương với các **Section**: Sách Toán, sách Lý, sách Hóa, truyện tranh). 
> * **Lúc phân loại (Linking View):** Bạn nhìn vào bìa từng cuốn sách để sắp xếp vào đúng vị trí.
> * **Lúc chuyển nhà (Execution View):** Để vác lên xe cho nhanh, bạn lấy một cái thùng carton lớn (tương đương với **Segment**), gom toàn bộ sách giáo khoa vào một thùng dán nhãn "Chỉ để đọc", gom toàn bộ truyện tranh vào thùng "Giải trí" để bốc xếp một lần.

---

## 3. Đi sâu vào các Header chỉ đường (Siêu dữ liệu)

### 🐧 3.1 ELF Header (`Elf32_Ehdr` / `Elf64_Ehdr`)
Nằm ở ngay **0 byte đầu tiên** của file. Đây là bộ chứng minh nhân dân chứa siêu dữ liệu để hệ điều hành hiểu cách xử lý tệp.

**Các trường thông tin cốt lõi:**
* `e_ident`: Mảng 16 byte chứa mã số đặc biệt (**Magic Bytes**). 4 byte đầu luôn luôn là `0x7F`, tiếp theo là 3 ký tự **`ELF`** (`0x7F 45 4C 46`). Nó giống như lời chào: *"Xin chào, tôi là một file chuẩn Linux đây!"*. Ngoài ra nó còn chứa thông tin `Class` để xác định file là 32-bit (`0x01`) hay 64-bit (`0x02`).
* `e_type`: Xác định loại file (Ví dụ: `ET_REL` là file object, `ET_EXEC` là file thực thi, `ET_DYN` là thư viện chia sẻ / file có bật PIE).
* `e_machine`: Chỉ định kiến trúc CPU mục tiêu (Ví dụ: `EM_X86_64` cho Intel/AMD 64-bit, `EM_ARM` cho các chip điện thoại).
* `e_entry`: Địa chỉ ảo của **Entry Point** (Điểm bắt đầu) – nơi CPU sẽ nhảy vào thực hiện câu lệnh đầu tiên của tiến trình.
* `e_phoff`: Độ lệch (Offset) tính từ đầu file dẫn tới vị trí của bảng **Program Header Table**.
* `e_shoff`: Độ lệch (Offset) tính từ đầu file dẫn tới vị trí của bảng **Section Header Table**.

### 📊 3.2 Program Headers: Các phân đoạn thời gian chạy (Runtime Segments)
Bảng này là **bắt buộc phải có** đối với các file thực thi. Nó cho trình nạp (Loader) biết những phần lớn nào của tệp cần được tải vào bộ nhớ ảo và tải bằng cách nào.

**Các trường chính trong mỗi mục (Entry):**
* `p_type`: Loại phân đoạn.
* `p_offset`: Vị trí bắt đầu của phân đoạn đó nằm ở đâu trong file tĩnh trên đĩa.
* `p_vaddr`: Địa chỉ ảo (`VA`) trên RAM mà phân đoạn này bắt buộc phải được nạp vào.
* `p_filesz` & `p_memsz`: Kích thước của phân đoạn đó trong file và kích thước thực tế khi nó phình ra trên RAM.

**Các loại phân đoạn (Segment) phổ biến cần lưu ý khi làm Lab:**
* **`LOAD`**: Chứa mã máy hoặc dữ liệu thực tế sẽ được nạp thẳng vào RAM. Thường một file ELF sẽ có ít nhất 2 Segment `LOAD`: Một cái chứa code mang quyền `RX` (Đọc/Thực thi) và một cái chứa biến mang quyền `RW` (Đọc/Ghi).
* **`DYNAMIC`**: Chứa các thông tin cấu hình phục vụ cho việc liên kết động (Ví dụ: danh sách các file `.so` cần nạp).
* **`INTERP`**: Chứa một chuỗi ký tự chỉ đường đến tên của trình liên kết động hệ thống (Thường là `/lib64/ld-linux-x86-64.so.2`).
* **`PT_GNU_STACK` (Chuyên dụng cho Pwnable):** Quyết định xem phân vùng Stack của chương trình có quyền thực thi mã máy (`X`) hay không. Nếu quyền là `RW`, tính năng bảo mật **NX (No-Execute)** đã bật, bạn cấm bắn Shellcode lên Stack để chạy mà phải dùng kỹ thuật **ROP (Return-Oriented Programming)**.

### 🗂️ 3.3 Section Headers: Thông tin liên kết và gỡ lỗi
Bảng này nằm ở cuối file ELF, chứa danh bạ mô tả chi tiết các phân đoạn nhỏ phục vụ cho việc biên dịch và gỡ lỗi.

**Các trường chính trong tiêu đề phần:**
* `sh_name`: Tên của phần (Được lưu dưới dạng một chỉ số trỏ vào bảng chuỗi `.strtab`).
* `sh_type`: Loại phần (Ví dụ: `SHT_PROGBITS` cho mã nguồn/dữ liệu hoặc `SHT_SYMTAB` cho bảng ký hiệu).
* `sh_flags`: Các cờ chỉ định thuộc tính bảo mật (Ví dụ: `SHF_EXECINSTR` nghĩa là phần này có chứa mã máy thực thi được).
* `sh_offset` & `sh_size`: Vị trí bắt đầu và kích thước của phần đó trong file.

---

## 4. Các Section phổ biến (Các ngăn tủ dữ liệu của Linux)

Tương tự như cấu trúc PE, file ELF chia nhỏ dữ liệu vào các ngăn tủ chuyên biệt để quản lý an toàn quyền truy cập:

* **`.text`**: Ngăn chứa **mã máy thực thi (Assembly)** của chương trình. CPU sẽ đọc các lệnh byte tại đây để chạy (Quyền mặc định: `RX`).
* **`.rodata` (Read-only Data)**: Ngăn chứa các chuỗi ký tự hằng số cố định (String định dạng như `"%d"`, `"Enter password: "`, `"Flag là: "`...). Phân vùng này chỉ cho đọc, cấm sửa (Quyền mặc định: `R`).
* **`.data`**: Ngăn chứa các biến toàn cục (Global) hoặc biến tĩnh (Static) **đã được gán sẵn giá trị** từ lúc code (Ví dụ: `int a = 1337;`). Quyền mặc định: `RW`.
* **`.bss`**: Ngăn chứa các biến toàn cục **chưa được khởi tạo giá trị** (Ví dụ: `int mang_chua_flag[100];`). 
    > 💡 **Mẹo tối ưu của ELF:** Biến trong `.bss` không tốn dù chỉ 1 byte dung lượng trên ổ cứng (đĩa cứng chỉ lưu tên và kích thước của vùng này). Nhưng khi được nạp lên RAM, Linux Loader sẽ tự động "phình" vùng này ra và xóa sạch tất cả về byte `0x00` (Quyền mặc định: `RW`).
* **`.symtab` (Symbol Table - Bảng ký hiệu):** Được ví như **"Danh bạ điện thoại"** của file. Nó lưu trữ tên và tọa độ địa chỉ của tất cả các hàm, các biến có trong chương trình. Khi file bị chạy lệnh `strip` (Xóa biểu đồ ký tự), bảng này sẽ biến mất khiến việc mò hàm trong IDA Pro trở nên khó khăn hơn.
* **`.strtab` (String Table - Bảng chuỗi):** Nơi lưu trữ thô toàn bộ các chuỗi tên của các hàm và biến để bảng `.symtab` trỏ vào lấy tên.
* **`.rel.text` (hoặc `.rela.text`):** Chứa thông tin về việc di dời địa chỉ. Nó giống như một **"tờ giấy ghi chú"** nhắc nhở trình liên kết: *"Nếu địa chỉ gốc của chương trình bị thay đổi, hãy cộng thêm khoảng lệch vào các vị trí này trong phân vùng `.text`"*.
* **`.plt` (Procedure Linkage Table) & `.got` (Global Offset Table)**: Hai ngăn tủ phối hợp với nhau để thực hiện cơ chế gọi các hàm hệ thống từ thư viện ngoài (Như `printf`, `system`).

---

## 5. Quy trình ELF được map vào Virtual Memory (RAM)

Khi một người dùng gõ lệnh thực thi một file ELF trên Terminal, Linux Kernel và hệ thống phối hợp nhịp nhàng qua 5 bước sau để đưa file từ trạng thái tĩnh lên trạng thái động:

```text
[BƯỚC 1]: Chạy file ELF ──> Kernel đọc ELF Header kiểm tra tính hợp lệ (Magic bytes, Kiến trúc CPU).
   │
   ▼
[BƯỚC 2]: Đọc Program Header Table ──> Quét các mục để biết có bao nhiêu Segment cần nạp, vị trí và quyền hạn.
   │
   ▼
[BƯỚC 3]: Ánh xạ ảo (Memory Mapping) ──> Kernel gọi hàm hệ thống mmap() để map thẳng các Segment LOAD vào RAM.
   │      (.text ──> Vùng code thực thi RX ; .data/.bss ──> Vùng biến dữ liệu RW)
   │
   ▼
[BƯỚC 4]: Nạp thư viện động (Shared Libraries) ──> Trình liên kết động (Dynamic Loader) đọc phân đoạn DYNAMIC.
   │      Nạp file "libc.so" vào RAM ──> Giải mã địa chỉ hàm hệ thống ──> Setup bảng phối hợp PLT/GOT.
   │
   ▼
[BƯỚC 5]: Nhảy tới Entry Point ──> Kernel chuyển giao quyền điều khiển, bắt CPU nhảy tới tọa độ e_entry.
          Chương trình chính thức khởi động!
```
🥊 Tư duy Pwnable nâng cao tại Bước 4 và Bước 5: 
   - Ảnh hưởng của cơ chế PIE (Position Independent Executable): Nếu file ELF được biên dịch có bật PIE, tại Bước 3, Kernel sẽ nạp chương trình vào một địa chỉ `Base Address` ngẫu nhiên hoàn toàn trên RAM. Địa chỉ trong IDA lúc này chỉ là khoảng lệch `RVA`. Bạn bắt buộc phải dùng kỹ thuật làm lộ địa chỉ (Leak địa chỉ) lúc runtime thì mới tính toán được tọa độ để khai thác:

        $$\text{Base Address} = \text{VA}_{\text{Thực tế lúc debug}} - \text{RVA}_{\text{Trong IDA}}$$
     
   -  Khai thác bảng GOT ở Bước 4: Do bảng GOT (Global Offset Table) mang quyền Ghi (`RW`) để Dynamic Loader cập nhật địa chỉ ở Bước 4, hacker thường tận dụng các lỗi tràn bộ nhớ để thực hiện kỹ thuật GOT Overwrite – ghi đè địa chỉ của một hàm hệ thống (`như puts@got`) bằng địa chỉ của hàm kích hoạt Shell (như system). Khi chương trình gọi `puts("hello")`, CPU sẽ bị lừa và nhảy thẳng vào chạy `system("hello")` ──> Chiếm quyền điều khiển Server thành công!
