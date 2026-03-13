# Chapter 9 - Process Relationships

## 這章在做什麼

如果 Chapter 8 講的是「一個 process 怎麼出生、執行、結束」，那 Chapter 9 講的就是：

- 多個 process 之間怎麼形成組織
- shell 為什麼能做 foreground / background job control
- terminal 為什麼知道 `Ctrl-C` 要打給誰
- `Ctrl-Z`、`fg`、`bg` 背後到底是什麼機制

這章最容易讓初學者卡住的原因是：  
很多人只記得 `PID` 和 `PPID`，但 UNIX 真正的互動控制還依賴：

- `process group`
- `session`
- `controlling terminal`
- `foreground process group`

你把這些關係看懂之後，shell 的很多行為就不再是魔法。

## 本章小節地圖

- `9.1 Introduction`
- `9.2 Terminal Logins`
- `9.3 Network Logins`
- `9.4 Process Groups`
- `9.5 Sessions`
- `9.6 Controlling Terminal`
- `9.7 tcgetpgrp, tcsetpgrp, and tcgetsid Functions`
- `9.8 Job Control`
- `9.9 Shell Execution of Programs`
- `9.10 Orphaned Process Groups`
- `9.11 FreeBSD Implementation`
- `9.12 Summary`

## 先抓住這章最重要的心智模型

### `parent-child` 關係只解決「誰生出誰」

`fork` 只告訴你：

- 哪個 process 建立了哪個 child
- 之後誰可以 `wait` 誰

但它沒有解決：

- 一整條 pipeline 應該被當成一份 job 來管
- terminal 輸入應該交給哪一群 process
- `Ctrl-C` 要一次打斷哪幾個 process

這些都不是 `PPID` 可以單獨處理的。

### `process group` 是 job control 的基本單位

你可以把 `process group` 想成：

- 一組相關 process 的「管理單位」

典型例子：

- `vim` 可能自己就是一個 process group
- `grep foo x | sort | less` 這整條 pipeline 也常被放進同一個 process group

terminal 送 signal 時，通常不是只送給一個 `PID`，而是送給 foreground process group。

### `session` 是更大的登入上下文

一個 session 底下可以有多個 process group。  
最典型的 session，就是：

- 一次登入
- 一個 login shell
- 該 shell 啟動的前景與背景 jobs

### `controlling terminal` 把 session 和 terminal 綁起來

一個 session 最多有一個 controlling terminal。  
terminal driver 會追蹤：

- 這個 terminal 屬於哪個 session
- 目前哪個 process group 是 foreground

因此它才知道：

- `Ctrl-C` 要送 `SIGINT`
- `Ctrl-Z` 要送 `SIGTSTP`
- 背景 process 不可以亂讀 terminal

## 9.1 Introduction

這章一開始就在鋪路：  
UNIX 的 process 並不是一坨互相獨立的 `PID`，而是有階層與群組關係。

APUE 這章的真正主題是：

- shell 如何管理 jobs
- terminal 如何和 jobs 互動

所以這章雖然看起來在講一些名詞，但實際上是在講 shell 的運作模型。

## 9.2 Terminal Logins

### 從終端登入到 shell 啟動，大致會經過哪些步驟

傳統 terminal login 的流程可以粗略想成：

1. `init` / `launchd` / `systemd` 類的啟動者準備好某個 terminal line
2. 啟動類似 `getty` 的程式
3. `getty` 打開 terminal，設定 line discipline，讀取使用者名稱
4. `getty` `exec` 成 `login`
5. `login` 驗證密碼、設定帳號資訊、切換身份
6. `login` 最後 `exec` 成 login shell

你可以把它想成：

```text
system starter -> getty -> login -> -bash
```

注意最後那個 shell 常常不是單純叫 `bash`，而是：

```text
-bash
```

也就是 `argv[0]` 前面帶一個 `-`，表示它是 `login shell`。

### `getty` 做的事，不只是顯示 login prompt

很多人以為 `getty` 只是印出：

```text
login:
```

其實它還負責很重要的 terminal 初始化，例如：

- 打開對應的 terminal device
- 把該 terminal 接到 `stdin` / `stdout` / `stderr`
- 設定 baud rate 或其他 terminal attributes
- 讀取使用者輸入的帳號名稱

