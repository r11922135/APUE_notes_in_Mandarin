# Chapter 18 - Terminal I/O

## 這章在做什麼

很多人第一次接觸 terminal programming 時，直覺都會是：

- terminal 不就是一個可以 `read/write` 的裝置嗎？

但 UNIX 的 terminal 完全不是這麼單純。

它背後有一整套：

- line discipline
- special input characters
- canonical mode
- noncanonical mode
- `termios` option flags
- window size

也就是說，當你對 terminal 做 I/O 時，kernel 不只是幫你搬 byte，還會：

- 幫你做行編輯
- 幫你 echo
- 幫你解讀 `Ctrl-C`
- 幫你判斷 EOF
- 幫你做 start/stop flow control

這章的任務，就是把這個黑盒子拆開。

## 本章小節地圖

- `18.1 Introduction`
- `18.2 Overview`
- `18.3 Special Input Characters`
- `18.4 Getting and Setting Terminal Attributes`
- `18.5 Terminal Option Flags`
- `18.6 stty Command`
- `18.7 Baud Rate Functions`
- `18.8 Line Control Functions`
- `18.9 Terminal Identification`
- `18.10 Canonical Mode`
- `18.11 Noncanonical Mode`
- `18.12 Terminal Window Size`
- `18.13 termcap, terminfo, and curses`
- `18.14 Summary`

## 先抓住這章最重要的心智模型

### terminal 是 stateful device，不是普通檔案

regular file 的典型心智模型是：

- 你寫什麼，就存什麼
- 你讀什麼，就拿什麼

terminal 不是。

terminal 的真正心智模型比較像：

- 你在跟一個有狀態的 driver 對話
- 這個 driver 會根據目前模式解讀你的輸入

所以同樣按下 `Ctrl-C`：

- 在某些模式下它會變成 `SIGINT`
- 在另一些模式下它只是 byte 值 `0x03`

### 很多「你以為是程式在做的事」，其實是 terminal driver 在做

例如：

- 按 backspace 會刪字
- `Ctrl-U` 會清掉整行
- `Ctrl-D` 會造成 EOF
- `Ctrl-S` / `Ctrl-Q` 會停 / 續 output

這些很多都不是應用程式自己實作的，而是：

- terminal line discipline

### 這章最重要的分野：canonical vs noncanonical

你可以先用一句白話來記：

- canonical mode：kernel 幫你把輸入當成「行」來處理
- noncanonical mode：kernel 不再幫你湊成一行，read 行為改由 `VMIN/VTIME` 控制

這兩種模式，會直接決定：

- `read` 何時回來
- special character 怎麼解讀
- 互動式程式該怎麼寫

## 18.1 Introduction

APUE 一開始就先說：

- terminal I/O 是一個很 messy 的領域

這不是誇張，而是歷史真的很亂。

### 歷史背景：System V 和 BSD 曾經各走各的 terminal 介面

在 POSIX 統一之前：

- System III / System V 走一套
- Version 7 / BSD 又走另一套

後來 POSIX 把這些接口整理成今天的：

- `termios`

所以這章很多內容看起來龐雜，背後原因其實是：

- 它承接了很長的 UNIX terminal 歷史

### 為什麼 terminal I/O 會這麼複雜

因為 terminal I/O 不只用在：

- 人打字的終端機

也用在：

- serial line
- modem
- printer
- 各種字元裝置

所以你會看到一些旗標很像：

- 人類互動相關

另一些又很像：

- serial device / line control 相關

## 18.2 Overview

### terminal input 有兩種大模式

APUE 先把 terminal input 分成兩種：

1. canonical mode input processing
2. noncanonical mode input processing

如果你什麼都不做，預設通常是：

- canonical mode

### canonical mode：以「行」為單位

在 canonical mode 下：

- 輸入會被湊成一行
- `read` 最多回傳一行

這很適合：

- shell
- 一般 command-line 輸入

