# Chapter 2 - UNIX Standardization and Implementations

## 這章在做什麼

這一章是在回答一個非常現實的問題：

- 為什麼同樣叫 UNIX / UNIX-like，不同系統之間還能有一定程度的可攜性？
- 又為什麼同一份程式有時候在 A 系統能編、在 B 系統卻不行？

答案就是：

- 有一套套標準在規範共同介面
- 但真實世界還有各家實作差異

所以這章不是單純講歷史，而是在幫你建立「怎麼寫可攜 UNIX 程式」的基本判斷力。

## 本章小節地圖

- `2.1 Introduction`
- `2.2 UNIX Standardization`
- `2.3 UNIX System Implementations`
- `2.4 Relationship of Standards and Implementations`
- `2.5 Limits`
- `2.6 Options`
- `2.7 Feature Test Macros`
- `2.8 Primitive System Data Types`
- `2.9 Differences Between Standards`
- `2.10 Summary`

## 先抓住這章的主線

- 標準是在定義「至少要長什麼樣」。
- 實作是在決定「實際上怎麼做、還額外多做哪些事」。
- 寫可攜程式時，不能只靠印象或作業系統名字，要學會用正確的方法查 limit、查 option、查 feature。

## 2.1 Introduction

作者一開始先講背景：UNIX 早期雖然本來就有一定可攜性，但 1980 年代各家分支越來越多，差異也越來越大，大型使用者和政府機構開始強烈要求標準化。

這章的重點因此有兩個：

- 看懂有哪些主流標準
- 知道程式該怎麼查 implementation-defined limits 與 options

也就是說，這章不是只講名詞，而是在幫你避免把程式寫成「只在我的機器上能動」。

## 2.2 UNIX Standardization

### 2.2.1 ISO C

`ISO C` 的重點不是 UNIX 專屬，而是讓 `C` 程式可以跨很多作業系統。

它規範兩件大事：

- C 語言本身的語法與語意
- 標準函式庫

這也是為什麼所有 UNIX 系統都會提供像下面這些大家很熟的 library 介面：

- `stdio`
- `stdlib`
- `string`
- `time`

書裡提到：

- 1989/1990 年是 ANSI C / ISO C 的重要里程碑
- 1999 年又更新成 C99

C99 對 APUE 特別有感的點之一是 `restrict` 關鍵字，但整體來說，這章真正要你理解的是：

- 你寫的 C 程式有一層來自 ISO C 的共同地板

### 2.2.2 IEEE POSIX

`POSIX` 是這章最重要的標準家族之一。

POSIX 的核心目標是：

- 讓 application 在不同 UNIX environment 之間更容易移植

它特別關心的是 operating system interface，也就是：

- process
- file I/O
- signals
- threads
- shell and utilities

書裡還特別提醒一個容易忽略的點：

- POSIX 規範的是 interface，不是 implementation

所以在 POSIX 語境裡，常常不特別區分 system call 跟 library function，統稱為 `function`。

### POSIX 為什麼重要

因為 APUE 後面大部分介面討論，都是以「這是不是 POSIX.1 定義的」作為基礎判斷。

換句話說，你查某個 API 時，應該常常問自己：

- 這是 ISO C 的嗎？
- 這是 POSIX.1 的嗎？
- 還是某家系統特有延伸？

### 2.2.3 The Single UNIX Specification

`Single UNIX Specification`，簡稱 `SUS`，可以把它看成更完整的一套 UNIX 規格文件。

它和 POSIX 的關係非常密切。書裡也指出後來的 POSIX.1 與 Base Specifications 基本上已經高度整合。

理解上，你先抓住這幾點就夠：

- POSIX 是 portability 的核心基礎
- SUS 則更接近完整 UNIX 規格與品牌認證脈絡
- `XSI`（X/Open System Interfaces）是其中的重要一塊

### 2.2.4 FIPS

書裡提到 `FIPS` 主要是歷史脈絡，和美國政府採購標準有關。現在讀這段，重點不是背標號，而是知道：

