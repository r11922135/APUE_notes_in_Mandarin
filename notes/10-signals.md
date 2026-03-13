# Chapter 10 - Signals

## 這章在做什麼

這章在講 UNIX 裡最古老、也最容易出錯的非同步機制之一：`signals`。

如果用一句話概括 signal：

- 它不是用來傳資料的
- 它是用來通知某件事發生了

例如：

- 使用者按了 `Ctrl-C`
- child process 結束了
- 計時器到了
- 你寫爆檔案大小限制了
- 程式做了非法記憶體存取

signal 的麻煩在於：

- 它是非同步來的
- 它可能在任何時間點打斷你
- handler 裡能做的事非常少

所以這章真正想教你的，不只是 API 名字，而是：

- 怎麼建立正確的 signal 心智模型
- 怎麼避免 race condition
- 怎麼寫出不會在 handler 裡自爆的程式

## 本章小節地圖

- `10.1 Introduction`
- `10.2 Signal Concepts`
- `10.3 signal Function`
- `10.4 Unreliable Signals`
- `10.5 Interrupted System Calls`
- `10.6 Reentrant Functions`
- `10.7 SIGCLD Semantics`
- `10.8 Reliable-Signal Terminology and Semantics`
- `10.9 kill and raise Functions`
- `10.10 alarm and pause Functions`
- `10.11 Signal Sets`
- `10.12 sigprocmask Function`
- `10.13 sigpending Function`
- `10.14 sigaction Function`
- `10.15 sigsetjmp and siglongjmp Functions`
- `10.16 sigsuspend Function`
- `10.17 abort Function`
- `10.18 system Function`
- `10.19 sleep, nanosleep, and clock_nanosleep Functions`
- `10.20 sigqueue Function`
- `10.21 Job-Control Signals`
- `10.22 Signal Names and Numbers`
- `10.23 Summary`

## 先抓住這章最重要的心智模型

### signal 的完整生命週期不是「送了就立刻跑 handler」

比較完整的理解是：

1. signal 被產生 (`generated`)
2. process 對該 signal 可能目前是 blocked
3. 如果 blocked，它會先變成 pending
4. 等到 unblocked 才可能 delivery
5. delivery 時再依 disposition 決定：
   - ignore
   - default action
   - 呼叫 handler

這個模型非常重要。  
你後面看 `sigprocmask`、`sigpending`、`sigsuspend` 時，全都靠這張圖理解。

### signal handler 不是普通 callback

很多人會把 handler 想成：

- 「喔，就是某個事件發生時，系統呼叫一個函式」

這只對一半。  
真正麻煩的是：它可能在你的程式執行到一半時，突然插進來。

例如：

- 你正在 `malloc`
- 你正在 `printf`
- 你正在修改某個全域資料結構

這時如果 handler 又去呼叫不安全的函式，就很容易毀掉程式狀態。

### 新程式幾乎都應該用 `sigaction`，不是老式 `signal`

本章雖然會講 `signal()`，但你在實務上應該優先記住：

- `sigaction`
- `sigprocmask`
- `sigsuspend`

這三個才是現代可攜且可靠的核心工具。

### signal 最重要的設計原則

你可以先背下這條實務守則：

- handler 裡少做事
- 只設 flag 或做非常小的 async-signal-safe 動作
- 真正工作回到主流程做

這會救你很多次。

## 10.1 Introduction

APUE 這章開頭先把 signal 定位成：

- software interrupt
- asynchronous event notification

也就是說，它不是 function call，也不是 message queue。  
它比較像：

- 「系統打斷你一下，告訴你某件事發生了」

這個機制很強，但也因為太非同步，所以出錯機率很高。

## 10.2 Signal Concepts

### signal 有名稱，也有編號

像：

- `SIGINT`
- `SIGTERM`
- `SIGKILL`
- `SIGCHLD`

都對應到某個 signal number。  
但你通常應該依賴名稱，不要把數字硬編在程式裡，因為不同實作的編號可能不同。

另外：

- signal number 必須大於 0
- `signal 0` 不是一般 signal，它常拿來做存在性 / 權限檢查

### signal 可能從哪裡來

常見來源有：

- terminal 產生，例如 `Ctrl-C`
- hardware exception，例如非法記憶體存取
- kernel 偵測到某種條件，例如 broken pipe
- 一個 process 對另一個 process 呼叫 `kill`
- timer 到時，例如 `alarm`

