# Chapter 13 - Daemon Processes

## 這章在做什麼

這章在講 UNIX 裡一種很常見、但很容易被誤解的 process 類型：`daemon`。

很多人第一次接觸 daemon，會把它理解成：

- 「就是丟到背景跑的程式」

這只對了一小半。  
APUE 這章真正要教的是：

- daemon 和一般互動式 process 在結構上有什麼不同
- 為什麼 daemon 必須脫離 controlling terminal
- 一個 daemon 在初始化時到底該做哪些事
- 沒有 terminal 之後，錯誤訊息要怎麼記
- 怎麼保證系統裡同時只跑一份 daemon
- 真正的 daemon 在 UNIX 世界裡通常遵守哪些慣例

這章也是把前面學過的：

- `fork`
- `setsid`
- process group / session / controlling terminal
- `umask`
- file descriptor
- signals
- record locking

全部串起來的一章。

## 本章小節地圖

- `13.1 Introduction`
- `13.2 Daemon Characteristics`
- `13.3 Coding Rules`
- `13.4 Error Logging`
- `13.5 Single-Instance Daemons`
- `13.6 Daemon Conventions`
- `13.7 Client–Server Model`
- `13.8 Summary`

## 先抓住這章最重要的心智模型

### daemon 的本質不是「背景」，而是「脫離互動環境」

一個 process 只是放到背景，並不代表它就成了合格 daemon。  
真正重要的是：

- 它不依賴使用者的登入 session
- 它沒有 controlling terminal
- 它能長時間穩定存在
- 它能自己管理 logging、signals、配置檔與啟動方式

### `daemonize` 不是宗教儀式，而是在切斷不必要的繼承

daemon 初始化時那些看起來很固定的步驟，例如：

- `fork`
- `setsid`
- `chdir("/")`
- 關閉所有 file descriptors
- 把 `0/1/2` 接到 `/dev/null`

背後都是為了避免「繼承來的環境」傷到 daemon。

### daemon 是 service process 的基礎模型

APUE 在這章最後把 daemon 和 client-server 模型接起來，是因為：

- 很多 server 本質上就是 daemon

例如：

- `syslogd`
- `cron`
- `sshd`
- `inetd`

所以你把 daemon 看懂，後面看 UNIX server 就會清楚很多。

## 13.1 Introduction

APUE 一開始先給 daemon 下了一個非常實用的定義：

- daemon 是活很久的 process

它們通常：

- 系統開機時就被啟動
- 系統關機時才結束
- 沒有 controlling terminal
- 在背景持續提供某種服務

這一章的重點不是介紹特定一個 daemon，而是：

- 看 daemon 在 process 結構上長什麼樣
- 看如何自己把一個普通程式初始化成 daemon

## 13.2 Daemon Characteristics

### 先從 `ps` 看 daemon 的外觀

APUE 用 `ps` 的輸出來觀察 daemon，這很值得學。  
因為 daemon 的特性不只是概念，從 process table 就看得出來。

典型觀察重點有：

- `PID`
- `PPID`
- `PGID`
- `SID`
- `TTY`
- command name

### daemon 常見的外觀

從 process 關係來看，多數 user-level daemons 會呈現：

- `TTY` 是 `?`
- 沒有 controlling terminal
- 常常自己就是 process group leader
- 常常自己也是 session leader
- parent 往往是 `init` 類程序

這裡最值得你連回 Chapter 9 的觀念：

- 沒有 controlling terminal
- 往往表示它已透過 `setsid` 或類似流程脫離原 session

### kernel daemons 和 user-level daemons 不一樣

APUE 也提醒你：

- `PID` 為 0 或由 kernel 建立的內核工作者，不是一般 user-space daemon

它們同樣可能長時間存在，但和我們這章要自己寫的 user-level daemon 是不同層次。

### 幾個典型 daemon 的角色

這章會提到一些常見例子，例如：

- `syslogd` / `rsyslogd`：集中式 logging
- `inetd`：網路服務啟動器
- `cron`：定時工作
- `atd`：一次性排程工作
- `sshd`：遠端登入服務

這些例子在學習上很有幫助，因為它們讓你知道：

- daemon 不是一種很抽象的 process 類型
- 它就是系統裡那些長期駐留、等請求、做服務的程式

### user-level daemon 的幾個典型特徵

