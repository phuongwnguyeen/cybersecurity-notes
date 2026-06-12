# Assembly & Calling Convention

---

## 1. Assembly là gì?
**Assembly (ASM)** là ngôn ngữ lập trình bậc thấp, biểu diễn trực tiếp các lệnh mã máy của CPU dưới dạng các từ gợi nhớ (**Mnemonics**) để con người có thể đọc hiểu và phân tích.

### 🔄 So sánh C++ và Assembly (x86-64)

Khi bạn viết một hàm bằng C++:
```cpp
int add(int a, int b) {
    return a + b;
}
```
Khi Compiler dịch sang Assembly, nó sẽ trông như thế này:
```Code snippet
mov eax, edi    ; Nạp tham số thứ 1 (biến a nằm ở EDI) vào thanh ghi EAX
add eax, esi    ; Cộng tiếp tham số thứ 2 (biến b nằm ở ESI) vào EAX
ret             ; Trả về kết quả (mặc định nằm trong thanh ghi EAX)
```
## 2. Kiến Trúc Thanh Ghi (Registers)
Thanh ghi là các ô nhớ dung lượng cực nhỏ nhưng có **tốc độ truy cập siêu nhanh** nằm ngay bên trong CPU để xử lý dữ liệu tức thời.

Vai trò của thanh ghi:
- lưu dữ liệu đang được xử lý
- lưu địa chỉ ô nhớ
- lưu lệnh hoặc kết quả trung gian
- giúp CPU truy xuất và tính toán nhanh hơn nhiều so với RAM

### Phân biệt x86 (32-bit) và x86-64 (64-bit)
Hiện nay hầu hết các hệ thống mục tiêu đều chạy kiến trúc 64-bit. Tên thanh ghi sẽ thay đổi ký tự đầu (`E` cho 32-bit và `R` cho 64-bit).

| Thanh ghi 32-bit | Thanh ghi 64-bit | Vai trò chính trong hệ thống |
| :--- | :--- | :--- |
| **EAX** | **RAX** | Chứa giá trị trả về của hàm (**Return Value**) |
| **ESP** | **RSP** | Con trỏ quản lý đỉnh Stack (**Stack Pointer**) |
| **EBP** | **RBP** | Con trỏ cố định đáy Stack Frame (**Base Frame Pointer**) |
| **EIP** | **RIP** | Con trỏ chỉ vào lệnh tiếp theo sẽ thực thi (**Instruction Pointer**) |
| **ECX** | **RCX** | Thanh ghi đếm (**Counter**), dùng trong vòng lặp |
| **EDX** | **RDX** | Dữ liệu phụ, bổ trợ phép toán hoặc chứa tham số |
| **ESI** | **RSI** | Con trỏ nguồn cho các chuỗi/mảng (**Source Index**) |
| **EDI** | **RDI** | Con trỏ đích cho các chuỗi/mảng (**Destination Index**) |

> 📌 **Lưu ý quan trọng khi RE x64:** Kiến trúc x86-64 mở rộng thêm 8 thanh ghi đa năng được đánh số từ **R8 đến R15** (R8, R9, R10, R11, R12, R13, R14, R15).
## 3. Quản Lý Bộ Nhớ (Memory & Stack)

### 📥 Thao tác với RAM qua cặp dấu `[]`
Dấu ngoặc vuông `[]` trong Assembly tương đương với toán tử giải bọc con trỏ (`*ptr`) trong ngôn ngữ C/C++.
* `mov eax, 5` $\rightarrow$ `eax = 5;` *(Gán giá trị thuần túy vào thanh ghi)*
* `mov eax, [5]` $\rightarrow$ `eax = *(int*)5;` *(Tìm đến địa chỉ ô nhớ số 5 và lấy giá trị tại đó nạp vào eax)*

---

### 🥞 Vùng nhớ Stack (LIFO - Last In First Out)
Stack hoạt động theo cơ chế **vào sau cùng - ra đầu tiên** (giống như một chồng sách, quyển nào đặt lên cuối cùng sẽ phải lấy ra đầu tiên).

