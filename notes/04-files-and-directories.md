# Chapter 4 - Files and Directories

## 這章在做什麼

如果說 Chapter 3 教你「怎麼對檔案做 I/O」，那 Chapter 4 就是在教你「檔案到底是什麼」。  
這章真正要建立的是 UNIX file system 的行為模型：

- 檔案名字不等於檔案本體
- directory 不是資料夾圖示，而是「filename -> i-node」的對照表
- 權限判斷看的是 process 的 IDs 和檔案 metadata
- hard link、symbolic link、rename、unlink 的語意都跟這個模型綁在一起

這章一旦搞懂，後面看 process、daemon、terminal、IPC 時，你會比較不容易把 pathname、descriptor、inode、permission 這幾層混在一起。

## 本章小節地圖

- `4.1 Introduction`
- `4.2 stat, fstat, fstatat, and lstat Functions`
- `4.3 File Types`
- `4.4 Set-User-ID and Set-Group-ID`
- `4.5 File Access Permissions`
- `4.6 Ownership of New Files and Directories`
- `4.7 access and faccessat Functions`
- `4.8 umask Function`
- `4.9 chmod, fchmod, and fchmodat Functions`
- `4.10 Sticky Bit`
- `4.11 chown, fchown, fchownat, and lchown Functions`
- `4.12 File Size`
- `4.13 File Truncation`
- `4.14 File Systems`
- `4.15 link, linkat, unlink, unlinkat, and remove Functions`
- `4.16 rename and renameat Functions`
- `4.17 Symbolic Links`
- `4.18 Creating and Reading Symbolic Links`
- `4.19 File Times`
- `4.20 futimens, utimensat, and utimes Functions`
- `4.21 mkdir, mkdirat, and rmdir Functions`
- `4.22 Reading Directories`
- `4.23 chdir, fchdir, and getcwd Functions`
- `4.24 Device Special Files`
- `4.25 Summary of File Access Permission Bits`
- `4.26 Summary`

## 先抓住這章最重要的心智模型

### 名字、檔案、內容是三層

很多人學 UNIX 最常混掉的是這三層：

- pathname / filename：名字
- directory entry：名字對應到哪個 i-node
- i-node：檔案 metadata 與資料位置

白話一點：

- 你 `rm a.txt` 時，先刪掉的是名字
- 你 `ln a.txt b.txt` 時，是多做一個名字指到同一個 i-node
- 你 `rename` 時，常常只是改 directory entry，不一定搬資料

### 權限判斷不是看你「感覺上有沒有權限」

kernel 在做權限判斷時，有很明確的依據：

- file 的 `st_uid`、`st_gid`、permission bits
- process 的 `effective user ID`
- process 的 `effective group ID`
- process 的 `supplementary group IDs`

不是看 login name 文字長什麼樣，也不是看 shell prompt 長怎樣。

### directory 的 execute bit 很重要

對一般檔案來說，`read` / `write` / `execute` 很直覺。  
但對 directory：

- `read`：能列出裡面的名字
- `execute`：能穿過這個 directory 做 pathname search
- `write`：能新增刪除目錄項目

所以 directory 的 `execute bit` 常被叫做 `search bit`。

## 4.1 Introduction

作者在這章承接前一章的 `open/read/write/lseek/close`，開始把焦點轉到：

- file attributes
- 權限與 owner
- file system 結構
- links
- directories

也就是說，Chapter 3 比較像「I/O mechanics」，Chapter 4 比較像「file system semantics」。

## 4.2 `stat`, `fstat`, `fstatat`, `lstat`

這四個函式是整章的中心。

```c
#include <sys/stat.h>

int stat(const char *restrict pathname, struct stat *restrict buf);
int fstat(int fd, struct stat *buf);
int lstat(const char *restrict pathname, struct stat *restrict buf);
int fstatat(int fd, const char *restrict pathname,
            struct stat *restrict buf, int flag);
```

### 四者差異

- `stat(path, &st)`：看 pathname 指到的檔案
- `fstat(fd, &st)`：看已開啟 `fd` 對應的檔案
- `lstat(path, &st)`：如果 path 是 symbolic link，就看 link 自己
- `fstatat(dirfd, path, &st, flag)`：相對於某個 open directory 去查

`fstatat` 是現在很重要的一個介面，因為它讓你：

- 不必依賴 current working directory
- 可以做 race condition 較少的 pathname 操作
- 可以搭配 `AT_SYMLINK_NOFOLLOW` 控制是否跟著 symbolic link

