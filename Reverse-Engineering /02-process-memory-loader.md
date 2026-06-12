# Process - Memory - Loader 

Thực tế, CPU không chạy trực tiếp file `.exe` hay file ELF trên đĩa cứng. 

Quá trình diễn ra như sau:

File trên đĩa (PE/ELF) ──> Loader (OS) ──> Khởi tạo Process ──> Áp xạ vào RAM (Memory Layout) ──> CPU thực thi

---

## 1. Phân biệt Program, Process và Thread

Để không bị nhầm lẫn khi phân tích, chúng ta cần phân biệt rõ 3 khái niệm nền tảng này:

### 🔹 Program (Chương trình)
Program là file tĩnh nằm im trên ổ cứng (Disk). 
* *Ví dụ:* Windows là `chall.exe`, Linux là `./chall`.
* Khi chưa chạy, nó chỉ là một file chứa: Header, Code, Data, danh sách các hàm Import, các Sections/Segments và Metadata.

### 🔹 Process (Tiến trình)
Process là một "phiên bản" (Instance) đang chạy của một chương trình. 
* Một file chương trình nằm im trên ổ cứng (như `notepad.exe`) chỉ là một file tĩnh. Nhưng khi ta double-click vào nó 3 lần, hệ điều hành (OS) sẽ tạo ra **3 Process riêng biệt** chạy song song với 3 mã định danh **PID (Process ID)** khác nhau, mặc dù chúng chung một file gốc.
* Có thể hiểu đơn giản: `Process = Không gian địa chỉ ảo riêng + Tài nguyên OS + Ít nhất một Thread`.
* Mỗi Process sẽ sở hữu độc lập: loaded executable, loaded DLL/SO, vùng nhớ Heap, Stack của các thread, các biến môi trường (environment variables), lệnh dòng lệnh (command line) và các quản lý tài nguyên (handles/file descriptors).

### 🔹 Thread (Luồng)
Thread mới là **luồng thực thi mã máy thật sự**. 
* `Process` đóng vai trò chứa tài nguyên và quản lý bộ nhớ, còn `Thread` mới là thứ cầm tay chỉ việc cho CPU chạy từng Instruction.
* Một Thread sở hữu riêng: Con trỏ lệnh (`EIP/RIP`), một vùng nhớ Stack riêng biệt và một ngữ cảnh thanh ghi riêng (Register Context).

> 💡 **Ghi chú cho RE:** Một process có thể có nhiều thread. Khi bạn dùng các công cụ như `x64dbg` hoặc `GDB`, bạn sẽ **Attach** (gắn) trình debugger vào một **Process đang chạy**. Hãy cẩn thận vì Malware rất hay tạo thêm các Thread phụ ngầm (như chạy payload, watchdog, keylogger hoặc network beacon) để đánh lừa analyst nếu chúng ta chỉ chăm chăm debug Main Thread.

---

## 2. Virtual Memory (Bộ nhớ ảo) và Khái niệm Page

Một sai lầm kinh điển là nghĩ chương trình ghi dữ liệu trực tiếp vào thanh RAM vật lý. Nếu vậy, hai chương trình ghi trùng vào một địa chỉ RAM sẽ làm crash máy nhau ngay lập tức. Để giải quyết, OS sử dụng **Virtual Memory** (Địa chỉ ảo).

Mỗi Process khi sinh ra đều "ảo tưởng" rằng mình đang sở hữu toàn bộ bộ nhớ của máy tính:
* **Kiến trúc 32-bit:** Thấy không gian địa chỉ từ `0x00000000` đến `0xFFFFFFFF` (4GB).
* **Kiến trúc 64-bit:** Thấy vùng không gian khổng lồ từ `0x0000000000000000` đến `0xFFFFFFFFFFFFFFFF`.

Các địa chỉ bạn nhìn thấy trong Debugger hay IDA (Ví dụ: `0x401000`, `0x7ffdf000`) đều là địa chỉ ảo bên trong Process đó.

* **Cơ chế hoạt động:** Chip **MMU (Memory Management Unit)** của CPU kết hợp với OS sẽ dịch địa chỉ ảo của từng thằng ra các vị trí vật lý (Physical Address) khác nhau trên thanh RAM thật. Ví dụ: Process A và Process B đều có một biến ở địa chỉ ảo `0x401000`, nhưng thực tế trên RAM chúng nằm ở hai nơi hoàn toàn khác nhau.
* **Khái niệm Page:** Bộ nhớ ảo không được quản lý theo từng byte đơn lẻ mà quản lý theo từng khối gọi là Page. Kích thước Page phổ biến nhất hiện nay là **`0x1000` byte (tương đương 4096 bytes)**.

---

## 3. Phân quyền bộ nhớ (Memory Permissions)

Mỗi vùng bộ nhớ (Page) trong Process đều được Hệ điều hành gán cho các quyền truy cập ngặt nghèo để bảo vệ an toàn hệ thống:

| Quyền | Ký hiệu | Ý nghĩa |
| :--- | :---: | :--- |
| **Read** | R | Cho phép đọc dữ liệu/đọc cấu trúc. |
| **Write** | W | Cho phép ghi hoặc sửa đổi dữ liệu. |
| **Execute** | X | Cho phép CPU thực thi (nhảy vào chạy mã máy). |

### Các tổ hợp quyền thực tế:
* **`RX` (Read/Execute):** Code chương trình bình thường (phân vùng `.text`) luôn có quyền này để CPU đọc và chạy, nhưng cấm sửa đổi code lúc đang chạy.
* **`RW` (Read/Write):** Dữ liệu thông thường, Heap, Stack sẽ có quyền này (cho ghi biến, ghi dữ liệu nhưng cấm thực thi để chống chạy mã độc).
* **`RWX` (Read/Write/Execute):** Quyền tối cao (Đọc/Ghi/Chạy). Rất hiếm khi xuất hiện ở chương trình bình thường trừ khi dùng cơ chế JIT Compiler hoặc các bộ Packer.

> 🎯 **Pattern RE / Malware quan trọng:** Các vùng nhớ có quyền `RWX` hoặc có hành vi chuyển quyền từ `RW -> RX` cực kỳ đáng nghi. Quy trình "thả độc" kinh điển của Packer/Malware:
> $$\text{Cấp phát vùng nhớ quyền RW} \longrightarrow \text{Ghi/Decrypt Payload vào đó} \longrightarrow \text{Đổi quyền vùng đó sang RX hoặc RWX} \longrightarrow \text{Call/Jmp vào chạy}$$

---

## 4. Memory Layout (Cấu trúc bộ nhớ của một Process)

Khi một chương trình được nạp vào bộ nhớ ảo, nó được chia thành các phân vùng rõ rệt từ địa chỉ thấp (Low Address) đến địa chỉ cao (High Address):