### process 收到 signal 時有哪些處理方式

對每個 signal，process 大致有三種 disposition：

- `SIG_DFL`：使用預設行為
- `SIG_IGN`：忽略
- 自訂 handler：收到時跑你註冊的函式

但有兩個例外一定要記住：

- `SIGKILL`
- `SIGSTOP`

這兩個不能被捕捉、不能被忽略、也不能被阻擋。

### default action 不只一種

不同 signal 的預設行為可能是：

- terminate
- terminate 並產生 core dump
- ignore
- stop
- continue

所以不能只記 signal 名字，還要知道不處理時會發生什麼。

### 幾個最常用、最值得先熟的 signals

#### 跟互動控制有關

- `SIGINT`：通常來自 `Ctrl-C`
- `SIGQUIT`：通常來自 `Ctrl-\`
- `SIGTSTP`：通常來自 `Ctrl-Z`
- `SIGTTIN`：背景 job 嘗試讀 terminal
- `SIGTTOU`：背景 job 嘗試寫 terminal 或改 terminal state
- `SIGCONT`：繼續一個 stopped process

#### 跟 process 生命週期有關

- `SIGCHLD`：child 狀態改變
- `SIGTERM`：要求程式正常終止
- `SIGKILL`：強制殺掉

#### 跟錯誤或資源有關

- `SIGSEGV`：非法記憶體存取
- `SIGPIPE`：往沒人讀的 pipe / socket 寫資料
- `SIGALRM`：`alarm` 到時
- `SIGXCPU`：CPU time limit 超過
- `SIGXFSZ`：file size limit 超過

#### 給應用程式自己用

- `SIGUSR1`
- `SIGUSR2`

## 10.3 `signal` Function

### 原型

```c
void (*signal(int signo, void (*func)(int)))(int);
```

這個型別看起來很醜，但實際上你可以把它想成：

- 指定某個 signal 的處理方式
- 回傳舊的處理方式

特殊常數有：

- `SIG_DFL`
- `SIG_IGN`
- `SIG_ERR`

### 為什麼 `signal()` 在教學上重要，但在實務上不夠好

歷史上不同 UNIX 對 `signal()` 的語意不完全一致。  
尤其早期系統在 handler 被呼叫後，可能：

- 自動把 disposition 重設回 default
- 導致訊號處理不可靠

所以 APUE 會講它，但也會一路帶你走向 `sigaction()`。

### `fork` 和 `exec` 對 signal disposition 的影響

這個規則一定要記：

- `fork` 後，child 繼承 parent 的 signal disposition
- `exec` 後，原本被 catch 的 signals 會重設為 default
- 但原本被 ignore 的 signals，通常維持 ignore

這也是為什麼 shell 在啟動程式後，該程式的 signal 行為常不像 shell 本身。

## 10.4 Unreliable Signals

### 早期 signal 為什麼被叫做 unreliable

主要有兩個老問題：

1. handler 執行一次後可能自動被重設回 default
2. 同類型 signal 可能被合併或遺失

所以老程式常得在 handler 一進來就重新安裝 handler，非常醜也容易漏。

### 最經典的 race：檢查旗標後才 `pause()`

很多人會直覺寫出這種程式：

```c
if (flag == 0)
    pause();
```

問題是：

- 你檢查完 `flag`
- 還沒 `pause`
- signal 就到了，handler 把 `flag` 設成 1
- handler 返回
- 你接著執行 `pause()`

結果就是：

- signal 已經來過了
- 你卻還是睡死在 `pause()` 裡

這就是為什麼後面要學 `sigsuspend()`。

## 10.5 Interrupted System Calls

### signal 可能把 blocking system call 打斷

假設你正在：

- `read`
- `write`
- `accept`
- `wait`

如果期間 signal 被 delivery，某些 system call 可能直接提早返回，並且：

- 回傳 `-1`
- `errno == EINTR`

### `EINTR` 不一定表示失敗，而是「被 signal 打斷」

這種情況下，程式常要自己決定：

- 重試
- 還是把控制權交回上層

最常見的處理模式：

```c
ssize_t n;

