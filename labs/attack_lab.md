# Attack Lab

**对应章节：** 第三讲（汇编）

**一句话点题：** 缓冲区溢出攻击的本质是"让 CPU 跳到你指定的地方执行"。

## 为什么不手写 payload

原 CSAPP Attack Lab 让你利用缓冲区溢出漏洞，注入代码或构造 ROP 链。

**AI 时代这件事的价值是"理解"而不是"动手"**——让 LLM 生成 payload 几秒钟的事。**你的工作是
解释每一行 payload 在做什么、为什么这样构造、现代防御怎么挡它。**

## 核心 insight

**程序的控制流是可以被篡改的。**

如果你能覆盖返回地址，你就能让程序跳到任何地方。这是所有代码执行漏洞的共同本质——**程序
做了你没打算让它做的事**。

类型系统安全、显式错误处理、边界检查都是防止这种情况的手段。

## AI 帮你做什么

**1. 让 AI 解释栈溢出时的 stack 行为**

让 LLM 画一个典型的栈布局：

```
高地址
  [调用者之前的位置]
  [返回地址]      ← call 写入
  [saved rbp]
  [buf[N]]        ← 低地址，gets/strcpy 目标
低地址
```

让 LLM 解释：为什么输入超过 48 字节会覆盖返回地址？saved rbp 在哪？返回地址在哪？

**2. 让 AI 生成简单的栈溢出 payload**

```bash
$ gdb -q ./vuln
> p &buf                # 看 buf 地址
> run <<< "AAAAAAAAAA...shellcode...new_ret_addr"
```

让 LLM 解释：

- payload 怎么构造？
- 怎么覆盖 saved rbp？（48 字节 buf + 8 字节 rbp 填充）
- 怎么覆盖返回地址？（再 8 字节是 ret addr）
- 返回地址指向哪？（buf 地址，但需要 ASLR 关闭或猜地址）

**3. 让 AI 解释 ROP gadget**

让 LLM 描述：

```asm
; gadget 1: pop %rdi; ret
; gadget 2: pop %rsi; ret
; gadget 3: call system
```

让 LLM 解释：怎么从 libc 找这些 gadget？`ROPgadget` 工具怎么用？怎么把 gadget 串成 ROP 链？

**4. 让 AI 解释现代防御**

让 LLM 解释：

- **NX/DEP**：栈上的数据不可执行——`mprotect(栈地址, PROT_READ | PROT_WRITE)`
- **ASLR**：栈地址随机化——`/proc/sys/kernel/randomize_va_space`
- **Stack Canary**：返回地址前放随机值——`__stack_chk_fail`
- **CFI**：限制 ret 的跳转目标——`__cfi_slowpath`

## 你应该能回答的判断题

1. "栈溢出攻击的本质是什么？" 答：覆盖返回地址，让 CPU 跳到攻击者指定的位置。
3. "代码注入和 ROP 的区别？" 答：代码注入是执行新代码，ROP 是复用已有 gadget。
4. "NX/DEP 防什么？" 答：栈上的数据不可执行，防代码注入。
5. "ASLR 防什么？" 答：地址随机化，攻击者不知道 shellcode 在哪。
6. "Stack Canary 防什么？" 答：检测返回地址被覆盖——canary 值被篡改就 abort。
7. "CFI 防什么？" 答：限制 ret 的合法目标，防止任意跳转。

## 攻击方式对比

### 方式一：代码注入

```text
栈布局（攻击前）：
  [返回地址: 0x401234]   ← call 写入
  [saved rbp: 0x7fff..]
  [buf[48]: 未初始化]
栈布局（攻击后）：
  [返回地址: 0x7fff..]   ← 改成 buf 地址（指向 shellcode）
  [saved rbp: junk]
  [buf[48]: shellcode ... + 填充]   ← shellcode 在段前
```

**核心**：buf 里放 shellcode，返回地址改成 buf 地址。函数返回时 CPU 跳到 shellcode 执行。

**现代防**：

- NX/DEP 让栈不可执行，shellcode 不能跑
- ASLR 让 buf 地址不可预测
- Stack Canary 让覆盖被检测

### 方式二：ROP (Return-Oriented Programming)

```text
栈布局（ROP 攻击）：
  [返回地址: gadget_1_addr]
  [gadget_1 的参数]
  [gadget_2_addr]
  [gadget_2 的参数]
  ...
  [最终函数地址]
```

**核心**：不注入代码，只复用 `mprotect`、pop、`syscall` 等已存在的 gadget。**NX 挡不住 ROP
因为它执行的是已有代码**。

**现代防**：

- CFI (Control Flow Integrity) 限制 ret 的合法目标
- Shadow Stack 让 ret 跳栈上记录的地址（不是攻击者覆盖的地址）

## 防御层次

```
层次 1：语言层默认安全
  Rust/Zig slice 默认边界检查，从源头防止溢出
层次 2：编译器层加固
  Stack Canary (-fstack-protector-strong)
  CFI (-fsanitize=cfi)
层次 3：操作系统层加固
  ASLR
  NX/DEP
层次 4：硬件层加固
  Intel CET (Control-flow Enforcement Technology) — Shadow Stack + IBT
```

**核心 insight**：安全是层次防御，不是单一手段。语言层挡住 90%，编译器层挡住 8%，OS
和硬件层挡住剩下的。

## 本实验 takeaway

1. 控制流可被篡改是攻击的本质
2. NX/DEP、Canary、ASLR、CFI 是四个层次的防御
3. 语言层（Rust/Zig）从源头挡是成本最低的方案
4. AI 帮你生成 payload，但你要能解释每一行在做什么
5. 现代防御不是单一手段，是 language + compiler + OS + hardware 的多层次

**核心 insight：安全漏洞的本质是"程序做了你没打算让它做的事"。类型安全、显式错误处理、边界
检查都是防止这种情况的手段。**