```text
    |           |
    |___________|
    |  Sách 3   |  <-- Vừa vào (Đỉnh Stack / Top)
    |___________|
    |  Sách 2   |
    |___________|
    |  Sách 1   |  <-- Vào đầu tiên (Đáy Stack)
    |___________|
```
📥 Các lệnh thao tác với Stack:
- `push eax`: Đẩy giá trị của thanh ghi `EAX` lên đỉnh Stack. Khi có dữ liệu mới đắp vào, vùng nhớ phình ra nên con trỏ `ESP/RSP` sẽ giảm xuống (vì Stack trong kiến trúc x86/x64 phình về phía địa chỉ thấp).

- `pop eax`: Lấy giá trị đang nằm ở đỉnh Stack nạp ngược lại vào thanh ghi `EAX`. Sau khi lấy ra, Stack thu nhỏ lại nên con trỏ `ESP/RSP` sẽ tăng lên.
## 4. Cấu Trúc Stack Frame (Khung Bộ Nhớ Của Hàm)

Mỗi khi một hàm được gọi (ví dụ: `foo(a, b)`), Compiler sẽ tự động dựng một cấu trúc **Stack Frame** để quản lý các biến cục bộ, lưu địa chỉ quay về và các tham số truyền vào.

### 🛠️ Cụm lệnh khởi tạo (Prologue)
Khi vừa bước vào một hàm, cấu trúc Frame sẽ được thiết lập bằng cụm lệnh sau:

```asm
push ebp          ; Lưu lại giá trị đáy Stack của hàm cũ (Old EBP)
mov ebp, esp      ; Lấy vị trí đỉnh Stack hiện tại làm đáy mới cho hàm này
sub esp, 0x20     ; Kéo đỉnh Stack xuống (trừ bớt bộ nhớ) để chừa chỗ cho biến cục bộ
```
### 📝 Layout trực quan của Stack Frame (Kiến trúc x86)
Đây là "bản đồ" bộ nhớ mà sẽ phải nhìn và phân tích hàng ngày khi làm **Reverse Engineering**:

```Plaintext
[ebp + 12]  -->  Tham số thứ 2 (b)
[ebp + 8]   -->  Tham số thứ 1 (a)
[ebp + 4]   -->  Return Address (Địa chỉ quay về của hàm gọi)
[ebp]       -->  Giá trị cố định của Old EBP
[ebp - 4]   -->  Biến cục bộ 1 (Local variable 1)
[ebp - 8]   -->  Biến cục bộ 2 (Local variable 2)  <-- ESP (Đỉnh Stack) nằm ở đây
```
> 💡 Mẹo nhớ nhanh khi RE: Cứ lấy EBP làm mốc: Các tham số truyền vào hàm sẽ nằm ở hướng cộng [ebp + X], còn các biến cục bộ tự sinh bên trong hàm sẽ nằm ở hướng trừ [ebp - X].
## 5. Các Tập Lệnh Cơ Bản Cần Thuộc Lòng

Đây là những lệnh Assembly xuất hiện với tần suất dày đặc trong mọi file thực thi. Khi RE, cần nhìn lệnh ASM và nảy số ngay sang tư duy ngôn ngữ C/C++.

### 📑 Bảng tra cứu lệnh cơ bản
| Lệnh ASM | Ý nghĩa bản chất | Minh họa bằng C/C++ |
| :--- | :--- | :--- |
| `mov a, b` | Copy dữ liệu từ nguồn `b` ghi đè vào đích `a` | `a = b;` |
| `lea eax, [ebp-8]` | **Load Effective Address:** Chỉ tính toán và lấy địa chỉ ô nhớ, KHÔNG lấy giá trị bên trong | `eax = &var;` |
| `add eax, 1` | Phép cộng tích lũy | `eax++;` hoặc `eax += 1;` |
| `sub eax, 1` | Phép trừ | `eax--;` hoặc `eax -= 1;` |
| `imul eax, 5` | Phép nhân có dấu | `eax *= 5;` |
| `xor eax, eax` | Phép toán logic XOR (Dùng để tối ưu clear thanh ghi về 0 nhanh hơn `mov`) | `eax = 0;` |
| `cmp eax, 5` | Thực hiện phép trừ ngầm (`eax - 5`) để kích hoạt các cờ trạng thái (Flags) | *(Lệnh nền để đưa ra quyết định nhảy)* |

