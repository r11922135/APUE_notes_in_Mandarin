# Chapter 5 - Standard I/O Library

## 這章在做什麼

Chapter 3 教的是 `open/read/write` 這種 `unbuffered I/O`。  
Chapter 5 則是在講我們平常更常寫到的那一層：

- `fopen`
- `fclose`
- `getc`
- `fgets`
- `fprintf`
- `fread`
- `fwrite`

也就是 `stdio`。

這章的真正重點不是背 API 名字，而是弄清楚三件事：

- `FILE *` 和 `fd` 的關係
- buffering 到底做了什麼
- 為什麼 `stdio` 好用，但也會帶來一些容易踩坑的語意差異

## 本章小節地圖

- `5.1 Introduction`
- `5.2 Streams and FILE Objects`
- `5.3 Standard Input, Standard Output, and Standard Error`
- `5.4 Buffering`
- `5.5 Opening a Stream`
- `5.6 Reading and Writing a Stream`
- `5.7 Line-at-a-Time I/O`
- `5.8 Standard I/O Efficiency`
- `5.9 Binary I/O`
- `5.10 Positioning a Stream`
- `5.11 Formatted I/O`
- `5.12 Implementation Details`
- `5.13 Temporary Files`
- `5.14 Memory Streams`
- `5.15 Alternatives to Standard I/O`
- `5.16 Summary`

## 先抓住這章最重要的心智模型

### `fd` 是 kernel 觀點，`FILE *` 是 library 觀點

在 Chapter 3：

- 你拿到的是 `file descriptor`

在 Chapter 5：

- 你拿到的是 `FILE *`

`FILE *` 底下通常還是綁著某個 `fd`，但它多包了一層 library state，例如：

- user-space buffer
- error flag
- EOF flag
- stream orientation
- 目前 buffer 裡有多少資料

所以：

- `read` / `write` 比較直接
- `stdio` 比較方便，但你得理解它的 buffer 什麼時候真的送出去

### `stdio` 的目標是「少做 system call」

它想解決的問題很簡單：

- `read(2)` / `write(2)` 很貴
- 不要每處理一個字元就切一次 kernel

所以它會把資料先放在 user-space buffer，累積到某個時機再一次送出去。

## 5.1 Introduction

APUE 一開始就點出：

- `stdio` 是 ISO C 的一部分
- 所以不只是 UNIX 才有

但 APUE 關心的是它在 UNIX 上怎麼和底層 file descriptor / system call 互動。

這章的關鍵不是「stdio 比較高階」這句空話，而是：

- 高階抽象帶來方便
- 也帶來另外一組必須理解的行為規則

## 5.2 Streams and `FILE` Objects

### `stream` 是什麼

在 `stdio` 裡，核心概念不是 descriptor，而是 `stream`。

當你：

```c
FILE *fp = fopen("data.txt", "r");
```

你就是把一個 stream 關聯到某個 file。

### `FILE *` 裡可能包含什麼

書裡提到，`FILE` object 通常會保存：

- 對應的 `fd`
- buffer 指標
- buffer 大小
- buffer 目前已有多少字元
- error flag
- EOF flag

你不應該自己去碰它的內部欄位，但你要知道它確實存在這一層狀態。

### orientation

`stream` 一開始可能是沒有 orientation 的。  
之後如果你先用：

- byte-oriented I/O，就變成 byte oriented
- wide-character I/O，就變成 wide oriented

可用 `fwide` 觀察或嘗試設定：

```c
#include <stdio.h>
#include <wchar.h>

int fwide(FILE *fp, int mode);
```

實務上，APUE 後面主要都在談 byte-oriented streams。

## 5.3 `stdin`, `stdout`, `stderr`

每個 process 一開始就有三個預先打開的標準 stream：

- `stdin`
- `stdout`
- `stderr`

它們分別對應 Chapter 3 的：

- `STDIN_FILENO`
- `STDOUT_FILENO`
- `STDERR_FILENO`

### 重要習慣

- 正常輸出用 `stdout`
- 錯誤訊息用 `stderr`

因為這可以讓：

- 正常輸出被 redirect
- 錯誤訊息仍直接出現在 terminal

## 5.4 Buffering

這節是整章最核心的部分。

### 三種 buffering 模式

#### fully buffered

資料先存在 buffer，等 buffer 滿了才真的做 I/O。

常見情境：

- regular file

#### line buffered

通常遇到 newline 或某些特定時機就 flush。

常見情境：

- terminal 上的 `stdout`

#### unbuffered

幾乎每次呼叫都盡快送出去。

常見情境：

