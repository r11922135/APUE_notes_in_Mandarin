# Chapter 6 - System Data Files and Information

## 這章在做什麼

這章在講一件很務實但很容易被忽略的事：

- UNIX programming 不只有 system call
- 系統本身還維護了很多描述「使用者、群組、主機、登入狀態、時間」的資料

而且真正好的設計不是直接去硬讀某個文字檔，而是透過標準 API 去取資料，讓底層實作可以換。

這章的核心心智模型是：

- system databases 有固定的資料模型
- library 提供查詢介面
- 你的程式應該依賴 API，而不是死綁某個檔案格式

## 本章小節地圖

- `6.1 Introduction`
- `6.2 Password File`
- `6.3 Shadow Passwords`
- `6.4 Group File`
- `6.5 Supplementary Group IDs`
- `6.6 Implementation Differences`
- `6.7 Other Data Files`
- `6.8 Login Accounting`
- `6.9 System Identification`
- `6.10 Time and Date Routines`
- `6.11 Summary`

## 先抓住這章最重要的心智模型

### 重點不是 `/etc/passwd` 這個檔名本身

這章雖然常拿：

- `/etc/passwd`
- `/etc/group`
- `/etc/shadow`

當例子，但 APUE 想教你的其實是：

- 這些資料有共同的 access pattern
- `get...`, `set...`, `end...` 這組 API 很常出現

### 資料格式可以變，介面最好不要變

小系統也許真的就是 ASCII text file。  
大系統可能會變成：

- hashed database
- directory service
- NIS
- LDAP

如果你的程式只知道「直接 parse 某個本機檔案」，可攜性就會很差。

## 6.1 Introduction

這章一開始就說明：

- 傳統 UNIX 的很多系統資料是 text files
- 但直接掃整個檔案在大系統上很慢
- 所以需要抽象化的 portable interfaces

這章除了帳號與群組資料，也會講：

- system identification
- time and date routines

## 6.2 Password File

### `password file` 存什麼

典型的 `struct passwd` 會包含：

- `pw_name`
- `pw_passwd`
- `pw_uid`
- `pw_gid`
- `pw_gecos`
- `pw_dir`
- `pw_shell`

有些系統還會有更多欄位。

### 一筆帳號資料常見長相

像：

```text
root:x:0:0:root:/root:/bin/bash
```

你可以把它讀成：

- login name
- password placeholder
- UID
- GID
- comment / gecos
- home directory
- login shell

### 幾個很重要的觀察

#### `root`

- 通常 `UID == 0`
- 代表 superuser

#### password 欄位常是 placeholder

現代系統通常不會把真正 encrypted password 放在 everyone-readable 的 `/etc/passwd`。  
所以常看到：

- `x`
- `*`
- 或其他 placeholder

真正內容可能放在 shadow file。

#### shell 欄位可以拿來禁用登入

像：

- `/dev/null`
- `/bin/false`
- `nologin`

常被用來讓某些 service account 無法互動登入。

#### `nobody`

常用來表示一個幾乎沒有特權的帳號。

### 常用 API

```c
#include <pwd.h>

struct passwd *getpwuid(uid_t uid);
struct passwd *getpwnam(const char *name);

struct passwd *getpwent(void);
void setpwent(void);
void endpwent(void);
```

### 兩類查詢方式

#### keyed lookup

- `getpwuid`
- `getpwnam`

#### sequential scan

- `getpwent`
- `setpwent`
- `endpwent`

### 一個重要實作細節

這些函式很多會回傳指向 static storage 的指標。  
也就是：

- 下一次再呼叫，前一次內容可能被覆蓋

如果你要保留資料，應該自己 copy。

### 範例：查目前 user 的帳號資訊

```c
#include <pwd.h>
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main(void) {
    struct passwd *pw = getpwuid(getuid());
    if (pw == NULL) {
        perror("getpwuid");
        return 1;
    }

    printf("name  = %s\n", pw->pw_name);
    printf("uid   = %u\n", pw->pw_uid);
    printf("gid   = %u\n", pw->pw_gid);
    printf("home  = %s\n", pw->pw_dir);
    printf("shell = %s\n", pw->pw_shell);
    return 0;
}
```

### `getpwent` 使用習慣

如果你要掃整個資料庫，典型流程是：

1. `setpwent()`
2. 一直 `getpwent()`
3. 用完 `endpwent()`

