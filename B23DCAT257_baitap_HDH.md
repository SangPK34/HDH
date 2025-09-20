# Bài tập Linux & Hệ điều hành
# Phạm Khắc Sang - B23DCAT257

## Câu 1 --- Mô-đun kernel "chào/tạm biệt" (C + CLI)

### a) Mã `hello_kmod.c`

``` c
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>

MODULE_LICENSE("GPL");
MODULE_AUTHOR("SangDZ");
MODULE_DESCRIPTION("Mo-dun chao/tam-biet don-gian");
MODULE_VERSION("1.0");

static int __init vao(void) {
    pr_info("[hello_kmod] Da nap: Xin chao tu kernel!\n");
    return 0;
}

static void __exit ra(void) {
    pr_info("[hello_kmod] Da go: Tam biet kernel!\n");
}

module_init(vao);
module_exit(ra);
```

### b) Makefile

``` make
obj-m += hello_kmod.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD  := $(shell pwd)

all:
    $(MAKE) -C $(KDIR) M=$(PWD) modules
clean:
    $(MAKE) -C $(KDIR) M=$(PWD) clean
```

### c) CLI thao tác

``` bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
make
sudo insmod hello_kmod.ko
dmesg | tail -n 20
lsmod | grep hello_kmod
modinfo hello_kmod.ko
sudo rmmod hello_kmod
dmesg | tail -n 20
make clean
```

------------------------------------------------------------------------

## Câu 2 --- Sơ đồ 7 trạng thái của tiến trình

**7 trạng thái:** 1. New (mới) 2. Ready (sẵn sàng) 3. Running (đang
chạy) 4. Blocked/Waiting (chờ) 5. Susp. Ready (treo-sẵn-sàng) 6. Susp.
Blocked (treo-đang-chờ) 7. Terminated/Exit (kết thúc)

**Chuyển trạng thái:** - New → Ready: tạo xong PCB. - Ready → Running:
scheduler cấp CPU. - Running → Ready: hết time-slice hoặc bị preempt. -
Running → Blocked: chờ I/O, sleep. - Blocked → Ready: sự kiện hoàn
tất. - Ready/Blocked → Susp. Ready/Susp. Blocked: bị swap/treo. - Susp.
Ready → Ready: resume xong. - Susp. Blocked → Blocked: resume nhưng vẫn
chờ. - Running → Terminated: gọi exit() hoặc kill.

------------------------------------------------------------------------

## Câu 3 --- C trên Linux: getpid(), getppid(), fork(), exit()

### a) Mã nguồn `proc_demo.c`

``` c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <stdlib.h>

int main(void) {
    pid_t p = fork();

    if (p < 0) {
        perror("fork");
        exit(1);
    }

    if (p == 0) {
        printf("[CON  ] pid=%d, ppid=%d\n", getpid(), getppid());
        exit(42);
    } else {
        int tt = 0;
        printf("[CHA  ] pid=%d (ppid=%d), con=%d\n", getpid(), getppid(), p);
        pid_t w = wait(&tt);
        if (w == -1) {
            perror("wait");
            exit(1);
        }
        if (WIFEXITED(tt)) {
            printf("[CHA  ] con %d thoat, ma=%d\n", w, WEXITSTATUS(tt));
        } else if (WIFSIGNALED(tt)) {
            printf("[CHA  ] con %d bi tin hieu=%d\n", w, WTERMSIG(tt));
        }
        exit(0);
    }
}
```

### b) Biên dịch & chạy

``` bash
gcc -Wall -O2 proc_demo.c -o proc_demo
./proc_demo
```

# Câu 4 — Lập trình đa luồng với `pthread` (C, Việt hoá, gọn)

## Yêu cầu
- Viết CT C đa luồng, tham số `argc` (hoặc tham số dòng lệnh) quy định **số luồng** cần tạo.
- Dùng API của `pthread.h`:
  - `int pthread_create(pthread_t *, const pthread_attr_t *, void *(*)(void *), void *);`
  - `void pthread_exit(void *);`
  - `int pthread_join(pthread_t, void **);`
- Theo dõi **giá trị & địa chỉ** của biến **cục bộ** (local), **toàn cục** (global), **thứ tự** thực thi luồng.
- Trả lời các câu hỏi ở cuối.

> Lưu ý: tên biến/chuỗi in ra **tiếng Việt** tối đa, chỉ giữ lại tên API/bắt buộc.

---

