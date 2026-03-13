# Chapter 16 - Network IPC: Sockets

## 這章在做什麼

前一章講的是：

- 同一台機器上的經典 IPC

這一章把視野往外拉，進入：

- `socket`

`socket` 是 UNIX / POSIX 世界裡最重要的 network IPC 介面之一。  
它不只是拿來做網路程式，也是一種非常核心的系統抽象：

- 你可以把 communication endpoint 表示成 file descriptor
- 再用統一風格的 API 去做連線、收送資料、控制行為

這一章的重點是 TCP/IP 上的 socket 程式設計基本骨架，也就是：

- 怎麼建立 socket
- 怎麼表示位址
- client 怎麼 `connect`
- server 怎麼 `bind` / `listen` / `accept`
- 資料怎麼傳
- socket 還能怎麼調整與控制

## 本章小節地圖

- `16.1 Introduction`
- `16.2 Socket Descriptors`
- `16.3 Addressing`
- `16.4 Connection Establishment`
- `16.5 Data Transfer`
- `16.6 Socket Options`
- `16.7 Out-of-Band Data`
- `16.8 Nonblocking and Asynchronous I/O`
- `16.9 Summary`

## 先抓住這章最重要的心智模型

### socket 本質上是「帶有通訊語意的 file descriptor」

這句話很重要。

socket 跟一般 file descriptor 的共通點是：

- 可以 `read`
- 可以 `write`
- 可以 `close`
- 可以拿去 `select` / `poll`

但它不是普通檔案，所以你不能把它想成：

- 一個可以任意 `lseek` 的 file

更正確的理解是：

- 這是一個 endpoint
- 它有 protocol family、type、protocol 等通訊屬性

### socket programming 其實就是四件事

你可以把這章壓縮成四個問題：

1. 這個 endpoint 要用哪種 transport？
2. 對方位址怎麼表示？
3. 怎麼建立關係？
4. 建立後怎麼收送與控制？

如果這四件事你都清楚，socket 就不會像一長串 API。

### stream 和 datagram 是兩種完全不同的思維

這章很容易犯的錯，是把 `SOCK_STREAM` 和 `SOCK_DGRAM` 當成只有「可靠性不同」。

其實不只如此。

`SOCK_STREAM`：

- connection-oriented
- reliable
- byte stream
- 沒有 message boundary

`SOCK_DGRAM`：

- connectionless
- message-oriented
- 保留 message boundary
- 不保證到達、不保證順序

也就是說，兩者連應用層 protocol 的寫法都不一樣。

## 16.1 Introduction

APUE 在這章把 socket 放在「network IPC」脈絡下介紹。  
重點是：

- socket API 可以用在同機器，也可以用在不同機器
- 但最常見、最有代表性的用途，還是 TCP/IP 網路通訊

所以這章雖然叫 network IPC，本質上也是在講：

- UNIX 如何把網路通訊整合進 file-descriptor 世界

這種設計非常成功，因為它讓你可以把很多現有 I/O 工具直接延伸到網路程式：

- `read` / `write`
- `close`
- `fcntl`
- `select` / `poll`

## 16.2 Socket Descriptors

### socket 是用 `socket()` 建立的

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

三個參數分別回答三件事：

- `domain`：位址家族
- `type`：通訊型態
- `protocol`：具體協定

### 常見的 address family

你最常看到的是：

- `AF_INET`：IPv4
- `AF_INET6`：IPv6
- `AF_UNIX`：本機 UNIX domain socket
- `AF_UNSPEC`：通常用於查詢時表示「不限 family」

這章主軸放在：

- `AF_INET`
- `AF_INET6`

### 常見的 socket type

- `SOCK_STREAM`
- `SOCK_DGRAM`
- `SOCK_SEQPACKET`
- `SOCK_RAW`

你一定要先搞懂前兩個。

#### `SOCK_STREAM`

特性：

- connection-oriented
- reliable
- full-duplex
- 資料是 byte stream

通常在 TCP/IP 裡，這就代表：

- TCP socket

#### `SOCK_DGRAM`

特性：

- connectionless
- message-oriented
- 每次傳送是一個 datagram

通常在 TCP/IP 裡，這就代表：

- UDP socket

#### `SOCK_SEQPACKET`

它同時保留：

- connection-oriented
- message boundary

但比前兩者少見。  
你知道有這種東西就好，入門實作最常用的還是 `STREAM` 和 `DGRAM`。

#### `SOCK_RAW`