記得 `endpwent()` 很重要，因為 library 不知道你什麼時候真的掃完了。

## 6.3 Shadow Passwords

### 為什麼需要 shadow file

如果 encrypted password 放在 everyone-readable 的資料庫裡，攻擊者就能拿去做離線猜密碼。

即使 encryption 是 one-way：

- 不能直接反推明文

但攻擊者仍可：

1. 猜一個密碼
2. 跑同樣演算法
3. 比對輸出

使用者又常選弱密碼，所以風險很大。

### shadow file 的核心設計

- 把 encrypted password 移到權限更嚴格的地方
- 普通程式照樣能查 `/etc/passwd` 的一般帳號資訊
- 只有少數 privileged program 能看 shadow data

### `struct spwd`

常見欄位包括：

- `sp_namp`
- `sp_pwdp`
- `sp_lstchg`
- `sp_min`
- `sp_max`
- `sp_warn`
- `sp_inact`
- `sp_expire`
- `sp_flag`

### 這些欄位在幹嘛

除了 encrypted password 本身，還記錄：

- 上次改密碼時間
- 最短多久才能改
- 最長多久必須改
- 到期前幾天警告
- inactive 規則
- 帳號到期時間

也就是所謂 `password aging`。

### 常用 API

```c
#include <shadow.h>

struct spwd *getspnam(const char *name);
struct spwd *getspent(void);
void setspent(void);
void endspent(void);
```

### 這組 API 的實務含義

如果你不是有適當 privilege 的程式：

- 多半根本不應碰這組介面

一般應用程式只需要 user name、UID、home directory 之類資訊，不需要摸 encrypted password。

## 6.4 Group File

UNIX 的權限模型不只看 user，也很看重 group。

### `struct group`

常見欄位：

- `gr_name`
- `gr_passwd`
- `gr_gid`
- `gr_mem`

### `gr_mem` 是什麼

它是：

- 指向多個 user name 字串的指標陣列
- 以 `NULL` 結尾

也就是說，一個 group 可以列出有哪些使用者屬於它。

### 常用 API

```c
#include <grp.h>

struct group *getgrgid(gid_t gid);
struct group *getgrnam(const char *name);

struct group *getgrent(void);
void setgrent(void);
void endgrent(void);
```

### 與 password file API 幾乎同型

這正是 APUE 想讓你看到的 pattern：

- 查單筆：`getgrgid`, `getgrnam`
- 掃全部：`getgrent`, `setgrent`, `endgrent`

## 6.5 Supplementary Group IDs

這節很重要，因為它直接影響檔案權限判斷。

### 歷史演進

早期 UNIX 某一時刻只讓一個 user 屬於一個主要 group。  
後來 4.2BSD 引入 `supplementary group IDs`，讓一個 process 同時屬於多個 groups。

### 好處

你不必一直：

- `newgrp`
- 在不同 project group 間來回切換

現在可以同時參與多個 groups，kernel 做 permission check 時也會一起看。

### 常用 API

```c
#include <unistd.h>

int getgroups(int gidsetsize, gid_t grouplist[]);
int setgroups(int ngroups, const gid_t grouplist[]);
int initgroups(const char *username, gid_t basegid);
```

### `getgroups`

用來取得目前 process 的 supplementary groups。

一個常見技巧：

- 先 `getgroups(0, NULL)` 拿需要的長度
- 再配置陣列
- 再真的取值

### `setgroups`

可以設定 supplementary groups，但通常是 privileged operation。

### `initgroups`

它會：

- 掃 group database
- 找出某個 user 屬於哪些 groups
- 再把 `basegid` 也加進去

`login` 這類程式很常需要它。

