# Chapter 15 - Interprocess Communication

## 這章在做什麼

前面幾章你已經學到：

- process 會 `fork`
- process 會 `exec`
- parent 可以 `wait`
- process 之間可以用 `signal` 通知彼此

但真正的系統程式通常不只需要「通知」，還需要：

- 傳資料
- 做 request / reply
- 共享狀態
- 控制誰先做、誰後做

這一章就是在講這些事，也就是 `IPC`，`interprocess communication`。

APUE 在這章主要介紹的是「同一台機器上的經典 IPC」：

- `pipe`
- `FIFO`
- XSI IPC
- POSIX semaphores

這些機制看起來很多，但你可以把它們理解成：  
每一種 IPC 都是在回答「process 要怎麼碰頭、怎麼傳資料、怎麼同步」這三個問題，只是回答方式不同。

## 本章小節地圖

- `15.1 Introduction`
- `15.2 Pipes`
- `15.3 popen and pclose Functions`
- `15.4 Coprocesses`
- `15.5 FIFOs`
- `15.6 XSI IPC`
- `15.7 Message Queues`
- `15.8 Semaphores`
- `15.9 Shared Memory`
- `15.10 POSIX Semaphores`
- `15.11 Client-Server Properties`
- `15.12 Summary`

## 先抓住這章最重要的心智模型

### IPC 其實在回答幾個固定問題

當你挑一種 IPC 機制時，實際上是在問：

- 兩邊怎麼找到彼此
- 資料是 byte stream 還是 message
- 有沒有 message boundary
- 彼此是不是一定要有親緣關係
- kernel 物件會活多久
- blocking 行為是什麼
- 誰負責同步

如果你先把這幾個問題釐清，這章就不會像在背一堆 API。

### 傳資料 和 同步，是兩件不同的事

這章很容易混淆的地方是：

- 有些工具主要拿來「搬資料」
- 有些工具主要拿來「做同步」

例如：

- `pipe` / `FIFO` / message queue：偏向傳資料
- shared memory：偏向共享資料本體
- semaphore：偏向同步

尤其 shared memory 最容易被誤會。  
它確實很快，但它只幫你「共享同一塊記憶體」，不會自動幫你解決 race condition。

### IPC 的差別，常常不是功能有沒有，而是語意細節

很多初學者會想：

- 反正都能傳資料，那差在哪？

差在這些地方：

- `pipe` 沒有 message boundary
- message queue 有 message boundary，還能依 type 收訊息
- shared memory 幾乎不需要 copy，但同步要自己做
- semaphore 幾乎不傳資料，它是在管「可不可以進去」
- `FIFO` 能讓無親緣關係的 process 溝通，但 request / reply 設計比較麻煩

這章的重點不是背 function 名字，而是理解這些 trade-off。

## 15.1 Introduction

APUE 一開始先提醒你：

- 其實 open file 也可以是一種 IPC 基礎
- filesystem 也能當某種程度的資料交換媒介

但如果你需要的是更直接、更系統化的 communication，就需要這章介紹的工具。

這裡還有兩個很重要的背景：

### 傳統 IPC 大多先鎖定同一台主機

這章多數機制的設計背景是：

- 兩個 process 在同一個 kernel 裡

如果你要跨機器，通常會用：

- `socket`

那是下一章的重點。

### `pipe` 在可攜角度上要當成 half-duplex

某些 UNIX 系統的 pipe 可以雙向用，但 SUS 的可攜保證不是這樣。  
所以你在寫 portable code 時，要把 pipe 當成：

- 一端讀
- 一端寫
- 單向資料流

如果你要雙向溝通，通常做法是：

- 用兩條 pipe

## 15.2 Pipes

`pipe` 是 UNIX 最老牌、也最經典的 IPC 機制之一。

你可以把它想成：

- kernel 幫你準備一個 buffer
- 一端拿來寫
- 一端拿來讀

資料從寫端進去，從讀端出來。

### `pipe` 建立後會拿到兩個 file descriptor

```c
#include <unistd.h>

int fd[2];
if (pipe(fd) < 0) {
    /* error */
}
```

