# Chapter 1 - UNIX System Overview

## 這章在做什麼

這一章不是要你一次學會所有 UNIX API，而是先把整本 APUE 的共同語言建立起來。作者故意用「快速導覽」的方式，把後面會反覆出現的核心概念先介紹一輪：`kernel`、`system call`、`shell`、`file descriptor`、`process`、`errno`、`signal`、`time`、`user ID`。

如果你把這章讀懂，後面每一章就比較不會像在背零散函式，而會知道每個主題放在整個 UNIX 世界裡是什麼位置。

## 本章小節地圖

- `1.1 Introduction`
- `1.2 UNIX Architecture`
- `1.3 Logging In`
- `1.4 Files and Directories`
- `1.5 Input and Output`
- `1.6 Programs and Processes`
- `1.7 Error Handling`
- `1.8 User Identification`
- `1.9 Signals`
- `1.10 Time Values`
- `1.11 System Calls and Library Functions`
- `1.12 Summary`

## 先抓住這章的三個大重點

- UNIX 程式設計最核心的兩個抽象是 `file` 和 `process`。
- 使用者程式不是直接碰硬體，而是透過 `system call` 向 `kernel` 要服務。
- 很多看似零碎的 API，其實都圍繞在「資源怎麼被 process 使用、共享、保護、回收」這件事。

## 1.1 Introduction

作者一開始就先講清楚：所有作業系統都會幫程式提供一些基本服務，例如：

- 執行新程式
- 開檔、讀檔、寫檔
- 配置記憶體
- 取得目前時間

APUE 的主題就是「UNIX 提供給程式設計者的這些服務」。

這一章的定位不是深入，而是先帶你認識後面會一直出現的關鍵詞。你可以把它想成整本書的暖身地圖。

## 1.2 UNIX Architecture

### 狹義的 operating system：`kernel`

書裡先用比較嚴格的定義說明：真正控制硬體資源、讓程式能執行的那層核心軟體叫 `kernel`。

`kernel` 主要負責：

- CPU 調度
- 記憶體管理
- 檔案系統
- 裝置管理
- process 管理

### `system call` 是進入 kernel 的入口

UNIX 不是讓應用程式直接任意操作硬體，而是提供一組受控的入口，這些入口就是 `system call`。

你可以把它想成：

- `kernel` 是資源管理機關
- `system call` 是正式申請窗口

像這些事情通常都要透過 `system call`：

- `open`
- `read`
- `write`
- `fork`
- `exec`

### library routine 在 `system call` 上面

很多常用功能不是直接 system call，而是 library 幫你包一層。

例如：

- `printf` 是 library function
- 它底下可能再呼叫 `write`

這一點在後面的 `stdio` 與 unbuffered I/O 比較時會非常重要。

### shell 是一種特殊 application

書中特別提醒：`shell` 不是 kernel，而是跑在 user space 的一個程式。

它的工作是：

- 讀使用者輸入
- 解析命令列
- 啟動其他程式
- 處理 redirection 與 pipeline

也就是說，平常你在終端機裡輸入命令，實際上先對話的是 shell，不是 kernel 本身。

### 狹義與廣義的作業系統

書裡還刻意區分兩種用法：

- 狹義：`operating system = kernel`
- 廣義：`operating system = kernel + shell + utilities + libraries + applications`

這也是為什麼很多人會直接把 Linux 稱為 operating system。嚴格說 Linux 是 kernel，但口語上大家常把整套環境一起叫 Linux。

## 1.3 Logging In

### login 時系統做了什麼

你登入 UNIX 系統時，輸入的是：

- login name
- password

系統會根據帳號資料決定：

- 你是誰
- 你的 `UID` / `GID`
- 你的 home directory
- 你的預設 shell 是哪個

書裡用 `/etc/passwd` 的歷史格式舉例，一筆資料傳統上有 7 個欄位：

- login name
- encrypted password 或 placeholder
- numeric user ID
- numeric group ID
- comment 欄位
- home directory
- shell program

今天大多數系統已經把真正的加密密碼搬到別處，但這個資料模型仍然很重要。

### shell 的角色

登入後通常會進到 shell。shell 是 command-line interpreter，來源可能是：

- terminal 的互動輸入
- script file

這兩種分別對應：

- interactive shell
- shell script

### 常見 shell

書裡整理了幾個重要 shell：

