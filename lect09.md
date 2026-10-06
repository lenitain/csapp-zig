# CSAPP 第九讲：虚拟内存——一切的基石

## 物理原理：物理内存有限且需要安全隔离

物理内存是有限的。一台机器可能只有 16 GB 内存，但同时跑几十个进程，每个都想用内存。

**更根本的问题**：如果所有进程共享物理内存，一个进程能读写另一个的数据——安全不可接受。

虚拟内存解决两个问题：

1. **隔离**：每个进程有自己的地址空间
2. **抽象**：进程以为自己有连续的、独占的内存

这是**硬件+OS 的协同设计**：

- CPU 有 MMU（Memory Management Unit），自动翻译虚拟→物理
- OS 维护页表（page table），是 MMU 的"规则表"
- 缺页异常（page fault）是 MMU + OS 协同的事件——MMU 检测到"页不在内存"，触发异常，OS
  处理（加载页面或杀进程）

### 地址翻译

```
虚拟地址: [VPN | Offset]
              ↓
         MMU 查页表
              ↓
物理地址: [PFN | Offset]
```

VPN（Virtual Page Number）通过页表查到 PFN（Physical Frame Number），偏移量不变。

### 多级页表

如果用单级页表，64 位系统需要 2^52 个条目，每个 8 字节——总共 32 PB，**装不进内存**。

解决：**多级页表**。把页表本身也分页，只分配实际使用的部分。

x86-64 用 4 级页表：

```
虚拟地址: [PML4 | PDP | PD | PT | Offset]
              ↓     ↓    ↓    ↓
           4 级查表
```

未使用的虚拟地址区域不需要分配页表页。**代价**：每次翻译需要 4 次内存访问。

**TLB** 是页表的缓存——存最近用的虚拟→物理映射。TLB 命中时翻译接近零延迟。

### 缺页异常：按需分页

进程虚拟地址空间很大（2^64 字节），物理内存有限。OS 不一开始就分配全部。

第一次访问某虚拟页时：
1. MMU 查页表发现页不在内存
2. 触发 page fault 异常
3. OS 处理：找空闲页帧、从磁盘读、更新页表
5. 重试指令

**这件事是虚拟内存的核心**——OS 只在需要时分配物理内存，"按需分页"。

## C 的设计选择：malloc/free 是手动内存管理

### 内存安全噩梦

```c
void leaky() {
    char *p = malloc(1024);
    if (error_condition) return;  // 忘了 free
    free(p);
}
```

- **内存泄漏**：忘了 free
- **Use-After-Free**：`free(p); p[0] = 'x';`——指针值没变，块已没清
- **Double Free**：`free(p); free(p);`——同一块释放两次
- **野指针**：`free(p);` 忘了 `p = NULL`

**这些 bug 的根源**：C 的指针 free 后还是"有效"的。类型系统不跟踪生命周期。

### mmap 暴露

```c
void *data = mmap(NULL, size, PROT_READ, MAP_PRIVATE, fd, 0);
```

`mmap` 把文件映射到虚拟地址空间——读这块内存就是读文件。**这件事 OS 提供，C 包装**——文件
I/O 和内存 I/O 同一个抽象。

## 现代视角：defer、Allocator、生命周期

### defer 让资源清理是作用域的

```rust
fn read_file() -> Result<(), io::Error> {
    let file = std::fs::File::open("data.txt")?;
    let data = std::fs::read_to_string(&file)?;  // 离开作用域自动释放
    Ok(())
}
```

`Vec` 实现 `Drop`，离开作用域自动释放。**C 的"每个 return 前清理"是原理缺陷**——defer
把清理绑到作用域，不需要人记。

### Allocator 注入，让分配策略可换

```rust
fn process(allocator: &mut dyn Allocator) { ... }  // 调用者选分配器

// 选 jemalloc
let mut allocator = Jemalloc::new();
process(&mut allocator);

// 选 arena
let mut arena = Arena::new();
process(&mut arena);
```

C 的 `malloc()` 是全局函数。Rust/Zig 的分配器是**参数**——"用什么分配器"是个决策，不是个
全局状态。

### 生命周期让指针安全是编译期可见

```rust
fn get_str<'a>(s: &'a str) -> &'a str { s }
```

Rust 的生命周期 `'a` 是**类型的一部分**。`&str` 知道它指向的数据活多久。**这件事 C 没有**——
C 的 `char*` 不知道指向的数据什么时候失效。

`valgrind`/`ASan`/`miri` 是**运行时检查**——Rust 编译器静态证明没有 use-after-free。
**AI 帮你跑 valgrind**，但静态证明是语言层的事。

## AI 时代怎么验证

**1. 让 AI 分析 page fault trace**

```bash
$ perf stat -e page-faults ./your_program
$ strace -e trace=mmap,mprotect,munmap ./your_program
```

让 LLM 解释：哪些操作触发了 page fault？是 mmap 还是 stack growth？mprotect 在保护什么？

**3. 让 AI 跑 ASan/valgrind 找 use-after-free**

```bash
$ clang -fsanitize=address t.c
$ valgrind ./your_program
```

让 LLM 解释：哪些地址被 use-after-free？分配器和释放器的调用栈是什么？

**4. 让 AI 解释 mmap 区域**

```bash
$ cat /proc/self/maps
```

让 LLM 解释：哪些区域是代码？哪些是 stack？哪些是 mmap？为啥 heap 在这？

**5. 让 AI 跑 miri 验证 unsafe Rust**

```bash
$ cargo install miri
$ cargo miri run
```

让 LLM 解释：哪些 unsafe 块有内存安全问题？miri 怎么静态模拟 UB？

## 本讲要点

1. 虚拟内存是硬件 + OS 协同设计：MMU 翻译、页表规则、按需分页
2. 多级页表节省空间，TLB 加速翻译
3. C 的 malloc/free 是手动内存管理——容易泄漏、UAF、double free
4. defer 让资源清理是作用域的，Allocator 注入让分配策略可换
5. Rust 的生命周期是类型的一部分——UAF 是编译期错误

## 下一讲

程序需要和外部世界交互——读写文件、收发网络数据。下一讲看系统级 I/O 和网络编程的入门——
以及 C 的隐藏缓冲区为什么是设计得很糟糕的。