### `fstatat` 要怎麼想

它本質上就是：

- 「在某個已開啟 directory 底下，對某個相對路徑做 `stat`」

幾個重要規則：

- 如果 `path` 是 absolute path，`dirfd` 會被忽略
- 如果 `dirfd == AT_FDCWD` 且 `path` 是相對路徑，就相對於 current working directory
- 如果 `flag` 有 `AT_SYMLINK_NOFOLLOW`，就像 `lstat`

### `struct stat` 裡常見欄位

書上列出的欄位很多，最重要的先記這些：

- `st_mode`：file type + permissions
- `st_ino`：i-node number
- `st_dev`：這個檔案所在 file system 的 device number
- `st_rdev`：special file 真正代表的 device number
- `st_nlink`：link count
- `st_uid`：owner user ID
- `st_gid`：owner group ID
- `st_size`：size in bytes
- `st_atim`：last access time
- `st_mtim`：last modification time
- `st_ctim`：last status change time
- `st_blksize`：建議 I/O block size
- `st_blocks`：實際配置的 disk blocks 數量

### `st_mode` 最容易混

`st_mode` 不是只有 permission bits，它包含兩大類資訊：

- file type bits
- permission bits

所以你不能把它單純當成 `0644` 那種 permission 數值理解。

### 範例：印出檔案基本資訊

```c
#include <stdio.h>
#include <sys/stat.h>

int main(int argc, char *argv[]) {
    struct stat st;

    if (argc != 2) {
        fprintf(stderr, "usage: stat-demo <path>\n");
        return 1;
    }

    if (lstat(argv[1], &st) == -1) {
        perror("lstat");
        return 1;
    }

    printf("inode  = %llu\n", (unsigned long long)st.st_ino);
    printf("mode   = %o\n", st.st_mode);
    printf("nlink  = %llu\n", (unsigned long long)st.st_nlink);
    printf("uid    = %u\n", st.st_uid);
    printf("gid    = %u\n", st.st_gid);
    printf("size   = %lld\n", (long long)st.st_size);
    return 0;
}
```

## 4.3 File Types

APUE 在這裡把 UNIX 常見 file types 一次列清楚：

- `regular file`
- `directory`
- `character special file`
- `block special file`
- `FIFO`
- `socket`
- `symbolic link`

### 重點不是背種類，而是知道「一切都盡量像 file」

這是 UNIX 很核心的設計哲學：

- 一般檔案像 file
- 裝置像 file
- pipe / FIFO 像 file
- socket 也在很多地方被放進 file model 裡

雖然行為不完全一樣，但 API 盡量統一。

### 用 `S_ISxxx` 判斷型別

```c
if (S_ISREG(st.st_mode))   { /* regular file */ }
if (S_ISDIR(st.st_mode))   { /* directory */ }
if (S_ISCHR(st.st_mode))   { /* char special */ }
if (S_ISBLK(st.st_mode))   { /* block special */ }
if (S_ISFIFO(st.st_mode))  { /* fifo */ }
if (S_ISLNK(st.st_mode))   { /* symlink */ }
if (S_ISSOCK(st.st_mode))  { /* socket */ }
```

### 為什麼偵測 symbolic link 時常用 `lstat`

因為 `stat` 會跟著 link 走，所以你很可能根本看不到「它原本是 link」這件事。  
要看 link 本身，應該優先想到 `lstat`。

## 4.4 Set-User-ID and Set-Group-ID

書裡先講 process 身上跟權限有關的 IDs：

- `real user ID`
- `real group ID`
- `effective user ID`
- `effective group ID`
- `supplementary group IDs`
- `saved set-user-ID`
- `saved set-group-ID`

### `real` 跟 `effective` 差在哪

- `real ID`：你原本是誰
- `effective ID`：kernel 做權限判斷時主要拿來看的身分

這點超重要，因為很多 file permission check 是看 `effective ID`。

### `set-user-ID bit` / `set-group-ID bit`

如果一個 executable file 有設定：

- `S_ISUID`
- `S_ISGID`

那它被執行時，process 的 `effective user ID` / `effective group ID` 可能會變成檔案的 owner / group。

### 為什麼這功能存在

典型例子就是 `passwd`：

- 普通使用者要能改自己的密碼
- 但密碼資料通常只有 `root` 能寫