### 範例：列出目前 process 的 supplementary groups

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    int n = getgroups(0, NULL);
    gid_t *groups;
    int i;

    if (n < 0) {
        perror("getgroups");
        return 1;
    }

    groups = malloc((size_t)n * sizeof(gid_t));
    if (groups == NULL) {
        perror("malloc");
        return 1;
    }

    if (getgroups(n, groups) < 0) {
        perror("getgroups");
        free(groups);
        return 1;
    }

    for (i = 0; i < n; i++)
        printf("%u\n", groups[i]);

    free(groups);
    return 0;
}
```

## 6.6 Implementation Differences

這節是在提醒你：

- 帳號資料「概念上」一致
- 但實作上各平台差很多

### 書裡舉的差異

#### FreeBSD

- `/etc/master.passwd`
- 還可能有 hashed databases，例如 `pwd.db`

#### Linux

- `/etc/passwd`
- `/etc/shadow`
- `/etc/group`

#### Mac OS X 書中年代的做法

- 正常 multiuser mode 多透過 `Directory Services`
- 不只是單純讀本機檔案

#### Solaris

- 類似 Linux，但某些欄位型別與語意細節略不同

### 為什麼這節重要

因為它再次說明：

- 不要把「某個平台剛好有的檔案格式」誤當成可攜標準

### 更大的現實世界：NIS / LDAP / `nsswitch`

許多系統會透過：

- `NIS`
- `NIS+`
- `LDAP`
- `/etc/nsswitch.conf`

決定 user / group / host 等資料到底從哪裡來。  
所以你越依賴標準查詢 API，越不容易被底層實作綁死。

## 6.7 Other Data Files

這節在做抽象總結。

### 通用 pattern

許多 system data files 都有類似介面：

1. `get...`：讀下一筆
2. `set...`：重設到開頭
3. `end...`：關閉

如果支援 keyed lookup，還會有：

- `get...byname`
- `get...byaddr`
- 類似風格的函式

### 典型例子

- hosts
- networks
- protocols
- services

### 這節真正要你記住的事

不是每個資料庫的函式名，而是「家族長相」：

- password / group / host / service 這些 API 長得很像

這是 UNIX library 設計的一種慣例。

## 6.8 Login Accounting

### `utmp` 和 `wtmp`

這兩個資料來源大致是：

- `utmp`：誰現在登入中
- `wtmp`：登入 / 登出歷史

### 常見用途

- `who`：看現在誰在線上
- `last`：看歷史登入紀錄

### 重點不是背結構細節

APUE 在這裡主要是讓你知道：

- 系統會維護這類 login accounting records
- 格式各平台差異很大
- 有些系統已經換成別種 logging 機制

所以這類資料也屬於：

- 應盡量透過平台提供介面存取

## 6.9 System Identification

### `uname`

```c
#include <sys/utsname.h>

int uname(struct utsname *name);
```

`struct utsname` 常見欄位：

- `sysname`
- `nodename`
- `release`
- `version`
- `machine`

### 這些欄位各代表什麼

- OS 名稱
- node / host 名稱
- release
- version
- hardware type

### 很重要的限制

`uname` 告訴你的是系統身分資訊，不是 POSIX 相容等級。  
如果你要知道規格層級，應該看：

- `_POSIX_VERSION`
- 或 `sysconf`

### `gethostname`

```c
#include <unistd.h>

int gethostname(char *name, int namelen);
```

它更聚焦在 host name 本身。

### 實務理解

- `uname`：比較像一包系統標識資料
- `gethostname`：比較像只問主機名字

## 6.10 Time and Date Routines

這節是 Chapter 6 的另一個大重點。

### UNIX 的基本時間模型

kernel 基本上是以：

- `Epoch` 以來過了多少秒

來記錄時間。

這個 `Epoch` 是：

- `1970-01-01 00:00:00 UTC`

對應型別通常是：

- `time_t`

### 為什麼是 UTC

UNIX 的基本設計是：

- 內部用 UTC 記
- 顯示給人看時再轉 local time

這樣比較一致，也比較容易處理 DST 等本地規則。

### `time`

```c
#include <time.h>

time_t time(time_t *calptr);
```

拿現在的 calendar time。

### 多種 clock：`clock_gettime`

```c
#include <sys/time.h>

