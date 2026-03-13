# Chapter 11 - Threads

## 這章在做什麼

這章是 APUE 第一次正式把「同一個 process 裡可以有多條 execution flow」講清楚。  
如果前面幾章主要在講：

- process 怎麼建立、結束、互相管理
- file descriptor、signal、memory 這些 process-level 資源怎麼運作

那這章就是把鏡頭往 process 內部再推一層，開始講：

- 同一個 process 裡的多個 thread 怎麼共用資源
- 為什麼 thread 雖然比 process 輕量，但同步問題更難
- `pthread` 提供了哪些基本 API
- 各種 synchronization primitive 應該怎麼選

你可以把這章看成兩條主線：

- 第一條：thread 的生命週期
- 第二條：thread 共享資料後，怎麼避免 race condition

## 本章小節地圖

- `11.1 Introduction`
- `11.2 Thread Concepts`
- `11.3 Thread Identification`
- `11.4 Thread Creation`
- `11.5 Thread Termination`
- `11.6 Thread Synchronization`
- `11.7 Summary`

## 先抓住這章最重要的心智模型

### thread 不是比較小的 process，而是 process 內部的執行單位

最白話的理解：

- process 比較像一個完整的執行容器
- thread 是這個容器裡的一條執行路徑

同一個 process 裡的 threads 共享很多東西：

- address space
- global memory
- heap
- open file descriptors
- current working directory
- signal disposition

但每個 thread 自己也有私有狀態：

- thread ID
- register set
- stack
- scheduling state
- signal mask
- thread-specific data
- 自己那份 `errno`

### thread 的優勢幾乎都來自「共享」，thread 的麻煩也幾乎都來自「共享」

thread 好用的原因：

- 建立成本通常比 process 小
- 溝通成本通常比 process 小
- 共用資料自然，不需要額外 IPC

thread 容易出錯的原因：

- 任何共享可寫資料都可能發生 race
- 問題常不是 crash，而是偶發錯誤
- bug 常和時序有關，所以很難重現

### 同步的本質不是讓程式變慢，而是讓程式有定義

很多初學者會把 lock 想成「效能殺手」。  
但在多 thread 程式裡，很多時候不加同步不是比較快，而是：

- 根本沒有正確語意

所以這章最核心的能力，不是背 API，而是知道：

- 哪些資料需要保護
- 應該用哪種 primitive 保護
- 保護到什麼粒度最合理

## 11.1 Introduction

APUE 在這章一開始先提醒你一件事：

- 以前你學的 UNIX process，通常只有一條 control flow

也就是說，一個 process 在某個時間點，概念上只做一件事。  
但如果把多條 control flow 放進同一個 process，就能在共享同一組 process 資源的前提下，同時做多件事。

這個模型的最大吸引力是：

- 共享變得很自然

但只要開始共享，就必須處理 consistency 問題。  
所以這章最後自然會走到 synchronization。

## 11.2 Thread Concepts

### thread 帶來哪些實際好處

APUE 列出的好處可以整理成四類：

#### 1. 把非同步問題拆成同步寫法

例如你有多種事件要處理：

- network request
- timer event
- log flushing

如果全部靠單一流程加大量 state machine 去硬寫，程式會很亂。  
thread 的好處是可以讓不同工作各自用比較直觀的同步流程去寫。

#### 2. 共享記憶體和 file descriptor 很自然

如果是多 process，想共享資料通常得靠：

- shared memory
- pipe
- socket
- message queue

但多 thread 在同一個 process 內，本來就共享 address space 和 open files。

#### 3. 可提升 throughput

如果有多個互相獨立的任務，一個 single-threaded 程式會隱含地把它們 serial 化。  
多 thread 則可以讓這些任務交錯甚至平行執行。

#### 4. 可提升互動程式的 response time

典型例子：

- 一條 thread 專心處理 UI / terminal interaction
- 另一條 thread 處理背景計算或 I/O

這樣程式不會因為某個慢動作就整體卡死。

### thread 的好處不只在 multi-core

這點 APUE 特別講得很好。  
很多人以為 thread 的價值建立在：

- 「要有多核心才有意義」

這是不完整的。  
即使在單核心上，thread 仍然有價值，因為：