- `stderr`

### 為什麼會有 line buffering

因為互動式程式很需要「我打一行、你回一行」這種節奏。  
如果都 fully buffered，你可能一直看不到輸出。

### line buffering 的兩個坑

書裡特別提醒兩件事：

1. buffer 不是無限大，太長的 line 還是可能先觸發 I/O
2. 當某些 input 操作需要真的去 kernel 取資料時，line-buffered output streams 可能會先被 flush

### `fflush` 到底做了什麼

```c
#include <stdio.h>

int fflush(FILE *fp);
```

它做的是：

- 把 `stdio` buffer 的資料交給 kernel

它沒保證：

- 資料一定落到磁碟 platters / SSD flash cell

如果你要的是 durability，想的是 `fsync` 類型，不是 `fflush`。

### `setbuf` / `setvbuf`

```c
#include <stdio.h>

void setbuf(FILE *restrict fp, char *restrict buf);
int setvbuf(FILE *restrict fp, char *restrict buf, int mode, size_t size);
```

這兩個讓你改 buffering 策略。

### `setvbuf` 三種模式

- `_IOFBF`：fully buffered
- `_IOLBF`：line buffered
- `_IONBF`：unbuffered

### 使用時機限制

它們必須在：

- stream 已經打開之後
- 但還沒做其他 I/O 操作之前

呼叫。

### 常見實務建議

- 除非有很明確理由，讓 library 自己決定 buffer 比較安全
- 不要隨便把自動變數的 buffer 丟給 `setbuf`，然後函式還沒 `fclose` 就 return

## 5.5 Opening a Stream

```c
#include <stdio.h>

FILE *fopen(const char *restrict pathname, const char *restrict type);
FILE *freopen(const char *restrict pathname, const char *restrict type,
              FILE *restrict fp);
FILE *fdopen(int fd, const char *type);
```

### 三者差異

- `fopen`：直接開 pathname
- `freopen`：把既有 stream 重新綁到另一個 file
- `fdopen`：把已存在的 `fd` 包成 `FILE *`

### `fdopen` 很重要的用途

因為有些東西不是用 `fopen` 開出來的，例如：

- `pipe`
- `socket`
- `accept`

這時你先拿到的是 `fd`，若想用 `fprintf` / `fgets` 那套，就要 `fdopen`。

### `type` 常見模式

- `r`
- `w`
- `a`
- `r+`
- `w+`
- `a+`

### `a` 模式的真正語意

append 模式不是「打開時先 `lseek` 到檔尾」而已。  
正確做法是底層要有 `O_APPEND` 的語意，這樣多個 process 同時寫才安全。

### 更新模式的兩條規則

當 stream 同時支援讀寫（有 `+`）時，書裡強調兩條規則：

- output 後不能直接 input，除非中間有 `fflush` / `fseek` / `fsetpos` / `rewind`
- input 後不能直接 output，除非中間有 `fseek` / `fsetpos` / `rewind`，或讀到 EOF

這是非常常見的考題與 bug 來源。

### 範例：更新模式正確切換

```c
#include <stdio.h>

int main(void) {
    FILE *fp = fopen("data.txt", "w+");
    char buf[32];

    if (fp == NULL) {
        perror("fopen");
        return 1;
    }

    fputs("hello\n", fp);
    fflush(fp);
    rewind(fp);

    if (fgets(buf, sizeof(buf), fp) != NULL)
        fputs(buf, stdout);

    fclose(fp);
    return 0;
}
```

### `fclose`

```c
int fclose(FILE *fp);
```

做的事情包括：

- flush 尚未輸出的 buffered data
- 丟掉 input buffer
- 關閉底層 `fd`
- 釋放 library 自己配置的 buffer

### process 正常結束時

如果程式是正常結束：

- `return from main`
- 或 `exit`

library 會幫你 flush 並關閉 open streams。

## 5.6 Reading and Writing a Stream

APUE 把 `stdio` 的 unformatted I/O 分成三種：

- character-at-a-time
- line-at-a-time
- direct / binary I/O

這節先講 character-at-a-time。

### 輸入：`getc`, `fgetc`, `getchar`

```c
int getc(FILE *fp);
int fgetc(FILE *fp);
int getchar(void);
```

### `getc` 與 `fgetc` 差異

書裡強調：

- `getc` 可以是 macro
- `fgetc` 保證是 function

所以：

1. `getc(x++)` 這種有 side effect 的寫法不安全
2. `fgetc` 可以取 address，`getc` 不一定行
3. `fgetc` 可能稍慢

### 為什麼回傳型別是 `int`

