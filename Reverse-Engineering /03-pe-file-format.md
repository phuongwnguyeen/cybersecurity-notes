# Cấu trúc Chi tiết File PE (Portable Executable Format)

Khi học Reverse Engineering (RE) hoặc Pwnable trên Windows, cấu trúc PE chính là "bản đồ kho báu". Nếu không thuộc lòng bản đồ này, bạn sẽ hoàn toàn "bị mù" khi đối mặt với các kỹ thuật nâng cao như: *Fix IAT khi unpack, viết Hooking, PE Injection, hay Patch Header để qua mặt Antivirus.*

---

## 1. PE File là gì?

**PE (Portable Executable)** là định dạng cấu trúc file tiêu chuẩn của hệ điều hành Windows (định dạng riêng cho cả Win32 và Win64). Mọi file bạn nhìn thấy trên Windows có đuôi mở rộng như:
* `.exe` (Chương trình thực thi độc lập)
* `.dll` (Thư viện hàm dùng chung)
* `.sys` (File Driver điều khiển phần cứng hệ thống)
* `.ocx`, `.cpl`...

👉 **Tất cả đều tuân theo cấu trúc PE file.** Khi nằm im trên đĩa cứng (Disk), file PE được tổ chức theo một cấu trúc phân tầng nghiêm ngặt từ trên xuống dưới. Nó được chia làm hai phần chính: **Các Header** (Chứa thông tin cấu trúc, đóng vai trò chỉ đường) và **Các Section** (Các ngăn chứa dữ liệu thực tế).

```text
+-----------------------------------+
|          DOS Header               |  <-- Chứa chữ ký "MZ" huyền thoại
+-----------------------------------+
|          DOS Stub                 |  <-- Đoạn code cổ điển "This program cannot be run in DOS mode"
+-----------------------------------+
|          PE File Header           |  <-- Chứa chữ ký "PE\0\0" và thông tin phần cứng
+-----------------------------------+
|          Optional Header          |  <-- [QUAN TRỌNG NHẤT] Chứa AddressOfEntryPoint, ImageBase, Data Directories
+-----------------------------------+
|          Section Table            |  <-- Danh sách thông tin quản lý các phân đoạn (.text, .data...)
+-----------------------------------+
|          Phân đoạn .text          |  <-- Chứa mã máy (Code Assembly) sau khi dịch
+-----------------------------------+
|          Phân đoạn .data          |  <-- Chứa các biến toàn cục đã khởi tạo
+-----------------------------------+
|          Phân đoạn .rdata         |  <-- Chứa Import Table, Export Table và chuỗi hằng
+-----------------------------------+
```

## 2. Các Header Cốt Lõi (Đầu não chỉ đường)

Header giống như tờ hướng dẫn sử dụng của file. Loader của Windows bắt buộc phải đọc phần này đầu tiên để biết cách xử lý và dựng file tĩnh từ ổ cứng lên thanh RAM.

### 🗂️ 2.1 DOS Header (`IMAGE_DOS_HEADER`)

Nằm ở ngay **đầu file** với độ dài cố định là `64 bytes`. Đây là phần di sản từ thời hệ điều hành DOS cổ đại để lại nhằm đảm bảo tính tương thích ngược. Dân RE chỉ cần nhớ đúng 2 trường quan trọng nhất sau:

* **Trường `e_magic` (Chữ ký nhận diện):** Đầu file luôn luôn bắt đầu bằng 2 ký tự **`MZ`** (`0x5A4D`, viết tắt tên của Mark Zbikowski - vị kỹ sư thiết kế ra định dạng này). Nó giống như một lời chào: *"Xin chào, tôi là một file thực thi hợp pháp của Windows đây!"*. Nếu bạn dùng Hex Editor xóa hoặc sửa 2 chữ này, Windows sẽ từ chối chạy file ngay lập tức.
* **Trường `e_lfanew` (Tối quan trọng):** Nằm ở vị trí cuối cùng của DOS Header (Offset `0x3C`). Trường này chứa **giá trị địa chỉ (Offset) dẫn tới "thông tin chính"** (tức là vị trí của PE Header thật sự phía dưới).
    * *Ví dụ trực quan:* Nếu `e_lfanew = 0x100`, Loader sẽ hiểu: *"Thông tin PE chính nằm ở offset 0x100 phía dưới, hãy nhảy xuống đó mà đọc"*.