換句話說，當 `login` 跑起來時，檔案描述元 `0`、`1`、`2` 通常已經指向那個 terminal。

### `login` 的工作是把「認證成功」變成「完整的使用者 session」

使用者成功登入後，`login` 不只是放行而已，它通常還會做：

- 設定 real / effective user ID 與 group ID
- 設定 supplementary groups
- `chdir` 到使用者家目錄
- 準備 environment variables
- 設定 `HOME`
- 設定 `SHELL`
- 設定 `USER` / `LOGNAME`
- 設定 `PATH`

然後才 `exec` 成使用者的 shell。

這裡有個很值得記住的觀念：

- shell 不是直接從 kernel 憑空生出來
- 它是從 login program 經過一串 process control 轉換後來的

## 9.3 Network Logins

### 網路登入本質上也在模擬一個 terminal 環境

APUE 接著說明 network login。  
概念上跟 terminal login 很像，只是資料來源不是實體終端，而是 network connection。

以傳統 `telnetd` 類的服務為例，流程大概是：

1. 某個網路服務 daemon 接到遠端連線
2. 它建立 child process 來服務這個連線
3. 配置一個 pseudo terminal (`pty`)
4. 把網路資料和 `pty` 串起來
5. 啟動 `login` 或直接啟動 shell

因此對後面的 shell 來說，它看到的仍然像是一個 terminal。

### 為什麼要用 `pseudo terminal`

因為很多互動式程式都需要 terminal 語意，例如：

- line editing
- job control
- special characters
- terminal-generated signals

如果只是把 network socket 直接接給 shell，不一定能得到完整的 terminal 行為。  
`pty` 的角色就是：

- 在 kernel 裡提供一對像終端一樣的 master / slave 端點

shell 通常接在 slave 端，所以它會以為自己真的在跟 terminal 溝通。

### 重點不是 `telnet`，而是「遠端登入最後仍會回到 session + terminal 模型」

即使今天不是 `telnetd`，而是其他遠端登入系統，你都可以用同樣角度理解：

- 需要有某種 session
- 需要有某種 terminal 或 terminal-like 裝置
- shell 還是要知道誰是 foreground job

這也是為什麼這章雖然在講 terminal login 和 network login，看起來像兩個主題，實際上它們是同一個模型的兩種入口。

## 9.4 Process Groups

### 什麼是 `process group`

`process group` 是一群相關 process 的集合。  
每個 process 都有一個 `process group ID` (`PGID`)。

幾個重要規則：

- 一個 process 一次只屬於一個 process group
- 一個 process group 可以有多個 process
- `PGID` 通常等於該 group leader 的 `PID`

所以 group leader 的特徵是：

- `PID == PGID`

### 為什麼需要 group，而不是只靠單一 `PID`

因為 shell 常常要把多個 process 當成一個 job 管。  
例如：

```sh
grep foo x | sort | less
```

這不是一個 process，而是多個 process 組成的 pipeline。  
但使用者在 shell 看來，這常常是一份 job。

如果你按：

- `Ctrl-C`
- `Ctrl-Z`

你通常預期整條 pipeline 一起被影響，不是只打中其中一個 process。  
這就是 process group 的用途。

### 常見 API

本章會碰到這幾個函式：

- `getpgrp()`
- `getpgid(pid_t pid)`
- `setpgid(pid_t pid, pid_t pgid)`

你可以先粗略記：

- `getpgrp()`：拿我自己的 `PGID`
- `getpgid(pid)`：拿別人的 `PGID`
- `setpgid()`：把某個 process 放進某個 group

### `setpgid` 最常見的用途

shell 在 `fork` child 之後，常會讓 child 進入新的 process group。  
例如啟動一個新 job：

```c
pid_t pid = fork();
if (pid == 0) {
    setpgid(0, 0);
    execlp("vi", "vi", (char *)0);
    _exit(127);
}

setpgid(pid, pid);
```

### 為什麼 parent 和 child 都可能呼叫 `setpgid`

這是個很經典的 race-avoidance 技巧。

如果只有 child 呼叫：

- parent 可能太快往下走
- 來不及在正確時機把 terminal foreground group 切過去

如果只有 parent 呼叫：

- child 可能太快 `exec`
- 或已經開始執行，造成時序問題

因此 shell 常常兩邊都呼叫一次 `setpgid`，誰先成功都可以。

