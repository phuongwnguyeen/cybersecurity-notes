# Phân tích Tĩnh (Static Analysis) vs. Phân tích Động (Dynamic Analysis)

Trong thế giới Reverse Engineering (RE) và Phân tích mã độc (Malware Analysis), Static và Dynamic Analysis chính là hai "trường phái" tư duy, hai vũ khí cốt lõi của một Analyst. Một Chuyên gia giỏi giống như một bác sĩ pháp y: vừa phải khám nghiệm tử thi lúc bất động (Static), vừa phải theo dõi nhịp tim, hành vi của bệnh nhân khi đang sống (Dynamic).

```text
                                File nghi ngờ (malware.exe / chall / sample.elf)
                                                       │
                             ┌─────────────────────────┴─────────────────────────┐
                             ▼                                                   ▼
                    [ PHÂN TÍCH TĨNH ]                                  [ PHÂN TÍCH ĐỘNG ]
               (Mổ xẻ cấu trúc, đọc code Assembly)                 (Kích hoạt chạy trong Sandbox/VM)
                     Cấm cho file chạy!                                 Quan sát hành vi Runtime!
```
# 1. Phân tích Tĩnh (Static Analysis) - Đi sâu vào cấu trúc và mã nguồn

## 1.1 Định nghĩa trực quan
Phân tích Tĩnh là quá trình mổ xẻ, quan sát và nghiên cứu cấu trúc, linh hồn của file thực thi khi nó nằm im trên ổ cứng (Disk) và **TUYỆT ĐỐI KHÔNG CHO CHẠY**.

> 💡 **Ví dụ ví von:** Giống như việc bạn đứng từ xa nhìn một quả bom, dùng kính lúp soi lớp vỏ, đọc nhãn mác, xem sơ đồ mạch điện của quả bom đó để đoán xem nó hoạt động như thế nào chứ tuyệt đối không bấm nút kích nổ.

---

## 1.2 Các kỹ thuật phân tích tĩnh từ cơ bản đến nâng cao

### 1. Nhận diện File & Thu thập thông tin (File Identification & Hashing)
Trước khi phân tích, bạn phải biết kẻ địch là ai. Sử dụng lệnh `file` (Linux) hoặc các công cụ như **Detect It Easy (DIE)**, **PEStudio** (Windows) để xác định:

* **Kiến trúc (Architecture):** File dành cho CPU x86, x64 hay ARM.
* **Định dạng (Format):** File PE (Windows) hay ELF (Linux).
* **Trình biên dịch (Compiler):** Được dựng bằng MSVC, GCC hay Go, Rust... (Định dạng Compiler quyết định hình thù của code Assembly trong IDA).
* **Băm file (Hashing):** Chạy lệnh lấy Hash SHA256 hoặc MD5 của file. Mã hash này giống như dấu vân tay duy nhất của file, dùng để tra cứu trên VirusTotal hoặc các cơ sở dữ liệu Malware xem có ai từng bắt gặp nó chưa (Được gọi là các chỉ số **IOC - Indicator of Compromise**).
    ```bash
    sha256sum malware.exe
    ```

### 2. Phân tích Siêu dữ liệu (Metadata Analysis)
Xem xét các thông tin ẩn của file như: Ngày giờ biên dịch (Compile time), Tên tác giả, Công ty sản xuất, Version...

> 🛠️ **Mẹo RE:** Malware rất hay giả mạo thông tin thành *"Microsoft Corporation"* hoặc *"Unknown publisher"*, hoặc cố tình sửa đổi mốc thời gian về năm 2000 (Kỹ thuật **Timestomping**) để trốn tránh việc bị điều tra theo dòng thời gian (Timeline).

### 3. Phân tích Chuỗi ký tự (String Analysis)
Dùng lệnh `strings <file>` để quét toàn bộ các chuỗi văn bản dạng ASCII/Unicode nằm trong file. Bạn cần săn tìm:

* Các địa chỉ IP, URL độc hại (Ví dụ: `http://evil-c2-server.com`).
* Các lệnh hệ thống (`cmd.exe`, `powershell.exe`).
* Các chuỗi thông báo lỗi hoặc Password ẩn.