for (;;) {
    n = read(fd, buf, sizeof(buf));
    if (n < 0 && errno == EINTR)
        continue;
    break;
}
```

### `SA_RESTART` 能幫你，但不是萬能

用 `sigaction` 安裝 handler 時，某些系統呼叫在 signal handler 返回後，可能因為 `SA_RESTART` 而自動重啟。

這可以減少很多麻煩，但你不能假設：

- 所有 blocking 呼叫都一定會被 restart
- 所有平台行為都完全一樣

實務上還是要知道 `EINTR` 是什麼，並在關鍵路徑妥善處理。

## 10.6 Reentrant Functions

### 什麼叫 `reentrant`

簡單講：

- 某函式如果在「前一次呼叫還沒完成」時又被呼叫，仍然安全
- 那它比較接近 reentrant

signal handler 之所以危險，就是因為它可能在任意時刻打斷主流程。  
如果主流程正在某個 library function 裡，handler 又呼叫同一類不安全函式，就可能炸掉。

### 為什麼 `printf` 在 handler 裡很危險

因為 `stdio` 內部通常有：

- buffer
- global state
- lock

如果主流程正在 `printf`，handler 又來一個 `printf`，內部狀態可能就壞掉。

同理：

- `malloc`
- `free`
- 大部分 `stdio` 函式

在 handler 裡通常都不應該碰。

### handler 裡的安全策略

最穩的做法通常是：

- 設一個 `volatile sig_atomic_t` flag
- 或用 `write` 寫一小段固定訊息

例如：

```c
#include <signal.h>
#include <unistd.h>

static volatile sig_atomic_t got_sigint = 0;

static void sigint_handler(int signo)
{
    const char msg[] = "caught SIGINT\n";
    (void)signo;
    got_sigint = 1;
    write(STDERR_FILENO, msg, sizeof(msg) - 1);
}
```

### 別忘了保存與還原 `errno`

如果 handler 裡做了 system call，可能改掉 `errno`。  
所以 handler 若會影響主流程的錯誤判斷，常見寫法是：

```c
static void handler(int signo)
{
    int saved_errno = errno;
    (void)signo;

    /* minimal work */

    errno = saved_errno;
}
```

## 10.7 `SIGCLD` Semantics

### `SIGCLD` 是歷史包袱，真正該記的是 `SIGCHLD`

這一節在講某些舊 System V 系統的 `SIGCLD` 行為和現代標準不完全一致。  
對現在的學習重點來說，最重要的是：

- 把 child 狀態變化相關 signal 當成 `SIGCHLD` 來理解
- 寫新程式不要依賴老的 `SIGCLD` 特殊語意

### 為什麼 APUE 仍然花篇幅講它

因為 signal 的歷史差異，正是很多「同樣的程式在不同 UNIX 上行為不一致」的來源。  
這章一直在提醒你：

- signal API 不是只有語法
- 還有歷史語意差異

## 10.8 Reliable-Signal Terminology and Semantics

### 幾個核心名詞一定要分清楚

#### `generation`

signal 被產生了。  
來源可能是：

- kernel
- 其他 process
- 自己呼叫 `raise`

#### `delivery`

signal 真正送到 process，開始依 disposition 被處理。

#### `pending`

signal 已經產生，但因為目前 blocked 或其他原因，尚未 delivery。

#### `blocked`

目前 signal mask 把這個 signal 擋住了，所以它不能立刻 delivery。

### `blocked` 不等於 `ignored`

這兩個常被搞混：

- blocked：暫時不能 delivery，之後可能還會送到
- ignored：送到了也不做動作

### 標準 signals 常常不排隊，而是合併

這是非常重要的實務觀念。

對很多傳統 / 標準 signal 來說：

- 如果同一種 signal 已經 pending
- 再來一個同種 signal

不一定會記成兩筆。  
常見情況是：

- 只記得「至少有一個這種 signal 在等」

因此 signal 不適合拿來做「次數精準」的事件計數。

## 10.9 `kill` and `raise` Functions

### `raise(signo)` 幾乎可視為送 signal 給自己

常見理解是：

```c
raise(signo)
```

等價於概念上的：

```c
kill(getpid(), signo);
```

### `kill` 的 `pid` 參數不是只有單一 process 這種意思

`kill(pid, signo)` 裡的 `pid` 有多種語意：

- `pid > 0`：送給指定 `PID`
- `pid == 0`：送給呼叫者所在 process group
- `pid < -1`：送給 `abs(pid)` 那個 process group
- `pid == -1`：送給所有你有權限送的 processes

這跟 Chapter 9 的 process group 就接起來了。

### `signal 0` 的特殊用途

```c
kill(pid, 0);
```

不是真的送 signal。  
它常用來測試：

- 該 process 是否存在
- 你是否有權限對它送 signal

但要注意，這不是原子保證。  
因為你檢查完之後，該 `PID` 可能立刻結束，甚至被系統重用。

### 權限規則

一般來說，送 signal 需要符合某些 real / effective / saved IDs 的權限條件。  
`SIGCONT` 另外有「同一 session」的特殊規則，因為它跟 job control 有密切關係。

### 範例：檢查 process 是否還活著

```c
#include <signal.h>
#include <errno.h>

