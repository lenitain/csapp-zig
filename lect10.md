# CSAPP 第十讲：系统级 I/O

## 物理原理：系统调用贵 + 一切皆文件

**用户态 ↔ 内核态切换是贵的**。每次 syscall 涉及：保存用户态寄存器、切换到内核态、执行
内核代码、恢复用户态寄存器。**典型开销 100-1000 纳秒**。

磁盘 I/O 还要触发硬件 DMA（直接内存访问，CPU 不参与），延迟更高——SSD 几十微秒，HDD 几毫秒。

**Unix 的"一切皆文件"**是 Ken Thompson 的设计选择：把磁盘、终端、管道、网络套接都抽象成"字节
流 + fd"。让 `cat file.txt | grep "error"` 可以用同一套 API 组合。

```
fd 0: stdin (终端)
fd 1: stdout (终端)
fd 2: stderr (终端)
fd 3+: 用户打开的文件、socket、pipe
```

**这件事的物理基础**：所有 fd 都是内核的"打开文件表"里的索引，OS 用同一个 struct file 抽象
所有 I/O 源。

## C 的设计选择：stdio 缓冲 + 短读

### stdio 缓冲：隐藏的内存分配

```c
FILE *f = fopen("data.txt", "r");
// 内部 malloc 了 FILE 结构 + 4-8KB 缓冲区
// 你看不到什么时候分配
```

`FILE*` 第一次 `fread` 时分配内存。`fclose` 时释放。**你看不到这些事**。

更糟——三种缓冲模式不可预测：

- **全缓冲**：文件默认，缓冲区满才 flush
- **行缓冲**：stdout 连接到终端，**遇到 `\n` flush**
- **无缓冲**：stderr，永远立即写

```c
printf("Enter name: ");  // 没有 \n——可能不显示
scanf("%s", name);         // 用户看不到提示
```

你以为是代码 bug，其实是缓冲模式不可预测。

### 短读：read 不一定返回请求数

```c
char buf[4096];
ssize_t n = read(fd, buf, sizeof(buf));  // n 可能 < 4096
```

`read` 不保证返回请求数。可能是：

- 文件剩余 < 4096 字节
- 被信号中断
- 网络包边界

你必须 `while (total < count)` 循环。**这是 C 抽象机制的原理问题**——`read` 是个 syscall，
syscall 可以部分完成。

### 文件锁是劝告性的

```c
flock(fd, LOCK_EX);  // 强制独占锁
```

但 `flock` 是**劝告性锁**——只有大家都用 `flock` 才有效。一个进程直接读写普通文件，绕过锁。

**OS 提供机制，不强制**。你可以遵守 lock，你的工具不公平。

## 现代视角：显式缓冲，类型安全的 fd

### 显式 buffered writer

```rust
let file = std::fs::File::create("data.txt")?;
let mut writer = std::io::BufWriter::new(file);  // 显式 wrap
writer.write_all(b"hello")?;
writer.flush()?;  // 显式 flush
```

缓冲是**你选择的**——`BufWriter` 默认 8KB，要 wrap，移去是不缓冲。

`writer.flush()` 是显式的——你看到 `flush` 就知道数据被写出。C 的 `fflush` 是隐式+不可预测。

### `read_to_end` 封装短读循环

```rust
let mut data = Vec::new();
file.read_to_end(&mut data)?;  // 内部 while 循环
```

Rust 的 `read_to_end` 封装了"读直到 EOF"。C 的 `read` 必须手写 while。

### io_uring：绕过 syscall 开销

```rust
// io_uring 是 Linux 5.1+ 的新 syscall 机制
// 用户态/内核态共享 ring buffer，避免切换
```

io_uring 把多个 I/O 请求放到用户态环形队列，内核异步处理——避免 syscall 开销。**这是
物理限制的现代解法**——OS 暴露底层机制，让你绕过单 syscall 模式。

## AI 时代怎么验证

**1. 让 AI 跑 strace 看实际 syscall**

```bash
$ strace -e trace=read,write,open,close ./your_program
```

让 LLM 解释：哪些库函数触发哪些 syscall？`printf` 触发了多少次 `write`？为什么？

**2. 让 AI 对比 buffered vs unbuffered I/O**

```bash
$ strace -c ./program_with_stdio
$ strace -c ./program_with_syscall_only
```

让 LLM 解释：syscall 数量差多少？性能差距对应实际时间差多少？

**3. 让 AI 验证缓冲模式不可预测**

```c
// 写一个程序 stdout 重定向到 pipe，跑一下
// 同一个 printf 在终端 vs pipe 下的行为差异
```

让 LLM 解释：为什么 stdout 连接到 pipe 时变全缓冲？fprintf 在两种情况下的 syscall 序列是什么？

**5. 让 AI 跑 io_uring benchmark**

```bash
# 写一个 io_uring echo server，让 AI 对比性能
```

让 LLM 解释：io_uring 怎么绕过 syscall 开销？ring buffer 怎么工作？性能数据是什么？

## 本讲要点

1. 系统调用贵，文件描述符是内核"打开文件表"的索引
2. stdio 的隐藏缓冲、三种缓冲模式、短读都是 C 抽象机制的原理问题
3. 现代 I/O 显式 bufWriter、显式 flush、不缓冲默认
5. `io_uring` 是绕过 syscall 开销的现代解法——共享 ring buffer
6. AI 时代 strace 是必备工具——看实际 syscall 序列

## 下一讲

文件是本地的数据，网络是远程的数据。下一讲看网络编程——以及 socket API 的设计为什么是
"一切皆文件"哲学的自然延伸。