這是一個很重要但常被忽略的 C 細節。

因為它必須同時能表示：

- 所有可能的 `unsigned char` 值
- 再加上一個特殊值 `EOF`

所以不能用 `char` 來接。

錯誤示範：

```c
char c;
c = getchar();
if (c == EOF) { ... }
```

正確示範：

```c
int c;
c = getchar();
if (c == EOF) { ... }
```

### `EOF` 不一定代表錯誤

`getc` 讀不到東西時，回傳 `EOF` 可能代表：

- 真正到檔尾
- 發生錯誤

要用：

- `feof(fp)`
- `ferror(fp)`

區分。

### `clearerr`

用來清掉：

- error flag
- EOF flag

### `ungetc`

```c
int ungetc(int c, FILE *fp);
```

它讓你把字元推回 stream，之後下一次讀會先讀到它。

這在 parser / tokenizer 很常用：

- 先偷看下一個字元
- 判斷完再塞回去

### 要注意的限制

- 不能 push back `EOF`
- POSIX / ISO C 只保證至少能推回 1 個字元
- 推回的是 library buffer，不是把底層檔案真的改回去

### 輸出：`putc`, `fputc`, `putchar`

```c
int putc(int c, FILE *fp);
int fputc(int c, FILE *fp);
int putchar(int c);
```

和輸入那組的差異概念幾乎一樣。

## 5.7 Line-at-a-Time I/O

### `fgets` 與 `gets`

```c
char *fgets(char *restrict buf, int n, FILE *restrict fp);
char *gets(char *buf);
```

### 先講結論

- `gets` 不要用

原因不是風格問題，而是它根本沒有 buffer size 參數，天生就可能 overflow。

### `fgets` 的行為

- 最多讀 `n - 1` 個字元
- 讀到 newline 也會收進 buffer
- 最後補 `'\0'`

所以用 `fgets` 時，你要習慣：

- buffer 裡常常會帶著尾端 newline

### `fputs` 與 `puts`

```c
int fputs(const char *restrict str, FILE *restrict fp);
int puts(const char *str);
```

### 差異

- `fputs`：原字串輸出，不會自動加 newline
- `puts`：輸出字串後會自動補 newline

APUE 的態度很務實：

- 為了避免記憶負擔，很多時候固定用 `fgets` / `fputs` 比較清楚

### 範例：逐行複製

```c
#include <stdio.h>

int main(void) {
    char buf[4096];

    while (fgets(buf, sizeof(buf), stdin) != NULL) {
        if (fputs(buf, stdout) == EOF) {
            perror("fputs");
            return 1;
        }
    }

    if (ferror(stdin)) {
        perror("stdin");
        return 1;
    }

    return 0;
}
```

## 5.8 Standard I/O Efficiency

這節的核心結論很重要：

- `stdio` 通常沒有你想像中那麼慢

### 為什麼

雖然你可能在程式碼層面是一個字元一個字元地 `getc`，但底下不是每次都呼叫 `read(2)`。  
多半是：

- 先一次抓一大塊進 buffer
- 之後在 user space 慢慢吐給你

### 真正省下的是 system call

相比直接用 `read(fd, &c, 1)`：

- `fgetc` 仍有很多 function calls
- 但 system calls 少非常多

這就是為什麼 stdio 版本往往快很多。

### 實務結論

對大多數應用程式：

- 不要因為「感覺高階就慢」而本能排斥 `stdio`
- 先用可讀性好的寫法
- 真的有瓶頸再測

## 5.9 Binary I/O

```c
size_t fread(void *restrict ptr, size_t size, size_t nobj,
             FILE *restrict fp);
size_t fwrite(const void *restrict ptr, size_t size, size_t nobj,
              FILE *restrict fp);
```

### 它們在做什麼

不是用「幾個 byte」思考，而是用：

- 每個 object 有多大 `size`
- 要讀/寫幾個 object `nobj`

### 常見兩種用法

#### 陣列片段

```c
float data[10];
fwrite(&data[2], sizeof(float), 4, fp);
```

#### 結構

```c
struct item {
    short count;
    long total;
    char name[32];
} x;

fwrite(&x, sizeof(x), 1, fp);
```

### `fread` / `fwrite` 回傳值怎麼看

- 回傳的是成功讀/寫的 object 數
- 如果少於預期：
  - 讀取時可能是 EOF 或 error
  - 寫入時通常就是 error

### 最大坑：binary portability

直接把 struct 原封不動寫進檔案，常常只適合同一台機器 / 同一種 ABI。

原因至少兩個：

