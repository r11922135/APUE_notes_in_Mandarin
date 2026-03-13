# Chapter 7 - Process Environment

## 這章在做什麼

這章是在進入 process control 之前，先把「單一 process 自己的內部環境」講清楚。  
你可以把它看成是 Chapter 8 的前置地基。

這章的主線包括：

- 一個 `C` 程式是怎麼啟動的
- process 怎麼結束
- `argv` 和 environment 怎麼進來
- process 記憶體通常怎麼分區
- heap memory 怎麼配置
- environment variable 怎麼查與改
- `setjmp` / `longjmp` 怎麼做 nonlocal goto
- resource limit 怎麼限制 process

如果這章沒建立好心智模型，後面在看 `fork`、`exec`、signals、threads 時，很容易把 stack、heap、environment、exit status、resource limits 這些概念混在一起。

## 本章小節地圖

- `7.1 Introduction`
- `7.2 main Function`
- `7.3 Process Termination`
- `7.4 Command-Line Arguments`
- `7.5 Environment List`
- `7.6 Memory Layout of a C Program`
- `7.7 Shared Libraries`
- `7.8 Memory Allocation`
- `7.9 Environment Variables`
- `7.10 setjmp and longjmp Functions`
- `7.11 getrlimit and setrlimit Functions`
- `7.12 Summary`

## 先抓住這章最重要的心智模型

### 一個 process 不只是「正在跑的程式」

當 kernel 啟動一個程式時，實際上給它的不只是 machine code，還有很多執行環境：

- command-line arguments
- environment
- user / group IDs
- open file descriptors
- signal mask
- current working directory
- memory layout
- resource limits

所以 `process environment` 不是指 shell 的環境變數而已，而是整個 process 執行狀態的上下文。

### 這章有兩條主線

#### 第一條：process 怎麼活著

- 怎麼開始
- 怎麼拿到參數
- 怎麼拿到環境
- 怎麼結束

#### 第二條：process 裡面的資源怎麼管理

- 記憶體怎麼分
- heap 怎麼配置
- environment list 怎麼擴張
- resource limit 怎麼限制它

## 7.1 Introduction

APUE 在這章先不碰 `fork` / `exec` 的細節，而是把焦點放在：

- 單一 process 自己的執行環境

這個順序是很合理的。  
因為如果你連一個 process 平常長什麼樣、有哪些狀態、怎麼起跑和結束都不清楚，就很難理解之後 process 之間的互動。

## 7.2 `main` Function

### `main` 不是程式真正的第一站

表面上看，`C` 程式都從：

```c
int main(int argc, char *argv[])
```

開始。  
但真正更精確的說法是：

- kernel 透過 `exec` 載入你的程式
- 然後先進入一段 start-up routine
- 這段 start-up routine 再幫你把 `argc`、`argv`、environment 等資料整理好
- 最後才呼叫 `main`

### 這段 start-up routine 在做什麼

通常包括：

- 取出 kernel 傳下來的 argument / environment
- 初始化 runtime
- 安排 `main` 回傳時要怎麼收尾

這也是為什麼很多實作在概念上都像：

```c
exit(main(argc, argv));
```

也就是：

- `main` 回傳後，不是直接神祕消失
- 而是被包進 `exit`

## 7.3 Process Termination

這節超重要，因為後面很多章都會用到 exit status、termination status、zombie 這些概念。

### process 結束的方式

APUE 把它分成：

#### 正常終止

1. `return` from `main`
2. 呼叫 `exit`
3. 呼叫 `_exit` 或 `_Exit`
4. 最後一個 thread 從 start routine return
5. 最後一個 thread 呼叫 `pthread_exit`

#### 非正常終止

6. 呼叫 `abort`
7. 收到某些 signal
8. 最後一個 thread 對 cancellation request 做出終止回應

這章先聚焦在 process 層，不深入 thread 版本。

### `exit`、`_exit`、`_Exit` 差在哪

```c
#include <stdlib.h>

void exit(int status);
void _Exit(int status);

#include <unistd.h>

void _exit(int status);
```