- Bourne shell
- Bourne-again shell (`bash`)
- C shell
- Korn shell
- TENEX C shell (`tcsh`)

你不用一開始就死背哪個 shell 發源於哪個系統，但要知道：

- 不同 shell 語法與功能略有差異
- APUE 範例通常盡量使用 Bourne / Korn / `bash` 都通用的用法

### 為什麼這裡要講 shell

因為你後面會一直看到像這樣的互動式範例：

```sh
$ ./a.out
$ ls > file.list
$ ./a.out < infile > outfile
```

如果你不知道 shell 幫你做了什麼，就很難理解標準輸入輸出、redirection、process control 為什麼會長這樣。

## 1.4 Files and Directories

### UNIX file system 是階層式的

所有東西從 root directory `/` 開始。UNIX 的檔案系統本質上是一棵樹狀結構。

### directory 在邏輯上是什麼

書裡先給一個邏輯模型：directory 是一個包含 directory entries 的檔案。

每個 directory entry 可以想成至少關聯到：

- filename
- 檔案屬性資訊

屬性包括：

- file type
- file size
- owner
- permissions
- last modification time

之後 Chapter 4 會再把這些 metadata 講得更完整。

### filename 的基本規則

UNIX 對 filename 的真正硬限制其實不多。書裡指出不能出現的兩個字元是：

- slash `/`
- null character

原因很直觀：

- `/` 用來分隔 pathname component
- `\0` 用來結束 C 字串

雖然理論上很多字元都能放進檔名，但實務上最好還是控制在安全字元集合內，避免 shell quoting 變複雜。

### dot 與 dot-dot

每個新目錄都會有：

- `.`
- `..`

它們分別表示：

- current directory
- parent directory

在 root directory 裡，`..` 會跟 `.` 一樣。

### pathname

一串用 `/` 分隔的 filename component 形成 `pathname`。

兩種重要分類：

- absolute pathname：以 `/` 開頭
- relative pathname：不以 `/` 開頭，會相對於 current working directory 解讀

### 這章的第一個實作範例：簡化版 `ls`

書裡用了 `opendir`、`readdir`、`closedir` 做一個最小版列目錄程式，目的不是要你背 API，而是要你開始接受這件事：

- directory 也是可以被程式打開並逐項讀取的東西

簡化後的核心寫法像這樣：

```c
#include <dirent.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    DIR *dp;
    struct dirent *dirp;

    if (argc != 2) {
        fprintf(stderr, "usage: myls dirname\n");
        return 1;
    }

    dp = opendir(argv[1]);
    if (dp == NULL) {
        perror("opendir");
        return 1;
    }

    while ((dirp = readdir(dp)) != NULL) {
        puts(dirp->d_name);
    }

    closedir(dp);
    return 0;
}
```

### manual page 記法

書裡順便介紹像 `ls(1)` 這種寫法。

意思是：

- `ls` 這個條目在 manual 的 section 1

常見 section 觀念：

- section 1: user commands
- section 2: system calls
- section 3: library functions

這個習慣很重要，因為後面查文件會一直用到。

## 1.5 Input and Output

### file descriptor

書裡先重新強調：對 kernel 來說，所有 open files 都用 `file descriptor` 表示。

`file descriptor` 是：

- 小的非負整數
- process 用來告訴 kernel「我要操作哪個 open file」的識別碼

### standard input / output / error

按照 shell 的慣例，每個新程式一啟動通常就有三個 descriptor：

- `0`: standard input
- `1`: standard output
- `2`: standard error

這是 shell 和應用程式共同遵守的約定，不是 kernel 自己神奇硬塞的宇宙真理。

### redirection

像下面這行：

```sh
ls > file.list
```

代表 shell 在啟動 `ls` 前，先把它的標準輸出改接到 `file.list`。

這是理解 shell I/O 模型的第一步。

### unbuffered I/O

作者這裡先介紹最底層的一組：

- `open`
- `read`
- `write`
- `lseek`
- `close`

這些常被稱為 unbuffered I/O，意思是每次 `read` / `write` 都會直接進 kernel 做 system call。

### 範例：從標準輸入拷貝到標準輸出

書裡用一個超經典的小程式展示 UNIX I/O 的最小骨架：

