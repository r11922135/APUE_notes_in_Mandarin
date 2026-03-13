# Chapter 3 - File I/O

## 這章在做什麼

這一章是整本 APUE 最重要的地基之一。作者用它把 UNIX 最底層的檔案 I/O 介面完整講一遍，主線非常清楚：

- 怎麼取得 `file descriptor`
- 怎麼讀寫資料
- 怎麼調整 file offset
- 多個 process 怎麼共享同一個檔案
- 哪些操作需要 atomicity
- 怎麼用 `dup`、`fcntl`、`fsync` 這些工具把行為調整到正確

如果這章沒吃透，後面的 pipe、socket、daemon、terminal、IPC 都會變得很碎。

## 本章小節地圖

- `3.1 Introduction`
- `3.2 File Descriptors`
- `3.3 open and openat Functions`
- `3.4 creat Function`
- `3.5 close Function`
- `3.6 lseek Function`
- `3.7 read Function`
- `3.8 write Function`
- `3.9 I/O Efficiency`
- `3.10 File Sharing`
- `3.11 Atomic Operations`
- `3.12 dup and dup2 Functions`
- `3.13 sync, fsync, and fdatasync Functions`
- `3.14 fcntl Function`
- `3.15 ioctl Function`
- `3.16 /dev/fd`
- `3.17 Summary`

## 先抓住這章最重要的心智模型

### 第一層：`fd`

對 process 來說，檔案 I/O 的入口是 `file descriptor`。

### 第二層：open file state

對 kernel 來說，光知道 `fd` 還不夠，還有：

- 檔案目前 offset 在哪
- 是否 append
- 是否 nonblocking
- 是否 close-on-exec

### 第三層：檔案本體

真正的檔案 metadata 與內容又是另一層。

也就是說，這章真正要你建立的是：

- `fd`
- open file state
- file object

這三層不要混在一起。

## 3.1 Introduction

作者一開始就把重點說白了：

- UNIX 很多 file I/O 只靠 `open`、`read`、`write`、`lseek`、`close` 這五個函式就能做很多事

這些介面常被稱為 `unbuffered I/O`，意思是：

- 每次 `read` / `write` 都直接進 kernel 做 system call

它和 Chapter 5 的 `stdio` 差別就在這裡。

## 3.2 File Descriptors

### `fd` 是什麼

`file descriptor` 是 kernel 回給 process 的非負整數，用來表示一個已開啟的檔案或 I/O 對象。

這裡的「檔案」要廣義理解，它可能是：

- regular file
- directory
- terminal
- pipe
- socket

### 標準的 0、1、2

書裡再次提醒：

- `0` 對應 standard input
- `1` 對應 standard output
- `2` 對應 standard error

這是 shell 與程式共同遵守的約定，所以實務上幾乎可以視為基本常識。

### 請不要硬寫魔法數字

POSIX 已經幫你定義好：

- `STDIN_FILENO`
- `STDOUT_FILENO`
- `STDERR_FILENO`

可讀性比直接寫 `0`、`1`、`2` 好很多。

### `OPEN_MAX`

書中提到 descriptor 範圍理論上是：

- `0` 到 `OPEN_MAX - 1`

但現代系統上常常實際上受到：

- 記憶體
- 系統管理員設定的 soft/hard limit

影響，不再只是固定小數字。

## 3.3 `open` and `openat`

### 基本原型

```c
#include <fcntl.h>

int open(const char *path, int oflag, ... /* mode_t mode */);
int openat(int fd, const char *path, int oflag, ... /* mode_t mode */);
```

### `open` 的兩大類旗標

第一類是 access mode，五選一：

- `O_RDONLY`
- `O_WRONLY`
- `O_RDWR`
- `O_EXEC`
- `O_SEARCH`

第二類是額外行為旗標，例如：

- `O_APPEND`
- `O_CLOEXEC`
- `O_CREAT`
- `O_DIRECTORY`
- `O_EXCL`
- `O_NOCTTY`
- `O_NOFOLLOW`
- `O_NONBLOCK`
- `O_SYNC`
- `O_TRUNC`
- `O_DSYNC`
- `O_RSYNC`

