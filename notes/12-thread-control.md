# Chapter 12 - Thread Control

## 這章在做什麼

如果 Chapter 11 是 threads 的基本觀念與基本同步，  
那 Chapter 12 就是在回答更實務的問題：

- thread 建出來之後，細節要怎麼調
- stack 要多大
- detached 要怎麼設
- synchronization object 本身也有 attributes 嗎
- 老 API 在 multithreaded world 怎麼變安全
- cancellation、signals、`fork`、I/O 跟 threads 混在一起時會出什麼問題

這章的本質其實是：

- 把「能跑」提升成「能安全控制」

如果你只讀 Chapter 11，通常只會寫出「會動的 thread 程式」。  
讀完 Chapter 12，才比較接近「知道哪些細節不能亂碰」。

## 本章小節地圖

- `12.1 Introduction`
- `12.2 Thread Limits`
- `12.3 Thread Attributes`
- `12.4 Synchronization Attributes`
- `12.5 Reentrancy`
- `12.6 Thread-Specific Data`
- `12.7 Cancel Options`
- `12.8 Threads and Signals`
- `12.9 Threads and fork`
- `12.10 Threads and I/O`
- `12.11 Summary`

## 先抓住這章最重要的心智模型

### thread programming 的難點不只在「多條 thread 同時跑」

更難的是：

- 每個 thread 有自己的局部狀態
- 但又共享 process-level 狀態
- 一部分 API 是 thread-safe
- 一部分只是 historical process API，必須小心補救

所以這章很多內容看起來像零碎 API，其實都在處理同一件事：

- 如何把傳統 UNIX / C world 的東西安全地搬到 multithreaded world

### attributes 是「建立前就先決定規則」的工具

`pthread_attr_t`、`pthread_mutexattr_t`、`pthread_condattr_t` 這類東西，本質上是在說：

- 建立物件之前，先定義它的行為

這比起「建立後再硬改內部狀態」乾淨很多，也更可攜。

### thread-safe 不等於 async-signal-safe

這是整章最值得牢記的觀念之一。

- thread-safe：多條 thread 同時呼叫仍安全
- async-signal-safe：被 signal handler 非同步打斷後重入也安全

前者已經不容易，後者更嚴格。  
很多函式最多做到 thread-safe，完全不代表你能在 signal handler 裡放心呼叫。

## 12.1 Introduction

這章一開始先把目標說得很清楚：

- 前一章學的是基本 threads 與 synchronization
- 這一章要看的是控制 thread 行為的細節

也就是從：

- 「thread 怎麼存在」

走到：

- 「thread 的屬性、私有資料、取消行為、和其他系統機制互動時怎麼安全」

## 12.2 Thread Limits

### thread 不是想開多少就開多少

很多人第一次寫 thread，會把它想成：

- 反正比 process 輕量，那就能無限開

這是錯的。  
thread 仍受系統資源限制，特別是：

- stack memory
- kernel / runtime 可管理的 thread 數量
- thread-specific data keys 數量

### APUE 這節特別列出的限制

可透過 `sysconf` 查的幾個重要值：

- `PTHREAD_DESTRUCTOR_ITERATIONS`
- `PTHREAD_KEYS_MAX`
- `PTHREAD_STACK_MIN`
- `PTHREAD_THREADS_MAX`

對應的 `sysconf` 參數像：

- `_SC_THREAD_DESTRUCTOR_ITERATIONS`
- `_SC_THREAD_KEYS_MAX`
- `_SC_THREAD_STACK_MIN`
- `_SC_THREAD_THREADS_MAX`

### 這些值的意義

#### `PTHREAD_DESTRUCTOR_ITERATIONS`

thread exit 時，thread-specific data destructor 最多會被重跑幾輪。  
因為某個 destructor 可能又把某些 key 設成 non-null，所以系統可能要多輪清理。

#### `PTHREAD_KEYS_MAX`

一個 process 最多能建立多少個 thread-specific data keys。

#### `PTHREAD_STACK_MIN`

thread stack 至少要多大，否則不合法。

#### `PTHREAD_THREADS_MAX`

一個 process 能建立 thread 數量的理論上限。  
但即使顯示「no limit」，也不代表真的無限，通常只是：

- 標準接口沒有給你固定上限
- 真實上限仍受資源約束

### 這節的實務意義

它不是要你背數字，而是提醒你：

