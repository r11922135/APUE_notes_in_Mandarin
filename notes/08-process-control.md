# Chapter 8 - Process Control

## 這章在做什麼

這章正式進入 UNIX process control 的核心地帶。  
如果 Chapter 7 是在講「一個 process 自己身上有哪些狀態」，那 Chapter 8 就是在講：

- process 怎麼被建立
- 怎麼換程式內容
- 怎麼結束
- parent 怎麼等 child
- 怎麼處理 zombie
- `exec` 之後哪些東西留下來、哪些被重設
- 權限和 user/group IDs 怎麼跟 process control 互動

這章最重要的觀念其實只有幾個 primitive：

- `fork`
- `exec` family
- `_exit` / `exit`
- `wait` / `waitpid`

後面很多高階行為，例如 shell、daemon、popen、system，本質上都是這幾個東西組起來的。

## 本章小節地圖

- `8.1 Introduction`
- `8.2 Process Identifiers`
- `8.3 fork Function`
- `8.4 vfork Function`
- `8.5 exit Functions`
- `8.6 wait and waitpid Functions`
- `8.7 waitid Function`
- `8.8 wait3 and wait4 Functions`
- `8.9 Race Conditions`
- `8.10 exec Functions`
- `8.11 Changing User IDs and Group IDs`
- `8.12 Interpreter Files`
- `8.13 system Function`
- `8.14 Process Accounting`
- `8.15 User Identification`
- `8.16 Process Scheduling`
- `8.17 Process Times`
- `8.18 Summary`

## 先抓住這章最重要的心智模型

### UNIX process control 的核心流程

絕大多數情況都能簡化成這條主線：

1. `fork` 出 child
2. child 視需要調整一些狀態
3. child `exec` 新程式
4. parent `wait` 或繼續做自己的事

### `fork` 與 `exec` 是兩件事

這是這章最重要的分界。

- `fork`：複製出一個新 process
- `exec`：用新程式替換目前 process 的程式映像

所以：

- `fork` 會產生新 PID
- `exec` 不會產生新 PID

### `wait` 解的是「誰來收 child 的結束狀態」

如果 parent 不收，child 可能變 `zombie`。  
所以 `wait` 不是只是「等一下」，它也是一種資源回收。

## 8.1 Introduction

這章主題很直白：

- process creation
- program execution
- process termination
- 各種 user/group IDs 與 process control 的互動
- interpreter file
- `system`
- process accounting

也就是說，這章不是只在講 `fork` 而已，而是在講 process 的整個生命週期。

## 8.2 Process Identifiers

### `pid`

每個 process 都有唯一的 `process ID`，型別通常是：

- `pid_t`

### PID 會重用

這點很重要。

- PID 在某一時刻是唯一
- 但 process 結束後，之後可能被別人重用

所以如果你只是看到「有同樣 PID」，不代表那一定還是同一個 process。

### 特殊 PID

歷史上常見：

- `0`：kernel scheduler / swapper 一類的內核 process
- `1`：`init` 或現代等效的系統啟動 process

PID 1 的角色非常重要，因為 orphan process 最後通常會被它收編。

### 常用查詢函式

```c
#include <unistd.h>

pid_t getpid(void);
pid_t getppid(void);
uid_t getuid(void);
uid_t geteuid(void);
gid_t getgid(void);
gid_t getegid(void);
```

分別拿：

- 自己 PID
- parent PID
- real/effective user ID
- real/effective group ID

## 8.3 `fork`

```c
#include <unistd.h>

pid_t fork(void);
```

### `fork` 最重要的語意

它呼叫一次，回傳兩次：

- parent 看到的是 child PID
- child 看到的是 `0`

### 為什麼 child 回 `0`

因為 child 永遠可以用：

- `getppid()`

知道自己的 parent。  
但 parent 可能同時有很多 child，所以 `fork` 必須直接告訴 parent「你剛生的是哪個 PID」。

### child 會複製到什麼

child 基本上是 parent 的副本，包含：

- data space
- heap
- stack
- open file descriptors
- signal dispositions
- current working directory
- environment
- resource limits

但 text segment 常常是共用的。

### Copy-on-Write