- 你的程式結構可以更清楚
- 一條 thread block 時，別條 thread 可能還能繼續跑

所以 thread 不只是效能工具，也是設計工具。

### thread 共享什麼、私有什麼

你可以直接記成兩張表。

共享的：

- executable code
- global variables
- heap memory
- file descriptors
- process credentials
- current working directory
- signal disposition

每條 thread 私有的：

- `pthread_t`
- register context
- stack
- signal mask
- scheduling priority / policy
- thread-specific data
- `errno`

### `pthreads` 是這章的主角

APUE 這章以 POSIX threads，也就是 `pthreads` 為主。  
關鍵觀念是：

- 這不是某家 OS 專屬 API
- 它是 UNIX / POSIX 世界的標準 thread 介面

編譯時常見會需要：

```sh
cc -pthread file.c
```

或類似平台選項。

## 11.3 Thread Identification

### thread ID 和 process ID 不一樣

每個 process 有 `pid_t`。  
每個 thread 則有 `pthread_t`。

兩個最重要的差別：

- `PID` 在整個系統通常有系統層級意義
- `pthread_t` 只在該 process 內有意義

### `pthread_t` 不能假設是整數

這是非常重要的可攜性觀念。  
不同平台可能把 `pthread_t` 實作成：

- unsigned integer
- unsigned long
- pointer
- 甚至某種 opaque structure

所以比較 thread ID 時，不要自己寫：

```c
if (tid1 == tid2) { ... }
```

應該用：

```c
pthread_equal(tid1, tid2)
```

### 取得自己的 thread ID

```c
#include <pthread.h>

pthread_t pthread_self(void);
int pthread_equal(pthread_t t1, pthread_t t2);
```

### 一個簡單例子

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

static void *worker(void *arg)
{
    (void)arg;
    printf("worker pid=%ld tid=%lu\n",
           (long)getpid(), (unsigned long)pthread_self());
    return NULL;
}

int main(void)
{
    pthread_t tid;
    pthread_create(&tid, NULL, worker, NULL);
    printf("main   pid=%ld tid=%lu\n",
           (long)getpid(), (unsigned long)pthread_self());
    pthread_join(tid, NULL);
    return 0;
}
```

這個例子最重要的觀察是：

- `PID` 一樣
- `pthread_t` 不一樣

### 新 thread 不應該直接偷看 shared `pthread_t` 變數

APUE 在這裡舉了一個很容易忽略的 race：

- parent 呼叫 `pthread_create`
- `pthread_create` 還沒返回
- new thread 就先開始跑
- 如果 new thread 直接去看 parent 還沒寫好的全域 `tid` 變數，可能讀到未初始化值

所以新 thread 要知道自己的 ID，最可靠的方式是：

- 直接呼叫 `pthread_self()`

## 11.4 Thread Creation

### 建立 thread 的核心 API

```c
#include <pthread.h>

int pthread_create(pthread_t *restrict tidp,
                   const pthread_attr_t *restrict attr,
                   void *(*start_rtn)(void *),
                   void *restrict arg);
```

### 四個參數怎麼看

- `tidp`：成功時回填新 thread 的 ID
- `attr`：thread attributes，先用 `NULL` 就是 default
- `start_rtn`：新 thread 的起點函式
- `arg`：丟給起點函式的單一參數

如果你要傳多個參數，做法通常不是塞很多全域變數，而是：

- 包成一個 structure
- 把 structure address 當 `arg` 傳進去

### thread 一建立後，誰先跑沒有保證

這句一定要記：

- `pthread_create` 成功後，caller thread 和 new thread 誰先繼續，沒有保證

所以你不能寫出依賴固定執行順序的程式。  
只要你的程式邏輯需要某種順序，就要自己同步。

### 新 thread 繼承哪些東西

APUE 特別提到：

- 共享 process address space
- 繼承 calling thread 的 floating-point environment
- 繼承 calling thread 的 signal mask
- 但 new thread 的 pending signal set 會被清空

這些細節之後會在 signals 和 thread control 章節接回來。

### `pthread` 錯誤回報和一般 POSIX API 不一樣

這點很容易踩坑。

很多 POSIX 函式失敗時是：

- return `-1`
- `errno` 設錯誤碼

但很多 `pthread_*` 函式失敗時是：

- 直接回傳 error number
- 不靠 `errno`

所以常見寫法是：

```c
int err = pthread_create(&tid, NULL, worker, NULL);
if (err != 0) {
    /* err 本身就是錯誤碼 */
}
```

### 一個基本建立範例

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

static void *worker(void *arg)
{
    int n = *(int *)arg;
    printf("worker got %d\n", n);
    return (void *)0;
}

int main(void)
{
    pthread_t tid;
    int value = 42;
    int err;

    err = pthread_create(&tid, NULL, worker, &value);
    if (err != 0) {
        fprintf(stderr, "pthread_create failed: %d\n", err);
        return 1;
    }

    pthread_join(tid, NULL);
    return 0;
}
```