- thread 數量設計本來就應該和 stack size、memory footprint、資源模型一起思考

## 12.3 Thread Attributes

### attributes object 的共同模式

這節很值得建立一個通用套路。  
大多數 pthread attribute API 都遵循這種模式：

1. 有一個 opaque attribute object
2. 先 `init`
3. 再 `get/set` 某些屬性
4. 用它去建立 thread 或 synchronization object
5. 最後 `destroy`

### thread attributes 的初始化與釋放

```c
int pthread_attr_init(pthread_attr_t *attr);
int pthread_attr_destroy(pthread_attr_t *attr);
```

`pthread_attr_init` 會把它設成 default attributes。  
`pthread_attr_destroy` 則負責清理由實作可能配置的內部資源。

### `detachstate`

最常用的 thread attribute 之一是：

- `detachstate`

可以是：

- `PTHREAD_CREATE_JOINABLE`
- `PTHREAD_CREATE_DETACHED`

對應 API：

```c
int pthread_attr_getdetachstate(const pthread_attr_t *restrict attr,
                                int *detachstate);
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);
```

### 什麼時候直接建立 detached thread 比較合理

如果你一開始就知道：

- 不需要 `join`
- 不需要 thread return value
- thread 做完就回收

那比起建立後再 `pthread_detach`，直接設：

- `PTHREAD_CREATE_DETACHED`

更乾淨。

### 一個建立 detached thread 的常見 helper

```c
int makethread(void *(*fn)(void *), void *arg)
{
    int err;
    pthread_t tid;
    pthread_attr_t attr;

    err = pthread_attr_init(&attr);
    if (err != 0)
        return err;

    err = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    if (err == 0)
        err = pthread_create(&tid, &attr, fn, arg);

    pthread_attr_destroy(&attr);
    return err;
}
```

### stack 相關 attributes

thread 和 process 很不同的一點是：

- 同一個 process 內可能有很多個 thread stacks

因此 stack size 就很實際。  
POSIX 提供這幾個相關能力：

- `stackaddr`
- `stacksize`
- `guardsize`

#### 1. `stacksize`

```c
int pthread_attr_getstacksize(const pthread_attr_t *restrict attr,
                              size_t *restrict stacksize);
int pthread_attr_setstacksize(pthread_attr_t *attr, size_t stacksize);
```

重點：

- 不能小於 `PTHREAD_STACK_MIN`
- 太小容易 stack overflow
- 太大則浪費虛擬記憶體，降低可建立 thread 數量

工程實務上很重要的一句話：

- 不要把大陣列或大物件隨手放在 thread stack 上

#### 2. `stackaddr` 與 `setstack`

```c
int pthread_attr_getstack(const pthread_attr_t *restrict attr,
                          void **restrict stackaddr,
                          size_t *restrict stacksize);
int pthread_attr_setstack(pthread_attr_t *attr,
                          void *stackaddr, size_t stacksize);
```

如果你要自己管理 stack，例如：

- 從 `malloc` 或 `mmap` 配空間

就可以用這組 API 指定 stack 位置。

要特別記住 APUE 的提醒：

- `stackaddr` 是 stack memory range 的最低可定位位址
- 不一定是「stack 成長的起點」

因為不同架構 stack 成長方向可能不同。

#### 3. `guardsize`

```c
int pthread_attr_getguardsize(const pthread_attr_t *restrict attr,
                              size_t *restrict guardsize);
int pthread_attr_setguardsize(pthread_attr_t *attr, size_t guardsize);
```

guard area 的用途是：

- 幫你在 stack overflow 時比較快發現問題

常見上會以 page 為單位保護。  
若你自己指定 `stackaddr`，系統通常視為：

- stack 由你自己管理
- guard buffer 也不再幫你處理

### 這節最重要的實務結論

- thread stack 是珍貴資源
- thread 數量與 stack size 是直接互相牽動的
- detached/joinable 應該在設計階段就先想清楚

## 12.4 Synchronization Attributes

這節是在說：

- 不只有 thread 自己有 attributes
- mutex、rwlock、condition variable、barrier 也有

### 12.4.1 Mutex Attributes

#### 初始化與銷毀

```c
int pthread_mutexattr_init(pthread_mutexattr_t *attr);
int pthread_mutexattr_destroy(pthread_mutexattr_t *attr);
```