### `fork` 與 `exec` 對 process group 的影響

- `fork` 後，child 一開始繼承 parent 的 `PGID`
- `exec` 不會改變 `PGID`

這句很重要。  
`exec` 只換程式映像，不會幫你重組 job control 結構。

## 9.5 Sessions

### 什麼是 `session`

如果說 process group 是 job control 的單位，那 session 就是更大的「登入上下文」。

一個 session 可以包含：

- 一個或多個 process groups

常見情境是：

- 一個 login shell 當 session leader
- 它底下有前景 job
- 也有背景 job

### `setsid` 做了什麼

建立新 session 的核心函式是：

```c
pid_t setsid(void);
```

當它成功時，會同時發生三件事：

1. 呼叫者變成新的 session leader
2. 呼叫者變成新的 process group leader
3. 呼叫者失去 controlling terminal

這三件事很常一起被考，也很常一起在 daemon 程式裡出現。

### 為什麼 `setsid` 不允許 process group leader 直接呼叫

這是為了避免 session / process group 結構變得模糊或衝突。  
所以常見做法是：

1. 先 `fork`
2. parent 結束
3. child 因為不是 process group leader，所以可以 `setsid`

簡化版寫法：

```c
pid_t pid = fork();
if (pid < 0) {
    return 1;
}
if (pid > 0) {
    _exit(0);
}

if (setsid() < 0) {
    return 1;
}
```

這就是很多 daemon 啟動步驟的核心片段。

### `getsid`

```c
pid_t getsid(pid_t pid);
```

它會回傳該 process 所屬 session 的 session leader 的 `PID`。  
你可以把它理解成該 session 的 `SID`。

### 關係圖

這張圖很值得記：

```text
session
  -> process group
       -> process
       -> process
  -> process group
       -> process
```

也就是：

- session 包 process group
- process group 包 process

## 9.6 Controlling Terminal

### `controlling terminal` 是 session 跟 terminal 之間的綁定

一個 session 最多有一個 controlling terminal。  
通常登入之後，那個使用中的 terminal 就會成為該 session 的 controlling terminal。

### 誰是 `controlling process`

建立這個 terminal 關聯的 session leader，常被稱為 controlling process。  
不過你在實作時，更重要的是理解：

- terminal 知道目前哪個 session 在用它
- terminal 也知道該 session 裡哪個 process group 是 foreground

### foreground group 和 background groups

同一個 controlling terminal 下：

- 只能有一個 foreground process group
- 其他都是 background process groups

這是 job control 的核心限制。  
terminal 的輸入與 terminal-generated signals，通常只直接服務 foreground group。

### 為什麼 terminal driver 要這樣設計

因為如果背景 job 也能隨便讀 terminal，畫面和輸入就會完全混亂。  
所以系統必須有規則：

- 誰可以讀
- 誰在前景
- special key 送給誰

### `/dev/tty` 的意義

`/dev/tty` 可以看成：

- 目前 process 的 controlling terminal 的別名

它很有用，因為就算你的 `stdin` / `stdout` 被 redirect 了，你還是可能想直接跟 controlling terminal 溝通。

例如：

- 程式輸出結果到檔案
- 但錯誤訊息或互動提示仍要寫到真正終端

### 斷線或 hangup 的影響

如果 controlling terminal 消失，例如：

- 實體線路斷掉
- 遠端連線結束

系統通常會送 `SIGHUP` 給相關 process。  
這也是為什麼很多 daemon 或長時間背景工作會特別處理 `SIGHUP`。

## 9.7 `tcgetpgrp`, `tcsetpgrp`, and `tcgetsid` Functions

### 這組 API 是 shell 管理 terminal 前景權的工具

最常見的是：

- `tcgetpgrp(fd)`
- `tcsetpgrp(fd, pgid)`

其中：

- `tcgetpgrp`：查某個 terminal 目前的 foreground `PGID`
- `tcsetpgrp`：把 terminal 的 foreground `PGID` 改成某個 group

### shell 為什麼需要 `tcsetpgrp`

假設 shell 啟動 `vi` 作為前景 job，大致流程是：

1. `fork`
2. child 放進新的 process group
3. shell 用 `tcsetpgrp` 把 terminal 前景權交給那個 group
4. shell 等待 job 結束或停止
5. job 結束後，shell 再把 terminal 前景權拿回來