### 這個範例也順便提醒一件事

如果你把 `arg` 指向某個會很快失效的資料，例如：

- 某個函式即將 return 的 stack 變數

那 worker thread 讀到的東西可能已經壞掉。  
這和 Chapter 7 講過的「不要把會消失的 stack address 傳到函式外」是同一個問題，只是 thread 讓它更常發生。

## 11.5 Thread Termination

### process 結束和 thread 結束要分開看

這一節最重要的第一句就是：

- 如果任何一條 thread 呼叫 `exit`、`_exit`、`_Exit`
- 整個 process 都會結束

所以：

- `exit` 是 process-level termination
- `pthread_exit` 才是 thread-level termination

### 單一 thread 結束的三種方式

1. 從 start routine 直接 `return`
2. 被同 process 其他 thread cancel
3. 呼叫 `pthread_exit`

### `pthread_exit`

```c
void pthread_exit(void *rval_ptr);
```

這個 `rval_ptr` 可以被其他 thread 透過 `pthread_join` 撿回來。

### `pthread_join`

```c
int pthread_join(pthread_t thread, void **rval_ptr);
```

它的效果很像 thread world 裡的 `waitpid`：

- 等指定 thread 結束
- 取得它的 termination status

如果對方是：

- 正常 `return`：拿到 return value
- `pthread_exit(x)`：拿到 `x`
- 被 cancel：拿到 `PTHREAD_CANCELED`

### joinable vs detached

預設 thread 是 joinable。  
也就是說，它結束後，termination status 還保留著，等其他 thread 來 `join`。

如果你不 `join`，某些 thread 資源就無法被回收。  
這個概念和 process 裡不 `wait` child 會留下 zombie，很像。

### 很重要的危險：不要回傳指向自己 stack 的指標

這是 APUE 在本節特別強調的坑。

錯誤示範：

```c
static void *worker(void *arg)
{
    int local = 123;
    return &local;
}
```

或：

```c
static void *worker(void *arg)
{
    struct foo x = {1, 2, 3};
    pthread_exit(&x);
}
```

問題不是語法，而是生命週期：

- thread 一結束，它的 stack 很可能就被回收或重用
- `pthread_join` 拿到的 pointer 可能已經失效

正確做法通常是：

- 用 heap (`malloc`)
- 或用靜態 / 全域資料
- 或由 caller 提供儲存空間

### `pthread_cancel`

```c
int pthread_cancel(pthread_t tid);
```

它不是「立刻把 thread 殺死」，而是：

- 對 target thread 發出 cancel request

真正何時取消、能不能取消，Chapter 12 會再細講。

### cleanup handlers

thread 也有點像 process 的 `atexit` 機制：

```c
void pthread_cleanup_push(void (*rtn)(void *), void *arg);
void pthread_cleanup_pop(int execute);
```

你可以把它理解成：

- 註冊 thread 結束或被 cancel 時要做的清理動作

而且這些 handler 是 stack-like 的：

- 後註冊的先執行

### cleanup handler 什麼時候會跑

APUE 說明的重點：

- thread 呼叫 `pthread_exit`
- thread 因 cancellation 結束
- 或 `pthread_cleanup_pop(execute != 0)`

這些情況都可能觸發 cleanup function。

### `pthread_cleanup_push/pop` 有語法限制

這兩個常被實作成 macro，所以一定要：

- 成對出現
- 在同一個 lexical scope 裡使用

而且 APUE 特別提醒：