現代系統通常不是真的一開始就整份 memory 複製。  
而是：

- parent 與 child 先共享頁面
- 直到某一方要修改時，kernel 才真的 copy 那頁

這就是 `copy-on-write`。

### `fork` + stdio 的經典坑

如果 `fork` 前你有一個還沒 flush 的 stdio buffer，child 會繼承這份 buffer。  
結果是：

- 同一段 `printf` 內容可能被 parent / child 各 flush 一次

這也是為什麼「互動式終端機一份、redirect 到檔案卻兩份」這種現象很常在面試或考試出現。

### `fork` 後的 file sharing

這點非常重要：

- parent 和 child 的 `fd` 不是互不相干
- 它們通常共享同一個 open file description

所以：

- file offset 是共享的
- file status flags 也是共享的

這個心智模型在 Chapter 3 其實就埋好了，這裡真正發揮作用。

### `fork` 的兩個典型用途

#### 1. 一分為二，各跑不同程式段

像 server：

- parent 等請求
- child 處理請求

#### 2. child 馬上 `exec` 別的程式

像 shell：

- 先 `fork`
- child `exec`
- parent 視情況 `wait`

### 範例：觀察 parent / child 的變數彼此獨立

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int glob = 1;

int main(void) {
    int local = 10;
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        glob++;
        local++;
        printf("child:  glob=%d local=%d\n", glob, local);
    } else {
        sleep(1);
        printf("parent: glob=%d local=%d\n", glob, local);
    }
    return 0;
}
```

### 但檔案 offset 通常共享

這要和前面「記憶體各自獨立」分開理解。  
變數空間各有一份，不代表 open file state 也各有一份。

## 8.4 `vfork`

### 先講結論

- 這是歷史上為了效率出現的東西
- 現在通常不建議新程式依賴它

### `vfork` 跟 `fork` 差在哪

`vfork` 的 child 在 `exec` 或 `_exit` 之前，會直接借用 parent 的 address space。

這代表：

- child 不能亂改資料
- 不能隨便 return
- 不該做複雜 function call

### 為什麼危險

因為 child 改到的東西，其實可能就是 parent 的 stack / data。  
這使得它雖然快，但很容易出現 undefined behavior。

### 另一個差異

`vfork` 保證 child 先跑，直到 child `exec` 或 `_exit` 之後，parent 才恢復。

### `_exit` 而不是 `exit`

這裡特別要用 `_exit`，因為 `exit` 可能動到 stdio 等 user-space 狀態，child 又是在借 parent 的地址空間，副作用會很可怕。

## 8.5 exit 相關函式與 process 結束

這節把 Chapter 7 的 process termination，拉回到 parent-child 關係脈絡裡看。

### 正常與異常終止

process 不管怎麼死，kernel 最後都會：

- 關閉 descriptors
- 釋放 memory
- 保留少量 termination info 給 parent 撿

### 為什麼 kernel 要保留 termination info

因為 parent 之後可能會想知道：

- child 是正常結束還是被 signal 殺掉
- exit status 是多少
- 有沒有 core dump

### zombie 是什麼

child 已經結束，但 parent 還沒 `wait` 它時：

- process 本體已經死了
- 但 termination status 還留在 kernel

這狀態就叫 `zombie`。

### orphan 是什麼

如果 parent 先死，child 還活著：

- child 變 orphan
- 通常會被 PID 1 收編

### 為什麼 orphan 不會永遠變 zombie

因為像 `init` 這種收編者，設計上就會 `wait` 它的 child，避免系統被 zombie 塞滿。

## 8.6 `wait` 和 `waitpid`

```c
#include <sys/wait.h>

