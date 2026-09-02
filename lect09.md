# CSAPP 第七讲：虚拟内存——一切的基石

## 为什么需要虚拟内存？

物理内存是有限的。一台机器可能只有 16GB 内存，但同时运行着几十个进程，每个进程都想用内存。

更根本的问题：如果所有进程共享物理内存，一个进程可以读写另一个进程的数据——这在安全上是不可接受的。

虚拟内存解决了这两个问题：

1. **隔离**：每个进程有自己的地址空间，A 进程的地址 0x1000 和 B 进程的地址 0x1000
   是不同的物理内存
2. **抽象**：进程以为自己有连续的、独占的内存，实际上物理内存可能四分五裂，甚至部分在磁盘上

## 地址翻译：虚拟 → 物理

进程访问的地址是**虚拟地址**。CPU 的内存管理单元 (MMU)
把它翻译成**物理地址**，然后访问真正的内存。

翻译过程用**页表** (page table)：虚拟地址空间被分成固定大小的"页" (page，通常
4KB)，每个页在页表里有一个条目，记录它对应的物理页帧 (page frame) 或者"不在内存中"。

```text
虚拟地址: [VPN | Offset]
              ↓
         页表查找
              ↓
物理地址: [PFN | Offset]
```

VPN (Virtual Page Number) 通过页表查到 PFN (Physical Frame Number)，偏移量不变。

## 多级页表：节省空间的技巧

如果用一个平坦的页表，64 位系统需要 2^52 个条目，每个条目 8 字节，总共需要 32PB
存储页表——这显然不现实。

解决方案：多级页表。把页表本身也分页，只分配实际使用的部分。

x86-64 用 4 级页表：

```text
虚拟地址: [PML4 | PDP | PD | PT | Offset]
              ↓     ↓    ↓    ↓
           4级查表，每级查一个4KB的页表
```

未使用的虚拟地址区域不需要分配页表页，节省了大量空间。但代价是：每次地址翻译需要 4
次内存访问。

这就是 TLB (Translation Lookaside Buffer)
存在的原因——它是页表的缓存，存储最近使用的虚拟→物理映射。TLB 命中时，地址翻译几乎零延迟。

## 缺页异常：按需分配

进程的虚拟地址空间可能很大（64 位系统上理论上是 2^64
字节），但物理内存有限。操作系统不需要一开始就分配全部物理内存。

当进程第一次访问某个虚拟页时，页表条目标记为"不在内存"，CPU
触发缺页异常。操作系统的缺页处理程序：

1. 检查访问是否合法（不是非法地址）
2. 在物理内存中找一个空闲页帧
3. 如果这页之前被换出到磁盘，从磁盘读回来
4. 更新页表，标记这页在内存中
5. 重新执行触发缺页的指令

这就是**按需分页** (demand paging)：只在需要时才分配物理内存。

## 内存映射：文件即内存

`mmap()`
系统调用把文件映射到进程的虚拟地址空间。之后，访问这块内存就像访问文件——读内存就是读文件，写内存就是写文件（如果映射是可写的）。

为什么这很有用？

- **大文件处理**：不需要把整个文件读到内存，只加载你访问的部分
- **共享内存**：多个进程映射同一个文件，一个进程的写入对其他进程可见
- **动态库加载**：.so 文件通过 mmap 加载到进程地址空间

```zig
const posix = std.posix;

// Zig 的 mmap 示例
const fd = try posix.open("data.bin", .{ .ACCMODE = .RDONLY }, 0);
defer posix.close(fd);

const data = try posix.mmap(
    null,
    4096,
    posix.PROT.READ,
    posix.MAP{ .TYPE = .PRIVATE },
    fd,
    0,
);
defer posix.munmap(data);
```

## C 的 malloc/free：内存安全的噩梦

C 的内存管理是手动的：你 malloc，你 free。这导致了大量经典 bug：

### 内存泄漏

```c
void leaky() {
    char *p = malloc(1024);
    if (error_condition) {
        return;  // 忘了 free(p)，内存泄漏
    }
    // ... 使用 p ...
    free(p);
}
```