```c
#include <unistd.h>

#define BUFFSIZE 4096

int main(void) {
    ssize_t n;
    char buf[BUFFSIZE];

    while ((n = read(STDIN_FILENO, buf, BUFFSIZE)) > 0) {
        if (write(STDOUT_FILENO, buf, n) != n) {
            return 1;
        }
    }

    return (n < 0) ? 1 : 0;
}
```

這段程式同時示範了：

- `STDIN_FILENO` 與 `STDOUT_FILENO`
- `read` 成功時回傳實際讀到的 byte 數
- EOF 時 `read` 回傳 `0`
- error 時 `read` 回傳 `-1`

### standard I/O

接著作者對比 `stdio`：

- `getc`
- `putc`
- `printf`
- `fgets`

這一層是 buffered interface，底下仍然會用到低階 I/O，但它幫你做了：

- buffer 管理
- 逐行輸入
- 格式化輸出

所以書裡很早就先讓你看到：

- `read/write` 比較底層
- `stdio` 比較方便

這個差別到了 Chapter 5 會正式展開。

## 1.6 Programs and Processes

### program 與 process 不一樣

書裡先把概念拆開：

- `program` 是磁碟上的 executable file
- `process` 是 program 被 kernel 載入並執行後的實例

這是後面所有 process control 的起點。

### process ID

每個 process 都有唯一的 numeric identifier，也就是 `PID`。

作者特別示範 `getpid()`，並提醒：

- `pid_t` 真正大小不一定相同
- 輸出時通常 cast 成 `long` 來配合 `printf`

### process control 的三大主角

這一章先預告後面最重要的三個：

- `fork`
- `exec`
- `waitpid`

### 書裡的 shell-like 範例在做什麼

作者用一個極簡 shell 來示範 UNIX process control 的經典流程：

1. `fgets` 讀一行命令
2. `fork` 出 child
3. child 用 `execlp` 執行命令
4. parent 用 `waitpid` 等 child 結束

這是整本書最重要的流程之一。

你可以把它記成：

- `fork`：生出新 process
- `exec`：把 child 換成新程式
- `waitpid`：parent 收尾

### 這個範例刻意保留的限制

作者也明講了這個小 shell 的侷限：

- 不支援命令參數
- 不支援 pipeline
- 不支援 redirection

但它的重點不是做出完整 shell，而是讓你先看懂 UNIX 產生新程式的基本套路。

### threads 只先點到

本章後面稍微提到 thread：

- 同一個 process 可以有多個 thread of control
- thread 共享 address space 和很多 process 資源
- 但各自有自己的 stack 與 thread ID

作者在這裡只是先埋伏筆，真正展開要到 Chapter 11、12。

## 1.7 Error Handling

### UNIX API 失敗時通常怎麼表示

書裡說明了常見慣例：

- 很多函式失敗回傳負值，例如 `-1`
- 有些回傳 pointer 的函式失敗時回傳 `NULL`
- 然後再透過 `errno` 告訴你失敗原因

### `errno` 的兩條基本規則

這章有兩條非常重要的規則：

