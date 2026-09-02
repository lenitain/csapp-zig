# Attack Lab

**对应章节：** 第三讲（汇编）

**一句话点题：** 缓冲区溢出攻击的本质是"让 CPU 跳到你指定的地方执行"。

## 为什么要做这个实验

CSAPP 的 Attack Lab 让你利用缓冲区溢出漏洞，注入代码或构造 ROP (Return-Oriented
Programming) 链来执行任意代码。

在 AI 时代，手写攻击 payload
的价值趋近于零——安全工具可以自动生成。但实验传达的**核心 insight** 仍然重要：

**程序的控制流是可以被篡改的。**
如果你能覆盖返回地址，你就能让程序跳到任何地方。这是所有代码执行漏洞的共同本质。

## 攻击方式一：代码注入

### 原理

```c
void vulnerable() {
    char buf[48];
    gets(buf);  // 没有长度限制
}
```

栈布局：

```text
[返回地址]      ← 高地址
[保存的 rbp]
[buf[48]]       ← 低地址
```

如果输入超过 48 字节，会覆盖 rbp 和返回地址。我们可以：

1. 在 buf 里写入 shellcode（恶意代码）
2. 把返回地址改成 buf 的地址
3. 函数返回时，CPU 跳到 shellcode 执行

### 思路

- 用 GDB 找到 buf 在栈上的地址
- 构造 payload：48 字节填充 + 8 字节覆盖 rbp + 8 字节返回地址（指向 buf）
- 在 buf 的前面放 shellcode

### 为什么现代系统已经防住了

- **NX/DEP**：栈上的数据不可执行，shellcode 无法运行
- **ASLR**：栈地址随机化，你不知道 buf 在哪
- **Stack Canary**：返回地址前放随机值，溢出时会检测到

## 攻击方式二：ROP (Return-Oriented Programming)

NX/DEP 防住了代码注入，但 ROP 绕过了它。

### 原理

不注入新代码，而是**复用已有代码的片段** (gadget)。每个 gadget 是一小段以 `ret`
结尾的代码：

```asm
; gadget 1: pop %rdi; ret
5f    pop %rdi
c3    ret

; gadget 2: pop %rax; ret
58    pop %rax
c3    ret
```

通过精心构造栈上的数据，让 `ret` 指令跳到一个 gadget，执行后 `ret` 跳到下一个
gadget，形成一个"链"。

### 思路

1. 在程序的代码段或共享库中找到有用的 gadget
2. 构造 ROP 链：[gadget1 地址][参数1][gadget2 地址][参数2]...
3. 覆盖返回地址指向链的开头
4. 程序返回时，沿着链执行 gadget 序列

### 为什么这很重要

ROP 是现代漏洞利用的核心技术。理解 ROP，你才能理解：

- 为什么 ASLR 重要（随机化 gadget 地址）
- 为什么 CFI (Control Flow Integrity) 重要（限制 ret 的跳转目标）
- 为什么 Zig 的边界检查从源头防止了溢出

## 本实验的 takeaway

1. 缓冲区溢出的本质是覆盖返回地址，控制程序的跳转目标
2. 代码注入是最简单的攻击，但 NX/DEP 防住了它
3. ROP 复用已有代码的片段，绕过了 NX/DEP
4. 现代防御（ASLR、CFI、Stack Canary）是针对这些攻击的对策
5. Zig 的边界检查从源头防止了溢出——不需要事后防御

**核心
insight：安全漏洞的本质是"程序做了你没打算让它做的事"。类型安全、边界检查、显式错误处理都是防止这种情况的手段。**