建立成功後：

- `fd[0]` 是 read end
- `fd[1]` 是 write end

這兩個 descriptor 和一般 file descriptor 很像，所以：

- 可以 `read`
- 可以 `write`
- 可以 `close`
- 可以 `dup2`

但不能拿去做像 `lseek` 這種不適合 stream 的事。

### pipe 最典型的用法是 `fork` 後 parent / child 溝通

因為 `fork` 會複製 file descriptor table，所以：

- parent 建 pipe
- 再 `fork`
- parent / child 都會拿到同一條 pipe 的兩端

這就是它最自然的使用方式。

### 一定要關掉不用的那一端

這是 pipe 最重要、也最容易踩坑的地方之一。

如果 parent 要寫給 child，典型寫法是：

- child 關 `fd[1]`
- parent 關 `fd[0]`

原因不是「整潔」而已，而是語意正確性。

因為 pipe 的 EOF 判定跟「所有寫端是否都關掉」有關。

### `read` 什麼時候會看到 EOF

對 pipe 而言，`read` 回傳 `0` 表示：

- 所有 write end 都已關閉
- 而且 buffer 裡的資料也讀完了

所以如果某個 process 明明不用寫，卻一直沒把 write end 關掉，讀端就可能：

- 一直等
- 永遠看不到 EOF

這是最常見的 pipe bug。

### 沒有人讀時去寫，會發生什麼事

如果你對 pipe 或 FIFO 的寫端 `write`，但系統已經沒有任何讀者，kernel 會：

- 送出 `SIGPIPE`

如果 process 沒特別處理，通常就直接被終止。  
如果你忽略或攔截了 `SIGPIPE`，那 `write` 會失敗並回傳：

- `-1`
- `errno = EPIPE`

這個語意非常重要，因為它讓 sender 知道：

- 對面已經不在了

### `PIPE_BUF` 保證的是「不被別人插隊」，不是「永遠寫得進去」

APUE 會特別提到 `PIPE_BUF`。

它的意思是：

- 如果多個 writer 同時往同一條 pipe 寫資料
- 且某次 `write` 的大小 `<= PIPE_BUF`
- 那這次寫入在 interleaving 上會是 atomic

也就是說，不會出現：

- 前半段來自 process A
- 後半段突然被 process B 插進來

但如果寫入量超過 `PIPE_BUF`，就可能被切開、交錯。

要注意：

- 這個保證是關於 interleaving
- 不是保證你永遠不會 block

### pipe 很適合線性資料流，不適合複雜拓撲

pipe 的優點：

- API 很小
- 和 `fork` / `exec` 天然配合
- 非常適合 shell pipeline 思維

pipe 的限制：

- 可攜角度上是單向
- 通常用在有共同祖先的 process
- 沒有 message boundary
- 不適合做複雜多 client 管理

### 範例：parent 透過 pipe 傳資料給 child

```c
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void) {
    int fd[2];
    pid_t pid;
    const char *msg = "hello from parent\n";
    char buf[128];
    ssize_t n;

    if (pipe(fd) < 0) {
        perror("pipe");
        return 1;
    }

    pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        close(fd[1]);

        while ((n = read(fd[0], buf, sizeof(buf) - 1)) > 0) {
            buf[n] = '\0';
            fputs(buf, stdout);
        }

        close(fd[0]);
        _exit(0);
    }

    close(fd[0]);
    write(fd[1], msg, strlen(msg));
    close(fd[1]);

    waitpid(pid, NULL, 0);
    return 0;
}
```

這個例子最值得你觀察的不是 `write` 本身，而是：

- child 一開始就關寫端
- parent 寫完就關寫端

這樣 child 的 `read` 才能正確讀到 EOF。

### `dup2` 讓 pipe 變成 shell pipeline 的基礎

為什麼 shell 可以做：

```sh
ls | wc -l
```

因為 shell 會在背後做類似這種事：

