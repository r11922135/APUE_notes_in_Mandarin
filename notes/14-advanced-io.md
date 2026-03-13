# Chapter 14 - Advanced I/O

## 這章在做什麼

這章把前面「基本 I/O」推進到真正寫系統程式會碰到的層次。  
如果 Chapter 3 教你的是：

- `open`
- `read`
- `write`
- `lseek`
- `close`

那 Chapter 14 教你的就是：

- 當單純 blocking `read/write` 已經不夠時，還有哪些工具

APUE 把這些工具統稱為 `advanced I/O`，包括：

- nonblocking I/O
- record locking
- I/O multiplexing
- asynchronous I/O
- `readv` / `writev`
- `readn` / `writen`
- memory-mapped I/O (`mmap`)

這章內容很多，而且主題看起來分散，但其實都圍繞同一件事：

- 如何在更複雜的 I/O 條件下，讓程式保持正確、有效率、而且可控制

## 本章小節地圖

- `14.1 Introduction`
- `14.2 Nonblocking I/O`
- `14.3 Record Locking`
- `14.4 I/O Multiplexing`
- `14.5 Asynchronous I/O`
- `14.6 readv and writev Functions`
- `14.7 readn and writen Functions`
- `14.8 Memory-Mapped I/O`
- `14.9 Summary`

## 先抓住這章最重要的心智模型

### 不是所有 I/O 問題都該用同一種工具

這章最大的陷阱就是：

- 看很多 API，卻不知道各自解什麼問題

你可以先這樣分：

- 不想一直卡住：`O_NONBLOCK`
- 想同時等很多 fd：`select` / `pselect` / `poll`
- 想鎖檔案區段：`fcntl record locking`
- 想真的排隊做 background I/O：`aio_*`
- 想一次處理多個 buffer：`readv` / `writev`
- 想保證讀滿 / 寫滿指定 byte 數：`readn` / `writen`
- 想把檔案當記憶體用：`mmap`

### advanced I/O 的重點不只是 API，而是語意細節

像：

- short read / short write
- blocking vs nonblocking
- EOF vs HUP
- advisory lock vs mandatory lock
- `close` 對 lock 的影響
- `select` 修改參數集合
- `mmap` 並不等於立刻寫回檔案

這些都是最容易讓程式「看起來能跑，實際上有 bug」的地方。

## 14.1 Introduction

這章一開始就很直接地說：

- 下面會講很多看似不同的 I/O 技術

它們之所以被放在同一章，是因為後面：

- IPC
- network programming
- server design
- database-style file access

都會用到它們。

換句話說，這章很像後面幾章的工具箱準備。

## 14.2 Nonblocking I/O

### 什麼是 `slow system call`

APUE 在 Chapter 10 講過：

- 有些 system call 可能 block 很久，甚至理論上可永遠不回來

例如：

- 從 pipe / terminal / network 讀資料，但現在沒資料
- 對 pipe / socket 寫資料，但現在對方緩衝區滿
- 開某些特殊裝置或 FIFO
- 某些 `ioctl`
- 某些 IPC 操作

這些就是 nonblocking I/O 想解的問題背景。

### nonblocking I/O 在解什麼

它的意思不是「I/O 變快」，而是：

- 如果現在做不到，就不要讓我無限等
- 立刻回來告訴我：現在會 block

### 設定方式

有兩種常見做法：

1. `open` 時直接加 `O_NONBLOCK`
2. 對既有 descriptor 用 `fcntl` 打開 `O_NONBLOCK`

範例：

```c
#include <fcntl.h>
#include <unistd.h>

int set_nonblock(int fd)
{
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags < 0)
        return -1;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}
```

### nonblocking 失敗時常見回傳

最常看到的是：

- `EAGAIN`
- `EWOULDBLOCK`

在現代 POSIX 系統裡，兩者往往等價或至少語意非常接近。  
白話理解就是：

- 現在不行，稍後再來

### partial write / partial read 是常態，不是例外

一旦 nonblocking 打開，就必須很習慣這類情況：

- 你想寫 10,000 bytes
- 這次只寫進去 800 bytes
- 剩下要之後再補