如果沒有這一步，`Ctrl-C` 和 `Ctrl-Z` 還是會打到 shell，而不是打到 `vi`。

### `tcgetsid`

```c
pid_t tcgetsid(int fd);
```

這個函式可以查：

- 某個 terminal 屬於哪個 session

雖然平常應用程式不一定常直接用它，但它有助於你理解 terminal、session、foreground group 之間的綁定。

### 使用上的注意點

如果背景 process group 裡的 process 嘗試改 terminal foreground 狀態，可能會被送：

- `SIGTTOU`

這不是 API 的小細節，而是 job control 保護規則的一部分。

## 9.8 Job Control

### `job control` 到底在控制什麼

job control 的目標不是管理所有 process，而是管理：

- 跟同一個 terminal session 互動的 process jobs

也就是讓使用者可以做這些事：

- 在前景跑程式
- 把 job 丟到背景
- 暫停 job
- 繼續 job
- 查 jobs 狀態

### 一份 job 常常對應一個 process group

這是最重要的實務對應：

- shell 裡的一份 job
- kernel 裡的一個 process group

尤其是 pipeline：

```sh
find . -type f | sort | less
```

這三個 process 雖然 `PID` 不同，但通常會在同一個 `PGID` 裡。

### terminal special characters 會變成 signals

幾個最重要的對應：

