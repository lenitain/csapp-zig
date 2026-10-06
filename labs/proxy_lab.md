# Proxy Lab

**对应章节：** 第十、十一、十二讲（I/O、网络编程、并发）

**一句话点题：** Web 代理是"中间人"——把客户端请求转发给后端服务器，再把响应返回给客户端。

## 为什么不手写代理

原 CSAPP Proxy Lab 让你实现一个 HTTP/1.0 Web 代理，支持并发连接和缓存。

**AI 时代这件事的价值是"理解 I/O + 网络 + 并发三个章节怎么串起来"**——让 LLM 写一个代理是几
分钟的事。**你的工作是解释每一段 in 怎么样、select/epoll/pthread 的 trade-off、HTTP 协议
解析的边界 case。**

## 核心 insight

**代理就是把三个章节的内容串起来：**

- **系统级 I/O**：读写 socket
- **网络编程**：HTTP 协议解析
- **并发编程**：多客户端处理

理解代理，**你就理解了一个完整的网络应用是怎么工作的**。

## AI 帮你做什么

**1. 让 AI 写一个简单 proxy 让 AI 解释**

```bash
# 让 AI 用 Rust + tokio 写一个 echo proxy
# - 监听端口
# - 接受客户端连接
# - 连接到后端
# - 双向转发
```

让 LLM 解释：

- 为什么用 tokio 而不是 pthread？
- `tokio::spawn` 怎么并发？
- `io::copy_bidirectional` 怎么双向转发？

**2. 让 AI 跑 strace 看 proxy 行为**

```bash
$ strace -f -e trace=accept,clone,read,write,recvfrom,sendto ./proxy
$ curl http://localhost:8080/httpbin.org/get
```

让 LLM 解释：

- accept 返回新连接的 fd
- clone 创建处理线程（或 fork）
- read 从 client 读 HTTP 请求
- recvfrom 向 backend 发请求
- sendto 把 backend 响应写回 client

**3. 让 AI 对比 select vs epoll vs pthread**

```bash
# 让 AI 写三个版本的 echo server：
# - select
# - epoll
# - pthread (每连接一个线程)
# 各跑 1000 并发连接

$ ab -n 10000 -c 1000 http://localhost:8080/
```

让 LLM 解释：

- select 的 fd_set 限制？1024 fd（FD_SETSIZE）
- epoll 为什么能处理 10 万 fd？
- pthread 的栈消耗？每线程 8MB 默认 → 1万线程 = 80GB
- tokio/async 为什么内存占用低？

**4. 让 AI 解释 HTTP 协议**

```bash
$ curl -v http://localhost:8080/
```

让 LLM 解释：

- Request line 是什么？
- Header 格式？
- Body 怎么分隔（`Content-Length` 或 `Transfer-Encoding: chunked`）？
- HTTP/1.0 vs HTTP/1.1 vs HTTP/3？

## 你应该能回答的判断题

1. "代理的工作原理是什么？" 答：accept client → read request → connect backend → forward request → read response → forward response → close。
2. "proxy 怎么处理并发？" 答：每连接一个线程/fork，或 I/O 多路复用。
3. "为什么 epoll 优于 select？" 答：select O(n) 扫所有 fd，epoll 用 callback + 事件驱动。
4. "HTTP 怎么分请求和响应？" 答：CRLF 分隔。`\r\n\r\n` 是 header/body 边界。
5. "proxy 为什么要缓存？" 答：同一资源多个客户端请求时，省一次 backend 网络往返。
6. "I/O 多路复用的限制是什么？" 答：单线程，没法多核并行（除非 multi-reactor 模型）。

## 经典模式

### 模式一：单线程 proxy（基础）

```c
while (1) {
    int connfd = accept(listenfd, ...);
    doit(connfd);  // 同步处理请求
    close(connfd);
}

void doit(int fd) {
    // 1. 读 client HTTP 请求
    // 2. 解析 URL → hostname/path/port
    // 3. connect backend
    // 4. forward request
    // 5. read response
    // 6. forward response
    // 7. close(client)
}
```

**核心**：accept loop + 同步处理。简单但慢——一个慢 client 阻塞所有其他人。

### 模式二：多线程 proxy

```c
while (1) {
    int connfd = accept(listenfd, ...);
    pthread_create(&tid, NULL, thread, &connfd);
}

void *thread(void *vargp) {
    int connfd = *((int *)vargp);
    pthread_detach(pthread_self());
    doit(connfd);
    close(connfd);
}
```

**核心**：每连接一个线程。简单但有栈消耗。

### 模式三：I/O 多路复用（epoll）

```c
struct epoll_event events[MAX_EVENTS];
int epfd = epoll_create1(0);
epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev);

while (1) {
    int n = epoll_wait(epfd, events, MAX_EVENTS, -1);
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == listenfd) {
            int connfd = accept(listenfd, ...);
            epoll_ctl(epfd, EPOLL_CTL_ADD, connfd, &ev);
        } else {
            handle(events[i].data.fd);
        }
    }
}
```

**核心**：epoll_wait 阻塞，等事件；事件就绪才调 handler。**这是事件驱动的核心模式**。

### 模式四：现代 async / proxy

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    loop {
        let (client, _) = listener.accept().await?;
        tokio::spawn(async move {
            let backend = TcpStream::connect("backend:80").await.unwrap();
            let (mut cr, mut cw) = client.into_split();
            let (mut br, mut bw) = backend.into_split();

            tokio::spawn(async move {
                tokio::io::copy(&mut cr, &mut bw).await.unwrap();
            });
            tokio::spawn(async move {
                tokio::io::copy(&mut br, &mut cw).await.unwrap();
            });
        });
    }
}
```

**核心**：async runtime + 双工转发。Nagle 算法禁用 + async 实现高性能 proxy。

## 本实验 takeaway

1. 代理 = I/O + 网络 + 并发三个章节的串接
2. 单线程简单但慢，多线程有栈消耗，epoll 事件驱动高性能
3. async runtime 把状态机从程序员手里抢过来——少线程高并发
4. AI 帮你写 proxy，但你要能解释每段 in 怎么样
5. HTTP 是文本协议——易调试但 parse 慢；HTTP/2 是二进制，更快

**核心 insight：代理把三个章节的内容串在一起。理解它，你就理解了一个完整的网络应用怎么工作。
AI 能帮你写，但你不能不懂原理。**