### 💾 2.2 DOS Stub

Ngay sau DOS Header là một đoạn mã máy nhỏ (thường chỉ chứa vài dòng lệnh). Nếu bạn đem file `.exe` này vào môi trường DOS cũ kỹ của thế kỷ trước để chạy, đoạn code này sẽ thực thi và in ra dòng chữ: `"This program cannot be run in DOS mode"` rồi tự động thoát.
* *Mẹo RE / Malware:* Trong phân tích thông thường vùng này bị bỏ qua, nhưng các hacker tinh vi rất thích lợi dụng khoảng trống "vô dụng" của DOS Stub này để giấu Shellcode độc hại hoặc các thông tin cấu hình đã mã hóa.

### 🫀 2.3 NT Headers (`IMAGE_NT_HEADERS`)

Đây chính là **"Trái tim" và là đầu não của cấu trúc PE**. Vị trí của nó trên đĩa được xác định chính xác bằng cách đọc giá trị trong trường `e_lfanew` ở trên. Phân vùng này luôn bắt đầu bằng chữ ký (Signature) gồm 4 byte: **`PE\0\0`** (`0x00004550`).

Windows đọc chỗ này để biết cấu trúc tổng quan của file thông qua 2 cấu trúc con nằm bên trong:

#### ① File Header (`IMAGE_FILE_HEADER`)
Chứa các thông tin mang tính chất tổng quan vật lý của file:
* `Machine`: Xác định file được biên dịch cho kiến trúc phần cứng nào (Mã `0x14C` cho file 32-bit hoặc `0x8664` cho file 64-bit).
* `NumberOfSections`: Cho biết chương trình này được chia làm bao nhiêu ngăn tủ dữ liệu (Section) phía dưới.
* `TimeDateStamp`: Lưu thời gian file được tạo ra bởi Compiler.
    * *Mẹo Malware:* Mã độc rất hay dùng kỹ thuật **Timestomping** (cố tình sửa đổi trường này về một mốc thời gian trong quá khứ hoặc ngẫu nhiên) để làm nhiễu thông tin, đánh lừa các nhà phân tích khi điều tra dòng thời gian cuộc tấn công.

#### ② Optional Header (`IMAGE_OPTIONAL_HEADER`)
Dù tên là "Optional" (Tùy chọn) nhưng đối với các file thực thi trên Windows, nó lại là phần **bắt buộc phải có và quan trọng nhất**. Đây là nơi Windows thu thập các thông số vàng như: entry point ở đâu, file chạy 32-bit hay 64-bit, địa chỉ muốn load vào RAM thế nào, và danh sách các cửa ngõ thư mục (Data Directories).

## 3. Các thông số "Vàng" trong Optional Header
Khi bạn ném một file vào các công cụ như **PE-bear** hay **PEStudio**, hãy tập trung phân tích ngay các trường dữ liệu sau:

* `Magic`: Xác định chế độ thực thi của file. Nếu giá trị là `0x10B` thì file là **PE32** (chạy 32-bit), nếu là `0x20B` thì file là **PE32+** (chạy 64-bit).

* `AddressOfEntryPoint` **(AoE)**: Chứa địa chỉ **RVA** (độ lệch tương đối) của câu lệnh đầu tiên chương trình sẽ thực thi khi nạp vào RAM. Đây chính là đích đến đầu tiên mà CPU nhảy vào.

  > ⚠️ Hiệu đính kỹ thuật quan trọng: Đối với lập trình viên, điểm bắt đầu là hàm `main()` hoặc `WinMain()`. Nhưng đối với cấu trúc PE, `AddressOfEntryPoint` thực tế trỏ vào đoạn code khởi tạo môi trường của ngôn ngữ (`Runtime Startup Code` như `mainCRTStartup`). Đoạn code hệ thống này sẽ chạy trước để setup bộ nhớ, biến môi trường xong xuôi rồi mới gọi đến hàm `main()` của bạn.