同理，read 也可能拿到：

- 少於你要求的 bytes

### polling 為什麼差

APUE 用一個例子說明：

- 你一直 `write`
- 一直拿到 `EAGAIN`
- 然後緊接著再試

這叫 polling。  
它的問題是：

- 很浪費 CPU

所以 nonblocking 通常真正合理的搭配不是裸迴圈，而是：

- `select` / `poll`
- 或 event loop

### thread 有時能替代 nonblocking，但不總是最好

APUE 也提醒一件很成熟的事：

- 有時候與其把所有 fd 都設 nonblocking，不如直接讓 thread block

這在某些程式裡會讓設計更簡單。  
但 thread 也會帶來同步成本，所以沒有一招通吃。

## 14.3 Record Locking

### 這章的「record」其實是 byte-range locking

APUE 很直接地說：

- UNIX kernel 根本不知道你檔案裡的「record」是什麼

所以 record locking 更精確的說法應該是：

- byte-range locking

你鎖的是：

- 某個 byte 範圍

不是資料庫意義上的一筆 record。

### 為什麼需要 record locking

如果多個 process 同時改同一個檔案，最後結果通常就是：

- 最後寫的人贏

但像資料庫、索引檔、spool file 這類情境，常需要：

- 讀某段資料時不想被別人改
- 改某段資料時不想被別人同時碰

這就是 record locking 的用途。

### POSIX 標準化的是 `fcntl` 版本

雖然歷史上有：

- `flock`
- `lockf`

但 APUE 這裡的主角是：

- `fcntl` record locking

### `struct flock`

```c
struct flock {
    short l_type;
    short l_whence;
    off_t l_start;
    off_t l_len;
    pid_t l_pid;
};
```

幾個欄位的意義：

- `l_type`：`F_RDLCK`、`F_WRLCK`、`F_UNLCK`
- `l_whence`：`SEEK_SET`、`SEEK_CUR`、`SEEK_END`
- `l_start`：起始 offset
- `l_len`：長度；若為 `0` 表示一直到 EOF
- `l_pid`：`F_GETLK` 時回報誰擋住你

### 三個核心指令

- `F_GETLK`
- `F_SETLK`
- `F_SETLKW`

#### `F_GETLK`

- 問現在這把鎖會不會被別人擋住

如果不會：

- `l_type` 會被設成 `F_UNLCK`

如果會：

- 結構內容會被覆蓋成那把阻擋你的 lock 資訊

#### `F_SETLK`

- 嘗試設鎖
- 如果目前不相容，立即失敗回來

常見錯誤碼：

- `EACCES`
- `EAGAIN`

#### `F_SETLKW`

- `W` 是 wait
- 如果目前不相容，就睡著等到可用或被 signal 打斷

### read lock 與 write lock 的相容性

基本規則：

- 多個 process 可以同時持有 read lock
- write lock 必須獨佔
- 有 write lock 時，別人不能再拿 read 或 write

但這個相容性規則只用在：

- 不同 process 之間

同一個 process 對同一範圍重設 lock，通常是：

- 新的取代舊的

### lock 範圍的幾個規則

- 可超過目前 EOF 往後延伸
- 不能延伸到檔案開頭之前
- `l_len == 0` 表示到 EOF 甚至未來追加內容
- 要鎖整個檔案時，常用 `SEEK_SET + start=0 + len=0`

### 一組很實用的 helper

```c
int lock_reg(int fd, int cmd, int type, off_t offset, int whence, off_t len)
{
    struct flock lock;

    lock.l_type = type;
    lock.l_start = offset;
    lock.l_whence = whence;
    lock.l_len = len;
    return fcntl(fd, cmd, &lock);
}

#define read_lock(fd, offset, whence, len) \
    lock_reg((fd), F_SETLK, F_RDLCK, (offset), (whence), (len))
#define write_lock(fd, offset, whence, len) \
    lock_reg((fd), F_SETLK, F_WRLCK, (offset), (whence), (len))
#define writew_lock(fd, offset, whence, len) \
    lock_reg((fd), F_SETLKW, F_WRLCK, (offset), (whence), (len))
#define un_lock(fd, offset, whence, len) \
    lock_reg((fd), F_SETLK, F_UNLCK, (offset), (whence), (len))
```

