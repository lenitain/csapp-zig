# CSAPP 第十二讲：并发编程

## 物理原理：多核 + cache 一致性

**为啥需要并发？** 两个原因：

1. **I/O 等待**：程序等磁盘/网络时，CPU 闲着
2. **多核利用**：现代 CPU 多核，串行程序只能用一个核心

**并发 vs 并行**：

- **并发**：逻辑上同时（context switch）
- **并行**：物理上同时（多 CPU 核心）

现代 CPU 多核 + 超标量 + 乱序，硬件并行能力是**物理存在**——你不利用就浪费。

### 共享内存 + cache 一致性

线程共享进程的地址空间：

```
代码段    (共享)
堆        (共享)
线程 A 栈 (私有)
线程 B 栈 (私有)
```

**关键事实**：多核各自有 L1/L2 cache。如果 core A 修改 `x`，core B 的 cache 还是旧值。
**MESI 协议** 同步 core 之间的 cache——写时 invalidate，读时 check invalid。

**伪共享** (false sharing)：两个不相关的变量在同一个 cache line 里，core A 写一个，core B 写另一个
——它们互相把对方的 cache line 弄无效。性能暴跌。

## C 的设计选择：pthread 没有 RAII，数据竞争是 UB

### 数据竞争是 UB

```c
// 两个线程同时执行
counter++;  // 三步：读、加、写
            // 如果两个线程同时执行，结果可能少加一次
```

**数据竞争**：多线程同时访问同一位置，至少一个写，无同步。在 C/C++ 里是 UB。

`counter++` 不是原子的——编译器**假设不会发生数据竞争**，优化可能让事情更糟。

### pthread 无 RAII

```c
void critical_section() {
    pthread_mutex_lock(&mutex);
    if (error) {
        return;  // 忘了 unlock——死锁
    }
    pthread_mutex_unlock(&mutex);
}
```

C 的 mutex **没有作用域绑定**——你必须在每个 return 前 unlock。这件事跟 malloc/free
的问题是同一个——手动管理容易出错。

### `void*` 参数类型擦除

```c
void *worker(void *arg) {
    int *data = (int *)arg;  // 强转
    // ...
    return NULL;
}
pthread_create(&thread, NULL, worker, &data);
```

pthread 的 worker 收 `void*`，调用方要传 `&data`、worker 强转回 `int*`——**类型擦除**，
编译器不会检查类型匹配。

## 现代视角：defer、Send/Sync、async

### defer 让锁的释放是作用域的

```rust
let mutex = Mutex::new(0);

{
    let mut guard = mutex.lock().unwrap();  // 锁
    // 临界区
    *guard += 1;
    // guard 离开作用域自动 unlock
}
```

`MutexGuard` 实现 `Drop`，离开作用域自动 unlock。**Rust 编译器静态证明没有"忘 unlock"**。

C 的 pthread mutex **没有 RAII**——你必须在每个 return 前 unlock。**这件事是 C 抽象机制的
原理问题**——资源管理不是类型层可见的。

### Send/Sync 是类型的一部分

```rust
fn spawn<T: Send>(t: T) { ... }   // T 必须能跨线程
fn share<T: Sync>(t: &T) { ... } // T 必须能跨线程共享
```

Rust 的 `Send`/`Sync` 是**自动派生的 trait**——如果你没实现，编译器会报错。

```rust
let rc = std::rc::Rc::new(5);  // Rc 不是 Send
std::thread::spawn(move || {
    println!("{}", rc);  // 编译错误！Rc 不能跨线程
});
```

**这件事是 C 抽象做不到的**——`Rc*` 在 C 里就是指针类型，能不能跨线程是程序员记的事。
Rust 让"能否跨线程"成为**类型系统**的事。

### async 把并发从线程中分离

```rust
async fn handle_client(socket: TcpStream) {
    let mut buf = [0; 1024];
    socket.read(&mut buf).await.unwrap();
    // ...
}
```

async fn 返回 `Future`——**不是立即执行，是可以被 runtime 挂起/恢复**。10000 并发连接 =
10000 个 Future = 1 个 OS thread = 1 个 stack frame。