### noncanonical mode：不再幫你湊成行

這種模式適合：

- `vi`
- full-screen editor
- 單鍵互動程式
- game
- terminal UI

因為這些程式常常需要：

- 看到單一按鍵就立刻反應

### input queue 與 output queue

APUE 用一張圖幫你建立 terminal 的邏輯模型：

- terminal 有 input queue
- terminal 有 output queue

幾個重點：

- 如果 echo 開著，輸入和輸出之間會有隱含連動
- input queue 大小有限，滿了行為依系統而異
- output queue 滿了時，寫入 process 通常會被 block
- `tcflush` 可以清 queue
- `tcsetattr` 可以選在 output drain 後再改設定

### line discipline 是關鍵中的關鍵

大多數 canonical 處理，是放在：

- terminal line discipline

你可以把它想成：

- generic `read/write` 和實際 terminal device driver 之間的一層

這層在做的事包含：

- special character 處理
- canonical line assembly
- echo
- signal generation

這個心智模型到 Chapter 19 講 `pty` 時還會再出現。

### `struct termios`

terminal 的幾乎所有設定都收在：

```c
struct termios {
    tcflag_t c_iflag;
    tcflag_t c_oflag;
    tcflag_t c_cflag;
    tcflag_t c_lflag;
    cc_t c_cc[NCCS];
};
```

你可以粗略理解成：

- `c_iflag`：input processing
- `c_oflag`：output processing
- `c_cflag`：裝置 / line control
- `c_lflag`：互動語意
- `c_cc[]`：special characters

## 18.3 Special Input Characters

這節是在教你：

- 哪些按鍵不是單純資料，而是被 terminal driver 特別解讀的控制字元

POSIX 定義了 11 個主要 special input characters，另外各平台可能還有延伸。

### 最值得先記住的幾組

#### signal 類