- 建 pipe
- `fork`
- 把前一個 command 的 `stdout` 用 `dup2` 接到 pipe 寫端
- 把下一個 command 的 `stdin` 用 `dup2` 接到 pipe 讀端
- 再 `exec`

這也是為什麼 APUE 常把 pipe 視為後面很多 IPC 工具的基礎教材。

## 15.3 `popen` and `pclose` Functions

如果你覺得：

- `pipe` + `fork` + `dup2` + `exec`

每次都自己拼很麻煩，那標準 I/O library 提供了一個高階包裝：

- `popen`

### `popen` 讓你把某個 command 當成一個 `FILE *`

```c
#include <stdio.h>

FILE *popen(const char *cmdstring, const char *type);
int pclose(FILE *fp);
```

`type` 只有兩種主要模式：

- `"r"`：你讀 command 的標準輸出
- `"w"`：你寫到 command 的標準輸入

也就是說：

- `"r"` 時，你的程式是 reader
- `"w"` 時，你的程式是 writer

### `popen` 背後通常還是 `pipe + fork + exec shell`

通常它會做：

- 建 pipe
- `fork`
- child 用 `/bin/sh -c cmdstring` 去執行命令
- parent 拿到對應方向的 `FILE *`

所以你要知道兩件事：

- command string 會經過 shell 解讀
- shell metacharacter、redirection、quote 都會生效

這很方便，但如果 command 是用不可信輸入拼出來的，就有 command injection 風險。

### `pclose` 不只是 `fclose`

`pclose` 會做的不只是把 stream 關掉，它還會：

- 等待子 process 結束

所以它回傳的其實是：

- 子 process 的 termination status

這點很重要，因為如果你只 `fclose`，你會漏掉 child 的收尾與 exit status。

### `popen` 很方便，但 buffering 要小心

因為你拿到的是 `FILE *`，所以你會受到 stdio buffering 影響。  
如果你在 `"w"` 模式下寫給對方，卻沒有：

- 換行
- `fflush`
- 關閉 stream

對方可能一直等不到資料。

### 範例：讀取某個 command 的輸出

```c
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    FILE *fp;
    char line[256];

    fp = popen("ls -1 notes", "r");
    if (fp == NULL) {
        perror("popen");
        return 1;
    }

    while (fgets(line, sizeof(line), fp) != NULL) {
        fputs(line, stdout);
    }

    if (pclose(fp) == -1) {
        perror("pclose");
        return 1;
    }

    return 0;
}
```

適合 `popen` 的情境是：

- 你只需要單向和某個 shell command 互動
- 你要快速把 command 結果整合進 C 程式

不適合的情境是：

- 你需要雙向互動
- 你需要精準控制 child 的執行細節
- 你不想讓 shell 介入

## 15.4 Coprocesses

`coprocess` 可以把它理解成：

- 你啟動一個 child process
- 但不是只單向丟資料給它
- 而是雙向互動

最典型的做法是：

- 一條 pipe：parent -> child
- 另一條 pipe：child -> parent

### 雙向溝通幾乎一定要兩條 pipe

如果你想做這種互動：

- parent 丟一行命令
- child 算完再吐一行結果

那就需要兩個方向都能走。  
可攜做法就是兩條 pipe。

### 雙向互動最常踩的是 buffering deadlock

例如：

- parent 用 stdio 寫資料，但沒 `fflush`
- child 在等完整一行才處理
- child 的輸出也被 buffer 住
- parent 又在等 child 回應

整個系統就卡住了。

所以做 coprocess 時你要特別小心：

- 對 `FILE *` 寫入後要不要 `fflush`
- 對方是 line-buffered 還是 fully buffered
- 有沒有兩邊都在等彼此先開口

### 範例：用兩條 pipe 跟 child 雙向互動