```text
High Address +-------------------------+
             | Stack (Phát triển xuống) |  <-- Quản lý biến cục bộ, tham số hàm, return address (Quyền: RW)
             +-------------------------+
             |            ↓            |
             |                         |
             |            ↑            |
             +-------------------------+
             | Heap  (Phát triển lên)  |  <-- Vùng bộ nhớ động dùng malloc/new (Quyền: RW)
             +-------------------------+
             | .bss  (Biến chưa khởi tạo)|  <-- Biến toàn cục chưa gán giá trị, mặc định = 0 (Quyền: RW)
             +-------------------------+
             | .data (Biến đã khởi tạo) |  <-- Biến toàn cục đã gán giá trị trước (Quyền: RW)
             +-------------------------+
             | .rdata / .rodata        |  <-- Chứa chuỗi hằng, dữ liệu chỉ đọc (Quyền: R)
             +-------------------------+
             | .text (Mã máy / Code)   |  <-- Chứa instruction mã máy để CPU đọc chạy (Quyền: RX)
Low Address  +-------------------------+
```
### Ví dụ minh họa trực quan bằng code C/C++:
```C
char global_buf[32];              // Biến toàn cục chưa khởi tạo -> Nằm ở phân vùng: .bss
char global_str[] = "hello";      // Biến toàn cục đã khởi tạo -> Nằm ở phân vùng: .data
const char *s = "secret";         // Chuỗi ký tự cố định -> Nằm ở phân vùng: .rdata / .rodata

int main() {
    char stack_buf[32];            // Biến cục bộ nằm trong hàm -> Nằm trên: Stack
    char *heap_buf = malloc(32);   // Xin cấp phát động -> Vùng nhớ 32 bytes nằm dưới: Heap
}
```
> ⚠️ Mẹo RE từ Strings Window: Khi mở Strings Window trong IDA, phần lớn các chuỗi nhạy cảm như "Wrong Password", "Keygen", "Correct"... đều được lôi ra từ phân vùng dữ liệu chỉ đọc .rdata / .rodata này.
## 5. Loader (Trình nạp) hoạt động như thế nào?

**Loader** là một thành phần cốt lõi của Hệ điều hành. Nhiệm vụ của nó là làm "kiến trúc sư" dựng file tĩnh từ đĩa cứng thành một Process sống động trên RAM.

### ⚙️ Flow tổng quát của Loader khi đưa một file executable từ disk vào memory:

```text
File Executable (Nằm trên Disk)
       │
       ▼
[OS Loader đọc cấu trúc Header] ───> Xem PE Header (Windows) hoặc ELF Header (Linux)
       │
       ▼
[Map Sections/Segments] ─────────> Ánh xạ các phân vùng (.text, .data...) vào Virtual Memory
       │
       ▼
[Cấp quyền bộ nhớ phù hợp] ──────> Gán quyền truy cập tương ứng (R, W, X, RX, RW...)
       │
       ▼
[Load thư viện phụ thuộc] ───────> Nạp các file thư viện dùng chung (.dll hoặc .so) vào RAM
       │
       ▼
[Resolve Imports/Symbols] ───────> Tìm và ghi địa chỉ thật của các hàm hệ thống (IAT hoặc GOT/PLT)
       │
       ▼
[Apply Relocations] ─────────────> Tính toán, sửa lại các địa chỉ tuyệt đối nếu Base Address thay đổi (ASLR)
       │
       ▼
[Chạy mã khởi tạo sớm] ──────────> Thực thi TLS Callbacks (Windows) hoặc .init_array (Linux) nếu có
       │
       ▼
[Nhảy tới Entry Point] ──────────> Bàn giao quyền điều khiển cho CPU nhảy vào thực thi điểm đầu tiên
       │
       ▼
[Runtime Startup] ───────────────> Thực hiện các thủ tục setup môi trường của ngôn ngữ
       │
       ▼
       Hàm main() chính thức được gọi!
```
> 🛑 ĐIỂM CỰC KỲ QUAN TRỌNG DÂN RE PHẢI NHỚ: Hàm main() KHÔNG PHẢI là thứ đầu tiên được chạy trong một chương trình! Trước main đã có một hàng dài code của Loader và Runtime Startup thực thi.
## 6. Điểm đặc trưng chuyên sâu giữa Windows Loader và Linux Loader

### 🪟 Trên môi trường Windows (Định dạng PE)

* **Resolve Import Address Table (IAT):** Một file thực thi thông thường không chứa code của các hàm hệ thống như `CreateFileA`, `ReadFile` hay `VirtualAlloc`. Nó chỉ chứa **cái tên** của hàm đó. Khi chạy, Windows Loader phải đi tìm các hàm đó trong các file thư viện tương ứng (như `kernel32.dll`) và ghi địa chỉ thật lúc runtime vào một bảng gọi là **IAT (Import Address Table)**.
  > 💡 **Mẹo RE từ Imports Window:** Nhìn vào cửa sổ **Imports** trong IDA giúp ta đoán được đến 70% hành vi của một file lạ. Ví dụ nếu thấy import các hàm: `InternetOpenA`, `CreateFileW`, `RegOpenKeyExW`, ta biết ngay con malware này có hành vi kết nối mạng, tạo file và can thiệp vào registry.
* **TLS Callback (Thread Local Storage Callback):** Đây là các hàm đặc biệt được Windows Loader thực thi **TRƯỚC CẢ ENTRY POINT / MAIN**. Các bộ bảo vệ (Anti-Cheat, Packer) hoặc Malware rất hay dùng TLS Callback để chạy payload sớm, phá hoại hoặc check anti-debug. Nếu analyst chủ quan và chỉ biết đặt breakpoint ở `main`, mã độc sẽ chạy xong xuôi và lẩn trốn từ trước đó.
* **DllMain:** Đối với các file thư viện dùng chung (`.dll`), khi được Loader nạp vào bộ nhớ, Windows có thể gọi ngay hàm khởi tạo:
  ```c
  DllMain(..., DLL_PROCESS_ATTACH, ...)
  ```
  -> Nhờ vậy, DLL có thể tự động thực thi code ngay khi vừa được load vào RAM mà không cần đợi bất kỳ một export function nào được gọi trực tiếp từ chương trình chính.
### 🐧 Trên môi trường Linux (Định dạng ELF)

