# Chapter 21 - Communicating with a Network Printer

## 這章在做什麼

這章表面上是在教你：

- 怎麼和 network printer 溝通

但它真正的價值是：

- 用一個完整、跨多個系統層的實戰案例，把前面學的 thread、socket、daemon、file I/O、directory handling、signals 全串起來

APUE 在這章要做兩個程式：

- `print`：把列印工作送給 spooler daemon 的 command
- `printd`：真正管理 queue 並和 printer 溝通的 daemon

所以這章其實是一個很完整的：

- small network service + background worker + queueing system

案例。

## 本章小節地圖

- `21.1 Introduction`
- `21.2 The Internet Printing Protocol`
- `21.3 The Hypertext Transfer Protocol`
- `21.4 Printer Spooling`
- `21.5 Source Code`
- `21.6 Summary`

## 先抓住這章最重要的心智模型

### 這章不是在教「印表機 API」，而是在教「分層通訊系統」

整個列印流程其實分成好幾層：

- client `print`
- local spooler daemon `printd`
- network printer
- `IPP`
- `HTTP`
- `TCP/IP`

也就是說，這章真正想讓你練的是：

- 如何把應用層協定、daemon、queue 與網路 I/O 接成一條完整資料流

### spooler 的本質是「把使用者提交」和「真正輸出到裝置」解耦

如果沒有 spooler，大家都直接對 printer 打：

- 容易互相搶資源
- printer 端協定處理分散
- 使用者程式必須自己等待與處理錯誤

有了 spooler 之後：

- user request 先被收下
- 存到本地 queue
- 再由 daemon 串行地送去 printer

這就是典型的：

- front-end submit
- back-end worker

架構。

### protocol parsing 只是其中一半，另一半是系統工程

你不能只看到：

- `IPP over HTTP`

就以為這章只是網路封包格式。  
它同時也在處理：

- thread 分工
- queue synchronization
- file spooling
- config reload
- signal handling
- least privilege

所以這章其實很像小型 production service 的縮影。

## 21.1 Introduction

APUE 先設定場景：

- 現代 network printer 通常透過 Ethernet 連到多台電腦
- 常見支援 PostScript，也常能處理 plain text

而應用程式與這些 printer 溝通，通常會用：

- `IPP`，`Internet Printing Protocol`

接著書裡說明它要實作的兩個元件：

- print spooler daemon
- submit print job 的 command

這樣的設計剛好能把很多前面章節的主題串起來。

## 21.2 The Internet Printing Protocol

### `IPP` 是什麼

`IPP` 是為了 network-based printing system 定義的協定。  
它的角色可以簡單理解成：

- 「如何用標準網路協定和 printer 對話」

### `IPP` 建在 `HTTP` 之上，而 `HTTP` 又建在 `TCP/IP` 之上

這是一定要先記住的分層：

- Ethernet
- IP
- TCP
- HTTP
- IPP
- document data

這也是為什麼這章不只是 printer 章，而是 network protocol 章。

### `IPP` 是 request-response protocol

client 發 request：

- 例如 print-job

printer 回 response：

- 告訴你成功或失敗

典型 operation 包括：

- submit print job
- cancel job
- get job attributes
- get printer attributes
- hold / release job
- pause / resume printer

本章主要用的是：

- `print-job`

### `IPP` header 的基本結構

核心欄位包括：

- version number
- operation ID 或 status code
- request ID
- attributes
- end-of-attributes tag
- optional document data

其中：

- request 裡第二欄是 operation ID
- response 裡第二欄是 status code

### binary encoding 與 network byte order

這是 `IPP` 一個比較煩的地方。

- 某些欄位是 binary integer
- 而且採 network byte order

所以像：

- request ID
- status code

這些欄位都要注意：

- `hton*`
- `ntoh*`

這跟純文字協定相比麻煩不少。

### attributes 是分群編碼的

每個 attribute group 先有：

- 1-byte group tag

每個 attribute 本身通常包含：

- 1-byte value tag
- 2-byte name length
- attribute name
- 2-byte value length
- value

所以你在 parse / build 時都必須非常注意：

- 長度
- 對齊
- byte order

### `print-job` request 的重要 attributes

APUE 特別列了幾個最關鍵的：

- `attributes-charset`
- `attributes-natural-language`
- `printer-uri`

這三個可以直接視為基本必備。

其他常見 optional attributes：

- `requesting-user-name`
- `job-name`
- `document-name`
- `document-format`
- `document-natural-language`
- `compression`

### `document-format` 為什麼重要

因為 printer 要知道你送的是：

- `text/plain`
- 還是 `application/postscript`

如果格式講錯，可能會：

- 列印結果不對
- printer 用錯解析模式

## 21.3 The Hypertext Transfer Protocol

### 為什麼這章要講 `HTTP`

因為 `IPP` 不是直接裸跑在 TCP 上，  
它是被包進：

- `HTTP`

裡面的。

### HTTP request 的骨架

一個 HTTP request 大致是：

1. start line
2. header lines
3. blank line
4. optional entity body

在這章裡，entity body 裡放的就是：