> 🎯 **Tư duy Crackme / CTF:** Tìm các từ khóa thành bại như *"Correct Password!"*, *"Wrong Code!"*, sau đó dùng IDA Pro trace ngược (`X-refs`) xem hàm nào gọi đến chuỗi này để bẻ khóa (Patch).
>
> ⚠️ **Hạn chế:** Malware hiện đại rất ít khi để chuỗi lộ thiên, chúng thường mã hóa chuỗi (Encrypt/Obfuscate), khi chạy lên RAM mới giải mã ra.

### 4. Phân tích hàm gọi hệ thống (Import/Export Analysis)
Đây là bước cực kỳ quan trọng để đoán trước hành vi của file thông qua bảng IAT (Import Address Table):

* Nếu file Import các hàm: `WinHttpOpen`, `InternetConnect`, `send`, `recv` $\rightarrow$ **Mã độc mạng (Network Malware)** chuyên kết nối Server để tải payload.
* Nếu file Import các hàm: `VirtualAlloc`, `WriteProcessMemory`, `CreateRemoteThread` $\rightarrow$ Dấu hiệu 99% của kỹ thuật **Process Injection** (Bơm mã độc vào một tiến trình sạch như `notepad.exe` để trốn Antivirus).
* Nếu file Import các hàm: `RegSetValueEx`, `CreateService` $\rightarrow$ File có hành vi **Persistence** (Cắm rễ, tự động khởi động cùng Windows).

### 5. Dịch ngược mã nguồn (Disassembly & Decompiler)
Ném file vào **IDA Pro**, **Ghidra**, hoặc **Binary Ninja** để:
* Biến mã máy `010101` thành mã **Assembly (Disassembly)**.
* Dịch ngược thành **mã giả C (Decompiler)** giúp đọc hiểu logic của code nhanh gấp 10 lần.

# 2. Phân tích Động (Dynamic Analysis) - Theo dõi hành vi Runtime

## 2.1 Định nghĩa trực quan
Phân tích Động là quá trình kích hoạt cho file chạy thực tế trong một môi trường được cô lập và giám sát chặt chẽ để trực tiếp quan sát các hành vi thời gian thực (Runtime) của nó trên thanh RAM.

> 💡 **Ví dụ ví von:** Bạn quyết định mang quả bom vào một phòng thí nghiệm kiên cố, kích nổ nó và dùng camera siêu tốc để ghi lại xem nó nổ mạnh thế nào, mảnh vỡ văng đi đâu, sinh ra khí độc gì.

---

## 2.2 Các kỹ thuật phân tích động thực chiến

### 1. Giám sát hệ thống (System & Process Monitoring)
Cho file chạy và dùng các công cụ chuyên dụng để "quay phim" lại toàn bộ hành vi:

* **Process Monitor (ProcMon):** Xem file có tự lén lút tạo ra file mới (`CreateFile`), sửa Registry để tự khởi động (`RegSetValue`), hay spawn ra tiến trình con độc hại không (Ví dụ: `malware.exe` sinh ra `cmd.exe`).
* **Process Explorer:** Xem file đang ẩn náu dưới thân xác của tiến trình hệ thống nào (Ví dụ: `svchost.exe`, `explorer.exe`).

### 2. Phân tích mạng (Network Analysis)
Sử dụng **Wireshark** hoặc `tcpdump` để bắt toàn bộ các gói tin (Traffic) phát ra. Quan sát các yêu cầu DNS request, các kết nối TCP/HTTP để tìm ra máy chủ điều khiển của Hacker (**C2 Server - Command and Control Server**).

### 3. Gỡ lỗi chuyên sâu (Debugging)
Sử dụng các trình Debugger như **x64dbg** (Windows) hoặc **GDB-Pwndbg** (Linux) để hoàn toàn làm chủ dòng chảy của file:

