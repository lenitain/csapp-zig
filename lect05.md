# CSAPP 第五讲：优化程序性能

## 物理原理：性能瓶颈不在算法

```c
// 版本 A
for (int i = 0; i < n; i++)
    a[i] = b[i] + c[i];

// 版本 B（两次遍历）
for (int i = 0; i < n; i++)
    a[i] = b[i];
for (int i = 0; i < n; i++)
    a[i] += c[i];
```

直觉：版本 B 多遍历一次，应该慢。

现实：不一定。如果数组很大，版本 B 对**缓存**更友好（每次工作集小），可能更快。

**程序性能不只取决于算法复杂度**——还取决于它和硬件（缓存、分支预测、流水线）的匹配。
O(n) 算法可能比 O(log n) 算法慢 100 倍——如果前者 cache miss 多。

## C 的设计选择：UB 是合法的优化依据

C 抽象机假设"程序顺序执行、不溢出、指针有效"。这个假设是**优化依据**：

### 案例 1：有符号溢出是 UB

```c
int arr[4] = {0};
for (int i = 0; i <= 4; i++) {  // i++ 不会溢出（i <= 4）
    arr[i] = 0;                   // i == 4 越界——但越界是 UB
}
```

`-O3` 可能把循环优化成 `while(true)`——因为 C 标准假设 `i++` 永远不会溢出（UB 不发生），
所以 `i <= 4` 永远真，循环无限。这是**优化依据**，不是编译器 bug。

### 案例 2：指针别名假设

```c
void foo(int *p, int *q) {
    *p = 1;
    *q = 2;
    if (*p == 1) { /* ... */ }    // 编译器可能删除这个分支
}
```

`p` 和 `q` 可能指向同一地址（别名）。`*q = 2` 会让 `*p = 2`，所以 `*p == 1` 不成立。

但如果 `p` 是 `restrict int*`（无别名），编译器假设 `*q = 2` 不影响 `*p`，删除 `if`。这是
**编译器信任程序员的代价**——违反 `restrict` 是 UB。

### 案例 3：死代码消除

```c
int foo(int x) {
    int y = x * 2;
    return x + 1;  // y 从未使用，整条赋值被删
}
```

`y` 没用，编译器删除。**优化**——但你要知道它删了什么，否则调试器会让你以为堆栈存在。

### 案例 5：函数内联

```c
inline int square(int x) { return x * x; }
int y = square(5);  // 变成 y = 25
```

消除调用开销。**代价**：可执行文件变大。**收益**：调用开销消除。编译器根据函数大小和调用
频率决定，不是你想内联就内联。

## 现代视角：确定性 ops，AI 帮你做优化

### 没有 UB，行为确定

```rust
let arr = [0i32; 4];
let mut i = 0;
while i <= 4 {
    arr[i] = 0;        // 运行时检查：i == 4 panic
    i += 1;
}
```

Rust 数组访问默认 panic。`-O3` **不能**假设 `i <= 4` 永远成立——因为 i == 4 会 panic，
不是 UB。**编译器不能优化成无限循环**。

你付出一点点性能（不能做激进 UB-aware 优化），换来**确定性**——Debug 和 Release 表现一致。

### Comptime：编译时计算是语言的一部分

```zig
fn fibonacci(comptime n: u32) u32 {
    if (n < 2) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}
const x = fibonacci(10);  // 编译时算出 55
```

这不是宏，不是模板元编程，是普通函数在编译时执行。编译器把能算的全算了，运行时零开销。

C++ 的 `constexpr`、Rust 的 `const fn` 都做这件事——但**带类型检查**。C 的 `#define FIB(n)`
是文本替换，编译器不会检查。

### 自动向量化是编译器的事

```c
// 你写的
for (int i = 0; i < n; i++) a[i] = a[i] * 2;

// LLVM 可能生成的（SIMD）
for (int i = 0; i < n; i += 4) {
    // 一条指令处理 4 个 float
}
```

LLVM 的 -O3 自动向量化。你不需要手写 SIMD intrinsics——除非极端场景。AI 可以帮你写 SIMD，
但默认不需要。

## AI 时代怎么验证

**1. 让 AI 对比 `-O0` 和 `-O3` 汇编**

```bash
$ clang -O0 -S t.c > O0.s
$ clang -O3 -S t.c > O3.s
$ diff O0.s O3.s    # 看差异
```

让 LLM 解释每处差异。重点看：哪些函数被内联？哪些分支被消除？循环被向量化了吗？

**2. 让 AI 跑 perf 找热点**

```bash
$ perf record ./your_program
$ perf report --sort=-generate
$ perf annotate
```

让 LLM 解释：哪些函数占 CPU 最多？是计算密集还是内存密集？branch-miss 率高吗？
**这不是看源代码能做的——必须看运行时数据。**

**3. 让 AI 解释 PGO (Profile-Guided Optimization)**

```bash
$ clang -fprofile-generate t.c -o program
$ ./program             # 跑一遍生成 profile
$ clang -fprofile-use t.c -o program   # 用 profile 重新编译
```

让 LLM 解释：PGO 为什么比 -O3 快？哪些优化依赖 profile？

**4. 让 AI 找 UB**

```bash
$ clang -fsanitize=undefined t.c    # UBSan 跑一遍
$ clang -fsanitize=address t.c      # ASan 跑一遍
```

让 LLM 跑 sanitizer 报告。重点看：哪些是"你以为不会发生"的 UB？每条 UBSan 警告都对应一个
**编译器利用 UB 优化的位置**。

## 本讲要点

1. 性能不只取决于算法复杂度，还取决于和硬件的匹配（cache、branch、pipeline）
2. C 的 UB 是合法的优化依据——编译器信任你遵守规则，违反触发 UB 优化
3. 现代语言的确定性 ops 让 Debug/Release 表现一致，代价是少一点优化空间
4. 自动向量化、自动内联、死代码消除是 LLVM 的事——你不需要手写
5. AI 时代 perf 工具比源代码更重要——性能是运行时属性

## 下一讲

编译器和处理器都在努力让程序更快，但它们都受制于一个物理限制：**快的存储贵，便宜的存储慢。**
下一讲看内存层级——以及为什么"缓存友好"是现代程序性能的关键。