pid_t wait(int *statloc);
pid_t waitpid(pid_t pid, int *statloc, int options);
```

### `wait` 的三種可能

1. block，因為 children 都還在跑
2. 立即回傳某個已結束 child 的 status
3. 立即失敗，因為根本沒有 child

### `waitpid` 比 `wait` 多什麼

#### 1. 可指定等誰

- 指定某個 PID
- 指定某個 process group
- 或像 `wait` 一樣等任意 child

#### 2. 可 nonblocking

- `WNOHANG`

#### 3. 可配合 job control

- `WUNTRACED`
- `WCONTINUED`

### `pid` 參數語意

- `pid == -1`：任意 child
- `pid > 0`：指定 child PID
- `pid == 0`：同 process group 的任意 child
- `pid < -1`：指定 process group

### 如何解讀 `status`

APUE 用這幾個 macro：

- `WIFEXITED`
- `WEXITSTATUS`
- `WIFSIGNALED`
- `WTERMSIG`
- `WCOREDUMP`（非所有平台都保證）
- `WIFSTOPPED`
- `WSTOPSIG`
- `WIFCONTINUED`

### 範例：wait status 解讀

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>

int main(void) {
    int status;
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0)
        exit(7);

    if (waitpid(pid, &status, 0) == -1) {
        perror("waitpid");
        return 1;
    }

    if (WIFEXITED(status))
        printf("exit status = %d\n", WEXITSTATUS(status));

    return 0;
}
```

### double fork 避免 zombie

書裡有個很經典技巧：

1. parent `fork` first child
2. first child 再 `fork` second child
3. first child 立刻 exit
4. parent wait first child
5. second child 被 PID 1 收編

這樣 original parent 就不用直接負責 second child 的回收。

## 8.7 `waitid`

`waitid` 是比較彈性的版本。

### 它的特色

- 用 `idtype` + `id` 分開指定要等誰
- 結果放在 `siginfo_t`
- 表達能力比 `waitpid` 更細

常見 `idtype`：

- `P_PID`
- `P_PGID`
- `P_ALL`

### 實務上怎麼看

如果你只是在一般 UNIX programming 語境下寫程式：

- `waitpid` 已經夠常用

但知道 `waitid` 存在很重要，因為它代表 wait API 也往更結構化的資訊前進。

## 8.8 `wait3` 和 `wait4`

這是 BSD 系列擴充。

### 它們多了什麼

除了 termination status，還能拿：

- resource usage

也就是類似：

- CPU time
- memory / I/O 等統計

### 心智模型

如果 `waitpid` 是「孩子死了沒、怎麼死的」，那 `wait3/wait4` 就是「孩子怎麼死的 + 生前用掉多少資源」。

## 8.9 Race Conditions

這節的重點不是某個 API，而是：

- parent 和 child 誰先跑是不保證的

### `sleep` 不是同步

很多初學者會寫：

- parent `sleep(1)`，假設 child 一定先做好事

這只是碰運氣，不是同步。

### race condition 典型長相

- 你以為某個狀態已經被 child 設好
- 但 parent 先往下跑了
- 或反過來

### 正確解法

要用真正的同步機制，例如：

- signal-based synchronization
- pipe
- 後面會出現的其他 IPC / synchronization tools

APUE 在 Chapter 10 也會給 parent-child sync 的 signal 範例。

## 8.10 `exec` 家族

這是整章另一個超大重點。

### `exec` 的本質

不是生新 process，而是：

- 把目前這個 process 的 text / data / heap / stack 換成新程式

所以：

- PID 不變
- 程式內容與 `main` 起點換掉

### 七個版本

```c
execl
execv
execle
execve
execlp
execvp
fexecve
```

### 名字怎麼記

#### `l`

- list
- argument 一個一個列出

#### `v`

- vector
- 傳 `argv[]`

#### `e`

- 額外指定 environment

#### `p`

- 用 `PATH` 搜尋 executable

#### `f`

- 用 file descriptor 指定 executable

### `exec` matrix 的核心理解

- `execl/execv`：要給 pathname
- `execlp/execvp`：給 filename，會查 `PATH`
- `execle/execve/fexecve`：自己指定 environment

### `execve` 是核心版本

很多系統上：

- 真正的 system call 只有 `execve`
- 其他只是 library 包裝

### `argv[0]` 由呼叫者決定

這點超重要。

你可以傳：

- 完整路徑
- 檔名
- 特別前綴

例如 `login shell` 常會把 `argv[0]` 前面加 `-`，讓 shell 知道自己是 login shell。

### PATH 搜尋

如果是 `execlp` / `execvp`：

- 若 filename 含 `/`，就直接當 pathname
- 否則依 `PATH` 各個 prefix 搜尋