* **Chạy từng bước (Single-stepping):** Bao gồm `Step Into` (đi vào trong hàm) và `Step Over` (chạy qua hàm).
* **Giám sát vùng nhớ:** Soi giá trị thay đổi liên tục của các thanh ghi (`EAX`, `EBX`, `RSP`...) và cấu trúc vùng nhớ Stack/Heap.
* **Sử dụng Điểm dừng (Breakpoint):**
    * **Software Breakpoint (`INT3` - `0xCC`):** Ghi đè mã máy để ép chương trình dừng lại tại câu lệnh ta muốn soi.
    * **Hardware Breakpoint:** Đặt bẫy trực tiếp trên các thanh ghi đặc biệt của CPU (`DR0`-`DR3`) để theo dõi khi nào có một hàm cố tình *Đọc/Ghi/Thực thi* trên một ô nhớ cụ thể.
    * **Conditional Breakpoint:** Chỉ dừng chương trình khi thỏa mãn điều kiện logic cụ thể (Ví dụ: Dừng lại khi vòng lặp chạy đến lần thứ `0x1337`).

### 4. Phân tích cuộc gọi hệ thống (System Call / API Analysis)
* **Trên Linux:** Dùng lệnh `strace ./program` để tóm gọn toàn bộ các hàm System Call mà file gọi xuống Nhân Kernel (Như `open()`, `read()`, `connect()`).
* **Trên Windows:** Dùng **API Monitor** để bắt sống các hàm API của hệ điều hành trước khi nó được thực thi.

### 5. Phân tích bộ nhớ (Memory Analysis)
Sử dụng công cụ **Volatility** để dump và phân tích RAM. 

> 🎯 **Tư duy RE nâng cao:** Kỹ thuật này cực kỳ lợi hại vì có những loại mã độc tinh vi chỉ tồn tại trên RAM (**Fileless Malware**). Trên đĩa cứng (Disk), file trông có vẻ hoàn toàn sạch (`clean.exe`), nhưng khi chạy lên RAM, đoạn code thực tế đã bị giải mã, sửa đổi hoặc hoán đổi hoàn toàn (**Modified code / Process Hollowing**).

# 3. Cuộc chiến phá giải Packing và Obfuscation

Tác giả bài Lab CTF hoặc các lập trình viên viết Malware sẽ không bao giờ để bạn phân tích một cách dễ dàng. Họ sẽ luôn sử dụng các kỹ thuật chống phá để làm khó bạn:

---

## 3.1 Packing (Nén bảo vệ) & Cách hóa giải
Sử dụng các công cụ như **UPX**, **Themida**, **VMProtect**... để nén/mã hóa toàn bộ code thật lại và chèn thêm một đoạn code mồi (**Stub**).

* 🔴 **Hậu quả với Static Analysis:** Khi mở file bằng IDA Pro/Ghidra, bạn chỉ thấy code của đoạn Stub (vài chục dòng thô sơ). Code thực tế đã bị giấu kín hoàn toàn, khiến việc phân tích tĩnh trở nên vô dụng.
* 🟢 **Hóa giải bằng Dynamic Analysis:** Khi bạn bấm cho file chạy trong Debugger (x64dbg), đoạn code Stub bắt buộc phải làm nhiệm vụ giải nén và nạp code thật lên RAM để CPU thực thi. Ta chỉ cần "rình" lúc nó giải nén xong tại điểm **OEP (Original Entry Point)**, sau đó tiến hành **Dump Memory** vùng nhớ đó ra đĩa cứng là thu được file sạch chứa mã nguồn ban đầu.

---

## 3.2 Kỹ thuật chống gỡ lỗi (Anti-Analysis)
Malware sẽ liên tục thăm dò môi trường xung quanh để xem mình có đang bị "mổ xẻ" hay không:

* **Anti-Debug:** Mã độc gọi các hàm hệ thống như `IsDebuggerPresent()`, `CheckRemoteDebuggerPresent()`, hoặc check trực tiếp cấu trúc `PEB (Process Environment Block)` để kiểm tra xem có bị ai debug không. Nếu phát hiện, nó sẽ lập tức tự sát hoặc rẽ hướng sang một nhánh code vô hại nhằm đánh lừa bạn.
* **Anti-VM:** Kiểm tra xem có đang chạy trong môi trường máy ảo (**VMware**, **VirtualBox**) không bằng cách quét tên Driver màn hình, các tiến trình đặc trưng (như `vmtoolsd.exe`), hoặc đếm số nhân CPU, dung lượng RAM (thường máy ảo của Analyst sẽ để cấu hình thấp).