1. structure padding / alignment 可能不同
2. integer / floating-point 的 byte order 與表示法可能不同

所以跨平台交換資料時，通常要：

- 設計明確的 serialization format

## 5.10 Positioning a Stream

### 三組 API

書裡分三類：

1. `ftell` / `fseek`
2. `ftello` / `fseeko`
3. `fgetpos` / `fsetpos`

### `ftell` / `fseek`

```c
long ftell(FILE *fp);
int fseek(FILE *fp, long offset, int whence);
void rewind(FILE *fp);
```

`whence` 和 `lseek` 一樣：

- `SEEK_SET`
- `SEEK_CUR`
- `SEEK_END`

### `ftello` / `fseeko`

差別只是 offset 型別改成 `off_t`，比較適合 large file。

### `fgetpos` / `fsetpos`

它們用比較抽象的 `fpos_t`，對可攜性更友善。

### text file 的位置不是永遠等於 byte offset

在 UNIX 上通常很好理解，但標準特別提醒：

- 非 UNIX 系統的 text file 表示可能不同

所以對 text stream 做 positioning 時，最好遵守標準規則，不要過度假設。

## 5.11 Formatted I/O

這節其實很大，但核心就是兩家族：

- `printf` family
- `scanf` family

### Formatted output：`printf` 家族

```c
int printf(const char *restrict format, ...);
int fprintf(FILE *restrict fp, const char *restrict format, ...);
int dprintf(int fd, const char *restrict format, ...);
int sprintf(char *restrict buf, const char *restrict format, ...);
int snprintf(char *restrict buf, size_t n,
             const char *restrict format, ...);
```

### 什麼時候用哪個

- `printf`：寫到 `stdout`
- `fprintf`：寫到某個 stream
- `dprintf`：直接寫到 `fd`
- `sprintf`：寫進記憶體字串
- `snprintf`：安全版字串格式化

### `sprintf` 的風險

它不知道你的 buffer 多大，所以很容易 overflow。

幾乎所有實務程式都應優先考慮：

- `snprintf`

### `snprintf` 的重要語意

回傳值是：

- 如果 buffer 夠大，本來會寫入多少字元

所以你可以判斷是否截斷：

```c
#include <stdio.h>

int main(void) {
    char buf[8];
    int n = snprintf(buf, sizeof(buf), "value=%d", 123456);

    printf("buf = %s\n", buf);
    printf("needed = %d\n", n);
    return 0;
}
```

如果 `n >= sizeof(buf)`，代表內容被截斷了。

### format 的大致結構

```text
%[flags][fldwidth][precision][lenmodifier]convtype
```

你不一定要死背所有細節，但要知道幾件事：

- `precision` 對字串與數字意義不同
- `lenmodifier` 決定參數大小，例如 `l`, `ll`, `z`
- `convtype` 決定怎麼解讀參數，例如 `d`, `u`, `x`, `f`, `s`, `p`

### Formatted input：`scanf` 家族

```c
int scanf(const char *restrict format, ...);
int fscanf(FILE *restrict fp, const char *restrict format, ...);
int sscanf(const char *restrict buf, const char *restrict format, ...);
```

### `scanf` 家族為什麼常讓人怕

因為它很方便，但有很多隱藏細節：

- whitespace matching
- 部分成功後停止
- 容易漏掉 field width
- 遇到格式不合時很難穩定恢復

### 最重要的實務提醒

如果你要掃字串，一定要給寬度限制，否則也可能 overflow。

錯誤示範：

```c
char name[16];
scanf("%s", name);
```

較安全：

```c
char name[16];
scanf("%15s", name);
```

### `m` assignment-allocation

書裡也提到某些規格支援 `m`，讓 library 幫你配置 buffer。  
但這不是每個人最常用的做法，知道它存在即可。

## 5.12 Implementation Details

### `fileno`

```c
#include <stdio.h>

int fileno(FILE *fp);
```

它讓你把 `FILE *` 對應回底層的 `fd`。

這在以下情境很有用：

- 想搭配 `fcntl`
- 想 `dup`
- 想和只接受 `fd` 的 API 溝通

### 為什麼這節重要

因為它提醒你：

- `stdio` 不是魔法
- 它最終還是建在 Chapter 3 的 system calls 上

只是中間多了一層 library state 和 buffering 策略。

### 也因此會有混用風險

對同一個底層檔案，如果你同時混用：

- `read/write`
- `stdio`

但沒有處理好 buffer 與 offset，同步就可能亂掉。

## 5.13 Temporary Files

### `tmpnam` 與 `tmpfile`