#### mutex 最重要的三個屬性

- process-shared
- robust
- type

#### `process-shared`

這決定 mutex 是：

- 只給同一 process 的 threads 用
- 還是可放在 shared memory，跨 process 一起用

值通常是：

- `PTHREAD_PROCESS_PRIVATE`
- `PTHREAD_PROCESS_SHARED`

API：

```c
int pthread_mutexattr_getpshared(const pthread_mutexattr_t *restrict attr,
                                 int *restrict pshared);
int pthread_mutexattr_setpshared(pthread_mutexattr_t *attr, int pshared);
```

如果 mutex 只在單一 process 內用，實作通常可以更有效率。

#### `robust`

這個屬性主要在處理一種很尷尬的情況：

- 某 process 持有 mutex 時直接死掉

如果這把 mutex 是跨 process 共享的，那其他等待者會卡死。  
robust mutex 就是用來改善這件事。

值有：

- `PTHREAD_MUTEX_STALLED`
- `PTHREAD_MUTEX_ROBUST`

相關 API：

```c
int pthread_mutexattr_getrobust(const pthread_mutexattr_t *restrict attr,
                                int *restrict robust);
int pthread_mutexattr_setrobust(pthread_mutexattr_t *attr, int robust);
int pthread_mutex_consistent(pthread_mutex_t *mutex);
```

如果你拿到 robust mutex 時前任 owner 已死，`pthread_mutex_lock` 可能回：

- `EOWNERDEAD`

這不表示你沒拿到鎖，反而表示：

- 你現在拿到鎖了
- 但被保護的狀態可能不一致
- 你要先修復資料

修好後要呼叫：

- `pthread_mutex_consistent`

如果你沒修就解鎖，之後其他等待者可能只看到：

- `ENOTRECOVERABLE`

表示這把 mutex 保護的狀態已經被宣告不可恢復。

#### `type`

POSIX 定義四種 mutex type：

- `PTHREAD_MUTEX_NORMAL`
- `PTHREAD_MUTEX_ERRORCHECK`
- `PTHREAD_MUTEX_RECURSIVE`
- `PTHREAD_MUTEX_DEFAULT`

最實際的差別是：

- `NORMAL`：快，但錯誤檢查少
- `ERRORCHECK`：重複 lock / 非 owner unlock 比較容易回錯誤
- `RECURSIVE`：同一 thread 可重複 lock，同次數 unlock 後才真正釋放
- `DEFAULT`：交給實作決定映射到哪種行為

設定 API：

```c
int pthread_mutexattr_gettype(const pthread_mutexattr_t *restrict attr,
                              int *restrict type);
int pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type);
```

#### recursive mutex 不是萬能解

它適合的場景通常是：

- 為了相容舊有 single-threaded API
- 某函式持鎖後又必須呼叫另一個同樣會持同一把鎖的函式

但 APUE 特別提醒：

- 如果 condition variable 搭配的是 recursive mutex，事情會很危險

因為 `pthread_cond_wait` 只會做一次 unlock。  
如果你之前把 recursive mutex lock 了多次，等價於：

- 那把鎖其實沒有真正完全放開

條件可能永遠無法被其他 thread 改變。

### 12.4.2 Reader-Writer Lock Attributes

rwlock attributes 比較單純。  
POSIX 標準定義的重點幾乎只有：

- process-shared

相關 API：

```c
int pthread_rwlockattr_init(pthread_rwlockattr_t *attr);
int pthread_rwlockattr_destroy(pthread_rwlockattr_t *attr);

int pthread_rwlockattr_getpshared(const pthread_rwlockattr_t *restrict attr,
                                  int *restrict pshared);
int pthread_rwlockattr_setpshared(pthread_rwlockattr_t *attr, int pshared);
```

### 12.4.3 Condition Variable Attributes

condition variable 有兩個值得注意的屬性：

- process-shared
- clock

初始化：

```c
int pthread_condattr_init(pthread_condattr_t *attr);
int pthread_condattr_destroy(pthread_condattr_t *attr);
```

process-shared 的意義和前面一致：

- 單 process 用
- 或跨 process 用

clock attribute 則決定：

- `pthread_cond_timedwait` 的 timeout 要用哪個 clock 來判斷

API：

```c
int pthread_condattr_getclock(const pthread_condattr_t *restrict attr,
                              clockid_t *restrict clock_id);
int pthread_condattr_setclock(pthread_condattr_t *attr,
                              clockid_t clock_id);
```