- 在 `push` 和 `pop` 中間直接 `return`
- 屬於 undefined behavior

可攜寫法如果要提早離開，應該偏向：

- `pthread_exit`

### detached thread

```c
int pthread_detach(pthread_t tid);
```

一旦 detach：

- thread 結束後資源可立即回收
- 不能再 `pthread_join`

所以 detached thread 適合這種工作：

- fire-and-forget background task
- 不需要回傳結果
- 不需要 caller 等它

## 11.6 Thread Synchronization

### 為什麼需要同步

只要多條 thread 可同時讀寫同一塊資料，就會有 race。  
這種 race 不只出現在「大動作」，連看似簡單的 `count++` 都不是原子操作。

因為 `count++` 通常至少分成：

1. 讀 memory
2. 在 register 裡加一
3. 寫回 memory

如果兩條 thread 交錯，就可能少算。

### 本節的五大 primitive

APUE 這章介紹五種最基本的同步工具：

- mutex
- reader-writer lock
- condition variable
- spin lock
- barrier

你可以把它們先粗分成：

- 互斥類：mutex、rwlock、spin lock
- 等待條件類：condition variable
- 集合點類：barrier

### 11.6.1 Mutexes

#### mutex 是最基本的 lock

mutex 全名是 mutual exclusion。  
核心語意非常直接：

- 一次只允許一條 thread 進 critical section

#### 基本 API

```c
int pthread_mutex_init(pthread_mutex_t *restrict mutex,
                       const pthread_mutexattr_t *restrict attr);
int pthread_mutex_destroy(pthread_mutex_t *mutex);

int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);
```

靜態初始化可用：

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
```

#### `pthread_mutex_trylock`

如果不能 block，可以試：

- `pthread_mutex_trylock`

它拿不到鎖時不會睡，而是回：

- `EBUSY`

#### 一個 reference count 範例

```c
struct foo {
    int count;
    pthread_mutex_t lock;
};

void foo_hold(struct foo *fp)
{
    pthread_mutex_lock(&fp->lock);
    fp->count++;
    pthread_mutex_unlock(&fp->lock);
}

void foo_rele(struct foo *fp)
{
    pthread_mutex_lock(&fp->lock);
    fp->count--;
    pthread_mutex_unlock(&fp->lock);
}
```

這個例子本身不難，但它建立了很重要的觀念：

- 共享狀態必須連「讀、改、判斷」一起視為需要保護的整體

不是只保護其中一半。

### 11.6.2 Deadlock Avoidance

#### deadlock 最常見的來源：lock ordering 不一致

如果 thread A 先鎖 `L1` 再鎖 `L2`，  
thread B 卻先鎖 `L2` 再鎖 `L1`，  
就可能互卡。

#### 最有效的預防方式：固定 lock hierarchy

例如整個程式規定：

- 永遠先拿 `hashlock`
- 再拿 object lock

只要所有地方都守同一順序，就能大幅降低 deadlock 風險。

#### 如果架構太複雜，可能得退而求其次

這時可以：

- 用 `pthread_mutex_trylock`
- 拿不到就放掉已持有的 lock
- 清理狀態後稍後再試

#### 粗鎖與細鎖的 trade-off

APUE 在這裡其實傳達了很實務的觀念：

- 鎖太粗：併發度差
- 鎖太細：程式太複雜，容易死鎖或寫錯

工程上通常不是追求「鎖越細越好」，而是追求：

- 正確
- 容易維護
- 效能夠用

### 11.6.3 `pthread_mutex_timedlock` Function

有時候你不想無限等一把 mutex，可以用：

```c
int pthread_mutex_timedlock(pthread_mutex_t *restrict mutex,
                            const struct timespec *restrict tsptr);
```

重點：

- 它等到某個 absolute time
- 超時回 `ETIMEDOUT`

這裡很容易搞錯：

- 不是「等 3 秒」
- 而是「等到某個時間點」

這和許多 POSIX timeout API 的設計一致。

### 11.6.4 Reader-Writer Locks

#### 什麼時候 rwlock 比 mutex 更合理

如果你的情境是：

- 讀很多
- 寫很少

那用 reader-writer lock 可能比 mutex 更好。  
因為它允許：

- 多個 readers 同時進來
- writers 則必須獨佔

#### 基本 API

```c
int pthread_rwlock_init(pthread_rwlock_t *restrict rwlock,
                        const pthread_rwlockattr_t *restrict attr);