這是更低層、直接碰原始封包的接口，通常需要較高權限。  
一般應用層 server / client 不會先從這裡入門。

### `protocol` 常常可以寫 `0`

很多時候你會看到：

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
```

因為：

- 對 `AF_INET + SOCK_STREAM`
- kernel 會自動選對應的預設 protocol，也就是 TCP

同理：

- `AF_INET + SOCK_DGRAM + 0`

通常就是 UDP。

### socket 是 fd，但不是所有 fd 操作都合理

你可以對 socket 做：

- `read` / `write`
- `recv` / `send`
- `close`
- `select` / `poll`
- `fcntl`

但你不該對 socket 做：

- `lseek`

因為 socket 不是可 seek 的檔案，通常會得到：

- `ESPIPE`

### `read/write` 和 `recv/send` 的關係

對很多 socket 而言：

- `read` 大致相當於 `recv(..., 0)`
- `write` 大致相當於 `send(..., 0)`

那為什麼還要 `recv` / `send`？

因為它們有 `flags`，可以啟用更多通訊控制語意。

### `close` 和 `shutdown` 不一樣

`close(sockfd)`：

- 關閉這個 descriptor
- 若這是最後一個 reference，socket 資源才真的被釋放

`shutdown(sockfd, how)`：

- 關閉通訊方向的一部分

常見的 `how`：

- `SHUT_RD`
- `SHUT_WR`
- `SHUT_RDWR`

這很重要，因為 network 程式常常有這種需求：

- 我已經送完 request 了
- 但我還想繼續讀 server 的回應

這時你可以：

- `shutdown(sockfd, SHUT_WR)`

而不是整顆 socket 直接 `close`。

### 範例：建立一個 TCP socket

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <stdio.h>
#include <unistd.h>

int main(void) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) {
        perror("socket");
        return 1;
    }

    puts("socket created");
    close(fd);
    return 0;
}
```

這段範例雖然短，但它代表的其實是：

- 我拿到了一個 network endpoint 的控制權

## 16.3 Addressing

socket 要通訊，不能只有 descriptor，還需要：

- address

在網路世界裡，address 通常包含：

- host
- service

也就是你熟悉的：

- IP address
- port number

### network byte order

網路協定慣例使用：

- big-endian

這叫做 `network byte order`。  
因此你在處理數值欄位時，常需要：

- `htons`
- `htonl`
- `ntohs`
- `ntohl`

你可以簡單記：

- `h` = host
- `n` = network
- `s` = short
- `l` = long

### 為什麼有 `struct sockaddr`

各種 address family 的地址格式不一樣，所以系統設計了一個通用介面概念：

- `struct sockaddr`

但真正 IPv4 常用的是：

- `struct sockaddr_in`

IPv6 常用的是：

- `struct sockaddr_in6`

呼叫 API 時通常會看到這種 cast：

```c
struct sockaddr_in sin;
bind(fd, (struct sockaddr *)&sin, sizeof(sin));
```

這不是因為 `sockaddr_in` 就等於 `sockaddr`，而是：

- API 用 generic pointer 型態接收不同 family 的地址結構

### IPv4 地址結構

觀念上最重要的是：

- family
- port
- address

像這樣：

```c
struct sockaddr_in sin;
sin.sin_family = AF_INET;
sin.sin_port = htons(8080);
```

IP 位址欄位也要用正確格式填入。

### 文字位址 與 二進位位址 的轉換

你平常看到：

- `"127.0.0.1"`
- `"::1"`

是文字格式。  
kernel 真正需要的是 binary network address。

常見工具函式：

- `inet_pton`
- `inet_ntop`

`inet_pton`：

- presentation to network

`inet_ntop`：

- network to presentation

### `getaddrinfo` 是現代程式該優先用的查詢接口

APUE 這章很重要的一個訊息是：

- 舊的主機名稱與服務查詢 API 有歷史包袱
- 比較建議用 `getaddrinfo`

它把：

- 主機名稱解析
- service 名稱解析
- IPv4 / IPv6 選擇
- socket type 過濾

都整合在一起。

常見搭配：

- `freeaddrinfo`
- `gai_strerror`

### 範例：用 `getaddrinfo` 解析位址

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <stdio.h>
#include <string.h>