- 沒有 controlling terminal
- 通常由系統啟動流程拉起來
- 很多都以 `root` 或特定 service account 執行
- 多半長期存在
- 透過 logging 而不是螢幕輸出對外報告狀態

## 13.3 Coding Rules

這一節是整章最實作導向的地方。  
APUE 直接列出一組「把程式變成 daemon」的基本規則。

### Rule 1: `umask` 設成可預期值

```c
umask(0);
```

原因很簡單：

- daemon 不應該繼承啟動者奇怪的 file mode creation mask

不然你之後建立檔案時，即使程式碼看起來指定了某些權限，也可能被 inherited `umask` 偷偷改掉。

但這裡也要理解一個更成熟的觀念：

- 不一定非得永遠用 `umask(0)`
- 重點是把它設成「你知道的值」

如果你的 daemon 後續建立檔案時想保守一點，也可以再設更明確的 `umask`。

### Rule 2: `fork`，讓 parent 結束

```c
pid_t pid = fork();
if (pid > 0)
    exit(0);
```

這麼做有兩個主要目的：

1. 如果是從 shell 啟動，讓 shell 以為命令已完成
2. 確保 child 不是 process group leader，才能安全 `setsid`

這裡是 Chapter 9 的知識直接回來用：

- `setsid` 不能由 process group leader 呼叫

### Rule 3: `setsid`

```c
setsid();
```

這步驟是 daemon initialization 的核心。

它會：

1. 建立新的 session
2. 讓呼叫者成為 session leader
3. 讓呼叫者成為新的 process group leader
4. 脫離 controlling terminal

所以如果你問：

- 「daemon 為什麼會沒有 controlling terminal？」

很大的答案就在 `setsid`。

### 為什麼常常還要再 `fork` 一次

APUE 在這裡也講了經典的 double-fork。

流程是：

1. 第一次 `fork`
2. child `setsid`
3. 再 `fork`
4. 第二次 parent 結束
5. 最後留下來的 child 繼續跑 daemon

目的在於：

- 讓最後留下的 daemon 不是 session leader
- 降低它重新取得 controlling terminal 的可能性

這是歷史上 System V 習慣很重的做法。  
即使某些現代環境不一定絕對需要，你也應該理解這個傳統的理由。

### Rule 4: `chdir("/")`

```c
chdir("/");
```

原因是：

- 如果 daemon 把 working directory 留在某個 mounted filesystem
- 那個 filesystem 可能就無法被卸載

這一點非常務實。  
因為 daemon 通常活很久，所以一個不小心就會把某個 mount point 綁死。

當然，有些 daemon 會刻意切到自己的 spool / work directory。  
重點不是一定要 `/`，而是：

- 你要有意識地選一個合理的 working directory

### Rule 5: 關閉不需要的 file descriptors

daemon 不應該隨手繼承一大堆 parent 的 descriptors。  
不關掉可能導致：

- 資源 leak
- 管線 EOF 永遠不到
- 檔案 / socket / 裝置被意外持有
- 安全問題

APUE 的做法通常是：

- 用 `getrlimit(RLIMIT_NOFILE, ...)` 找上限
- 從 `0` 關到最大值

### Rule 6: 把 `stdin/stdout/stderr` 接到 `/dev/null`

這很重要，因為就算 daemon 沒有 terminal，某些 library code 還是可能去碰：

- `stdin`
- `stdout`
- `stderr`

常見作法：

```c
int fd0 = open("/dev/null", O_RDWR);
int fd1 = dup(0);
int fd2 = dup(0);
```

這樣可以確保：

- `0`、`1`、`2` 都是有效 descriptor
- 但不會真的對終端造成影響

### 一個典型 `daemonize` 骨架

