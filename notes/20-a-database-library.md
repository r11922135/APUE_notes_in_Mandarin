# Chapter 20 - A Database Library

## 這章在做什麼

這一章看起來像是在寫一個小型 database，但它真正的教學重點不是：

- 教你做出商業級 DBMS

而是：

- 用一個夠完整、夠真實的 case study，把前面學過的 UNIX system programming 工具串起來

尤其是：

- file I/O
- record locking
- process concurrency
- 資料格式設計
- library interface 設計

APUE 在這章要做的是一個簡化版、多 process 可安全共用的 `key-value store`，  
重點不是 SQL、query language，也不是 transaction engine，而是：

- UNIX 介面足不足以支撐一個可並行存取的資料庫函式庫

## 本章小節地圖

- `20.1 Introduction`
- `20.2 History`
- `20.3 The Library`
- `20.4 Implementation Overview`
- `20.5 Centralized or Decentralized?`
- `20.6 Concurrency`
- `20.7 Building the Library`
- `20.8 Source Code`
- `20.9 Performance`
- `20.10 Summary`

## 先抓住這章最重要的心智模型

### 這章在教的其實是「帶鎖的檔案格式設計」

如果你把這章只當成：

- 一堆 `db_*` API

那會錯過它最有價值的地方。  
這章真正核心是：

- 你怎麼把資料放進檔案
- 怎麼找到它
- 怎麼刪掉它
- 怎麼在多 process 同時操作時還保持一致

### lookup 結構 和 concurrency control 同等重要

很多人直覺會先想：

- hash table 要怎麼設計？

但 APUE 在這章真正不斷強調的是：

- 沒有 locking，再漂亮的資料結構也會壞

所以這章其實同時在處理兩條線：

- 資料要怎麼定位
- 競爭要怎麼控制

### 這不是高階資料庫理論，而是 UNIX 風格的工程取捨

這個 database library 很刻意地做了一些取捨：

- 只支援字串 key / data
- 一筆 record 只有一個 key
- 沒有 secondary index
- 不追求 B-tree 或動態 hash 的高級特性

這些取捨不是因為作者不會做，而是因為這章的目標是：

- 用足夠簡單的設計，把 concurrency / file format / implementation detail 講清楚

## 20.1 Introduction

APUE 先回顧歷史背景：

- 在 UNIX 很早期的年代，多人共用 database 並不容易做

原因很直接：

- 沒有像樣的 IPC
- 沒有 byte-range locking

但隨著 UNIX 演進，到了後來：

- record locking
- 更完整的 I/O / IPC

都成熟了，UNIX 其實已經足以支撐可靠的 multiuser database。

這章的定位就是：

- 用一個小型 key-value store 來驗證這件事

## 20.2 History

這節主要在幫你建立歷史脈絡。

### `dbm` / `ndbm` / `db`

UNIX 世界裡早就有資料庫函式庫，例如：

- `dbm`
- `ndbm`
- `db`

這些函式庫很重要，但 APUE 特別指出它們的一個問題：

- 多數早期版本不提供多 process 同時更新時需要的 concurrency control

也就是說，它們能查得快，不代表它們能安全地被多個 writer 同時用。

### 商業系統為什麼還得自己做 locking

商業資料庫系統通常會：

- 使用 advisory locking
- 有時甚至自己實作更低成本的 locking primitive

因為在大量並發下，光是 system call 開銷都可能是成本。

### 這章的重點不是發明新資料結構

APUE 也提到常見資料結構像：

- B+ tree
- linear hashing
- extendible hashing

但這章沒有打算完整重現那些設計。  
它要做的是：

- 一個能清楚說明 UNIX 介面與並行問題的教學版本

## 20.3 The Library

這節先把 public interface 定義清楚。

### `DBHANDLE`

database 被打開後，會回傳一個：

- `DBHANDLE`

它是 opaque handle，外部不需要知道內部結構長什麼樣。

這個設計和：

