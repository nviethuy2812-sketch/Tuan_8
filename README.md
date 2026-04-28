# BÁO CÁO THỰC HÀNH: CÔNG CỤ GỠ LỖI VÀ ĐÁNH GIÁ HIỆU NĂNG

**Sinh viên thực hiện:** [Điền tên của bạn]
**Mã số sinh viên:** [Điền MSSV]

---

## Mục tiêu
- Cài đặt thành công gdbserver, valgrind, perf, strace, ltrace trên Target (BeagleBone Black).
- Thực hiện sử dụng gdb để điều khiển luồng cơ bản từ host -> target.
- Phân tích lỗi Memory Leak bằng Valgrind.
- Bật giám sát và phân tích Core Dump.
- Phân tích hiệu năng bằng perf.
- Giám sát System call và Library call bằng strace, ltrace.

---

## Bài 1: Cài đặt công cụ Debugging vào Buildroot
**Bước 1: Cấu hình Buildroot**
```bash
cd ~/buildroot-2024.02.1
make menuconfig
```
Di chuyển đến **Target packages  --->  Debugging, profiling and benchmark**, nhấn dấu cách để bật `[*]`:
- `gdb` (chọn thêm `Build gdb server`)
- `valgrind`
- `strace`
- `ltrace`

Tiếp theo, thoái ra Menu chính, di chuyển đến **Kernel  --->  Linux Kernel Tools**, nhấn dấu cách để bật `[*]`:
- `perf`

**Bước 2: Build lại hệ thống**
```bash
make
```

**Bước 3: Ghi image mới vào thẻ SD**
```bash
sudo dd if=output/images/sdcard.img of=/dev/sdb bs=4M status=progress
sync
```
*Rút thẻ SD, cắm vào BeagleBone Black và khởi động lại.*

> <img width="882" height="427" alt="image" src="https://github.com/user-attachments/assets/4e524bec-404b-474c-8c2e-68c0f386778f" />

---

## Bài 2: Sử dụng GDB điều khiển luồng cơ bản (Host -> Target)
**Bước 1: Tạo chương trình thử nghiệm trên Host**
```bash
mkdir -p ~/debug_workspace
cd ~/debug_workspace
nano gdb_test.c
```
*Nội dung `gdb_test.c`:*
```c
#include <stdio.h>

int main() {
    int a = 5;
    int b = 10;
    int sum = 0;
    
    printf("Bat dau chuong trinh\n");
    for(int i = 0; i < 3; i++) {
        sum += a + b;
        printf("Vong lap %d: sum = %d\n", i, sum);
    }
    return 0;
}
```

**Bước 2: Biên dịch và chép sang Target**
*(Lưu ý: Bắt buộc dùng cờ `-g` để bật mode debug)*
```bash
~/buildroot-2024.02.1/output/host/bin/arm-buildroot-linux-gnueabihf-gcc -g gdb_test.c -o gdb_test

# Chép vào Target (gắn thẻ nhớ vào PC rồi chép)
sudo mount /dev/sdb2 /mnt/rootfs
sudo cp gdb_test /mnt/rootfs/root/
sync
sudo umount /mnt/rootfs
```

**Bước 3: Chạy gdbserver trên Target (BeagleBone)**
Cắm thẻ nhớ, khởi động BBB, đăng nhập root và chạy:
```bash
cd /root
gdbserver :1234 ./gdb_test
```
> **[<img width="632" height="407" alt="image" src="https://github.com/user-attachments/assets/46a85a94-7c0e-4af1-9eda-704b47b61ddc" />
]**

**Bước 4: Kết nối và điều khiển từ Host (Ubuntu)**
Mở Terminal trên Ubuntu:
```bash
cd ~/debug_workspace
gdb-multiarch ./gdb_test```
Trong dấu nhắc `(gdb)`, gõ các lệnh điều khiển:
```text
(gdb) target remote 192.168.7.2:1234
(gdb) b main              # Đặt breakpoint ở hàm main
(gdb) b 10                # Đặt breakpoint ở dòng 10
(gdb) c                   # Continue để chương trình chạy đến breakpoint
(gdb) n                   # Next qua từng dòng
(gdb) p sum               # In giá trị biến sum
(gdb) set var sum = 100   # Ép giá trị biến sum thành 100
(gdb) info registers      # Xem trạng thái thanh ghi
```
> **[Chèn hình ảnh 3: Cửa sổ GDB trên Ubuntu thực thi các lệnh b, n, p, set var, info registers]**

---

## Bài 3: Phân tích Memory Leak bằng Valgrind
**Bước 1: Tạo chương trình rò rỉ bộ nhớ**
```bash
nano leak_test.c
```
*Nội dung `leak_test.c`:*
```c
#include <stdlib.h>
#include <stdio.h>

int main() {
    char *ptr = (char *)malloc(100);
    printf("Da cap phat 100 bytes nhung khong thu hoi.\n");
    return 0; // Quên gọi free()
}
```