- `VINTR`：通常是 `Ctrl-C`，產生 `SIGINT`
- `VQUIT`：通常是 `Ctrl-\`，產生 `SIGQUIT`
- `VSUSP`：通常是 `Ctrl-Z`，產生 `SIGTSTP`

#### 行編輯 / 輸入控制類

- `VERASE`：通常是 backspace，刪前一字
- `VKILL`：通常是 `Ctrl-U`，刪整行
- `VEOF`：通常是 `Ctrl-D`，觸發 EOF 語意
- `VWERASE`：通常是 `Ctrl-W`，刪前一個 word
- `VREPRINT`：通常是 `Ctrl-R`，重印未讀輸入
- `VLNEXT`：通常是 `Ctrl-V`，下一個字元取消特殊意義

#### flow control 類

- `VSTOP`：通常是 `Ctrl-S`，停止 output
- `VSTART`：通常是 `Ctrl-Q`，恢復 output

### `_POSIX_VDISABLE`

POSIX 允許你把某個 special character 關掉。  
作法是把對應的 `c_cc[]` 元素設成：

- `_POSIX_VDISABLE`

實務上常用：

- `fpathconf(fd, _PC_VDISABLE)`

去查當前系統使用哪個值。

### 範例：關掉 interrupt key，改 EOF key

```c
#include <termios.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    struct termios term;
    long vdisable;

    if (!isatty(STDIN_FILENO)) {
        return 1;
    }

    vdisable = fpathconf(STDIN_FILENO, _PC_VDISABLE);
    if (vdisable < 0) {
        return 1;
    }

    if (tcgetattr(STDIN_FILENO, &term) < 0) {
        perror("tcgetattr");
        return 1;
    }

    term.c_cc[VINTR] = (cc_t)vdisable;
    term.c_cc[VEOF] = 2; /* Control-B */

    if (tcsetattr(STDIN_FILENO, TCSAFLUSH, &term) < 0) {
        perror("tcsetattr");
        return 1;
    }

    return 0;
}
```

這裡最重要的觀念是：

- 關掉 `VINTR` 只是關掉「那個按鍵觸發 signal 的能力」
- 不是讓程式從此收不到 `SIGINT`

你仍然可以用 `kill` 送 signal。

### `VEOF` 常被誤會

很多人會以為：

- `Ctrl-D` 就是送出一個真正的 EOF 字元

其實更準確的說法是：

- terminal driver 看到 `VEOF` 後，決定讓 `read` 呈現 EOF 語意

如果目前沒有任何待讀資料：

- `read` 會回 `0`

如果已經有一批資料在 queue 裡：

- 那批資料會先被送給程式

### `VLNEXT` 很實用

如果某個字元平常有特殊意義，例如：

- `Ctrl-C`

那你想把它「當普通資料」送給程式時，可以先打：

- `VLNEXT`，常見是 `Ctrl-V`

這個功能對 terminal 測試很有用。

### BREAK 不是普通字元

APUE 也提醒一件常被忽略的事：

- `BREAK` 不是一般 input character

它其實是 serial transmission 上的一種條件：

- 一串持續足夠久的 0 bit

它的處理和 `BRKINT`、`IGNBRK` 等旗標有關。

## 18.4 Getting and Setting Terminal Attributes

### 最核心的兩個函式

```c
int tcgetattr(int fd, struct termios *termptr);
int tcsetattr(int fd, int opt, const struct termios *termptr);
```

你可以把它想成：

- `tcgetattr`：把 terminal 當前設定抓出來
- `tcsetattr`：把你修改後的設定套回去

這兩個函式幾乎是整章的主軸。

### 千萬不要憑空造一份 `termios`

正確用法通常是：

1. `tcgetattr`
2. 修改少數你要改的 bit
3. `tcsetattr`

原因是：

- `termios` 裡很多欄位和系統 / 裝置初始狀態有關
- 你只想改自己真的知道的部分

### `tcsetattr` 的三種時機選項

最重要的 `opt` 有三個：

- `TCSANOW`：立即生效
- `TCSADRAIN`：等 output 全送完再生效
- `TCSAFLUSH`：等 output 送完，並丟棄未讀 input

你可以這樣記：

- 改輸出格式時，常偏向 `TCSADRAIN`
- 改輸入模式時，常偏向 `TCSAFLUSH`

因為你通常不想讓舊模式下累積的 input，在新模式下被誤解讀。

### `tcsetattr` 可能有 partial success

APUE 特別提醒：

- `tcsetattr` 回 `0`，不保證每個你要求的 bit 都真的套上了

所以在一些關鍵模式切換 helper 中，會做法比較嚴格：

1. `tcsetattr`
2. 再 `tcgetattr`
3. 驗證設定是否真的生效

這點很有系統程式風格。

### 範例：安全地關閉 `ECHO` 和 `ICANON`

```c
#include <termios.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    struct termios oldt, newt;

    if (tcgetattr(STDIN_FILENO, &oldt) < 0) {
        perror("tcgetattr");
        return 1;
    }

    newt = oldt;
    newt.c_lflag &= ~(ECHO | ICANON);
    newt.c_cc[VMIN] = 1;
    newt.c_cc[VTIME] = 0;

    if (tcsetattr(STDIN_FILENO, TCSAFLUSH, &newt) < 0) {
        perror("tcsetattr");
        return 1;
    }

    /* ... do interactive work ... */

    tcsetattr(STDIN_FILENO, TCSAFLUSH, &oldt);
    return 0;
}
```

這段程式碼最重要的不是 bit 操作，而是：

- 先保存舊狀態
- 事情做完後恢復

## 18.5 Terminal Option Flags

這一節如果硬背全部旗標，通常會很痛苦。  
比較好的讀法是先按功能分群。

### `c_lflag`：最常影響人類互動體驗

這一組通常是你最常碰的：

- `ECHO`：輸入要不要回顯
- `ICANON`：是否 canonical mode
- `ISIG`：是否啟用 terminal-generated signals
- `IEXTEN`：是否啟用延伸特殊字元處理
- `NOFLSH`：signal 產生後不要自動 flush queue
- `TOSTOP`：背景 process 寫 controlling terminal 時送 `SIGTTOU`

如果你只先記住一組，優先記這組。

### `c_iflag`：輸入預處理

這一組控制輸入如何被 driver 解讀：

- `ICRNL`：把 CR 映成 NL
- `IGNCR`：忽略 CR
- `INLCR`：把 NL 映成 CR
- `IXON`：啟用 `Ctrl-S` / `Ctrl-Q` output flow control
- `IXOFF`：啟用 input flow control
- `BRKINT`：BREAK 產生 `SIGINT`
- `ISTRIP`：去掉第八 bit
- `INPCK`：輸入 parity checking

### `c_oflag`：輸出後處理

這組最值得先記的是：

- `OPOST`：啟用 output processing
- `ONLCR`：把 NL 輸出成 CR-NL

很多 terminal 程式在 raw mode 會把 `OPOST` 關掉。  
這也是為什麼你有時會看到：

- 輸出看起來「沒有正常換行」

### `c_cflag`：比較偏裝置 / serial line

常見的有：

- `CS8`：8 bits/char
- `PARENB`：啟用 parity
- `PARODD`：odd parity
- `CSTOPB`：stop bit 設定
- `CREAD`：enable receiver
- `CLOCAL`：忽略 modem status lines

如果你不是在寫 serial device 程式，這組平常碰得較少。  
但在 raw mode helper 裡，`CS8` 很常出現。

### 不要追求一次背完 70 個旗標

APUE 書裡列了非常多平台差異旗標。  
真正要先掌握的是：

- 這四大群在做什麼
- 幾個最常用的 bit 影響哪種行為

### 幾個超常見組合

#### 想做「單鍵即時輸入」

通常你至少會碰到：

- 關 `ICANON`
- 關 `ECHO`
- 設 `VMIN` / `VTIME`

#### 想做「更接近 raw mode」

通常還會再碰到：

- 關 `ISIG`
- 關 `IEXTEN`
- 關 `ICRNL`
- 關 `IXON`
- 關 `OPOST`
- 設 `CS8`

## 18.6 `stty` Command

`stty` 很值得懂，因為它是：

- `termios` 的 command-line 視窗

### `stty -a`

這個指令會列出目前 terminal 設定，例如：

- speed
- rows / columns
- local flags
- input flags
- output flags
- control flags
- special characters

所以當你在讀書上某個 bit 名稱時，最好的做法常常是：

- 直接開一個 terminal 跑 `stty -a`

這樣抽象概念會立刻落地。

### 為什麼 `stty` 對學習很有幫助

因為你能立刻把：

- `ICANON`
- `ECHO`
- `ixon`
- `erase`
- `intr`

和你眼前的 terminal 行為對起來。

## 18.7 Baud Rate Functions

這節比較偏 serial line，但觀念還是值得知道。

### API

```c
speed_t cfgetispeed(const struct termios *termptr);
speed_t cfgetospeed(const struct termios *termptr);
int cfsetispeed(struct termios *termptr, speed_t speed);
int cfsetospeed(struct termios *termptr, speed_t speed);
```

### 核心觀念

這些函式只是：

- 在 `termios` 結構裡讀寫 baud rate 欄位

真正要讓裝置生效，還是要：

- `tcsetattr`

### `B0` 的語意

`B0` 很特別，它通常代表：

- hang up

如果被設成 output baud rate，modem control lines 可能被撤除。

### 實務上你要記什麼

大多數一般 terminal app 不會常改 baud rate，  
但如果你接 serial device，這組函式就很重要。

## 18.8 Line Control Functions

這節的四個函式都很實用：

```c
int tcdrain(int fd);
int tcflow(int fd, int action);
int tcflush(int fd, int queue);
int tcsendbreak(int fd, int duration);
```

### `tcdrain`

意思是：

- 等目前 output queue 全部送完

很適合在你要切換某些輸出屬性前使用。

### `tcflow`

控制 input / output flow。

常見 action：

- `TCOOFF`：暫停 output
- `TCOON`：恢復 output
- `TCIOFF`：送 STOP 字元，要求對方暫停輸入
- `TCION`：送 START 字元，要求對方恢復輸入

### `tcflush`

把 queue 裡尚未處理的資料丟掉。

常見選項：

- `TCIFLUSH`：丟 input queue
- `TCOFLUSH`：丟 output queue
- `TCIOFLUSH`：都丟

這在切模式時非常常見。

### `tcsendbreak`

送一段 BREAK condition。  
它比較偏 serial line / terminal control，平常一般 shell app 不常直接碰到。

## 18.9 Terminal Identification

這節是在回答：

- 我現在是不是在連 terminal？
- 這個 terminal 叫什麼？
- controlling terminal 的名字是什麼？

### `ctermid`

```c
char *ctermid(char *ptr);
```

它回傳的是：

- controlling terminal 的名字

在很多 UNIX 系統上通常是：

- `/dev/tty`

### `isatty`

```c
int isatty(int fd);
```

這個超常用。  
它讓你知道：

- 某個 fd 是不是 terminal device

很多程式會用它判斷：

- 目前是在互動式 terminal 裡
- 還是被 pipe / redirect 了

### `ttyname`

```c
char *ttyname(int fd);
```

它讓你知道：

- 這個 terminal 對應的裝置名稱

例如可能是：

- `/dev/ttys003`
- `/dev/pts/2`

### 範例：辨識 stdin/stdout/stderr 是不是 tty

```c
#include <unistd.h>
#include <stdio.h>

