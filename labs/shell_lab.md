# Shell Lab

**对应章节：** 第八讲（异常控制流）

**一句话点题：** Shell 不是一个神秘东西。Shell 就是一个循环：读命令 → fork → exec → 等待。

## 为什么不手写 shell

原 CSAPP Shell Lab 让你实现一个简单的 Unix shell，支持前后台进程、信号处理、作业控制。

**AI 时代这件事的价值是"理解"而不是"动手"**——让 LLM 写一个简单 shell 是几分钟的事。**你
的工作是理解 fork/exec 的时序、信号处理为什么是回调地狱、作业控制的状态机。**

## 核心 insight

**Shell 就是一个普通程序。** 它用 OS 提供的 API（fork、exec、wait、signal）实现进程管理。

```
while (1) {
    printf("> ");
    fgets(cmdline, MAXLINE, stdin);
    eval(cmdline);  // 解析 → fork → exec → wait
}
```

核心逻辑：

1. **读**：从 stdin 读命令
2. **解析**：拆成 argv，判断是否后台（`&` 结尾）
3. **fork**：创建子进程
4. **exec**：子进程执行新程序
5. **wait**：父进程等子进程结束（前台）

## AI 帮你做什么

**1. 让 AI strace 一个 shell**

```bash
$ strace -f -e trace=clone,execve,wait4 bash -c "ls /tmp"
```

让 LLM 解释：

- `clone` 系统调用是什么？怎么创建子进程？
- `execve` 替换地址空间——确认子进程从哪一步开始跑 `ls`
- `wait4` 怎么等待——子进程 exit status 怎么传回

**2. 让 AI 解释 fork 的写时拷贝 (COW)**

```bash
$ strace -e trace=clone,mmap bash -c "ls /tmp"
```

让 LLM 解释：

- fork 后父子进程地址空间怎么共享？
- 写时拷贝（copy-on-write）在哪一刻发生？
- 为什么 fork 几乎没成本？

**3. 让 AI 追踪 SIGCHLD**

```bash
$ strace -e trace=clone,execve,wait4,rt_sigaction bash -c "ls /tmp &"
```

让 LLM 解释：

- SIGCHLD 在什么时候触发？
- handler 的执行顺序——什么时候 waitpid 能拿到子进程状态？
- 为什么信号不能排队（标准信号）

**4. 让 AI 写一个简单 shell 让 AI 调试**

```bash
# 写一个 shell:
# - 读命令
# - 处理 SIGINT（不杀死 shell 自己）
# - 处理 SIGCHLD（reap 子进程）
# - 内建命令 (jobs, fg, bg, quit)
```

让 LLM 写一遍，**你**去解释每一段为什么这样写。

## 你应该能回答的判断题

1. "fork + exec 为什么分开？" 答：fork 复制进程，exec 替换地址空间。分开能实现任意进程模式（后台、shell 管道、守护进程）。
2. "shell 怎么处理 Ctrl-C？" 答：父进程（shell 自己）忽略 SIGINT，子进程继承这个设置但用 exec 重置为 SIG_DFL。
4. "waitpid 怎么等子进程结束？" 答：阻塞直到子进程 exit 或被信号中断。`WNOHANG` 让它非阻塞。
5. "为什么 SIGCHLD 重要？" 答：通知 shell 子进程状态变化，让 shell 回收僵尸进程。

## 经典模式

### 模式一：简单的 eval()

```c
void eval(char *cmdline) {
    char *argv[MAXARGS];
    int bg = parseline(cmdline, argv);
    if (argv[0] == NULL) return;  // 空命令

    pid_t pid = Fork();
    if (pid == 0) {  // 子进程
        Execve(argv[0], argv, environ);
    }

    if (!bg) {  // 前台：父进程等
        int status;
        waitpid(pid, &status, 0);
    } else {
        printf("%d %s", pid, cmdline);  // 后台
    }
}
```

**核心**：fork + parent + exec + wait。在 bash 里 fork/exec/wait 是异步 shell pipeline 的基础。

### 模式二：信号处理

```c
// shell 本身不应该被 Ctrl-C 杀死
Signal(SIGINT, SIG_IGN);

// 子进程继承这个 SIG_IGN
// 但 exec 会把 SIG_IGN 重置为 SIG_DFL（默认行为）

// 子进程退出时触发 SIGCHLD
Signal(SIGCHLD, reap);  // reap 调用 waitpid 回收
```

**核心**：信号 handler 是回调地狱——不能在 handler 里调 printf、malloc。

### 模式三：作业控制

```c
// bg %1: 把 job 1 送到后台
// fg %1: 把 job 1 送到前台
// jobs: 列出所有 job
```

shell 维护一个 job list，每个 job 有 pid、status (running/stopped/done)。

SIGCHLD 通知 shell 子进程状态变化，shell 更新 job list。

**核心**：状态机——每个 job 状态由 SIGCHLD 转换。

## 本实验 takeaway

1. shell 是 fork + exec + wait 的循环——没有魔法
2. fork 复制进程、exec 替换地址空间，分开是任意进程模式的基础
3. 信号是异步的——handler 是回调地狱，必须 async-signal-safe
4. 作业控制是状态机——SIGCHLD 驱动状态转换
5. AI 帮你写 shell，但你要能解释 fork/exec/wait 的 syscall 序列

**核心 insight：Shell 不是魔法。理解它，你就理解了进程管理、信号、I/O 重定向的全部。**