### `F_GETLK` + `F_SETLK` 不是原子操作

這是很重要的 race。

你不能這樣想：

1. 先 `F_GETLK` 看沒人鎖
2. 再 `F_SETLK`
3. 所以就安全

因為在兩次呼叫中間，別的 process 完全可以先插進來拿走鎖。

### deadlock detection

如果兩個 process 各拿一把鎖，又用 `F_SETLKW` 去等彼此那一把，就可能 deadlock。  
某些實作會偵測到這種情況並回類似：

- `EDEADLK`

APUE 的範例裡就有：

- parent 鎖 byte 1
- child 鎖 byte 0
- 彼此再去等對方持有的 byte

最後 kernel 會挑一邊報 deadlock。

### 三條非常重要的 lock 生命週期規則

這是整節最容易考也最容易踩坑的地方。

#### 1. lock 是跟「process + file」綁在一起

不是跟某個特定 fd 綁在一起。  
這導致一個很反直覺的結果：

- 只要同一 process 裡有任何 referencing same file 的 descriptor 被 `close`
- 該 process 在那個 file 上的 locks 就可能被釋放

也就是說，下面這種情況非常危險：

```c
fd1 = open(path, O_RDWR);
read_lock(fd1, 0, SEEK_SET, 1);
fd2 = dup(fd1);
close(fd2);   /* 可能把 lock 也弄掉 */
```

#### 2. lock 不會跨 `fork` 繼承

parent 拿到的 `fcntl` record lock，child 不會自動繼承。  
這很合理，因為 lock 的存在本來就是為了防不同 process 互撞。

#### 3. lock 會跨 `exec` 保留

如果 descriptor 還活著，record locks 會跟著保留到 `exec` 後的新程式。  
但如果該 fd 設了 `FD_CLOEXEC`，exec 時 descriptor 被關掉，對應 lock 也會釋放。

### EOF-relative lock 很容易出錯

APUE 特別提醒：

- 用 `SEEK_END` 鎖與解鎖時要小心

因為 kernel 會把 lock 範圍轉成當下的 absolute offset 記住。  
所以你以為自己在解開「從 EOF 開始往後的鎖」，實際上可能只是在解某個當時位置之後的範圍，而留下你剛 append 的那個 byte 仍被鎖住。

### advisory vs mandatory

#### advisory locking

這是最常見也最推薦理解的模式：

- 只有「合作中的程式」會遵守鎖
- 不合作的程式還是能直接寫

這聽起來像很弱，但在很多系統裡已足夠。  
前提是：

- 所有正規存取路徑都走同一套規則

#### mandatory locking

這模式更強：

- kernel 真的會在 read / write / 某些 open 路徑檢查鎖

但 APUE 對它的態度其實很保留，因為：

- 可攜性差
- 行為複雜
- 容易出現意料之外的互動

Linux / Solaris 有支援，但不是 SUS 的核心標準要求。

### mandatory locking 的重要觀察

- 啟用方式通常和檔案 mode bits 有關
- blocking 與 nonblocking descriptor 反應不同
- 某些工具甚至可以繞過它，例如「寫臨時檔再 rename」這類做法

所以 APUE 的暗示很明確：

- 除非你非常清楚需求，否則不要迷信 mandatory locking

### 和 daemon chapter 的連結

Chapter 13 的 single-instance daemon，正是用：

- 對整個 pid/lock file 加 write lock

來做 mutual exclusion。  
這就是 record locking 最常見也最實用的應用之一。

## 14.4 I/O Multiplexing

### 為什麼需要 I/O multiplexing

如果你的程式只處理一個 descriptor，簡單 blocking `read/write` 就很好。  
但一旦你同時要關心很多個 fd，例如：

- terminal input
- network socket
- pipe

你就不能天真地在其中一個上無限 block。

APUE 用 `telnet` 的例子講這件事，非常經典：

- 一邊要看 terminal
- 一邊要看 network

你不可能只顧其中一邊。

### 幾個可能解法

APUE 先把其他解法都拿出來比較：