1. 只有當函式的回傳值顯示「真的出錯」時，才去看 `errno`
2. 成功時函式不會主動把 `errno` 清成 0`

所以錯誤寫法是：

- 沒先檢查回傳值就直接印 `errno`

### `strerror` 與 `perror`

- `strerror(errnum)`：把錯誤碼轉成字串
- `perror(msg)`：印出 `msg: 對應錯誤訊息`

簡單示例：

```c
#include <errno.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[]) {
    fprintf(stderr, "EACCES: %s\n", strerror(EACCES));
    errno = ENOENT;
    perror(argv[0]);
    return 0;
}
```

### fatal 與 nonfatal error

作者還把錯誤分成兩類：

- `fatal error`：沒有合理補救方式，通常只能記錄後結束
- `nonfatal error`：可能只是暫時資源不足，可以延遲後重試

典型 nonfatal 狀況像：

- `EAGAIN`
- `ENOBUFS`
- `ENOSPC`
- 某些情況下的 `EINTR`

實務上常見的恢復策略是：

- sleep 一下
- retry
- 或做 exponential backoff

## 1.8 User Identification

### `UID` 與 `GID`

系統用 numeric `UID` 和 `GID` 表示使用者與群組身分。這些值在帳號建立時由系統管理員分配。

### 為什麼用數字，不直接用名字

書裡解釋得很實際：

- 檔案系統儲存數字比存字串省空間
- 權限判斷時比對整數比比對字串快

所以：

- 人類用 login name / group name
- kernel 與檔案系統主要用 numeric ID

### superuser

`UID = 0` 的使用者就是 `root`，也叫 superuser。

它的特權包括：

- 大多數 permission check 可被繞過
- 某些一般使用者無法做的系統操作可以做

### supplementary groups

除了 primary group 之外，一個使用者還可以屬於多個 supplementary groups。這在檔案共享與團隊協作上很重要。

## 1.9 Signals

### signal 是什麼

`signal` 是通知 process 某個事件發生的一種機制。

例如：

- 除以 0 可能收到 `SIGFPE`
- 按 `Ctrl-C` 可能收到 `SIGINT`
- process 之間也可以用 `kill` 送 signal

### process 面對 signal 的三種選擇

書裡用很標準的方式整理成三種：

- ignore
- 使用 default action
- catch 它，自己提供 handler

### 範例：shell 攔截 `SIGINT`

作者把前面的簡化 shell 多加幾行：

- 用 `signal(SIGINT, sig_int)` 註冊 handler
- 按 `Ctrl-C` 時不要直接結束，而是印出 `interrupt` 再回到 prompt

這個例子在觀念上很重要，因為它讓你看到：

- shell 自己可以決定怎麼處理 signal
- signal 不是「只能殺 process」而已

## 1.10 Time Values

### 兩種時間

書裡先分兩大類：

- `calendar time`
- `process time`

### calendar time

它以 `time_t` 表示，概念上是從 Epoch 開始累積的秒數。

Epoch 指的是：

- `00:00:00 January 1, 1970, UTC`

這種時間常用來表示：

- 檔案最後修改時間
- 現在時間

### process time

這是 CPU 角度的時間，通常用 `clock_t` 表示，歷史上以 clock ticks 計算。

書裡又把它拆成三個你以後一定要會分的值：

- `clock time` 或 `wall clock time`
- `user CPU time`
- `system CPU time`

差別是：

- `wall clock time`：真實經過多久
- `user CPU time`：程式在 user mode 執行多久
- `system CPU time`：kernel 代你做事花多久

### `time` 指令

作者示範可以直接用 shell 的 `time` 來量命令執行時間，這對理解效能測量很重要。

## 1.11 System Calls and Library Functions

### user 眼中看起來都像 C function

這章最後回到一個很關鍵的概念：從寫程式的角度看，system call 和 library function 都長得像 C function。

但它們本質上不一樣。

### system call

- 是進 kernel 的正式入口
- 直接請作業系統提供服務

### library function

- 跑在 user space
- 可能會呼叫 system call
- 也可能完全不碰 kernel

例如：

- `printf` 可能會呼叫 `write`
- `strcpy`、`atoi` 就完全不需要 kernel 幫忙

### `malloc` 與 `sbrk` 的分工

書裡用一個很經典的例子說明兩層分工：

- `sbrk` 是 system call，負責向 kernel 調整 process address space
- `malloc` 是 library function，負責在 user space 管理怎麼分配那塊記憶體

重點是：

- kernel 不替你做高階配置策略
- library 幫你把底層能力包成較好用的介面

### 時間 API 也是同一個邏輯

UNIX 的 system call 先提供簡單原始時間值，至於怎麼轉成人類可讀的日期與時區表示，交給 library 處理。

這就是 UNIX 很典型的分層設計：

- kernel 提供基本機制
- library 提供常用政策與包裝

## 1.12 Summary

### 這章讀完你應該真的記住的事

- UNIX 程式設計的地基是 `file`、`process`、`ID`、`signal`、`time`。
- shell 是互動入口，但不是 kernel。
- `system call` 與 `library function` 在介面上都像 C function，角色卻不同。
- `errno` 的使用有固定規矩，不能亂讀。
- `fork/exec/waitpid` 會是後面整本書最常出現的一條主線。

### 建議你現在立刻做的事

- 把本章幾個小程式自己手打一次。
- 用 `man 1 ls`、`man 2 open`、`man 3 printf` 比較 section 差異。
- 實際在 shell 玩一次 `>`, `<`, `|`，把它跟標準輸入輸出對起來。

### 一句總結

這章真正的目的，是把你從「我只是在寫 C 程式」拉進「我在跟 UNIX 這個執行環境合作」的視角。這個視角一旦建立，後面的章節就會順很多。