- `FILE *`

有點像，都是：

- 對外提供抽象 handle
- 內部藏 implementation detail

### `db_open` / `db_close`

```c
DBHANDLE db_open(const char *pathname, int oflag, ... /* int mode */);
void db_close(DBHANDLE db);
```

`db_open` 成功後會建立 / 打開兩個檔案：

- `pathname.idx`
- `pathname.dat`

也就是：

- index file
- data file

### `db_store`

```c
int db_store(DBHANDLE db, const char *key, const char *data, int flag);
```

這是最重要的寫入介面。  
`flag` 有三種：

- `DB_INSERT`
- `DB_REPLACE`
- `DB_STORE`

你可以這樣記：

- `DB_INSERT`：只准插入新 key
- `DB_REPLACE`：只准改既有 key
- `DB_STORE`：存在就 replace，不存在就 insert

### `db_store` 的回傳語意要特別記

這裡有一個很容易寫錯 / 誤判的細節：

- 一般錯誤回 `-1`
- 但如果你指定 `DB_INSERT`，而 key 已存在，回傳值是 `1`

也就是：

- `1` 不是成功插入
- 而是「不是系統錯誤，但插入沒做成，因為 key 已存在」

這是很典型的 library API design 細節。

### `db_fetch` / `db_delete`

```c
char *db_fetch(DBHANDLE db, const char *key);
int db_delete(DBHANDLE db, const char *key);
```

`db_fetch`：

- 找到就回 data pointer
- 找不到回 `NULL`

`db_delete`：

- 找到並刪除回 `0`
- 找不到回 `-1`

### `db_rewind` / `db_nextrec`

```c
void db_rewind(DBHANDLE db);
char *db_nextrec(DBHANDLE db, char *key);
```

這組讓你可以：

- 從頭掃描整個 database

但 APUE 明講：

- record 回傳順序沒有排序保證

因為底層不是 B-tree，而是：

- hash + chaining

所以 `db_nextrec` 比較像：

- 順著檔案實際存放順序掃過去

不是：

- 照 key 排序迭代

### 範例：最小 API 使用方式

```c
#include "apue_db.h"
#include <fcntl.h>
#include <stdio.h>

int main(void) {
    DBHANDLE db;
    char *data;

    db = db_open("demo", O_RDWR | O_CREAT | O_TRUNC, 0644);
    if (db == NULL) {
        perror("db_open");
        return 1;
    }

    if (db_store(db, "alice", "engineer", DB_INSERT) != 0) {
        perror("db_store");
        db_close(db);
        return 1;
    }

    data = db_fetch(db, "alice");
    if (data != NULL) {
        printf("alice => %s\n", data);
    }

    db_close(db);
    return 0;
}
```

## 20.4 Implementation Overview

這節開始真正講底層設計。

### 為什麼拆成兩個檔案

APUE 用：

- `.idx`
- `.dat`

兩個檔案分工。

原因是這是很常見、也很容易說明的設計：

- index file 負責 key -> data location
- data file 負責真正 payload

### 為什麼 key / data 都存成字串

書裡刻意選擇：

- key 和 data 都是 null-terminated 字串

而不是任意 binary blob。

原因是：

- 可攜性高
- 檔案可以直接用一般 UNIX 工具觀察
- 範例簡單

代價是：

- 空間效率不如 binary encoding

這是標準的工程取捨。

### 單一 key、沒有 secondary key

這個資料庫很刻意地只做：

- 每筆 data 一個 key

不做：

- secondary indexes
- 一筆資料多重索引

這讓整體實作能聚焦在：

- primary hash lookup
- concurrency

### index file 的三大部分

index file 包含：

1. free-list pointer
2. hash table
3. index records

你可以想成：

- free list：管理刪除後可重用的 index slots
- hash table：每個 bucket 指向一條 hash chain
- index records：真正的 key 與 data location 資訊

### fixed-size hash table + chaining

APUE 選的是：