- 標準化在某個時代不只是技術問題，也和政府與大型採購需求密切相關

## 2.3 UNIX System Implementations

作者接著談幾個實作系統或血統來源：

- `UNIX System V Release 4`
- `4.4BSD`
- `FreeBSD`
- `Linux`
- `Mac OS X`
- `Solaris`

## 讀這節時要有的正確心態

書中的版本比較是歷史快照。因為 APUE 3rd Edition 的比較平台是：

- FreeBSD 8.0
- Linux 3.2.0
- Mac OS X 10.6.8
- Solaris 10

你今天不需要把這些版本當成最新現況，但要理解作者用這些平台在示範一件事：

- 就算都很像 UNIX，細節上仍然會不同

這也是為什麼後面那麼多表格都在比較各系統是否支援某個 header、某個 limit、某個 option。

## 2.4 Relationship of Standards and Implementations

這一節是整章最關鍵的觀念句之一：

- 標準定義的是實作的一個子集合

也就是說：

- 真實系統通常比標準多
- 標準不會把每個實作細節都規定死

作者的策略很清楚：

- 盡量以 POSIX.1 為共同基礎
- 遇到平台差異就明確指出
- 避免把過時、非標準的傳統介面當成主要寫法

例如書裡提到：

- Solaris 同時保留 `O_NONBLOCK` 與舊的 `O_NDELAY`
- 但書中主線只用 POSIX 的 `O_NONBLOCK`

這個做法很值得學，因為它代表你寫教科書級、可攜程式時，應優先站在標準側，而不是某個平台的歷史包袱側。

## 2.5 Limits

### 為什麼作者要花一整大節講 limit

因為很多舊程式會把各種 magic number 寫死，例如：

- 最長 pathname 假設 1024
- 最多 open files 假設 20
- filename 長度假設 14 或 255

這些寫法一旦碰到不同 file system、不同 kernel、不同 runtime limit，就很容易出事。

### 這章把 limit 分成三類

1. compile-time limits
2. runtime limits，且不和特定檔案或目錄綁定
3. runtime limits，且和特定檔案或目錄綁定

這個分類非常重要，因為它直接對應到你該去哪裡查。

### 2.5.1 ISO C Limits

這類通常放在：

- `<limits.h>`
- `<float.h>`

例如：

- `CHAR_BIT`
- `INT_MAX`
- `LONG_MAX`

這些值在同一個編譯環境下通常固定。

書裡也提醒一個常被忽略的點：

- `char` 到底是 signed 還是 unsigned，實作可能不同

所以你真的在意時，不要靠感覺，要看平台定義。

### 2.5.2 POSIX Limits

POSIX 這裡列了很多常數，例如：

- `_POSIX_OPEN_MAX`
- `_POSIX_NAME_MAX`
- `_POSIX_PATH_MAX`
- `_POSIX_PIPE_BUF`

最容易誤解的是：這些名字裡雖然有 `MAX`，但它們常常是「標準要求的最小下限」。

換句話說：

- `_POSIX_OPEN_MAX = 20` 的意思不是「所有系統最多只能開 20 個 fd」
- 而是「符合 POSIX 的系統至少得保證到這個程度」

這是本章最值得記住的概念之一。

### 2.5.3 XSI Limits

有些 POSIX minimum 實在太小，不夠現代系統使用，所以 `XSI` 又補了一些較高的最小值，例如：

- `_XOPEN_NAME_MAX`
- `_XOPEN_PATH_MAX`
- `_XOPEN_IOV_MAX`

這表示標準世界本身也承認：

- 基礎 POSIX 下限有時只是「保底」，不是實務舒適值

### 2.5.4 `sysconf`, `pathconf`, `fpathconf`

這三個函式是本章實作面最重要的內容。

它們的角色分工是：

- `sysconf`：查不綁定特定路徑的 runtime limit
- `pathconf`：查跟某個 pathname 有關的 limit
- `fpathconf`：查跟某個已開啟 file descriptor 有關的 limit

