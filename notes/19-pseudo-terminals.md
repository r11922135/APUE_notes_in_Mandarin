# Chapter 19 - Pseudo Terminals

## 這章在做什麼

前一章你學到的是：

- 真正 terminal 裝置的 line discipline 與 `termios`

這一章則是在講一個非常巧妙的技術：

- `pseudo terminal`
- 簡稱 `pty`

它的核心用途是：

- 讓某個 process 以為自己正在跟「真的 terminal」互動
- 但實際上另一端是另一個程式在代理它

這個技術是：

- terminal emulator
- `ssh` / `telnet`
- `script`
- `expect`
- 各種互動式 automation

背後的核心之一。

## 本章小節地圖

- `19.1 Introduction`
- `19.2 Overview`
- `19.3 Opening Pseudo-Terminal Devices`
- `19.4 pty_fork Function`
- `19.5 pty Program`
- `19.6 Using the pty Program`
- `19.7 Advanced Features`
- `19.8 Summary`

## 先抓住這章最重要的心智模型

### `pty` 不是普通 pipe，而是「可被代理的 terminal」

如果你只把 `pty` 想成：

- 一種雙向 byte channel

那你會低估它的關鍵價值。  
`pty` 真正厲害的地方是：

- slave 端看起來像 terminal
- 所以 line discipline、special characters、canonical/noncanonical mode、job control 都能成立

這和普通 pipe 完全不同。

### `pty` 有 master / slave 兩端

你可以先把它想成：

- slave：給被控制的程式用，它覺得自己在連 terminal
- master：給控制程式用，它可以讀寫對方看到的 terminal 流量

這就是為什麼 `pty` 很適合做：

- terminal relay
- 遠端登入
- 自動化互動

### 為什麼有些程式接 pipe 不正常，接 `pty` 就正常

因為很多互動式程式會依據：

- `isatty`
- controlling terminal
- line buffering / terminal mode

去決定行為。

所以：

- 接 pipe 時，它發現自己不是 terminal，行為改變
- 接 `pty` slave 時，它以為自己真的是在 terminal 上，行為就正常了

這個差異，是整章最重要的直覺。

## 19.1 Introduction

APUE 一開始用 network login 當背景。

當使用者從真正 terminal 登入時：

- terminal semantics 是自然存在的

但如果登入來自：

- network connection

那連進來的本來只是一條資料流，不會自動有：

- line discipline
- controlling terminal
- terminal special characters

這時就需要：

- pseudo terminal device driver

來補上 terminal semantics。

也就是說，`pty` 的歷史主因之一，就是：

- 讓不是實體 terminal 的連線，也能像 terminal 一樣工作

## 19.2 Overview

### 典型 `pty` 架構

最經典的流程是：

1. 某個程式先開 `pty master`
2. `fork`
3. child 建立新 session
4. child 開對應的 `pty slave`
5. child 把 slave 接到 `stdin/stdout/stderr`
6. child `exec` 真正要跑的互動程式
7. parent 持有 master，代理資料流

### 從 child 的角度看

child 看到的是：

- 標準輸入 / 輸出 / 錯誤都連到 terminal device

所以它可以正常使用：

- Chapter 18 的 terminal I/O functions
- canonical / noncanonical mode
- signal-generating special characters

### 從 parent 的角度看

parent 持有：

- `pty master`

所以它可以：

- 把資料寫進 master，讓 slave 那邊像是「使用者輸入」
- 從 master 讀資料，看到 slave 那邊吐出的 terminal output

### 這就像雙向 pipe，但多了 terminal semantics

APUE 的說法可以濃縮成：

- `pty` 在資料流上像 bidirectional pipe
- 但因為 slave 上面有 terminal line discipline，所以能力遠超 pipe

### 為什麼這麼多系統都靠 `pty`

因為它能解決好幾類問題：

- network login server
- windowing system terminal emulator
- `script`
- `expect`
- 讓 coprocess 以為自己在 terminal 上，避免 stdio buffering deadlock
- 監看 long-running program 的即時輸出

### 幾個經典使用場景

#### network login server

像：

- `telnetd`
- `rlogind`

這類 server 會把 network connection 接到 `pty master`，  
再讓遠端 shell 跑在 `pty slave` 上。

#### terminal emulator

像圖形介面的 terminal app：

- 每個 shell 視窗其實常常都對應一組 `pty`

terminal emulator 持有 master，shell 跑在 slave。

#### `script`

`script` 把自己插在使用者 terminal 和新的 shell 之間，  
把所有看到的輸入 / 輸出抄一份到檔案。

#### `expect`

它不是只會搬資料，而是會：

