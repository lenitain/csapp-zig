# Bomb Lab

**对应章节：** 第三讲（汇编）

**一句话点题：** 读懂汇编就是读懂 CPU 在想什么。

## 为什么要做这个实验

CSAPP 的 Bomb Lab 给你一个二进制可执行文件（"炸弹"），你需要通过反汇编分析找到正确的输入序列来"拆除"它。炸弹有多个阶段，每个阶段需要特定的输入。

在 AI 时代，这个实验的"解题"价值趋近于零——AI 可以直接分析二进制文件。但实验传达的**核心 insight** 仍然重要：

**汇编不是天书，它是有规律的。** 函数调用有约定，循环有模式，条件判断有套路。读懂汇编，你就能理解编译器在做什么、程序在怎么运行。

## 一个简化版的炸弹

假设我们有这样一个 C 程序编译后的二进制：

```c
void phase_1(char *input) {
    if (strcmp(input, "Strings are fun!") != 0) {
        explode_bomb();
    }
}
```

### 反汇编看到的

```asm
phase_1:
    sub    $0x8, %rsp
    lea    0x1234(%rip), %rsi    ; "Strings are fun!"
    call   strcmp
    test   %eax, %eax
    je     .Lpassed
    call   explode_bomb
.Lpassed:
    add    $0x8, %rsp
    ret
```

### 思路

1. 看到 `call strcmp`，知道是比较两个字符串
2. 第二个参数在 `%rsi`（x86-64 调用约定），来自 `lea` 指令
3. `lea 0x1234(%rip), %rsi` 是加载一个字符串常量的地址
4. `test %eax, %eax` 检查 strcmp 的返回值
5. `je .Lpassed` 如果返回 0（相等），跳过爆炸

**解题：输入 "Strings are fun!" 就行了。**

## 更复杂的阶段：循环和数组

```c
void phase_2(char *input) {
    int nums[6];
    read_six_numbers(input, nums);
    for (int i = 1; i < 6; i++) {
        if (nums[i] != nums[i-1] * 2) {
            explode_bomb();
        }
    }
}
```

### 思路

1. 看到 `call read_six_numbers`，知道要输入 6 个数
2. 看到循环结构（`cmp` + `jl`），知道在遍历数组
3. 看到 `imul` 或 `shl`，知道在做乘法
4. 看到比较和跳转，知道在检查条件

**解题：输入 6 个数，每个是前一个的 2 倍。比如 "1 2 4 8 16 32"。**

## 本实验的 takeaway

1. 汇编有规律：函数调用有约定，循环有模式，条件判断有套路
2. 反汇编工具（objdump、GDB、Ghidra）是你的朋友
3. 编译器的输出是可预测的——你知道 C 代码长什么样，就能猜到汇编长什么样
4. 在 AI 时代，你不需要手写反汇编，但你需要能看懂——调试、性能分析、安全分析都需要

**核心 insight：汇编是 CPU 的"源码"。理解汇编，你就能理解程序到底在做什么。**
