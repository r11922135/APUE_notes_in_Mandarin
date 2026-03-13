# Chapter 17 - Advanced IPC

## 這章在做什麼

前一章你看到的是：

- `pipe`
- `FIFO`
- message queue
- semaphore
- shared memory

這一章則把 IPC 再往前推一步，進入比較「UNIX 味」也比較「系統服務味」的做法：

- `UNIX domain socket`
- unique connection
- passing file descriptors
- open server

如果說 Chapter 15 比較像：

- process 之間怎麼傳資料

那 Chapter 17 更像是在講：

- 本機 process 之間怎麼做像 service / daemon 一樣的合作
- 怎麼傳遞「已經打開的資源」
- 怎麼讓 server 對每個 client 都有一條乾淨的專屬通道

這些主題在很多 daemon、privileged helper、service broker、supervisor 類工具裡都非常重要。

## 本章小節地圖

- `17.1 Introduction`
- `17.2 UNIX Domain Sockets`
- `17.3 Unique Connections`
- `17.4 Passing File Descriptors`
- `17.5 An Open Server, Version 1`
- `17.6 An Open Server, Version 2`
- `17.7 Summary`

## 先抓住這章最重要的心智模型

### 這章的核心不是「再學一種 socket」，而是「本機服務架構」

`UNIX domain socket` 表面上看起來只是：

- network socket API 的本機版

但這一章真正要你看到的是：

- 本機通訊不只是在傳 byte
- 還可以傳「access rights」
- 還可以讓 server 幫 client 打開資源
- 還可以讓一個 daemon 同時服務多個不相關 process

### file descriptor 不是小整數而已，而是 process 對 kernel 物件的入口

這章最關鍵的一個觀念是：

- 傳 file descriptor，不是把一個 int 數字丟給別人

真正發生的是：

- kernel 在接收端建立一個新的 descriptor entry
- 讓接收者也能存取同一個 open file description / socket / device

這代表你能在 process 間轉交的，不只是資料，而是：

- 已經打開的檔案
- 已經建立好的 socket
- 已經驗證過權限的資源

### local IPC 的一個高階套路：well-known endpoint + per-client connection

前一章很多 IPC 比較像：

- 大家共用某個 queue
- 或共用某塊 memory

這章更像 modern server 的思維：

- server 公告一個 well-known 名字
- client 主動連進來
- kernel 幫每個 client 建一條專屬連線

這樣做的好處是：

- request / reply 比較乾淨
- 權限與 client 身份比較容易管理
- 同時服務多個 client 也比較自然

## 17.1 Introduction

APUE 一開始就點出這章的主角：

- `UNIX domain socket`

它跟 Chapter 16 的 Internet domain socket 一樣屬於 socket 世界，  
但使用範圍主要是：

- 同一台主機上的 process

書裡特別強調幾件事：

- server 可以把自己的 descriptor 綁上一個名字
- client 可以用這個名字找到 server
- server 可以對每個 client 建立一條 unique IPC channel
- process 之間甚至可以傳遞 open file descriptors

也就是說，這章是在教你：

- 怎麼把本機 IPC 提升到 service-style communication

## 17.2 UNIX Domain Sockets

### 什麼是 UNIX domain socket

`UNIX domain socket` 是一種本機 IPC 機制，使用：

- `AF_UNIX` 或 `AF_LOCAL`

它最大的特點是：

- API 長得像 socket
- 使用範圍卻是本機

所以你可以用 Chapter 16 學過的許多操作：

- `socket`
- `bind`
- `listen`
- `accept`
- `connect`
- `sendmsg`
- `recvmsg`

但它不像 TCP/IP 那樣需要：

- protocol processing
- network header
- checksum
- sequence number
- acknowledgement

因此在本機通訊上通常更有效率。

### UNIX domain socket 支援 stream 和 datagram

它可以有：

- `SOCK_STREAM`
- `SOCK_DGRAM`

而且 APUE 特別提醒：

- UNIX domain datagram service 是 reliable 的
- message 不會平白遺失，也不會亂序送達

這點和很多人對「datagram = 不可靠」的直覺不同。  
那個直覺主要來自：

- Internet domain 的 UDP

### `socketpair` 是本章第一個超實用工具

```c
#include <sys/socket.h>

int socketpair(int domain, int type, int protocol, int sockfd[2]);
```

最常用的形式是：