- `fork` 成兩個 process：可行，但生命週期管理變麻煩
- 用兩個 threads：可行，但要同步
- nonblocking + polling：可行，但浪費 CPU
- signal-based async I/O：可行，但限制很多

最後才推出更好的通用工具：

- I/O multiplexing

也就是：

- 把一堆 fd 丟給 kernel
- 問它：哪幾個現在 ready

### 14.4.1 `select` and `pselect` Functions

#### `select`

```c
#include <sys/select.h>

int select(int maxfdp1, fd_set *restrict readfds,
           fd_set *restrict writefds, fd_set *restrict exceptfds,
           struct timeval *restrict tvptr);
```

### `select` 讓你一次問三種事情

- 哪些 fd 可讀
- 哪些 fd 可寫
- 哪些 fd 有 exception condition

### timeout 三種模式

#### `tvptr == NULL`

- 無限等

#### `tv_sec == 0 && tv_usec == 0`

- 完全不等，立刻回來
- 這相當於 poll 一下狀態

#### 其他正值

- 最多等指定時間

如果 timeout 到了還沒 ready：

- 回 `0`

### `fd_set` 的基本操作

```c
FD_ZERO(&set);
FD_SET(fd, &set);
FD_CLR(fd, &set);
FD_ISSET(fd, &set);
```

典型寫法：

```c
fd_set rset;
FD_ZERO(&rset);
FD_SET(STDIN_FILENO, &rset);
FD_SET(sockfd, &rset);

if (select(sockfd + 1, &rset, NULL, NULL, NULL) < 0)
    /* handle error */;
```

### `maxfdp1` 是什麼

它不是「fd 個數」，而是：

- 你要檢查的最大 fd 編號 + 1

所以如果最高 fd 是 `7`，那就傳：

- `8`

### `select` 會修改你的集合

這是新手超容易忘的點。

call 前：

- `readfds` 表示你「想檢查誰」

call 後：

- `readfds` 只留下「真的 ready 的那些」

所以如果你在 loop 裡反覆用 `select`，每次都必須重新初始化集合。

### return value 怎麼看

- `-1`：錯誤，例如被 signal 打斷
- `0`：timeout
- `>0`：ready descriptor 的數量總和

### 什麼叫 ready

- 在 read set 裡：`read` 不會 block
- 在 write set 裡：`write` 不會 block
- 在 exception set 裡：有例外條件，例如 out-of-band data

### 幾個很重要的小細節

#### regular file 幾乎總是 ready

對 regular file 而言，`select` 通常很快就說：

- 可讀
- 可寫
- exception 也沒什麼特別可等

所以 `select` 真正最常用在：

- terminal
- pipe
- socket

#### EOF 會讓 fd 被視為 readable

這非常重要。  
很多人誤以為 EOF 會出現在 exception set，錯。

正確是：

- `select` 說它可讀
- 你去 `read`
- `read` 回 `0`

這才是 EOF。

#### fd 自己是不是 nonblocking，不影響 `select` 會不會等

就算 fd 是 nonblocking：

- `select` 仍然可以正常 block 等待 ready

這兩件事是不同層次。

### `pselect`

```c
int pselect(int maxfdp1, fd_set *restrict readfds,
            fd_set *restrict writefds, fd_set *restrict exceptfds,
            const struct timespec *restrict tsptr,
            const sigset_t *restrict sigmask);
```

`pselect` 相對 `select` 的兩個重要差異：

1. timeout 用 `timespec`，精度更高
2. 可在等待時原子地暫時切換 signal mask

第二點特別重要，因為它可以避免：

- 解除 signal block 與進入等待之間的 race

### 14.4.2 `poll` Function

```c
#include <poll.h>

int poll(struct pollfd fdarray[], nfds_t nfds, int timeout);
```

`poll` 和 `select` 解的是同一類問題，但介面不同。

### `struct pollfd`

```c
struct pollfd {
    int   fd;
    short events;
    short revents;
};
```

這比 `fd_set` 更直觀，因為：

- 每個陣列元素直接對應一個 fd
- `events` 表示你想等什麼
- `revents` 表示實際發生什麼

### 常見 `events`