> 🎯 **Nghệ thuật của RE (The Power of Patching):** 
> Sự kết hợp hoàn hảo giữa Tĩnh và Động! Ta dùng phân tích Tĩnh (IDA Pro) để tìm ra vị trí các hàm check `IsDebuggerPresent()`. Sau đó, thực hiện **Patch byte** — sửa các câu lệnh nhảy điều kiện (`JZ`/`JNZ`) hoặc ép cấu trúc kiểm tra thành câu lệnh vô thưởng vô phạt `NOP` (No Operation). Bằng cách này, ta triệt hạ hoàn toàn các cơ chế phòng thủ của mã độc trước khi bấm nút Debug thực tế!

# 4. Bảng so sánh toàn diện: Static vs. Dynamic Analysis

| Tiêu chí so sánh | Phân tích Tĩnh (Static Analysis) | Phân tích Động (Dynamic Analysis) |
| :--- | :--- | :--- |
| **Kích hoạt chạy file** | ❌ **KHÔNG** (Giữ file nằm im trên ổ cứng) | ✅ **CÓ** (Cho file thực thi trực tiếp) |
| **Mức độ an toàn** | 🟢 **An toàn tuyệt đối** (Không sợ nhiễm độc hệ thống) | 🟡 **Rủi ro cao** (Cần cách ly nghiêm ngặt trong VM/Sandbox) |
| **Tầm nhìn (Coverage)** | 🌐 **Toàn cảnh (Global View):** Thấy mọi nhánh rẽ, điều kiện ẩn, hàm ẩn của code. | 🔎 **Cục bộ (Local View):** Chỉ thấy được những nhánh code thực tế đang được kích hoạt chạy. |
| **Đối phó file bị Pack** | 🔴 **Rất khó khăn** (Chỉ thấy code bị mã hóa, obf hoặc code rác của Stub) | 🟢 **Dễ dàng hơn** (Chỉ cần đợi file tự unpack, giải mã chính nó lên RAM) |
| **Thời gian thực hiện** | ⏳ **Tốn thời gian:** Đòi hỏi kiến thức Assembly sâu và tư duy kiên trì. | ⚡ **Nhanh chóng:** Thấy ngay kết quả hành vi (File sinh ra, IP kết nối) trong vài phút. |

# 5. Quy trình làm việc chuẩn (Standard Workflow)
🚩 Trong giải đấu CTF Reverse Engineering:

``` Plaintext
[strings / DIE] ──> [Kiểm tra Imports] ──> [IDA Pro / Ghidra] ──> [Tìm hàm main / WinMain] ──> [Phân tích logic hàm] ──> [Đặt Breakpoint Debug] ──> [Lấy Flag]
```

☣️ Trong phân tích Mã độc chuyên nghiệp (Malware Threat Hunting):

```Plaintext
[Nhận mẫu Sample] ──> [Tính Hash SHA256] ──> [Quét Static cơ bản] ──> [Thả vào Sandbox tự động] ──> [Advanced Dynamic Debugging] ──> [Dump RAM / Khôi phục IAT] ──> [Trích xuất IOC / Viết báo cáo] bản edit git
```
## 🛠️ Kịch bản phá giải: Mã hóa chuỗi + Chống Debug
Plaintext
[ BƯỚC 1: STATIC ] ──> Định vị hàm Anti-debug (Dựa vào PE/ELF Imports hoặc String thô)
       │
       ▼
[ BƯỚC 2: STATIC ] ──> Patch Byte hàm check (Biến câu lệnh nhảy check thành NOP / JMP)
       │
       ▼
[ BƯỚC 3: DYNAMIC] ──> Ném file đã Patch vào Debugger an toàn ──> Điểm dừng (OEP / Hàm giải mã)
       │
       ▼