- 固定大小 hash table
- bucket 裡用 linked list chaining

這不是最先進，但非常適合教學，因為它讓你清楚看到：

- hash value 先定位 bucket
- bucket 再沿 chain 找實際 record

### index record 長什麼樣

index record 大致包含：

- chain ptr
- idx len
- key
- data offset
- data length

而且很多欄位是以：

- ASCII 文字

存起來的。

這讓你用 `cat` 就能直接看 database 檔案，非常有教學效果。

### data file 很單純

data file 本質上就是：

- 一筆筆 data record 串起來

index file 負責記：

- 某筆 data 在 `.dat` 的 offset
- 長度是多少

### free list 是刪除重用的關鍵

當 record 被刪掉時，不是立刻把檔案中間空洞整體整理掉，  
而是：

- 把對應 index slot 接到 free list

之後 `db_store` 可能重用這塊空間。

這是很多檔案型資料結構會用到的做法。

### 範例：教材裡的簡化觀察程式

APUE 有個例子會先建立資料庫，再寫幾筆資料，  
然後直接用：

- `cat db4.idx`
- `cat db4.dat`

去觀察實際檔案內容。

這個例子的教育意義很大，因為它讓你直接看到：

- hash chain pointer 怎麼串
- data offset / length 怎麼對應

也就是說，這不是「黑盒資料庫」，而是你真的能打開看懂的 on-disk format。

## 20.5 Centralized or Decentralized?

這節是在談整體架構，而不是單一 API。

### 兩條路

有多個 process 要用同一個 database 時，可以走兩條路：

1. centralized
2. decentralized

### centralized

意思是：

- 只有一個 central db manager 真正碰資料檔
- 其他 process 都透過 IPC 向它請求操作

優點：

- recovery 較集中
- 可能較容易做統一排程 / 優先順序控制

缺點：

- 每次操作都多一層 IPC
- 資料常要多 copy 一次

### decentralized

意思是：

- 每個使用 library 的 process 都直接操作檔案
- 但大家要共同遵守 locking discipline

優點：

- 少掉 central manager 與 IPC 開銷
- 速度通常比較好

缺點：

- locking 較難設計
- recovery 與一致性責任分散

### APUE 這章選的是 decentralized

因為本章最想展示的是：

- UNIX 的 file I/O + record locking 能不能讓多 process 安全共用資料庫

答案是：

- 可以，但你要把 locking 設計好

## 20.6 Concurrency

這節是整章的靈魂之一。

### coarse-grained locking

最簡單做法是：

- 整個 database 當成一個鎖

例如用 index file 的某個固定 byte 當：

- 全局 read lock / write lock

這樣：

- `db_fetch` / `db_nextrec` 拿 read lock
- `db_store` / `db_delete` / `db_open` 拿 write lock

優點：

- 簡單

缺點：

- 併發度很差

因為：

- 某個 process 只是在動 hash chain A
- 另一個 process 明明只想讀 hash chain B
- 卻還是被擋住

### fine-grained locking

APUE 採用更細的方式：

- 先鎖對應的 hash chain
- 需要動 free list 時再鎖 free list
- append 新 index / data record 時，再鎖對應 append 區段

這樣做的效果是：

- 不同 bucket 上的操作比較能並行

### 為什麼 data file 不像 index file 那樣全面加鎖

因為這套設計主要是：

- 透過 index 來定位資料

而讀取 data record 時，很多情境不需要像 hash chain 那樣的結構鎖。  
真正需要特別處理的是：

- append 時要避免兩個 writer 同時往檔尾搶位置

### 為什麼不能用 stdio 來實作這章

APUE 很明確說：

- 這裡直接用 `read/write/readv/writev`
- 不用 stdio

原因是 stdio buffering 很危險：

- 你可能讀到很久以前 buffer 裡的舊資料
- 但另一個 process 早就已經改過檔案

在需要準確 concurrency semantics 的資料庫裡，這種風險不能接受。