#### `exit`

做的是「完整收尾」：

- 呼叫 `atexit` handlers
- flush / close standard I/O streams
- 然後才回到 kernel

#### `_exit` / `_Exit`

比較像「直接離開 user-space cleanup」：

- 不跑 `atexit` handlers
- 在 UNIX 上通常也不 flush stdio
- 直接回 kernel

### 什麼時候要用 `_exit`

典型情況是：

- `fork` 之後 child 要結束，但不想碰 parent 複製下來的 stdio buffer

這在 Chapter 8 會反覆出現。

### `main` 回傳值與 exit status

如果：

```c
return 0;
```

本質上就等價於：

```c
exit(0);
```

### 重要觀念：exit status 與 termination status 不完全一樣

- `exit status`：你傳給 `exit` / `_exit` 的值
- `termination status`：kernel 整理後讓 parent 用 `wait` 家族取回的狀態

後者除了正常 exit value，也可能表示：

- 被哪個 signal 殺掉
- 有沒有 core dump

### `atexit`

```c
#include <stdlib.h>

int atexit(void (*func)(void));
```

它讓你註冊在 `exit` 時自動執行的 handlers。

#### 順序

- 後註冊的先執行
- 同一個函式註冊多次，就會被呼叫多次

### 範例：`atexit` 執行順序

```c
#include <stdio.h>
#include <stdlib.h>

static void cleanup1(void) { puts("cleanup1"); }
static void cleanup2(void) { puts("cleanup2"); }

int main(void) {
    atexit(cleanup2);
    atexit(cleanup1);
    puts("main done");
    return 0;
}
```

輸出順序會是：

```text
main done
cleanup1
cleanup2
```

### 常見誤解

- `return from main` 並不是繞過 `exit`
- `_exit` 不是比較「高級」，只是更直接、更少 cleanup
- `atexit` handler 只會在正常結束流程中跑，不保證在 signal abnormal termination 下執行

## 7.4 Command-Line Arguments

### `argc` / `argv`

這是最基本但也最容易理所當然的一塊。

```c
int main(int argc, char *argv[])
```

- `argc`：argument count
- `argv`：argument vector

### `argv[0]` 是什麼

很多初學者以為它一定是程式檔名。  
比較精確地說：

- 它是呼叫者放進去給新程式的第 0 個字串

通常是程式名或路徑，但這是一種慣例，不是鐵律。

### `argv[argc] == NULL`

這個保證很有用。  
所以你除了用 `argc` 迴圈，也能用 null terminator 方式掃。

### 範例：印出所有 arguments

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    int i;
    for (i = 0; i < argc; i++)
        printf("argv[%d] = %s\n", i, argv[i]);
    return 0;
}
```

## 7.5 Environment List

### `environ`

除了 argument list，process 還會拿到 environment list。

它本質上是：

- 一個 `char **`
- 每個元素指向一條 `name=value` 字串
- 最後以 `NULL` 結尾

傳統上可透過：

```c
extern char **environ;
```

取得。

### environment 的本質

environment 是給 application 自己解讀的，kernel 不會去理解其中大多數內容。

例如：

- `PATH`
- `HOME`
- `SHELL`
- `TERM`
- `TZ`

都是 user-space 程式與 library 在用。

### `main` 第三個參數

有些系統或歷史做法會讓 `main` 長這樣：

```c
int main(int argc, char *argv[], char *envp[])
```

但 POSIX 的建議是：

- 用 `environ`

不要把 `envp` 當成主要介面。

## 7.6 C Program 的 Memory Layout

這節是系統程式設計最基本的地圖。

### 常見分區

#### text segment

- machine instructions
- 常常 read-only
- 常常可 shared

#### initialized data segment

- 已初始化的 global / static 變數

例如：

```c
int maxcount = 99;
```

#### uninitialized data segment (`bss`)

- 未初始化的 global / static 變數
- 由 kernel 在程式開始前清成 0

例如：

```c
long sum[1000];
```

#### stack

- automatic variables
- function call frames
- return address
- 暫存資料

#### heap

- dynamic allocation
- `malloc` / `free` 主要活動區

### 很重要的理解

這是一個「典型模型」，不是所有平台都保證完全長得一樣。  
但這個模型非常適合幫你理解：

- 為什麼 recursive function 可以運作
- 為什麼 stack variable 不能 return 給外面用
- 為什麼 `malloc` 出來的空間能跨函式存在

### `size` 指令

書裡也提到：

- `size` 可以看 text / data / bss 大小

但 stack / heap 不是可執行檔裡靜態固定的一部分，所以不會像這三者那樣直接被列出固定大小。

## 7.7 Shared Libraries

### 核心目的

把常用 library code 從每個 executable 中抽出來，共用同一份。

### 好處

- executable file 變小
- memory usage 降低
- library 更新時不必全部重連

### 代價

- 載入時或第一次呼叫時可能有額外 overhead

### 這節真正想傳達的重點

shared library 不是只是節省磁碟空間，它其實也影響：

- process 啟動方式
- code sharing
- executable 與 runtime loader 的關係

## 7.8 Memory Allocation

這節是 C / UNIX 實作很核心的一塊。

### 三大配置函式

```c
#include <stdlib.h>