### `fexecve` 的安全意義

這個版本很值得記：

- 先打開 / 驗證檔案
- 再透過 `fd` 執行

這能降低 path-based TOCTTOU race。

### `exec` 後哪些東西會保留

很多屬性會留下來，例如：

- PID / PPID
- real UID / GID
- process group / session
- cwd
- root directory
- umask
- signal mask
- resource limits

### 哪些東西會重設或改變

#### 記憶體映像整個換掉

- text / data / heap / stack 全新

#### signal disposition

- 原本被 catch 的 signal 會回到 default
- 原本被 ignore 的 signal 通常保持 ignore

#### effective UID / GID

如果新執行檔有：

- set-user-ID
- set-group-ID

則 effective IDs 可能變。

#### open file descriptors

預設會保留，但若設了：

- `FD_CLOEXEC`

就會在 `exec` 時自動關掉。

### 範例：`execvp`

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    char *argv[] = { "ls", "-l", NULL };
    execvp("ls", argv);
    perror("execvp");
    return 1;
}
```

## 8.11 改變 User IDs 與 Group IDs

這節很關鍵，因為它把 Chapter 4 的 set-user-ID 概念和 process control 接起來。

### `setuid` / `setgid`

```c
#include <unistd.h>

int setuid(uid_t uid);
int setgid(gid_t gid);
```

### 最核心規則

#### 如果 process 有 superuser privilege

- `setuid` 會改 real / effective / saved set-user-ID

#### 如果沒有 superuser privilege

- 只能把 effective user ID 改成 real user ID 或 saved set-user-ID
- 不能亂設成任意值

### saved set-user-ID 的價值

這是寫安全 `set-user-ID` 程式的核心。

它讓程式可以：

1. 先降權，用一般使用者身份跑大部分邏輯
2. 需要特權時，再暫時切回 saved privileged ID

這比一路帶著高權限跑安全得多。

### `setreuid` / `setregid`

歷史上 BSD 提供這組，能更靈活地交換 real/effective IDs。

### `seteuid` / `setegid`

這組只改 effective ID，不像 `setuid` 那麼全面。

### least privilege

書裡特別強調：

- 程式應該在需要時才拿高權限
- 平常盡量用最低權限執行

這是安全程式設計最重要的原則之一。

## 8.12 Interpreter Files

這就是我們熟悉的 `shebang`。

### 格式

```text
#! pathname [optional-argument]
```

最常見是：

```text
#!/bin/sh
```

### kernel 在做什麼

當 `exec` 遇到這種檔案時：

- 真正被執行的不是 script 本身
- 而是第一行指定的 interpreter

### 為什麼這很重要

它讓：

- shell script
- awk script
- perl/python 等解譯器腳本

可以像一般 executable 一樣被直接執行。

### optional argument

第一行還可以帶一個額外參數，這也是 kernel 幫你組進 interpreter 的 argument list。

## 8.13 `system`

```c
#include <stdlib.h>

int system(const char *cmdstring);
```

### 本質

它大致做的是：

1. `fork`
2. child `exec("/bin/sh", "sh", "-c", cmdstring, ...)`
3. parent `waitpid`

### 好處

非常方便。  
很多事情你本來可以自己手刻：

- `fork`
- `exec`
- `wait`

但 `system("date > file")` 一行就解決。

### 回傳值三類

1. `fork` 或 `waitpid` 失敗：回 `-1`
2. shell 根本無法 `exec`：像 `exit(127)`
3. shell 成功執行：回傳 shell 的 termination status

### 為什麼 `system` 不能用在 set-user-ID 程式

因為它會把 command string 丟給 shell parse。  
這會引入太多風險：

- shell expansion
- environment variable 污染
- PATH / IFS 類問題
- 權限混用

所以：

- set-user-ID / set-group-ID program 不該呼叫 `system`

## 8.14 Process Accounting

### 是什麼

某些 UNIX 系統可開啟 process accounting：

- 每個 process 結束時，kernel 寫一筆 binary accounting record

### 會記什麼

通常包括：

- command name
- real UID / GID
- start time
- user CPU time
- system CPU time
- elapsed time
- I/O 統計
- abnormal termination 資訊

### 重點不是背結構每個欄位

而是理解：

- accounting 是「process 結束時寫一筆」
- 不是「每次 exec 就新建一筆」

所以如果 A `exec` 成 B，再 `exec` 成 C，最後只可能看到一筆，且 command name 通常對應最後那個程式，但 CPU 等統計可能是整串累積。

### 另一個重點

record 順序通常是：

- termination order

不是 start order。

## 8.15 User Identification

### 為什麼不只 `getuid`

有時你想知道的不是 UID，而是：

- 這位使用者登入時的 login name 是什麼

### `getlogin`

```c
#include <unistd.h>