每个 return 路径都必须 free。如果有多个资源，清理代码会变得极其复杂。

### Use-After-Free

```c
char *p = malloc(1024);
free(p);
// ... 一些代码 ...
p[0] = 'x';  // use-after-free：p 已经 free 了，但还在用
```

free 之后指针 p
的值没变（还是指向那块内存），但那块内存已经被回收了。访问它是未定义行为——可能工作，可能崩溃，可能被攻击者利用。

### Double Free

```c
char *p = malloc(1024);
free(p);
free(p);  // double free：同一块内存 free 两次，破坏堆元数据
```

### 野指针

```c
char *p = malloc(1024);
free(p);
p = NULL;  // 如果忘了这行，p 就是野指针
```

这些问题的根源是：**C 的指针在 free 之后还是"有效"的。**
类型系统不跟踪指针的生命周期，完全靠程序员自觉。

### Zig 的 defer：资源清理的正确方式

```zig
fn readFile() !void {
    const file = try std.fs.cwd().openFile("data.txt", .{});
    defer file.close(); // 作用域结束时自动关闭，不管怎么退出

    const data = try file.readToEndAlloc(allocator, 1024 * 1024);
    defer allocator.free(data); // 作用域结束时自动释放

    // ... 使用 data ...
    // 如果这里出错，defer 会自动清理 file 和 data
}
```

`defer` 确保资源在作用域结束时释放，不管函数是正常返回还是因为错误退出。你不需要在每个
return 路径都写清理代码。

```zig
fn leaky() !void {
    const p = try allocator.alloc(u8, 1024);
    defer allocator.free(p); // 一行搞定

    if (error_condition) return error.Something;
    // ... 使用 p ...
    // 退出时 defer 自动 free
}
```

### Zig 的 Allocator 接口：可替换的分配策略

C 的 `malloc()` 是全局的——整个程序用同一个分配器。如果你想换分配器（比如用 arena
分配器提高性能），得改代码。

```zig
// Zig 的分配器是传入的参数
fn process(allocator: std.mem.Allocator) !void {
    const data = try allocator.alloc(u8, 1024);
    defer allocator.free(data);
    // ...
}

// 可以用不同的分配器
var gpa = std.heap.GeneralPurposeAllocator(.{}){};
try process(gpa.allocator());

var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();
try process(arena.allocator());
```

Arena
分配器只分配不释放，最后一次性释放。这在"分配很多小对象，用完一起丢"的场景下性能极好。C
里实现这个需要手写分配器或用第三方库，Zig 里是标准库的一部分。

## 垃圾回收 vs 手动管理

C 和 Zig 都是手动内存管理：你分配，你释放。忘记释放就是内存泄漏，释放后还访问就是
use-after-free。

Go、Java、Python 等语言用垃圾回收 (GC)：自动检测不再使用的内存并释放。GC
消除了内存安全问题，但有性能开销和不可预测的暂停。

Zig 选择手动管理，但通过设计减少错误：

- `defer` 确保资源在作用域结束时释放
- `errdefer` 在错误路径上释放资源
- Allocator 接口让内存管理可测试、可替换

这不是说 GC
不好，而是说在系统编程领域，手动管理仍然是必要的——你需要知道内存什么时候分配、什么时候释放、花了多少时间。

## 本讲要点

1. 虚拟内存让每个进程以为自己独占全部内存
2. 页表是虚拟→物理翻译的核心，TLB 缓存加速翻译
3. 缺页异常实现按需分配，mmap 实现文件即内存
4. C 的 malloc/free 是内存安全的噩梦：泄漏、use-after-free、double free
5. Zig 的 defer 和 Allocator 接口让内存管理更安全、更灵活

## 下一讲

程序需要和外部世界交互——读写文件、收发网络数据。下一讲看系统级 I/O 和并发编程的入门——以及 C
的隐藏缓冲区为什么是一个设计得很糟糕的设计。