void *malloc(size_t size);
void *calloc(size_t nobj, size_t size);
void *realloc(void *ptr, size_t newsize);
void free(void *ptr);
```

### `malloc`

- 配 `size` bytes
- 初值不保證

### `calloc`

- 配 `nobj * size` bytes
- 全部設為 0 bits

### `realloc`

- 改變既有配置區大小
- 可能原地長大
- 也可能搬家

### `realloc` 的兩個重要規則

#### 它的第二個參數是新總大小

不是差值。

#### `realloc(NULL, size)` 等於 `malloc(size)`

這是很常用的寫法。

### `free`

- 把空間還給 allocator
- 通常先回到 allocator pool，不一定立刻還給 kernel

### 對齊保證

書裡提醒：

- `malloc/calloc/realloc` 回傳的指標要夠對齊，能安全放任何基本型別

這不是小事，因為某些架構對 alignment 很敏感。

### 最常見的錯誤

#### memory leak

- `malloc` 了但忘記 `free`

#### double free

- 同一塊記憶體 `free` 兩次

#### invalid free

- 拿不是 allocator 回傳的指標去 `free`

#### buffer overrun / underrun

- 寫超過配置區尾端
- 或寫到配置區前面

這種錯常常很晚才爆，而且爆在和真正 bug 地點完全不同的位置。

### `realloc` 的實務寫法

因為它可能失敗，所以常見安全寫法是：

```c
void *tmp = realloc(ptr, newsize);
if (tmp == NULL) {
    /* ptr 仍有效，不能直接丟失 */
} else {
    ptr = tmp;
}
```

### `alloca`

`alloca` 很值得知道：

- 介面像 `malloc`
- 但記憶體來自 stack frame
- function return 就自動消失

優點：

- 不用 `free`

缺點：

- 可攜性與風險都比較高
- 配太大容易炸 stack

### 範例：動態成長陣列

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t cap = 4, n = 0;
    int *a = malloc(cap * sizeof(int));
    int i;

    if (a == NULL) {
        perror("malloc");
        return 1;
    }

    for (i = 0; i < 20; i++) {
        if (n == cap) {
            int *tmp;
            cap *= 2;
            tmp = realloc(a, cap * sizeof(int));
            if (tmp == NULL) {
                free(a);
                perror("realloc");
                return 1;
            }
            a = tmp;
        }
        a[n++] = i * i;
    }

    for (i = 0; i < (int)n; i++)
        printf("%d\n", a[i]);

    free(a);
    return 0;
}
```

## 7.9 Environment Variables

### `getenv`

最基本的讀法是：

```c
#include <stdlib.h>

char *getenv(const char *name);
```

它回的是：

- value 的指標
- 找不到就 `NULL`

### 為什麼應優先用 `getenv`

