# CSAPP 第十讲：系统级 I/O

## Unix 哲学：一切皆文件

Unix 的设计者 Ken Thompson 有一个天才的想法：**把所有 I/O 统一抽象成"文件"。**

- 普通文件：磁盘上的数据
- 目录：文件名到 inode 的映射
- 设备：/dev/sda、/dev/tty
- 管道：进程间通信
- 网络套接字：网络通信

它们都用同一套
API：`open()`、`read()`、`write()`、`close()`。你不需要知道"文件"背后是磁盘、终端还是网络连接。

这个抽象之所以强大，是因为它让**程序可以组合**：一个程序的输出可以作为另一个程序的输入（管道），而不需要它们知道对方的存在。

```bash
# 管道：一个程序的输出是另一个程序的输入
cat file.txt | grep "error" | wc -l
```

## 文件描述符：进程的 I/O 句柄

进程打开一个文件时，操作系统返回一个小的非负整数——**文件描述符** (file
descriptor)。之后所有的 I/O 操作都通过这个整数进行。

标准约定：

- 0：标准输入 (stdin)
- 1：标准输出 (stdout)
- 2：标准错误 (stderr)

文件描述符是进程私有的。进程 A 的 fd 3 和进程 B 的 fd 3 可能指向完全不同的文件。

### 文件描述符表

每个进程有一个文件描述符表，记录了所有打开的文件。这个表是进程状态的一部分，fork
时会被复制。

```text
进程的文件描述符表：
fd 0 → stdin (终端)
fd 1 → stdout (终端)
fd 2 → stderr (终端)
fd 3 → data.txt
fd 4 → socket
```

## read/write：最基本的 I/O

```c
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

`read()` 从 fd 读取最多 count 字节到 buf，返回实际读取的字节数。`write()` 把 buf 中的 count
字节写入 fd。

关键点：**`read()` 的返回值可能小于 count。**
这不是错误——可能是文件剩余数据不足，也可能是被信号中断。你必须在一个循环里处理短读 (short
read)：

```c
ssize_t n;
size_t total = 0;
while (total < count) {
    n = read(fd, buf + total, count - total);
    if (n <= 0) break; // 错误或 EOF
    total += n;
}
```

这个模式在 C 里非常常见。Zig 的 `reader.readAll()` 封装了这个循环。

## C 的缓冲 I/O：隐藏的复杂性

`read()`/`write()`
是**无缓冲**的——每次调用都触发一次系统调用。系统调用有固定的开销（用户态↔内核态切换），如果你一个字节一个字节地读，开销会非常大。

C 标准库的 `fread()`/`fwrite()`
是**有缓冲**的——它们在用户空间维护一个缓冲区，减少系统调用次数。

### 隐藏的内存分配

```c
FILE *f = fopen("data.txt", "r");
// fopen 内部分配了缓冲区（通常是 4KB 或 8KB）
// 你不知道它什么时候分配，什么时候释放
```

`fopen` 在堆上分配一个 `FILE` 结构体和缓冲区。你调用 `fclose` 时释放。但如果你忘了
`fclose`，内存就泄漏了。

### 隐藏的缓冲区状态

```c
printf("Hello");
// "Hello" 可能还在缓冲区里，没有输出到终端
// 因为 stdout 默认是行缓冲的，没有 '\n' 就不刷新