int main(void) {
    printf("fd 0: %s\n", isatty(0) ? "tty" : "not a tty");
    printf("fd 1: %s\n", isatty(1) ? "tty" : "not a tty");
    printf("fd 2: %s\n", isatty(2) ? "tty" : "not a tty");
    return 0;
}
```

## 18.10 Canonical Mode

### canonical mode 的核心：以行為單位

在 canonical mode 下：

- terminal driver 會幫你做 line editing
- `read` 最多回傳一整行
- special character 如 `ERASE`、`KILL`、`EOF` 會被特別處理

這是一般 shell 預設的工作方式。

### 為什麼這模式很方便

因為大多數 command-line 工具只想要：

- 使用者打一整行
- 按 Enter
- 程式再處理

這樣程式本身就不用自己處理：

- backspace
- line erase
- EOF key

### canonical mode 下 `read` 的感覺

如果使用者還沒輸入一個完整 line delimiter，例如：

- newline
- EOL
- EOL2
- 或 `VEOF`

那 `read` 往往就還不回來。

這就是為什麼 canonical mode 很「像行緩衝」。

### `VEOF` 在 canonical mode 特別重要

如果使用者在行首輸入 `VEOF`：

- `read` 可能直接回 `0`

這就是很多互動式程式感受到 EOF 的方式。

## 18.11 Noncanonical Mode

這一節是整章另一個大重點。

### noncanonical mode 不是單一模式，而是一整組行為組合

當你關掉：

- `ICANON`

代表的是：

- terminal driver 不再幫你把輸入湊成一整行

但 `read` 什麼時候回來，還要看：

- `VMIN`
- `VTIME`

### `VMIN` / `VTIME` 是 noncanonical mode 的靈魂

你可以把它理解成：

- `VMIN`：至少要幾個 byte 才肯回
- `VTIME`：要不要加 timeout

但細節比這再微妙一點。

### Case 1：`VMIN > 0`，`VTIME == 0`

這是最直觀的一種：

- 一定要等到至少 `VMIN` 個 byte
- 沒有 timer

也就是：

- 不夠就一直等

### Case 2：`VMIN == 0`，`VTIME > 0`

這時：

- timer 會立刻開始
- 只要有至少 1 byte 到，就回來
- 如果 timeout 前都沒資料，`read` 回 `0`

這很像：

- 有 timeout 的「等一點看看」

### Case 3：`VMIN > 0`，`VTIME > 0`

這是最容易搞混的模式。

它不是「總等待時間」，而比較像：

- interbyte timer

典型理解是：

- 先等第一個 byte
- 第一個 byte 到了後才開始計時
- 若收滿 `VMIN` 個 byte，就回來
- 否則在 interbyte timeout 到時回來

### Case 4：`VMIN == 0`，`VTIME == 0`

這是最像 polling 的模式：

- 有資料就回資料
- 沒資料就立刻回

### `cbreak` 和 `raw` 不是完全一樣

APUE 用 helper 函式展示兩種經典模式：

- `tty_cbreak`
- `tty_raw`

#### cbreak 的精神

- 關 `ICANON`
- 關 `ECHO`
- 保留 signal characters
- 通常 `VMIN=1`、`VTIME=0`

也就是：

- 單字元輸入
- 但 `Ctrl-C` 之類還是有效

#### raw 的精神

raw 更徹底，除了關 canonical / echo 外，通常還會：

- 關 `ISIG`
- 關 `IEXTEN`
- 關 `ICRNL`
- 關 `INPCK`
- 關 `ISTRIP`
- 關 `IXON`
- 關 `OPOST`
- 設 `CS8`

這時輸入 / 輸出更接近：

- 不加工的 byte stream

### 範例：做一個最小 raw-mode 讀字元程式

```c
#include <termios.h>
#include <unistd.h>
#include <stdio.h>

