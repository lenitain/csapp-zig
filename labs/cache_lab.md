# Cache Lab

**对应章节：** 第六讲（内存层级）

**一句话点题：** 缓存行为是确定性的。给定 cache 参数，你可以精确预测每一次内存访问是 hit
还是 miss。

## 为什么不手写 cache 模拟器

原 CSAPP Cache Lab 让你写一个 cache 模拟器（Part 1），优化矩阵转置（Part 2）。

**AI 时代这件事的价值是"理解"而不是"动手"**——让 LLM 写模拟器是几分钟的事。让 LLM 跑
csim 给你 hit/miss 数据也是几秒钟的事。**你的工作是解释每个 miss 的原因、为什么 blocking 不
能加速、为什么按行遍历比按列快。**

## 核心 insight

**缓存的本质是哈希表：地址 → cache set → tag 匹配。**

给定：
- 地址 X
- cache 大小 C
- 块大小 B
- 关联度 E

你可以精确计算：
```
set_index = (X / B) % (C / (B * E))   # 这是组号
tag = X / (B * (C / (B * E)))         # 这是标记
```

然后在 `cache[set_index]` 里找 `tag`——找到就是 hit，没找到就是 miss。

## AI 帮你做什么

**1. 让 AI 解释 valgrind cachegrind 输出**

```bash
$ valgrind --tool=cachegrind ./your_program
$ cg_annotate cachegrind.out.<pid>
```

让 LLM 解释：

- 总访问数多少？D1 命中率？LL 命中率？
- 哪些函数 miss 率高？是 instruction 还是 data？
- L1 miss 但 L2 hit 占多少？main memory 占多少？

**2. 让 AI 对比按行 vs 按列 vs blocking**

```bash
$ perf stat ./matmul_row_major
$ perf stat ./matmul_col_major
$ perf stat ./matmul_blocked
```

让 LLM 解释：

- 为什么按行比按列快？（空间局部性：相邻元素在同一个 cache line）
- 为什么 blocking 快？（工作集变小，全部塞进 cache）
- block size 怎么选？（= cache size / sizeof(int) / N - 经验值）

**3. 让 AI 跑 csim 模拟器**

```bash
$ ./csim -s 4 -E 1 -b 4 -t trace.txt
```

让 LLM 解释：

- L1 大小 16 字节，块 4 字节，1 路组相联
- `set_index = addr / 4 % 4` 怎么算
- `tag = addr / 16` 怎么算

**4. 让 AI 解释 false sharing**

```bash
$ perf c2c record ./your_program
$ perf c2c report
```

让 LLM 解释：

- 哪些地址在多核之间互相 ping-pong？
- 是因为数据布局还是编译器没对齐？
- 怎么改 struct 让每个字段独占 cache line？

## 你应该能回答的判断题

1. "给定 cache 参数，怎么算 hit/miss？" 答：set_index = (addr / block_size) % num_sets；tag = addr / (block_size * num_sets)。
2. "为什么按行比按列快？" 答：cache line 包含相邻元素，按行访问一个 cache line 全用上了，按列每次都重新加载。
3. "什么是 false sharing？" 答：两个不相关变量在同一个 cache line，多核互相 invalid 对方的 cache。
4. "blocking 怎么选 size？" 答：cache size / sizeof(element) — 让工作集塞进 cache。
5. "为什么 prefetcher 对顺序访问有效？" 答：检测到 stride 模式，提前把后面的数据加载。

## 三个经典模式

### 模式一：按行 vs 按列

```c
// 慢：按列遍历（cache miss 接近 100%）
for (int j = 0; j < N; j++)
    for (int i = 0; i < N; i++)
        sum += a[i][j];

// 快：按行遍历（cache miss 接近 0%）
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        sum += a[i][j];
```

C 数组按行存储。按行访问：相邻元素在同一个 cache line，命中率接近 100%。
按列访问：每次跨行，重新加载。

**核心 insight**：数据和访问模式匹配决定性能。

### 模式二：blocking

```c
// 不 blocking：每次都要扫大块内存
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        for (int k = 0; k < N; k++)
            c[i][j] += a[i][k] * b[k][j];

// blocking：分块，每次只处理小块
int B = 8;
for (int ii = 0; ii < N; ii += B)
    for (int jj = 0; jj < N; jj += B)
        for (int kk = 0; kk < N; kk += B)
            for (int i = ii; i < ii + B; i++)
                for (int j = jj; j < jj + B; j++)
                    for (int k = kk; k < kk + B; k++)
                        c[i][j] += a[i][k] * b[k][j];
```

**核心 insight**：分块让工作集变小，全部塞进 cache 后命中率上去。

### 模式三：SoA vs AoS

```c
// AoS (Array of Structs) - 不友好
struct Particle { float x, y, z; int id; };
Particle particles[N];

// SoA (Struct of Arrays) - 友好
struct Particles {
    float x[N], y[N], z[N];
    int id[N];
};
```

如果你只处理 x/y/z，SoA 让它们各自连续，cache 利用率高。

## 本实验 takeaway

1. cache 是哈希表——地址 → set → tag 匹配
2. 空间局部性（相邻元素）比时间局部性（重复访问）更容易利用
3. blocking 是通用的 cache 优化技巧
4. AI 帮你跑 cachegrind/perf c2c，但你需要懂数据布局
5. cache 行为是确定性的——可以预测，不能猜

**核心 insight：性能优化不是"猜测"，是"计算"。你理解 cache 的结构，就能预测程序的性能。
AI 帮你跑实验，但你要做判断。**