* `ImageBase` **(Địa chỉ gốc mong muốn)**: Là tọa độ bộ nhớ ảo lý tưởng mà chương trình muốn được Windows nạp vào RAM. Mặc định đối với file `.exe` thường là `0x400000` (hoặc `0x140000000` cho 64-bit), còn file `.dll` là `0x10000000`.

* `SectionAlignment` và `FileAlignment` **(Quy tắc căn lề)** :
   
   - FileAlignment: Quy định kích thước căn lề của các Section khi nằm trên đĩa cứng (thường là bội số của `0x200` bytes - kích thước 1 Sector ổ đĩa).

   - SectionAlignment: Quy định kích thước căn lề của các Section khi được nạp trên RAM (thường là bội số của `0x1000` bytes - tương đương kích thước 1 Page bộ nhớ ảo).

   > Ý nghĩa: Đây chính là nguyên nhân trực tiếp làm cho cấu trúc file trên đĩa cứng và trên RAM bị lệch nhau (Xem chi tiết ở Mục 5).

## 4. Bảng danh mục Data Directories (Cửa ngõ kết nối)
Nằm ở cuối cùng của Optional Header là một mảng cấu trúc cực kỳ đặc biệt gồm 16 phần tử gọi là Data Directory. Mỗi phần tử đóng vai trò như một "cửa ngõ" trỏ đến vị trí của các bảng quản lý tính năng nâng cao của file PE. Trong RE, bạn bắt buộc phải thuộc lòng 4 thư mục đầu tiên:

| Chỉ số (Index) | Tên Thư Mục | Vai trò trong phân tích RE / Khai thác và Unpack |
| :---: | :--- | :--- |
| **0** | **`EXPORT Directory`** | Quản lý danh sách các hàm mà file này "xuất bản" cho các file khác gọi dùng chung. Cực kỳ quan trọng khi phân tích file `.dll`. |
| **1** | **`IMPORT Directory`** | [TỐI QUAN TRỌNG] Quản lý danh sách các hàm hệ thống mà chương trình phải đi "mượn" từ các DLL ngoài (như `printf`, `MessageBoxA`) để ghi vào cấu trúc Import Table. |
| **2** | **`RESOURCE Directory`**| Quản lý tài nguyên đi kèm như Icon, ảnh giao diện, hệ thống Menu, và thông tin phiên bản phần mềm (`Version Info`). |
| **5** | **`BASERELOC Directory`**| Chứa bảng Base Relocation, phục vụ việc tính toán và sửa lại các địa chỉ tuyệt đối trong code khi cơ chế ngẫu nhiên hóa bộ nhớ **ASLR** được kích hoạt. |

## 5. Section Table & Các Phân Đoạn Bộ Nhớ (Ngăn tủ chứa đồ)
Nếu các Header phía trên đóng vai trò là bản đồ chỉ đường, thì các **Section** chính là các ngăn riêng biệt trong một tủ đồ lớn. Để đảm bảo an toàn hệ thống, dữ liệu không được để lẫn lộn mà được phân chia vào từng ngăn tủ dựa theo mục đích sử dụng và phân quyền bảo vệ (`R` - Read, `W` - Write, `X` - Execute):

* **`.text` (hoặc `CODE`)**: Ngăn chứa **mã máy (Opcode / Assembly)** sau khi biên dịch. Đây là phân vùng duy nhất CPU sẽ nhảy vào để đọc và thực thi lệnh.
    * *Phân quyền mặc định:* **`RX`** (Read / Execute) - Cho phép đọc và thực thi, tuyệt đối cấm sửa đổi (Write) để ngăn chương trình tự làm hỏng code hoặc bị chèn mã độc lúc đang chạy.
