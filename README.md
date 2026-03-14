# APUE 3rd Edition 中文筆記

這份目錄依照 *Advanced Programming in the UNIX Environment, 3rd Edition* 的章節架構整理，每章一份 Markdown，重點是把觀念講清楚、講白話，並保留對實作真正有用的 API 與系統語意。

## 筆記標準

這套筆記目前統一採用下面的寫法：

- 依教材章節與小節順序整理，不是只做主題式摘要。
- 中文盡量白話，但保留重要專有名詞英文。
- 每章都先建立心智模型，再往下講 API、語意細節、常見誤解與實務意義。
- 補有助於理解的 `C` 範例程式碼，而不是只列函式名稱。
- 每章結尾都整理「讀完應該會什麼」、「最容易踩坑的地方」、「建議立刻動手做」。

## 章節索引

1. [Chapter 1 - Overview](./01-overview.md)  
   UNIX programming 的總覽，建立 system call、library、shell、file descriptor、process 等共同語言。
2. [Chapter 2 - UNIX Standardization and Implementations](./02-unix-standardization-and-implementations.md)  
   整理 POSIX、Single UNIX Specification、feature test macros 與不同 UNIX 實作差異。
3. [Chapter 3 - File I/O](./03-file-io.md)  
   `open/read/write/lseek/close` 的核心語意、atomicity、file offset 與基本 I/O 心智模型。
4. [Chapter 4 - Files and Directories](./04-files-and-directories.md)  
   檔案屬性、inode、directory、link、permission、`stat`、`umask`、`chmod`、`unlink` 的完整基礎。
5. [Chapter 5 - Standard I/O Library](./05-standard-io-library.md)  
   `FILE *`、buffering、`fopen/fgets/fputs/fread/fwrite`、memory stream、temporary file 與 stdio 陷阱。
6. [Chapter 6 - System Data Files and Information](./06-system-data-files-and-information.md)  
   `passwd/group/shadow`、login/accounting 資料、system identification 與 time/date 相關 API。
7. [Chapter 7 - Process Environment](./07-process-environment.md)  
   `main` 參數、environment、memory layout、`malloc/free`、`setjmp/longjmp`、resource limits。
8. [Chapter 8 - Process Control](./08-process-control.md)  
   `fork/vfork/exec/wait`、process lifetime、zombie、orphan 與 process image replacement。
9. [Chapter 9 - Process Relationships](./09-process-relationships.md)  
   process group、session、controlling terminal、job control 的完整關係圖與操作語意。
10. [Chapter 10 - Signals](./10-signals.md)  
    signal 的心智模型、`sigaction`、masking、pending、`sigsuspend`、`EINTR` 與實務寫法。
11. [Chapter 11 - Threads](./11-threads.md)  
    POSIX threads 基礎、`pthread_create/join/detach`、thread ID、共享資源與基本同步原語。
12. [Chapter 12 - Thread Control](./12-thread-control.md)  
    thread attributes、mutex / rwlock / condvar / spin lock / barrier、cancellation、TSD、signals。
13. [Chapter 13 - Daemon Processes](./13-daemon-processes.md)  
    daemonize 流程、`syslog`、single-instance daemon、daemon 設計原則與實作細節。
14. [Chapter 14 - Advanced I/O](./14-advanced-io.md)  
    nonblocking I/O、record locking、`select/poll`、AIO、`readv/writev`、`readn/writen`、`mmap`。
15. [Chapter 15 - Interprocess Communication](./15-interprocess-communication.md)  
    `pipe`、`popen`、coprocess、`FIFO`、XSI IPC、message queue、shared memory、POSIX semaphores。
16. [Chapter 16 - Network IPC: Sockets](./16-network-ipc-sockets.md)  
    socket 基本模型、addressing、`bind/listen/accept/connect`、`send/recv`、socket options、nonblocking。
17. [Chapter 17 - Advanced IPC](./17-advanced-ipc.md)  
    `UNIX domain socket`、`socketpair`、unique connection、`SCM_RIGHTS`、fd passing、open server。
18. [Chapter 18 - Terminal I/O](./18-terminal-io.md)  
    `termios`、special characters、canonical / noncanonical mode、`VMIN/VTIME`、window size、`curses`。
19. [Chapter 19 - Pseudo Terminals](./19-pseudo-terminals.md)  
    `pty` 的 master/slave 模型、`posix_openpt` 流程、`pty_fork`、`script`、coprocess 與 interactive automation。
20. [Chapter 20 - A Database Library](./20-a-database-library.md)  
    `key-value store` library 的 API、`.idx/.dat` 檔案格式、hash chain、free list、record locking 與效能比較。
21. [Chapter 21 - Communicating with a Network Printer](./21-communicating-with-a-network-printer.md)  
    `IPP over HTTP`、print spooler daemon、client/server protocol、thread 分工、job queue 與 network printer 溝通。

## 建議閱讀路線

- 想先打底：`Chapter 1` 到 `Chapter 6`
- 想把 process / thread 學紮實：`Chapter 7` 到 `Chapter 12`
- 想補 daemon、I/O、IPC、socket、terminal：`Chapter 13` 到 `Chapter 19`
- 想看大型 case study：`Chapter 20`、`Chapter 21`
- 想查實作用法：優先回看 `Chapter 10`、`Chapter 14`、`Chapter 15`、`Chapter 16`、`Chapter 18`、`Chapter 21`

## 補充

- 目前每章範例以「幫助理解 API 與語意」為主，還不是一套可直接編譯執行的完整專案。
- 如果下一步要繼續擴充，可以再補：
  - 每章練習題與解題方向
  - 可編譯執行的範例程式專案
  - 附錄整理
