# Bomb Lab

**对应章节：** 第三讲（汇编）

**一句话点题：** 汇编有规律，规律可推。读懂汇编就是读懂 CPU 在想什么。

## 为什么不手写反汇编

原 CSAPP Bomb Lab 给你一个 binary，需要通过反汇编找到正确的输入序列来"拆除"它。

**AI 时代这件事的核心价值是"判断"而不是"动手"**——把 binary 丢给 LLM，它能反汇编 + 解释
每条指令。**你的工作是验证它说的对不对、看它漏了什么 case、看它解释能不能让你理解整体逻辑。**

## 核心 insight

**汇编不是天书，它是有规律的。**

- 函数调用有约定（前 6 个整型参数：rdi, rsi, rdx, rcx, r8, r9）
- 循环有模式（`cmp` + `jl/jle/jg/jge`）
- 条件判断有套路（`test` + `je/jne`）
- 字符串比较用 `strcmp` (调用 libc)
- 字符串字面量用 `lea 0x1234(%rip), %rsi` 加载

**读懂汇编，你就能理解编译器在做什么、程序在怎么运行。**

## AI 帮你做什么

**1. 让 AI 反汇编整个 bomb binary**

```bash
$ objdump -d bomb > bomb.asm
$ objdump -s -j .rodata bomb   # 看到字符串常量
```

让 LLM 解释每个 phase 函数。重点：

- `phase_1` 字符串比较
- `phase_2` 循环 + 数组
- `phase_3` switch
- `phase_4` 递归
- `phase_5` 链表
- `phase_6` 排序

**2. 让 AI 用 Ghidra/IDA 反编译**

让 LLM 解释反编译出来的伪代码。这件事比直接看汇编更易读——AI 帮你把汇编翻译成
C-like 伪代码。

**3. 让 AI 对比 -O0 和 -O3 的反汇编**

```bash
$ clang -O0 -S bomb_source.c
$ clang -O3 -S bomb_source.c
```

让 LLM 解释：哪些函数被内联？哪些循环被向量化？哪些死代码被消除？

## 你应该能回答的判断题

1. "怎么从汇编里找到字符串常量？" 答：`lea 0x1234(%rip), %rsi` 或 `objdump -s -j .rodata`
2. "怎么从汇编里看到循环结构？" 答：找 `cmp + ` inconditional-jump 循环开头，中间是循环体。
4. "x86-64 函数调用约定是什么？" 答：前 6 个整型参数 rdi/rsi/rdx/rcx/r8/r9，返回值 rax。
5. "为什么 jne 而不是 jz？" 答：`test %eax, %eax` 后 ZF 标记位置位，`je` 检查 ZF=1，`jne` 检查 ZF=0。

## 经典 phase 模式

### Phase 1：字符串比较

```asm
phase_1:
    lea    0x1234(%rip), %rsi    ; 加载字符串字面量
    call   strcmp                 ; 调用 libc
    test   %eax, %eax
    je     .Lpassed
    call   explode_bomb
.Lpassed:
    ret
```

读到 `call strcmp` + `lea` → 字符串比较；第二个参数 `%rsi` 来自 `%lea` → 字符串字面量。

**判断题**：上面 phase_1 的字符串字面量是哪个？答：从 `lea 0x1234(%rip)` 看地址，从
`objdump -s -j .rodata bomb` 看那个地址的内容。

### Phase 2：循环 + 数组

```asm
; 假设 read_six_numbers(input, nums)
; 然后检查 nums[i] = nums[i-1] * 2
```

读循环结构：`cmp` + `jle`，每次循环体里有 `imul`/`shl` 做乘法。

**判断题**：phase_2 输入的前 6 个数是什么？答：每个是前一个的 2 倍，比如 "1 2 4 8 16 32"。

### Phase 3：switch

```asm
; switch (x) {
;     case 0: ...
;     case 1: ...
;     ...
; }
; 通常用 jump table
```

读 `jmp *0x1234(,%rax,8)` → 这是 jump table。jump table 地址是数据段。

**判断题**：怎么从汇编找到 jump table？答：`jmp *0x1234(,%rax,8)`。

### Phase 4：递归

```asm
func_0:
    cmp $0, %rdi
    je .Lbase
    ; ... 调用自己 ...
    ret
.Lbase:
    mov $42, %rax
    ret
```

递归有 `call 自己` + base case。

### Phase 5：链表

```asm
; node->next 偏移 + node->value 偏移
```

读链表节点结构 + 遍历（`mov` + `test` + `je`）。

### Phase 6：排序

```asm
; 通常有 swap 函数 + 多个 cmp
```

读 `cmov` 或 `cmp + jmp` 看排序规则。

## 本实验 takeaway

1. 汇编有规律——调用约定、字符串函数、循环结构都是规则手册
2. 反汇编工具（objdump、Ghidra）是你的眼睛
3. AI 能帮你反汇编 + 解释，你的工作是判断 + 验证
4. 调试、性能分析、安全分析都需要读汇编

**核心 insight：汇编是 CPU 的"源码"。读懂它，你才能理解程序到底在做什么。AI
能帮你读，但要你做判断。**