* **`.data`**: Ngăn chứa các **biến toàn cục (Global variable)** và biến tĩnh (Static variable) đã được lập trình viên gán sẵn giá trị cố định từ đầu.
    * *Phân quyền mặc định:* **`RW`** (Read / Write) - Cho phép đọc và thay đổi, cập nhật giá trị của biến khi chương trình vận hành.
* **`.rdata` (Read-only Data)**: Ngăn chứa dữ liệu chỉ cho đọc nhưng cấm sửa. Thường là nơi lưu trữ các chuỗi ký tự hằng số cố định (Ví dụ: các chuỗi thông báo hiển thị trên giao diện như `"Wrong Password!"`, `"Key hợp lệ"`, hoặc các URL kết nối ẩn của malware) và các hằng số `const`.
    * *Phân quyền mặc định:* **`R`** (Read-Only).
    * *Lưu ý kỹ thuật:* Bảng quản lý hàm hệ thống (Import Table) cũng thường được các trình biên dịch hiện đại giấu ở phân vùng này.
* **`.idata`**: Phân vùng chuyên biệt chứa cấu trúc bảng **Import Table** (ở một số compiler, bảng này sẽ được gộp chung trực tiếp vào phân vùng `.rdata`).
* **`.rsrc` (Resource)**: Ngăn chứa toàn bộ tài nguyên giao diện ứng dụng bao gồm: Icon phần mềm, hình ảnh hiển thị, hệ thống giao diện Menu, và thông tin phiên bản bản quyền (`Version Info`).
    > ⚠️ **Dấu hiệu nhận biết Malware:** Mã độc rất thích nén hoặc mã hóa nguyên một file thực thi độc hại khác rồi nhét ẩn vào ngăn `.rsrc` này. Khi chạy, nó sẽ gọi chuỗi API hệ thống: `FindResource` ──> `LoadResource` ──> `LockResource` để lôi file ẩn này ra từ vùng tài nguyên, tiến hành giải nén trên bộ nhớ và kích hoạt ngầm nhằm né tránh sự rà quét của Antivirus trên ổ đĩa.

## 6. Cơ chế Nạp Bộ Nhớ (Memory Mapping)

> 🚨 **QUY TẮC CỐT LÕI:** Cấu trúc file tĩnh trên ổ cứng (Disk) **KHÔNG BAO GIỜ GIỐNG** cấu trúc file khi đang chạy trong bộ nhớ (RAM).

Khi bạn double-click để chạy một file `.exe`, Windows Loader sẽ không đơn thuần là copy nguyên xi file đó thả vào RAM. Thay vào đó, nó sẽ thực hiện một quá trình gọi là **Memory Mapping** (Ánh xạ bộ nhớ) để dựng cấu trúc file từ trạng thái tĩnh sang trạng thái động.

### 📐 Lý do có sự khác biệt: Quy tắc căn lề (Alignment)

Sự khác biệt này xuất phát từ hai thông số vàng đã được quy định sẵn trong Optional Header:
* **`FileAlignment` (Mặc định thường là `0x200` bytes):** Quy định dữ liệu nằm trên đĩa cứng phải được xếp theo từng khối (Sector) có kích thước là bội số của `0x200`.
* **`SectionAlignment` (Mặc định thường là `0x1000` bytes):** Quy định khi lên RAM, mỗi Section phải bắt đầu tại một đầu trang bộ nhớ ảo (Page) có kích thước là bội số của `0x1000`.

### 🔄 Quy trình dịch chuyển từ Đĩa lên RAM của Loader:

1. **Bung file vào bộ nhớ ảo:** Loader đọc các Header để nắm thực đơn cấu trúc, sau đó cấp phát một vùng không gian bộ nhớ ảo bắt đầu từ địa chỉ `ImageBase`.
2. **"Kéo dãn" các ngăn tủ (Section):** Do kích thước căn lề trên RAM (`0x1000`) lớn hơn trên đĩa (`0x200`), Loader bắt buộc phải đẩy các Section ra xa nhau để khớp với các đầu phân trang bộ nhớ.
3. **Chèn byte rác (Padding):** Để lấp đầy khoảng trống bị kéo dãn giữa các Section, Loader sẽ tự động chèn thêm các chuỗi byte trống `0x00` vào.