```c
#include <fcntl.h>
#include <signal.h>
#include <stdlib.h>
#include <sys/resource.h>
#include <syslog.h>
#include <unistd.h>

void daemonize(const char *cmd)
{
    struct rlimit rl;
    struct sigaction sa;
    pid_t pid;
    int i, fd0, fd1, fd2;

    umask(0);

    if (getrlimit(RLIMIT_NOFILE, &rl) < 0)
        exit(1);

    if ((pid = fork()) < 0)
        exit(1);
    else if (pid != 0)
        exit(0);

    setsid();

    sa.sa_handler = SIG_IGN;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGHUP, &sa, NULL);

    if ((pid = fork()) < 0)
        exit(1);
    else if (pid != 0)
        exit(0);

    chdir("/");

    if (rl.rlim_max == RLIM_INFINITY)
        rl.rlim_max = 1024;
    for (i = 0; i < (int)rl.rlim_max; i++)
        close(i);

    fd0 = open("/dev/null", O_RDWR);
    fd1 = dup(0);
    fd2 = dup(0);

    openlog(cmd, LOG_CONS, LOG_DAEMON);
    if (fd0 != 0 || fd1 != 1 || fd2 != 2)
        exit(1);
}
```

### 這個 `daemonize` 的幾個細節不要只是抄

- `SIG_IGN` 給 `SIGHUP`：避免中間流程被 hangup 影響
- 第二次 `fork`：避免成為 session leader 後又碰回 terminal 問題
- `openlog` 放在這裡：後面就能用 syslog 做錯誤回報

## 13.4 Error Logging

### daemon 最直接的問題：錯誤訊息往哪裡去

一般互動式程式出錯時，很自然會：

- 寫到 `stderr`

但 daemon 沒有 terminal，這條路通常行不通。  
而且也不適合讓每個 daemon 自己亂寫一個獨立 log file，會把系統管理搞得很亂。

所以 UNIX 需要一個中央 logging 機制：

- `syslog`

### `syslog` facility 的結構

APUE 用一張圖說明：

- user process 呼叫 `syslog`
- 訊息送到 `/dev/log` 這類 UNIX domain socket
- `syslogd` / 類似 daemon 收到後
- 再依設定決定寫到檔案、console、登入使用者，或轉送到別台機器

你可以把它想成：

- daemon 不直接決定最終輸出去哪
- daemon 只負責把「這是一條什麼性質的訊息」交給 syslog 系統

### 基本 API

```c
#include <syslog.h>

void openlog(const char *ident, int option, int facility);
void syslog(int priority, const char *format, ...);
void closelog(void);
int setlogmask(int maskpri);
```

### `openlog`

`openlog` 讓你設定幾件事：

- `ident`：每則訊息前面帶的程式名稱
- `option`：一些控制選項
- `facility`：預設訊息類別

### 常見 `openlog` options

#### `LOG_PID`

- 每條訊息附上 `PID`

對於會 `fork` 子程序的 daemon 很實用。

#### `LOG_CONS`

- 如果沒法透過 syslogd 正常送出，就寫到 console

#### `LOG_NDELAY`

- 立刻打開連到 syslogd 的 socket，而不是等第一筆訊息才開

#### `LOG_PERROR`

- 除了送 syslog，也順便寫到 `stderr`

這對除錯時有幫助，但不是所有平台都一定有。

### facility 與 level

syslog 訊息不是只有一條字串，還會帶分類資訊。

#### facility

表示訊息來自哪類子系統，例如：

- `LOG_DAEMON`
- `LOG_AUTH`
- `LOG_CRON`
- `LOG_LPR`
- `LOG_MAIL`
- `LOG_USER`
- `LOG_LOCAL0` ~ `LOG_LOCAL7`

#### level

表示嚴重程度，例如由高到低大致有：

- `LOG_EMERG`
- `LOG_ALERT`
- `LOG_CRIT`
- `LOG_ERR`
- `LOG_WARNING`
- `LOG_NOTICE`
- `LOG_INFO`
- `LOG_DEBUG`

### `syslog` 呼叫方式

```c
openlog("mydaemon", LOG_PID, LOG_DAEMON);
syslog(LOG_ERR, "open error for %s: %m", path);
```

這裡 `%m` 很值得記：

- 會自動展開成 `errno` 對應的錯誤字串

這是 syslog format 的實用小特性。

如果沒先 `openlog`，也不是完全不能用，因為：

- 第一次 `syslog` 時通常會自動幫你做類似初始化

但顯式 `openlog` 會更清楚。

### `setlogmask`

有些時候你不想讓太低優先級的訊息真的被記錄，可以用：

- `setlogmask`

把 process 的 logging mask 設好。  
這對 debug / production 切換很有用。

### `vsyslog`

一些系統還提供：

```c
void vsyslog(int priority, const char *format, va_list arg);
```

它對寫 logging wrapper 很方便。

### 這節的實務重點