```c
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void) {
    int to_child[2];
    int from_child[2];
    pid_t pid;
    char buf[256];
    ssize_t n;

    if (pipe(to_child) < 0 || pipe(from_child) < 0) {
        perror("pipe");
        return 1;
    }

    pid = fork();
    if (pid < 0) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        dup2(to_child[0], STDIN_FILENO);
        dup2(from_child[1], STDOUT_FILENO);

        close(to_child[0]);
        close(to_child[1]);
        close(from_child[0]);
        close(from_child[1]);

        execlp("tr", "tr", "a-z", "A-Z", (char *)0);
        _exit(127);
    }

    close(to_child[0]);
    close(from_child[1]);

    write(to_child[1], "hello, coprocess\n", 17);
    close(to_child[1]);

    while ((n = read(from_child[0], buf, sizeof(buf) - 1)) > 0) {
        buf[n] = '\0';
        fputs(buf, stdout);
    }

    close(from_child[0]);
    waitpid(pid, NULL, 0);
    return 0;
}
```

這個例子看起來簡單，但背後其實已經是很多 command filter 的基本模型。

## 15.5 FIFOs

`FIFO` 可以想成：

- 有名字的 pipe

也就是：

- pipe 的資料流語意還在
- 但它不再只能靠 `fork` 繼承 descriptor 才能相遇

它在 filesystem 裡有 pathname，所以不相關的 process 也可以透過這個名字碰頭。

### 建立 FIFO 用 `mkfifo`

```c
#include <sys/stat.h>

int mkfifo(const char *path, mode_t mode);
```

例如：

```c
if (mkfifo("/tmp/apue_fifo", 0666) < 0) {
    /* error */
}
```

建立後，別的 process 就可以：

- 開這個 pathname 來讀
- 開這個 pathname 來寫

### FIFO 的 blocking open 行為很重要

FIFO 的難點不是建立，而是 `open` 的語意。

一般情況下：

- 只讀開啟 FIFO：會等到有 writer
- 只寫開啟 FIFO：會等到有 reader

如果你加 `O_NONBLOCK`：

- 讀端通常可以先打開
- 寫端若此時沒有任何 reader，`open` 可能失敗並回 `ENXIO`

這些細節會直接影響你的 client / server 設計。

### FIFO 解掉了「必須有共同祖先」的限制

這是 FIFO 最大的價值。

pipe 常見於：

- parent / child

FIFO 可以用在：

- 完全不相關的兩個 process
- 只要大家都知道同一個 pathname

### 但 FIFO 還是 byte stream，不是 message queue

它保留了 pipe 的核心特性：

- 沒有 message boundary
- 讀多少、寫多少，要自己設計 protocol

所以如果多個 client 同時寫同一個 FIFO，你還是要思考：

- request 要怎麼 framing
- 哪些寫入大小必須控制在 `PIPE_BUF` 內

### FIFO 做 client / server 的典型模型

常見做法是：

- server 有一個 well-known FIFO，例如 `/tmp/server.fifo`
- client 把 request 寫進去
- server 回應時，不直接共用同一條回傳
- 而是每個 client 自己再準備一條 reply FIFO

這樣才能避免：

- 多個 client 回應混在一起

實務上還有一個很常見的小技巧：

- server 會把自己的 well-known FIFO 同時以 read / write 方式打開

這樣做的目的不是要讓單一 FIFO 變雙向，而是：

- 避免當前沒有任何 client writer 時，server 的讀端太早看到 EOF

### 範例：建立 FIFO 並接收資料

```c
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    const char *path = "/tmp/apue_demo_fifo";
    char buf[128];
    ssize_t n;
    int fd;

    mkfifo(path, 0666);

    fd = open(path, O_RDONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    while ((n = read(fd, buf, sizeof(buf))) > 0) {
        fwrite(buf, 1, (size_t)n, stdout);
    }

    close(fd);
    return 0;
}
```

這裡最值得注意的是：

- `open(path, O_RDONLY)` 可能 block

如果你不理解這點，寫出來的程式常常會像「卡住」，其實只是還沒等到另一端。

## 15.6 XSI IPC

接下來 APUE 進入比較「System V / XSI 風格」的 IPC。

包含三種：

- message queues
- semaphores
- shared memory

這三種機制的共同特色是：

- 都不是用 file descriptor 辨識
- 而是用 kernel 管理的 IPC identifier

### XSI IPC 很依賴 key 的概念