```c
socketpair(AF_UNIX, SOCK_STREAM, 0, fd);
```

它會直接幫你建立一對：

- 已經互相連好的 UNIX domain socket

### `socketpair` 很像 full-duplex pipe

APUE 把這種東西看成：

- full-duplex pipe

也就是：

- `fd[0]` 可讀可寫
- `fd[1]` 可讀可寫

這和普通 `pipe` 不同。  
普通 `pipe` 在可攜語意上應視為：

- half-duplex

所以如果你需要雙向溝通、又只在本機，`socketpair` 常常比兩條 pipe 更乾淨。

### 範例：用 `socketpair` 做雙向通訊

```c
#include <sys/socket.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    int fd[2];
    pid_t pid;
    char buf[128];
    ssize_t n;

    if (socketpair(AF_UNIX, SOCK_STREAM, 0, fd) < 0) {
        perror("socketpair");
        return 1;
    }

    pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        close(fd[0]);
        write(fd[1], "child -> parent\n", 16);
        n = read(fd[1], buf, sizeof(buf) - 1);
        if (n > 0) {
            buf[n] = '\0';
            fputs(buf, stdout);
        }
        close(fd[1]);
        _exit(0);
    }

    close(fd[1]);
    n = read(fd[0], buf, sizeof(buf) - 1);
    if (n > 0) {
        buf[n] = '\0';
        fputs(buf, stdout);
    }
    write(fd[0], "parent -> child\n", 16);
    close(fd[0]);
    waitpid(pid, NULL, 0);
    return 0;
}
```

這個例子最值得你注意的是：

- 同一個 endpoint 同時可讀可寫

這正是 `socketpair` 和普通 pipe 最大的使用感差異。

### 用 UNIX domain socket 間接把 message queue 接到 `poll/select`

APUE 在這節有一個很有代表性的例子：

- XSI message queue 本身不能直接拿去 `poll/select`
- 因為它不是 file descriptor

怎麼辦？

- 用 thread 卡在 `msgrcv`
- message 一到，就寫進 UNIX domain socket
- 主程式只監看這個 socket

這個技巧很有啟發性。它告訴你：

- socket 不只是「一種通訊管道」
- 還可以當成「可被事件系統監看的橋接層」

### 17.2.1 Naming UNIX Domain Sockets

`socketpair` 建出來的是：

- unnamed、已互連的本機 socket

如果你要讓不相關的 process 也能找到 server，就要像網路 socket 一樣：

- 為 socket 綁名字

### `sockaddr_un`

UNIX domain socket 的地址結構是：

- `struct sockaddr_un`

重點欄位：

- `sun_family = AF_UNIX`
- `sun_path = pathname`

也就是它用一個 filesystem pathname 來當 rendezvous 名字。

### `bind` 之後，檔案系統裡會出現 socket 節點

當 server 對 UNIX domain socket 做：

- `bind`

系統會在檔案系統中建立一個 type 為：

- `S_IFSOCK`

的節點。

這個檔案存在的意義主要是：

- 讓 client 知道要連哪個名字

不是拿來 `open` 當普通檔案用的。

### 一定要記得 `unlink`

這是 UNIX domain socket server 常見坑點。

如果 socket pathname 已存在，再次 `bind` 同一路徑通常會失敗，常見錯誤是：

- `EADDRINUSE`

而且就算 server 結束了：

- 這個 pathname 也不會自動消失

所以實務上常見流程是：

1. `unlink(old_path)`
2. `bind`
3. 退出前再清理一次

### `sockaddr_un` 的長度計算不要偷懶

APUE 特別提醒：

- 有些實作在 `sockaddr_un` 前面欄位不同

因此計算地址長度時，常用：

```c
size = offsetof(struct sockaddr_un, sun_path) + strlen(un.sun_path);
```

這比直接：

```c
sizeof(struct sockaddr_un)
```

更貼近教材想強調的可攜觀念。

### 範例：bind 一個 named UNIX domain socket

```c
#include <sys/socket.h>
#include <sys/un.h>
#include <stddef.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    int fd;
    struct sockaddr_un un;
    socklen_t len;

    fd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (fd < 0) {
        perror("socket");
        return 1;
    }

    memset(&un, 0, sizeof(un));
    un.sun_family = AF_UNIX;
    strcpy(un.sun_path, "/tmp/apue.sock");

    unlink(un.sun_path);

    len = (socklen_t)(offsetof(struct sockaddr_un, sun_path) +
        strlen(un.sun_path));

    if (bind(fd, (struct sockaddr *)&un, len) < 0) {
        perror("bind");
        close(fd);
        return 1;
    }

    puts("socket bound");
    close(fd);
    unlink("/tmp/apue.sock");
    return 0;
}
```