- `POLLIN`
- `POLLOUT`
- `POLLPRI`

### 常見 `revents`

- `POLLIN`
- `POLLOUT`
- `POLLERR`
- `POLLHUP`
- `POLLNVAL`

其中最後三個很重要，因為它們可能在你沒特別要求時也會回來。

### `poll` 和 `select` 的一個大差異

`poll` 不會像 `select` 那樣改掉你「想監看的條件」。  
也就是：

- `events` 不會被內核改寫
- 結果只放進 `revents`

這讓 loop 寫起來通常比較直。

### timeout 語意

- `-1`：無限等
- `0`：完全不等
- `>0`：等指定毫秒數

### `POLLHUP` 與 EOF 不一樣

這是容易搞混的點。

- EOF 是 read 回 `0`
- `POLLHUP` 表示對端掛掉 / hangup

而且即使出現 `POLLHUP`，有時候 descriptor 上仍可能還有資料可讀。

### `select` / `poll` 被 signal 打斷

這兩個 API 都可能因為收到 signal 而提早返回：

- `-1`
- `errno == EINTR`

所以你的 loop 設計要考慮 signal interruption。

## 14.5 Asynchronous I/O

### synchronous notification vs asynchronous notification

`select` / `poll` 本質上是：

- 你主動去問 kernel

這叫同步式等待。  
而 asynchronous I/O 的想法則是：

- 先把工作交出去
- 等 kernel 主動通知你完成

### 但 AIO 並不是免費午餐

APUE 對 AIO 很誠實：

- 它很強
- 但介面複雜度高

你會多出很多狀態管理：

- 提交操作成功沒
- 真正 I/O 成功沒
- 查詢完成狀態時又有沒有別的錯

所以 AIO 並不一定比 threads 或 `select/poll` 更簡單。

### 14.5.1 System V Asynchronous I/O

這套是舊世界做法，主要和：

- STREAMS
- `SIGPOLL`

有關。

它的限制很大，現代一般學習上只要知道：

- 歷史上存在
- 範圍受限
- 不適合作為通用可攜方案

### 14.5.2 BSD Asynchronous I/O

BSD 系統傳統上用：

- `SIGIO`
- `SIGURG`

來通知 descriptor 狀態。

典型步驟：

1. 裝 `SIGIO` handler
2. 用 `fcntl(..., F_SETOWN, ...)` 指定誰收 signal
3. 用 `fcntl(..., F_SETFL, O_ASYNC)` 啟用非同步通知

問題是：

- 只適合 terminal / network 之類特定 fd
- 同一個 process 的 signal 數量有限
- 如果很多 fd 都開 async I/O，光靠 signal 本身不夠告訴你是哪個 fd ready

### 14.5.3 POSIX Asynchronous I/O

這才是現代標準化主角。  
它的核心物件是：

- `struct aiocb`

```c
struct aiocb {
    int             aio_fildes;
    off_t           aio_offset;
    volatile void  *aio_buf;
    size_t          aio_nbytes;
    int             aio_reqprio;
    struct sigevent aio_sigevent;
    int             aio_lio_opcode;
};
```

### 幾個關鍵欄位

- `aio_fildes`：哪個 fd
- `aio_offset`：從哪個 offset 開始
- `aio_buf`：buffer
- `aio_nbytes`：大小
- `aio_sigevent`：完成通知方式

### AIO 很重要的一個特性：顯式 offset

POSIX AIO 不會去動一般 file offset。  
這很好，因為：

- 不容易和其他 conventional I/O 混成一團

但也意味著：

- 你要自己明確指定每次 I/O 的 offset

### 通知完成的方式：`sigevent`

常見幾種：

- `SIGEV_NONE`：完成了也不通知
- `SIGEV_SIGNAL`：送 signal
- `SIGEV_THREAD`：開一條 detached thread 執行 callback

### 基本操作

```c
int aio_read(struct aiocb *aiocb);
int aio_write(struct aiocb *aiocb);
```

這兩個函式回成功時，只代表：

- request 已成功排進系統

不代表：

- 真正 I/O 已完成且成功

### 查狀態與取結果