### 什麼時候該用誰

- 想查一個 process 最多可開多少 fd：`sysconf(_SC_OPEN_MAX)`
- 想查某目錄底下 filename 最長多長：`pathconf(path, _PC_NAME_MAX)`
- 想查某已開目錄的 path limit：`fpathconf(fd, _PC_PATH_MAX)`

### 這三個函式最難的地方：`-1` 不一定表示 error

這是書裡刻意反覆講的地方。

它們回傳 `-1` 有兩種可能：

- 真的出錯，而且 `errno` 被設值
- limit 是 indeterminate / no limit，此時 `errno` 不變

所以正確模式通常是：

```c
errno = 0;
long val = sysconf(_SC_OPEN_MAX);
if (val < 0) {
    if (errno != 0) {
        perror("sysconf");
    } else {
        puts("no limit / indeterminate");
    }
}
```

這個 pattern 非常值得記熟。

### 2.5.5 Indeterminate Runtime Limits

這一節是在講一個很實際的尷尬場景：

- 某個 limit 在 compile-time 沒定義
- runtime 查它時，它又可能是 indeterminate

作者用兩個例子說明這種麻煩。

### 例子一：pathname buffer 到底該配多大

很多舊程式會直接寫：

- `char path[1024];`

但這不一定真的正確。

書裡的策略是：

1. 先看 `PATH_MAX` 有沒有定義
2. 沒有就用 `pathconf("/", _PC_PATH_MAX)` 查
3. 如果還是 indeterminate，只能猜一個保守值
4. 某些 API 若回報 `ERANGE`，就 `realloc` 後重試

這個思維比「亂猜一個 4096」成熟得多。

### 例子二：最多 open files 到底是多少

daemon 常要關掉所有繼承來的 fd。很多老程式會寫：

```c
for (i = 0; i < 20; i++)
    close(i);
```

這種寫法非常不可攜。

書裡的做法是：

1. 優先查 `OPEN_MAX`
2. 沒定義就用 `sysconf(_SC_OPEN_MAX)`
3. 若 indeterminate，再退而求其次猜一個 reasonable value

作者也提醒：

- 某些實作會回 `LONG_MAX`
- 如果你拿它直接當 loop 上限，程式可能白白關幾十億次 fd

這是 textbook 級的避坑提醒。