## 17.3 Unique Connections

前一節只是說：

- server 可以有一個 well-known name

這一節更進一步講的是：

- client 連進來之後，server 怎麼得到一條「屬於這個 client 的唯一連線」

### 觀念上和 TCP server 很像

流程很像 Chapter 16 的 TCP：

- server：`bind` + `listen`
- client：`connect`
- server：`accept`

一旦 `accept` 成功：

- server 會拿到一個新的 connected socket
- 這條 connection 專屬於那個 client

這樣你就不再是「大家混在同一條資料流裡」。

### 為什麼 unique connection 很重要

因為它解決了很多 server 設計上的混亂：

- 多 client 的 request / reply 不會混線
- 每個 client 可以有自己的狀態
- 權限檢查和記錄較清楚
- connection 關閉時，server 也比較容易知道是哪個 client 離開

### APUE 封裝了三個 helper 的概念

書裡設計了三個函式：

- `serv_listen`
- `serv_accept`
- `cli_conn`

你不用死背它們的實作，但要懂它們在做什麼：

- `serv_listen(name)`：server 公告一個 well-known name
- `serv_accept(listenfd, &uid)`：接受新連線，並盡量取得 client 身份資訊
- `cli_conn(name)`：client 連到 server 名字

### 這些 helper 背後在幹嘛

本質上還是：

- 建 socket
- `bind`
- `listen`
- `connect`
- `accept`

只是在 UNIX domain socket 上，額外還要處理：

- pathname
- 權限
- socket file 的清理

### 身份資訊是本機服務設計的重要資產

APUE 的 helper 會想辦法讓 server 知道：

- 這個 client 是誰

這對本機服務非常重要，因為很多 server 不只看：

- 你送了什麼 request

還會看：

- 你是誰
- 你有沒有資格要求這件事

這也是為什麼本機 UNIX domain service 常常比單純 byte pipe 更適合做 privileged service。

## 17.4 Passing File Descriptors

這一節是整章最招牌的主題。

### passing file descriptor 到底是什麼

意思不是：

- 把數字 `3` 傳給另一個 process

因為對方自己的 `fd 3` 可能完全不是同一件東西。

真正的意思是：

- kernel 幫接收端建立一個新的 descriptor
- 讓它指向發送端原本已打開的那個資源

所以你傳遞的是：

- 存取權
- 已打開的 kernel object 入口

### 為什麼這件事這麼強

因為你可以做這種設計：

- low-privilege client 把開檔需求交給 privileged server
- server 驗證允許後，替 client `open`
- server 再把這個 open fd 傳回 client

如此 client 最後拿到的是：

- 可以直接 `read/write` 的 fd

但它自己不需要一開始就具備開那個檔案的權限。

### 只能透過 UNIX domain socket 傳

APUE 很明確：

- access rights 只能跨 UNIX domain socket 傳

不是一般 TCP socket、也不是 pipe 隨便就能做。

### 用 `sendmsg` / `recvmsg`，不是 `write` / `read`

因為除了普通 data 外，你還要傳：

- ancillary data

這就要用：

- `sendmsg`
- `recvmsg`

### `msghdr` / `cmsghdr` 是這套機制的骨架

你會遇到幾個重要結構：

- `struct msghdr`
- `struct cmsghdr`

以及幾個重要 macro：

- `CMSG_DATA`
- `CMSG_FIRSTHDR`
- `CMSG_NXTHDR`
- `CMSG_LEN`

你不必一開始背到滾瓜爛熟，但至少要知道：

- control message 不是 payload 本身
- 它放在 `msg_control`
- descriptor 就塞在 `SCM_RIGHTS` 對應的 control message 裡

### `SCM_RIGHTS`

要傳 file descriptor 時，control message 典型設定是：

- `cmsg_level = SOL_SOCKET`
- `cmsg_type = SCM_RIGHTS`

這個名字可以直接記成：

- socket-level control message for access rights

### 範例：用 `SCM_RIGHTS` 傳一個 fd

下面是簡化版骨架，重點是理解資料擺放方式：

