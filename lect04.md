# CSAPP 第四讲：处理器体系结构

## 物理原理：CPU 不是你想象的那样

**你以为：** CPU 是一条条执行指令的——取一条，执行，取下一条。

**现实：** 现代 CPU 是**超标量** (superscalar) + **乱序执行** (out-of-order) + **分支预测**
(branch prediction)。它同时维护几十条指令的"飞行窗口"：

- 两条没数据依赖的指令可以真正并行
- 一条慢指令（cache miss）不会阻塞后面的无关指令
- 指令的"顺序"在微观上可能和你写的完全不同

这是**物理实现**：单个 ALU 太慢，现代 CPU 有 4-8 个执行单元（int ALU、float ALU、load、
store、branch），必须并行才能喂饱。

### 流水线是工厂

把执行分成取指、译码、执行、访存、写回——5 个阶段用独立硬件：

```
时钟周期:  1   2   3   4   5   6   7   8
指令1:    IF  ID  EX  MEM WB
指令2:        IF  ID  EX  MEM WB
指令3:            IF  ID  EX  MEM WB
```

理想吞吐：每周期一条指令。延迟 5 周期，吞吐 1 IPC。

### 三种流水线冒险

**数据冒险**：后一条指令要用前一条的结果。

```
add %rax, %rbx    ; 写 rbx
sub %rbx, %rcx    ; 读 rbx——依赖
```

解决：**转发** (forwarding/bypassing)——第一条指令执行完，结果直接传给第二条的执行阶段，
不等写回。

**结构冒险**：两条指令同时要用同一个硬件单元。比如两个 ALU 操作同时想用 ALU。**这是真硬件
限制**，只能 stall 一个。

**控制冒险**：分支指令不知道下一条是什么。CPU 得等分支结果。

### 分支预测：CPU 赌装

解决控制冒险：**猜**分支方向，提前执行猜测路径。

```
if (cond) {
    // 路径 A
} else {
    // 路径 B
}
```

CPU 猜 `cond == true`，开始执行路径 A。猜对：0 周期代价。猜错：丢弃路径 A 工作，**惩罚
15-20 周期**。

现代 CPU 预测准确率 > 95%——它记录分支历史模式（循环分支总是 99 次进 1 次出）。但 5%
预测错时，代价巨大。

**无分支编程的真正原因**：`cmov` (conditional move) 不需要分支预测。编译器自动把简单
`if-else` 转成 `cmov`。这件事 AI 也能帮你做——但你需要知道为什么 cmov 快。

### 投机执行与 Spectre

2018 年 Spectre 揭示：**CPU 会投机执行不该执行的代码。**

```
; 推测执行这段（虽然 condition 是 false）
if (secret_array[i] < bound) {
    cache_probe[secret_array[i]];  // 把数据放到 cache
}
// 即使 condition false，cache 状态留下了痕迹
// 通过测量 cache 访问时间能推断出 secret_array[i]
```

这是**真实漏洞**。修复需要：硬件 mitigations（retpoline、microcode updates）+ 软件 patches。
AI 可以帮你复现 Spectre 解释原理，但你需要知道它是真的必须靠硬件+软件协同解决的事。

## C 的设计选择：volatile 和 restrict 是用来跟 CPU 玩游戏的

C 抽象机假设"程序是单线程顺序执行的"。**这跟硬件完全不符**。所以 C 给你两个关键字来"修正"：

### `volatile`：告诉编译器"别优化这个读"

```c
volatile uint32_t *sensor = (volatile uint32_t *)0x1000;
while (*sensor == 0) { /* spin */ }
```

`volatile` 阻止编译器把 `*sensor` 缓存到寄存器。每次访问都重新读内存。**这件事是 CPU 缓存惹的**
——硬件缓存让编译器看不到变化，编译器把读优化掉，程序死循环。

`volatile` 不提供原子性。它只阻止编译器优化，**不阻止 CPU 重排**。要原子性要 `atomic` 或锁。

### `restrict`：告诉编译器"指针不别名"