> 💡 **Hệ quả đối với RE:** > Do hiện tượng "kéo dãn" này, vị trí byte chính xác tính từ đầu file khi nằm im trên đĩa (**File Offset**) và địa chỉ tuyệt đối của byte đó khi chương trình đang chạy (**Virtual Address - VA**) sẽ bị lệch nhau hoàn toàn. Khi làm Lab, bạn bắt buộc phải tính toán qua tọa độ trung gian **RVA** thì mới có thể đối chiếu chính xác dữ liệu giữa IDA (Static) và x64dbg (Dynamic).

## 7. Hệ tọa độ Bộ nhớ: ImageBase, VA và RVA

Để định vị chính xác vị trí của một dòng lệnh code (Assembly) hoặc vị trí của một biến khi file PE đã được Loader nạp lên RAM, dân RE bắt buộc phải sử dụng hệ tọa độ tương đối dưới đây. 



### 🌐 7.1 Định nghĩa các khái niệm cốt lõi

* **`ImageBase` (Địa chỉ gốc):** Tọa độ xuất phát (điểm đặt chân đầu tiên) của chương trình bên trong không gian bộ nhớ ảo. Windows luôn ưu tiên nạp file vào địa chỉ này nếu vùng nhớ đó còn trống.
* **`VA` (Virtual Address - Địa chỉ ảo tuyệt đối):** Là địa chỉ thực tế, chính xác của một byte dữ liệu cụ thể khi chương trình đang chạy trên thanh RAM ảo. Đây chính là địa chỉ bạn nhìn thấy khi đặt breakpoint trong các trình Debugger (như x64dbg).
* **`RVA` (Relative Virtual Address - Địa chỉ ảo tương đối):** Là khoảng cách (độ lệch / offset) tính từ địa chỉ gốc `ImageBase` cho đến vị trí byte dữ liệu bạn cần tìm.

```text
Địa chỉ ảo VA ───────────────────────────────────────────────┐
                                                             ▼
Thanh RAM ảo: [  ImageBase  | ... Khoảng lệch RVA ... |  Dữ liệu cần tìm  ]
              ▲                                       
              └───────────────────────────────────────┘
```
### 🧮 7.2 Công thức liên hệ toán học

Mối quan hệ giữa ba đại lượng này được thể hiện qua hai công thức cốt lõi sau:
$$\text{RVA} = \text{VA} - \text{ImageBase}$$
$$\text{VA} = \text{ImageBase} + \text{RVA}$$

#### 💡 Ví dụ minh họa thực tế:

Giả sử bạn mở **IDA Pro** (Phân tích tĩnh) lên và thấy một chuỗi thông báo thành công nằm ở địa chỉ ảo mặc định là:
$$\text{VA}_{\text{IDA}} = \text{0x401500}$$

Biết rằng địa chỉ gốc giả định ban đầu của file trong `Optional Header` là:
$$\text{ImageBase}_{\text{Mặc định}} = \text{0x400000}$$

* **Bước 1 (Tính RVA tĩnh):** Khoảng cách lệch tương đối của chuỗi đó tính từ đầu file là:
  $$\text{RVA} = \text{0x401500} - \text{0x400000} = \text{0x1500}$$
* **Bước 2 (Tìm địa chỉ Runtime):** Khi chạy thực tế trong Debugger, do cơ chế bảo mật **ASLR** kích hoạt, Loader đẩy chương trình sang nạp ở một địa chỉ gốc hoàn toàn mới, ví dụ:
  $$\text{ImageBase}_{\text{Runtime}} = \text{0x7ff600000}$$
* **Bước 3 (Ánh xạ địa chỉ sống):** Lúc này, để tìm thấy đúng chuỗi đó trong **x64dbg** nhằm đặt breakpoint, địa chỉ ảo thực tế (`VA`) bạn cần tìm sẽ là:
  $$\text{VA}_{\text{Runtime}} = \text{0x7ff600000} + \text{0x1500} = \text{0x7ff601500}$$