這個細節很實際，因為：

- timeout 的正確性會受到時鐘來源影響

### 12.4.4 Barrier Attributes

barrier 目前標準上最重要的屬性也是：

- process-shared

API：

```c
int pthread_barrierattr_init(pthread_barrierattr_t *attr);
int pthread_barrierattr_destroy(pthread_barrierattr_t *attr);
int pthread_barrierattr_getpshared(const pthread_barrierattr_t *restrict attr,
                                   int *restrict pshared);
int pthread_barrierattr_setpshared(pthread_barrierattr_t *attr, int pshared);
```

## 12.5 Reentrancy

### thread-safe 到底是什麼

如果一個函式可被多條 thread 同時呼叫而不出錯，我們會說它：

- thread-safe

很多函式之所以不 thread-safe，經典原因是：

- 回傳指向 static buffer 的指標
- 使用未保護的 global state

### thread-safe 不代表 signal handler 也能安全呼叫

這節最重要的區分是：

- thread-safe
- async-signal-safe

一個函式可以對多 thread 安全，但仍然不能在 signal handler 裡用。  
這兩個層級不能混為一談。

### `_r` 版本函式

POSIX 常提供 reentrant / thread-safe 的替代版，例如：

- `getpwnam_r`
- `getpwuid_r`
- `getgrgid_r`
- `gmtime_r`
- `localtime_r`
- `strtok_r`
- `strerror_r`
- `ttyname_r`

共同設計思路通常是：

- 不再用共享 static buffer
- 改由 caller 提供 buffer

### `getenv` 為什麼是經典教材

舊式 `getenv` 若回傳的是共用靜態 buffer，多條 thread 同時呼叫就可能彼此覆蓋。  
要把它改成 reentrant，最常見的方法是：

- caller 自備 buffer

這就是 `getenv_r` 風格的設計。

### `pthread_once`

當某個全域初始化只該做一次，而且多條 thread 可能同時搶進來時，要用：

```c
int pthread_once(pthread_once_t *initflag, void (*initfn)(void));
```

典型用途：

- 初始化一把全域 mutex
- 建立一個 thread-specific key

### `FILE *` 也有 thread-safe 控制手段

標準 I/O 雖然通常會內建鎖，但有時你想把多個操作包成一個 atomic sequence。  
這時可以用：

```c
int ftrylockfile(FILE *fp);
void flockfile(FILE *fp);
void funlockfile(FILE *fp);
```

如果你已經自己鎖住 `FILE *`，就可以用：

- `getc_unlocked`
- `putc_unlocked`
- `getchar_unlocked`
- `putchar_unlocked`

來降低每個字元都重複 lock/unlock 的成本。

### 但再提醒一次

thread-safe 不代表 async-signal-safe。  
APUE 在這裡很明確地說：

- 就算你把 `getenv` 改得對 threads 安全
- 也不代表它能在 signal handler 裡安全使用

## 12.6 Thread-Specific Data

### 為什麼明明 thread 鼓勵共享，還需要 thread-private data

APUE 給了兩個理由，很值得記：

1. 有些資料本來就應該 per-thread
2. 有些舊介面必須被改造成 multithread-friendly

最經典的例子就是：

- `errno`

它原本看起來像 process-global，但在 multithreaded world 裡必須變成每條 thread 自己一份。

### 基本機制：key + per-thread value

流程大致是：

1. 建立一個 key
2. 每條 thread 針對同一個 key 可存不同 value
3. 用 `getspecific` / `setspecific` 存取自己的那份

### 建立 key

```c
int pthread_key_create(pthread_key_t *keyp, void (*destructor)(void *));
int pthread_key_delete(pthread_key_t key);
```

`destructor` 的角色是：

- 當 thread 正常結束或被 cancel 時
- 若該 key 在此 thread 有 non-null value
- 就呼叫 destructor 幫你清理

### destructor 什麼時候會跑，什麼時候不會跑

會跑的典型情況：

- thread `return`
- thread `pthread_exit`
- thread 被 cancel

不會跑的典型情況：

- process 直接 `exit`
- `_exit`
- `_Exit`
- `abort`
- 非正常整個 process 結束

這很重要，因為這代表 thread-specific destructor 不是 process cleanup 的萬靈丹。