解法就是讓 `passwd` 成為 `set-user-ID root` 的程式。  
程式跑起來後，雖然 real user 不是 root，但 effective user 會是 root。

### 這也是高風險區

只要程式拿到比呼叫者更高的權限，就一定要非常小心：

- 不要信任環境變數
- 不要亂跟 symbolic link 互動
- 不要有 race condition

這也是後面 Chapter 8 會反覆提的安全主題。

## 4.5 File Access Permissions

每個檔案都有九個基本 permission bits：

- user: `rwx`
- group: `rwx`
- other: `rwx`

對應常數像：

- `S_IRUSR`, `S_IWUSR`, `S_IXUSR`
- `S_IRGRP`, `S_IWGRP`, `S_IXGRP`
- `S_IROTH`, `S_IWOTH`, `S_IXOTH`

### `user` / `group` / `other` 的意思

- `user`：檔案 owner
- `group`：檔案 group owner
- `other`：不是 owner，也不在該 group 的其他人

不要把 `o` 想成 owner，`chmod` 裡的 `o` 是 `other`。

### 對 regular file 的意義

- `read`：可讀檔案內容
- `write`：可寫檔案內容
- `execute`：可拿來執行

### 對 directory 的意義

這裡是本章超重要觀念：

- `read`：可以讀出 directory entries，也就是列名字
- `write`：可以在裡面新增 / 刪除 / 改 directory entry
- `execute`：可以 search / traverse 這個 directory

### `read` 跟 `execute` 對 directory 不一樣

很多人第一次卡在這裡。

例如你有某個 directory 的：

- `read` 但沒有 `execute`：你可以知道裡面有哪些名字，但不一定能真正打開它們
- `execute` 但沒有 `read`：你不能列目錄，但如果你已經知道檔名，可能可以直接穿過去打開

### kernel 權限判斷順序

APUE 把判斷規則講得很清楚：

1. 如果 `effective user ID == 0`，通常直接允許
2. 如果 process 擁有該檔案，就看 user bits
3. 否則如果 process 屬於該檔案 group 或 supplementary groups，就看 group bits
4. 否則看 other bits

這四步是依序判斷的，不是全部一起看。

### 很重要的幾個規則

- 只要想經過 pathname 中的某個 directory，就要有該 directory 的 `execute`
- 建立新檔案需要 parent directory 的 `write + execute`
- 刪檔案也是看 parent directory 的 `write + execute`
- 刪檔案通常不看檔案本身有沒有 write permission

這點很反直覺，但非常重要：

- 刪掉的是 directory entry
- 不是直接把 file 內容拿筆塗掉

## 4.6 新建立檔案與目錄的 owner

建立新 file / directory 時：

- `user ID` 通常設成 process 的 `effective user ID`
- `group ID` 則依系統策略不同

POSIX 允許兩種常見策略：

- 用 process 的 `effective group ID`
- 繼承 parent directory 的 `group ID`

### 為什麼會想繼承 directory 的 group

因為這對專案協作很方便。  
例如某個共享目錄就是給某個團隊用，如果所有新檔案都自動繼承同一個 group，管理會容易很多。

### `set-group-ID` on directory

在一些系統上，directory 的 `set-group-ID bit` 會影響：

- 新檔案是否繼承該 directory 的 group
- 新建立的 subdirectory 是否也帶著同樣的 group 規則往下傳

## 4.7 `access` 和 `faccessat`

`open` 權限判斷看的是 `effective IDs`。  
但有時候你想知道的是：

- 以「真正呼叫這個程式的人」的身分，能不能存取某個檔案？

這時就要用 `access` / `faccessat`。

```c
#include <unistd.h>

int access(const char *pathname, int mode);
int faccessat(int fd, const char *pathname, int mode, int flag);
```

### `mode` 常用值

- `F_OK`：檔案是否存在
- `R_OK`：是否可讀
- `W_OK`：是否可寫
- `X_OK`：是否可執行

### 最重要差異

- `access`：用 `real user/group ID` 做測試
- `open`：用 `effective user/group ID` 做測試

### 這在 `set-user-ID` 程式特別重要

因為一個 `set-user-ID root` 程式可能：

- `access(path, R_OK)` 失敗
- 但 `open(path, O_RDONLY)` 卻成功

代表：

- 真正的使用者本來沒權限
- 但程式因為有較高 effective privilege，所以打得開

### `faccessat` 的額外控制