int main(void) {
    struct termios oldt, raw;
    char c;

    if (tcgetattr(STDIN_FILENO, &oldt) < 0) {
        perror("tcgetattr");
        return 1;
    }

    raw = oldt;
    raw.c_lflag &= ~(ECHO | ICANON | IEXTEN | ISIG);
    raw.c_iflag &= ~(BRKINT | ICRNL | INPCK | ISTRIP | IXON);
    raw.c_cflag &= ~(CSIZE | PARENB);
    raw.c_cflag |= CS8;
    raw.c_oflag &= ~(OPOST);
    raw.c_cc[VMIN] = 1;
    raw.c_cc[VTIME] = 0;

    if (tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw) < 0) {
        perror("tcsetattr");
        return 1;
    }

    while (read(STDIN_FILENO, &c, 1) == 1) {
        if ((unsigned char)c == 127) {
            break;
        }
        printf("%03o\n", (unsigned char)c);
    }

    tcsetattr(STDIN_FILENO, TCSAFLUSH, &oldt);
    return 0;
}
```

### 改 terminal mode 的程式一定要小心恢復

APUE 非常重視這件事。  
因為如果你把 terminal 留在 raw / cbreak mode 就結束：

- 使用者的 shell 會看起來壞掉

因此正確作法通常是：

- 保存舊設定
- 正常結束時恢復
- 用 signal handler / `atexit` 盡量補救

## 18.12 Terminal Window Size

### kernel 會記住 terminal 的視窗大小

很多系統會為每個 terminal / pty 維護：

```c
struct winsize {
    unsigned short ws_row;
    unsigned short ws_col;
    unsigned short ws_xpixel;
    unsigned short ws_ypixel;
};
```

最常用的是：

- row
- col

### 取得與設定

常見 `ioctl`：

- `TIOCGWINSZ`：get window size
- `TIOCSWINSZ`：set window size

### `SIGWINCH`

如果視窗大小改變，kernel 會送：

- `SIGWINCH`

給 foreground process group。

這個 signal 預設通常是：

- ignore

所以應用程式若要在 resize 時重畫畫面，必須自己捕捉它。

### 範例：印出 terminal window size

```c
#include <sys/ioctl.h>
#include <signal.h>
#include <unistd.h>
#include <stdio.h>