* **Cơ chế Relocations / GOT / PLT:** Tương tự như Windows, các file ELF trên Linux cũng sử dụng liên kết động (Dynamic Linking) cho các hàm thư viện như printf() hay puts(). Khi chương trình gọi một hàm, ví dụ call puts@plt, nó sẽ nhảy vào bảng PLT (Procedure Linkage Table), sau đó tra cứu địa chỉ thực của hàm trong thư viện libc.so thông qua bảng GOT (Global Offset Table) do dynamic linker (ld-linux) quản lý để thực thi.
* **Entry Point thực tế không phải là main:** Chương trình C/C++ trên Linux khi dịch sang mã máy sẽ chạy theo luồng:
  ```Plaintext
  _start ──> __libc_start_main ──> main
  ```
  -> Vì vậy trong IDA/Ghidra, nếu thấy điểm dừng mặc định (Entry Point) ở _start thì đó chưa phải là logic chính của bài lab/challenge, đó mới chỉ là đoạn code khởi tạo (Runtime Startup Code) của hệ thống mà thôi.

* **.init_array (Constructors):** Đây là phân vùng chứa danh sách các hàm khởi tạo được Linux tự động chạy trước khi hàm main được gọi. Phân vùng này có vai trò và hành vi tương tự như TLS Callback bên Windows, là nơi lý tưởng để giấu mã độc chạy sớm.

* **Nạp thư viện động thủ công bằng code:** Ngoài việc để Loader tự động nạp lúc khởi động, lập trình viên (hoặc malware) có thể tự load thêm shared library (.so) ngay trong lúc chạy (Runtime) bằng các hàm API:
  - dlopen: Nạp file thư viện .so vào bộ nhớ.
  - dlsym: Tìm kiếm và trả về địa chỉ của một symbol/hàm cụ thể trong thư viện đó.
  *(Cơ chế này tương đương hoàn toàn với cặp hàm LoadLibrary / GetProcAddress bên Windows).*
## 7. Địa chỉ ảo VA, RVA và File Offset (Chìa khóa nối liền Static và Dynamic)

Đây là phần cực kỳ quan trọng khi bạn cần kết hợp giữa phân tích tĩnh (Static Analysis - đọc file trong IDA) và phân tích động (Dynamic Analysis - khi debug chương trình đang chạy). Nếu không phân biệt được các loại địa chỉ này, bạn sẽ bị loạn khi đối chiếu dữ liệu giữa các công cụ.

### 📁 File Offset (Vị trí trong file)
File offset là vị trí byte chính xác tính từ đầu file khi nó đang nằm im trên ổ cứng (Disk). 
* *Ví dụ:* Khi bạn dùng Hex Editor (như HxD) mở file lên và thấy một chuỗi nằm ở `offset 0x400`, đó chính là File Offset.

### 📍 VA (Virtual Address - Địa chỉ ảo tuyệt đối)
VA là địa chỉ của code hoặc dữ liệu khi file đã được Loader ánh xạ (map) vào không gian bộ nhớ ảo lúc chương trình đang chạy.
* *Ví dụ:* Địa chỉ bạn nhìn thấy khi đặt breakpoint trong các trình Debugger như x64dbg hay GDB (như `0x401000`, `0x140001000`) đều là địa chỉ VA.

### 📐 RVA (Relative Virtual Address - Địa chỉ ảo tương đối)
RVA là khoảng cách chênh lệch (offset) tính từ địa chỉ ảo cần tìm cho tới địa chỉ gốc (`ImageBase`) của chương trình trong bộ nhớ.
* Công thức cốt lõi:
$$\text{RVA} = \text{VA} - \text{ImageBase}$$

**Ví dụ minh họa cụ thể:**
Giả sử một file PE chương trình có địa chỉ gốc được nạp vào bộ nhớ là `ImageBase = 0x400000`.
* Nếu một hàm có địa chỉ ảo tuyệt đối lúc chạy là $\text{VA} = \text{0x401000}$.
* Thì địa chỉ ảo tương đối của hàm đó sẽ là: $\text{RVA} = \text{0x401000} - \text{0x400000} = \text{0x1000}$.

---

Trong thực tế, do cấu trúc căn lề và phân bổ dung lượng (Alignment) của file khi nằm trên đĩa (thường là từng block `0x200` byte) và khi được Loader bung ra trên RAM (theo từng Page `0x1000` byte) hoàn toàn khác nhau, dẫn đến một quy luật xương máu:
$$\text{File Offset} \neq \text{VA}$$

Địa chỉ nhìn thấy trong IDA hay Debugger không bao giờ trùng khớp hoàn toàn với vị trí byte trên ổ đĩa. Ta bắt buộc phải hiểu và áp dụng linh hoạt các khái niệm này trong các trường hợp:

1. **Hex Patch file:** Khi ta tìm ra đoạn code cần sửa (ví dụ đổi lệnh `JZ` thành `JMP` để crack bypass) lúc đang Debug ở địa chỉ VA, ta không thể lấy nguyên địa chỉ VA đó áp vào Hex Editor để sửa file trên đĩa được. Ta phải tìm xem địa chỉ VA đó thuộc Section nào, tính RVA rồi quy đổi ngược về File Offset thì mới patch đúng chỗ.
2. **Xử lý ASLR:** Khi cơ chế bảo mật ASLR được bật, địa chỉ tĩnh trong IDA và địa chỉ thực tế khi Debug sẽ khác nhau hoàn toàn sau mỗi lần chạy. Cách duy nhất để tìm đúng hàm mục tiêu lúc debug là lấy `Module Base thực tế lúc chạy + RVA tìm được từ IDA`.
3. **Dump Memory:** Khi chương trình chạy và tự giải mã (Unpack) một file PE/ELF khác nằm ẩn trong bộ nhớ, bạn cần xác định đúng địa chỉ VA bắt đầu và kết thúc của vùng nhớ đó, tính toán kích thước dựa trên cấu trúc Section để tiến hành Dump phân vùng bộ nhớ đó ra thành file hoàn chỉnh.

## 8. ASLR và Cơ chế Relocation

Khi phân tích một file nhị phân, ta sẽ gặp trường hợp địa chỉ hiển thị trong công cụ phân tích tĩnh (IDA/Ghidra) không khớp với địa chỉ thực tế khi chương trình đang chạy trong Debugger. Hiện tượng này được quyết định bởi hai yếu tố: **ASLR** và **Relocation**.

### 🎲 ASLR (Address Space Layout Randomization) là gì?
ASLR là một cơ chế bảo mật của Hệ điều hành nhằm ngẫu nhiên hóa sơ đồ bộ nhớ của Process. Mỗi lần chương trình được chạy, Loader sẽ chọn một địa chỉ gốc (`Module Base Address` hay `ImageBase`) ngẫu nhiên để nạp chương trình vào RAM, thay vì dùng một địa chỉ cố định.

* **Mục đích:** Chống lại các cuộc tấn công khai thác bộ nhớ (Exploit) như Buffer Overflow. Nếu hacker không biết chính xác code của hàm hệ thống nằm ở địa chỉ nào, họ không thể xây dựng chuỗi payload (như ROP chain) để thực thi mã độc một cách cố định được.