通常流程是：

- 先決定一個 `key_t`
- 再用這個 key 去建立或取得 IPC object

常見來源有：

- `ftok(pathname, id)`
- `IPC_PRIVATE`

`IPC_PRIVATE` 的意思不是「私有權限」，而是：

- 幫我建立一個新的 IPC object，不拿既有 key 來共用

### 建立或取得時常搭配 `IPC_CREAT` / `IPC_EXCL`

你可以把它想成類似 `open` 的：

- `O_CREAT`
- `O_EXCL`

例如：

- `IPC_CREAT`：不存在就建
- `IPC_CREAT | IPC_EXCL`：如果已存在就報錯

### 權限資訊存在 `ipc_perm`

XSI IPC 物件有點像「不在一般檔案系統裡的 kernel object」，  
所以它們也會有類似 owner / group / mode 的資訊，放在：

- `struct ipc_perm`

### XSI IPC 物件的生命週期和 file descriptor 不一樣

這類物件很常見的特性是：

- process 結束了，物件不一定自動消失

也就是說，你可能會遇到：

- 程式已經沒在跑
- 但 message queue / semaphore / shared memory 還留在 kernel 裡

所以實務上常搭配：

- `ipcs` 檢查
- `ipcrm` 清理

或在程式內用對應的 control function 做移除。

## 15.7 Message Queues

message queue 和 pipe / FIFO 最大的差別是：

- 它有明確的 message boundary

你送的是一則 message，不是模糊的 byte stream。

### message queue 的 message 有 type

典型 message 結構會長這樣：

```c
struct mymsg {
    long mtype;
    char mtext[128];
};
```

注意：

- 第一個欄位必須是 `long mtype`

這個 type 很重要，因為 receiver 可以選擇：

- 收某一類 message
- 或先收 queue 裡最前面的 message

### 基本 API

常用函式是：

- `msgget`
- `msgsnd`
- `msgrcv`
- `msgctl`

你可以這樣理解：

- `msgget`：取得 queue
- `msgsnd`：送一則 message
- `msgrcv`：收一則 message
- `msgctl`：查狀態、改設定、刪除 queue

### `msgsz` 不包含 `mtype`

這是超常見考點與 bug 點。

`msgsnd` / `msgrcv` 的 size 參數不是整個 struct 大小，而是：

- 不含 `long mtype` 的 payload 大小

也就是通常你會寫：

```c
sizeof(msg.mtext)
```

或：

```c
sizeof(struct mymsg) - sizeof(long)
```

### `msgrcv` 的 type 選擇很有彈性

`msgtyp` 常見語意如下：

- `0`：收 queue 最前面的 message
- `> 0`：收第一個 type 等於它的 message
- `< 0`：收 type 小於等於 `abs(msgtyp)` 的最小 type message

這種「按 type 收訊息」的能力，是 pipe / FIFO 沒有的。

### message queue 保留 message boundary

這代表：

- sender 送的是一筆 request
- receiver 收到的也是一筆 request

你不用像 byte stream 一樣自己在內容中另外定義：

- 哪裡是一筆資料的開頭
- 哪裡是結尾

這讓很多 command-style 協定比較好寫。

### 範例：送一則 message，再收回來

```c
#include <sys/ipc.h>
#include <sys/msg.h>
#include <stdio.h>
#include <string.h>

struct mymsg {
    long mtype;
    char mtext[128];
};

int main(void) {
    key_t key;
    int msqid;
    struct mymsg msg;

    key = ftok("/tmp", 'Q');
    msqid = msgget(key, IPC_CREAT | 0666);
    if (msqid < 0) {
        perror("msgget");
        return 1;
    }

    msg.mtype = 1;
    snprintf(msg.mtext, sizeof(msg.mtext), "hello, message queue");

    if (msgsnd(msqid, &msg, sizeof(msg.mtext), 0) < 0) {
        perror("msgsnd");
        return 1;
    }

    memset(msg.mtext, 0, sizeof(msg.mtext));
    if (msgrcv(msqid, &msg, sizeof(msg.mtext), 1, 0) < 0) {
        perror("msgrcv");
        return 1;
    }

    puts(msg.mtext);
    msgctl(msqid, IPC_RMID, NULL);
    return 0;
}
```