### 簡單範例：查幾個常用 limit

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    long openmax = sysconf(_SC_OPEN_MAX);
    long namemax = pathconf(".", _PC_NAME_MAX);

    printf("OPEN_MAX = %ld\n", openmax);
    printf("NAME_MAX = %ld\n", namemax);
    return 0;
}
```

## 2.6 Options

### limit 跟 option 不一樣

`limit` 問的是：

- 一個值多大

`option` 問的是：

- 某個功能有沒有支援

這兩種問題不能混在一起。

### option 的三種查法

和 limit 一樣，option 也有三種來源：

- compile-time macro
- `sysconf`
- `pathconf` / `fpathconf`

### compile-time 常數的三種含義

書裡把判讀方式整理得很清楚：

1. 未定義或定義為 `-1`：編譯時可視為不支援
2. 大於 0：支援
3. 等於 0：要再用 `sysconf` / `pathconf` 查 runtime 狀態

### 常見重要 option

系統層面你常會看到：

- `_POSIX_THREADS`
- `_POSIX_MAPPED_FILES`
- `_POSIX_SEMAPHORES`
- `_POSIX_TIMERS`
- `_POSIX_THREAD_SAFE_FUNCTIONS`

路徑層面則有：

- `_PC_NO_TRUNC`
- `_PC_CHOWN_RESTRICTED`
- `_PC_2_SYMLINKS`

### `_SC_VERSION` 與 `_SC_XOPEN_VERSION`

這兩個特別實用：

- `_SC_VERSION`：POSIX 版本
- `_SC_XOPEN_VERSION`：XSI / SUS 版本

例如書裡指出：

- POSIX.1-2008 對應 `200809L`
- SUSv4 / XSI Version 4 對應 `700`

這讓你在 runtime 可以知道自己站在哪個標準等級上。

## 2.7 Feature Test Macros

### 這節超實用，因為它直接影響你能不能看到某些宣告

很多人第一次遇到這些 macro 會覺得很煩，但它其實是在告訴標頭檔：

- 我要哪一套標準視圖

### `_POSIX_C_SOURCE`

如果你想只打開 POSIX 那部分定義，常見寫法是：

```c
#define _POSIX_C_SOURCE 200809L
```

或在編譯時加：

```sh
cc -D_POSIX_C_SOURCE=200809L file.c
```

### `_XOPEN_SOURCE`

如果你要 XSI / SUS 那邊的功能，常見寫法是：

```c
#define _XOPEN_SOURCE 700
```

或：

```sh
gcc -D_XOPEN_SOURCE=700 -std=c99 file.c -o file
```

### 什麼時候會需要它

當你遇到這些狀況時，通常就要想到 feature test macro：

- 某個函式明明系統有，但編譯器說沒宣告
- 某些常數沒定義
- 某個結構欄位看不到

很多時候不是系統沒有，而是你沒有把對應標準視圖打開。

## 2.8 Primitive System Data Types

這節的重點是 portability。

書裡列出很多常見 `_t` 型別，例如：

- `pid_t`
- `uid_t`
- `gid_t`
- `mode_t`
- `off_t`
- `size_t`
- `ssize_t`
- `time_t`
- `clock_t`
- `pthread_t`

### 為什麼一定要用這些型別

因為它們把實作細節藏起來了。

你不應該假設：

- `pid_t` 一定是 `int`
- `off_t` 一定是 32-bit
- `size_t` 跟 `unsigned int` 一定一樣大

用正確型別，才能讓程式跟著平台一起移動。

## 2.9 Differences Between Standards

這節在提醒你：標準之間不一定完美一致。

### `clock_t` 的單位可能不同

ISO C 的 `clock()` 和 POSIX 的 `times()` 都會用到 `clock_t`，但單位可能不同。

所以你不能看到 `clock_t` 就直接拿來亂比，還要知道：

- 它是哪個 API 回來的
- 該用什麼常數或函式換算

### `signal` 也是經典例子

ISO C 有 `signal()`，但 POSIX 對 signal 機制的要求更完整、更嚴格。這也是為什麼實務上大家常更偏好：

- `sigaction`

而不是把 `signal()` 當成萬用做法。

## 2.10 Summary

### 這章讀完你應該真的會的事

- 知道 `ISO C`、`POSIX.1`、`SUS/XSI` 彼此大概是什麼關係
- 知道標準定義的是共同地板，不是真實系統的全部
- 知道 limit 分 compile-time 和 runtime 兩大類
- 會用 `sysconf`、`pathconf`、`fpathconf` 查系統值
- 理解 `-1` 可能是錯，也可能是 indeterminate
- 知道 `_POSIX_C_SOURCE` 和 `_XOPEN_SOURCE` 是幹嘛的

### 這章真正的實戰價值

它讓你從「我在某台機器上寫程式」提升到：

- 「我知道哪些東西是標準保證的」
- 「我知道哪些東西要查」
- 「我知道哪些東西不能寫死」

這就是寫 UNIX 系統程式時，可攜性思維的起點。

### 建議你現在立刻做的事

- 自己跑一次 `sysconf(_SC_OPEN_MAX)`。
- 自己跑一次 `pathconf(".", _PC_NAME_MAX)`。
- 試著在一個小程式最上面加 `_POSIX_C_SOURCE`，觀察 header 暴露的差異。

### 一句總結

這章看似抽象，其實是在教你不要亂猜環境。真正成熟的 UNIX 程式，不是「以為世界都一樣」，而是知道該向系統查什麼、該相信哪些保證、該避開哪些假設。