**Hệ quả đối với RE:**
* Giả sử trong IDA (Static Analysis), ta thấy một hàm nhạy cảm nằm ở địa chỉ tuyệt đối là:

  $$\text{VA}_{\text{IDA}} = \text{0x140001000}$$
  
  *(Do IDA đang giả định chương trình được nạp ở địa chỉ gốc mặc định ImageBase(IDA)= 0x140000000 )*
  
* Khi ta bật Debugger (Dynamic Analysis) lên, ta tìm mãi không thấy địa chỉ `0x140001000` đâu. Thực tế, lúc này Loader đã nạp chương trình ở một địa chỉ gốc hoàn toàn khác, ví dụ:

  $$\text{ImageBase}_{\text{Runtime}} = \text{0x7ff612340000}$$

### 🛠️ Giải pháp: Sử dụng RVA để ánh xạ địa chỉ
Khi chương trình bật ASLR, ta **không bao giờ** được sử dụng địa chỉ VA tuyệt đối cứng nhắc. Thay vào đó, ta phải tính toán dựa trên địa chỉ ảo tương đối (RVA):

1. **Bước 1:** Lấy địa chỉ ảo trong IDA trừ đi ImageBase mặc định của IDA để tìm ra khoảng lệch RVA:
   $$\text{RVA} = \text{0x140001000} - \text{0x140000000} = \text{0x1000}$$
2. **Bước 2:** Lấy Module Base thực tế thu được trong Debugger cộng với RVA vừa tính để tìm ra địa chỉ Runtime chính xác:
   $$\text{VA}_{\text{Runtime}} = \text{0x7ff612340000} + \text{0x1000} = \text{0x7ff612341000}$$

Đặt breakpoint tại địa chỉ `0x7ff612341000` trong Debugger, ta sẽ bắt đúng hàm mục tiêu cần phân tích.

---

### 🗂️ Cơ chế Base Relocation là gì?

Khi trình biên dịch (Compiler) tạo ra file, trong code sẽ có những câu lệnh sử dụng địa chỉ tuyệt đối (Hardcoded Address). 

* *Ví dụ:* Lệnh di chuyển dữ liệu `mov rax, [0x140005020]` yêu cầu CPU tìm chính xác dữ liệu tại địa chỉ `0x140005020`.

Tuy nhiên, nếu ASLR ngẫu nhiên hóa và đẩy chương trình sang nạp ở Base mới là `0x7ff612340000`, thì địa chỉ `0x140005020` kia sẽ hoàn toàn vô nghĩa hoặc trỏ vào vùng nhớ rác, khiến chương trình bị crash ngay lập tức. Để giải quyết việc này, file thực thi phải chứa một bảng đặc biệt gọi là **Relocation Table** (Bảng tái định vị địa chỉ).

#### Quy trình xử lý Relocation của Loader:
1. **Đọc bảng:** Khi nạp file vào RAM ở một Base ngẫu nhiên, Loader sẽ tìm đến phân vùng chứa thông tin Relocation (Section `.reloc` trên Windows).
2. **Tính Delta (Khoảng chênh lệch):** Loader tính toán xem địa chỉ Base thực tế lệch bao nhiêu so với Base mặc định ban đầu:
   $$\Delta = \text{ImageBase}_{\text{Thực tế}} - \text{ImageBase}_{\text{Mặc định}}$$
3. **Sửa code (Patching):** Loader duyệt qua danh sách tất cả các vị trí chứa địa chỉ tuyệt đối được ghi nhận trong bảng Relocation, lấy địa chỉ cũ cộng thêm giá trị $\Delta$ để cập nhật thành địa chỉ mới hợp lệ trước khi cho chương trình chạy.

> 💡 **Ý nghĩa trong RE:** Việc hiểu Relocation giúp ích rất nhiều khi ta thực hiện kỹ thuật **Dump Memory**. Nếu bạn dump một payload từ RAM ra đĩa để phân tích tĩnh, file đó có thể không chạy lại được vì các địa chỉ tuyệt đối đã bị Loader sửa đổi mất rồi. Bạn sẽ cần fix lại bảng Relocation (Fix Reloc) thì file mới có thể thực thi độc lập.

## 9. Các hàm API quản lý Memory cần nhận diện (Dấu hiệu nhận biết Malware)

Khi thọc một file lạ vào các công cụ phân tích, việc soi danh sách các hàm API hệ thống mà chương trình sử dụng (thông qua bảng Imports) là bước đi khôn ngoan nhất. Nếu thấy xuất hiện các hàm quản lý bộ nhớ dưới đây, bạn cần đưa chúng vào tầm ngắm ngay lập tức vì chúng là những "vũ khí" kinh điển của Malware và Packer.

### 📌 Trên môi trường Windows (Win32 APIs)

#### 1. Các hàm quản lý bộ nhớ trong cùng Tiến trình (Local Memory)
* **`VirtualAlloc`:** Hàm dùng để xin Hệ điều hành cấp phát một vùng nhớ ảo mới trong không gian bộ nhớ của chính chương trình đó.
  > ⚠️ **Dấu hiệu đáng nghi:** Nếu bạn thấy tham số phân quyền của hàm này được set là `PAGE_EXECUTE_READWRITE` (`0x40`), nghĩa là chương trình đang xin tạo một vùng nhớ **`RWX`** (vừa cho Ghi dữ liệu vừa cho Chạy code). Đây là một pattern cực kỳ nhạy cảm, thường dùng để chứa mã độc (Shellcode).

* **`VirtualProtect`:** Hàm dùng để thay đổi quyền truy cập (Permission) của một vùng nhớ đã có sẵn.
  > 🎯 **Pattern Unpacking / Decryption:** Malware rất hay dùng hàm này theo kịch bản: Ban đầu cấp vùng nhớ `RW` để ghi/giải mã payload, sau đó gọi `VirtualProtect` để đổi quyền sang `RX` (Read/Execute) rồi nhảy vào thực thi nhằm né tránh cơ chế bảo mật DEP/NX của OS.

* **`CreateFileMapping` + `MapViewOfFile`:** Cặp hàm dùng để ánh xạ một file từ đĩa cứng trực tiếp thành một vùng bộ nhớ ảo. Khác với `ReadFile` (phải copy bytes dữ liệu vào một mảng buffer), cơ chế này map file thành bộ nhớ để CPU đọc/ghi trực tiếp. Malware thường dùng cách này để tự parse cấu trúc file PE hoặc map thủ công một file payload khác thay cho Loader của Windows.

#### 2. Các hàm tương tác liên Tiến trình (Process Injection APIs)
Nếu bạn thấy một file thực thi gọi đồng thời chuỗi hàm dưới đây, có đến 100% khả năng nó đang thực hiện kỹ thuật **Process Injection** (Tiêm mã độc) hoặc **Process Hollowing** (Chiếm đoạt tiến trình) vào một ứng dụng hợp pháp (như `notepad.exe`, `explorer.exe`) để lẩn trốn:

* **`OpenProcess`:** Mở quyền truy cập vào một tiến trình mục tiêu khác đang chạy trên hệ thống dựa vào PID.

* **`VirtualAllocEx`:** Chữ `Ex` (Extended) đại diện cho việc hàm này có thể can thiệp sang tiến trình khác. Hàm này giúp malware tự ý cấp phát một vùng nhớ trống bên trong không gian bộ nhớ của tiến trình mục tiêu.

* **`WriteProcessMemory`:** Ghi trực tiếp dữ liệu (mã độc/Shellcode) từ tiến trình hiện tại vào vùng nhớ vừa được cấp phát của tiến trình mục tiêu.

* **`CreateRemoteThread`:** Tạo và kích hoạt một Thread mới chạy ngầm bên trong tiến trình mục tiêu, ép Thread đó nhảy vào địa chỉ chứa Shellcode vừa ghi để thực thi.

* **`SetThreadContext` + `ResumeThread`:** Cặp hàm dùng trong kỹ thuật *Process Hollowing*. Malware sẽ tạo một tiến trình hợp pháp ở trạng thái đóng băng (`CREATE_SUSPENDED`), "gột rỗng" ruột code của nó, dùng `WriteProcessMemory` nạp payload mới vào, rồi dùng `SetThreadContext` sửa thanh ghi `EIP/RIP` trỏ về payload mới và gọi `ResumeThread` để kích hoạt.

---

### 📌 Trên môi trường Linux (Syscalls / APIs)

* **`mmap`:** Hàm tối cao trên Linux dùng để tạo một vùng ánh xạ bộ nhớ ảo mới (Có vai trò tương đương hoàn toàn với `VirtualAlloc` bên Windows). Khi phân tích, bạn cần đặc biệt kiểm tra các cờ phân quyền (Protection Flags) như `PROT_READ | PROT_WRITE | PROT_EXEC` (vùng nhớ quyền `RWX`).

* **`mprotect`:** Thay đổi quyền truy cập của một vùng nhớ ảo đã có sẵn (Tương đương `VirtualProtect` bên Windows). Đây là hàm mấu chốt cần đặt breakpoint để bắt sống thời điểm mã độc tự unpack và đổi quyền sang `RX` để thực thi code.

* **`brk` / `sbrk`:** Các hàm hệ thống cấp thấp dùng để thay đổi kích thước của phân vùng Heap truyền thống.

* **`dlopen` + `dlsym`:** Cặp hàm dùng để nạp một file shared library (`.so`) vào bộ nhớ và tìm kiếm địa chỉ của một hàm cụ thể ngay trong lúc chương trình đang chạy. Kỹ thuật nạp thư viện động (Runtime Dynamic Linking) này giúp malware ẩn giấu danh sách các hàm nhạy cảm, không cho chúng xuất hiện trong bảng Imports tĩnh.

* **`ptrace`:** Hàm hệ thống tối cao dùng để theo dõi, kiểm soát và thao túng bộ nhớ/thanh ghi của một tiến trình khác (Đây chính là lõi công cụ của các trình Debugger như GDB).
  > 🛡️ **Mẹo Anti-Debugging:** Malware Linux rất hay tự gọi lệnh `ptrace(PTRACE_TRACEME, 0, 1, 0)`. Theo quy tắc của Linux, một tiến trình chỉ có thể bị bám đuôi (trace) bởi duy nhất một thực thể tại một thời điểm. Khi malware tự "bám" lấy chính nó, nếu bạn bật GDB để debug, GDB sẽ bị từ chối truy cập và báo lỗi ngay lập tức.

## 10. Cách quan sát và giám sát bộ nhớ Runtime (Thực hành thực tế)

Khi phân tích động (Dynamic Analysis), việc chỉ nhìn vào từng dòng lệnh Assembly đơn lẻ là chưa đủ. Bạn cần có một cái nhìn tổng quan từ trên xuống (Top-down view) thông qua **Sơ đồ bộ nhớ (Memory Map)** để biết chính xác cấu trúc bộ nhớ của Process đang thay đổi như thế nào tại thời điểm đó.

---

### 🐧 1. Thao tác trên môi trường Linux

Để xem cấu trúc phân bổ bộ nhớ thực tế của một chương trình đang chạy trên Linux, bạn có thể sử dụng các lệnh hệ thống sau:

#### Xem trực tiếp từ Terminal:
```bash
# Chạy chương trình ở chế độ ngầm (Background) để lấy PID
./chall &

# Đọc file maps trong thư mục /proc của tiến trình đó
cat /proc/<PID>/maps

# Hoặc sử dụng lệnh pmap để hiển thị sạch đẹp, chia cột rõ ràng
pmap -x <PID>
```
Kết quả trả về từ `/proc/<PID>/maps` sẽ có dạng như sau:

```Plaintext
Vùng địa chỉ ảo       Phân quyền   Offset     Đường dẫn file/Phân vùng
555555554000-555555555000 r--p     00000000   /home/user/chall  <-- Main Binary Header
555555555000-555555556000 r-xp     00001000   /home/user/chall  <-- [Code Segment / .text]
555555758000-555555779000 rw-p     00000000   [heap]            <-- Vùng nhớ Heap
7ffffffde000-7ffffffff000 rw-p     00000000   [stack]           <-- Vùng nhớ Stack
7ffff7a0d000-7ffff7bcd000 r-xp     00000000   /lib/libc.so      <-- Thư viện dùng chung Libc
```
##### Các lệnh điều hướng và bẫy bộ nhớ nâng cao trong GDB:

Khi đang debug bằng GDB, thay vì chạy lệnh mù quáng, hãy sử dụng các lệnh chuyên sâu sau để kiểm soát bộ nhớ:

* **info proc mappings** -> Xem sơ đồ phân vùng bộ nhớ của tiến trình hiện tại ngay trong lòng GDB (Tương đương lệnh `cat /proc/.../maps`).
  
* **x/20gx $rsp** -> Khảo sát dữ liệu trên Stack: Xem 20 hàng dữ liệu dạng hex (64-bit) tính từ đỉnh Stack (Thanh ghi RSP).
  
* **x/s 0xĐịa_Chỉ_Ảo** -> Đọc chuỗi ký tự (String) đang ẩn giấu tại một địa chỉ cụ thể trên RAM.

* **catch syscall mmap**  -> Đặt bẫy đóng băng chương trình ngay khi nó chuẩn bị gọi hệ thống mmap để xin cấp phát bộ nhớ.

* **catch syscall mprotect** -> Đặt bẫy dừng chương trình khi nó gọi mprotect để đổi quyền bộ nhớ (Mẹo tóm sống thời điểm Malware đổi quyền vùng nhớ sang RX để chạy Payload).