```c
void add(int * restrict a, int * restrict b, int * restrict c, int n) {
    for (int i = 0; i < n; i++) a[i] = b[i] + c[i];
}
```

`restrict` 是程序员的断言：以后加法，**任何写入 a 都不会影响 b/c 的值**。这样编译器可以做激进
优化（比如把 `b[i]` 和 `c[i]` 预取到寄存器，知道它们不会变）。

**C 标准的限制**：C 不能静态证明不别名（两个指针可能指向同一块内存）。`restrict` 是程序员
**担保**，违反是 UB。

### C 抽象机 vs 现实物理：

```
j + 1 < 0;  // 抽象机：永远 false 因为 int 在 C 里不能溢出
            // -O3：编译器可以删除这个分支
```

这是上一讲说的 UB-as-feature。**编译器和硬件都赌你遵守 C 抽象机的规则**，赌赢了激进优化，
赌输了 undefined behavior。

## 现代视角：让硬件抽象不再是程序员的责任

### C++ memory order 和 Rust acquire/release

Rust 用 `Acquire`/`Release` 语义表达同步原语，**编译器负责不优化重排**。你不需要写
`asm volatile` 或 volatile 指针。

```rust
use std::sync::atomic::{AtomicBool, Ordering};
static READY: AtomicBool = AtomicBool::new(false);

// producer
let data = compute_heavy_stuff();
READY.store(true, Ordering::Release);

// consumer
while !READY.load(Ordering::Acquire) { /* spin */ }
use(data);
```

`Ordering::Release` 告诉编译器：**这一行之前的所有内存写入不能越过它**。这件事在 C 里要
volatile + 屏障指令 + 原子操作组合，Rust 是类型系统层的语义。

### spectre 的理论证明

Spectre 的存在让"硬件应该安全执行任意代码"这件事**不成立**。这迫使操作系统和编译器
加入 retpoline、`lfence` 屏障、`STIBVEC` 等抑制措施。

AI 可以帮你写 retpoline 模板，但**你需要知道为什么有这个东西**——硬件层默认不安全。

### 无分支是编译器的事

`if (x > 0) x else 0` 编译成 `cmov` 是 LLVM 自动做的。**你不需要关心**。但你需要知道这件事——
AI 优化代码时它会写 `cmov` 而不是分支，你需要问它"为什么用 cmov"。

## AI 时代怎么验证

**1. 让 AI 跑 perf 看分支预测 miss 率**

```bash
$ perf stat -e branch-misses ./your_program
$ perf annotate your_program
```

让 LLM 解释：哪些函数 branch-miss 率高？是循环边界还是 if 条件？为什么？是不是数据依赖？

**2. 让 AI 解释 μops**

```bash
$ llvm-mca --mcpu=x86-64 code.s    # 模拟 CPU 调度
```

让 LLM 跑 `phi -M` 看你的汇编有多少 μops、吞吐怎么样。**这件事是看 CPU 调度能力的，不是看
源代码的。**

**3. 让 AI 解释 spectre PoC**

让 LLM 给一段 spectre 的 PoC 代码（读 `secret_array`）。重点看：
- 训练数组 `cache_probe`
- 投机执行条件的边界检查
- cache 探针如何泄露数据

**4. 让 AI 解释 retpoline**

问：retpoline 是什么？为啥需要？哪些 CPU 受影响？软件怎么绕过？

## 本讲要点

1. 现代 CPU 是超标量 + 乱序 + 分支预测，单条指令不是顺序执行的
2. 流水线是 5 阶段工厂，吞吐 1 IPC，但分支预测失败 15-20 周期
3. `volatile` 和 `restrict` 是 C 抽象机跟硬件现实的"补丁"
4. Spectre 是硬件漏洞，让"硬件应该安全执行任意代码"这件事不成立
5. 无分支编程 (`cmov`) 是编译器自动做的——但你要知道为什么

## 下一讲

处理器再快，程序也可能很慢。下一讲看编译器优化——以及 C 的 undefined behavior 给了编译器
"合法伤害"你的权力。