### 一定要分清楚 access mode 與其他 flag

access mode 是「你打算怎麼開」。

其他 flag 是「附帶行為要怎麼做」。

例如：

```c
int fd = open("log.txt", O_WRONLY | O_CREAT | O_APPEND, 0644);
```

這段的意思是：

- 只寫
- 不存在就建立
- 每次寫都 append 到檔尾

### `O_CREAT` 時才需要 mode

這是 C 語法上很常見的變長參數例子。

只有在要建立新檔時，才需要再給一個 `mode_t`。

### `open` 回傳什麼

書裡特別強調：

- `open` 回傳的一定是「目前最小的可用 descriptor」

這件事很重要，因為很多 redirection 技巧就是建立在這個性質上。雖然後面 `dup2` 會提供更穩定的方法，但你得先知道這個基本規則。

### `openat` 為什麼存在

這章很漂亮的一點，是作者沒有把 `openat` 當成只是多一個參數的版本，而是直接講它解決兩個問題：

1. 多 thread 程式共享 current working directory，難以安全地各自在不同目錄工作
2. 避免某些 `TOCTTOU`（time-of-check to time-of-use）問題

### `openat` 的三種情況

- `path` 是 absolute pathname：`fd` 被忽略
- `path` 是 relative pathname，且 `fd` 是某個目錄 fd：從那個目錄開始解析
- `path` 是 relative pathname，且 `fd == AT_FDCWD`：等同從目前 working directory 開始

### `openat` 的實際感覺

你可以把它想成：

- `open` 是「從現在所在目錄去找」
- `openat` 是「從我指定的那個目錄把手去找」

這在安全性與多 thread 程式中都很有價值。

### 範例：用 `openat` 在特定目錄下開檔

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(void) {
    int dfd = open("/tmp", O_RDONLY | O_DIRECTORY);
    if (dfd < 0) {
        perror("open dir");
        return 1;
    }

    int fd = openat(dfd, "apue.log", O_WRONLY | O_CREAT | O_APPEND, 0644);
    if (fd < 0) {
        perror("openat");
        close(dfd);
        return 1;
    }

    write(fd, "hello\n", 6);
    close(fd);
    close(dfd);
    return 0;
}
```

### filename / pathname truncation

書裡還額外談到一個常被忽略的歷史問題：

- 某些舊系統對過長檔名會直接截斷
- 有些系統則回傳 `ENAMETOOLONG`

POSIX 用 `_POSIX_NO_TRUNC` 來描述這個行為差異。重點是：

- 你不能假設所有系統都對長檔名做同樣處理

## 3.4 `creat`

### `creat` 今天為什麼幾乎只是歷史名詞

書裡直接給出等價式：

```c
creat(path, mode)
```

等價於：

```c
open(path, O_WRONLY | O_CREAT | O_TRUNC, mode)
```

`creat` 的歷史背景是早期 `open` 還沒有現在這麼完整時，必須用另一個 system call 來建立新檔。

今天實務上通常直接用 `open`。

### `creat` 的缺點

它只能 write-only 開檔，所以如果你想建立後馬上讀，還得再關掉重開。

## 3.5 `close`

### `close` 不只是把 fd 整數丟掉

呼叫 `close(fd)` 代表：

- 這個 process 不再持有該 descriptor
- 相關資源引用計數可能減少
- 該 process 在該檔案上的 record lock 也會被釋放

### process 結束時 kernel 會自動 close

書裡提到很多小程式不會明寫 `close`，因為 process 結束時 kernel 會幫忙關掉所有 open files。

這句話的重點不是叫你以後都不寫 `close`，而是讓你知道：

- 檔案描述子的生命週期和 process 綁得很深

## 3.6 `lseek`

### current file offset

每個 open file 都有一個 current file offset。

`read` / `write` 通常都從這個位置開始，然後自動往後推進。

### `lseek` 在做什麼

```c
off_t lseek(int fd, off_t offset, int whence);
```

它不做 I/O，只是改變 kernel 記錄的當前 offset。

`whence` 常見三種：

- `SEEK_SET`
- `SEEK_CUR`
- `SEEK_END`

### `lseek` 的兩個常見用途

1. 查目前位置
2. 跳到指定位置或檔尾

例如查目前 offset：

```c
off_t pos = lseek(fd, 0, SEEK_CUR);
```

### 不是所有 `fd` 都能 seek

如果 `fd` 指向的是：

- pipe
- FIFO
- socket

那 `lseek` 會失敗，`errno` 通常設成 `ESPIPE`。

所以 `lseek` 其實也可以拿來測試某個 `fd` 是否可定位。

### hole 與 sparse file

這節最經典的內容之一，就是「你可以把 offset 跳過檔尾之後再寫」，這樣中間就形成 hole。

書裡強調：

- 中間沒寫過的區域讀回來會是 0
- 但 file system 不一定真的為那段配置實體磁碟區塊

這就是 sparse file。

### 範例：建立一個有 hole 的檔案

```c
#include <fcntl.h>
#include <unistd.h>