- daemon 不要亂 `printf`
- daemon 也不要自己各寫一套隨意 log 機制
- 先用 `syslog` 這個系統級管道，通常才是 UNIX 風格

## 13.5 Single-Instance Daemons

### 為什麼有些 daemon 必須全系統只能一份

有些 daemon 若同時跑多份，會直接出事。  
例如：

- 都去操作同一個裝置
- 都去執行同一批排程
- 都去佔用同一組服務狀態

像 `cron` 如果跑兩份，很可能同一個工作會被執行兩次。

### 一個標準做法：lock file

APUE 推薦的方式是：

- 建立一個固定名稱的檔案
- 對整個檔案加 write lock

如果成功：

- 代表你是目前唯一 instance

如果失敗且錯誤是：

- `EACCES`
- `EAGAIN`

就表示通常已經有別的 instance 持有鎖。

### 為什麼這招比單純看 pid file 更可靠

只寫 pid file 不夠，因為：

- pid file 可能是殘留的
- process 可能早就死了

而 record lock 的好處是：

- process 死掉時，kernel 會自動釋放 lock

所以 recovery 比較乾淨。

### 一個典型 `already_running`

```c
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <syslog.h>
#include <unistd.h>

#define LOCKFILE "/var/run/mydaemon.pid"
#define LOCKMODE (S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH)

static int lockfile(int fd)
{
    struct flock fl;

    fl.l_type = F_WRLCK;
    fl.l_start = 0;
    fl.l_whence = SEEK_SET;
    fl.l_len = 0;
    return fcntl(fd, F_SETLK, &fl);
}

int already_running(void)
{
    int fd;
    char buf[32];

    fd = open(LOCKFILE, O_RDWR | O_CREAT, LOCKMODE);
    if (fd < 0) {
        syslog(LOG_ERR, "can't open %s: %s", LOCKFILE, strerror(errno));
        exit(1);
    }

    if (lockfile(fd) < 0) {
        if (errno == EACCES || errno == EAGAIN) {
            close(fd);
            return 1;
        }
        syslog(LOG_ERR, "can't lock %s: %s", LOCKFILE, strerror(errno));
        exit(1);
    }

    ftruncate(fd, 0);
    snprintf(buf, sizeof(buf), "%ld\n", (long)getpid());
    write(fd, buf, strlen(buf));
    return 0;
}
```

### 為什麼寫 pid 前要 `ftruncate`

因為上一個 instance 的 pid 可能比較長。  
如果不清掉，例如：

- 舊 pid 是 `12345`
- 新 pid 是 `999`

你可能最後檔案裡留成像：

- `99945`

這種髒資料。

## 13.6 Daemon Conventions

這節不是 POSIX 法規，而是 UNIX 世界長年形成的實務習慣。

### lock file 常放哪裡

通常放：

- `/var/run/name.pid`

其中 `name` 往往是 daemon 名稱。

### config file 常放哪裡

通常放：

- `/etc/name.conf`

這使系統管理者知道：

- 這個服務的設定大概在哪找

### daemon 怎麼啟動

可以手動命令列啟動，但通常更常見是：

- 開機初始化腳本
- service manager
- 類似 `init` / `launchd` / `systemd` 的機制

APUE 這裡提的是較傳統的 `/etc/rc*`、`/etc/init.d/*` 與 `inittab` 風格，你把它理解成：

- 系統層會替 daemon 管生命週期

### 為什麼很多 daemon 重讀設定用 `SIGHUP`

這是個經典慣例。

daemon 啟動時通常只讀一次 config。  
如果管理者後來改了設定，不想整個 daemon stop/restart，就常會：

- 對 daemon 送 `SIGHUP`

讓它重讀 config file。

這麼做在 daemon 身上很合理，因為：

- daemon 本來就不該有 terminal
- 不會莫名因使用者掛斷終端而收到那種互動式的 `SIGHUP`

所以 `SIGHUP` 在 daemon world 很常被重新詮釋成：

- reload configuration

### multithreaded daemon 的 signal 處理方式

APUE 這裡把 Chapter 12 的 thread-signal 模型拿來用：

- block signals
- 開一條專門 thread 用 `sigwait`

這是很好的設計，因為它讓 daemon 的 signal handling 比較同步、可控。

簡化版骨架：