int process_exists(pid_t pid)
{
    if (kill(pid, 0) == 0)
        return 1;
    if (errno == EPERM)
        return 1;
    return 0;
}
```

這個函式只適合做粗略檢查，不適合當作強同步機制。

## 10.10 `alarm` and `pause` Functions

### `alarm`：每個 process 同一時間只有一個

```c
unsigned int alarm(unsigned int seconds);
```

它的意思是：

- 請在幾秒後送我一個 `SIGALRM`

但一個 process 同時只能掛一個 alarm。  
新的 `alarm()` 會覆蓋舊的設定。

### `pause`：睡到收到一個 signal

```c
int pause(void);
```

它會一直睡到有 signal 被捕捉，然後回傳：

- `-1`
- `errno == EINTR`

### 為什麼 `alarm + pause` 很容易寫出 race

如果你想自己做 `sleep`，直覺可能會寫：

1. 設 `alarm`
2. 呼叫 `pause`

但這中間仍然有 race：  
如果 signal 在你進 `pause` 前就到了，你會永遠睡住。

APUE 用這些例子是要告訴你：

- signal-based timeout 如果寫得不夠嚴謹，非常容易有時序 bug

### 用 `alarm` 包住 I/O timeout 的限制

你有時會看到這種模式：

```c
alarm(5);
n = read(fd, buf, size);
alarm(0);
```

概念是：

- 5 秒內沒讀到，就讓 `SIGALRM` 打斷 `read`

這種寫法有時有效，但要注意：

- 前提是該系統呼叫真的會被 signal 打斷
- 如果 `SA_RESTART` 或平台行為讓它重啟，就不一定達成 timeout
- 它還可能干擾程式裡其他使用 alarm 的地方

## 10.11 Signal Sets

### `sigset_t` 是操作 signal mask 的基本型別

你不會直接拿一個整數去表示 signal set，而是用：

```c
sigset_t set;
```

常見操作函式：

- `sigemptyset`
- `sigfillset`
- `sigaddset`
- `sigdelset`
- `sigismember`

### 常見初始化方式

```c
sigset_t set;

sigemptyset(&set);
sigaddset(&set, SIGINT);
sigaddset(&set, SIGTERM);
```

這代表：

- 先建空集合
- 再把 `SIGINT`、`SIGTERM` 放進去

## 10.12 `sigprocmask` Function

### 這個函式用來改目前 process 的 signal mask

原型：

```c
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```

`how` 常見值：

- `SIG_BLOCK`
- `SIG_UNBLOCK`
- `SIG_SETMASK`

### 最常見用途：保護 critical section

有些程式片段如果中途被 signal 打斷，狀態可能不一致。  
這時可以先 block 某些 signals，做完再恢復。

```c
sigset_t newmask, oldmask;

sigemptyset(&newmask);
sigaddset(&newmask, SIGINT);

sigprocmask(SIG_BLOCK, &newmask, &oldmask);

/* critical section */

sigprocmask(SIG_SETMASK, &oldmask, NULL);
```

### `SIGKILL` 與 `SIGSTOP` 不能被 block

這是系統保留的保底控制手段。  
不然一個 process 只要把所有關鍵 signal 都 block 掉，系統就很難處理它了。

### 小心「先解除 block，再等待」的 race

很多 signal bug 就出在：

- 你想等某個 signal
- 先手動解除 mask
- 再呼叫 `pause`

這會留下時序空窗。  
解法就是後面的 `sigsuspend()`。

## 10.13 `sigpending` Function

### 這個函式讓你看到「已經 pending 但尚未 delivery」的 signals

```c
int sigpending(sigset_t *set);
```

它拿到的是：

- 目前 blocked 且 pending 的 signal set

### 典型用途

這函式比較常出現在教學與除錯，幫你確認：

- 某 signal 是不是已經來了
- 只是因為 block 還沒被處理

簡單例子：

```c
sigset_t pend;

