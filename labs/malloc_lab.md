# Malloc Lab

**对应章节：** 第九讲（虚拟内存）

**一句话点题：** 内存分配器就是堆上的数据结构问题——怎么组织空闲块，怎么快速找到合适大小的
块，怎么合并相邻空闲块。

## 为什么不手写 malloc

原 CSAPP Malloc Lab 让你实现 `malloc`、`free`、`realloc`——用户空间内存分配器。

**AI 时代这件事的核心价值是"理解"而不是"动手"**——手写 malloc 的工程价值还没体现到，glibc 的
ptmalloc 已经优化了几十年。**你的工作是理解数据结构选择对性能的影响、为什么 jemalloc 比 glibc
malloc 快、arena 分配器为什么在某些场景下比通用分配器快。**

## 核心 insight

**分配器是堆上的数据结构问题。**

- 怎么组织空闲块？（隐式链表、显式链表、分离空闲链表、红黑树）
- 怎么快速找合适大小的块？（首次适配、最佳适配、二分查找）
- 怎么合并相邻空闲块？（立即合并、延迟合并）
- 怎么处理碎片？（内部碎片、外部碎片）

这些问题和链表、哈希表、树是同一类问题——**算法和数据结构选择对性能有决定性影响**。

## AI 帮你做什么

**1. 让 AI 解释 glibc malloc 源码**

```bash
$ apt source glibc  # 或从 gitlab 拉 glibc 源码
$ cd glibc/malloc
$ wc -l malloc.c    # 看代码量
```

让 LLM 解释：

- `arena` 是什么？为什么有多个 arena？
- `bin` 是什么？怎么分组小块？
- `top chunk` 是什么？高地址空闲块
- `chunk` 结构：prev_size + size + data + ...

**2. 让 AI 跑 trace 看 malloc/free 模式**

```bash
$ ltrace -e malloc+free ./your_program
```

让 LLM 解释：

- 每次 malloc 请求多大？
- 分配器选哪个 bin？命中率多少？
- 分配耗时分布——哪些操作是热点？

**3. 让 AI 对比 jemalloc vs glibc malloc**

```bash
# 让 AI 写一个程序：
# - 大量并发线程各自 malloc/free
# - 大量小对象分配
# - 大量大对象分配

$ LD_PRELOAD=/usr/lib/libjemalloc.so ./your_program
$ LD_PRELOAD=/usr/lib/libstdc++.so ./your_program  # 默认 glibc malloc
```

让 LLM 解释：

- jemalloc 为什么多线程更快？per-thread arena
- jemalloc 为什么小对象分配更快？size-class 分组

**4. 让 AI 解释 arena allocator**

```rust
let mut arena = Arena::new();
{
    let x = arena.alloc(5);  // 不调 free
    let y = arena.alloc(10);
}  // 离开作用域 arena.drop() 一次性释放所有
```

让 LLM 解释：

- arena 适合什么场景？分配很多次、用完一起丢
- 性能为什么好？没有 free 调用
- 代价？必须一起释放

## 你应该能回答的判断题

1. "分配器的核心问题是什么？" 答：怎么组织空闲块 + 怎么找 + 如何处理碎片。
3. "显式 vs 隐式空闲链表的区别？" 答：显式只遍历空闲，隐式遍历所有。
4. "分离空闲链表怎么做？" 答：按大小分组，每组一个链表。glibc malloc 用这个。
5. "arena 分配器适合什么场景？" 答：分配很多次、用完一起丢。
6. "jemalloc 比 glibc malloc 快在哪？" 答：per-thread arena + size-class 分组。
7. "分配器为什么是数据结构问题？" 答：堆就是一块连续内存，分配是连续的，每块有元数据，组织方式影响性能。

## 三个策略对比

### 隐式空闲链表

```text
+--------+--------+--------+--------+
| 头部   | 负载   | ...    | 填充   |
+--------+--------+--------+--------+
```

**核心**：块在内存连续，通过块头部大小算下一个块位置。

**缺点**：分配要扫整块找大块。O(n) 分块。

### 显式空闲链表

```text
+--------+--------+--------+--------+--------+
| 头部   | 前驱   | 后继   | 负载   | ...    |
+--------+--------+--------+--------+--------+
```

**核心**：空闲块多 prev/next 指针。分配只遍历空闲块。

**性能**：只遍历空闲，O(空闲块数)。

### 分离空闲链表

```text
size_class 1 (8-16 字节)  → 链表 A
size_class 2 (17-32 字节)  → 链表 B
...
size_class N              → 链表 N
```

**核心**：按大小分组，每组一个链表。分配直接去对应链表找。

**性能**：O(1) 分配，O(1) 释放。jemalloc 用这个。

## AI 时代：选分配器

**分配器选择不是语言层的事**——它是**架构决策**。

```rust
// Rust 动——如果是 perf-critical
let mut arena = Arena::new();
process(&mut arena);

// 普通场景
let mut gpa = GlobalAlloc::new();
process(&mut gpa);
```

你**能选分配器**这件事是设计选择。C 的 malloc 全局没法换——程序同一时间只能用同一个分配器。

## 本实验 takeaway

1. 分配器是堆上的数据结构问题——链表/树/分组都影响性能
2. 隐式 vs 显式 vs 分离——三种策略，三种性能
3. jemalloc 比 glibc malloc 快——per-thread arena + size-class 分组
4. AI 帮你看分配器源码，跑 trace，但你要懂数据结构选择
5. 现代语言把"分配策略"作为架构决策——Allocator 注入

**核心 insight：malloc 不是魔法。它是个工程问题——在分配速度、释放速度、碎片率之间找平衡。
理解这个，可以像一块代码的"频繁 malloc/free"实际是分配器负载过重。**