因為如果你只是要找單一變數，直接掃 `environ` 太低階，也不夠清楚。

### POSIX 常見 environment variables

像：

- `HOME`
- `PATH`
- `SHELL`
- `TERM`
- `TZ`
- `LANG`
- `LOGNAME`
- `PWD`

這些在不同 API 或 shell 行為裡都會出現。

### 修改 environment 的函式

```c
#include <stdlib.h>

int putenv(char *str);
int setenv(const char *name, const char *value, int rewrite);
int unsetenv(const char *name);
```

有些系統還有：

- `clearenv`

### `putenv` 和 `setenv` 的重要差異

#### `putenv`

你給它的是完整字串：

```c
"NAME=value"
```

而且實作常會直接把這個字串指標塞進 environment list。  
所以：

- 千萬不要把 stack 上的字串傳給 `putenv`

不然 function return 後，那塊記憶體就無效了。

#### `setenv`

- 傳 `name` 與 `value` 分開
- library 會自己配置 / 複製
- `rewrite` 控制是否覆蓋既有值

### `unsetenv`

- 移除變數
- 就算原本不存在也不算錯

### environment list 為什麼難改

因為原始 environment list 常在 process memory 上方一塊區域，不像 heap 那樣方便擴張。  
所以真正新增或放大項目時，實作常得：

1. 在 heap 另外配新的 `name=value` 字串
2. 另外配新的 pointer list
3. 把 `environ` 指到新位置

這也說明了：

- environment modification 本質上和動態記憶體管理綁得很緊

## 7.10 `setjmp` 和 `longjmp`

### 它們在解什麼問題

C 的 `goto` 不能跳到別的函式。  
但有時你在很深的 call chain 裡發現錯誤，想直接跳回外層主循環。

這就是：

- `setjmp`
- `longjmp`

的用途。

```c
#include <setjmp.h>

int setjmp(jmp_buf env);
void longjmp(jmp_buf env, int val);
```

### 心智模型

- `setjmp`：先在某個位置設一個「返回點」
- `longjmp`：之後從別的深層函式直接跳回來

### `setjmp` 的回傳值

- 直接呼叫時回傳 `0`
- 從 `longjmp` 跳回來時回傳非零值

### stack unwinding

`longjmp` 會把中間那些 stack frames 全部丟掉。  
所以它很像「跨很多層 return 一次跳回去」。

### 最重要的坑：automatic / register variables

如果一個 automatic variable：

- 在 `setjmp` 之後有被改過
- 而且不是 `volatile`

那 `longjmp` 回來後，它的值是 indeterminate。

這是這節最重要、也最容易被忽略的規則。

### 為什麼 `volatile` 有關

因為 optimizer 可能把變數放在 register 裡。  
`longjmp` 回來後，register 與 memory 恢復狀態可能和你直覺不同。

### 實務規則

如果某個 automatic variable 在：

- `setjmp` 之後會被改
- 而且你在 `longjmp` 回來後還想可靠使用

就要考慮：

- 把它變 `volatile`
- 或改成 static / global
- 或重新設計控制流程

### 範例：簡化版 nonlocal error return

```c
#include <setjmp.h>
#include <stdio.h>
#include <stdlib.h>

static jmp_buf env;

static void deep_fail(void) {
    longjmp(env, 1);
}

int main(void) {
    if (setjmp(env) != 0) {
        puts("recovered from deep error");
        return 0;
    }

    puts("before deep call");
    deep_fail();
    puts("never reached");
    return 0;
}
```

### 另一個重要坑：不要讓外部繼續用已失效的 automatic object

書裡那個 `setvbuf` 例子很經典：

- function 內的自動陣列被拿去當 stdio buffer
- function return 後，那塊 stack 空間被重用
- stdio 卻還以為那塊 buffer 有效

結果一定亂掉。

所以基本規則是：

- function return 後，不能再依賴那個 function 的 automatic variable

## 7.11 `getrlimit` 和 `setrlimit`

每個 process 都有一組 resource limits。