int pthread_rwlock_destroy(pthread_rwlock_t *rwlock);

int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_tryrdlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_trywrlock(pthread_rwlock_t *rwlock);
int pthread_rwlock_unlock(pthread_rwlock_t *rwlock);
```

#### 很適合的例子：搜尋多、修改少的 queue / map

如果多條 worker thread 經常查 queue 裡有沒有自己的 job，  
但真正 insert / remove 比較少，就很適合：

- 查找用 `rdlock`
- 修改用 `wrlock`

### 11.6.5 Reader-Writer Locking with Timeouts

跟 mutex 一樣，rwlock 也有 timeout 版本：

```c
int pthread_rwlock_timedrdlock(pthread_rwlock_t *restrict rwlock,
                               const struct timespec *restrict tsptr);
int pthread_rwlock_timedwrlock(pthread_rwlock_t *restrict rwlock,
                               const struct timespec *restrict tsptr);
```

超時一樣是：

- `ETIMEDOUT`

而且時間一樣是：

- absolute time

### 11.6.6 Condition Variables

#### condition variable 不是 lock

這是很多人第一次學時最容易搞混的地方。

condition variable 的用途不是互斥，而是：

- 讓 thread 等某個條件成立

它通常必須和 mutex 一起使用。

#### 心智模型

你可以把 condition variable 想成：

- 一個安全的睡覺與叫醒機制

但真正被保護的是「條件本身」，不是 condition variable 物件。

#### 基本 API

```c
int pthread_cond_init(pthread_cond_t *restrict cond,
                      const pthread_condattr_t *restrict attr);
int pthread_cond_destroy(pthread_cond_t *cond);

int pthread_cond_wait(pthread_cond_t *restrict cond,
                      pthread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
                           pthread_mutex_t *restrict mutex,
                           const struct timespec *restrict tsptr);

int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
```

#### `pthread_cond_wait` 最重要的語意

呼叫時你必須已經鎖住 mutex。  
然後它會：

1. 原子地把自己掛到等待隊列
2. 原子地解開 mutex
3. 睡著
4. 被叫醒後再重新鎖住 mutex
5. 然後返回

這個「原子地解鎖並睡眠」是 condition variable 的靈魂。  
它就是為了避免：

- 檢查條件後到睡著前的 race window

#### 為什麼一定要用 `while`，不要只用 `if`

正確寫法：

```c
pthread_mutex_lock(&lock);
while (workq == NULL)
    pthread_cond_wait(&ready, &lock);
/* 現在 workq != NULL */
pthread_mutex_unlock(&lock);
```

理由有兩個：

- 被叫醒時條件不一定還成立
- 可能有 spurious wakeup 或其他 thread 先把條件改掉

#### `signal` vs `broadcast`

- `pthread_cond_signal`：至少喚醒一條等待中的 thread
- `pthread_cond_broadcast`：喚醒全部等待中的 threads

如果所有 waiters 都該重新檢查條件，就常用 `broadcast`。

#### 一個 producer-consumer 風格範例

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t ready = PTHREAD_COND_INITIALIZER;
int has_work = 0;

void *worker(void *arg)
{
    (void)arg;
    pthread_mutex_lock(&lock);
    while (!has_work)
        pthread_cond_wait(&ready, &lock);
    has_work = 0;
    pthread_mutex_unlock(&lock);
    return NULL;
}

void submit_work(void)
{
    pthread_mutex_lock(&lock);
    has_work = 1;
    pthread_mutex_unlock(&lock);
    pthread_cond_signal(&ready);
}
```

### 11.6.7 Spin Locks

#### spin lock 和 mutex 的差別

兩者都能做互斥。  
差別在等待方式：

- mutex 拿不到通常會睡
- spin lock 拿不到會 busy-wait 一直轉

#### 什麼時候 spin lock 才可能合理

通常只有在：

- critical section 很短
- 不想付出 sleep / wakeup 成本
- 或某些 low-level / kernel-like 場景

才可能值得用。

#### user-space 程式裡通常沒那麼好用

