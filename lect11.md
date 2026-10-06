# CSAPP 第十一讲：网络编程

## 物理原理：网络也是文件描述符

Unix "一切皆文件"延伸到网络——**socket 也是 fd**。`read`/`write` 收发数据，和读写文件是
同一套 API。

```
socket()     → 创建 socket，返回 fd
bind()       → 绑定本地地址
listen()     → 服务器监听连接
accept()     → 服务器接受连接，返回新 fd
connect()    → 客户端连接服务器
read/write   → 收发数据
close()      → 关闭
```

**这件事的物理基础**：网卡是 I/O 设备，OS 用同一个"打开文件表"抽象所有 I/O 源。socket 是
**文件 + 网络协议栈**的复合。

### 字节序：硬件大小端不统一

网络协议规定**网络字节序是大端** (big-endian)。x86 是小端。ARM 默认小端。

```c
addr.sin_port = htons(8080);  // host-to-network-short
```

`htons` 把主机字节序转网络字节序。**这件事是设计协议时决定的**——网络协议选大端作为标准。

## C 的设计选择：sockaddr 强转 + 字节序易忘

### 类型不安全的 API

```c
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
addr.sin_addr.s_addr = INADDR_ANY;

bind(sockfd, (struct sockaddr *)&addr, sizeof(addr));  // 强转
```

C 用 `struct sockaddr *` 作为通用类型，实际是 `sockaddr_in` 或 `sockaddr_in6`。**强转绕过了
类型系统**——传错类型没事，运行时才报。

### IPv4/IPv6 两套 API

```c
struct sockaddr_in addr4;   // IPv4
struct sockaddr_in6 addr6; // IPv6
```

两套结构、两套函数。**C 的 socket API 是 IPv4 时代发明的，IPv6 补丁上去的**——不是统一
设计。

### 阻塞 I/O 是默认

```c
int n = read(sockfd, buf, sizeof(buf));  // 没数据时阻塞
```

socket 默认是阻塞模式。**这是 1972 年 Unix 哲学**——IO 是同步的，OS 调度任务管理代码。

## 现代视角：类型安全 socket + async

### 类型安全的 socket

```rust
use tokio::net::TcpListener;

let listener = TcpListener::bind("127.0.0.1:8080").await?;
loop {
    let (socket, addr) = listener.accept().await?;
    tokio::spawn(handle(socket));
}
```

Rust 的 socket API：

- IPv4/IPv6 统一 `SocketAddr` 枚举
- 自动字节序转换
- 编译期类型检查

### 非阻塞 I/O 默认

```rust
// tokio 的 socket 默认非阻塞
// 事件循环 + epoll/kqueue 调度
```

现代 async runtime 把 socket 默认设成非阻塞，事件循环调度。**这是 OS epoll/kqueue 的
物理能力**——Linux 2.6+、BSD/macOS 都支持。

### `BufReader` 适配 socket

```rust
let mut buf_reader = tokio::io::BufReader::new(socket);
let mut line = String::new();
buf_reader.read_line(&mut line).await?;
```

显式缓冲，**自动处理短读**。C 的 `read` 短读得自己 while 循环。

## HTTP：文本协议

```
GET /index.html HTTP/1.1
Host: www.example.com
Connection: close

```

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 13

Hello, World!
```

HTTP 是**文本的**——可以用 `tcpdump` 抓包看，可以用 `curl` 手动发。

**这是协议设计的取舍**：

- **文本**：人类可读、易调试、容易写 parser（`sscanf` / regex）
- **二进制**：更紧凑、parse 更快

HTTP 选文本是为了早期 Web 易调试。

### 简单的 HTTP server (Rust)

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    loop {
        let (mut socket, _) = listener.accept().await?;
        tokio::spawn(async move {
            let mut buf = [0; 1024];
            socket.read(&mut buf).await.unwrap();
            let response = "HTTP/1.1 200 OK\r\nContent-Length: 13\r\n\r\nHello, World!";
            socket.write_all(response.as_bytes()).await.unwrap();
        });
    }
}
```

20 多行代码，但能处理真实 HTTP 请求。

## AI 时代怎么验证

**1. 让 AI 用 tcpdump 抓 HTTP 流量**

```bash
$ tcpdump -i lo -A port 8080
```

让 LLM 解释：哪是请求头？哪是响应体？`Content-Length` 为什么是 13？`Connection: close` 什么意思？

**2. 让 AI 跑 strace 看 HTTP server 的 syscall**

```bash
$ strace -e trace=accept,read,write,clone ./http_server
```

让 LLM 解释：accept 返回什么？fork/clone 在哪？为什么 `read` 可能返回短读？

**3. 让 AI 对比 epoll vs select vs pthread**

```bash
# 让 AI 写三个版本的 echo server
# - select
# - epoll
# - pthread (每连接一个线程)
# 各跑 10000 并发
```

让 LLM 解释：epoll 为什么 epoll 优于 select？pthread 的栈消耗是多少？fd 限制在哪？

**4. 让 AI 解释字节序**

```python
import struct
# 让 AI 解释为什么 8080 在 wire 上是 1F 90
```

让 LLM 解释：`htons(8080)` 在 x86 上的字节是什么？为什么 x86 和 PowerPC 看到的不一样？

## 本讲要点

1. socket 是 fd——网络 I/O 和文件 I/O 是同一套 API
2. 网络字节序是大端，x86 是小端——`htons`/`ntohl` 是必要桥接
3. C 的 socket API 是 IPv4 时代的——IPv6 补丁、强转、阻塞默认
4. Rust/tokio 是类型安全 + async + 非阻塞默认——OS epoll/kqueue 的能力
5. HTTP 是文本协议——可调试但 parse 慢；现代 binary 协议（gRPC/QUIC）换紧凑
6. AI 时代 tcpdump + strace 是调试网络的必备工具

## 下一讲

网络让程序可以和远程的程序通信。但本地的程序之间也需要通信——下一讲看并发编程，以及为什么
"共享内存"既是并发的基础，也是并发 bug 的根源。