### `pthread_key_delete` 不會幫你跑 destructor

這點很常被誤解。

- 刪 key，不等於幫所有 thread 清理 value

如果你要回收那些 memory，還要自己設計更完整的應用層清理流程。

### `pthread_once` 幾乎是 key 初始化的標配

如果你這樣寫：

```c
if (!inited) {
    inited = 1;
    pthread_key_create(&key, destructor);
}
```

就有 race。  
正確做法是：

```c
static pthread_key_t key;
static pthread_once_t once = PTHREAD_ONCE_INIT;

static void init_key(void)
{
    pthread_key_create(&key, free);
}
```

然後每條 thread 先：

```c
pthread_once(&once, init_key);
```

### 存與取

```c
void *pthread_getspecific(pthread_key_t key);
int pthread_setspecific(pthread_key_t key, const void *value);
```

如果尚未設定，`pthread_getspecific` 通常回：

- `NULL`

### 一個典型模式：每條 thread 一個 buffer

```c
static pthread_key_t key;
static pthread_once_t once = PTHREAD_ONCE_INIT;

static void key_init(void)
{
    pthread_key_create(&key, free);
}

char *get_buffer(void)
{
    char *buf;

    pthread_once(&once, key_init);
    buf = pthread_getspecific(key);
    if (buf == NULL) {
        buf = malloc(4096);
        if (buf != NULL)
            pthread_setspecific(key, buf);
    }
    return buf;
}
```

這類寫法常用來把「本來靠單一 static buffer 的舊函式」改到對 threads 安全。

## 12.7 Cancel Options

### cancellation 有兩個重要屬性

- cancelability state
- cancelability type

它們不放在 `pthread_attr_t` 裡，而是 thread 執行期間自己調。

### `cancelability state`

可設成：

- `PTHREAD_CANCEL_ENABLE`
- `PTHREAD_CANCEL_DISABLE`

API：

```c
int pthread_setcancelstate(int state, int *oldstate);
```

### 預設是 enable

如果 thread 處於：

- `PTHREAD_CANCEL_ENABLE`

那別人呼叫 `pthread_cancel` 後，該 thread 會在適當時機回應取消。

如果處於：

- `PTHREAD_CANCEL_DISABLE`

那 cancel request 不會立刻生效，而是：

- 先 pending

等你重新 enable 後，到了下一個 cancellation point 才可能真正取消。

### cancellation point

這是本節的關鍵詞。  
在 deferred cancellation 模式下，thread 並不是被立刻砍掉，而是會在某些點檢查：

- 有沒有 pending cancel request

常見 cancellation point 包括：

- `read`
- `write`
- `open`
- `close`
- `wait`
- `sleep`
- `select`
- `pthread_join`
- `pthread_cond_wait`
- `sigwait`

### 自己加 cancellation point

如果你的程式長時間都不會碰到標準 cancellation points，例如：

- 長時間 compute-bound loop

就可以自己插：

```c
void pthread_testcancel(void);
```

### `cancelability type`

有兩種：

- `PTHREAD_CANCEL_DEFERRED`
- `PTHREAD_CANCEL_ASYNCHRONOUS`

設定 API：

```c
int pthread_setcanceltype(int type, int *oldtype);
```

### 為什麼預設用 deferred，而不是 asynchronous

因為 asynchronous cancellation 太危險。  
它代表：

- thread 可能在任何時間點被中斷

如果那時它正在：

- 持有 mutex
- 修改共享資料
- 更新複雜不變量

你的程式很容易直接進入不一致狀態。  
所以實務上若不是非常特殊情境，通常應該偏向：

- deferred cancellation

## 12.8 Threads and Signals

### thread world 裡，signal 變更麻煩

在 multithreaded process 裡：

- 每條 thread 有自己的 signal mask
- 但 signal disposition 是整個 process 共用的

這句一定要背熟。

### 這句話的實際意思

每條 thread 可以各自決定：

- 哪些 signals 要 block

但如果某一條 thread 改了某個 signal 的處理方式，例如：

- 改成 ignore
- 改成某個 handler

那是整個 process 共享的，不是它自己獨享。

### signal 送到 multithreaded process 時，會去哪

APUE 指出：

- 若 signal 來自 hardware fault，通常會送給造成 fault 的那條 thread
- 其他一般 signals，可能送給 process 裡某一條符合條件的 thread