- 看輸出內容
- 根據 prompt 決定下一步要送什麼

所以它比單純 relay 的 `pty` 程式更高階。

#### coprocess buffering 問題

如果某個程式在 pipe 上會因 stdio fully buffered 而卡住，  
把它放到 `pty` slave 上，常常能讓它回到：

- line buffered

因為它以為自己是在 terminal 上。

## 19.3 Opening Pseudo-Terminal Devices

### 現代 POSIX / XSI 常見開法

最常見流程是：

1. `posix_openpt`
2. `grantpt`
3. `unlockpt`
4. `ptsname`

### `posix_openpt`

```c
int posix_openpt(int oflag);
```

它負責：

- 打開下一個可用的 `pty master`

常見會用：

- `O_RDWR`

### `grantpt`

```c
int grantpt(int fd);
```

它的角色是：

- 調整對應 slave device 的擁有權 / 權限
- 讓之後真正要使用 slave 的 process 可以打開它

### `unlockpt`

```c
int unlockpt(int fd);
```

它負責：

- 把 slave 解除鎖定，允許被打開

### `ptsname`

```c
char *ptsname(int fd);
```

它讓你知道：

- 這個 master 對應的 slave device 路徑名稱是什麼

### 範例：打開 master 並取得 slave 名稱

```c
#include <fcntl.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    int fdm;
    char *name;

    fdm = posix_openpt(O_RDWR);
    if (fdm < 0) {
        perror("posix_openpt");
        return 1;
    }
    if (grantpt(fdm) < 0 || unlockpt(fdm) < 0) {
        perror("grantpt/unlockpt");
        close(fdm);
        return 1;
    }
    name = ptsname(fdm);
    if (name == NULL) {
        perror("ptsname");
        close(fdm);
        return 1;
    }

    printf("slave = %s\n", name);
    close(fdm);
    return 0;
}
```

### `ptym_open` / `ptys_open`

APUE 接著把上面流程包成兩個 helper：

- `ptym_open`：開 master，回傳 slave 名稱
- `ptys_open`：開 slave

這樣下一節的 `pty_fork` 就能把流程整理得很清楚。

### 為什麼不一次全做完

因為：

- child 要先 `setsid`
- 再開 slave

這樣 slave 才能成為 child 新 session 的 controlling terminal。

### 平台差異：Solaris STREAMS 與 BSD 的 controlling terminal 語意

APUE 在這節也提醒：

- Solaris 上的 `pty` 還可能牽涉 STREAMS modules
- FreeBSD / Linux / Mac / Solaris 在「open slave 是否自動成為 controlling terminal」上也有差異

尤其 FreeBSD 類會需要顯式：

- `ioctl(..., TIOCSCTTY, ...)`

這一點會在 `pty_fork` 再出現。

## 19.4 `pty_fork` Function

這一節把 `pty` 實務用法濃縮成一個很漂亮的 helper：

- `pty_fork`

### 這個 helper 在幹嘛

它把這些事一次包起來：

- 開 master
- 取得 slave 名稱
- `fork`
- child `setsid`
- child 開 slave
- child 讓 slave 變 controlling terminal
- child 把 slave 接到 `stdin/stdout/stderr`

最後結果是：

- parent 拿到 master fd
- child 像在真 terminal 上一樣跑

### 函式介面想傳達的意思

APUE 的版本除了 master fd / child pid 之外，還允許你提供：

- 初始 `termios`
- 初始 `winsize`

這很重要，因為常見需求是：

- 讓 slave 一開始就跟目前使用者 terminal 狀態一致

### `setsid` 為什麼一定要先做

因為你希望 child：

- 建立新 session
- 脫離舊 controlling terminal
- 再把 `pty slave` 變成新 controlling terminal

這是整個 `pty` setup 的核心步驟之一。

### 為什麼標準輸入 / 輸出 / 錯誤都要接到 slave

因為大多數互動式程式會預期：

- `stdin`
- `stdout`
- `stderr`

都指向 terminal。

所以 `pty_fork` 會把 slave `dup2` 到：

- `STDIN_FILENO`
- `STDOUT_FILENO`
- `STDERR_FILENO`

### 範例：用 `pty_fork` 跑一個 shell 的骨架

```c
#include <termios.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <stdio.h>

pid_t pty_fork(int *ptrfdm, char *slave_name, int slave_namesz,
               const struct termios *slave_termios,
               const struct winsize *slave_winsize);

int main(void) {
    int masterfd;
    pid_t pid;

    pid = pty_fork(&masterfd, NULL, 0, NULL, NULL);
    if (pid < 0) {
        perror("pty_fork");
        return 1;
    }

    if (pid == 0) {
        execlp("sh", "sh", (char *)0);
        _exit(127);
    }

    /* parent can now proxy data via masterfd */
    printf("child pid = %ld, masterfd = %d\n", (long)pid, masterfd);
    close(masterfd);
    return 0;
}
```