---

### 🔀 Các lệnh nhảy có điều kiện (Jump Instructions)
Các lệnh nhảy này thường đứng ngay sau lệnh `cmp` hoặc `test` để điều hướng luồng thực thi của chương trình (Control Flow).

* **`jz` / `je` (Jump if Zero / Jump if Equal):** Nhảy nếu hai giá trị bằng nhau.
  ```asm
  cmp eax, 5
  je equal_label   ; Nếu eax == 5, nhảy đến equal_label
  ```
* **`jnz` / `jne` (Jump if Not Zero / Jump if Not Equal):** Nhảy nếu hai giá trị khác nhau.
* **`jl` (Jump if Less):** Nhảy nếu nhỏ hơn (Dành cho số có dấu <).
* **`jg` (Jump if Greater):** Nhảy nếu lớn hơn (Dành cho số có dấu >).
* **`jle` / `jge`:** Nhảy nếu nhỏ hơn hoặc bằng / lớn hơn hoặc bằng (<= / >=)
> 📌 Lưu ý: Đối với số không dấu (Unsigned), Compiler sẽ không dùng **jl/jg** mà dùng **jb (Jump if Below)** và **ja (Jump if Above)**.
## 6. Cơ Chế Gọi Hàm & Calling Convention

Quy ước gọi hàm (**Calling Convention**) là luật chung giữa hàm gọi (**Caller**) và hàm được gọi (**Callee**) về việc: truyền tham số qua đâu, thứ tự nào, và ai dọn dẹp Stack.

Một Calling Convention tiêu chuẩn sẽ định nghĩa 3 điều:
1. **Truyền tham số ở đâu:** Đẩy vào Stack (RAM) hay nạp vào các Thanh ghi (Registers)?
2. **Thứ tự truyền tham số:** Truyền từ Trái qua Phải hay từ Phải qua Trái?
3. **Ai dọn dẹp Stack:** Sau khi hàm chạy xong, **Caller** hay **Callee** sẽ là người hoàn trả con trỏ `ESP/RSP` về vị trí cũ?

---

### 🟦 1. cdecl (Mặc định trên Linux x86 32-bit).

* **Truyền tham số:** Đẩy vào Stack theo thứ tự **Từ Phải Qua Trái**.
* **Trình dọn dẹp Stack:** **Caller** (Hàm gọi) chịu trách nhiệm dọn dẹp Stack bằng lệnh `add esp`.

```asm
; Minh họa hàm add(5, 3) trong cdecl
push 3            ; Đẩy tham số thứ 2 vào trước (Phải qua Trái)
push 5            ; Đẩy tham số thứ 1 vào sau
call add          ; Gọi hàm add (mã máy đẩy Return Address lên Stack)
add esp, 8        ; Caller tự dọn Stack (2 tham số int = 8 bytes)
```
### 🟩 2. stdcall (Chuẩn Windows API x86 32-bit)

* **Truyền tham số:** Đẩy vào Stack theo thứ tự **Từ Phải Qua Trái** (giống cdecl).
* **Trình dọn dẹp Stack:** Callee (Hàm được gọi) tự dọn dẹp bộ nhớ Stack bằng lệnh `ret X` ở cuối hàm.

```asm
; --- Phía Caller ---
push 3            ; Đẩy tham số b
push 5            ; Đẩy tham số a
call add          ; Gọi hàm, không cần dọn dẹp sau đó

; --- Phía Callee (Cuối hàm add) ---
ret 8             ; Tự giải phóng 8 bytes của tham số trên Stack rồi mới nhảy về
```
### 🟨 3. fastcall (Tối ưu hóa tốc độ)
Thay vì tốn thời gian ghi/đọc dữ liệu trên ô nhớ Stack **(RAM)**, fastcall tận dụng tối đa tốc độ của CPU bằng cách đưa dữ liệu thẳng vào thanh ghi.