所以如果你沒有設計清楚，signal 行為會變得很難推理。

### `sigprocmask` 在 multithreaded process 裡不要用

POSIX 的說法很明確：

- multithreaded process 裡，`sigprocmask` 的行為是 undefined

應改用：

```c
int pthread_sigmask(int how, const sigset_t *restrict set,
                    sigset_t *restrict oset);
```

### 最推薦的 thread signal 處理模型：`sigwait`

```c
int sigwait(const sigset_t *restrict set, int *restrict signop);
```

它的核心策略是：

1. 先在所有 threads 把某些 signals block 起來
2. 開一條專門的 signal-handling thread
3. 讓那條 thread 用 `sigwait` 同步等待 signals

這麼做的好處很大：

- signal 不再用傳統 async handler 模式打斷一般執行緒
- 你可以在一般 thread context 裡處理 signal 事件
- 可呼叫的函式空間比 signal handler 大很多

### `sigwait` 的一個關鍵前提

在呼叫 `sigwait` 前，目標 signals 必須先被 block。  
不然會出現 timing window：

- signal 可能在 `sigwait` 真正睡下去前就先被送走

### `pthread_kill`

```c
int pthread_kill(pthread_t thread, int signo);
```

它是把 signal 送給某條 thread。  
但請注意：

- 如果該 signal 的 default action 是 terminate process
- 那整個 process 還是會死，不是只有那條 thread 死

### `alarm` 是 process resource，不是 thread resource

這句很重要。  
同一個 process 裡多條 thread 共用 alarm timer，所以：

- 多條 thread 若都想用 `alarm`，會互相干擾

### 一個典型 signal thread 範例

```c
static sigset_t mask;

static void *signal_thread(void *arg)
{
    int signo;
    (void)arg;

    for (;;) {
        sigwait(&mask, &signo);
        if (signo == SIGTERM)
            break;
    }
    return NULL;
}
```

這種設計通常比到處裝 signal handler 容易維護得多。

## 12.9 Threads and `fork`

### 多 thread process 呼叫 `fork`，最危險的不是 `fork` 本身，而是 lock state

`fork` 後 child 會得到整個 address space 的 copy。  
但如果 parent 原本有很多 threads，問題來了：

- child 只會留下呼叫 `fork` 的那條 thread

其他 threads 不會跟著存在。

### 這會導致什麼問題

如果 parent 其他 thread 當時持有：

- mutex
- rwlock
- condition variable 內部相關狀態

那 child 會繼承那些「看起來已上鎖」的狀態，卻沒有對應持鎖者可解鎖。  
結果就是：

- child 可能一開始就卡死

### 最安全的策略：`fork` 後立刻 `exec`

這是 POSIX 和實務世界都非常推崇的模式。  
因為一旦 `exec`：

- 舊 address space 整個被換掉
- 爛掉的 thread lock state 也一起消失

### `fork` 和 `exec` 之間只能做很少事

POSIX 規定：

- multithreaded process 的 child 在 `fork` 到 `exec` 之間
- 只應呼叫 async-signal-safe functions

這限制非常硬。  
意思是你不能把 child 當成「fork 出來再慢慢收拾世界」。

### `pthread_atfork`

如果 child 不是立刻 `exec`，APUE 提供的工具是：

```c
int pthread_atfork(void (*prepare)(void),
                   void (*parent)(void),
                   void (*child)(void));
```

三個 handler 的意義：

- `prepare`：在 parent 裡、`fork` 前執行，通常先把相關 locks 全拿住
- `parent`：`fork` 後在 parent 裡執行，釋放那些 locks
- `child`：`fork` 後在 child 裡執行，也釋放那些 locks

### handler 呼叫順序也有規則

如果註冊了多組 `pthread_atfork` handlers：

- `prepare` 會以註冊相反順序呼叫
- `parent` / `child` 會以註冊順序呼叫

這是為了讓 lock hierarchy 還能成立。

### 但 `pthread_atfork` 不是萬靈丹

APUE 非常務實地指出它有很多限制：

- complex synchronization object 很難完整重建狀態
- condition variable / barrier 沒有好清理方式
- error-checking / recursive mutex 的 child cleanup 可能有平台問題
- child fork handler 本身若呼叫非 async-signal-safe 函式，也有風險
- 如果 `fork` 發生在 signal handler 內，限制更嚴格