如果加 `AT_EACCESS`，就改成用 `effective IDs` 測試。

## 4.8 `umask`

`umask` 是 process 的「建立檔案時要遮掉哪些 permission bits」。

```c
#include <sys/stat.h>

mode_t umask(mode_t cmask);
```

### 最重要觀念

`umask` 不是「最後權限值」，而是「要關掉哪些 bits」。

也就是說：

```c
requested_mode & ~umask
```

才是最後實際權限。

### 例子

如果你：

- 建檔要求 `0666`
- `umask` 是 `022`

最後就會得到：

- `0644`

### 常見 `umask`

- `002`：其他人不能寫
- `022`：group 與 other 都不能寫
- `027`：group 沒寫權，other 幾乎全關

### 實務提醒

如果程式真的需要某些 bit 一定要打開，不能只寫死 `creat(..., 0666)` 然後覺得事情就結束了，因為 `umask` 可能把它遮掉。

## 4.9 `chmod`, `fchmod`, `fchmodat`

這組函式改的是既有檔案的 mode bits。

```c
#include <sys/stat.h>

int chmod(const char *pathname, mode_t mode);
int fchmod(int fd, mode_t mode);
int fchmodat(int fd, const char *pathname, mode_t mode, int flag);
```

### 誰可以改

- 檔案 owner
- 或 superuser

### 能改哪些 bits

除了九個基本 permission bits 外，還有：

- `S_ISUID`
- `S_ISGID`
- `S_ISVTX`

### 絕對設定 vs 相對設定

你可以直接設定成固定值：

```c
chmod("file", S_IRUSR | S_IWUSR | S_IRGRP | S_IROTH);
```

也可以先 `stat`，再在原值上微調：

```c
mode_t newmode = (st.st_mode & ~S_IXGRP) | S_ISGID;
```

### `ls` 看到大寫 `S` / `T` 是什麼

如果：

- `setuid` 或 `setgid` 開著
- 但對應 execute bit 沒開

`ls -l` 會顯示大寫 `S`。  
`sticky bit` 也是類似道理，可能看到 `T`。

## 4.10 Sticky Bit

`sticky bit` 歷史上原本和 executable 的快取有關，但現在你在現代 UNIX 系統最需要知道的是：

- 它對 directory 有實際意義

### 套在 directory 上代表什麼

就算很多人都對該 directory 有 `write`，也不是每個人都能隨便刪別人的檔案。  
通常還要滿足以下條件之一：

- 你是檔案 owner
- 你是 directory owner
- 你是 superuser

### 典型例子：`/tmp`

`/tmp` 常是很多人都能寫，但為了避免互相亂刪檔，通常會設 sticky bit。

## 4.11 `chown`, `fchown`, `fchownat`, `lchown`

這組函式改 owner 與 group。

### 幾個重要現實規則

- 一般使用者通常不能隨便把檔案 owner 改成別人
- 如果系統有 `_POSIX_CHOWN_RESTRICTED`，限制會更明顯
- 檔案 owner 有時只能把 group 改成自己所屬的 group

### `lchown` 的用途

它是改 symbolic link 本身的 owner，而不是 link 指向的檔案。

### 一個常被忽略的副作用

非 superuser 若成功 `chown`，常常會讓：

- `set-user-ID`
- `set-group-ID`

被清掉。

理由很合理：避免某人把檔案 owner / group 改完後，還保有危險的 privilege bits。

## 4.12 File Size

`st_size` 對不同 file type 意義不同。

### regular file

- 就是檔案大小（bytes）

### directory

- 是 directory file 自己的大小

### symbolic link

- 是 link 內容中字串長度
- 不包含 C string 結尾的 `'\0'`

### `st_blksize` 和 `st_blocks`

- `st_blksize`：比較理想的 I/O block size
- `st_blocks`：實際配置了多少磁碟 block

### sparse file / holes

這是 APUE 很喜歡強調的點。

如果你：

1. `lseek` 跳到很後面
2. 再寫幾個 bytes

中間沒真的寫過的區域就可能形成 `hole`。

好處是：

- 檔案邏輯大小很大
- 實際磁碟占用可能很小

壞處是：

- 用一般工具複製後，hole 可能被真的 0 bytes 補滿，導致磁碟占用暴增

## 4.13 File Truncation

```c
#include <unistd.h>

int truncate(const char *pathname, off_t length);
int ftruncate(int fd, off_t length);
```

### 做什麼