## Mã nguồn `thread_demo.c`
```c
// thread_demo.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>
#include <time.h>

// --- bien toan cuc ---
static int g = 0;              // dem mang tinh minh hoa (co the xay ra race)
static pthread_mutex_t k = PTHREAD_MUTEX_INITIALIZER; // khoa (neu muon tranh race)

typedef struct {
    int stt;                   // so thu tu luong (0..n-1)
} goi_tin;

void *chay(void *arg) {
    goi_tin *gt = (goi_tin *)arg;
    int x = 100 + gt->stt;     // bien cuc bo (local)
    
    // Tạo chút ngẫu nhiên để xáo trộn thứ tự in
    usleep(1000 * (rand() % 100));

    // KHONG khoa: de minh hoa thu tu thuc thi lung tung va data race
    // -> neu muon an toan, mo khoa 2 dong mutex lock/unlock ben duoi
    // pthread_mutex_lock(&k);
    g += 1;
    printf("[LUONG %2d] pid_tu = %lu | dia_chi_local=&x:%p, val_local=%d | dia_chi_global=&g:%p, val_global=%d\n",
           gt->stt, (unsigned long)pthread_self(), (void*)&x, x, (void*)&g, g);
    // pthread_mutex_unlock(&k);

    // Tra ve ma thoat = stt (convert qua (void*))
    pthread_exit((void*)(long)gt->stt);
}

int main(int argc, char **argv) {
    srand((unsigned)time(NULL));

    if (argc < 2) {
        fprintf(stderr, "Dung: %s <so_luong>\n", argv[0]);
        return 1;
    }
    int n = atoi(argv[1]);
    if (n <= 0) {
        fprintf(stderr, "so_luong phai > 0\n");
        return 1;
    }

    pthread_t *tid = (pthread_t*)malloc(n * sizeof(pthread_t));
    goi_tin  *args = (goi_tin*)malloc(n * sizeof(goi_tin));
    if (!tid || !args) {
        perror("malloc");
        return 1;
    }

    int x_main = 999; // bien cuc bo o main
    printf("[MAIN ] tao %d luong | dia_chi_local_main=&x_main:%p, val=%d | dia_chi_global=&g:%p, val=%d\n",
           n, (void*)&x_main, x_main, (void*)&g, g);

    // Tao luong
    for (int i = 0; i < n; ++i) {
        args[i].stt = i;
        int rc = pthread_create(&tid[i], NULL, chay, &args[i]);
        if (rc != 0) {
            fprintf(stderr, "Khong tao duoc luong %d (rc=%d)\n", i, rc);
            return 2;
        }
    }

    // Gop (join) tat ca
    for (int i = 0; i < n; ++i) {
        void *tra = NULL;
        int rc = pthread_join(tid[i], &tra);
        if (rc != 0) {
            fprintf(stderr, "Khong join duoc luong %d (rc=%d)\n", i, rc);
            return 3;
        }
        printf("[MAIN ] da gop xong LUONG %2d, tra_ve=%ld\n", i, (long)tra);
    }

    printf("[MAIN ] ket thuc | g=%d (co the != n neu data race, neu muon dung thi dung mutex)\n", g);

    free(tid);
    free(args);
    return 0;
}
```

---

## Biên dịch & chạy (CLI)
```bash
# Cai thu vien pthread (thuong co san), bien dich voi -pthread
gcc -Wall -O2 thread_demo.c -o thread_demo -pthread

# Chay voi N luong, vi du 5
./thread_demo 5

# Thu doi N (1, 2, 8, 16...) de quan sat thu tu
```

### Ví dụ kết quả (một lần chạy)
```
[MAIN ] tao 5 luong | dia_chi_local_main=&x_main:0x7ffcc4e2e8ec, val=999 | dia_chi_global=&g:0x55b5e8f1a018, val=0
[LUONG  3] pid_tu = 139985932777216 | dia_chi_local=&x:0x7f51239fed7c, val_local=103 | dia_chi_global=&g:0x55b5e8f1a018, val_global=1
[LUONG  1] pid_tu = 139985949562560 | dia_chi_local=&x:0x7f51241ffd7c, val_local=101 | dia_chi_global=&g:0x55b5e8f1a018, val_global=2
[LUONG  4] pid_tu = 139985924384512 | dia_chi_local=&x:0x7f51231fdd7c, val_local=104 | dia_chi_global=&g:0x55b5e8f1a018, val_global=3
[LUONG  0] pid_tu = 139985957955264 | dia_chi_local=&x:0x7f5124a00d7c, val_local=100 | dia_chi_global=&g:0x55b5e8f1a018, val_global=4
[LUONG  2] pid_tu = 139985941169856 | dia_chi_local=&x:0x7f5123a00d7c, val_local=102 | dia_chi_global=&g:0x55b5e8f1a018, val_global=5
[MAIN ] da gop xong LUONG  0, tra_ve=0
[MAIN ] da gop xong LUONG  1, tra_ve=1
[MAIN ] da gop xong LUONG  2, tra_ve=2
[MAIN ] da gop xong LUONG  3, tra_ve=3
[MAIN ] da gop xong LUONG  4, tra_ve=4
[MAIN ] ket thuc | g=5 (co the != n neu data race, neu muon dung thi dung mutex)
```

---

## Giải đáp câu hỏi

- **CT có bao nhiêu luồng?**  
  Bằng đúng tham số chạy `N` (ví dụ `./thread_demo 5` → có **5** luồng con) **+ 1** luồng **main**.

- **Main có gộp (join) các luồng không?**  
  Có. Vòng `pthread_join` gộp **từng** luồng → đảm bảo main **đợi** tất cả con kết thúc.

- **Các luồng có kết thúc theo thứ tự tạo không?**  
  **Không đảm bảo.** Lập lịch HĐH + ngủ ngẫu nhiên làm **thứ tự in/thoát thay đổi** mỗi lần.

- **Chạy nhiều lần, kết quả có đổi không?**  
  **Có.** Do lịch CPU + ngẫu nhiên, thứ tự, thời điểm, và **giá trị `g` có thể lệch** (nếu **không** dùng mutex).  
  Nếu muốn `g == N` ổn định, **bật khoá** (mở hai dòng `pthread_mutex_lock/unlock`).

- **Địa chỉ biến cục bộ & toàn cục khác gì?**  
  - **Local (`x`)**: mỗi luồng **có bản riêng** (địa chỉ khác nhau trên stack mỗi luồng).  
  - **Global (`g`)**: mọi luồng **cùng một địa chỉ** → truy cập chung → dễ **race** nếu không đồng bộ.

---

## Tuỳ chọn: bật khoá để tránh race
Trong hàm `chay()` bỏ comment 2 dòng:
```c
pthread_mutex_lock(&k);
...
pthread_mutex_unlock(&k);
```
Khi đó `g` sẽ tăng chính xác lên `N` sau khi gộp.