```c
#include <sys/socket.h>
#include <string.h>
#include <unistd.h>

int send_fd_simple(int sock, int fd_to_send) {
    struct msghdr msg;
    struct iovec iov;
    char data = 0;
    char control[CMSG_LEN(sizeof(int))];
    struct cmsghdr *cmsg;

    memset(&msg, 0, sizeof(msg));
    memset(control, 0, sizeof(control));

    iov.iov_base = &data;
    iov.iov_len = 1;
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    msg.msg_control = control;
    msg.msg_controllen = sizeof(control);

    cmsg = CMSG_FIRSTHDR(&msg);
    cmsg->cmsg_level = SOL_SOCKET;
    cmsg->cmsg_type = SCM_RIGHTS;
    cmsg->cmsg_len = CMSG_LEN(sizeof(int));
    *(int *)CMSG_DATA(cmsg) = fd_to_send;

    return sendmsg(sock, &msg, 0);
}
```

這裡最值得注意的是：

- 真的要送的 descriptor 不在普通 payload 裡
- 而是在 control data 裡

### APUE 的 `send_fd` / `recv_fd` 還多做了一層小協定

書裡沒有只是「純傳 fd」，還做了一個簡單 protocol：

- 如果成功，就傳一個狀態值 0 再附帶 fd
- 如果失敗，就傳錯誤訊息與負狀態

這樣 client 就能同時收到：

- 成功時的 descriptor
- 或失敗時的錯誤原因

這是很典型的 IPC protocol 設計思維。

### credentials passing：server 不只想知道你傳了什麼，還想知道你是誰

APUE 接著還提到：

- 有些系統支援在 UNIX domain socket 上一起傳送 credentials

但各平台接口不同：

- FreeBSD 類：`SCM_CREDS`
- Linux 類：`SCM_CREDENTIALS` 與 `ucred`

這部分最重要的不是背平台差異，而是知道：

- 本機 IPC 有機會把「身份資訊」和「資源轉交」整合在一起

這讓本機 daemon 設計更強大。

## 17.5 An Open Server, Version 1

這一節把前面學到的 fd passing 變成一個完整案例：

- open server

### open server 在解什麼問題

它不是把檔案內容讀回 client，而是：

- server 幫 client 開檔
- 然後把「開好的 descriptor」傳回去

這樣的好處很大：

- 不限 regular file
- device、socket 等也能處理
- 真正透過 IPC 傳的只有 request 與 descriptor，不是整個檔案內容

### 版本 1 的架構：一個 client 對一個 server child

這一版流程大致是：

1. client 建立一條 full-duplex `fd_pipe`
2. client `fork`
3. child `exec` 出 open server 程式
4. client 用 pipe 送 request
5. server 幫忙 `open`
6. server 用 `send_fd` 把 descriptor 傳回去

### request protocol 很簡單

APUE 設計的 request 格式大意是：

- `"open <pathname> <openmode>\0"`

也就是：

- command name
- pathname
- open flag 的 ASCII 數字
- 末尾 `\0`

這裡很值得學的不是格式本身，而是：

- IPC protocol 不一定複雜
- 只要雙方約好，簡單到一個 null-terminated command 也可以

### 為什麼把 server 寫成獨立可執行程式

APUE 說了幾個好處：

- 任何 client 都能重用
- server 改版時，不必每個 client 都重編
- server 可以是 set-user-ID 程式，取得 client 沒有的權限

這是很典型的「privileged helper」設計思維。

### `csopen` 在幹嘛

client 端的 `csopen` 可以把它理解成：

- `open` 的遠端代理版

它做的事是：

- 第一次呼叫時建立通道並啟動 server
- 之後把 filename / oflag 寫給 server
- 再用 `recv_fd` 等 server 回一個 descriptor

對呼叫端而言，它看起來有點像：

- `open(path, flags)`

只是真正去做 `open` 的是另一個 process。

### 這版的優點和限制

優點：

- 架構直觀
- 容易測試
- 很適合示範 child -> parent descriptor passing

限制：

- 每個 client 都要 `fork + exec` 一個 server
- 如果 client 很多，成本較高

這就導向下一節的 daemon 版。

## 17.6 An Open Server, Version 2

第 2 版改成：

- 單一 daemon server 處理所有 client

### 這版的核心變化

不再是：

- client 自己 `fork + exec` server

而是：