- 把檔案縮到指定大小
- 或把檔案延長到指定大小

### 兩種情況

- 原本比 `length` 大：尾巴被切掉
- 原本比 `length` 小：中間新增區域讀起來是 0，通常形成 hole

### `O_TRUNC` 是特殊情況

`open(..., O_TRUNC)` 可以看成「把長度直接變 0」的簡化版 truncate。

## 4.14 File Systems

這節很關鍵，因為它解釋了很多前面看似怪異的行為。

### file system 的抽象圖

可以把它想成：

- 磁碟被切成 partitions
- partition 裡有 file system
- file system 裡有 i-nodes、data blocks、directories

### i-node 裡放什麼

i-node 大致放：

- file type
- permissions
- owner / group
- timestamps
- link count
- data block pointers

### directory entry 放什麼

directory entry 主要是：

- filename
- i-node number

所以 directory 不是把整個檔案內容塞進去，它只是做 mapping。

### hard link 為什麼不能跨 file system

因為 directory entry 裡的 i-node number，只能指向同一個 file system 內的 i-node。  
跨 file system 根本沒有共同的 i-node 空間。

### rename 為什麼常常很便宜

如果 rename 沒有跨 file system，常常只是：

- 新增一個 directory entry
- 移掉舊的 directory entry

資料本體通常不用搬。

### directory 的 link count 怎麼來

目錄的 `link count` 不是亂數，它跟：

- 自己名字那個 entry
- 目錄中的 `.`
- 子目錄中的 `..`

有關。

所以每多一個子目錄，parent directory 的 link count 也常會加 1。

## 4.15 `link`, `linkat`, `unlink`, `unlinkat`, `remove`

### `link`

`link` 是建立 `hard link`：

```c
#include <unistd.h>

int link(const char *existingpath, const char *newpath);
```

它做的不是複製資料，而是：

- 多做一個 directory entry 指到同一個 i-node

### `hard link` 的限制

- 通常不能跨 file system
- 通常不能對 directory 建立 hard link

### `unlink`

`unlink(path)` 刪的是 directory entry，不是立刻保證刪除資料本體。

檔案真正被回收通常要同時滿足：

- link count 變成 0
- 沒有 process 還開著它

這是理解 UNIX 暫存檔技巧的核心。

### 為什麼 open 之後再 unlink 仍可用

因為：

- pathname 已經不在 namespace 裡
- 但 open file description 還活著

只要 process 還持有那個 open file，資料就還在。

### 經典暫存檔模式

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(void) {
    int fd = open("tempfile", O_RDWR | O_CREAT | O_TRUNC, 0600);
    if (fd == -1) {
        perror("open");
        return 1;
    }

    if (unlink("tempfile") == -1) {
        perror("unlink");
        close(fd);
        return 1;
    }

    write(fd, "hello\n", 6);
    close(fd);
    return 0;
}
```

這種做法的意思是：

- 先開
- 立刻把名字拿掉
- 等最後 `close` 時再真正回收

### `remove`

`remove` 比較像高階包裝：

- 如果是 file，就像 `unlink`
- 如果是 directory，就像 `rmdir`

## 4.16 `rename` 和 `renameat`

`rename` 是非常重要的 atomic 操作。

### 它保證了什麼

對同一個 file system 裡的 rename，常見用途是：

- 先寫新檔
- 寫完後 `rename` 蓋掉舊檔

這樣能讓「看見新版本」這件事幾乎是一次切換，不會出現半寫好的狀態。

### 幾個重要規則

- old 是 file，new 若存在，不能是 directory
- old 是 directory，new 若存在，必須是空 directory
- 不處理 symbolic link 指向的目標，而是處理 link 本身
- 不能 rename `.` 或 `..`
- 如果 old 和 new 本來就是同一個檔案，直接成功但不做事

### 很務實的理解

`rename` 是做「名字切換」，不是做「內容複製」。

## 4.17 Symbolic Links

`symbolic link` 和 `hard link` 最大差別：

- `hard link` 直接指向 i-node
- `symbolic link` 只是存一段 pathname 字串

### symbolic link 的優點

- 可以跨 file system
- 可以指向 directory
- 可以指向根本不存在的 target

### 你一定要知道哪些函式會 follow symlink

書裡強調這件事，因為不同函式對 symbolic link 的態度不同。  
幾個直覺版記法：

- `stat`, `open`, `chdir`, `opendir`：通常 follow
- `lstat`, `readlink`, `unlink`, `remove`, `rename`：多半針對 link 自己

### `O_CREAT | O_EXCL` 的特殊規則

如果 `open` 同時帶：

- `O_CREAT`
- `O_EXCL`

遇到 symbolic link 會失敗。  
這是故意的安全設計，避免 privileged program 被騙去寫錯地方。

### symlink loop

symbolic link 可以做出 loop，例如：

- `foo/testdir -> ../foo`

之後遞迴走目錄就可能繞圈圈。  
很多函式在這種情況下會回 `ELOOP`。

## 4.18 建立與讀取 symbolic link

```c
#include <unistd.h>