int main(void) {
    struct addrinfo hints;
    struct addrinfo *res;
    int err;

    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;

    err = getaddrinfo("localhost", "8080", &hints, &res);
    if (err != 0) {
        fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(err));
        return 1;
    }

    puts("address resolved");
    freeaddrinfo(res);
    return 0;
}
```

### server 常用 `AI_PASSIVE`

如果你是在做 server，要準備一個「本地端可綁定的位址」，常會這樣設：

```c
hints.ai_flags = AI_PASSIVE;
```

然後把 host 傳 `NULL`，表示：

- 幫我找一個適合 `bind` 的本地位址

## 16.4 Connection Establishment

這一節是整章的核心。  
你一定要把 client 和 server 的流程分清楚。

### client 端通常做什麼

典型流程：

1. `socket`
2. `connect`

如果用 `getaddrinfo`，通常會變成：

1. `getaddrinfo`
2. 逐一嘗試每個結果
3. `socket`
4. `connect`

### server 端通常做什麼

典型 TCP server 流程：

1. `socket`
2. `bind`
3. `listen`
4. `accept`

這四步請務必牢記。

### `bind` 是把 socket 跟本地位址綁在一起

server 需要讓 client 找得到，所以要把 socket 綁到：

- 某個 local address
- 某個 port

這就是 `bind`。

client 不一定非得自己 `bind`，因為 kernel 往往可以自動挑一個臨時 port。  
但 server 幾乎一定要 `bind`。

### `listen` 把主動 endpoint 變成被動 endpoint

TCP server 在 `bind` 後，還要呼叫：

- `listen`

這代表：

- 我不是要拿這個 socket 主動連出去
- 我是要等別人連進來

`listen` 的 backlog 參數可以理解成：

- 等待被 `accept` 的連線佇列提示值

它不是「無限排隊」。

### `accept` 會回傳新的 connected socket

這是 server 端最容易搞混的一點。

`accept` 不是讓 listening socket 本身變成已連線狀態，而是：

- 回傳一個新的 socket fd
- 這個新 fd 才是和某個 client 的 connection

所以 server 通常同時持有兩種 socket：

- listening socket：專門等新客戶
- connected socket：專門跟某個客戶收送資料

### 範例：典型 TCP server 骨架

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(void) {
    struct addrinfo hints, *res, *rp;
    int listenfd = -1;
    int connfd;
    int on = 1;

    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE;

    if (getaddrinfo(NULL, "8080", &hints, &res) != 0) {
        return 1;
    }

    for (rp = res; rp != NULL; rp = rp->ai_next) {
        listenfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
        if (listenfd < 0) {
            continue;
        }

        setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));

        if (bind(listenfd, rp->ai_addr, (socklen_t)rp->ai_addrlen) == 0) {
            break;
        }

        close(listenfd);
        listenfd = -1;
    }

    freeaddrinfo(res);

    if (listenfd < 0) {
        return 1;
    }

    if (listen(listenfd, 16) < 0) {
        perror("listen");
        close(listenfd);
        return 1;
    }

    connfd = accept(listenfd, NULL, NULL);
    if (connfd < 0) {
        perror("accept");
        close(listenfd);
        return 1;
    }

    write(connfd, "hello\n", 6);

    close(connfd);
    close(listenfd);
    return 0;
}
```

### 範例：典型 TCP client 骨架

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

int main(void) {
    struct addrinfo hints, *res, *rp;
    int fd = -1;

    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;

    if (getaddrinfo("127.0.0.1", "8080", &hints, &res) != 0) {
        return 1;
    }

    for (rp = res; rp != NULL; rp = rp->ai_next) {
        fd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
        if (fd < 0) {
            continue;
        }

        if (connect(fd, rp->ai_addr, (socklen_t)rp->ai_addrlen) == 0) {
            break;
        }

        close(fd);
        fd = -1;
    }

    freeaddrinfo(res);

    if (fd < 0) {
        return 1;
    }

    puts("connected");
    close(fd);
    return 0;
}
```

## 16.5 Data Transfer

連線建好後，下一步就是收送資料。

### 常見 API

- `send`
- `recv`
- `sendto`
- `recvfrom`
- `sendmsg`
- `recvmsg`

如果你只是在 stream socket 上做最基本的讀寫，也可以直接用：

- `read`
- `write`

### stream socket 沒有 message boundary

這是 TCP 程式設計最核心的一點。

如果你連續做兩次：

- `write(fd, "abc", 3)`
- `write(fd, "def", 3)`

對方不一定會在兩次 `read` 中剛好讀到：

- `"abc"`
- `"def"`

它可能一次讀到：

- `"abcdef"`

也可能先讀到：

- `"ab"`

後面再讀到：

- `"cdef"`

所以 TCP 應用層一定要自己定義 framing，例如：

- 固定長度
- length-prefixed
- delimiter-based

### datagram socket 保留 message boundary

UDP 的一個 datagram 就是一筆訊息。  
receiver 一次 `recvfrom` 對應一個 datagram。

這跟 TCP 的 byte stream 思維完全不同。

### `recv` 回傳 `0` 的意義

對 stream socket 而言，`recv` 或 `read` 回傳 `0` 通常表示：

- 對方做了 orderly shutdown

最常見理解就是：

- 對方已經把寫方向關掉了

這和一般 file 的 EOF 類似，但語意背景是 connection state。

### `send` / `recv` 也可能 short

很多人以為：

- network send 一次就一定把整個 buffer 送完

這是錯的。

尤其在 nonblocking 或高負載情況下：

- `send` 可能只送出一部分
- `recv` 也可能只收到一部分

這也是為什麼 Chapter 14 的：

- `readn`
- `writen`

在 network code 很常用。

### 範例：簡單的收送迴圈

```c
#include <sys/socket.h>
#include <unistd.h>
#include <stdio.h>