```c
static sigset_t mask;

static void reread(void)
{
    /* reload config */
}

static void *sig_thread(void *arg)
{
    int signo;
    (void)arg;

    for (;;) {
        sigwait(&mask, &signo);
        switch (signo) {
        case SIGHUP:
            syslog(LOG_INFO, "re-reading configuration");
            reread();
            break;
        case SIGTERM:
            syslog(LOG_INFO, "got SIGTERM; exiting");
            exit(0);
        }
    }
}
```

### single-threaded daemon 也可以直接裝 handler

如果 daemon 沒用 threads，也可以：

- 直接 `sigaction(SIGHUP, ...)`
- 直接 `sigaction(SIGTERM, ...)`

只是要很清楚 signal handler 裡能做多少事。  
若 reload 動作很複雜，實務上常見做法是：

- handler 只設 flag
- 主流程再做真正重讀

## 13.7 Client-Server Model

### 很多 daemon 本質上就是 server

APUE 在這裡把 daemon 放回更大的系統圖景裡：

- daemon 常常是在等 client request 的 server

例如 `syslogd`：

- client 送 log message
- daemon 收下並處理

### server 常常會 `fork/exec` 出別的程式

這時就有一個很現實的安全與資源問題：

- server 原本打開的很多 descriptors，不應該被 child 意外繼承到新程式裡

不然可能發生：

- log file 被無關程式持有
- config file descriptor 洩漏
- client socket 被不該碰的程式看到
- 安全風險上升

### 解法：`FD_CLOEXEC`

```c
#include <fcntl.h>

int set_cloexec(int fd)
{
    int val;

    val = fcntl(fd, F_GETFD, 0);
    if (val < 0)
        return -1;

    val |= FD_CLOEXEC;
    return fcntl(fd, F_SETFD, val);
}
```

這樣在 `exec` 時：

- 該 descriptor 會自動被關閉

這是 server / daemon 寫法裡非常值得養成的習慣。

### 這節真正要你建立的觀念

- daemon 不只是獨自待在背景
- 它常常是服務架構中的常駐 server
- 所以 descriptor hygiene、signal handling、logging、instance control 都要做好

## 13.8 Summary

這章看似在講一種特殊 process，實際上是在教你：

- 如何把一個普通 UNIX 程式收斂成一個可長期運作的系統服務

其核心動作包括：

- 切斷對 terminal 與登入環境的依賴
- 清理繼承來的 process state
- 建立可靠 logging
- 用 lock file 確保 single instance
- 遵守慣例，讓系統管理與維護更一致

如果你把整章濃縮成幾句實務原則，大概就是：

- daemon 不應該依賴互動式 I/O
- daemon 要有可預期的工作目錄、`umask` 和 file descriptor 狀態
- daemon 出錯要靠 `syslog`
- daemon 若需要單例，就用 lock + pid file，而不是土法猜測
- daemon 通常會把 `SIGHUP` 當 reload，把 `SIGTERM` 當正常退出

### 這章讀完你應該真的會的事

- 解釋 daemon 和一般背景 job 的差別
- 理解為什麼 `fork + setsid + 可能再 fork` 能讓 process 脫離 controlling terminal
- 自己寫出基本的 `daemonize` 函式
- 用 `syslog` 做 daemon logging
- 用 lock file 保證 single-instance daemon
- 解釋為什麼 server/daemon 常要設 `FD_CLOEXEC`

### 這章最容易踩坑的地方

- 以為 `&` 背景執行就等於 daemon
- 忘記 `umask` 與 working directory 是 inherited state
- 沒關掉多餘 descriptors，導致資源與安全問題
- 沒處理 `0/1/2`，讓 library code 意外寫到奇怪地方
- 只寫 pid file 卻不加 lock
- 在 multithreaded daemon 裡亂用 signal handler，而不是 `sigwait`

### 建議你現在立刻動手做

1. 寫一個最小 `daemonize` 版本，執行後用 `ps` 檢查 `PPID`、`PGID`、`SID`、`TTY`
2. 幫它接上 `syslog`
3. 再加上 `/var/run/name.pid` lock file
4. 最後加 `SIGHUP` reload 與 `SIGTERM` clean shutdown

### 一句總結

Chapter 13 的核心不是「程式放背景」，而是「把程式變成一個真正獨立、可長期運作、可管理、可維護的系統服務」。