int symlink(const char *actualpath, const char *sympath);
ssize_t readlink(const char *restrict pathname,
                 char *restrict buf, size_t bufsize);
```

### `symlink`

建立一個新的 symbolic link，內容是 `actualpath`。  
重點是：

- target 不需要真的存在
- 不需要和 link 在同一個 file system

### `readlink`

`readlink` 會把 link 裡存的字串讀出來。  
但最重要的坑是：

- 它不會自動補 `'\0'`

所以常見正確寫法是：

```c
#include <stdio.h>
#include <unistd.h>

int main(void) {
    char buf[1024];
    ssize_t n = readlink("mylink", buf, sizeof(buf) - 1);
    if (n == -1) {
        perror("readlink");
        return 1;
    }

    buf[n] = '\0';
    printf("target = %s\n", buf);
    return 0;
}
```

## 4.19 File Times

每個檔案有三個超重要時間：

- `st_atim`：last access time
- `st_mtim`：last modification time
- `st_ctim`：last status change time

### 最容易搞混的是 `ctime`

`ctime` 不是 creation time。  
它是：

- i-node status 最後改變時間

例如：

- `chmod`
- `chown`
- link count 改變

都會動到 `ctime`，即使檔案內容沒變。

### 三者白話區分

- `atime`：最後一次被讀
- `mtime`：檔案內容最後一次被改
- `ctime`：metadata 最後一次被改

## 4.20 `futimens`, `utimensat`, `utimes`

這組函式可以改檔案的 access / modification time。

### `times` 參數的四種語意

1. `times == NULL`：兩個時間都設成現在
2. `tv_nsec == UTIME_NOW`：對應時間設成現在
3. `tv_nsec == UTIME_OMIT`：對應時間不改
4. 給一般數值：設成指定秒數與奈秒數

### 權限規則

如果只是設成現在，條件比較寬：

- owner
- 或有 write permission
- 或 superuser

但如果你要設成某個特定歷史時間，通常要：

- owner
- 或 superuser

單純有 write permission 不夠。

### 注意

你不能直接手動指定 `ctime`。  
因為 `ctime` 本來就是系統拿來記錄 metadata 改變的時間。

## 4.21 `mkdir`, `mkdirat`, `rmdir`

### `mkdir`

建立新 directory，並自動建立：

- `.`
- `..`

### 建 directory 時最常犯的錯

把 mode 寫得像一般檔案，例如只給 `rw-`。  
但 directory 通常至少要有某些 `execute` bits，否則連 search 都做不了。

### 新目錄權限也會被 `umask` 影響

你指定的 `mode` 不是最終值，還是會先經過 `umask`。

### `rmdir`

只能刪空目錄。  
空目錄的意思是：

- 只有 `.`
- 和 `..`

如果還有別的 entry，就不行。

## 4.22 Reading Directories

POSIX 提供目錄讀取 API：

```c
#include <dirent.h>