* **Truyền tham số:** 2 tham số đầu tiên được nạp thẳng vào thanh ghi ECX và EDX. Nếu có tham số thứ 3 trở đi mới đẩy vào Stack.
* **Trình dọn dẹp Stack:** Thường do Callee dọn dẹp (tương tự stdcall).

```asm
; Minh họa hàm add(5, 3) trong fastcall
mov ecx, 5        ; Tham số 1 nạp thẳng vào ECX
mov edx, 3        ; Tham số 2 nạp thẳng vào EDX
call add          ; Gọi hàm, truy cập cực nhanh vì không đụng tới RAM
```
### 🚀 4. System V AMD64 ABI (Chuẩn Linux x64 hiện nay)
Kiến trúc 64-bit có lợi thế vượt trội là sở hữu rất nhiều thanh ghi rộng (RAX, RBX, RDI, RSI,...). Do đó, các hệ điều hành hiện đại đã nâng cấp hoàn toàn cơ chế gọi hàm.

Trên Linux x64, **6 tham số đầu tiên** bắt buộc phải truyền qua các thanh ghi theo đúng thứ tự nghiêm ngặt sau:

* **Tham số 1** $\rightarrow$ Nạp vào thanh ghi **`RDI`**
* **Tham số 2** $\rightarrow$ Nạp vào thanh ghi **`RSI`**
* **Tham số 3** $\rightarrow$ Nạp vào thanh ghi **`RDX`**
* **Tham số 4** $\rightarrow$ Nạp vào thanh ghi **`RCX`**
* **Tham số 5** $\rightarrow$ Nạp vào thanh ghi **`R8`**
* **Tham số 6** $\rightarrow$ Nạp vào thanh ghi **`R9`**

> 💡 **Mẹo nhớ nhanh:** Hãy thuộc lòng câu thần chú thứ tự thanh ghi này khi làm RE x64: **`RDI` $\rightarrow$ `RSI` $\rightarrow$ `RDX` $\rightarrow$ `RCX` $\rightarrow$ `R8` $\rightarrow$ `R9`**.

> ⚠️ Nếu hàm có từ tham số thứ 7 trở đi, các tham số thừa đó mới được đẩy vào Stack theo thứ tự từ **Phải qua Trái**.

> Trình dọn dẹp Stack: Do **Caller** dọn dẹp (nếu có sử dụng Stack cho tham số thứ 7+).

```asm
; Minh họa hàm hiển thị foo(1, 2, 3, 4, 5, 6) trên Linux x64
mov rdi, 1        ; Arg 1
mov rsi, 2        ; Arg 2
mov rdx, 3        ; Arg 3
mov rcx, 4        ; Arg 4
mov r8, 5         ; Arg 5
mov r9, 6         ; Arg 6
call foo          ; Thực thi hàm
```
🧠 Tóm tắt tư duy
* **Trên Windows x86:** Thấy cuối hàm có dạng `ret 4`, `ret 8`, `ret 0xC` -> Kết luận ngay hàm dùng stdcall.
* **Trên Linux x64:** Thấy trước lệnh call có một chuỗi lệnh nạp dữ liệu vào `rdi`, `rsi`, `rdx` -> Định hình ngay trong đầu thứ tự các tham số truyền vào hàm để khôi phục lại mã nguồn C chuẩn xác.
## 7. Cách Biên Dịch Cấu Trúc Khối Lệnh Từ C++ Sang ASM
---
### 🔀 1. Cấu Trúc Điều Kiện (`If - Else`)

Compiler sẽ dùng lệnh `cmp` hoặc `test` để kiểm tra điều kiện, sau đó dùng các lệnh nhảy đảo ngược lại logic để bỏ qua khối lệnh nếu điều kiện sai.

#### 🟢 Khi viết bằng C++:
```cpp
if (x == 5) {
    print_success();
} else {
    print_failed();
}
```
#### 🔴 Mã máy Assembly sinh ra
```asm
cmp eax, 5        ; Giả sử biến x đang nằm trong thanh ghi EAX. Lấy EAX trừ 5 ngầm.
jne L_ELSE        ; Nếu KHÔNG BẰNG (eax != 5), nhảy ngay xuống nhãn L_ELSE
call print_success; Nếu BẰNG (eax == 5), chạy tiếp xuống đây để gọi hàm thành công
jmp L_END         ; Chạy xong hàm thành công thì phải nhảy qua nhóm Else để kết thúc

L_ELSE:
call print_failed ; Nơi thực thi khi điều kiện ban đầu bị sai (eax != 5)

L_END:
; ... Tiếp tục luồng chương trình chính ...
```
### 🔁 2. Vòng Lặp (For Loop / While Loop)