ssize_t writen(int fd, const void *buf, size_t n);

int echo_once(int fd) {
    char buf[1024];
    ssize_t n;

    n = recv(fd, buf, sizeof(buf), 0);
    if (n < 0) {
        perror("recv");
        return -1;
    }
    if (n == 0) {
        return 0;
    }

    if (writen(fd, buf, (size_t)n) != n) {
        perror("writen");
        return -1;
    }

    return 1;
}
```

### `shutdown(SHUT_WR)` 在 request / reply 很常見

假設 client 把 request 送完後，希望告訴 server：

- 我不會再送資料了
- 但我還要讀你的回應

這時就很適合：

```c
shutdown(fd, SHUT_WR);
```

這比直接 `close(fd)` 更精準。

### APUE 的經典例子：網路服務搭配 `popen`

這章常見的教學模式是：

- server 接受 client 連線
- server 執行某個本地 command
- 再把 command 輸出送回 client

例如把：

- `/usr/bin/uptime`

的輸出透過 socket 傳出去。  
這個例子很適合你觀察：

- process control
- standard I/O
- network socket

是怎麼串在一起工作的。

## 16.6 Socket Options

socket 不只是建立完就用，它還能透過 option 調整行為。

### 基本 API

```c
int getsockopt(int sockfd, int level, int optname,
               void *optval, socklen_t *optlen);

int setsockopt(int sockfd, int level, int optname,
               const void *optval, socklen_t optlen);
```

你可以把它理解成：

- `level`：這是哪一層的選項
- `optname`：你要調哪個選項
- `optval`：你給它或取回的值

常見 `level`：

- `SOL_SOCKET`
- 某些 protocol-specific level，例如 `IPPROTO_TCP`

### 最常見的例子：`SO_REUSEADDR`

server 重啟時，很常遇到：

- `bind` 失敗
- `EADDRINUSE`

原因常常和前一個連線狀態，例如 `TIME_WAIT`，有關。  
這時很多 server 會在 `bind` 前設定：

- `SO_REUSEADDR`

```c
int on = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
```

這幾乎是 server 骨架的標準配備。

### 其他常見 option

- `SO_KEEPALIVE`：偵測長時間失聯的 peer
- `SO_RCVBUF` / `SO_SNDBUF`：收送 buffer 大小
- `SO_LINGER`：控制 `close` 的 linger 行為
- `TCP_NODELAY`：關閉 Nagle algorithm，降低小封包延遲

你不用一次把全部背完，但要知道：

- socket 行為很多都能透過 option 微調

## 16.7 Out-of-Band Data

這節介紹的是：

- TCP urgent / out-of-band data

它聽起來像：

- 一條額外的高速通道

但實際上你不要這樣想。  
更接近的理解是：

- TCP stream 中有一個 urgent 標記機制

### 為什麼這功能很少在現代應用層協定當主角

因為它有幾個問題：

- 語意不如名字直覺
- 不同系統的細節與期待常讓人誤解
- 很多應用層自己做 control message 反而更清楚

常見相關機制：

- `MSG_OOB`
- `SIGURG`

你知道它存在、知道不要把它當萬能優先通道，就夠了。

### 最重要的實務觀念

如果你只是想在 protocol 裡插入：

- 高優先級命令
- 中止要求
- 特殊控制訊號

大多數情況直接在應用層 protocol 中設計 control message，會比依賴 OOB data 更可預測。

## 16.8 Nonblocking and Asynchronous I/O

這一節本質上是在把 Chapter 14 的工具，帶回 socket 世界。

### nonblocking socket

你可以透過 `fcntl` 設定：

- `O_NONBLOCK`

之後像：

- `connect`
- `accept`
- `read`
- `write`
- `recv`
- `send`

若當下不能立刻完成，就不會一直卡住，而是回傳：

- `EAGAIN`
- `EWOULDBLOCK`

某些情況下是：

- `EINPROGRESS`

### nonblocking `connect` 是常見考點

對 nonblocking socket 呼叫 `connect` 時，如果連線還在進行中，常見結果是：

- `-1`
- `errno = EINPROGRESS`

這不代表失敗，而是表示：

- 連線還沒完成

接下來你通常會：

- 用 `select` / `poll` 等它變 writable
- 再用 `getsockopt(..., SO_ERROR, ...)` 檢查最後結果

這個流程非常常見。

### signal-driven I/O

某些系統也支援用：

- `O_ASYNC`
- `SIGIO`

做 signal-driven I/O。  
也就是事件來時，kernel 用 signal 通知 process。

這種模型能用，但設計上通常比：

- `select`
- `poll`

更難寫對。

### 真正的重點不是 API，而是 event model

socket 一旦進入 nonblocking 模式，你的程式設計模型就會改變成：

- 嘗試操作
- 如果現在不能做，就等事件
- 準備好之後再繼續

這和「一路直線 blocking 呼叫」是完全不同的思考方式。

### 範例：nonblocking `connect` 的骨架

```c
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/select.h>
#include <fcntl.h>
#include <errno.h>
#include <stdio.h>
#include <unistd.h>