DIR *opendir(const char *pathname);
struct dirent *readdir(DIR *dp);
int closedir(DIR *dp);
void rewinddir(DIR *dp);
long telldir(DIR *dp);
void seekdir(DIR *dp, long loc);
```

### `DIR *` 就像 directory 版的 `FILE *`

它是 library 幫你維護的讀目錄狀態。

### `struct dirent`

最重要成員通常是：

- `d_ino`
- `d_name`

其中 `d_name` 是 null-terminated filename。

### `readdir` 的使用習慣

- 成功：回傳某一筆 entry
- 結束或錯誤：回傳 `NULL`

如果真的需要分辨錯誤，通常要搭配 `errno`。

### 遞迴走目錄時為什麼常用 `lstat`

因為如果用 `stat`，你可能一路 follow symbolic link，最後：

- 重複計數
- 甚至撞到 loop

### 範例：列出目錄所有名字

```c
#include <dirent.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
    DIR *dp;
    struct dirent *ent;

    if (argc != 2) {
        fprintf(stderr, "usage: lsmini <dir>\n");
        return 1;
    }

    dp = opendir(argv[1]);
    if (dp == NULL) {
        perror("opendir");
        return 1;
    }

    while ((ent = readdir(dp)) != NULL) {
        printf("%s\n", ent->d_name);
    }

    closedir(dp);
    return 0;
}
```

## 4.23 `chdir`, `fchdir`, `getcwd`

### current working directory 是 process 屬性

這句非常重要。

所以如果你寫一個程式呼叫 `chdir("/tmp")`，它只會改自己那個 process 的 cwd，不會改呼叫它的 shell。

這就是為什麼：

- `cd` 必須是 shell built-in

### `getcwd`

拿目前工作目錄的 absolute pathname。

```c
#include <unistd.h>

char *getcwd(char *buf, size_t size);
```

### `fchdir`

如果你先把某個 directory 打開，之後可以直接：

- `fchdir(fd)`

回到那個位置。  
這常比字串版 `getcwd` / `chdir` 更穩。

### `getcwd` 和 symbolic link

書裡有一個很經典的例子：  
如果你是透過 symbolic link 走到某個地方，`getcwd` 算出來的路徑可能不會保留你原本經過的 symbolic link 名字，而是給你實際走上去後推回來的絕對路徑。

## 4.24 Device Special Files

這節在澄清 `st_dev` 與 `st_rdev`。

### `st_dev`

表示：

- 這個檔案所在 file system 的 device number

### `st_rdev`

只有 character special file / block special file 才有意義，表示：

- 這個 special file 代表的實際裝置號碼

### `major` / `minor`

device number 通常可拆成：

- `major`：裝置驅動大類
- `minor`：具體子裝置

所以：

- 同一顆磁碟不同 partition，可能 `major` 相同但 `minor` 不同

## 4.25 權限 bits 總整理

這章看完，建議你把 permission bits 分成兩大組記：

### 特殊 bits

- `S_ISUID`
- `S_ISGID`
- `S_ISVTX`

### 三組基本 bits

- `S_IRWXU`
- `S_IRWXG`
- `S_IRWXO`

再展開就是：

- `S_IRUSR`, `S_IWUSR`, `S_IXUSR`
- `S_IRGRP`, `S_IWGRP`, `S_IXGRP`
- `S_IROTH`, `S_IWOTH`, `S_IXOTH`

### 對 regular file 與 directory 的語意不一樣

這一定要分開記：

- 對 regular file：讀、寫、執行
- 對 directory：列名、改 entry、search path

## 4.26 Summary

### 這章讀完你應該真的會的事

- 知道 `stat` / `lstat` / `fstatat` 分別適合什麼情境
- 知道 pathname、directory entry、i-node 是不同層次
- 理解 UNIX 權限判斷是怎麼依 user/group/other 逐步決定
- 分得清 `hard link` 和 `symbolic link`
- 理解 `unlink` 是刪名字，不一定立刻刪資料
- 知道 `ctime` 不是 creation time
- 會用 `opendir` / `readdir` / `closedir`
- 知道 `cwd` 是 process 屬性，不是 shell magically 跟著變

### 這章最容易踩坑的地方

- 把 `stat` 當成一定能看出 symbolic link
- 以為刪檔案需要檔案本身的 write permission
- 把 directory 的 `read` 與 `execute` 當成同一件事
- 把 `ctime` 誤認成建立時間
- 忘記 `readlink` 不會補 `'\0'`
- 認為 `rename` 等於資料搬家
- 以為 `umask` 是最後權限，而不是遮罩

### 建議你現在立刻動手做

- 寫一個小工具，用 `lstat` 印出檔案型別與 link count
- 建一個 sparse file，比較 `ls -l` 和 `du`
- 練習 `open` 後 `unlink` 的暫存檔模式
- 手做幾個 symbolic link，觀察 `stat` / `lstat` / `readlink` 結果
- 寫一個遞迴列目錄程式，先用 `stat` 再改成 `lstat`，比較差異

### 一句總結

這章真正教你的不是一堆零散 API，而是 UNIX file system 的基本世界觀：名字、metadata、資料、權限、連結、目錄，其實是彼此解耦但又嚴密配合的幾層抽象。