```c
int aio_error(const struct aiocb *aiocb);
ssize_t aio_return(const struct aiocb *aiocb);
```

`aio_error` 可能告訴你：

- `0`：已成功完成
- `EINPROGRESS`：還沒完成
- 其他 error code：I/O 本身失敗

如果確定完成，再用：

- `aio_return`

去取真正結果。

### 很重要的生命週期規則

在 AIO 完成前：

- `aiocb` 結構不可亂動
- buffer 內容不可被重用或釋放

不然你就是在讓 kernel 對一塊可能已失效的記憶體做 I/O。

### `aio_suspend`

如果你手上掛了一堆 pending AIO，最後想等其中一個完成，可以用：

```c
int aio_suspend(const struct aiocb *const list[], int nent,
                const struct timespec *timeout);
```

它相當於：

- AIO 世界裡的等待點

### `aio_cancel`

```c
int aio_cancel(int fd, struct aiocb *aiocb);
```

它只是「嘗試取消」，不是保證取消。  
回傳可能是：

- `AIO_ALLDONE`
- `AIO_CANCELED`
- `AIO_NOTCANCELED`

### `lio_listio`

如果你想一次提交多個 AIO request：

```c
int lio_listio(int mode, struct aiocb *restrict const list[restrict],
               int nent, struct sigevent *restrict sigev);
```

模式可分：

- `LIO_WAIT`
- `LIO_NOWAIT`

這能讓你做 batch 式 I/O 佈局。

### 這節的務實結論

POSIX AIO 功能完整，但使用成本高。  
真實工程上你要權衡：

- 複雜度
- 可攜性
- 你是不是真的需要它

## 14.6 `readv` and `writev` Functions

### 這兩個函式在解什麼

它們解的是：

- 資料不在一塊連續 buffer 裡
- 但你又不想自己先 copy 拼成一塊

### scatter read / gather write

```c
#include <sys/uio.h>

ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```

搭配：

```c
struct iovec {
    void   *iov_base;
    size_t  iov_len;
};
```

### `writev`

- 從多個 buffers 依序 gather 起來寫出去

### `readv`

- 把讀到的資料依序 scatter 到多個 buffers

### 典型用途

例如你要送：

- 一個固定格式 header
- 加上一塊 payload

與其：

1. 先 `malloc`
2. 再 `memcpy` header
3. 再 `memcpy` payload
4. 再 `write`

不如直接：

```c
struct iovec iov[2];

iov[0].iov_base = header;
iov[0].iov_len  = header_len;
iov[1].iov_base = body;
iov[1].iov_len  = body_len;

writev(fd, iov, 2);
```

### 為什麼它有價值

- 減少 system call 次數
- 可能減少不必要的 user-space buffer copy
- 對 protocol 封包、log record、database page 組裝很有用

## 14.7 `readn` and `writen` Functions

### 為什麼需要這兩個 helper

對 pipe、FIFO、terminal、network 這類 descriptor：

- `read` 可能少讀
- `write` 可能少寫

這不一定是錯，而是正常語意。

如果你的協定或程式邏輯要求：

- 一次就是要讀滿 `N` bytes
- 或寫滿 `N` bytes

那你就需要在上層自己 loop。

### `readn` / `writen` 的概念

它們不是標準函式，而是 APUE 自己包的實用 helper：

```c
ssize_t readn(int fd, void *buf, size_t nbytes);
ssize_t writen(int fd, const void *buf, size_t nbytes);
```

### `readn`

- 一直 `read`
- 直到湊滿 `nbytes`
- 或遇到錯誤
- 或遇到 EOF

### `writen`

- 一直 `write`
- 直到把指定 bytes 全送出去
- 或遇到錯誤

### 為什麼它們很常見

因為很多 binary protocol 或固定長度封包都需要：

- 精確收 / 發固定 byte 數

這種情境你不能只呼叫一次 `read` 或 `write` 就假設事情完成。

### 一個簡化實作

```c
ssize_t writen(int fd, const void *buf, size_t n)
{
    size_t nleft = n;
    const char *ptr = buf;
    ssize_t nwritten;

    while (nleft > 0) {
        nwritten = write(fd, ptr, nleft);
        if (nwritten <= 0)
            return (nleft == n) ? -1 : (ssize_t)(n - nleft);
        nleft -= nwritten;
        ptr += nwritten;
    }
    return (ssize_t)n;
}
```