> 📌 **Bài học xương máu:** Trong RE, địa chỉ tuyệt đối `VA` có thể thay đổi liên tục sau mỗi lần bật/tắt máy do ASLR, nhưng khoảng lệch `RVA` của một hàm hay một biến thì **luôn luôn cố định**. Do đó, hãy luôn tìm mọi cách quy đổi địa chỉ về `RVA` để phân tích!

## 8. Cơ chế Hoạt Động của Import Table & Ứng dụng trong Unpacking

Khi chương trình của bạn sử dụng hàm `printf()` của ngôn ngữ C hoặc hàm hiển thị hộp thoại `MessageBoxA()`, code xử lý thực tế của các hàm này không nằm trong file `.exe` của bạn, mà chúng nằm ở các file thư viện dùng chung của hệ điều hành Windows gọi là **DLL (Dynamic Link Library)**.

Để "mượn" được các hàm này, file PE phải sử dụng cơ chế liên kết động (Dynamic Linking) thông qua cấu trúc **Import Table**.

---

### ⚙️ 8.1 Quy trình liên kết động lúc Runtime

Quá trình phân giải địa chỉ hàm từ đĩa cứng lên thanh RAM được hệ điều hành phối hợp thực hiện theo 4 bước nghiêm ngặt sau:

1. **Ghi chép tĩnh (Trong file trên đĩa):** Trình biên dịch tạo sẵn một bảng Import nằm ở ngăn `.idata`. Trong bảng này ghi rõ các cặp thông tin ràng buộc: Chữ ký chuỗi tên file DLL (Ví dụ: `"user32.dll"`) và danh sách chuỗi tên hàm cần gọi (Ví dụ: `"MessageBoxA"`).
2. **Đọc và Nạp (Lúc khởi động chương trình):** Khi bạn double-click chạy file, Windows Loader sẽ đọc bảng Import này, tự động đi tìm file thư viện `"user32.dll"` trong hệ thống và nạp nó vào chung không gian bộ nhớ ảo của tiến trình.
3. **Tra cứu và Sửa bảng (Resolve địa chỉ):** Loader tiến hành tra tìm tọa độ địa chỉ thực tế của hàm `MessageBoxA` đang nằm trên RAM của thư viện `"user32.dll"`. Sau đó, nó **ghi đè địa chỉ thực tế (địa chỉ sống)** này vào một bảng ô nhớ gọi là **IAT (Import Address Table)** nằm trong vùng nhớ của chương trình.
4. **Thực thi:** Khi CPU chạy đến câu lệnh gọi hàm (Ví dụ: `call dword ptr [0x405000]`), nó sẽ nhảy vào ô nhớ thuộc bảng IAT tại địa chỉ `0x405000` để lấy địa chỉ thực của hàm hệ thống và thực thi mượt mà.

---

### 🥊 8.2 Tư duy Unpacking / Dump Memory chuyên sâu

Các bộ phần mềm bảo vệ file (Packer) hoặc Malware nâng cao rất thích phá hủy hoặc xáo trộn bảng Import này nhằm chống lại các Analyst phân tích tĩnh. 

* **Cơ chế ẩn giấu:** Khi file bị Pack, bảng IAT tĩnh trên đĩa chỉ chứa các địa chỉ rác, tên hàm giả hoặc bị xóa hoàn toàn. Chương trình sẽ tự giải mã payload và tự động "vá" (dựng lại) bảng IAT đúng nghĩa trên RAM ngay trong lúc đang chạy (Runtime).
* **Kỹ thuật phục hồi (IAT Rebuilding):** Nếu một Analyst thực hiện kỹ thuật **Dump Memory** (gột/trích xuất toàn bộ bộ nhớ từ RAM ra thành một file PE tĩnh trên đĩa) mà quên không thực hiện kỹ thuật **IAT Rebuilding** (dò tìm lại tọa độ các hàm hệ thống gốc và xây dựng lại cấu trúc Import Table hợp lệ cho file), file thu được sẽ bị crash lập tức khi chạy vì cấu trúc liên kết hàm đã bị gãy hoàn toàn.