- `IPP` header
- 以及要列印的 document data

### IPP 使用的 method

書裡很明確：

- `IPP` 用的是 `POST`

例如：

```http
POST /ipp HTTP/1.1
Content-Length: ...
Content-Type: application/ipp
Host: printer-host:631
```

### 這裡最重要的 HTTP headers

- `Content-Length`
- `Content-Type: application/ipp`
- `Host`

其中 `Content-Length` 特別重要，因為：

- spooler 在讀 response 時，需要知道後面 entity body 的長度

### HTTP response 要看什麼

對這章的 spooler 而言，最重要的是：

- status line

例如：

- `HTTP/1.1 200 OK`

如果這裡就不是 success，那後面的 IPP body 通常也不用高興太早。

### 這章的實務提醒

APUE 在後面 `printer_status` 的實作裡其實讓你看到：

- 真實世界的 protocol parsing 不能假設「一次 read 就拿到完整訊息」
- 也不能假設沒有 interim response

例如：

- 可能先收到 informational HTTP response
- header 和 body 可能分多次到

## 21.4 Printer Spooling

### 為什麼要有 spooler

UNIX 本來就有各種 print spooler，例如：

- BSD `lpd`
- `CUPS`
- System V spooler

但 APUE 在這章不是要教那些現成系統，而是要自己做一個簡化版，  
好讓你看到整體架構。

### 角色分工

這套系統裡有兩個主要程式：

- `print`：提交工作
- `printd`：管理 queue 並送去 printer

### `printd` 的 thread 分工

APUE 把工作拆成多個 threads：

- 一個 thread 監聽 client request
- 每來一個 client，spawn 一個 client worker thread 複製檔案到 spool area
- 一個 printer thread 專門和 printer 通訊，逐一送 job
- 一個 signal thread 專門處理 signals

這種分工很值得學，因為它讓每條 thread 主線都很清楚。

### spool 目錄與檔案

這章用到的典型路徑包括：

- `/etc/printer.conf`
- `/var/spool/printer`
- `/var/spool/printer/data`
- `/var/spool/printer/reqs`
- `/var/spool/printer/jobno`

你可以把它理解成：

- `data/`：真正待印文件副本
- `reqs/`：每筆 request 的控制資訊
- `jobno`：下一個 job ID

### `printer.conf`

設定檔最重要的兩個欄位是：

- `printserver`
- `printer`

也就是：

- 哪台主機跑 spooler daemon
- 哪台主機是實際 printer

這讓 `print` command 不需要硬編 printer 位址。

### security 設計重點

APUE 在這裡特別談安全，非常值得注意。

因為 spooler daemon 一開始可能需要：

- 綁 privileged port

所以它可能先以高權限啟動。  
但設計上應該盡快：

- drop privileges
- 轉成 `lp` 或其他低權限帳號

此外還要：

- 避免 buffer overflow
- 限制 spool 檔案權限
- 記錄可疑行為

這是一個很典型的 daemon 安全設計觀念。

## 21.5 Source Code

這節是整章最像真實專案的部分。

### 檔案分工

書裡把程式分成：

- `ipp.h`
- `print.h`
- `util.c`
- `print.c`
- `printd.c`

這種拆法很合理，因為：

- protocol definition
- common config / structs
- utility functions
- client
- daemon

各自責任清楚。

### `ipp.h`

這個 header 定義了：

- IPP status code class / values
- operation IDs
- attribute tags
- value tags
- `struct ipp_hdr`

這一層的重點是：

- 把協定常數集中管理

對系統程式來說，這種整理非常重要，否則 protocol code 會很難維護。

### `print.h`

這個 header 定義：

- config file 路徑
- spool directory 路徑
- daemon 使用的帳號名稱
- buffer size
- `IPP_PORT = 631`
- `struct printreq`
- `struct printresp`

### `printreq` / `printresp`

這兩個 struct 是：

- `print` command 與 `printd` 之間的 private protocol

`printreq` 大致包含：

- job size
- flags
- user name
- job name

`printresp` 大致包含：

- return code
- job ID
- error message

這是一個很好的案例，展示：

- 就算在同一個系統內，client 和 daemon 之間最好也定義清楚的 message format

### `util.c`

`util.c` 提供這類共用工具：

- host / printer address lookup
- 讀 `printer.conf`
- timed read
- `tread` / `treadn`
- server / connect helper

這一層的價值在於：

- 把和商業邏輯無關但很吵的 networking / config 細節抽出去

### `print.c`：client 端的主線

`print` command 大致會做：

1. 解析使用者選項
2. 開啟待印檔案
3. 取得檔案大小 / metadata
4. 連到 spooler daemon
5. 送出 `printreq`
6. 再送真正檔案內容
7. 收 `printresp`

這是一個典型的：

- request header
- then bulk data

的簡單 client protocol。

### `printd.c`：daemon 端的主線

這個檔案最有學習價值。

#### job queue

daemon 維護一個 job list，並用：

- mutex
- condition variable

保護它。

這樣：

- client worker thread 可以把新 job 放進 queue
- printer thread 沒工作時可以睡在 condvar 上