### 什麼時候適合用

- 你真的知道對方應該傳多少 bytes
- 你的資料格式是 fixed-size 或有長度欄位

如果根本不知道總長度，硬用 `readn` 反而可能把自己卡住。

## 14.8 Memory-Mapped I/O

### `mmap` 的核心概念

`mmap` 讓你把檔案的一段映射到 process address space。  
之後你對那塊記憶體做：

- 讀
- 寫

就像在碰普通 memory，但背後其實是在讀寫檔案。

這是這章最「不像傳統 I/O」的一招。

### 基本 API

```c
#include <sys/mman.h>

void *mmap(void *addr, size_t len, int prot, int flag, int fd, off_t off);
```

失敗時回：

- `MAP_FAILED`

### 幾個參數怎麼理解

- `addr`：想映射到哪；通常傳 `0` 讓系統選
- `len`：映射長度
- `prot`：保護模式
- `flag`：共享 / 私有等行為
- `fd`：哪個檔案
- `off`：從檔案哪個 offset 開始

### `prot`

常見值：

- `PROT_READ`
- `PROT_WRITE`
- `PROT_EXEC`
- `PROT_NONE`

注意：

- 你不能要求比檔案 open mode 更多的權限

例如 fd 是唯讀開的，就不能要求 `PROT_WRITE`。

### `MAP_SHARED` vs `MAP_PRIVATE`

#### `MAP_SHARED`

- 你對 mapped memory 的修改，會反映到 underlying file

#### `MAP_PRIVATE`

- 採 copy-on-write
- 你改的是 private copy，不會回寫原檔

這兩者的差別非常重要。

### `MAP_FIXED`

可要求映射必須在某個指定地址，但可攜性與安全性都比較差。  
一般情況：

- 不要主動用
- 讓系統決定比較穩

### offset 與 page size 對齊

很多系統要求：

- `off` 要 page-aligned
- 若指定 `addr`，它也要 page-aligned

所以 `mmap` 不是任意 byte offset 都能隨便對。

### 檔案尾端與映射大小的陷阱

如果檔案大小不是 page size 的整數倍，系統常會把最後一頁多出來的部分補零映射給你。  
但這些超出原檔範圍的 bytes：

- 不是讓你直接拿來 append 檔案的

換句話說：

- 你不能靠「多寫出 mapped 區尾端一點點」來安全擴張檔案

### 想 map 一個更大的輸出檔，先 `ftruncate`

這是 APUE file copy 範例的關鍵。

如果你要把 output file map 成可寫區域，卻沒有先把檔案長度設到足夠大，  
第一次寫進去時，可能直接吃：

- `SIGBUS`

正確作法：

```c
ftruncate(fdout, desired_size);
```

然後再 `mmap`。

### `SIGSEGV` 與 `SIGBUS`

這兩個和 `mmap` 很有關。

#### `SIGSEGV`

通常表示：

- 你碰了不該碰的 mapping 權限
- 例如對只讀 mapping 寫入

#### `SIGBUS`

通常表示：

- 你碰的 mapped 區對應到底層檔案狀態已經不合理
- 例如檔案被截短後，你還去碰尾端那段 mapping

### `fork` 與 `exec` 的互動

- `fork` 會繼承 mapping
- `exec` 不會繼承舊 mapping

因為 mapping 本來就是 address space 的一部分。

### `mprotect`

```c
int mprotect(void *addr, size_t len, int prot);
```

可在既有 mapping 上改保護屬性。  
例如某些情況你想先 read-only，再切 writable。

### `msync`

`MAP_SHARED` 的修改不保證立刻寫回檔案。  
kernel 常會延後、批次、按 page 寫回。

如果你想主動 flush：

```c
int msync(void *addr, size_t len, int flags);
```

常見 flags：

- `MS_ASYNC`
- `MS_SYNC`
- `MS_INVALIDATE`

### `munmap`

```c
int munmap(void *addr, size_t len);
```

它只是解除映射，不等於：