```c
#include <sys/resource.h>

int getrlimit(int resource, struct rlimit *rlptr);
int setrlimit(int resource, const struct rlimit *rlptr);
```

### `struct rlimit`

```c
struct rlimit {
    rlim_t rlim_cur;  /* soft limit */
    rlim_t rlim_max;  /* hard limit */
};
```

### soft limit vs hard limit

#### soft limit

- 目前實際生效限制

#### hard limit

- soft limit 最高能拉到哪

### 三條規則

1. process 可把 soft limit 調到不超過 hard limit
2. process 可把 hard limit 往下調，但這通常不可逆
3. 只有 privileged process 才能把 hard limit 往上調

### `RLIM_INFINITY`

表示：

- 沒有限制

### 常見 limits

#### `RLIMIT_CORE`

- core file 最大大小

#### `RLIMIT_CPU`

- CPU 秒數上限
- 超過 soft limit 會送 `SIGXCPU`

#### `RLIMIT_FSIZE`

- 可建立檔案最大大小
- 超過 soft limit 會送 `SIGXFSZ`

#### `RLIMIT_NOFILE`

- 每個 process 最多 open files 數量

#### `RLIMIT_DATA`

- data segment 大小上限

#### `RLIMIT_STACK`

- stack 大小上限

#### `RLIMIT_AS`

- 總 address space 上限

#### `RLIMIT_NPROC`

- 每個 real user 可建立 child processes 數量上限

### 這些限制的實務意義

它們影響：

- process 能吃多少記憶體
- 能開多少檔案
- 能跑多久 CPU
- 能不能產生 core

shell 裡的：

- `ulimit`
- `limit`

本質上就是在改這些值。

### 範例：查 `RLIMIT_NOFILE`

```c
#include <stdio.h>
#include <sys/resource.h>

int main(void) {
    struct rlimit rl;

    if (getrlimit(RLIMIT_NOFILE, &rl) == -1) {
        perror("getrlimit");
        return 1;
    }

    printf("soft = %llu\n", (unsigned long long)rl.rlim_cur);
    printf("hard = %llu\n", (unsigned long long)rl.rlim_max);
    return 0;
}
```

## 7.12 Summary

### 這章讀完你應該真的會的事

- 知道 `main` 前面其實還有 start-up routine
- 分得清 `exit`、`_exit`、`_Exit`
- 會解讀 `argc` / `argv` / `environ`
- 理解 text / data / bss / stack / heap 的典型分工
- 會用 `malloc` / `calloc` / `realloc` / `free`
- 知道 `putenv` 與 `setenv` 的差異
- 理解 `setjmp` / `longjmp` 是 nonlocal goto
- 知道 `volatile` 在 `longjmp` 情境下為什麼重要
- 會看 soft limit / hard limit，知道 `ulimit` 背後是什麼

### 這章最容易踩坑的地方

- 以為 `main` 就是程式真正第一個執行點
- `fork` 後 child 用 `exit` 導致重複 flush stdio
- 用 `char *` 接 `getenv` 結果後又以為它是自己擁有的可長期改寫空間
- 把 stack 上字串傳給 `putenv`
- `realloc` 失敗時直接覆蓋掉原指標
- `longjmp` 回來後還相信被 optimizer 動過的 automatic variable
- 把 stdio buffer 設在 function 的 automatic array 上

### 建議你現在立刻動手做

- 寫個程式印出 `argv` 與 `environ`
- 用 `malloc` + `realloc` 做一個會自動長大的陣列
- 用 `setenv` / `unsetenv` 改一個自訂變數，觀察 child process 是否繼承
- 寫個 `setjmp` / `longjmp` 小程式，試 `volatile` 與非 `volatile` 的差異
- 用 `getrlimit` 印出 `RLIMIT_NOFILE`、`RLIMIT_STACK`、`RLIMIT_CORE`

### 一句總結

這章真正教你的，是 process 在執行前後與執行當下到底帶著哪些狀態與資源，這些東西後面幾乎都會在 `fork`、`exec`、signals、threads 裡再次出現。