**这是物理能力的现代利用**——OS 提供 epoll/kqueue，async runtime 调度 Future 状态机，
你不用手写状态机。

### Atomic 操作代替锁

```rust
use std::sync::atomic::{AtomicU32, Ordering::Relaxed};
static COUNTER: AtomicU32 = AtomicU32::new(0);

COUNTER.fetch_add(1, Ordering::Relaxed);
```

对于简单操作，atomic 比 mutex 快——**atomic 是硬件支持的不可中断操作**（CAS 指令）。

## AI 时代怎么验证

**1. 让 AI 跑 ThreadSanitizer 找数据竞争**

```bash
$ clang -fsanitize=thread t.c
$ ./your_program
```

让 LLM 解释：哪些指令是竞争的？竞争的两个线程调用栈是什么？什么同步原语能解决？

**2. 让 AI 跑 perf c2c 看 false sharing**

```bash
$ perf c2c record ./your_program
$ perf c2c report
```

让 LLM 解释：哪些地址是 false sharing？每个地址的 contention 结构是什么？怎么 padding 解决？

**3. 让 AI 解释 MESI 状态机**

```bash
# 让 AI 画 MESI 状态机，解释每个状态转换的触发事件
```

让 LLM 解释：什么时候 M→E，什么时候 E→M？写 invalidation 怎么广播？

**5. 让 AI 对比 pthread 和 tokio**

```bash
# 10000 并发 echo server：
# - pthread 10000 线程
# - tokio 1 线程 + 10000 Future
```

让 LLM 解释：tokio 为什么内存占用低？Future 是怎么存根的？awake 是怎么恢复的？

**6. 让 AI 找死锁**

```bash
# 写一个 pthread 互锁程序，让 AI 分析
```

让 LLM 解释：哪些线程锁了哪些锁？等待链是什么？怎么打破死锁？

## 本讲要点

1. 多核 + cache 一致性是硬件物理能力——不利用就浪费
2. 数据竞争是 C 的 ABI 定义，编译器信任你不发生竞争——发生了是 UB
3. pthread 没有 RAII，C 的 mutex 释放是程序员记的事
4. Rust 的 Send/Sync 让"能否跨线程"成为类型系统的事
5. async runtime 把"状态机"从程序员手里抢过来——不用手写 Future
6. AI 时代 perf c2c、ThreadSanitizer 是 debug 并发的必备工具

## 总结

这十二讲覆盖了 CSAPP 的全部内容：**数据表示、汇编、处理器体系结构、程序优化、内存层级、链接、
异常控制流、虚拟内存、系统级 I/O、网络编程、并发编程。**

每一讲我们走了同一个结构：

| 环节 | 你看到什么 |
|------|------------|
| 物理原理 | CPU 是数字逻辑、存储有速度差、隔离需要硬件支持 |
| C 的设计选择 | 历史阶段的工具拼接——`#include`、malloc/free、errno、void* |
| 现代视角 | 模块系统、defer、Result 类型、Send/Sync 替代 C 的隐藏状态 |
| AI 验证 | 让 LLM 跑 perf、strace、sanitizer、cachegrind |

**这套结构传达的不是"哪个方法更好"——而是"原理不变，载体在变"。**

**C 的设计选择是历史的产物**——`#include` 因为 1972 年没有模块系统，`malloc/free` 因为
1972 年没有时间预算做 GC。理解它们不是要学它们，是为了接触 OS/ABI 这些必读 C 的地方时不被
绊倒。

**现代工具暴露 C 隐藏的东西**——Rust/Zig 让 UB、隐式分配、生命周期这些成为编译期错误，
不是为了取代 C，是为了让你看清 C 哪里在"悄悄做事"。

**AI 干脏活，你定方向**——反汇编、跑 trace、生成测试用例、读 stack——这些都丢给 LLM。
你做的是判断它说的对不对、看它漏了什么 case。

**系统心智模型不会过时**，因为它们是物理现实的反映。AI 可以写代码，但理解代码背后的系统，
是你的不可替代的能力。