static void show_size(int fd) {
    struct winsize ws;
    if (ioctl(fd, TIOCGWINSZ, &ws) == 0) {
        printf("%d rows, %d columns\n", ws.ws_row, ws.ws_col);
    }
}

static void on_winch(int signo) {
    (void)signo;
    puts("SIGWINCH received");
    show_size(STDIN_FILENO);
}

int main(void) {
    signal(SIGWINCH, on_winch);
    show_size(STDIN_FILENO);
    for (;;) {
        pause();
    }
}
```

## 18.13 `termcap`, `terminfo`, and `curses`

這節是在回答一個不同層次的問題：

- terminal control 和 terminal capability database 有什麼關係？

### `termcap` / `terminfo` 解的是「不同 terminal 型號如何操作」

它們不是在管：

- canonical mode
- `VINTR`
- `VMIN`
- `tcsetattr`

它們主要是在管：

- 清畫面要送什麼 escape sequence
- 移動 cursor 要送什麼字串
- 某 terminal 支不支援某個功能

### `termcap`

較早期的做法，核心是一份文字資料庫：

- `/etc/termcap`

### `terminfo`

後來發展出的改良版，特色是：

- terminal description 通常是編譯過的
- 查找更快

### `curses`

`curses` 建立在這些 terminal capability 資訊之上，提供更高階的：

- screen handling
- window-like 操作
- cursor movement
- raw/cbreak/echo 開關的實用封裝

### 這節最重要的一句話

`termcap` / `terminfo` / `curses` 並不能取代 `termios`。

比較準確的關係是：

- `termios`：處理 terminal driver 模式與控制字元
- `termcap` / `terminfo`：描述 terminal 能力
- `curses`：建立在能力資料與 terminal 控制上的高階 UI 函式庫

## 18.14 Summary

這章最大的收穫應該是：

- 你不再把 terminal 當成普通 file descriptor

而是知道它背後有一套：

- line discipline
- mode
- special characters
- queue
- signal

的完整系統行為。

真正重要的主線有三條：

- 用 `termios` 檢查 / 修改 terminal 屬性
- 理解 canonical 與 noncanonical mode 的差異
- 知道 `VMIN/VTIME`、`SIGWINCH`、`isatty/ttyname` 這些常用工具在解什麼問題

### 這章讀完你應該真的會的事

- 知道為什麼 `Ctrl-C`、backspace、`Ctrl-D` 常常不是程式自己處理的
- 看得懂 `termios` 四大旗標群各自在管什麼
- 能正確使用 `tcgetattr` / `tcsetattr`
- 能分清 `TCSANOW`、`TCSADRAIN`、`TCSAFLUSH`
- 能解釋 canonical mode 與 noncanonical mode 的差異
- 能說清楚 `VMIN/VTIME` 四種組合的大意
- 知道如何取得 terminal 名稱與 window size

### 這章最容易踩坑的地方

- 直接亂造 `termios`，而不是先 `tcgetattr`
- 改完 terminal mode 沒恢復，害使用者 shell 壞掉
- 把 `Ctrl-D` 誤以為是普通字元 EOF
- 不理解 `VMIN/VTIME`，導致 `read` 行為完全出乎預期
- 把 `termcap/terminfo` 和 `termios` 混成同一層東西

### 建議你現在立刻動手做

1. 用 `stty -a` 對照書上的 `ECHO`、`ICANON`、`ISIG`、`VINTR`、`VERASE`。
2. 自己寫一個小程式，在 raw mode 下逐字讀鍵盤，觀察 `Ctrl-C` 和 backspace 變成什麼 byte。
3. 再寫一個 `SIGWINCH` 範例，拖拉 terminal 視窗，感受 window-size 機制真的存在。

### 一句總結

這章是在教你：terminal I/O 的關鍵不是 `read/write` 本身，而是 terminal driver 會在中間替你做多少事，以及你能不能精準地控制這些行為。