fflush(stdout); // 手动刷新
```

C 的 `stdout` 有三种缓冲模式：

- **全缓冲**：缓冲区满了才刷新（文件）
- **行缓冲**：遇到 '\n' 刷新（终端）
- **无缓冲**：立即刷新（stderr）

你必须知道当前是什么模式，否则输出顺序可能和你预期的不同。

```c
// 一个常见的 bug
printf("Enter your name: ");  // 没有 \n，可能不显示
scanf("%s", name);            // 用户看不到提示
```

### Zig 的态度：显式缓冲

```zig
// Zig 的 print 使用栈上的缓冲区，不分配堆内存
const stdout = std.io.getStdOut().writer();
try stdout.print("Hello, World!\n", .{});
```

Zig 的 `print` 使用栈上的缓冲区（或你传入的缓冲区），不触发隐藏的内存分配。你看到
`print`，就知道它不会分配堆内存。

```zig
// 需要缓冲？显式创建
var buf_writer = std.io.bufferedWriter(std.io.getStdOut().writer());
const writer = buf_writer.writer();
try writer.print("Hello", .{});
try writer.print("World\n", .{});
try buf_writer.flush(); // 显式刷新
```

缓冲是**你选择的**，不是隐藏的。你用 `bufferedWriter` 就有缓冲，不用就没有。

## 共享文件：一个容易出错的设计

多个进程可以同时打开同一个文件。Unix 的文件系统用**引用计数**管理文件：一个文件的 inode
记录有多少个文件描述符指向它，当引用计数归零时，文件才真正被删除。

但"同时写"是有问题的。如果两个进程同时写同一个文件的不同位置，结果是什么？取决于它们的写入顺序和原子性。

### 文件锁

POSIX 提供了文件锁机制：

```c
// 咨询锁（advisory lock）
flock(fd, LOCK_EX);  // 独占锁
// ... 操作文件 ...
flock(fd, LOCK_UN);  // 解锁
```

但文件锁是**劝告性锁** (advisory
lock)——只有所有进程都遵守规则才有效。一个不遵守规则的进程可以直接读写文件，绕过锁。

这是 Unix 设计哲学的一部分：**操作系统提供机制，不提供策略。**
你可以选择是否使用文件锁，操作系统不强制。

## 标准 I/O 的设计缺陷

C 的标准 I/O 库（stdio）有几个设计缺陷：

### FILE 是不透明的

```c
FILE *f = fopen("data.txt", "r");
// 你不能直接访问 f 的内部状态
// 不能直接设置缓冲区大小
// 不能直接获取文件描述符（需要用 fileno()）
```

### 缓冲模式不可预测

```c
// stdout 的缓冲模式取决于实现
// 有的系统在连接到终端时是行缓冲，否则是全缓冲
// 你不能依赖特定行为
```

### 线程不安全

```c
// 标准 I/O 的大部分函数不是线程安全的
// 需要用 flockfile/funlockfile 保护
```

### Zig 的设计：更透明

```zig
const file = try std.fs.cwd().openFile("data.txt", .{});
defer file.close();

// 文件句柄是具体类型，不是不透明指针
// 你可以直接访问内部状态
const fd = file.handle; // 获取文件描述符
```

## I/O 多路复用：一个线程处理多个连接

传统的并发模型：一个连接一个线程。但线程有开销（栈空间、上下文切换），如果有几万个连接，线程数就爆了。

解决方案：**I/O 多路复用**。一个线程同时监视多个文件描述符，哪个就绪就处理哪个。

```c
// select：最古老的多路复用
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(fd1, &readfds);
FD_SET(fd2, &readfds);
select(maxfd + 1, &readfds, NULL, NULL, NULL);
// 检查哪个 fd 就绪
```

Linux 的 `epoll`、macOS 的 `kqueue` 是更高效的实现。

对于高并发场景，I/O 多路复用 + 事件驱动是现代网络编程的标准模式。Node.js、Nginx、Redis
都用这个模型。

## 本讲要点

1. Unix 的"一切皆文件"抽象让程序可以组合
2. 文件描述符是进程 I/O 的句柄，read/write 是最基本的 I/O 操作
3. `read()` 可能返回少于请求的字节数，必须处理短读
4. C 的缓冲 I/O 隐藏了内存分配和缓冲区状态
5. Zig 的 I/O 是显式的：缓冲是你选择的，不是隐藏的
6. 文件锁是劝告性的，操作系统不强制
7. I/O 多路复用让一个线程处理多个连接

## 下一讲

文件是本地的数据，网络是远程的数据。下一讲看网络编程——以及 socket API
的设计为什么是"一切皆文件"哲学的自然延伸。