### 🪟 2. Thao tác trên môi trường Windows

Trên Windows, chúng ta sử dụng kết hợp giữa trình **Debugger (x64dbg / WinDbg)** và bộ công cụ huyền thoại **Sysinternals Suite** để giám sát Runtime:

#### Các công cụ giám sát trực quan:
1. Tab Memory Map (trong x64dbg): Đây là cửa sổ tối quan trọng. Tại đây, bạn có thể nhìn thấy toàn bộ Module Base của file .exe chính, danh sách các file DLL được nạp đi kèm, và đặc biệt là cột Protection (Quyền bảo vệ: ER, RW, RWX).

   **🔍 Mẹo bắt Payload:** Nếu thấy xuất hiện một vùng nhớ dạng Private Alloc (vùng nhớ tự xin cấp phát độc lập, không liên kết với file .dll hay .exe nào trên đĩa) mà lại mang quyền RWX hoặc RX, hãy click chuột phải và chọn Dump vùng đó ra. Khả năng cao Payload thật của Malware sau khi gột rỗng/giải mã đang nằm tại đây.

2. VMMap (Sysinternals): Công cụ chuyên dụng để trực quan hóa bộ nhớ của một Process cụ thể. Nó phân loại rõ ràng phần bộ nhớ nào dành cho Image, phần nào cho Thread Stack, phần nào cho Heap để bạn dễ dàng theo dõi dung lượng biến động.

3. Process Explorer / Process Hacker / Process Monitor: Giúp giám sát danh sách các Handle tài nguyên (File, Registry) mà chương trình đang chiếm giữ, đồng thời theo dõi danh sách các Thread phụ đang chạy ngầm để phát hiện hành vi tiêm nhiễm mã độc.

#### Danh sách Breakpoint API bắt buộc phải đặt khi Debug chương trình lạ:

Để tóm sống hành vi can thiệp bộ nhớ hoặc tiêm mã độc của chương trình ngay khi nó vừa thực thi, hãy đặt sẵn breakpoint ở các hàm API Windows nhạy cảm sau:

* **Hàm cấp phát/đổi quyền:** `VirtualAlloc`, `VirtualProtect` (Bắt hành vi tự unpack/decrypt code).

* **Hàm can thiệp file/thư viện:** `LoadLibraryA/W`, `GetProcAddress`, `CreateFileMapping`, `MapViewOfFile`.

* **Hàm Injection/Hollowing:** `VirtualAllocEx`, `WriteProcessMemory`, `CreateRemoteThread`, `SetThreadContext`, `ResumeThread`.
## 11. Các Pattern RE / Malware quan trọng

Khi phân tích một chương trình thực tế, thay vì đọc từng dòng lệnh rải rác, việc nhận diện ra các **Pattern** (khuôn mẫu hành vi) dưới đây sẽ giúp bạn hiểu ngay ý đồ chiến thuật của tác giả mã độc hoặc bộ bảo vệ.

---

### 📦 Pattern 1: Unpacking (Tự giải mã bộ nhớ)
Dùng khi binary gốc đã bị nén hoặc mã hóa (Packed) nhằm chống phân tích tĩnh. Khi chạy, nó sẽ tự "bung" code thật ra RAM.

*   **Dấu hiệu nhận biết:** Static analysis (IDA) chỉ thấy một đoạn code rất ngắn (Stub code), các phân vùng có tên lạ hoặc Strings Window không có thông tin gì nhạy cảm.
*   **Flow hành vi thực tế:**
```text
Windows: VirtualAlloc (Quyền RW) ──> Ghi/Decrypt Payload ──> VirtualProtect (Đổi sang RX) ──> Call/Jmp vào Payload
Linux  : mmap (Quyền RW)         ──> Ghi/Decrypt Payload ──> mprotect (Đổi sang RX)       ──> Call/Jmp vào Payload
```
-> Ý nghĩa RE: Bạn cần đặt breakpoint tại hàm VirtualProtect (Windows) hoặc mprotect (Linux) để bắt sống thời điểm Payload thật vừa được giải mã sạch sẽ trên Memory trước khi nó thực thi.

### 💉 Pattern 2: Process Injection (Tiêm mã độc liên tiến trình)
Malware không tự chạy hành vi phá hoại dưới tên của nó mà "gửi ké" code độc vào một tiến trình hợp pháp khác đang chạy (như notepad.exe, svchost.exe) để qua mặt Task Manager và Antivirus.

* **Flow hành vi thực tế (Windows):**

  ```Plaintext
  OpenProcess (Mở tiến trình nạn nhân)
       │
       ▼
  VirtualAllocEx (Cấp phát vùng nhớ trống TRONG tiến trình nạn nhân)
       │
       ▼
  WriteProcessMemory (Ghi mã độc/Shellcode từ tiến trình mình sang tiến trình nạn nhân)
       │
       ▼
  CreateRemoteThread (Kích hoạt một Thread mới bên trong nạn nhân để chạy vùng mã độc đó)
  ```

### 💀 Pattern 3: Process Hollowing (Chiếm đoạt / Gột rỗng tiến trình)

Đây là kỹ thuật nâng cao của Injection. Thay vì gửi ké một đoạn code nhỏ, malware sẽ tạo hẳn một tiến trình hợp pháp mới nhưng "hút ruột" và thay thế bằng toàn bộ file thực thi độc hại của nó.

* **Flow hành vi thực tế (Windows):**

  ```Plaintext
  CreateProcess (Bật tiến trình hợp pháp ở trạng thái đóng băng: CREATE_SUSPENDED)
       │
       ▼
  NtUnmapViewOfSection (Gột rỗng, xóa sạch toàn bộ vùng nhớ code gốc của tiến trình đó)
       │
       ▼
  VirtualAllocEx + WriteProcessMemory (Bơm toàn bộ file thực thi độc hại mới vào vùng nhớ trống)
       │
       ▼
  SetThreadContext (Sửa thanh ghi EIP/RIP của tiến trình đang bị đóng băng trỏ về Entry Point mới)
       │
       ▼
  ResumeThread (Kích hoạt cho tiến trình chạy tiếp với diện mạo hợp pháp nhưng ruột độc hại)
  ```
  
### 🛡️ Pattern 4: Anti-Debug trên Linux

Malware sử dụng các đặc trưng quản lý tiến trình của nhân Linux để phát hiện và ngăn chặn các Analyst bám đuôi gỡ lỗi.