- server 先在背景跑起來
- 公告一個 well-known UNIX domain socket 名字
- client 用 `cli_conn` 連進去
- server 同時處理多個 client

### well-known name

APUE 用了像這樣的名字：

```c
"/tmp/opend.socket"
```

這個名字的角色就像：

- 本機 service 的固定入口

### 版本 2 真正示範的是「不相關 process 之間的 fd passing」

版本 1 還有 parent / child 關係。  
版本 2 則是：

- 完全不相關的 client 和 daemon
- 仍然可以透過 UNIX domain socket 交換 descriptor

這點非常重要，因為它讓 fd passing 變成真正可用於系統服務架構的能力。

### server 要維護 client 狀態

既然一個 daemon 要服務多個 client，它就必須管理：

- 哪些 client 已連上
- 每個 client 的 fd 是多少
- 每個 client 的 uid 是多少

APUE 用：

- `Client` array

來記錄這些狀態。

### `select` 版與 `poll` 版

書裡用兩種方式實作事件迴圈：

- `select`
- `poll`

這裡真正想讓你複習的是 Chapter 14：

- 一個單進程 daemon 可以靠 I/O multiplexing 同時服務多個 client

### `select` 版重點

它會：

- 把 `listenfd` 放進 `fd_set`
- 若 `listenfd` readable，表示新 client 到來
- 若某個 client fd readable，表示有 request 或對方關閉連線

這是經典的：

- event loop + accept + per-connection state

### `poll` 版重點

`poll` 版和 `select` 版本質一樣，只是資料結構換成：

- `struct pollfd`

APUE 也順便讓你看到：

- client 數量多時，陣列可能要動態擴張
- `POLLIN` 與 `POLLHUP` 要分開處理

### 這版的 `handle_request`

核心流程其實沒變：

1. 驗證 request 格式
2. 解析 pathname / open mode
3. `open`
4. 成功就 `send_fd`
5. 失敗就 `send_err`

變的是：

- 現在這段程式是在 daemon 裡跑
- 所以錯誤處理要偏向 logging，而不是直接寫 stderr

### 版本 2 的真正價值

這一節不只是 open server 範例，而是在示範一種典型架構：

- daemon 持有較高權限
- client 用 well-known socket 接入
- server 驗證 request 與身份
- server 把已開資源轉交回 client

這種模式比「直接讓所有人都擁有高權限」安全很多，也更好管理。

## 17.7 Summary

這章表面上是在講：

- `UNIX domain socket`

但本質上是在講：

- 本機服務程式的高階 IPC 模型

你應該把這章整理成幾個關鍵能力：

- 用 `socketpair` 做 full-duplex 本機通訊
- 用 named UNIX domain socket 做 service rendezvous
- 用 `accept` 為每個 client 建 unique connection
- 用 `SCM_RIGHTS` 傳遞 descriptor
- 用 daemon + `select/poll` 同時服務多 client

### 這章讀完你應該真的會的事

- 知道為什麼本機 service 常用 UNIX domain socket，而不是硬用 TCP loopback
- 知道 `socketpair` 和普通 pipe 的使用差異
- 知道 named UNIX domain socket 要處理 pathname 與清理
- 知道 fd passing 傳的是 access rights，不是單純數字
- 知道 `sendmsg/recvmsg` 與 `SCM_RIGHTS` 在幹嘛
- 看得懂 open server 兩個版本背後的系統設計意圖

### 這章最容易踩坑的地方

- 忘記 `unlink` 舊的 socket pathname，導致 `bind` 失敗
- 以為把整數 fd 值傳出去就等於分享同一個 descriptor
- 把 `sendmsg/recvmsg` 的 ancillary data 和普通 payload 混在一起
- 忘記 server 和 client 可以是完全不相關的 process
- 只看懂 open server 程式碼，卻沒看懂它其實是在示範 privileged helper 架構

### 建議你現在立刻動手做

1. 自己寫一個 `socketpair` 範例，和兩條 pipe 版本比較，感受 full-duplex 的差異。
2. 寫一個最小的 UNIX domain socket echo server，先練 `bind/listen/accept/connect`。
3. 再進一步做一個「server 幫 client 開檔並回傳 fd」的簡化版，真正把 `SCM_RIGHTS` 用起來。

### 一句總結

這章是在教你：本機 IPC 不只可以傳資料，還可以建立真正像 service 一樣的連線模型，甚至把已開好的資源安全地轉交給別的 process。