這段程式的教育價值在於：

- 它把「讓 child 以為自己在 terminal 上」這件事抽成一個乾淨介面

## 19.5 `pty` Program

APUE 接著用 `pty_fork` 寫出一個小工具：

- `pty`

它的目標很直白：

- 讓你把 `prog arg...`
- 換成 `pty prog arg...`

然後這個程式就會在自己的 `pty` session 裡跑。

### `pty` 的基本工作流程

1. 解析命令列選項
2. 如果目前是互動式 terminal，就抓目前的 `termios` 與 `winsize`
3. 呼叫 `pty_fork`
4. child `execvp` 目標程式
5. parent 進入 relay loop

### 幾個重要選項

- `-e`：關掉 slave line discipline 的 echo
- `-i`：忽略 stdin EOF
- `-n`：視為 noninteractive
- `-v`：印更多資訊
- `-d driver`：接一個 driver program

### 為什麼 parent 自己的 tty 要進 raw mode

如果目前是互動式使用情境，`pty` parent 會把：

- 使用者真正的 terminal

設成 raw mode。  
這樣使用者打進來的 byte 才能比較直接地送進 `pty master`，避免：

- 真的 terminal 自己又先做一次 canonical / echo / signal 處理

這個設計很關鍵，因為你其實是在做：

- terminal 到 terminal 的 relay

### relay loop 的本質

APUE 的 `loop` 做兩件事：

- stdin -> pty master
- pty master -> stdout

它甚至用兩個 process 分開搬：

- child 搬使用者輸入到 master
- parent 搬 master 輸出到 stdout

並用 `SIGTERM` 協調彼此何時結束。

這其實就是一個非常精簡的：

- terminal proxy

## 19.6 Using the `pty` Program

這節非常重要，因為它不是再教新 API，而是在幫你建立：

- `pty` 到底解了哪些真實問題

### 1. 讓程式跑在新的 pseudo terminal 上

例如：

```sh
pty ttyname
```

如果 `ttyname` 是 Chapter 18 的測試程式，  
它會看到自己的 `stdin/stdout/stderr` 都是：

- 某個新的 `pty slave`

這直接證明：

- 被 `pty` 包起來的程式真的以為自己在 terminal 上

### 2. `utmp` 不一定會記這些 pseudo terminal

APUE 提醒：

- 遠端登入 server 的 `pty` 通常應該記錄在 `utmp`
- 但一般視窗系統、`script`、自訂 `pty` 程式，不一定會記

所以：

- `who`
- `w`

看到的東西，不一定能完整反映所有 `pty` 使用情況。

### 3. job control interaction 可能很微妙

如果你在 `pty` 下跑一個真正的 job-control shell，通常還行。  
但如果你直接：

```sh
pty cat
```

然後按：

- `Ctrl-Z`

事情就不一定如你直覺。

原因在於：

- signal 是在哪一層 line discipline 被產生
- 哪個 process group 是 orphaned
- 目前 foreground / background 關係如何

這裡其實把 Chapter 9 的：

- session
- process group
- controlling terminal
- orphaned process group

全部串起來了。

### 4. 監看 long-running program 的即時輸出

如果某個程式平常把 stdout fully buffer 到檔案，  
你直接：

```sh
prog > file
```

常常不容易即時看到進度。

但如果改成：

```sh
pty -i prog < /dev/null > file &
```

程式會以為自己的 stdout 是 terminal，往往變成：

- line buffered

於是輸出更即時。

### 5. 用 shell script 做出簡化版 `script`

APUE 直接示範：

```sh
pty "${SHELL:-/bin/sh}" | tee typescript
```

這個點子非常漂亮。  
因為它表示：

- `pty` 已經能夠幫你把 terminal session 整個轉接出來

加上一個 `tee`，就能把 transcript 存檔。

### 6. 用 `pty` 解決 coprocess stdio buffering deadlock

這是本章最實用的案例之一。

前面章節提過：

- 某些 coprocess 若透過 pipe 溝通，因為 stdio fully buffering 會卡住

如果把它改成跑在 `pty` 上，對方會以為自己連的是 terminal，  
stdio 往往就改成：

- line buffered

所以整個互動流就順了。

### 為什麼 `-e` 選項很重要

當你把 coprocess 接到 `pty` 時，如果 slave 端 echo 沒關掉，就可能出現：

- 你送過去的東西被 line discipline 又 echo 一份

