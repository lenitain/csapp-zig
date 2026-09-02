# CSAPP 第十二讲：并发编程

## 为什么需要并发？

程序需要并发的原因有两个：

1. **I/O 等待**：程序在等待磁盘或网络时，CPU 闲着。如果有多个任务，可以让它们交替执行。
2. **多核利用**：现代 CPU 有多个核心，串行程序只能用一个核心。并发程序可以利用所有核心。

并发 (concurrency) 和并行 (parallelism) 是不同的概念：

- **并发**：逻辑上的"同时执行"，可以由操作系统模拟（上下文切换）
- **并行**：物理上的"同时执行"，需要多个处理器核心

## 三种并发模型

### 1. 多进程

每个进程有独立的地址空间，通过 IPC（管道、共享内存、消息队列）通信。

优点：进程隔离，一个进程崩溃不影响其他进程。缺点：进程创建和切换开销大，IPC 复杂。

```c
pid_t pid = fork();
if (pid == 0) {
    // 子进程处理一个任务
    handle_request(request);
    exit(0);
} else {
    // 父进程继续接受新请求
}
```

### 2. 多线程

线程共享地址空间，通过共享内存通信。

优点：线程创建和切换开销小，共享内存通信简单。缺点：共享内存意味着一个线程的 bug
可能影响其他线程。

```c
pthread_t thread;
pthread_create(&thread, NULL, worker, &arg);
pthread_join(thread, NULL);
```

### 3. I/O 多路复用

一个线程同时监视多个文件描述符，哪个就绪就处理哪个。

优点：单线程，没有竞态条件。缺点：编程模型复杂（回调地狱）。

```c
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(fd1, &readfds);
FD_SET(fd2, &readfds);
select(maxfd + 1, &readfds, NULL, NULL, NULL);
```

## 线程的内存模型

线程共享进程的地址空间。这意味着：

- 全局变量：所有线程共享
- 堆内存：所有线程共享（malloc 的内存）
- 栈：每个线程有自己的栈
- 寄存器：每个线程有自己的寄存器状态

```text
进程地址空间：
+------------------+
| 代码段 (共享)     |
+------------------+
| 数据段 (共享)     |
+------------------+
| 堆 (共享)         |
+------------------+
| ...              |
+------------------+
| 线程 A 的栈       |
+------------------+
| 线程 B 的栈       |
+------------------+
```

## 竞态条件：并发的核心问题

```c
// 两个线程同时执行
counter++; // 看起来是一条语句，实际上是三步：
           // 1. 读取 counter 到寄存器
           // 2. 寄存器加 1
           // 3. 写回 counter
```

如果两个线程同时执行这三步，结果可能少加一次。这就是**竞态条件** (race condition)。

竞态条件的根源：**多个线程同时访问共享数据，且至少有一个是写操作。**

### 数据竞争 vs 竞态条件

- **数据竞争** (data race)：多个线程同时访问同一内存位置，且至少有一个是写操作，且没有同步
- **竞态条件** (race condition)：程序的结果取决于线程的执行顺序

数据竞争是未定义行为（在 C 和 C++ 中）。竞态条件是逻辑错误。

## 互斥锁：保护共享数据

解决方案：**互斥锁** (mutex)。在访问共享变量前加锁，访问完解锁。

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
int counter = 0;

void *worker(void *arg) {
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&mutex);
        counter++;
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}
```

### C 的互斥锁没有作用域

```c
void critical_section() {
    pthread_mutex_lock(&mutex);
    // ... 临界区 ...
    if (error) {
        return;  // 忘了 unlock，死锁！
    }
    pthread_mutex_unlock(&mutex);
}
```

C 的 `pthread_mutex_lock/unlock` 没有 RAII 机制。如果临界区中间有 return，你必须记得在每个
return 前 unlock。这和 malloc/free 的问题一样——手动管理容易出错。

### Zig 的态度：defer 解决一切

```zig
var mutex = std.Thread.Mutex{};
var counter: u32 = 0;

fn worker() void {
    var i: u32 = 0;
    while (i < 1000000) : (i += 1) {
        mutex.lock();
        defer mutex.unlock(); // 作用域结束时自动解锁
        counter += 1;
    }
}
```

`defer` 让锁的释放和作用域绑定，不需要手动管理。这和内存管理的 `defer` 是同一个模式——Zig 用
`defer` 统一了所有资源清理。

```zig
// 即使有多个 return 路径，defer 也能正确清理
fn complexFunction() void {
    mutex.lock();
    defer mutex.unlock();

    if (condition1) return; // defer 自动解锁
    if (condition2) return; // defer 自动解锁
    // ... 临界区 ...
}
```

## 条件变量：线程间的通信

互斥锁只解决了"互斥"问题。有时候线程需要"等待某个条件成立"：

```c
// 生产者-消费者模型
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int buffer = 0;
int ready = 0;

// 生产者
void producer() {
    pthread_mutex_lock(&mutex);
    buffer = 42;
    ready = 1;
    pthread_cond_signal(&cond);  // 通知消费者
    pthread_mutex_unlock(&mutex);
}