char *getlogin(void);
```

### 什麼時候可能失敗

如果 process 沒有連到一個真正的 login terminal，例如 daemon，就可能失敗。

### 為什麼別信 `LOGNAME`

因為 environment variable 可以被 user 改。  
如果你要拿來做驗證，應該優先信系統介面，不要信可被任意修改的環境字串。

## 8.16 Process Scheduling

這節主要講經典 UNIX 的 `nice`。

### 核心概念

- `nice` 越高，priority 越低
- 也就是越「nice」，越禮讓 CPU

### `nice`

```c
#include <unistd.h>

int nice(int incr);
```

它是調整「自己的」nice value。

### `getpriority` / `setpriority`

```c
#include <sys/resource.h>

int getpriority(int which, id_t who);
int setpriority(int which, id_t who, int value);
```

這組可以針對：

- process
- process group
- user

做優先權查詢 / 設定。

### `NZERO`

是系統的預設 nice 基準值。

### 實務理解

別把它想成精密 real-time scheduler 控制，而是：

- 給 scheduler 一個「這個 process 可以比較不急」的 hint

## 8.17 Process Times

```c
#include <sys/times.h>

clock_t times(struct tms *buf);
```

### `struct tms`

包含：

- `tms_utime`：自己的 user CPU time
- `tms_stime`：自己的 system CPU time
- `tms_cutime`：terminated children 的 user CPU time
- `tms_cstime`：terminated children 的 system CPU time

### 重要觀念

`times()` 的回傳值是：

- 某個任意起點以來的 wall clock ticks

所以它的絕對值不太有意義，真正有意義的是：

- 前後兩次呼叫的差值

### 換算成秒

要用：

- `sysconf(_SC_CLK_TCK)`

把 ticks 轉秒。

## 8.18 Summary

### 這章讀完你應該真的會的事

- 理解 `fork` 的雙重回傳與 copy-on-write 模型
- 知道 parent / child 共享 open file state，但不共享一般變數
- 知道 `vfork` 為何危險、為何通常不該新用
- 理解 zombie 與 orphan 的差別
- 會用 `waitpid` 與各種 `WIF...` macro 解讀 child 狀態
- 會分辨七個 `exec` 函式各自的用途
- 知道 `FD_CLOEXEC` 的價值
- 理解 saved set-user-ID 如何支援 least privilege
- 知道 `system` 為何方便、又為何不能亂用在 privileged program
- 會看 `nice`、`times` 這類 process 層級資訊

### 這章最容易踩坑的地方

- 以為 `exec` 會生新 process
- 忘記 `fork` 前 stdio buffer 可能被複製
- child 結束時用 `exit` 而不是 `_exit`
- parent 不 `wait` 導致 zombie
- 把 `wait` 撿到的 status 當成單純 exit code
- 把 `system` 用在 set-user-ID 程式
- 以為 `argv[0]` 一定等於真實執行檔名
- 以為 `nice` 是精準配額，不是 scheduler hint

### 建議你現在立刻動手做

- 改寫一個小程式，同時用 `printf` 和 `write`，觀察 `fork` 後的差異
- 練習 `waitpid` + `WIFEXITED/WIFSIGNALED`
- 自己寫一個簡化版 shell：`fork` + `execvp` + `waitpid`
- 做一個 double-fork 範例，觀察 second child 的 parent PID
- 寫一個 `execle` 範例，傳入自訂 environment

### 一句總結

這章真正教你的，是 UNIX process control 並不靠很多神祕機制，而是靠少數幾個非常穩定、非常原始的 primitive 組出整個 process 生命週期。