```c
char *tmpnam(char *ptr);
FILE *tmpfile(void);
```

### `tmpnam` 的問題

它只先給你一個「目前看起來沒被用的名字」。  
但從：

- 名字產生
- 到你真的建立檔案

中間有一個 race window。

別的 process 可能搶先建了同名檔案。

### `tmpfile`

會直接建立一個 temporary file，通常在：

- `fclose`
- 或程式結束

時自動移除。

### 更推薦：`mkstemp` / `mkdtemp`

```c
#include <stdlib.h>

char *mkdtemp(char *template);
int mkstemp(char *template);
```

### 為什麼更好

因為它們把：

- 產生唯一名字
- 真的建立檔案 / 目錄

合成一個 atomic-ish 操作，沒有 `tmpnam` 那種空窗。

### 超重要坑：template 必須是可改的字串

這是書裡很經典的例子。

正確：

```c
char template[] = "/tmp/demoXXXXXX";
int fd = mkstemp(template);
```

錯誤：

```c
char *template = "/tmp/demoXXXXXX";
int fd = mkstemp(template);
```

第二種很可能 segfault，因為 string literal 通常在 read-only 區段，`mkstemp` 會修改那六個 `X`。

## 5.14 Memory Streams

這是很多人第一次讀 APUE 才注意到的好東西。

### `fmemopen`

```c
FILE *fmemopen(void *restrict buf, size_t size,
               const char *restrict type);
```

它讓你把一塊記憶體當成 stream 來操作。

### 什麼時候好用

- 想沿用 `fprintf` / `fgets` 這類 stream API
- 但不想真的碰磁碟
- 想在記憶體裡暫時組字串或測試 parser

### append 模式的特別語意

memory stream 在 append 模式下會找第一個 `'\0'` 當資料尾端。  
所以它不太適合任意 binary data。

### `open_memstream`

```c
FILE *open_memstream(char **bufp, size_t *sizep);
```

這個版本更像：

- library 幫你管理一個會自動成長的字串 buffer

好處是：

- 不用自己猜 buffer 大小
- 幾乎天然避免 buffer overflow

### 使用規則

- `fflush` 或 `fclose` 後，`bufp` / `sizep` 才有穩定可讀值
- 下一次寫入後，buffer 可能被 realloc，舊指標可能失效

## 5.15 Alternatives to Standard I/O

這節不是要你捨棄 `stdio`，而是提醒你：

- `stdio` 不是唯一選擇
- 也不是完美設計

書裡提到像：

- `fio`
- `sfio`
- `mmap` 類型介面

它們很多都是想減少資料拷貝，或讓 stream abstraction 更有彈性。

### 但 APUE 的重點不是要你改用別套 library

真正重點是：

- 了解 `stdio` 的優缺點
- 知道它是工程折衷

## 5.16 Summary

### 這章讀完你應該真的會的事

- 知道 `FILE *` 與 `fd` 的差異與連結
- 理解 fully buffered / line buffered / unbuffered
- 知道為什麼 `fflush` 不等於資料已落盤
- 會用 `fopen` / `fdopen` / `freopen`
- 會處理 `getc` 回傳 `EOF` 時的 `feof` / `ferror`
- 知道 `gets` 不能用、`sprintf` 風險很大
- 會用 `fread` / `fwrite` 做 binary I/O
- 知道 update stream 讀寫切換需要中介動作
- 知道 `mkstemp` 比 `tmpnam` 安全
- 理解 memory stream 適合哪些情境

### 這章最容易踩坑的地方

- 把 `fflush` 當成 `fsync`
- 用 `char` 接 `getc` 的回傳值
- 讀寫同一個 `r+` / `w+` stream 時忘了 `fflush` 或 `fseek`
- 在 `scanf("%s", buf)` 忘記寬度限制
- 直接用 `sprintf`
- 用 string literal 當 `mkstemp` template
- 混用 `stdio` 與底層 `read/write` 卻沒同步 buffer / offset

### 建議你現在立刻動手做

- 寫一個 `cat` 版本，分別用 `getc/putc`、`fgets/fputs`、`read/write` 比較
- 把 `stdout` 連到 terminal 與 regular file，觀察 buffering 差異
- 寫一個 `snprintf` 範例，故意讓 buffer 太小，觀察回傳值
- 用 `mkstemp` 建一個安全 temporary file
- 用 `open_memstream` 做一個動態字串 builder

### 一句總結

這章真正教你的，是 `stdio` 這層 abstraction 如何用 buffer、formatting、stream state 換來好寫與高效率，同時又帶來一套你必須尊重的語意規則。