- `Ctrl-C` -> `SIGINT`
- `Ctrl-\` -> `SIGQUIT`
- `Ctrl-Z` -> `SIGTSTP`

而且這些 signal 的目標，通常是 foreground process group，不是單一 process。

### 背景 job 讀 terminal 會怎樣

如果 background process group 嘗試從 controlling terminal 讀資料，通常會收到：

- `SIGTTIN`

這是系統在說：

- 你不是 foreground job，不能直接來搶 terminal 輸入

### 背景 job 寫 terminal 會怎樣

寫入的規則比較寬鬆。  
很多情況下背景 job 還是能輸出到螢幕。

但如果 terminal 開了 `TOSTOP`，那背景 job 嘗試寫 terminal 時，可能會收到：

- `SIGTTOU`

所以你有時候會看到背景工作把畫面弄亂，有時候又會被停掉，這跟 terminal flags 有關。

### job control 其實是三方合作

不是 shell 一個人完成全部工作，而是三方一起配合：

- shell
- terminal driver
- kernel signal / process group 機制

少了任何一個，完整 job control 都做不起來。

## 9.9 Shell Execution of Programs

### 沒有 job control 的 shell，模型比較簡單

在沒有 job control 的情況下，shell 執行一個命令時，大致就是：

1. `fork`
2. child `exec`
3. parent `wait`

child 往往沿用 shell 原本的 process group。  
這時候 terminal 不太需要做前景切換。

### 有 job control 的 shell，會多做兩件事

如果 shell 支援 job control，那執行前景 job 時通常還要：

1. 為該 job 建立新的 process group
2. 把 controlling terminal 的 foreground group 切到那個 job

也就是：

```text
shell fork child
child / parent setpgid
shell tcsetpgrp -> child group
child exec program
shell wait job state change
shell tcsetpgrp -> shell group
```

### 背景 job 的差別

背景 job 也常被放到新的 process group，差別是：

- shell 不會把 terminal foreground 權交給它

因此背景 job：

- 可以繼續跑
- 但不應該直接讀 terminal

### pipeline 的實作差異

不同 shell 對 pipeline 建構順序可能不同：

- 有的 shell 由 shell 當所有 pipeline processes 的 parent
- 有的 shell 讓 pipeline 中某些 process 再生出其他 process

但不管細節如何，重點通常都一樣：

- 同一條 pipeline 會被放進同一個 process group

這樣才方便把它當成同一份 job 管理。

### 你應該從 shell 行為反推底層模型

看到這裡時，最有用的學習方式不是死背 API，而是反推：

- 為什麼 `fg` 可以把 job 拉回前景
- 為什麼 `Ctrl-C` 不會直接殺掉 shell 自己
- 為什麼 `cat &` 嘗試讀輸入時會被停住

如果你能用 process group + session + controlling terminal 解釋這些現象，就表示你真的懂了。

## 9.10 Orphaned Process Groups

### 什麼叫 `orphaned process group`

這個概念乍看很繞，但其實是在處理一個實際問題：

- 如果一整組 process 已經沒有合適的上層管理者了，那被 stop 住的它們該怎麼辦

POSIX 對 orphaned process group 有精確定義，但你先抓核心直覺就好：

- 這個 group 裡的 process 已經失去原本那種能管理它們的父層關係

### 為什麼這很重要

想像一個 stopped background job 還留在系統裡，但原本的 shell 已經退出。  
如果系統不處理，它可能會永遠停在那裡，沒有人再把它喚醒。

因此 POSIX 規定：

- 如果某個 process group 變成 orphaned
- 而且裡面有 stopped process

系統就要送：

1. `SIGHUP`
2. `SIGCONT`

### 為什麼是先 `SIGHUP` 再 `SIGCONT`

直覺上可以理解成：

- 先通知「你的終端 / 原控制情境沒了」
- 再讓你醒來繼續跑，決定要怎麼善後或退出

很多 shell 在結束前，也會主動對 background jobs 做類似通知。

### orphaned background group 讀 terminal 的特殊處理

正常背景 job 讀 terminal 會收到 `SIGTTIN`。  
但如果它所在的是 orphaned process group，系統不能再把它 stop 住，否則可能永遠沒人管。

所以這時常見行為是：

- 相關系統呼叫失敗，回傳 `EIO`

這個細節很容易忘，但很能體現 POSIX 設計背後的考量：

- 不要讓 orphaned stopped jobs 永遠卡死

## 9.11 FreeBSD Implementation

### 這一節不是要你背 kernel 結構欄位

APUE 在這裡拿 FreeBSD 當例子，是為了讓你看到：

- session
- process group
- tty
- process

在 kernel 裡其實是有具體資料結構互相連起來的。

重點不是死背某個 struct 名稱，而是理解關係：

- 一個 `tty` 會知道自己的 foreground process group
- 一個 `process group` 會知道自己屬於哪個 session
- 一個 `session` 會知道自己的 controlling terminal

### 你要記的是「連接圖」，不是「欄位名稱」

把這節當成 wiring diagram 比較好：

```text
session <-> controlling tty
session -> process groups
process group -> member processes
process -> pid, ppid, pgid, sid
```

這樣你在除錯 shell、daemon、job control 行為時，腦中會有一張清楚的圖。

## 9.12 Summary

這章真正要你建立的是一個 shell / terminal 的系統觀：

- `PID` 只能描述單一 process
- `process group` 才是 job control 的基本管理單位
- `session` 把一批 jobs 放進同一個登入上下文
- `controlling terminal` 讓 terminal 與 session 綁定
- foreground / background 的差別，會直接影響 input 與 signals

### 這章讀完你應該真的會的事

- 解釋 `process group`、`session`、`controlling terminal` 三者的關係
- 解釋為什麼 shell 要用 `setpgid` 與 `tcsetpgrp`
- 解釋為什麼 `Ctrl-C`、`Ctrl-Z` 會作用在整個前景 job
- 解釋背景 job 為什麼不能隨便讀 terminal
- 解釋 orphaned process group 為什麼需要 `SIGHUP` 與 `SIGCONT`

### 這章最容易踩坑的地方

- 以為 job control 只靠 `PPID`，其實主要靠 `PGID`
- 以為 `exec` 會重設 process group，實際上不會
- 以為一個 terminal 可以同時有多個 foreground jobs，實際上不行
- 忘記 `Ctrl-C` 打的是 foreground process group，不是 shell 指定的單一 `PID`
- 不理解 shell 常同時在 parent / child 兩邊呼叫 `setpgid` 是為了避開 race

### 建議你現在立刻動手做

1. 寫一個小 shell 雛形，只支援 `fork`、`exec`、`waitpid`
2. 幫它加上 `setpgid`，把每個命令放進自己的 process group
3. 再加上 `tcsetpgrp`，讓前景 job 能正確接收 `Ctrl-C`
4. 用 `cat &`、`sleep 100`、`vim` 這類程式觀察行為差異

### 一句總結

Chapter 9 的核心不是背名詞，而是理解：UNIX 之所以能有像 shell 一樣自然的互動行為，是因為 process 不只是一個個 `PID`，而是被組織成 `process group + session + controlling terminal` 這套完整模型。