### message queue 的優點與代價

優點：

- 有 message boundary
- kernel 幫你排隊
- 可以依 type 收訊息

代價：

- API 比 pipe 複雜
- 有 kernel queue 大小限制
- 物件生命週期要自己管理

## 15.8 Semaphores

XSI semaphore 是這章最容易讓人「看懂 API，卻沒真的懂語意」的主題。

你要先記住：

- semaphore 主要不是拿來搬資料
- 它是拿來管「誰現在可以做某件事」

### XSI semaphore 的特色是「一組 semaphore」

XSI semaphore 跟很多人直覺不同，不是一次只管理一個 semaphore。  
它的基本單位是：

- semaphore set

也就是：

- 一個 `semid`
- 底下可以有多個 semaphore

### 基本 API

常用的是：

- `semget`
- `semctl`
- `semop`

可以這樣背：

- `semget`：建立或取得 semaphore set
- `semctl`：控制與初始化
- `semop`：做 P / V 類型操作

### `semop` 的威力在於「多個操作原子化」

`semop` 使用 `struct sembuf`：

```c
struct sembuf {
    unsigned short sem_num;
    short sem_op;
    short sem_flg;
};
```

`sem_op` 的語意：

- `> 0`：增加 semaphore 值
- `< 0`：要求資源，必要時等待
- `== 0`：等到 semaphore 值變成 0

其中最值得注意的是：

- 一次可以傳入多個 `sembuf`
- kernel 會把整組操作當成一個 atomic 單位處理

這是 XSI semaphore 很強、但也很重的原因。

### `SEM_UNDO` 是一個很實用但要理解成本的旗標

如果你在操作裡指定：

- `SEM_UNDO`

kernel 會在 process 異常結束時，嘗試把先前對 semaphore 的調整還原。  
這對防止資源永遠被鎖住很有幫助，但也表示 kernel 要替你記帳。

### 初始化是實務上的大坑

當多個 process 可能同時啟動時，常見問題是：

- 某人已經 `semget` 成功拿到舊的 semid
- 但真正的初始值還沒設定完

所以你在設計時要明確決定：

- 誰負責建立
- 誰負責初始化
- 別人什麼時候才能開始使用

這不是 API 本身會自動幫你處理的。

### 範例：單一 semaphore 做簡單互斥

```c
#include <sys/ipc.h>
#include <sys/sem.h>
#include <stdio.h>

union semun {
    int val;
    struct semid_ds *buf;
    unsigned short *array;
};

int main(void) {
    key_t key = ftok("/tmp", 'S');
    int semid = semget(key, 1, IPC_CREAT | 0666);
    union semun arg;
    struct sembuf lock = {0, -1, 0};
    struct sembuf unlock = {0, 1, 0};

    if (semid < 0) {
        perror("semget");
        return 1;
    }

    arg.val = 1;
    if (semctl(semid, 0, SETVAL, arg) < 0) {
        perror("semctl");
        return 1;
    }

    if (semop(semid, &lock, 1) < 0) {
        perror("semop lock");
        return 1;
    }

    puts("critical section");

    if (semop(semid, &unlock, 1) < 0) {
        perror("semop unlock");
        return 1;
    }

    semctl(semid, 0, IPC_RMID);
    return 0;
}
```

## 15.9 Shared Memory

shared memory 通常被認為是：

- 本機 process 間最快的大量資料交換方式之一

原因很直觀：

- 大家 attach 到同一塊 memory
- 後續讀寫直接在那塊共享區域上進行
- 不需要像 pipe / queue 那樣每次經過額外的 message copy 邏輯

### 基本 API

常用的是：

- `shmget`
- `shmat`
- `shmdt`
- `shmctl`

你可以這樣理解：

- `shmget`：建立或取得 shared memory segment
- `shmat`：把它 attach 到本 process address space
- `shmdt`：detach
- `shmctl`：控制與刪除