[ BƯỚC 4: DYNAMIC] ──> Rình lúc hàm Unpack/Decrypt chuỗi chạy xong ──> Tóm sống chuỗi sạch trên RAM

### Bước 1: Phân tích Tĩnh (Static) để tìm "Hàng rào phòng thủ"
Bạn không thể ném ngay file vào Debugger (Dynamic) vì dính Anti-debug, file sẽ lập tức tự sát hoặc chuyển sang nhánh code rác. Phân tích tĩnh lúc này đóng vai trò "trinh sát":

Quét bảng Import Table của file. Nếu là Windows, hãy tìm các hàm như IsDebuggerPresent, CheckRemoteDebuggerPresent, hoặc NtQueryInformationProcess. Nếu là Linux, chú ý các hàm đọc file /proc/self/status (trường TracerPid).

Đưa file vào IDA Pro để định vị chính xác địa chỉ của các hàm kiểm tra này.

### Bước 2: Triệt hạ Anti-debugging bằng kỹ thuật Patch Byte (Static)
Khi đã biết đoạn code check Debugger nằm ở đâu trong IDA, ta tiến hành "phẫu thuật" nó:

Thay vì để câu lệnh rẽ nhánh kiểm tra (ví dụ: JZ - nhảy nếu là debugger), ta dùng tính năng Edit -> Patch program -> Assemble để sửa câu lệnh đó thành NOP (No Operation - không làm gì cả, cho trôi qua luôn) hoặc biến thành lệnh JMP cố định sang nhánh code sạch.

Xuất (Export) file đã vá này ra thành một bản mới (ví dụ: malware_patched.exe). Lúc này, "nọc độc" chống gỡ lỗi của file đã bị bẻ gãy hoàn toàn.

### Bước 3: Phân tích Động (Dynamic) để bắt file "Tự khai"
Bây giờ, do file malware_patched.exe đã mất khả năng nhận biết debugger, ta hoàn toàn có thể tự tin mở nó bằng x64dbg hoặc GDB-Pwndbg:

Vì các chuỗi (String) đã bị mã hóa tĩnh, ta không thể đọc được bằng lệnh strings thông thường. Tuy nhiên, quy luật bất biến của máy tính là: CPU không thể hiểu được dữ liệu đã mã hóa, muốn thực thi hay sử dụng, chương trình bắt buộc phải giải mã chuỗi đó về dạng Text thuần túy (Plaintext) trên RAM.

Đặt một điểm dừng (Breakpoint) ở ngay sau phân vùng giải mã chuỗi hoặc tại các hàm nhạy cảm mà ta nghi ngờ nó sẽ sử dụng chuỗi đó (ví dụ đặt breakpoint tại hàm InternetConnect hoặc ShellExecute).

### Bước 4: Tóm sống Payload và chuỗi sạch (Dynamic Memory Dump)
Bấm F9 cho chương trình chạy. Khi chương trình bị khựng lại tại Breakpoint của bạn, hãy nhìn ngay xuống cửa sổ Dump (Memory View) hoặc vùng nhớ Stack.

Toàn bộ các chuỗi URL bí mật, IP của C2 Server, hay các lệnh hệ thống lúc này đã được hàm giải mã của malware "dâng sẵn" lên RAM ở dạng không che giấu. Bạn chỉ cần Copy ra hoặc dùng tính năng Dump Memory của Debugger là thu hồi được toàn bộ dữ liệu sạch.

> 🥊 Lời kết tư duy:
> - Phân tích Tĩnh (Static) giúp ta vượt qua hàng rào bảo vệ (Anti-debug) một cách an toàn.
> - Phân tích Động (Dynamic) giúp ta ép file tự động làm công việc rã đông, giải mã dữ liệu (Decrypt String) mà không cần tốn hàng tuần trời ngồi tự giải thuật toán mã hóa bằng tay.

## 6. Bảng tổng hợp bộ công cụ "Gối đầu giường" của dân ATTT

