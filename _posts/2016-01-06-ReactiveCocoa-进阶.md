---
layout:     post
title:      Mypeloton 源码解析：日志部分
subtitle:   Mypeloton 日志部分的原理分析，关键代码解析
date:       2022-7-27
author:     BY
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - 内存数据库
    - C++
---

> 此博客是对开源数据库Peloton中日志系统的源码分析

 [peloton源码](https://github.com/vittvolt/15721-peloton)



---

### WAL：预写式日志

#### 传统磁盘数据库中的WAL

**中心概念**：数据文件（存储着表和索引）的修改必须在这些动作被日志记录之后才被写入，即在描述这些改变的日志记录被刷到持久存储以后

**运行逻辑**：

* *变更发生时*：先将变更后内容记入WAL Buffer，再将更新后的数据写入Data Buffer。
* *Commit*：WAL Buffer持久化到磁盘即算提交成功，Data Buffer写磁盘可以在提交之后发生。
* *Checkpoint*：同步当前Data Buffer中内容到磁盘，确保检查点前的事务已经持久化入磁盘。

**优势**：

* 日志记录体量小于实际数据，持久化速度更快，作为提交标准能提高数据库吞吐率。
* 将日志记录批量更新到磁盘，顺序读写效率高。（不同于每遇到一次更改就调用IO将日志记录写入磁盘）

#### Mypeloton中的WAL

**差别**：

* *变更发生时*：不用在事务操作存入log，而是将变更记录入一个rw_set中。
* *Commit*：提交前，根据 rw_set 将更改信息写入日志系统，并等待日志记录刷新到硬盘后提交事务。

### Mypeloton日志实现

#### 运行逻辑

事务在完成所有操作之后，commit之前，会将变更提交给LogManager进行日志记录。

开始记录时，LogManager会为每一个线程单独绑定一个BackendLogger（Thread Local），每个BackendLogger负责将收到的变更信息写入LogBuffer。

![image.png](https://img-blog.csdnimg.cn/img_convert/5605459258fa8fabf017c91e7aca9961.png)

LogBuffer分为两个循环队列：available_pool，persist_pool（其中队列节点是128kb的Buffer块）。available_pool用于提供空闲Buffer块，写满了日志信息的Buffer块则装入persist_pool。

FrontendLogger作为一个单独的线程，在后台每隔0.2ms一次循环，每次循环则收集所有LogBuffer的persist_pool中的日志信息，并将其写入磁盘，这个操作称为Flush。

![image.png](https://img-blog.csdnimg.cn/img_convert/a6990d14607a9c2cb2f20e229084c690.png)

而对于每一次事务commit，需要等待其log写入磁盘后才能提交成功，这个等待操作由一对由条件变量实现的函数实现：FrontendLogger每次写完，释放锁；LogManager收到锁后，获取当前已Flush的最大事务id（max_flushed_cid），并根据当前事务的id（current_cid）判断此事务log是否已经Flush，若没有，等待下一次锁的释放。若有，则事务可以完成commit。

#### 具体函数流程

![image.png](https://img-blog.csdnimg.cn/img_convert/5cfcc117e30edbcb97883b1d62810458.png)

**1 初始化**

1. `TransactionManager`在结束事务所有操作时，进入 `TransactionManager::CommitTransaction()`函数。
2. `TransactionManager::CommitTransaction()`函数能获取到一个此事务的Insert，Update，Delete操作列表。
3. `TransactionManager::CommitTransaction()`调用 `LogManager::LogBeginTransaction()`，记录下事务的开始信息。
4. `LogManager::LogBeginTransaction()` 随即调用 `LogManager::GetBackendLogger()` 获取或者新建 `BackendLogger` 对象，一个线程对应一个单独的 `BackendLogger`。

**2 记录更改**

1. `TransactionManager::CommitTransaction()`函数遍历上述操作列表，将该事务所有的Insert，Update，Delete操作通过三个函数传入log信息：

```cpp
LogManager::LogInsert()
LogManager::LogDelete()
LogManager::LogUpdate()
```

2. `BackendLogger` 调用 `BackendLogger::Log()` 将log信息记录入一个特殊的Log Buffer Pool：

```cpp
buffer // 指pool中一个128kb的单元。

// 这两个池都是CircularBufferPool的实例对象
available_pool // 当前可用的空闲池，将log信息直接放入。
persist_pool // 当 available_pool 中一个 buffer 满了之后，将其推送至此。
```

**3 记录入盘**

* `FrontendLogger` 通过 `FrontendLogger::MainLoop()`函数定期循环。
* 每次睡醒后，从 `persist_pool`获取 `buffer`，并将其中的记录写入磁盘。
* 写入磁盘后，更新 `max_flushed_cid`：目前已经提交了的最大的事务号（是事务号么）。
* 每一次刷新入磁盘完毕，都会调用 `LogManager::FrontendLoggerFlushed()`函数，告知 `TransactionManager`此次刷新完毕。

**4 完成提交**

* `TransactionManager::CommitTransaction()`在完成所有操作后，调取 `LogManager::LogCommitTransaction()`，提交Commit记录。
* 最后， `LogManager::LogCommitTransaction()`则会调取 `LogManager::WaitForFlush(`)函数，等待 `FrontendLogger`将此事务刷新到磁盘。

#### 其他事项

* MyPeloton日志暂时没有 `checkpoint`以及恢复系统。

---

### Mypeloton源码解析

#### 功能类

##### LogManager

整个日志的核心，日志的管理器，以单例模式运行：整个数据库程序中只有一个LogManager实例

![img60.jpg](https://img-blog.csdnimg.cn/img_convert/a147d84c8e55ba399937cce835f17902.png)

**主要成员函数**

> ***LogBeginTransaction()***

```cpp
void LogBeginTransaction(cid_t commit_id);
```

*变量*： `commit_id`：申请 commit 的对应 transaction 的 id

*功能*：TransactionManager在commit事务时，调用此系列函数，进行日志记录。

* 获取对应线程的专属 `BackendLogger`并将record交由其存储入缓冲池
* 记录的是该事务的信，记录类型：`TransactionRecord`

*同类函数*：其中记录类型有所不同，类型是 `TupleRecord`

```cpp
// Insert 和 Update 会记录其修改的tuple, delete不会
void LogManager::LogUpdate(cid_t commit_id, const ItemPointer &old_version, const ItemPointer &new_version);
void LogManager::LogInsert(cid_t commit_id, const ItemPointer &new_location);
void LogManager::LogDelete(cid_t commit_id, const ItemPointer &delete_location);
```

> ***LogCommitTransaction()***

```cpp
void LogManager::LogCommitTransaction(cid_t commit_id);
```

*功能*：标记此事务的日志记录完成，注意：调用此函数的TransactionManager在此函数结束后，才会标记事务状态为提交状态。

* 此函数结束前，会使用 `WaitForFlush()`确保当前cid对应的记录已经刷新到磁盘。

> ***GetBackendLogger()***

```cpp
BackendLogger *GetBackendLogger(unsigned int hint_idx);
```

变量：`hint_idx`：在 `LoggerMappingStrategyType == MANUAL`的时候有用

功能：获取或者新建 `BakendLogger`实例，并将其绑定到 `FrontendLogger`

* 如果 `LogManager::backend_logger`不为空，则返回。
* 如果为空，则构建一个新的 `BakendLogger`并指向它，将其根据Maping策略放置到指定的 `frontend_logger`内。

附加：（Mapping 策略枚举类）

```cpp
enum class LoggerMappingStrategyType {
  INVALID = INVALID_TYPE_ID,
  ROUND_ROBIN = 1, // 轮询方式
  AFFINITY = 2,
  MANUAL = 3 // 根据传入的hint_idx选择
};
```

> ***FrontendLoggerFlushed()***
> 
> ***WaitForFlush()***

```cpp
void LogManager::FrontendLoggerFlushed()
void LogManager::WaitForFlush(cid_t cid)
```

*功能*：结合使用，用于保证事务的提交是按次序的。

前者在 `FrontendLogger`中使用，将日志缓存池中内容刷新到磁盘后使用。

后者在 `LogManager::LogCommitTransaction()`中使用，判断当前 `transaction`对应的日志是否已经刷新到磁盘（`cid `<= `PersistentFlushedCommitId`）

> ***GetPersistentFlushedCommitId()***

```cpp
cid_t GetPersistentFlushedCommitId();
```

功能：找到 `PersistentFlushedCommitId`：当前已经刷新到磁盘的最大cid。

* 当只有一个 `FrontendLogger`时，是已经刷新到磁盘的最大cid
* 当有多个 `FrontendLogger`时，是所有 `FrontendLogger`中获取到的 `max_flushed_cid`中的最小值（为了确保比此 `cid`小的 `transaction`都已经刷新到磁盘）

> ***StartStandbyMode()***

```cpp
void StartStandbyMode();
```

*功能*：为 `FrontendLogger`创建单独的线程，并调用 `FrontendLogger:MainLoop`让其进入待命模式，当模式转变为Logging模式时，其会进行刷盘。

**主要成员变量**

> ***frontend_loggers***

```cpp
// There is only one frontend_logger of some type
// either write ahead or write behind logging
std::vector<std::unique_ptr<FrontendLogger>> frontend_loggers;
```

`FronTedLogger`实例数量，Mypeloton中只有一个实例（线程）。

> ***backend_logger***

```cpp
// Each thread gets a backend logger
thread_local static BackendLogger *backend_logger = nullptr;
```

这个不能算是成员变量，没有声明在类内。这个变量确保了每一个线程独有一个 `BackendLogger`实例

#### 日志记录类

##### LogRecord

所有记录的父类，没有特殊用法

有两个重要的函数和对应的成员变量，会在其子类中进行使用和初始化。供给BakendLogger持久化使用。

```cpp
// serialized message
char *message = nullptr;
char *GetMessage(void) const { return message; }

// length of the message
size_t message_length = 0; 
size_t GetMessageLength(void) const { return message_length; }
```

##### TupleRecord

```cpp
class TupleRecord : public LogRecord, Printable {
```

tuple类型的记录。有Insert，Update，Delete 三种类型，在存储时将记录序列化，并传入 `BakendLogger`用于存储。

**记录内容：**

* 头部

```cpp
long(log_record_type)
int(header_size) // 记录头部的长度
long(db_oid)
long(table_oid)
long(cid)
long(insert_location.block) // 只有Insert类型的该数据是有效数据
long(insert_location.offset) // 只有Insert类型的该数据是有效数据
long(delete_location.block) // 只有Delete类型的该数据是有效数据
long(delete_location.offset) // 只有Delete类型的该数据是有效数据
```

* 内容

```cpp
// 只有Insert 和 Update 记录需要存入tuple
Serialized(Tuple)
```

![img30.jpg](https://img-blog.csdnimg.cn/img_convert/8e9b80519add876009cb2a1d719cf1f9.png)

**主要成员函数**

> ***Serialize()***

```cpp
bool Serialize(CopySerializeOutput &output);
```

参数：一个帮助序列化且暂存信息的类

功能：将记录序列化并装入数组中（`message`成员变量），在BakendLogger中被获取并装入LogBuffer。

* 其中头部的序列化是调用 `SerializeHeader()`函数完成。
* 只有Update类型的record记录了tuple的数据（实际上Insert类型也需要记录，但代码逻辑中并没有实现）

**主要成员变量**

> ***message***
> 
> ***message_length***

```cpp
// serialized message
  char *message = nullptr;

  // length of the message
  size_t message_length = 0;
```

父类中声明的变量，用于存储序列化之后的数据，用于存入Log。

---

##### TransactionRecord

```cpp
class TransactionRecord : public LogRecord, Printable {
```

事务记录，只有在 `LogManager::LogBeginTransaction()`和 `LogManager::LogCommitTransaction()`中构造此类型的记录，也就事务开始log和结束log的时候使用。

**记录内容：**（相当于只有头部）

```cpp
long(log_record_type)
int(header_size) // 记录头部的长度
long(cid)
```

**主要成员函数和变量同TupleRecord大同小异，不做赘述。**

---

#### 内存池类

##### LogBuffer

日志缓存池（`CircularBufferPool`）的一个单位，大小为128KB。

实例化后，被 `BakendLogger`获取，用于装入日志记录。

![img69.jpg](https://img-blog.csdnimg.cn/img_convert/410e9e11fc0a8bfb04fba029ae03e0ee.png)

**主要成员函数**

> *GetData()*

```cpp
char *GetData() { return elastic_data_.get(); }
```

获取数据时只能获取序列化后存储的整块内存，没有提供  `De-serialized`功能。

> *WriteRecord()*

```cpp
bool WriteRecord(LogRecord *);
```

写入数据时同样传入序列化记录，依次排列在空闲内存后。

当此buffer满了，返回 `false`，`Log()`函数会尝试重新获取一个空闲的buffer。

**主要成员变量**

```cpp
size_t size_;
size_t capacity_;
```

最大容量，根据构造函数得出最大容量大小为128KB

```cpp
std::unique_ptr<char[]> elastic_data_;
```

存储log数据的数组，大小跟 `capacity_`相同

注意，这个 `capacity_`可能会变化：在写入第一条数据时，此数据就大于 `capacity_`，会将其乘2。

```cpp
cid_t max_log_id;
```

目前已经写入了的最大的 transaction 的 id ，在 `BakendLoger::Log()`中通过调用 `LogBuffer::SetMaxLogId()`对其进行设置。

---

##### BufferPool

日志缓存池的抽象父类，没有特殊作用

---

##### CircularBufferPool

```
class CircularBufferPool : public BufferPool {
```

日志缓存池的实现，在 `BackendLogger`中会实例化为 `BackendLogger::available_buffer_pool_`和 `BackendLogger::persist_buffer_pool_`，用于缓存日志记录。

可看作是 `LogBuffer`组成的循环队列，从尾部获取空闲buffer，头部装入满buffer，读写/序列化的细节不由缓存池提供。

![img70.jpg](https://img-blog.csdnimg.cn/img_convert/f3cec88161d9e0ee14f566d3c8c99f72.png)

**主要成员函数**

```cpp
// put a buffer to buffer pool. blocks if over capacity
bool Put(std::unique_ptr<LogBuffer>);

// get a buffer from buffer pool. blocks if none available
std::unique_ptr<LogBuffer> Get();
```

这两个函数相当于队列的出队，入队。

**主要成员变量**

```cpp
std::unique_ptr<LogBuffer> buffers_[BUFFER_POOL_SIZE];
std::atomic<unsigned int> head_;
std::atomic<unsigned int> tail_;
```

#### 日志器类

##### Logger

日志类型的抽象父类，没有特殊作用

五种日志模式：

```cpp
// 1. Standby -- Bootstrap
// 2. Recovery -- Optional
// 3. Logging -- Collect data and flush when commit
// 4. Terminate -- Collect any remaining data and flush
// 5. Sleep -- Disconnect backend loggers and frontend logger from manager
```

---

##### BackendLogger

用于记录日志记录，将记录写入内存中的log_buffer_pool。

```cpp
class BackendLogger : public Logger {
  friend class FrontendLogger;
```

![img37.jpg](https://img-blog.csdnimg.cn/img_convert/ecefe5ef11db440d918796b33a189e7c.png)

**主要成员函数**

> ***Log()***

```cpp
void Log(LogRecord *record);
```

*参数*： `record`：封装好的日志记录类

*功能：* 将日志记录写道内存缓存池中。

* 从 `available_buffer_pool_`获取一个空闲 `log_buffer_`，将传入的日志记录写入 `log_buffer_`，如果此buffer满了，则将其推送到 `persist_buffer_pool_`中。
* 如果是commit类型的record，要求其cid（txn_id）必须大于目前已提交记录的最大cid。
* 将目前最大的cid更新入 `log_buffer_`，FrontedLogger将用于计算 `max_flushed_cid`的值。

> ***GetBackendLogger()***

```cpp
static BackendLogger GetBackendLogger(LoggingType logging_type);
```

*参数*： `logging_type`：目前只实现了WAL（向前写日志）

*功能*：返回一个新的  `WriteAheadBackendLogger` 实例（此类公共继承了 `BackendLogger`，拥有其所有公用函数）

> ***PrepareLogBuffers()***

```cpp
std::pair<cid_t, cid_t> PrepareLogBuffers();
```

*功能*：将 `persist_buffer_pool_`中的内容推送到 `local_queue`内，`FrontedLogger`会从中获取，用于刷新到磁盘。

* 如果 `log_buffer_`不为空，则将其的内容也附加到 `local_queue`内。

*返回*：返回一对commit_id，第一个是记录器可以提交的值的下界，第二个是这个Logger已经提交过的的最大cid

> ***GetLogBuffers()***

```cpp
std::vector<std::unique_ptr<LogBuffer>> &GetLogBuffers() { return local_queue; }
```

*功能*：供给 `FrontedLogger`获取准备好的日志记录。

> ***GrantEmptyBuffer()***

```cpp
void GrantEmptyBuffer(std::unique_ptr<LogBuffer> empty_buffer)
```

*功能*：`FrontedLogger`通过此函数给 `available_buffer_pool_`传入新的LogBuffer页。

**主要成员变量**

> ***log_buffer_***

```cpp
std::unique_ptr<LogBuffer> log_buffer_;
```

指向当前 `LogBuffer`的指针，用于在 `Log()`函数中装入信息。

> ***available_buffer_pool_***

> ***persist_buffer_pool_***

```cpp
std::unique_ptr<BufferPool> available_buffer_pool_;
std::unique_ptr<BufferPool> persist_buffer_pool_;
```

`available_buffer_pool_`：当前可用的空闲池，`Log()`从其中获取空闲的 `LogBuffer`单位（用上述的 `log_buffer_`指向此单位）
`persist_buffer_pool_`： 当 `log_buffer_`满了之后，将其推送至此。

注意：新写入的log记录，要被推送入了 `persist_buffer_pool_`，要么还留在 `log_buffer_`，`available_buffer_pool_`中永远是空闲的LogBuffer队列，不会写入内容。

> ***local_queue***

```cpp
// temporary local_queue used by backend
std::vector<std::unique_ptr<LogBuffer>> local_queue;
```

调用 `PrepareLogBuffers()`时将 `persist_buffer_pool_`中的内容推送至此，`FrontendLogger`通过 `BakendLogger::GetLogBuffers()`获取其中的内容用于刷新到磁盘。

> ***highest_logged_commit_message***

```cpp
cid_t highest_logged_commit_message;
```

在Log()中被更新，存储了目前记录过的**已经提交过的**最大transaction_id即commit_id。

* 在 `PrepareLogBuffers()`中被使用，是构建返回值的变量之一，返回给 `FrontendLogger`使用。

---

##### FrontendLogger

单独作为一个线程，设定好固定休眠时间间隔，每次醒来 ，将log_buffer_pool中的日志记录flush到磁盘。

```cpp
class FrontendLogger : public Logger {
```

![img50.jpg](https://img-blog.csdnimg.cn/img_convert/e6178a8bd07d5eb09d5498efc41b7e9a.png)

**主要成员函数**

> ***MainLoop()   🚀️***

```cpp
void MainLoop(void)
```

*功能*：让FrontendLogger进入循环模式，设定好固定休眠时间间隔，每次醒来将log_buffer_pool中的日志记录flush到磁盘。

> ***CollectLogRecordsFromBackendLoggers()***

```cp
void CollectLogRecordsFromBackendLoggers(void);
```

*功能*：将所有BackendLogger中的 `persist_buffer_pool_`中的内容通过 `BackendLogger::PrepareLogBuffers()`收集起来。

**主要成员变量**

> ***wait_timeout***

```
int wait_timeout;
```

每次 `CollectLogRecordsFromBackendLoggers()`需要等待的时间（默认为0）

> ***max_seen_commit_id***
> 
> ***max_collected_commit_id***

```cpp
cid_t max_seen_commit_id;
cid_t max_collected_commit_id;
```

在只有单个FrontendLogger的情况下，这两个值的大小相等，都等于：目前FrontendLogger收集到的最大的已经提交了的commit_id，即从BackendLogger中获取的 `highest_logged_commit_message`。（之所以强调收集到的，是因为BakendLogger是单独的线程， `highest_logged_commit_message`随时可能更新）

> ***max_flushed_commit_id***

```
cid_t max_flushed_commit_id;
```

每次将缓冲区内容刷新到磁盘后，将此变量更新为 `max_seen_commit_id`。

##### WriteAheadFrontendLogger

![img55.jpg](https://img-blog.csdnimg.cn/img_convert/5243144a8d507aa31d3edb58f74f111b.png)

> ***FlushLogRecords()***

```cpp
void FlushLogRecords(void)
```

*功能*：将日志缓存池内的数据全部刷新到硬盘上。缓存池的内容是从BakendLogger中的 `persist_buffer_pool_`中取得。

##### WriteAheadBackendLogger

`BackendLogger` 的子类，无特殊用法，BakendLogger实例化时用的是此子类。

只封装了两个构造Tuple的函数，其中作用只是在日志记录类型上加上WAL标记。