結論很務實：

- multithreaded process 裡，能 `fork` 後直接 `exec` 就盡量這樣做

## 12.10 Threads and I/O

### 多 thread 共用 file descriptor 時，最常見的坑是 shared file offset

同一個 process 的 threads 共享 open file descriptors。  
而 file descriptor 背後通常共享同一個 open file description，也就是：

- file offset 也可能是共享的

### 為什麼 `lseek + read` 會出事

假設兩條 thread 都在操作同一個 `fd`：

```c
/* thread A */
lseek(fd, 300, SEEK_SET);
read(fd, buf1, 100);

/* thread B */
lseek(fd, 700, SEEK_SET);
read(fd, buf2, 100);
```

如果 A 做完 `lseek` 後，B 插進來也做 `lseek`，  
最後 A 的 `read` 就可能讀錯地方。

### 正解：`pread` / `pwrite`

```c
ssize_t pread(int fd, void *buf, size_t nbytes, off_t offset);
ssize_t pwrite(int fd, const void *buf, size_t nbytes, off_t offset);
```

它們把：

- 指定 offset
- 執行讀寫

合成單一操作，因此不會被別條 thread 亂掉共享 file offset。

### 什麼情境應優先想到 `pread/pwrite`

- 多 threads 隨機讀同一個檔案
- 多 threads 對同一個檔案不同區段做 I/O
- 不想額外用 mutex 串行化全部檔案操作

## 12.11 Summary

這章把 thread programming 從「基本會用」推進到「知道邊界在哪」。

你應該把它理解成幾個主題的整合：

- 用 attributes 微調 thread 與 synchronization object 的行為
- 用 thread-safe / reentrant 技巧改造舊介面
- 用 thread-specific data 管理 per-thread 狀態
- 了解 cancellation、signals、`fork`、I/O 在 multithreaded 世界裡的特殊規則

如果要濃縮成最重要的幾條實務原則，大概是：

- detached / stack size / guardsize 要在設計階段決定
- recursive mutex 只在真的必要時才用
- thread-safe 不等於 async-signal-safe
- `pthread_once` 是初始化共享資源的標配
- signal 處理優先考慮 `pthread_sigmask + sigwait`
- multithreaded process 中 `fork` 最安全的模式是 `fork` 後立刻 `exec`
- 同一個 fd 被多 thread 用時，優先考慮 `pread/pwrite`

### 這章讀完你應該真的會的事

- 會用 `pthread_attr_t` 設定 detached thread、stack size、guard size
- 理解 mutex 的 `process-shared`、`robust`、`type` 屬性差異
- 分辨 thread-safe、reentrant、async-signal-safe
- 會用 `pthread_once`、`pthread_key_create`、`pthread_setspecific`
- 理解 deferred cancellation 與 asynchronous cancellation 的差別
- 會用 `pthread_sigmask` 和 `sigwait` 設計專用 signal thread
- 理解多 thread 環境下 `fork` 與 `lseek+read` 為什麼危險

### 這章最容易踩坑的地方

- 以為 thread 數量只看 CPU，不看 stack 和 memory
- 設了太小 stack，卻在 thread 裡放大區域陣列
- 把 recursive mutex 當成萬能 deadlock 解法
- 誤以為 thread-safe 函式就能安全地在 signal handler 裡呼叫
- 用手寫布林旗標初始化 thread-specific key，而不是 `pthread_once`
- 在 multithreaded process 裡用 `sigprocmask`
- `fork` 之後還在 child 裡做一堆複雜 pthread 操作
- 多 thread 共用同一個 `fd` 卻還用 `lseek + read/write`

### 建議你現在立刻動手做

1. 寫一個 detached worker thread helper，實際設一次 `detachstate`
2. 寫一個 per-thread buffer 範例，練 `pthread_once + pthread_key_create`
3. 把某個舊式 static-buffer 函式改成 thread-safe 版本
4. 做一條專門 `sigwait` 的 signal thread
5. 用兩條 thread 對同一個檔案測試 `lseek+read` 和 `pread` 的差異

### 一句總結

Chapter 12 的核心不是多背幾個 `pthread_*` 名字，而是理解：multithreaded 程式真正困難的地方，在於你得知道哪些狀態是 thread-local、哪些是 process-shared、哪些 API 只是看起來能用但其實在某些情境下非常危險。