Vòng lặp trong Assembly được cấu thành từ 3 thành phần: Khởi tạo biến đếm $\rightarrow$ Kiểm tra điều kiện đầu mỗi vòng $\rightarrow$ Tăng/giảm biến đếm ở cuối vòng và nhảy ngược lên đầu.

#### 🟢 Khi viết bằng C++:
```cpp
for (int i = 0; i < 10; i++) {
    do_something();
}
```
#### 🔴 Mã máy Assembly sinh ra
```asm
mov ecx, 0         ; Khởi tạo biến đếm i = 0 (nạp vào thanh ghi ECX)

L_LOOP_START:
    cmp ecx, 10        ; So sánh biến đếm i (ECX) với số 10
    jge L_LOOP_END     ; Nếu i >= 10 (Lớn hơn hoặc bằng), thoát ngay vòng lặp
    
    ; --- Phân vùng thực thi Logic bên trong vòng lặp ---
    push ecx           ; Lưu lại ECX vào Stack đề phòng hàm do_something làm thay đổi nó
    call do_something  ; Thực thi hàm bên trong vòng lặp
    pop ecx            ; Khôi phục lại giá trị ECX sau khi gọi hàm xong
    
    inc ecx            ; Tăng biến đếm: i++ (Tương đương add ecx, 1)
    jmp L_LOOP_START   ; Quay ngược lại đầu vòng lặp để kiểm tra điều kiện tiếp

L_LOOP_END:
    ; ... Vòng lặp kết thúc, đi tiếp ...
```
### 📦 3. Mảng (Array) & Cấu Trúc (Struct)

CPU và Assembly hoàn toàn không có khái niệm về "tên thuộc tính" hay "chỉ số mảng [i]". Tất cả mọi thứ đều được quy về công thức:

$$\text{Địa chỉ phần tử} = \text{Địa chỉ gốc (Base Address)} + \text{Khoảng cách dịch chuyển (Offset)}$$

#### 🟢 Khi viết bằng C++:
```cpp
int arr[5];
arr[2] = 20;       // Gán phần tử thứ 3 của mảng bằng 20 (int chiếm 4 bytes)

struct Player {
    int id;        // Nằm ở đầu Struct (Offset +0)
    int hp;        // Cách đầu Struct 4 bytes (Offset +4)
};
Player p1;
p1.hp = 100;       // Gán thuộc tính hp bằng 100
```
#### 🔴 Mã máy Assembly sinh ra
``` asm
; --- Trường hợp xử lý MẢNG (Array) ---
mov eax, [ebp - 0x20]  ; Giả sử địa chỉ gốc (Base) của mảng arr nằm ở [ebp - 0x20]
mov edx, 2             ; Gán index i = 2 vào EDX
mov [eax + edx * 4], 20; Địa chỉ = Gốc + (Index * kích thước kiểu int). Gán ô nhớ đó = 20
                       ; (edx * 4 tương đương với việc dịch đi 8 bytes trong RAM)

; --- Trường hợp xử lý CẤU TRÚC (Struct) ---
mov ebx, [ebp - 0x40]  ; Giả sử địa chỉ gốc của struct p1 nằm tại [ebp - 0x40]
mov [ebx + 4], 100     ; Gán giá trị 100 vào ô nhớ cách địa chỉ gốc 4 bytes (Chính là p1.hp)
```
> 📌 Bí kíp đọc hiểu khi RE:
> * Khi thấy các lệnh có dạng nhân kích thước kiểu dữ liệu như [eax + edx * 4] hoặc [eax + edx * 8], $100\%$ đoạn đó đang truy cập vào một Mảng (Array).
> * Khi thấy các lệnh cộng dồn một hằng số cố định vào thanh ghi như [ebx + 4], [ebx + 0xC], đoạn đó đang truy cập vào các thuộc tính của một Cấu trúc (Struct) hoặc Object.
## 8. Phân Vùng Bộ Nhớ Trong File Thực Thi (PE / ELF)