int wait_connect(int fd, const struct sockaddr *sa, socklen_t salen) {
    int flags, err;
    socklen_t len = sizeof(err);
    fd_set wset;

    flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);

    if (connect(fd, sa, salen) == 0) {
        return 0;
    }
    if (errno != EINPROGRESS) {
        return -1;
    }

    FD_ZERO(&wset);
    FD_SET(fd, &wset);

    if (select(fd + 1, NULL, &wset, NULL, NULL) <= 0) {
        return -1;
    }

    if (getsockopt(fd, SOL_SOCKET, SO_ERROR, &err, &len) < 0) {
        return -1;
    }

    if (err != 0) {
        errno = err;
        return -1;
    }

    return 0;
}
```

這段最值得你記住的不是程式碼本身，而是流程：

- `connect`
- 遇到 `EINPROGRESS`
- 等 writable
- 查 `SO_ERROR`

## 16.9 Summary

這章其實是在教你如何把：

- network communication

放進 UNIX 的 file-descriptor 思維裡。

最核心的幾件事是：

- socket 是 endpoint，也是 fd
- 先決定 family / type / protocol
- 位址表示要處理 family 與 byte order
- client 走 `connect`
- server 走 `bind` / `listen` / `accept`
- stream 與 datagram 的應用層寫法完全不同
- socket 可以進一步用 options、nonblocking、I/O multiplexing 做更成熟的設計

### 這章讀完你應該真的會的事

- 知道 `socket()` 三個參數各自在決定什麼
- 能分清 `SOCK_STREAM` 和 `SOCK_DGRAM`
- 能說出 client 與 server 建立連線的完整流程
- 知道為什麼 `accept` 會產生新的 connected socket
- 知道 TCP 沒有 message boundary
- 知道 `SO_REUSEADDR`、`shutdown`、`O_NONBLOCK` 這些工具在實務上的用途

### 這章最容易踩坑的地方

- 把 TCP 當成一筆一筆 message 自然分隔的協定
- 以為一次 `send` / `recv` 就一定處理完整 buffer
- 把 listening socket 和 connected socket 搞混
- 忘記 port 要用 network byte order
- nonblocking `connect` 遇到 `EINPROGRESS` 就誤判為失敗
- 不理解 `close` 與 `shutdown` 的差別

### 建議你現在立刻動手做

1. 自己寫一個最小 TCP echo server 和 client，只用 `read` / `write`，體會 byte stream 沒有 message boundary。
2. 把 server 加上 `SO_REUSEADDR`，然後反覆重啟，觀察 `bind` 行為差異。
3. 把 client 改成 nonblocking `connect`，再用 `select` 等待完成，真正理解 `EINPROGRESS` 的流程。

### 一句總結

這章是在教你：socket 不是一堆零散 API，而是一整套把「位址、連線、資料流、事件控制」統一進 UNIX I/O 模型的通訊抽象。
