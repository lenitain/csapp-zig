# CSAPP 第六讲：内存层级——速度的代价

## 物理原理：快的贵，便宜卖的贵

CPU 每秒几十亿条指令。主存访问 100+ 周期。如果每条指令都等内存，CPU 99% 闲着。

**造不出又快又便宜的内存**——这是物理限制。SRAM 一个比特 6 晶体管，DRAM 一个比特 1 晶体管
+ 一个电容（要刷新）。SRAM 快但贵，DRAM 慢但便宜。

**解决方案不是"造更快内存"，是分层**：

```
寄存器    1 ns      < 1 KB
L1 cache   ~ 1 ns    32-64 KB
L2 cache   ~ 3 ns    256 KB - 1 MB
L3 cache   ~ 10 ns   几 MB - 几十 MB
主存 (DRAM) ~ 100 ns 几 GB
磁盘 (SSD)   ~ 100 μs 几 TB
磁盘 (HDD)   ~ 10 ms 几 TB
```

每一层比下一层**快 5-10 倍**，但**容量小 10-100 倍**。整个内存层级是物理约束的具象化。

## C 的设计选择：编译器不知道你的访问模式

### 缓存是怎么工作的

CPU 维护一组缓存行（典型 64 字节）。当你读一个字节，整个 64 字节加载到 L1。

**为什么是 64 字节？** SRAM 不是堆出来的。是历史标准 + 工业折衷。AMD/Intel/ARM 现在还是 64 字节
（Apple M4 是 128 字节）。

**空间局部性**：你访问 `a[i][j]`，相邻 `a[i][j+1]` 很快也会被访问——一行一加载就用上了。
**时间局部性**：你访问 `a[i][j]`，紧接着又访问 `a[i][j]`——已经在 cache 里了。

### 三种 cache miss

1. **冷不命中** (cold miss)：第一次访问。不可避免。
2. **容量不命中** (capacity miss)：工作集超过 cache 容量。换 cache 或改算法。
3. **冲突不命中** (conflict miss)：多个地址映射到同一 cache set，互相驱逐。最阴险。

### 按行遍历 vs 按列遍历

```c
// 二维数组 int[N][N]

// 快（按行）
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        sum += a[i][j];

// 慢（按列）
for (int j = 0; j < N; j++)
    for (int i = 0; i < N; i++)
        sum += a[i][j];
```

两个循环访问同样的数据、做同样的计算，但可能差 10 倍。

为啥？C 数组按行存储（row-major）。按行遍历，每次访问 `a[i][j]`、`a[i][j+1]`，是同一行的相邻
元素——都在同一个 cache line，命中率接近 100%。

按列遍历，每次访问 `a[i][j]`、`a[i+1][j]`，跨行——每次都是新 cache line。命中率接近 0%。

**这不是算法问题**，是数据和硬件访问模式的匹配。

### C 隐藏的内存分配

```c
printf("Hello, World!\n");
```

`printf` 内部有缓冲区。**你调用 printf 触发了一次隐藏的 malloc**——stdio 第一次会分配 4-8KB
的 FILE 结构。你看不到。

更糟：

```c
size_t len = strlen(s);
```

`strlen` 遍历整个字符串直到 `\0`。**没有长度信息就得扫一遍**。这件事编译器没法优化（除非能
证明字符串不变）。

## 现代视角：让"长度"和"缓冲"成为类型

### 长度是 string 的一部分

```rust
let s: &str = "hello";
let len = s.len();  // O(1)，不需要遍历
```

Rust 的 `&str` 是 `(ptr, len)`，长度是常量。C 的 `strlen(s)` 必须扫。**这件事不是优化问题，
是设计选择**——C 字符串没有长度信息。

### Slice 让你选缓冲策略

```rust
let file = std::fs::File::open("data.txt")?;
let mut reader = std::io::BufReader::new(file);  // 显式缓冲
// vs
let mut reader = std::fs::File::open("data.txt")?;  // 无缓冲
```

Rust/Zig 的 I/O 默认无缓冲。要缓冲要显式 wrap。**缓冲是你选择的，不是 stdio 隐藏的。**

这件事**AI 帮你调优**——你可以让 LLM 解释 `BufReader` 内部做了什么、为什么默认 8KB。

### Allocator 注入

```rust
fn process(allocator: &mut dyn Allocator) { ... }    // 调用者选分配器
```

C 的 `malloc` 是全局函数，整个程序共用一个分配器。Rust/Zig 的分配器是**参数**——你决定
用什么分配器（GPA、Arena、mimalloc、jemalloc）。

这件事**不是性能优化，是设计**：让"内存分配策略"成为可测试、可替换的变量，不是全局状态。

## AI 时代怎么验证

**1. 让 AI 跑 cachegrind 看 hit rate**

```bash
$ valgrind --tool=cachegrind ./your_program
$ cg_annotate cachegrind.out.<pid>
```

让 LLM 解释：哪些函数 miss rate 高？是 instruction 还是 data miss？大小端，cache line 怎么划的？
**这件事，源代码看不出来。**

**2. 让 AI 对比按行 vs 按列遍历的 perf**

```bash
$ perf stat ./row_major
$ perf stat ./col_major
$ perf stat ./blocked   # 分块版
```

让 LLM 解释：blocked 版为什么快？block size 怎么选？

**3. 让 AI 跑 perf c2c 看 false sharing**

```bash
$ perf c2c record ./your_program
$ perf c2c report
```

让 LLM 解释：哪些地址在多核之间互相 ping-pong？是数据布局，还是编译器没对齐？

**4. 让 AI 解释 SoA vs AoS**

```rust
// AoS (Array of Structs) - 不太友好
struct Particle { x: f32, y: f32, z: f32, id: u32 }
let particles: Vec<Particle> = ...;

// SoA (Struct of Arrays) - 更友好
struct Particles { x: Vec<f32>, y: Vec<f32>, z: Vec<f32>, id: Vec<u32> }
```

让 LLM 跑一遍两个版本的 SIMD 访问。`x`/`y`/`z` 连续排列，对 SIMD 更友好。

## 本讲要点

1. 内存层级是物理约束：SRAM 快但贵，DRAM 慢但便宜，所以分层
2. 缓存 line 是硬件读写的，不是写出来的——64 字节是工业折衷
3. 访问模式决定性能：按行 vs 按列遍历可能差 10 倍
4. C 隐藏了 stdio 缓冲、字符串长度，这些是 stdio 库的"决策"
5. 现代语言让长度、缓冲、Allocator 成为类型层可见——不是你看不到的隐藏状态
6. 性能是运行时属性——必须用 perf/cachegrind 看，不是看源代码

## 下一讲

程序由多个文件编译而来，它们怎么拼装成一个可执行文件？下一讲看链接——以及为什么 C
的头文件是糟糕的设计（但我们都得读它），而 Zig/Rust 模块系统怎么修正它。