int main(void) {
    int fd = creat("file.hole", 0644);
    char buf1[] = "abcdefghij";
    char buf2[] = "ABCDEFGHIJ";

    write(fd, buf1, 10);
    lseek(fd, 16384, SEEK_SET);
    write(fd, buf2, 10);
    close(fd);
    return 0;
}
```

學這個例子不是為了炫技，而是要建立一個觀念：

- file size
- file offset
- disk block allocation

這三者不是同一件事。

### 大檔案與 `off_t`

書裡還談到 32-bit / 64-bit file offset 的問題，提醒你：

- `off_t` 大小不一定固定
- 某些平台需要特定 data model 或 feature macro 才能打開 64-bit offsets

所以寫程式時不要把 file offset 當成一般 `int`。

## 3.7 `read`

### 原型與回傳值

```c
ssize_t read(int fd, void *buf, size_t nbytes);
```

回傳值規則：

- `> 0`：實際讀到多少 bytes
- `0`：EOF
- `-1`：error

### 為什麼 `read` 常常不會讀滿你要求的大小

書裡列了幾種典型情況：

- regular file 快碰到 EOF
- terminal 一次通常讀到一行
- network buffering
- pipe / FIFO 目前只有部分資料
- record-oriented device 一次只給一個 record

這一節的真正重點是：

- `read(fd, buf, 4096)` 的意思是「最多讀 4096」
- 不是「保證給你 4096」

這個觀念後面做 network I/O 時尤其重要。

## 3.8 `write`

`write` 和 `read` 一樣，不要把它想成保證完整完成。

對 regular file，你常常會看到它一次寫完，但那不是所有情況下的語意保證。對 pipe、socket、nonblocking I/O 而言，更不能天真假設一次全部成功。

核心心法是：

- 每次都檢查回傳值
- 有需要就自己補寫剩下的部分

## 3.9 I/O Efficiency

這一節作者實際測不同 buffer size 的讀寫成本。你不用死背表格數字，但一定要理解結論：

- 一次讀寫太小，system call 次數暴增，效能很差
- 一次讀寫太大，也不一定有額外好處
- 常見合理值通常會接近 file system block size 或 page size

這章前面的 `BUFFSIZE = 4096` 並不是亂寫的，它在很多系統上都是很自然的起點。

### 這節最重要的效能直覺

- I/O 效率不只看你的 C 迴圈
- 更重要的是你進 kernel 幾次

## 3.10 File Sharing

這一節是 Chapter 3 最關鍵也最容易被新手忽略的觀念核心。

### kernel 內至少有三層資料結構

作者用概念圖說明 open file 的關係可以拆成三層：

1. process 自己的 file descriptor table
2. kernel 的 file table entry
3. v-node / i-node 這類代表檔案本體資訊的結構

### 為什麼這麼分很重要

因為：

- 兩個 process 各自 `open` 同一個檔案，通常會有不同 file table entry，但指到同一個檔案本體
- `dup` 或 `fork` 之後，可能多個 descriptor 共享同一個 file table entry

### 哪些狀態在 file table entry

書裡點出幾個關鍵：

- file status flags
- current file offset

所以如果兩個 descriptor 共用同一個 file table entry，它們就會共享：

- append 狀態
- nonblocking 狀態
- current offset

### file descriptor flags 與 file status flags 不同

這一點超重要：

- file descriptor flags 只屬於某一個 descriptor
- file status flags 屬於 file table entry，會影響所有共享它的 descriptor

這個差別到了 `fcntl` 會正式派上用場。

## 3.11 Atomic Operations

### 先想像 append log 的競爭情境

如果兩個 process 都做：

1. `lseek(fd, 0, SEEK_END)`
2. `write(fd, ...)`

那兩者之間可能被排程切開，結果就會互相覆寫。

### `O_APPEND` 的真正價值

`O_APPEND` 不是方便用而已，它的重點是：

- 「定位到檔尾 + 寫入」這件事由 kernel 原子地完成

也就是說，這個保證不能靠你自己用兩個 function call 拼出來。

### `pread` / `pwrite`

書裡接著介紹：

- `pread`
- `pwrite`

它們的意義是把：

- 指定 offset
- 執行 I/O

合成一次完成，而且不改變原本 current file offset。

這在多 thread / 多 descriptor 邏輯裡很好用。

### `O_CREAT | O_EXCL`

這又是另一個 atomicity 範例。

如果你用：

- 先檢查檔案不存在
- 再 `creat`

中間仍然可能被別人插隊。

`open(path, O_CREAT | O_EXCL, mode)` 的價值在於：

- 檢查與建立是一個原子操作

## 3.12 `dup` and `dup2`

### 它們不是複製檔案內容

`dup` / `dup2` 做的是：

- 複製一個 descriptor

而且新舊 descriptor 會指向同一個 file table entry。

因此它們共享：

- current file offset
- file status flags

### `dup`

- 回傳最小可用的新 descriptor

### `dup2`

- 直接把目標 descriptor 變成你指定的數字
- 若目標本來已開，先關掉
- 但整個動作是 atomic 的

書裡特別強調：

- `dup2` 不是單純 `close(fd2); fcntl(fd, F_DUPFD, fd2);`

因為那樣中間會有 race window。

### 最常見用途：redirection

```c
int fd = open("out.txt", O_WRONLY | O_CREAT | O_TRUNC, 0644);
dup2(fd, STDOUT_FILENO);
close(fd);
```

這樣之後寫到 `stdout` 的東西就會流到 `out.txt`。

## 3.13 `sync`, `fsync`, `fdatasync`

### delayed write 是什麼

書裡先提醒你，普通 `write` 通常不是資料立刻落盤，而是：

- kernel 先把資料收進 buffer cache / page cache
- 稍後再寫到 disk

這叫 delayed write。

### 三個函式的分工

- `sync()`：把所有修改過的 block 排進寫回佇列，但不等全部真正寫完
- `fsync(fd)`：只針對指定檔案，並且等寫回完成
- `fdatasync(fd)`：也等指定檔案資料寫完，但主要關注 data，metadata 只處理必要部分

### 你應該怎麼記

- `sync`：整體系統層面，廣而不等
- `fsync`：單一檔案，等到真的完成
- `fdatasync`：單一檔案，但只同步必要資料面

這三個對資料庫、log durability、crash consistency 都非常重要。

## 3.14 `fcntl`

### 這是 Chapter 3 的萬用調整器

`fcntl` 很強，書裡把它的用途整理成五大類：

1. duplicate descriptor
2. get/set file descriptor flags
3. get/set file status flags
4. get/set async I/O ownership
5. get/set record locks

### 最常用的幾個 command

- `F_DUPFD`
- `F_DUPFD_CLOEXEC`
- `F_GETFD`
- `F_SETFD`
- `F_GETFL`
- `F_SETFL`

### file descriptor flags

目前最重要的代表就是：

- `FD_CLOEXEC`

它控制的是：

- process `exec` 之後，這個 descriptor 要不要自動關掉

### file status flags

像這些屬於 file status flags：

- `O_APPEND`
- `O_NONBLOCK`
- `O_SYNC`
- `O_DSYNC`

它們是跟 file table entry 綁在一起，不是只跟單一 descriptor 綁。

### `F_SETFL` 不是什麼都能改

書裡特別指出：

- 不是所有 open flag 都能在開檔後隨便改

通常能改的是像：

- `O_APPEND`
- `O_NONBLOCK`
- 某些 sync 相關旗標

### 好用範例：把某個 descriptor 設成 append 或 nonblocking

```c
#include <fcntl.h>
#include <stdio.h>