### advisory vs mandatory locking

這章測試了：

- no locking
- advisory locking
- mandatory locking

no locking 的結果很直接：

- 多 process 下會出現隨機錯誤與資料不一致

advisory locking：

- 靠 cooperating processes 自律遵守

mandatory locking：

- kernel 幫你更強硬地 enforce

但通常成本更高。

## 20.7 Building the Library

這節偏工程實務。

### static library

教材示範了怎麼把 `db.c` 編成：

- `libapue_db.a`

### shared library

也示範了怎麼編成：

- `libapue_db.so.1`

### 這節真正想讓你注意的是

- 一個像 database 這樣的功能，不一定要做成 daemon
- 也可以做成可重用的 library

這和前幾章的 service / daemon 設計形成對照。

## 20.8 Source Code

這節是整章最硬的一部分。  
你不一定要背每個 helper 名字，但要掌握資料流。

### `apue_db.h`

public header 做了幾件事：

- 定義 `DBHANDLE`
- 宣告 public API
- 定義 `DB_INSERT` / `DB_REPLACE` / `DB_STORE`
- 定義實作上的長度限制

### `DB` private struct

內部 `DB` 結構會記：

- index/data file descriptors
- internal buffers
- 目前讀到的 idx/data offset 與 length
- hash table 大小
- 各種統計計數器

這表示 library 不是純 stateless 的；  
它有一個 per-open-database 的 state object。

### 這章常見的 private helper 類型

你會看到一些職責很清楚的 helper，例如：

- hash 計算
- 找 record 並加鎖
- 讀 index record
- 讀 data record
- 寫 index record
- 寫 data record
- 找 free slot
- 執行 delete

重點不是名稱，而是這種拆法：

- 每個 helper 都同時在處理「資料定位」和「鎖的正確性」

### `db_open`

`db_open` 除了打開檔案，也會在新建資料庫時初始化：

- free list
- hash table 區域

這也是為什麼它在建立資料庫時需要寫鎖。

### `db_fetch` 的流程

大致上是：

1. 對 key 算 hash
2. 找到對應 hash chain
3. 在 chain 上逐筆比對 key
4. 找到後再去 `.dat` 讀出 data record

這就是最典型的：

- hash + chain traversal

### `db_store` 的幾種情境

這是整章最值得理解的函式之一。  
它至少要處理這些 case：

- key 不存在，要 insert
- key 已存在，要 replace
- replace 時新舊 data 長度相同
- replace 時新舊 data 長度不同
- 刪除過的 free slot 能不能重用

#### 長度相同時

如果新舊紀錄大小相同，很多時候可以：

- 直接覆寫原位置

這是最快也最乾淨的 case。

#### 長度不同時

如果長度不同，通常就不能簡單原地覆寫，  
因為會影響後面 record 的邊界。

常見做法是：

- 把舊 record 視為刪除
- 新 record 重新 append 或重找 free slot

### free list 重用策略其實很保守

APUE 這個版本只在：

- key 長度匹配
- data 長度匹配

時才重用 deleted slot。

這很保守，但它換來的是：

- 實作簡單
- 不必處理更複雜的空間切割 / 合併

### `db_delete`

刪除不是把檔案整體搬動，而是：

- 把 key 塗成空白之類的 deleted 狀態
- 更新鏈結
- 把 slot 掛回 free list

這就是典型的：

- logical delete + freelist recycle

### `db_rewind` / `db_nextrec`

這組 API 不沿著 hash chain 走，而是：

- 順著 index file 掃描

因此它會遇到：

- 已刪除的空白 record

所以 `db_nextrec` 要主動跳過這些 deleted entries。

### `db_nextrec` 為什麼要鎖 free list

這是很重要的細節。

因為當你在 sequential scan 時，別的 process 可能正好在：

- delete record
- 把 record 連回 free list

如果不加這個保護，就可能：

- index 剛讀到一筆
- data 卻已被別人「刪乾淨」