sigpending(&pend);
if (sigismember(&pend, SIGINT))
    write(STDOUT_FILENO, "SIGINT pending\n", 15);
```

## 10.14 `sigaction` Function

### 這才是現代 signal 安裝 API

原型：

```c
int sigaction(int signo, const struct sigaction *act,
              struct sigaction *oact);
```

`struct sigaction` 裡最重要的欄位有：

- `sa_handler`
- `sa_sigaction`
- `sa_mask`
- `sa_flags`

### `sa_handler` 與 `sa_sigaction`

如果你只需要傳統 handler：

```c
void handler(int signo);
```

就用 `sa_handler`。

如果你想拿更多資訊，例如：

- 送 signal 的 `PID`
- fault address
- child exit status
- `sigqueue` 帶來的值

那就配 `SA_SIGINFO`，使用：

```c
void handler(int signo, siginfo_t *info, void *context);
```

### `sa_mask` 是「handler 執行期間要額外 block 哪些 signals」

這點很容易被忽略。  
它的意思不是「永遠 block」，而是：

- 當這個 handler 跑起來時
- 額外把哪些 signals 擋住

另外，正在處理的那個 signal 通常也會自動被 block，避免 handler 重入；除非你用了 `SA_NODEFER`。

### 常見 `sa_flags`

#### `SA_RESTART`

盡量讓某些被 signal 打斷的 system calls 自動 restart。

#### `SA_SIGINFO`

使用三參數 handler，拿到 `siginfo_t`。

#### `SA_NOCLDSTOP`

對 `SIGCHLD` 而言，child stop / continue 時不要一直通知你。

#### `SA_NOCLDWAIT`

避免 child 變成 zombie，常見於不打算 `wait` child 的情況。

#### `SA_RESETHAND`

handler 執行一次後重設為 default。

#### `SA_NODEFER`

處理某 signal 時，不自動 block 同一 signal；通常要非常小心用。

### 推薦的安裝寫法

```c
#include <signal.h>
#include <string.h>

int install_handler(int signo, void (*handler)(int))
{
    struct sigaction act;

    memset(&act, 0, sizeof(act));
    act.sa_handler = handler;
    sigemptyset(&act.sa_mask);
    act.sa_flags = SA_RESTART;

    return sigaction(signo, &act, NULL);
}
```

### 為什麼 `sigaction` 比 `signal` 好

- 語意更明確
- 可攜性更好
- 可設定 mask
- 可設定 flags
- handler 不會因為歷史語意不一致而讓你中雷

## 10.15 `sigsetjmp` and `siglongjmp` Functions

### 為什麼 signal 情境下不能只靠 `setjmp` / `longjmp`

Chapter 7 你已經看過非本地跳躍。  
到了 signal 章，問題變成：

- signal handler 裡如果要跳回主流程
- signal mask 要不要一起恢復

這就是 `sigsetjmp` / `siglongjmp` 的用途。

### `sigsetjmp(env, savemask)`

如果 `savemask != 0`，它會連當前 signal mask 一起保存。  
這點非常重要，因為 handler 執行時 mask 常常已被改動。

### 常見模式：`volatile sig_atomic_t canjump`

APUE 常用這個技巧避免 handler 在程式尚未準備好時亂跳：

```c
static sigjmp_buf env;
static volatile sig_atomic_t canjump;

static void sig_usr1(int signo)
{
    (void)signo;
    if (canjump == 0)
        return;
    siglongjmp(env, 1);
}
```

主流程大致會是：

```c
if (sigsetjmp(env, 1) != 0) {
    /* returned from handler */
}

canjump = 1;
```

### 這是高階工具，不是日常萬用解

能不用非本地跳躍就不用。  
但在某些「深層 call stack 被 signal 打斷，需要快速回到安全點」的情境，它很有用。

## 10.16 `sigsuspend` Function

### `sigsuspend` 解決的是「原子地換 mask 並等待 signal」

原型：

```c
int sigsuspend(const sigset_t *mask);
```

它的精髓是：

- 暫時把目前 mask 換成你指定的 mask
- 然後睡眠等待 signal
- handler 返回後，自動恢復舊 mask

這整個動作是原子的。  
也就是說，它能補上 `unblock` 和 `pause` 之間那個最危險的 race。

### 典型等待模式

```c
volatile sig_atomic_t quitflag = 0;