* **Flow hành vi thực tế:**

  - **Sử dụng Ptrace:** Gọi hàm hệ thống `ptrace(PTRACE_TRACEME, ...)`. Do Linux chỉ cho phép một thực thể trace tiến trình tại một thời điểm, việc tự trace chính mình sẽ khiến GDB bị block khi cố tình attach vào.

  - **Kiểm tra trạng thái hệ thống:** Chương trình tự đọc file cấu trúc hệ thống của chính nó tại đường dẫn `/proc/self/status`. Nếu nó parse dữ liệu và phát hiện trường `TracerPid` có giá trị khác 0 (nghĩa là đang có một tiến trình khác như GDB hay ltrace bám theo), nó sẽ tự động crash hoặc rẽ hướng sang luồng code giả để đánh lừa Analyst.

### ⏳ Pattern 5: Pre-main Execution (Thực thi mã sớm trước hàm main)

Kỹ thuật đánh lừa các Analyst nghiệp dư – những người có thói quen chỉ nhảy vào đặt breakpoint tại hàm main().

* Dấu hiệu nhận biết: Mọi hành vi độc hại, kết nối mạng hoặc anti-debug đã diễn ra và kết thúc thành công trước khi Debugger kịp dừng lại ở câu lệnh đầu tiên của hàm main.

* Cơ chế thực hiện:

  - Trên Windows: Tác giả giấu code trong các hàm TLS Callback (Thread Local Storage) hoặc viết code phá hoại trực tiếp bên trong hàm khởi tạo `DllMain` của các file DLL đi kèm khi cờ `DLL_PROCESS_ATTACH` được kích hoạt.

  - Trên Linux: Tác giả cấu trúc code rơi vào phân vùng `.init_array` hoặc khai báo các hàm dưới dạng `__attribute__((constructor))`. Loader của OS sẽ duyệt qua và ép CPU thực thi toàn bộ các hàm này trước khi bàn giao quyền điều khiển cho hàm `main()`.
## 12. Quy trình phân tích bộ nhớ Runtime cơ bản

Khi đối mặt với một file thực thi lạ trong các giải CTF hoặc khi phân tích mã độc, đừng lao vào đọc code một cách mù quáng. Hãy tuân thủ quy trình 9 bước tiêu chuẩn sau để kiểm soát bộ nhớ tiến trình:

```text
[BƯỚC 1]: Xác định định dạng file (PE hay ELF, kiến trúc x86 hay x64) bằng lệnh `file` hoặc PE-bear.
   │
   ▼
[BƯỚC 2]: Kiểm tra Entry Point ảo trong IDA và tìm xem có hàm main thực tế không.
   │
   ▼
[BƯỚC 3]: Chạy thử chương trình trong môi trường ảo an toàn (Sandbox / VM cách ly).
   │
   ▼
[BƯỚC 4]: Mở Memory Map lên để quan sát toàn cảnh kiến trúc các phân vùng RAM lúc vừa chạy.
   │
   ▼
[BƯỚC 5]: Xác định rõ các tọa độ: Phân vùng code nằm ở đâu? Heap ở đâu? Stack ở đâu? Thư viện nằm ở đâu?
   │
   ▼
[BƯỚC 6]: Đặt Breakpoint tại hàm main hoặc tại các hàm API quản lý bộ nhớ quan trọng (VirtualAlloc, mmap).
   │
   ▼
[BƯỚC 7]: Theo dõi sát sao sự biến động của dung lượng và quyền truy cập (Protection) nếu nghi ngờ file bị nén (Packed).
   │
   ▼
[BƯỚC 8]: Áp dụng công thức RVA để ánh xạ địa chỉ động (Runtime) ngược về địa chỉ tĩnh trong IDA.
   │
   ▼
[BƯỚC 9]: Ghi lại bản đồ phân vùng bộ nhớ thu được để phục vụ cho việc viết script khai thác (Exploit) hoặc viết báo cáo.
```
- Tư duy khi làm CTF: Tập trung theo dõi luồng dữ liệu của người dùng dịch chuyển thế nào: `Input từ bàn phím ──> Nạp vào Stack/Heap ──> Đi qua hàm Check ──> Quyết định Success/Fail`.

- Tư duy khi phân tích Malware: `Tập trung theo dõi hành vi tương tác hệ thống: Entry Point ──> Tự Unpack/Giải mã bộ nhớ ──> Gọi các API ẩn ──> Thực hiện hành vi chiếm đoạt (Injection/Hollowing) ──> Thiết lập kết nối mạng`.
## 13. Bảng tra cứu nhanh (Cheatsheet) Lệnh và Công cụ

Học RE/Pwn bắt buộc phải đi đôi với thực hành. Dưới đây là bảng tổng hợp toàn bộ các công cụ và hệ thống câu lệnh cốt lõi mà bạn sẽ phải sử dụng liên tục khi làm các bài Lab hoặc phân tích bộ nhớ tiến trình:

### 🐧 1. Môi trường Linux (Định dạng file ELF)

#### Bảng lệnh khảo sát file tĩnh và động hệ thống:
| Câu lệnh | Mục đích sử dụng chuyên sâu |
| :--- | :--- |
| `file chall` | Kiểm tra nhanh định dạng file (ELF 32-bit/64-bit), kiến trúc CPU và xem file có bị xóa biểu đồ ký tự (`stripped`) hay không. |
| `readelf -h chall` | Đọc thông tin ELF Header để tìm địa chỉ điểm khởi đầu của chương trình (`Entry point address`). |
| `readelf -l chall` | Xem danh sách các Program Headers. Giúp nhận diện các phân đoạn `PT_LOAD` sẽ được Loader ánh xạ trực tiếp vào RAM. |
| `readelf -S chall` | Xem danh sách chi tiết các Sections trong file tĩnh trên đĩa (`.text`, `.data`, `.bss`, `.rodata`). |
| `checksec --file chall` | Kiểm tra các cơ chế bảo mật bộ nhớ được cấu hình trên file (NX/DEP, ASLR/PIE, Stack Canary, RELRO). |
| `cat /proc/<PID>/maps` | Xem sơ đồ Memory Layout thực tế với đầy đủ quyền truy cập (`rwx`) của tiến trình mục tiêu đang chạy. |
| `pmap -x <PID>` | Hiển thị chi tiết dung lượng phân bổ bộ nhớ của tiến trình theo dạng bảng phân cột sạch đẹp. |

#### Hệ thống câu lệnh bẫy bộ nhớ thần chú trong GDB:
Khi đang debug một bài Lab bằng GDB, thay vì gõ lệnh một cách mù quáng, hãy sử dụng các bộ lệnh chuyên sâu về bộ nhớ sau:

```text
(gdb) set disassembly-flavor intel       # Chuyển cú pháp hiển thị ASM sang dạng Intel cho dễ đọc
(gdb) break main                          # Đặt breakpoint tại hàm main
(gdb) run                                 # Kích hoạt cho chương trình chạy
(gdb) info proc mappings                  # Xem trực tiếp sơ đồ phân vùng bộ nhớ (Memory Map) lúc runtime
(gdb) x/20gx $rsp                         # Khảo sát Stack: Xem 20 hàng dữ liệu (64-bit Hex) tính từ đỉnh Stack (RSP)
(gdb) x/s 0xĐịa_Chỉ                       # Đọc chuỗi ký tự (String) đang ẩn giấu tại một tọa độ bộ nhớ cụ thể
(gdb) catch syscall mmap                  # Đặt bẫy đóng băng chương trình ngay khi nó gọi mmap để xin cấp phát bộ nhớ
(gdb) catch syscall mprotect              # Đặt bẫy dừng chương trình khi nó gọi mprotect để thay đổi quyền bộ nhớ
```
### 🪟 2. Môi trường Windows (Định dạng file PE)

Trên Windows, chúng ta sẽ phối hợp nhịp nhàng giữa các công cụ phân tích cấu trúc file tĩnh và các phần mềm giám sát RAM lúc Runtime:

| Tên công cụ | Phân loại | Vai trò cốt lõi trong phân tích bộ nhớ |
| :--- | :--- | :--- |
| **x64dbg / WinDbg** | Dynamic Analysis | Trình gỡ lỗi động. Sử dụng tab **Memory Map** để theo dõi Base Address thực tế của các Module, danh sách DLL được nạp và tìm kiếm các vùng nhớ trống tự cấp phát (`Private Allocation`) mang quyền `RWX` lạ. |
| **PE-bear / PEStudio**| Static Analysis | Công cụ phân tích cấu trúc file PE tĩnh trên đĩa cứng. Giúp kiểm tra độ lệch cấu trúc Section, soi bảng Imports và phát hiện sớm các dấu hiệu file đã bị Pack/Mã hóa. |
| **VMMap (Sysinternals)**| Monitor Tool | Công cụ trực quan hóa bộ nhớ đỉnh cao. Phân loại chi tiết và tô màu trực quan các vùng không gian bộ nhớ dành cho Image, Thread Stack, Private Data hay Heap. |
| **Process Explorer** | Monitor Tool | Giám sát danh sách các Handle tài nguyên (File, Registry) mà tiến trình đang chiếm giữ, đồng thời theo dõi sự dịch chuyển và tạo mới của các luồng Thread ngầm. |

#### 🎯 Các API Windows nhạy cảm cần đặt sẵn Breakpoint khi Debug:

Để tóm sống hành vi can thiệp bộ nhớ hoặc tiêm mã độc của một file lạ, hãy cấu hình đặt sẵn breakpoint ở các hàm sau:

* **Cấp phát & Đổi quyền RAM:** `VirtualAlloc`, `VirtualProtect`, `VirtualAllocEx`, `VirtualProtectEx`.
* **Thao túng/Tiêm nhiễm tiến trình:** `OpenProcess`, `WriteProcessMemory`, `CreateRemoteThread`, `SetThreadContext`, `ResumeThread`.
* **Nạp file / Ánh xạ thư viện:** `LoadLibraryA/W`, `GetProcAddress`, `CreateFileMapping`, `MapViewOfFile`.
## 14. Tổng hợp những sai lầm chí mạng của người mới học RE / Pwn

Để không tự biến mình thành một "gà mờ" khi debug và phân tích bộ nhớ, hãy luôn ghi nhớ và soi chiếu bản thân qua 7 lỗi kinh điển sau:

* **❌ Sai lầm 1: Nghĩ Process chỉ là Code**
    > **Thực tế:** Process là một thực thể lớn, đóng vai trò như một "thùng chứa" bao gồm: Không gian địa chỉ ảo riêng + Tài nguyên hệ thống được OS cấp phát + Các luồng Thread thực thi. Mã máy (Code) chỉ là một phần nhỏ nằm gọn trong phân vùng `.text` mà thôi.
* **❌ Sai lầm 2: Nghĩ Entry Point luôn là hàm `main()`**
    > **Thực tế:** Hoàn toàn sai! Entry Point mặc định của file thực thi luôn trỏ vào đoạn code khởi tạo hệ thống (mã `_start` trên Linux hoặc Runtime Startup Code trên Windows). Hàm `main()` thực tế nằm ở phía sau đó khá xa và chỉ được gọi sau khi môi trường đã setup xong.
* **❌ Sai lầm 3: Không phân biệt được File Offset và VA**
    > **Thực tế:** Do quy luật $\text{File Offset} \neq \text{VA}$, việc lấy địa chỉ ảo tuyệt đối (VA) tìm được trong các trình Debugger để đập thẳng vào Hex Editor nhằm vá file (Patch file) trên đĩa cứng sẽ làm hỏng cấu trúc file ngay lập tức. Bạn bắt buộc phải tính toán qua RVA để quy đổi về vị trí trên đĩa.
* **❌ Sai lầm 4: Quên mất sự tồn tại của ASLR**
    > **Thực tế:** Rất nhiều bạn ngồi thắc mắc tại sao địa chỉ trong IDA khác hoàn toàn địa chỉ thực tế lúc Debug. Hãy nhớ khi ASLR bật, Base Address sẽ bị xáo trộn ngẫu nhiên sau mỗi lần chạy. Bạn phải luôn tư duy và tìm hàm mục tiêu theo công thức: $\text{Địa chỉ Runtime} = \text{Module Base thực tế lúc debug} + \text{RVA}$.
* **❌ Sai lầm 5: Bỏ qua các đoạn code chạy trước `main()`**
    > **Thực tế:** Việc lười biếng, không thèm kiểm tra bảng **TLS Callback** (Windows) hoặc phân vùng **`.init_array`** (Linux) sẽ khiến bạn hoàn toàn "bất lực" trước các kỹ thuật Anti-Debug hoặc các hàm giải mã chạy sớm của mã độc.
* **❌ Sai lầm 6: Thấy phân quyền `RWX` là kết luận Malware ngay**
    > **Thực tế:** Đáng nghi thì có, nhưng chưa thể khẳng định 100%. Các chương trình sử dụng trình biên dịch JIT (Just-In-Time) như trình duyệt Web chạy JavaScript, máy ảo Java (JVM) hoặc các bộ Packer hợp pháp vẫn phải xin quyền `RWX` để tối ưu hiệu năng biên dịch và thực thi. Ta cần đánh giá dựa trên ngữ cảnh và hành vi cụ thể của hàm.
* **❌ Sai lầm 7: Chỉ nhìn phân tích tĩnh (Static), lười bật Memory Map**
    > **Thực tế:** Đối với các file bị mã hóa, nén hoặc hoán đổi code liên tục (Self-modifying code), phân tích tĩnh ban đầu trong IDA chỉ toàn thấy code rác hoặc stub code vô nghĩa. "Ruột" thực sự của bài toán hay Payload độc hại chỉ lộ diện rõ ràng nhất trên sơ đồ bộ nhớ (**Memory Map**) khi bạn chạy chương trình ở Runtime.