int set_fl(int fd, int flags) {
    int val = fcntl(fd, F_GETFL, 0);
    if (val < 0)
        return -1;

    val |= flags;
    return fcntl(fd, F_SETFL, val);
}
```

這段程式的重點不是語法，而是流程：

1. 先取舊值
2. 修改你想動的 bit
3. 再設回去

不可以直接亂覆蓋，否則可能把原本已開啟的其他 flag 關掉。

## 3.15 `ioctl`

`ioctl` 是 UNIX 很典型的 catchall 介面。

當某些 I/O 行為很難塞進：

- `read`
- `write`
- `lseek`
- `fcntl`

這些既有模型時，很多裝置控制就會放到 `ioctl`。

書裡強調 terminal I/O 過去非常依賴它。你到 Chapter 18、19 還會再看到它。

理解上你只要先記住：

- `ioctl` 很常是 device-specific
- 它不是一般檔案 I/O 的主路徑

## 3.16 `/dev/fd`

### 它的概念很漂亮

打開 `/dev/fd/n`，概念上就像 duplicate 現有的 descriptor `n`。

例如：

```c
open("/dev/fd/0", mode)
```

在很多系統上就接近：

```c
dup(0)
```

### 為什麼它有用

它讓原本只接受 pathname 的程式，也能自然地處理標準輸入輸出。

例如 shell 可把 `/dev/fd/0` 當成一個正常 pathname 傳給程式，而不是要求程式額外發明 `-` 這種特殊語法。

### 書裡提醒的 Linux 例外

Linux 的 `/dev/fd` 實作比較像 symbolic link 到底層真實檔案，所以某些行為和其他 UNIX 系統不完全一樣。這再次提醒你：

- 就算概念相同，平台細節仍可能不同

## 3.17 Summary

### 這章讀完你應該真的會的事

- 知道 `fd` 是 process 眼中的 I/O 把手
- 知道 `open/read/write/lseek/close` 的基本語意
- 理解 short read / short write 是正常世界的一部分
- 理解 `O_APPEND`、`O_CREAT|O_EXCL` 這種 atomicity 設計的必要性
- 理解 `dup` / `dup2` 與 file sharing 的關係
- 會區分 `sync`、`fsync`、`fdatasync`
- 知道 `fcntl` 管的是 descriptor 層與 open-file 層的旗標

### 這章最容易搞混的四件事

- `fd` 不等於檔案本體
- `lseek` 只是改 offset，不會自己做 I/O
- `dup` 不是複製資料，而是共享 open file state
- `write` 回來不等於資料一定已落盤

### 建議你現在立刻做的事

- 自己手打一個 `cp` 小程式，用 `read/write` 完成。
- 自己做一個 sparse file，觀察檔案大小與實際磁碟占用差異。
- 用 `dup2` 把 `stdout` 重導到檔案。
- 練習用 `fcntl` 打開 `O_NONBLOCK`。

### 一句總結

這章真正教你的，不只是幾個 I/O 函式怎麼呼叫，而是 UNIX 檔案 I/O 的行為模型。這個模型一旦清楚，後面你再看 pipe、socket、terminal、IPC，很多東西都會變得很自然。