因為 user thread 可能被 scheduler preempt。  
如果持鎖 thread 被換下去，其他 threads 還在那裡狂 spin，就只是白白燒 CPU。

#### 基本 API

```c
int pthread_spin_init(pthread_spinlock_t *lock, int pshared);
int pthread_spin_destroy(pthread_spinlock_t *lock);

int pthread_spin_lock(pthread_spinlock_t *lock);
int pthread_spin_trylock(pthread_spinlock_t *lock);
int pthread_spin_unlock(pthread_spinlock_t *lock);
```

#### 使用原則

- 持鎖時間要非常短
- 持鎖期間不要做可能睡眠的事
- 不要在不確定平台行為的情況下把它當萬能加速器

### 11.6.8 Barriers

#### barrier 的用途

barrier 不是保護共享資料，而是：

- 讓一群 threads 在某個同步點會合

每條 thread 都做到某個階段後，在 barrier 等。  
等到所有合作 threads 都到齊，再一起繼續。

#### 基本 API

```c
int pthread_barrier_init(pthread_barrier_t *restrict barrier,
                         const pthread_barrierattr_t *restrict attr,
                         unsigned int count);
int pthread_barrier_destroy(pthread_barrier_t *barrier);
int pthread_barrier_wait(pthread_barrier_t *barrier);
```

#### `count` 代表什麼

- 必須有多少條 thread 抵達 barrier，大家才會一起通過

#### `pthread_barrier_wait` 的特殊返回值

通過 barrier 時：

- 多數 thread 看到 `0`
- 其中一條 arbitrary thread 會看到 `PTHREAD_BARRIER_SERIAL_THREAD`

這很有用，因為你可以把那條 thread 當「通關後做彙總工作的人」。

#### 一個典型例子

假設把大工作切成 8 份：

1. 8 條 worker threads 各自處理自己的部分
2. 全部在 barrier 等待
3. 到齊後某一條 thread 或 main thread 做 merge

這比手工管理一堆 counter 和 signal 乾淨很多。

## 11.7 Summary

這章真正要你建立的，不只是「會開 thread」，而是：

- 你知道 thread 是共用 process 資源的多條 control flow
- 你知道共享一旦涉及可寫資料，就一定要同步
- 你知道不同同步 primitive 解的是不同問題

如果把這章濃縮成一張圖，大概就是：

- `pthread_create`：建立 thread
- `pthread_exit` / `return` / `cancel`：結束 thread
- `pthread_join` / `pthread_detach`：管理 thread 結束後的資源
- `mutex`：一條一條進
- `rwlock`：多讀單寫
- `condvar`：等待條件成立
- `spin lock`：短時間忙等
- `barrier`：所有人集合後再一起走

### 這章讀完你應該真的會的事

- 分辨 process 與 thread 在共享資源上的本質差異
- 正確使用 `pthread_create`、`pthread_join`、`pthread_detach`
- 理解為什麼不能回傳 thread stack 上的位址
- 用 mutex 保護共享狀態
- 用 condition variable 寫出不漏訊號的等待邏輯
- 判斷什麼時候該用 rwlock、spin lock、barrier

### 這章最容易踩坑的地方

- 以為 `pthread_t` 一定能直接當整數比較
- 忘記 `pthread` 函式常直接回傳 error code，而不是設 `errno`
- 讓 thread 回傳指向自己 stack 的 pointer
- 以為 `pthread_cancel` 是立即殺死對方
- 用 `if` 包 `pthread_cond_wait`，而不是 `while`
- 同時用多把 mutex，卻沒有固定 lock ordering
- 在 user-space 把 spin lock 當效能萬靈丹

### 建議你現在立刻動手做

1. 寫一個主 thread 加兩個 worker threads 的小程式
2. 先故意不加鎖去做共享 counter 累加，觀察結果不穩定
3. 再用 mutex 修正
4. 再改成 producer-consumer，加入 condition variable
5. 最後做一個分段計算後用 barrier 集合同步的版本

### 一句總結

Chapter 11 的核心不是「thread 可以同時跑」，而是「thread 因為共享而方便，也因為共享而危險」；真正的功力在於你能不能用對同步工具，把共享變成可控而不是災難。
