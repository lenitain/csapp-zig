# Cache Lab

**对应章节：** 第六讲（内存层级）

**一句话点题：** 缓存的行为是可预测的——理解它，你就能写出更快的代码。

## 为什么要做这个实验

CSAPP 的 Cache Lab 有两部分：
1. 写一个缓存模拟器，根据内存访问 trace 判断 hit/miss
2. 优化矩阵转置函数，减少缓存 miss

在 AI 时代，手写缓存模拟器或手写缓存优化代码的价值趋近于零。但实验传达的**核心 insight** 仍然重要：

**缓存的行为是确定性的。** 给定缓存大小、行大小、关联度，你可以精确预测每一次内存访问是 hit 还是 miss。理解这个，你就能理解为什么某些代码比其他代码快。

## Part 1：缓存模拟器

### 输入

Valgrind 的内存访问 trace：
```
I 0400d7d4,8      # 指令读取
 L 04f6b868,8      # 数据读取
 S 7ff0005c8,8     # 数据写入
 M 04f6b860,8      # 数据修改（先读后写）
```

### 核心逻辑

```python
def cache_access(addr, size, cache):
    set_index = (addr >> b) & (2**s - 1)  # 组索引
    tag = addr >> (s + b)                   # 标记

    # 在对应的组中查找
    for line in cache[set_index]:
        if line.valid and line.tag == tag:
            return HIT  # 命中

    # 未命中，需要加载
    evict_line = find_lru(cache[set_index])
    evict_line.valid = True
    evict_line.tag = tag
    return MISS
```

### 思路

1. 从地址中提取组索引和标记
2. 在对应的组中查找匹配的缓存行
3. 找到就是 hit，没找到就是 miss
4. miss 时需要驱逐一个旧行（LRU 策略）

就这么简单。**缓存就是一个哈希表，key 是地址，value 是数据。**

## Part 2：矩阵转置优化

### 问题

```c
void transpose(int N, int M, int A[N][M], int B[M][N]) {
    for (int i = 0; i < N; i++)
        for (int j = 0; j < M; j++)
            B[j][i] = A[i][j];
}
```

这个写法对缓存不友好：A 按行访问（好），B 按列访问（坏）。

### 优化：分块 (Blocking)

```c
void transpose_blocked(int N, int M, int A[N][M], int B[M][N]) {
    int block = 8;  // 缓存行大小 / sizeof(int)
    for (int i = 0; i < N; i += block)
        for (int j = 0; j < M; j += block)
            for (int ii = i; ii < min(i + block, N); ii++)
                for (int jj = j; jj < min(j + block, M); jj++)
                    B[jj][ii] = A[ii][jj];
}
```

### 思路

把大矩阵分成小块，每个小块能放进缓存。在小块内部，A 和 B 都是按行访问的，缓存命中率高。

**核心 insight：分块让工作集变小，对缓存友好。** 这个技巧在矩阵运算、图像处理、数据库查询中都有应用。

## 本实验的 takeaway

1. 缓存的行为是确定性的——你可以预测每一次访问是 hit 还是 miss
2. 缓存的本质是哈希表：地址 → 组索引 → 标记匹配
3. 空间局部性（访问相邻数据）比时间局部性（重复访问同一数据）更容易利用
4. 分块 (blocking) 是通用的缓存优化技巧

**核心 insight：性能优化不是"猜测"，是"计算"。你理解缓存的结构，就能预测程序的性能。**