APUE 明確用鎖避免這個 race。

### append 時的 locking 很講究

當 `db_store` 要把新 record append 到檔尾時，  
APUE 不是簡單粗暴鎖整個世界，而是：

- `index file` 從 해당 hash chain 結尾到 EOF 做 write lock
- `data file` 在 append 階段做適當 write lock

目標是：

- 只擋住會彼此衝突的 append writer
- 不去妨礙其他不相干的 reader / writer

這就是 fine-grained locking 的工程味道。

## 20.9 Performance

這節很有價值，因為它不是只說「fine-grained 比較好」，而是實測。

### 測試工作負載

每個 child 大致會做：

1. 寫入很多 records
2. 依 key 讀回
3. 大量隨機 fetch
4. 間歇性 delete / insert / replace
5. 最後刪掉自己寫的 records

這種 workload 很合理，因為它混合了：

- read-heavy
- update
- delete

### no locking 的結果

多 process 下幾乎就是：

- 隨機錯誤
- record 找不到
- race condition 導致不一致

這章用實驗再次證明：

- 沒有鎖的 shared-file update 基本上不可用

### 單 process 下 locking 也有成本

即使沒有 contention：

- advisory locking 也會增加 system call 成本

APUE 的實測結論之一是：

- 單 process 下，advisory locking 相比 no locking 的 clock time 會多出一大段
- mandatory locking 又比 advisory 更重

### 多 process 下 fine-grained locking 較好

當 process 數量上升時：

- fine-grained locking 的 clock time 明顯比 coarse-grained locking 好

理由不是神祕優化，而是很直觀：

- 鎖持有時間較短
- 不同 process 較不容易互相睡住
- block / wakeup 次數較少

### mandatory locking 的系統成本更高

APUE 觀察到：

- mandatory fine-grained locking 比 advisory fine-grained locking 會再多出顯著 system time

這代表：

- 讓 kernel 更強硬地介入一致性控制，是有成本的

### clock time 不只是 CPU time

書裡還提醒：

- clock time 的非線性成長，不只來自你程式碼本身
- 還包含 context switching、blocking、wakeup 等系統行為

這對讀 benchmark 非常重要。

## 20.10 Summary

這章透過一個小型 database library 告訴你：

- 只靠 UNIX 的檔案、locking 與函式庫抽象，就能做出可多 process 共用的資料庫核心雛形

你應該把這章的收穫整理成四件事：

- public API 怎麼設計
- on-disk format 怎麼設計
- free list / hash chain 怎麼運作
- locking 粒度怎麼影響正確性與效能

### 這章讀完你應該真的會的事

- 能說出這個資料庫為什麼要拆成 `.idx` 和 `.dat`
- 知道 `db_store` 三種 flag 的語意差別
- 知道 fixed-size hash table + chaining 在這裡怎麼工作
- 知道 free list 的角色是什麼
- 能解釋 coarse-grained 與 fine-grained locking 的差別
- 知道 `db_nextrec` 為什麼不是排序遍歷

### 這章最容易踩坑的地方

- 把重點放在 hash lookup，卻忽略 concurrency
- 以為 stdio buffering 在這類資料庫場景也沒問題
- 忘記 replace 時新舊 record 長度可能不同
- 以為 sequential scan 會照 key 排序
- 不理解 free list 和 delete 之間的 race condition

### 建議你現在立刻動手做

1. 自己畫一次 `.idx` / `.dat` 檔案格式，把 free list、hash table、index record 關係畫清楚。
2. 用最小程式實際呼叫 `db_store` / `db_fetch` / `db_delete` / `db_nextrec`，觀察 API 語意。
3. 如果你要挑戰自己，試著改 free-list 重用策略，讓「較大的空洞」也能被重用。

### 一句總結

這章是在教你：資料庫函式庫的難點不只是怎麼找資料，而是怎麼在檔案格式、空間回收、鎖的粒度與效能之間做出正確的工程取捨。
