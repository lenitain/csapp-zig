# CSAPP 第八讲：异常与进程——操作系统的入口

## 物理原理：硬件层面的中断与异常

CSAPP 把异常分四类，本质是**硬件检测到事件，强制跳到 OS 的处理代码**：

| 类型 | 原因 | 同步/异步 | 返回行为 |
|------|------|-----------|----------|
| 中断 | 外部设备（网卡、磁盘） | 异步 | 返回下一条指令 |
| 陷阱 | 程序主动（syscall） | 同步 | 返回下一条指令 |
| 故障 | 可恢复错误（缺页） | 同步 | 重试或终止 |
| 终止 | 不可恢复错误 | 同步 | 不返回 |

**关键事实**：硬件中断时，CPU 切换到内核模式，跳到中断描述符表 (IDT) 里的处理程序。
这件事是**硬件物理实现**——不是 OS 的发明，是 CPU 设计的必然。

### 进程是 OS 的运行时抽象

进程是"一个正在运行的程序"。每个进程有自己的：

- 虚拟地址空间
- 程序计数器
- 寄存器状态
- 文件描述符表

OS 通过**上下文切换**让多个进程"同时"运行：保存状态、加载另一个、恢复。**这件事本质是
PCB (Process Control Block) 的保存和加载**——硬件不强制，是 OS 设计的选择。

## C 的设计选择：fork + exec + errno + signal

### fork 的天才设计

```c
pid_t pid = fork();
if (pid == 0) {
    // 子进程
    execve(...);
} else {
    // 父进程
    waitpid(pid, ...);
}
```

`fork()` 复制当前进程（地址空间、文件描述符……）。`execve()` 用新程序替换地址空间。

为什么是 fork + exec 而不是直接 `create + exec`？因为 fork 把"创建进程"和"执行新程序"分开
了，能实现任意进程模式（后台、shell 管道、守护进程）。

### errno：全局错误状态

```c
FILE *f = fopen("file.txt", "r");
if (f == NULL) {
    printf("错误: %s\n", strerror(errno));
}
```

C 用全局 `errno` 表示具体错误。问题：

- **全局变量**：多线程里每个线程要自己的 errno（用 `__errno_location()` 实现）
- **容易忘**：C 不强制检查返回值
- **不丰富**：errno 是整数，没有上下文

### 信号：异步事件通知

```c
signal(SIGINT, handler);  // Ctrl-C 触发
signal(SIGCHLD, reap);    // 子进程退出触发
```

信号是 OS 给进程的"异步通知"。问题是**异步**——handler 任何时候都可能执行，必须用
async-signal-safe 函数。**回调地狱的早期形态**。

```c
void handler(int sig) {
    // 不能调用 printf、malloc 等不安全函数
    write(1, "got signal\n", 11);  // 只能用 write
}
```

## 现代视角：错误和并发是类型的一部分

### 类型化 Result，错误必须处理

```rust
fn read_file(path: &str) -> Result<String, io::Error> {
    let mut file = std::fs::File::open(path)?;  // ? 传播错误
    let mut s = String::new();
    file.read_to_string(&mut s)?;
    Ok(s)
}
```

Rust 的 `Result<T, E>` 让错误是**返回值的一部分**。`?` 强制你处理错误——忘处理就编译错误。

C 的 `if (ret < 0) return ret;` 是手动检查。**类型系统强制的事** vs **人脑要记的事**——

### 没有 errno，没有全局状态

Rust/Zig 没有 `errno`。每个 syscall 都返回 `Result<T, Errno>`——错误在返回值里，不在
全局状态里。多线程不需要 thread-local errno。

### Async runtime 替代信号回调

```rust
// tokio async runtime
async fn handle_signal() {
    tokio::signal::ctrl_c().await.unwrap();
    println!("got SIGINT");
}
```

Rust 的 async runtime 让信号处理是**结构化并发**的一部分——不是信号回调，不是回调地狱。
OS 信号映射成 future，整个程序是一个状态机。

### defer 替代手动清理

```c
// C：每个 return 前都要检查
int foo() {
    int fd = open(...);
    if (error1) { close(fd); return -1; }
    char *buf = malloc(...);
    if (error2) { free(buf); close(fd); return -1; }
    // ...
}
```

```rust
// Rust：defer 1 行搞定
fn foo() -> Result<(), Error> {
    let fd = open(...)?;
    let buf = vec![0u8; 1024];  // Vec 实现 Drop，离开作用域自动释放
    // ...
    Ok(())
}
```

`defer` 让资源清理**作用域绑定**。C 的每个 return 路径都要手动清理是**原理缺陷**。

## AI 时代怎么验证

**1. 让 AI strace 一个 shell**

```bash
$ strace -f -e trace=clone,execve,wait4 bash -c "ls /tmp"
```

让 LLM 解释：哪些 `clone` 创建子进程？哪些 `execve` 替换程序？哪些 `wait4` 等待？

**2. 让 AI 跑 SIGCHLD 追踪**

```c
// 写一个 parent fork child，然后让 AI strace
```

让 LLM 看 SIGCHLD 在什么时候触发，handler 的执行顺序。

**3. 让 AI 解释 fork + copy-on-write**

```bash
$ strace -e trace=clone,mmap bash -c "ls /tmp"
```

让 LLM 解释：fork 后父子进程地址空间怎么共享？copy-on-write 在哪一刻发生？

**4. 让 AI 验证 errno vs Result**

```c
// 写一个 C 程序，不检查 fopen 返回值，让 AI 跑 sanitizer
```

让 LLM 找未检查的返回值。重点：每个 libc 调用都可能失败——这是 C 抽象机制的问题。

**5. 让 AI 对比 async runtime 和 pthread**

```bash
$ time ./pthread_program  // 10000 连接，每连接一个线程
$ time ./async_program    // tokio async
```

让 LLM 解释：为什么 async 程序内存占用低？状态机怎么建？callback hell 是怎么解决的？

## 本讲要点

1. 异常/陷阱/中断是硬件层面的事件分发——CPU 切到内核态跳 IDT
2. 进程 = PCB 抽象，OS 通过上下文切换实现"同时"运行
3. fork+exec 是 Unix 天才设计，errno 是全局错误的妥协
4. 信号回调地狱是异步编程的早期形态——async runtime 解决了它
5. Result 类型让错误处理编译期可见——C 的 errno 是历史遗留

## 下一讲

进程有自己的虚拟地址空间，但这个"虚拟"到底是怎么回事？下一讲看虚拟内存——操作系统如何用有限
的物理内存骗过每个进程，让它们以为自己独占全部内存。