// 消费者
void consumer() {
    pthread_mutex_lock(&mutex);
    while (!ready) {
        pthread_cond_wait(&cond, &mutex);  // 等待条件
    }
    printf("Got: %d\n", buffer);
    pthread_mutex_unlock(&mutex);
}
```

`pthread_cond_wait()` 释放锁，等待信号，然后重新获取锁。这避免了忙等待（busy waiting）。

### Zig 的条件变量

```zig
var mutex = std.Thread.Mutex{};
var condition = std.Thread.Condition{};
var buffer: i32 = 0;
var ready: bool = false;

fn producer() void {
    mutex.lock();
    defer mutex.unlock();
    buffer = 42;
    ready = true;
    condition.signal(); // 通知消费者
}

fn consumer() void {
    mutex.lock();
    defer mutex.unlock();
    while (!ready) {
        condition.wait(&mutex); // 等待条件
    }
    std.debug.print("Got: {}\n", .{buffer});
}
```

## 死锁：并发的另一个陷阱

如果线程 A 持有锁 1 等待锁 2，线程 B 持有锁 2 等待锁 1，两个线程永远等下去。这就是**死锁**
(deadlock)。

```text
线程 A: lock(1) → 等待 lock(2)
线程 B: lock(2) → 等待 lock(1)
→ 死锁
```

### 死锁的四个必要条件

1. **互斥**：资源不能同时被多个线程使用
2. **持有并等待**：线程持有资源，同时等待其他资源
3. **不可抢占**：已获得的资源不能被强制释放
4. **循环等待**：存在线程的循环等待链

### 避免死锁

最简单的方法：**所有线程按相同的顺序获取锁。**

```c
// 总是先锁 1，再锁 2
pthread_mutex_lock(&mutex1);
pthread_mutex_lock(&mutex2);
// ...
pthread_mutex_unlock(&mutex2);
pthread_mutex_unlock(&mutex1);
```

## C 的 pthread：历史包袱

C 语言最初没有线程支持。POSIX 线程 (pthread) 是后来标准化的，API 设计有历史包袱：

```c
// void* 参数，类型安全完全丧失
void *worker(void *arg) {
    int *data = (int *)arg; // 强制转换
    // ...
    return NULL;
}

pthread_t thread;
int arg = 42;
pthread_create(&thread, NULL, worker, &arg);
```

### Zig 的线程：类型安全

```zig
fn worker(data: i32) void {
    // data 是类型安全的，不需要强制转换
}

const thread = try std.Thread.spawn(.{}, worker, .{42});
thread.join();
```

Zig 的 `std.Thread.spawn` 接受具体的函数和参数类型，编译器会检查类型匹配。

## 原子操作：比锁更轻量的同步

对于简单的操作（比如递增计数器），互斥锁太重了。原子操作可以在硬件层面保证操作的原子性，不需要锁。

```c
#include <stdatomic.h>
atomic_int counter = 0;

void *worker(void *arg) {
    for (int i = 0; i < 1000000; i++) {
        atomic_fetch_add(&counter, 1); // 原子递增
    }
    return NULL;
}
```

### Zig 的原子操作

```zig
var counter: std.atomic.Value(u32) = std.atomic.Value(u32).init(0);

fn worker() void {
    var i: u32 = 0;
    while (i < 1000000) : (i += 1) {
        _ = counter.fetchAdd(1, .seq_cst); // 原子递增
    }
}
```

## 本讲要点

1. 并发有三种模型：多进程、多线程、I/O 多路复用
2. 竞态条件是并发的核心问题：多个线程同时访问共享数据
3. 互斥锁保护共享数据，但 C 的 pthread 没有 RAII，手动管理容易出错
4. Zig 的 defer 让锁的释放和作用域绑定，更安全
5. 条件变量让线程可以等待条件成立，避免忙等待
6. 死锁是并发的另一个陷阱，避免方法是按固定顺序获取锁
7. 原子操作比锁更轻量，适合简单的操作

## 总结

这十二讲覆盖了 CSAPP
的全部内容：数据表示、汇编、处理器体系结构、程序优化、内存层级、链接、异常控制流、虚拟内存、系统级
I/O、网络编程、并发编程。

每讲我们都对比了 C 和 Zig 的设计选择：

| 主题 | C 的问题 | Zig 的解决方案 |
|------|----------|---------------|
| 数据表示 | 隐式类型转换，UB | 显式转换，确定行为 |
| 缓冲区溢出 | gets() 不检查边界 | slice 带长度，边界检查 |
| 编译器优化 | UB 给编译器"合法伤害权" | 没有 UB，行为确定 |
| 内存管理 | malloc/free 手动管理 | defer + Allocator 接口 |
| 错误处理 | errno 全局变量 | 错误是类型的一部分 |
| I/O 缓冲 | 隐藏的内存分配 | 显式缓冲 |
| 网络编程 | 类型不安全，大小端易忘 | 类型安全，自动处理 |
| 并发 | pthread 无 RAII | defer 统一资源清理 |

这些对比不是为了说"C 不好，Zig
好"。而是为了让你理解：**C 的设计选择是历史的产物，Zig 的设计选择是对 C 问题的修正。**
理解这些选择，你才能理解系统是怎么工作的，以及为什么新工具会这样设计。

系统心智模型不会过时，因为它们是物理现实的反映。AI
可以写代码，但理解代码背后的系统，是你的不可替代的能力。