Dưới đây là các vũ khí cốt lõi được phân chia theo hai trường phái Phân tích Tĩnh (Static) và Phân tích Động (Dynamic) mà một Reverse Engineer/Malware Analyst bắt buộc phải nằm lòng.

| Phân nhóm | Tên Công Cụ | Hệ điều hành | Vai trò cốt lõi trong phân tích | Mẹo Thực Chiến |
| :--- | :--- | :--- | :--- | :--- |
| **Static** | **IDA Pro / Ghidra** | Windows / Linux | Trình dịch ngược mã máy thành Assembly và mã giả C hàng đầu. | Sử dụng phím tắt `F5` trong IDA để xem mã giả C (Decompiler) giúp đọc hiểu logic nhanh gấp 10 lần. |
| **Static** | **Detect It Easy (DIE)** | Windows / Linux / macOS | Kiểm tra Compiler, Kiến trúc file (x86/x64) và phát hiện các loại Packer. | Hãy luôn check độ hỗn loạn của entropy (Entropy value). Nếu entropy tiến gần mức `8.0`, file chắc chắn đã bị Pack hoặc mã hóa. |
| **Static** | **PEStudio / CFF Explorer** | Windows | Phân tích sâu các cấu trúc Header, Section, bảng Import/Export của file PE. | PEStudio tự động gắn cờ cảnh báo (Blacklist) các hàm API nguy hiểm và các chuỗi khả nghi mà không cần chạy file. |
| **Static** | **readelf / objdump** | Linux | Trích xuất siêu dữ liệu, bảng Program Header, Section Header của file định dạng ELF. | Dùng `objdump -d <file>` để nhanh chóng disassemble một hàm cụ thể trên môi trường terminal Linux mà không cần mở giao diện lớn. |
| **Dynamic** | **x64dbg / WinDbg** | Windows | Trình Debugger tối cao ở chế độ User-mode (x64dbg) và Kernel-mode (WinDbg). | Trong x64dbg, sử dụng tab `Symbols` để tìm nhanh các hàm API hệ thống, sau đó chuột phải đặt Breakpoint hàng loạt. |
| **Dynamic** | **GDB + Pwndbg** | Linux | Trình gỡ lỗi tiêu chuẩn vàng kết hợp extension chuyên dụng để khai thác nhị phân (Pwnable). | Lệnh `context` của Pwndbg cho phép quan sát đồng thời các thanh ghi, stack, code disassembly và các hàm backtrace tại mỗi bước nhảy. |
| **Dynamic** | **Process Monitor (ProcMon)** | Windows | Giám sát thời gian thực mọi hành vi ghi file, sửa Registry, tạo tiến trình con. | Sử dụng tính năng `Filter (Ctrl+L)` để lọc chính xác tên tiến trình (`Process Name is malware.exe`), loại bỏ nhiễu từ hệ thống. |
| **Dynamic** | **Wireshark** | Windows / Linux / macOS | Bắt và phân tích các gói tin mạng thời gian thực, lật tẩy kết nối của hacker. | Sử dụng bộ lọc `http.request.method == "POST"` hoặc `dns` để tóm sống gói tin gửi dữ liệu đánh cắp về C2 Server. |
| **Dynamic** | **Volatility** | Windows / Linux / macOS | Phân tích, mổ xẻ và điều tra chứng cứ số sâu bên trong file Dump RAM bộ nhớ. | Sử dụng lệnh `malfind` của Volatility để quét và phát hiện các phân vùng bộ nhớ có thuộc tính thực thi bất thường (PAGE_EXECUTE_READWRITE) - dấu vết của Process Injection. |

> 📌 **Nguyên tắc chọn vũ khí:** 
> * Khi nhận được một file nhị phân mới, quy trình luôn bắt đầu bằng nhóm công cụ **Static** (`DIE` -> `PEStudio/readelf` -> `IDA Pro`) để nắm thóp cấu trúc.
> * Chỉ khi đã cô lập hoàn toàn môi trường máy ảo an toàn, ta mới kích hoạt nhóm công cụ **Dynamic** (`ProcMon` -> `x64dbg/GDB` -> `Wireshark`) để tóm sống hành vi thực tế.