int clock_gettime(clockid_t clock_id, struct timespec *tsp);
int clock_getres(clockid_t clock_id, struct timespec *tsp);
int clock_settime(clockid_t clock_id, const struct timespec *tsp);
```

### 常見 clock

- `CLOCK_REALTIME`
- `CLOCK_MONOTONIC`
- `CLOCK_PROCESS_CPUTIME_ID`
- `CLOCK_THREAD_CPUTIME_ID`

### 最重要的理解

- `CLOCK_REALTIME`：真實系統時間，可能被調整
- `CLOCK_MONOTONIC`：不應往回跳，適合量測經過時間

所以如果你是在量 timeout / latency：

- 通常別用 `CLOCK_REALTIME`

### `gettimeofday`

```c
int gettimeofday(struct timeval *restrict tp, void *restrict tzp);
```

它是較舊但仍很常見的 API，解析度到 microseconds。  
規格上雖然被標成比較舊，但現實世界仍很常見。

### `struct tm`

這是所謂 `broken-down time`：

```c
struct tm {
    int tm_sec;
    int tm_min;
    int tm_hour;
    int tm_mday;
    int tm_mon;
    int tm_year;
    int tm_wday;
    int tm_yday;
    int tm_isdst;
};
```

### `gmtime` 與 `localtime`

```c
struct tm *gmtime(const time_t *calptr);
struct tm *localtime(const time_t *calptr);
```

- `gmtime`：轉成 UTC broken-down time
- `localtime`：轉成 local time broken-down time

### `mktime`

```c
time_t mktime(struct tm *tmptr);
```

把 broken-down local time 轉回 `time_t`。

### `strftime`

```c
size_t strftime(char *restrict buf, size_t maxsize,
                const char *restrict format,
                const struct tm *restrict tmptr);
```

它就像時間版的 `printf`。  
你給：

- broken-down time
- format string

它幫你產生可讀字串。

### 常見 format specifiers

- `%Y`：四位數年份
- `%m`：月份
- `%d`：日期
- `%H`：24 小時制小時
- `%M`：分鐘
- `%S`：秒
- `%F`：`YYYY-MM-DD`
- `%T`：`HH:MM:SS`
- `%z`：UTC offset
- `%Z`：time zone name

### `strptime`

```c
char *strptime(const char *restrict buf, const char *restrict format,
               struct tm *restrict tmptr);
```

它是 `strftime` 的反向操作：

- 把字串 parse 成 `struct tm`

### `TZ` environment variable

這章也提醒你：

- `localtime`
- `mktime`
- `strftime`

都會受 `TZ` 影響。

如果你把 `TZ` 改掉，很多格式化結果也會跟著變。

### 範例：印出目前本地時間

```c
#include <stdio.h>
#include <time.h>

int main(void) {
    time_t now;
    struct tm *tm_now;
    char buf[64];

    time(&now);
    tm_now = localtime(&now);
    if (tm_now == NULL) {
        perror("localtime");
        return 1;
    }

    if (strftime(buf, sizeof(buf), "%F %T %Z", tm_now) == 0) {
        fprintf(stderr, "buffer too small\n");
        return 1;
    }

    puts(buf);
    return 0;
}
```

## 6.11 Summary

### 這章讀完你應該真的會的事

- 知道 `struct passwd`、`struct group` 大致在描述什麼
- 會用 `getpwuid` / `getpwnam` / `getgrgid` / `getgrnam`
- 理解 shadow password 存在的安全理由
- 知道 supplementary groups 會影響權限判斷
- 理解 system databases 常見的 `get/set/end` API pattern
- 知道 `uname` / `gethostname` 分別適合什麼問題
- 會分辨 `time_t`、`struct tm`、`timespec`、`timeval`
- 會用 `localtime` + `strftime` 把時間轉成人類可讀格式
- 知道量測經過時間時應優先思考 `CLOCK_MONOTONIC`

### 這章最容易踩坑的地方

- 直接硬 parse `/etc/passwd`，而不是用 API
- 忘記 `getpwuid` / `getgrgid` 這些常回傳 static storage
- 把 `ctime` 當建立時間，或把 `time_t` 直接當顯示格式
- 量測 elapsed time 卻用 `CLOCK_REALTIME`
- 以為所有系統的帳號資料一定都只存在本機純文字檔
- 忘記 `TZ` 會影響 local time 相關函式

### 建議你現在立刻動手做

- 寫一個小程式，用 `getpwuid(getuid())` 印出自己的 home 與 shell
- 用 `getgroups` 列出目前 process 的 supplementary groups
- 寫一個 `uname` 小工具，把所有欄位印出來
- 用 `clock_gettime` 同時印 `CLOCK_REALTIME` 與 `CLOCK_MONOTONIC`
- 改變 `TZ` 後再跑 `strftime`，觀察輸出差異

### 一句總結

這章真正教你的，是 UNIX 系統裡那些看似零碎的帳號、群組、主機、登入、時間資訊，其實都有穩定的資料模型與查詢介面；會用這些介面，程式才真的算是在和整個系統合作。