Khi nạp một file thực thi (file `.exe` trên Windows - định dạng PE, hoặc file binary trên Linux - định dạng ELF) vào các công cụ Reverse Engineering như **IDA Pro** hay **Ghidra**, toàn bộ chương trình sẽ được ánh xạ vào bộ nhớ và chia thành các phân vùng quản lý riêng biệt (gọi là các **Sections**).

---

### 🗺️ Bảng Tra Cứu Các Phân Vùng Bộ Nhớ Cốt Lõi

| Tên Phân Vùng (Section) | Quyền Hạn (Permissions) | Chức Năng Chính | 🎯 Mục Tiêu Trong RE / PWN |
| :--- | :---: | :--- | :--- |
| **`.text`** | **R - X** *(Read / Execute)* | Chứa toàn bộ **mã máy (Machine Code)** và các lệnh Assembly của chương trình. | Nơi sẽ đọc code chính để tìm logic thuật toán, tìm hàm ẩn (Win function) hoặc tìm các đoạn mã hữu ích (**Gadgets**) để dựng ROP Chain. |
| **`.data`** | **R W -** *(Read / Write)* | Chứa các **biến toàn cục** (Global) hoặc biến tĩnh (Static) **đã được khởi tạo** giá trị từ trước. | Chứa các cấu hình mặc định, giá trị biến toàn cục có thể bị thao túng để thay đổi luồng chạy của chương trình. |
| **`.bss`** | **R W -** *(Read / Write)* | Chứa các **biến toàn cục** hoặc biến tĩnh **chưa được khởi tạo** giá trị (sẽ được cấp phát bằng 0 khi chạy). | Phân vùng tuyệt vời để chọn làm nơi ghi đè dữ liệu, chèn **Shellcode** hoặc thực hiện kỹ thuật **Stack Pivoting**. |
| **`.rdata`** / **`.rodata`** | **R - -** *(Read Only)* | **Read-Only Data:** Chứa các hằng số, dữ liệu chỉ đọc, đặc biệt là các **chuỗi ký tự (Strings)**. | Nơi chứa các chuỗi thông báo lỗi, chuỗi flag bí mật, hoặc các chuỗi bẫy (`"Wrong Password!"`, `"Access Granted!"`). |

---

### 🌐 Hệ Thống Hàm Liên Kết Ngoài: IAT (Windows) & PLT-GOT (Linux)

Khi chương trình sử dụng các hàm có sẵn của hệ điều hành (như `printf`, `scanf`, `malloc`, `system`), mã nguồn của các hàm này không nằm trong file binary mà nằm ở các thư viện động bên ngoài (như `libc.so` trên Linux hoặc `msvcrt.dll` trên Windows). Để gọi được chúng, chương trình phải qua một trạm trung chuyển:

* **Trên Windows (IAT - Import Address Table):** Một bảng chứa danh sách địa chỉ các hàm được nhập từ các file `.dll` bên ngoài.
* **Trên Linux (PLT & GOT):** * **PLT (Procedure Linkage Table):** Chứa các đoạn mã ngắn (stub) để nhảy đến địa chỉ hàm thật.
  * **GOT (Global Offset Table):** Một bảng chứa địa chỉ thực tế của các hàm nằm trong thư viện `libc` sau khi được nạp vào RAM.

> 💥 **Mẹo**: Trong các bài toán Pwnable nâng cao trên Linux (như **Format String** hoặc **Heap Exploit**), kỹ thuật **GOT Overwrite** là một đòn chí mạng. Mình sẽ tìm cách ghi đè địa chỉ của một hàm thông thường (ví dụ `printf`) trong bảng GOT thành địa chỉ của hàm `system`. Khi chương trình gọi `printf("/bin/sh")`, nó sẽ vô tình kích hoạt `system("/bin/sh")` và trao quyền kiểm soát Server (Shell) cho mình!