### shared memory 只幫你共享資料，不幫你同步

這是這一節最重要的一句話。

如果兩個 process 同時改同一份資料：

- kernel 不會自動幫你排隊
- 不會自動幫你加 lock
- 不會自動保證欄位更新次序

所以 shared memory 幾乎總是要搭配：

- semaphore
- file lock
- 或其他同步手段

### 不要把 process-local pointer 直接塞進 shared memory

這是 shared memory 的經典陷阱。

因為不同 process attach 同一個 segment 時：

- 虛擬位址不保證一樣

所以如果你在 shared memory 裡放：

- 真正的 pointer value

別的 process 拿來解參考時，常常根本不是正確位址。

比較安全的做法是放：

- offset
- index
- 長度
- 固定大小陣列

### 刪除 shared memory 的語意

shared memory 很像其他 XSI IPC 一樣，不會因為某個 process 離開就自動消失。  
通常你會透過：

- `shmctl(shmid, IPC_RMID, NULL)`

把它標記為刪除。  
真正回收時間則和：

- 是否還有 process attached

有關。

### 範例：建立 shared memory 並寫入字串

```c
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdio.h>
#include <string.h>

struct shared_block {
    size_t len;
    char data[256];
};

int main(void) {
    key_t key = ftok("/tmp", 'M');
    int shmid;
    struct shared_block *p;

    shmid = shmget(key, sizeof(struct shared_block), IPC_CREAT | 0666);
    if (shmid < 0) {
        perror("shmget");
        return 1;
    }

    p = shmat(shmid, NULL, 0);
    if (p == (void *)-1) {
        perror("shmat");
        return 1;
    }

    snprintf(p->data, sizeof(p->data), "shared hello");
    p->len = strlen(p->data);

    printf("stored %zu bytes: %s\n", p->len, p->data);

    shmdt(p);
    shmctl(shmid, IPC_RMID, NULL);
    return 0;
}
```

這個例子只有示範 attach / detach。  
如果真的要讓多個 process 同時安全讀寫，請一定再加同步機制。

## 15.10 POSIX Semaphores

APUE 在這裡把 POSIX semaphore 拉進來，是因為它們在很多情境下比 XSI semaphore 更直覺。

### POSIX semaphore 分成 named 與 unnamed

named semaphore：

- 用名字在 system 中辨識
- 主要 API：`sem_open` / `sem_close` / `sem_unlink`

unnamed semaphore：

- 直接是一個 `sem_t` 物件
- 主要 API：`sem_init` / `sem_destroy`

共同的操作函式：

- `sem_wait`
- `sem_trywait`
- `sem_timedwait`
- `sem_post`

### named semaphore 比較像「有名字的同步物件」

```c
#include <semaphore.h>

sem_t *sem = sem_open("/apue_sem", O_CREAT, 0666, 1);
```

它的使用感受有點像：

- 透過一個名字找到 kernel 裡的 semaphore

當你用完後要記得：

- `sem_close`
- 必要時 `sem_unlink`

### unnamed semaphore 比較像把同步工具直接放進記憶體

如果是 thread 之間共用，`pshared` 通常設成 `0`。  
如果是 process 之間共用，則：

- `pshared` 要非 0
- 而且這個 `sem_t` 必須放在共享記憶體裡

這個組合很常見：

- shared memory 放資料
- unnamed POSIX semaphore 放同步控制

### POSIX semaphore 常比 XSI semaphore 更簡單

XSI semaphore 強在：

- 可以一次操作一整組
- 有 wait-for-zero 等較底層能力

POSIX semaphore 強在：

- API 比較小
- 單一 semaphore 使用直覺
- 寫一般同步問題時更順手

### 範例：named semaphore 做簡單同步

```c
#include <fcntl.h>
#include <semaphore.h>
#include <stdio.h>

int main(void) {
    sem_t *sem = sem_open("/apue_demo_sem", O_CREAT, 0666, 1);
    if (sem == SEM_FAILED) {
        perror("sem_open");
        return 1;
    }

    sem_wait(sem);
    puts("inside critical section");
    sem_post(sem);

    sem_close(sem);
    sem_unlink("/apue_demo_sem");
    return 0;
}
```