結果就是：

- 輸出看起來重複

所以 `-e` 會去關：

- `ECHO`
- 相關 echo flag
- 甚至 `ONLCR`

### 7. 為什麼 `pty` 不能直接取代 `expect`

這是非常重要的一點。

`pty` 只是：

- 把雙方資料搬來搬去

它不會：

- 讀懂 prompt
- 等待特定輸出再回應
- 根據狀態決定下一步

所以如果你想自動登入一個需要互動提問的程式，例如：

- `telnet`
- `passwd`

單純把一堆輸入先塞進去，往往沒用。  
因為真正需要的是：

- 看見 prompt 再回應

這就是 `expect` 類工具在做的事。

### 8. `-d driver`：讓 `pty` 外接一個 driver process

APUE 提供了一個折衷方案：

- `pty` 可以接一個 driver process

這個 driver 的 stdin/stdout 會接到 `pty`，  
但 driver 本身仍然可以透過：

- `/dev/tty`

和真正使用者互動。

這讓 `pty` 不至於升級成完整的 `expect`，  
但已經足以應付一些更聰明的互動控制場景。

## 19.7 Advanced Features

這節提到幾個更進階的 `pty` 能力。

### Packet Mode

packet mode 的目的是：

- 讓 master 端知道 slave 端發生了哪些狀態改變

例如：

- read queue 被 flush
- write queue 被 flush
- output 被 stop / restart
- XON/XOFF flow control 狀態改變

這種能力對：

- `rlogin`
- 遠端 terminal relay

很有價值。

在 BSD / Linux / Mac 類常見會用：

- `TIOCPKT`

### Remote Mode

remote mode 是：

- 告訴 slave 上的 line discipline 不要再做平常那些輸入處理

它適合這種情境：

- 你的應用程式自己就想負責 line editing

APUE 提到常見相關 command 是：

- `TIOCREMOTE`

### Window Size Changes

master 端可以用：

- `TIOCSWINSZ`

去設定 slave 的 window size。  
如果大小真的改了，kernel 就會送：

- `SIGWINCH`

給 slave 端 foreground process group。

這也是 terminal emulator 視窗 resize 時，shell 裡的全螢幕程式能知道要重畫的原因。

### Signal Generation

master 端有些系統還支援主動對 slave 端 process group 送 signal，  
常見 `ioctl` 包括：

- `TIOCSIG`
- 或 Solaris 的 `TIOCSIGNAL`

這代表 `pty` 不只是在搬資料，還能代理某些 terminal-control 級事件。

## 19.8 Summary

這章最重要的價值，是把你對 terminal 的理解從：

- 「使用者直接對 terminal 打字」

推進到：

- 「程式也可以替另一個程式模擬 terminal 環境」

也因此你會看到：

- `pty` 是 network login 的基礎之一
- `pty` 是 terminal emulator 的基礎之一
- `pty` 可以拿來做 transcript、automation、coprocess buffering 修正

核心技術線索則是：

- master / slave 架構
- `posix_openpt` / `grantpt` / `unlockpt` / `ptsname`
- `setsid`
- controlling terminal
- `pty_fork`

### 這章讀完你應該真的會的事

- 知道 `pty` 和 pipe 的根本差別
- 知道 master / slave 各自扮演什麼角色
- 知道為什麼 child 要先 `setsid` 再開 slave
- 能說明 `pty_fork` 這個 helper 背後做了哪些事
- 知道 `pty` 為什麼能讓互動式程式「看起來像在真 terminal 上」
- 知道 `pty` 在 `script`、terminal emulator、network login、coprocess 中的用途

### 這章最容易踩坑的地方

- 把 `pty` 當成只是 fancy pipe
- 忘記 controlling terminal 與 session 的角色
- 不理解為什麼程式在 pipe 上和在 `pty` 上的 buffering 行為不同
- 以為 `pty` 就能自動處理互動流程，結果拿去做 `expect` 才該做的事
- 忘記 resize、signal、echo、job control 其實都還在發生

### 建議你現在立刻動手做

1. 自己寫一個最小 `posix_openpt -> grantpt -> unlockpt -> ptsname` 範例，先拿到 master/slave 名稱。
2. 再寫一個小程式用 `pty_fork` 啟動 `/bin/sh`，親手觀察 parent 持有 master、child 持有 slave 的關係。
3. 最後把某個用 stdio 的互動式小程式放到 `pty` 下跑，觀察它和 pipe 模式下 buffering 行為的差異。

### 一句總結

這章是在教你：`pty` 的本質不是單純轉送 byte，而是替另一個 process 建出一個「看起來像真 terminal」的執行環境。