while (quitflag == 0)
    sigsuspend(&zeromask);
```

配合：

- 先 block 某 signal
- handler 把 `quitflag` 設成 1
- 主流程用 `sigsuspend` 等待

這是經典且正確的寫法。

### 為什麼這麼重要

如果你真的理解 `sigsuspend`，你就已經跨過 signal 最常見的 race 陷阱了。

## 10.17 `abort` Function

### `abort()` 的目的是讓 process 以 `SIGABRT` 終止

```c
void abort(void);
```

它通常用在：

- 發現嚴重、不該繼續執行的內部錯誤
- 想保留 core dump 供除錯

### 為什麼它不是單純 `raise(SIGABRT)` 那麼簡單

標準要求：

- 即使 `SIGABRT` 被 block 或被 handler 攔截
- `abort()` 最終也必須讓 process 終止

所以實作通常會：

- 先確保 `SIGABRT` 沒被 block
- 送出 `SIGABRT`
- 如果 handler 返回，重設 disposition 再送一次

也就是說，`abort` 的設計目標是：

- 「盡量保證你真的死掉」

## 10.18 `system` Function

### `system()` 跟 signal 的糾葛比你想像中多

Chapter 8 你已經知道 `system(cmd)` 大致相當於：

- `fork`
- child `exec /bin/sh -c cmd`
- parent `waitpid`

但在 signal 層面，POSIX 還要求它妥善處理：

- `SIGINT`
- `SIGQUIT`
- `SIGCHLD`

### 為什麼要這樣

如果 parent 在等 shell 時，自己也把 `SIGINT` / `SIGQUIT` 當成平常那樣處理，行為可能會亂掉。  
因此標準要求 `system()` 在執行命令期間，parent 需要適當：

- ignore `SIGINT`
- ignore `SIGQUIT`
- block `SIGCHLD`

等 child 結束後再恢復原本設定。

### 實務提醒

- `system()` 方便，但可控性差
- 會牽涉 shell expansion
- 不適合 setuid / setgid 程式
- signal 與狀態管理都比手寫 `fork`/`exec`/`waitpid` 複雜

## 10.19 `sleep`, `nanosleep`, and `clock_nanosleep` Functions

### `sleep` 很簡單，但歷史包袱多

```c
unsigned int sleep(unsigned int seconds);
```

它好用，但不同實作歷史上對：

- `alarm`
- 被 signal 打斷後剩餘時間

的互動不完全相同。

### `nanosleep` 提供更乾淨的高解析度睡眠

```c
int nanosleep(const struct timespec *req, struct timespec *rem);
```

優點：

- 時間精度更高
- 若被 signal 打斷，可透過 `rem` 知道剩多少時間

這讓你比較容易實作「睡滿指定時間」。

### `clock_nanosleep` 更進一步支援指定 clock 與 absolute time

它可以指定：

- 用哪個 clock
- 是 relative sleep 還是 absolute sleep

absolute sleep 很有用，因為它能避免週期性任務累積 drift。

例如你想每秒整點做一次事，比起：

- 每次工作完再 `sleep(1)`

更穩的是：

- 算下一個 deadline
- 用 absolute `clock_nanosleep`

## 10.20 `sigqueue` Function

### `sigqueue` 讓 signal 可以帶一點 payload

原型：

```c
int sigqueue(pid_t pid, int signo, const union sigval value);
```

跟普通 `kill` 相比，它多了：

- 可以附一個整數或指標型 payload

接收端若用 `SA_SIGINFO` handler，就能從 `siginfo_t` 取出：

- `si_value`

### 這比較接近「有資料的 signal」

但仍然要注意：

- 它不是大型資料傳輸機制
- 量很有限
- 受系統佇列限制

如果送太多，可能得到：

- `EAGAIN`

### 簡單範例

```c
static void rt_handler(int signo, siginfo_t *info, void *ctx)
{
    (void)signo;
    (void)ctx;
    printf("from pid=%ld, value=%d\n",
           (long)info->si_pid, info->si_value.sival_int);
}
```

安裝時要配：

- `SA_SIGINFO`

### real-time signals

`sigqueue` 常和 real-time signals 搭配。  
這些 signal 相比傳統 signals，通常更接近真正排隊的語意。

## 10.21 Job-Control Signals

### 這一節把 signal 和 Chapter 9 完整接起來

跟 job control 最密切的 signals 包括：

- `SIGCHLD`
- `SIGCONT`
- `SIGSTOP`
- `SIGTSTP`
- `SIGTTIN`
- `SIGTTOU`

### 最常見的使用者互動

- `Ctrl-Z` -> `SIGTSTP`
- shell 用 `waitpid(..., WUNTRACED)` 看到 job stopped
- 使用者輸入 `bg` -> shell 送 `SIGCONT`
- 使用者輸入 `fg` -> shell `tcsetpgrp` + `SIGCONT`

### 互動式全螢幕程式為什麼常要特別處理 `SIGTSTP`

像 `vi`、`less` 這種程式若直接被停掉，terminal 狀態可能留在：

- raw mode
- no echo

畫面會爛掉。  
所以它們常用的做法是：

1. 抓 `SIGTSTP`
2. 先把 terminal 恢復正常
3. 把 disposition 改回 default
4. 再送 `SIGTSTP` 給自己真的停下來
5. 被 `SIGCONT` 喚醒後，重設 terminal 並重畫畫面

這是 signal 和 terminal state 管理結合得很漂亮的經典例子。

## 10.22 Signal Names and Numbers

### 不要把 signal number 寫死

雖然每個 signal 都有編號，但不同系統不一定一致。  
可攜程式應該依賴：

- `SIGINT`
- `SIGTERM`
- `SIGCHLD`

這些名稱常數。

### 輔助函式

幾個常用的輔助工具：

- `strsignal(signo)`：把 signal 轉成人類可讀字串
- `psignal(signo, prefix)`：印出 signal 說明
- `psiginfo(info, prefix)`：印出 `siginfo_t` 內容摘要

這些在除錯 signal 問題時很好用。

## 10.23 Summary

這章的真正核心不是「signal 有很多名字」，而是：

- signal 是非同步事件
- 非同步就意味著 race、reentrancy、interrupted syscalls
- 可靠 signal 程式設計靠的是正確的 mask 管理與等待模式

如果你要把這章濃縮成一組實務原則，大概就是：

- 用 `sigaction`，不要迷信老式 `signal`
- handler 裡只做最少的事
- 知道 `EINTR` 是什麼
- 用 `sigprocmask` 保護 critical section
- 用 `sigsuspend` 做正確等待
- 不要把 signal 當精準事件佇列，除非你很清楚自己在用 real-time signal

### 這章讀完你應該真的會的事

- 解釋 signal 的 generation / pending / delivery / blocked 模型
- 分辨 `SIG_DFL`、`SIG_IGN`、custom handler 的差異
- 知道為什麼 handler 裡不該用 `printf`、`malloc`
- 會用 `sigaction` 安裝可靠 handler
- 會用 `sigprocmask` 與 `sigsuspend` 解 race
- 會判讀 `EINTR` 與被 signal 打斷的 system call 行為

### 這章最容易踩坑的地方

- 用 `signal()` 寫新程式，卻沒意識到歷史語意差異
- 在 handler 裡呼叫不具 async-signal-safe 保證的函式
- 以為 blocked signal 等於 ignored signal
- 用 `pause()` 等 signal，卻忽略檢查條件與睡眠之間的 race
- 把傳統 signals 當成可靠排隊事件來源
- 忘記 `SIGKILL`、`SIGSTOP` 不能被 catch / ignore / block

### 建議你現在立刻動手做

1. 寫一個程式用 `sigaction` 捕捉 `SIGINT`
2. 在 handler 裡只設 `volatile sig_atomic_t` flag
3. 主流程用 `sigprocmask + sigsuspend` 等待該 flag
4. 再試著做一個 `alarm` timeout 版 `read`
5. 用 `Ctrl-C`、`Ctrl-Z`、`kill -TERM`、`kill -USR1` 觀察不同 signal 行為

### 一句總結

Chapter 10 真正要你學會的是：signal 不是背表格，而是學會在非同步世界裡維持程式狀態一致；做得到這件事，你才算真的會寫 UNIX signal code。