## 15.11 Client-Server Properties

這一節很重要，因為它不是再教新 API，而是在幫你建立：

- 什麼情境該選哪種 IPC

你可以用下面這張表快速整理。

| 機制 | 命名方式 | 資料單位 | 是否要求親緣關係 | 適合什麼 | 主要缺點 |
| --- | --- | --- | --- | --- | --- |
| `pipe` | 靠繼承到的 fd | byte stream | 通常要有共同祖先 | parent-child 單向資料流 | 無 message boundary，無法自然服務不相關 process |
| `FIFO` | pathname | byte stream | 不需要 | 簡單本機 rendezvous | open 語意麻煩，多 client reply 設計不自然 |
| message queue | XSI key | message | 不需要 | typed request / reply | API 較重，生命週期管理麻煩 |
| shared memory | XSI key | 共享位元組區 | 不需要 | 大量共享資料、最高吞吐 | 不自帶同步，設計不當容易 race |
| semaphore | XSI key 或 POSIX name/object | 同步原語，不是資料通道 | 不需要 | 控制臨界區、資源計數 | 本身不搬資料，要和別的機制配合 |

把這張表讀懂後，你應該會發現：

- 根本沒有「最強的 IPC」
- 只有「哪一種語意最適合你現在的 communication pattern」

### client / server 最常見的幾個判斷點

如果你在設計本機 client / server，可以先問：

- request 是一行一行的，還是大塊資料？
- 要不要 message boundary？
- client 數量多不多？
- 回應是單向還是 request / reply？
- 是否需要共享大資料結構？
- 同步問題要不要和資料通道分開？

通常可以得到這些直覺：

- 簡單線性資料流：`pipe`
- 不相關 process 的簡單入口：`FIFO`
- 明確一筆一筆 request：message queue
- 大量共享狀態：shared memory + semaphore

如果你再多一個需求：

- 要跨主機

那下一章的 `socket` 就會變成主角。

## 15.12 Summary

這章其實是在教你：

- process 之間除了 `signal` 以外，還能如何「真正合作」

它把經典 IPC 分成幾種很不同的思路：

- stream 型：`pipe` / `FIFO`
- message 型：message queue
- shared-state 型：shared memory
- synchronization 型：semaphore

如果你用一句話總結這章：

- IPC 的關鍵不在於能不能傳資料，而在於資料語意、同步需求、命名方式與生命週期是否符合你的系統設計

### 這章讀完你應該真的會的事

- 知道 pipe / FIFO 差別在哪
- 知道為什麼 pipe 用完一定要關掉不用的端點
- 知道 `popen` 只是高階包裝，不是魔法
- 知道 shared memory 快在哪，也知道危險在哪
- 知道 semaphore 是同步工具，不是資料傳輸工具
- 能根據 communication pattern 選合理的 IPC 機制

### 這章最容易踩坑的地方

- 忘記關 pipe 的寫端，導致讀端永遠等不到 EOF
- 把 pipe / FIFO 當成有 message boundary
- 用 `popen` 卻忘記 shell 會介入
- 雙向 coprocess 沒處理 buffering，結果 deadlock
- XSI IPC 建了不刪，kernel 裡留下垃圾物件
- 在 shared memory 裡存 raw pointer
- 以為 semaphore 可以順便搬資料

### 建議你現在立刻動手做

1. 自己寫一個 parent / child pipe 範例，刻意不關 write end，觀察 child 讀 EOF 的差異。
2. 用 `mkfifo` 做一個簡單本機聊天室入口，感受 FIFO 的 blocking open 行為。
3. 用 shared memory 存一個 struct，再用 semaphore 保護它，體會「共享資料」和「同步控制」是兩件事。

### 一句總結

這章是在教你：process 要合作時，不是只問「怎麼傳」，而是要同時想清楚「怎麼碰頭、怎麼同步、資料邊界在哪裡、物件多久消失」。