#### `get_newjobno` / `update_jobno`

這組函式負責：

- 發新 job ID
- 把目前 next job number 寫回 spool metadata

這裡也展現了：

- shared state 要用 mutex 保護

#### `client_thread`

每個 client 連進來後，daemon 會開一條 worker thread 去：

- 收 request
- 分配 job ID
- 把要列印的檔案複製到 spool `data/`
- 寫對應 request/control 檔到 `reqs/`
- 把 job 掛進 queue
- 回 `printresp`

這樣主 accept path 不會被慢 client 拖住。

#### `signal_thread`

這條 thread 專門等 signal。  
例如：

- `SIGHUP` 時設一個 `reread` flag

表示之後要重讀 config。

這是很典型的多 thread daemon 技巧：

- 把 signal handling 集中在單一 thread

#### `printer_thread`

這是最核心的 worker。

它會：

1. 等 queue 裡有 job
2. 必要時重新讀 config
3. 連到 printer
4. 建 IPP request
5. 再包成 HTTP request
6. 送出 header + file data
7. 讀 printer response
8. 成功後清理 spool 檔

這整條路線非常像實戰系統。

### `add_option`

APUE 用 `add_option` 這類 helper 去組 IPP attributes。  
這很合理，因為 IPP attribute encoding 很機械、很容易寫錯：

- tag
- name length
- name
- value length
- value

抽 helper 能避免主流程被 protocol encoding 細節淹沒。

### 組 HTTP header

printer thread 會建立像這樣的 request：

```http
POST /ipp HTTP/1.1
Content-Length: ...
Content-Type: application/ipp
Host: printer-name:631
```

接著再把：

- IPP header
- 真正文件內容

一起送出去。

這裡很適合複習 Chapter 14 的：

- `writev`

因為它能把多段 buffer 一次送出。

### `document-format`

這個 attribute 會根據 job flag 設成：

- `text/plain`
- 或 `application/postscript`

這是 client 告訴 printer：

- 後面文件內容要怎麼理解

### `printer_status`

這個函式很值得讀，因為它是實戰級 parsing 範例。

它要處理：

- HTTP response 可能分多次讀進來
- 可能先收到 informational response
- 要在 header 裡找到 `Content-Length`
- 要找到 blank line 才知道 HTTP header 結束
- 之後才去 parse IPP body
- IPP 內的 status / request ID 還要做 network-to-host 轉換

這整段的教育重點是：

- 真實世界 protocol parsing 一定要對 partial read 保持警覺

### `printer_status` 的一個重要技巧

它先用 HTTP status line 過濾：

- `100 Continue` 這類 informational 就繼續讀
- 非 success 的 HTTP status 就先視為錯誤

然後才檢查：

- IPP status
- request ID 是否和當前 job 對得上

這代表它是：

- 先看 transport/application container 層
- 再看真正 protocol body 層

分層思路很正確。

### 這章的一個真實世界提醒

APUE 最後還提到一個實務坑：

- 某些 printer 的 autosense 行為不一定完全照你期望

也就是說，就算 protocol 寫對了，硬體行為仍可能讓你需要 workaround。  
這很符合現實系統工程。

## 21.6 Summary

這章最大的價值在於：

- 它不是局部 API 示範，而是一個完整可運作的 daemon + client + protocol 案例

你可以把它看成：

- Chapter 11/12 的 threads
- Chapter 13 的 daemon
- Chapter 14 的 I/O
- Chapter 16 的 sockets

在同一個案例裡一起落地。

最重要的主線有三條：

- `IPP over HTTP over TCP/IP`
- spooler queue 與 thread 協作
- daemon 的安全與工程結構

### 這章讀完你應該真的會的事

- 知道為什麼 printer communication 常要經過 spooler daemon
- 知道 `IPP` 與 `HTTP` 在這章的分工
- 看得懂 `print` command 和 `printd` 之間的自訂 request/response protocol
- 能說出 printer daemon 裡各 thread 的責任分工
- 知道為什麼 `printer_status` 不能假設一次 `read` 就拿到完整 response
- 知道 least privilege 在 daemon 設計中的重要性

### 這章最容易踩坑的地方

- 把這章誤以為只是 printer-specific hack，而忽略它其實是完整 service case study
- 忘記 `IPP` 內含 binary 欄位，需要注意 byte order
- 以為 HTTP response 會一次完整到齊
- 不處理 queue synchronization，讓多 thread 搶同一份 job list
- 讓 daemon 長時間持有 root privileges

### 建議你現在立刻動手做

1. 自己畫一次資料流：`print -> printd -> printer`，把 `printreq`、HTTP、IPP、document data 分層標出來。
2. 用最小 socket 範例模擬一個只收 `printreq` 的假 spooler，先熟悉 client/server protocol。
3. 再寫一個簡化版 worker queue，用 mutex + condvar 做一條 producer / consumer 管線。

### 一句總結

這章是在教你：一個看似「只是送檔去印表機」的問題，實際上會牽涉協定分層、背景工作排程、thread 同步、daemon 安全與真實世界 I/O 處理的完整工程設計。