**Bước 2: Biên dịch và copy sang BBB**
```bash
~/buildroot-2024.02.1/output/host/bin/arm-buildroot-linux-gnueabihf-gcc -g leak_test.c -o leak_test
# Tương tự, copy file leak_test sang /root của BBB
```

**Bước 3: Chạy bằng Valgrind trên BBB**
```bash
valgrind --leak-check=full ./leak_test
```
> **[Chèn hình ảnh 4: Màn hình BBB báo lỗi "definitely lost: 100 bytes in 1 blocks" của Valgrind]**

**Bước 4: Sửa lỗi**
Sửa file `leak_test.c`, thêm `free(ptr);` ngay trước dòng `return 0;`.
Biên dịch lại, chép lại sang BBB và chạy lại Valgrind.
> **[Chèn hình ảnh 5: Màn hình BBB hiển thị "All heap blocks were freed -- no leaks are possible"]**

---

## Bài 4: Phân tích Core Dump
**Bước 1: Bật tính năng Core Dump trên BBB**
Mở Terminal trên BBB và gõ:
```bash
ulimit -c unlimited
```

**Bước 2: Tạo chương trình lỗi Segmentation Fault**
Trên máy Host:
```bash
nano crash_test.c
```
*Nội dung `crash_test.c`:*
```c
#include <stdio.h>

void cause_crash() {
    int *ptr = NULL;
    *ptr = 42; // Ghi vào vùng nhớ NULL
}

int main() {
    cause_crash();
    return 0;
}
```
Biên dịch (với `-g`) và chép file `crash_test` sang BBB.

**Bước 3: Chạy chương trình tạo Core Dump**
Trên BBB gõ:
```bash
./crash_test
ls -l core*
```
> **[Chèn hình ảnh 6: Chương trình văng lỗi "Segmentation fault (core dumped)" và sinh ra file 'core']**

**Bước 4: Đọc file Core bằng GDB**
Trên BBB gõ:
```bash
gdb ./crash_test core
```
Trong GDB, gõ lệnh truy vết `bt` (backtrace) để xem dòng code gây lỗi.
> **[Chèn hình ảnh 7: Kết quả GDB backtrace chỉ rõ nguyên nhân crash nằm ở dòng *ptr = 42 trong hàm cause_crash]**

---

## Bài 5: Đo hiệu năng bằng Perf
**Bước 1: Tạo chương trình giả lập tải nặng**
Trên máy Host:
```bash
nano perf_test.c
```
*Nội dung `perf_test.c`:*
```c
#include <stdio.h>

int main() {
    long long sum = 0;
    for(long long i = 0; i < 500000000; i++) {
        sum += i;
    }
    printf("Sum = %lld\n", sum);
    return 0;
}
```
Biên dịch (dùng `-O0` để tránh bị tối ưu) và chép sang BBB:
```bash
~/buildroot-2024.02.1/output/host/bin/arm-buildroot-linux-gnueabihf-gcc -O0 -g perf_test.c -o perf_test
```

**Bước 2: Đo lường tổng quan bằng perf stat**
Trên BBB:
```bash
perf stat ./perf_test
```
> **[Chèn hình ảnh 8: Màn hình trả về các chỉ số CPU cycles, instructions, context switches của perf stat]**

**Bước 3: Record và Report**
Trên BBB, ghi lại quá trình chạy:
```bash
perf record -g ./perf_test
```
Xem lại báo cáo:
```bash
perf report
```
> **[Chèn hình ảnh 9: Màn hình giao diện perf report đánh giá % CPU của hàm main]**

---

## Bài 6: Tracing với strace và ltrace
**Bước 1: Viết chương trình truy cập file**
Trên máy Host:
```bash
nano trace_test.c
```
*Nội dung `trace_test.c`:*
```c
#include <stdio.h>
#include <unistd.h>

int main() {
    printf("Test Library Call\n");
    sleep(1);
    
    FILE *f = fopen("test.txt", "w");
    if(f) {
        fprintf(f, "Testing file IO\n");
        fclose(f);
    }
    return 0;
}
```
Biên dịch và chép `trace_test` sang BBB.

**Bước 2: Dùng strace giám sát System Calls**
Trên BBB gõ:
```bash
strace ./trace_test
```
> **[Chèn hình ảnh 10: strace hiển thị các lệnh giao tiếp nhân như write, nanosleep, openat, close]**

**Bước 3: Dùng ltrace giám sát Library Calls**
Trên BBB gõ:
```bash
ltrace ./trace_test
```
> **[Chèn hình ảnh 11: ltrace hiển thị các hàm gọi thư viện C như puts, sleep, fopen, fprintf, fclose]**