- 幫你把資料一定同步寫回檔案

尤其在 `MAP_PRIVATE` 下，你改的東西根本不會回原檔。

另外一個很重要的點：

- `close(fd)` 不會自動 `munmap` 那塊區域

descriptor 關了，mapping 仍可能繼續存在。

### 一個典型 file copy 骨架

```c
int fdin = open(srcpath, O_RDONLY);
int fdout = open(dstpath, O_RDWR | O_CREAT | O_TRUNC, 0644);
struct stat st;
void *src, *dst;

fstat(fdin, &st);
ftruncate(fdout, st.st_size);

src = mmap(NULL, st.st_size, PROT_READ, MAP_SHARED, fdin, 0);
dst = mmap(NULL, st.st_size, PROT_READ | PROT_WRITE, MAP_SHARED, fdout, 0);

memcpy(dst, src, st.st_size);
msync(dst, st.st_size, MS_SYNC);

munmap(src, st.st_size);
munmap(dst, st.st_size);
```

### `mmap` 不一定比較快

這點 APUE 也講得很實際。

`mmap + memcpy` 有時比較快，有時不一定。  
因為你是在拿：

- page fault
- VM 管理成本

去交換：

- 傳統 `read/write` 的 system call 與 buffer copy 成本

所以 `mmap` 的優勢不該被神話。  
它常常真正的價值在：

- 某些演算法會變簡單
- 對隨機存取檔案很自然

## 14.9 Summary

這章像是一大包工具箱，但其實每個工具都在回答同一個問題：

- 當單純 blocking `read/write` 不足以描述你的需求時，該怎麼辦

你可以把整章壓成這張對照表：

- `O_NONBLOCK`：不要卡死
- `fcntl record lock`：協調多 process 對檔案區段的使用
- `select/pselect/poll`：同時等很多 fd
- `aio_*`：把 I/O 交給系統背景處理並追蹤完成
- `readv/writev`：一次處理多塊 buffer
- `readn/writen`：包掉 short read / short write
- `mmap`：把檔案當記憶體來操作

### 這章讀完你應該真的會的事

- 分辨 blocking 與 nonblocking I/O 的語意
- 知道 `EAGAIN/EWOULDBLOCK` 在 nonblocking 模式下的真正意思
- 用 `fcntl` 設定與解除 record lock
- 解釋 `select`、`pselect`、`poll` 的差異與適用情境
- 理解 POSIX AIO 的基本流程：submit、query、return、cancel
- 會在合適情境下使用 `readv/writev`、`readn/writen`
- 理解 `mmap`、`msync`、`munmap` 的核心語意與常見陷阱

### 這章最容易踩坑的地方

- 把 nonblocking 當成「自動高效能」，結果寫出瘋狂 polling 迴圈
- 誤以為 `F_GETLK` 再接 `F_SETLK` 就沒有 race
- 忘記 record locks 是跟 process + file 綁，不是跟單一 fd 綁
- 忘記 `close` 某個 referencing same file 的 fd 可能把整個 process 的 lock 放掉
- 把 EOF 當成 `select` 的 exception condition
- 忘記 `select` 會改寫 `fd_set`
- 對 AIO 還沒完成的 `aiocb` / buffer 亂動
- 以為 `mmap` 寫下去就等於已安全落盤
- 沒先 `ftruncate` 就 map 輸出檔做寫入

### 建議你現在立刻動手做

1. 寫一個 nonblocking pipe 範例，觀察 `EAGAIN`
2. 寫兩個 process 對同一檔案不同區段加 `fcntl` lock
3. 用 `select` 同時監看 `stdin` 和一個 socket / pipe
4. 寫一個 `readn/writen` 小工具來傳固定長度封包
5. 寫一個 `mmap` 版 file copy，並和 `read/write` 版比較

### 一句總結

Chapter 14 的核心不是 API 數量很多，而是你開始真正接觸 UNIX I/O 的現實世界：I/O 不一定一次完成、檔案可能被多人同時碰、很多 descriptor 要一起等、資料不一定要靠 `read/write` 來回搬；會不會選對工具，決定了你的系統程式是穩定還是脆弱。
