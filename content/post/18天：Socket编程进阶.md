---
title: 18天：Socket编程进阶
date: 2025-11-10 15:24:38
categories: Network 
tags: Network, Book, Python
---
# 第18天：Socket编程进阶

## 今日目标

- 理解阻塞和非阻塞Socket
- 掌握IO多路复用技术（select/poll/epoll）
- 学习Socket选项配置
- 了解高性能服务器设计模式
- 实现多客户端并发服务器
- 构建实战项目：多人聊天室

---

## 1. 阻塞与非阻塞Socket

### 1.1 什么是阻塞？

**阻塞（Blocking）**：

```python
生活类比：
你去银行取钱（阻塞模式）：
1. 排队等待
2. 叫到你的号之前，你只能等待
3. 什么也做不了
4. 叫到号了才能办理业务

阻塞Socket：
sock.recv(1024)
↓
如果没有数据，就一直等待
什么也做不了
直到有数据到达
```

**阻塞Socket示例**：

```python
import socket

# 默认是阻塞模式
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(('www.baidu.com', 80))

# 发送请求
sock.send(b'GET / HTTP/1.1\r\nHost: www.baidu.com\r\n\r\n')

# 阻塞在这里，直到收到数据
data = sock.recv(1024)  # ⏸️ 阻塞等待
print(data)

sock.close()
```

**阻塞的问题**：

```
单线程服务器：

while True:
    client_sock, addr = server_sock.accept()  # 阻塞1
    data = client_sock.recv(1024)             # 阻塞2
    client_sock.send(response)
    client_sock.close()

问题：
- accept()阻塞 → 只能等待一个客户端
- recv()阻塞 → 处理一个客户端时，其他客户端无法连接
- 无法同时服务多个客户端 ❌
```

### 1.2 非阻塞Socket

**非阻塞（Non-blocking）**：

```python
生活类比：
你去银行取钱（非阻塞模式）：
1. 取号
2. 没叫到号，去旁边玩手机
3. 每隔几分钟看一眼叫号屏幕
4. 叫到号了再去办理

非阻塞Socket：
sock.setblocking(False)
try:
    data = sock.recv(1024)
except BlockingIOError:
    # 没有数据，立即返回
    # 可以做其他事情
    pass
```

**非阻塞Socket示例**：

```python
import socket
import errno

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# 设置为非阻塞
sock.setblocking(False)

try:
    sock.connect(('www.baidu.com', 80))
except BlockingIOError:
    # 连接正在进行中
    pass

# 发送数据
try:
    sock.send(b'GET / HTTP/1.1\r\nHost: www.baidu.com\r\n\r\n')
except BlockingIOError:
    # 缓冲区满，暂时无法发送
    pass

# 接收数据
try:
    data = sock.recv(1024)
    print(data)
except BlockingIOError:
    # 没有数据可读
    print("暂时没有数据")
```

**非阻塞的问题**：

```python
# 需要不断轮询
while True:
    try:
        data = sock.recv(1024)
        if data:
            process(data)
    except BlockingIOError:
        pass  # 继续轮询

    # CPU 100%！不断循环检查 ❌
```

---

## 2. IO多路复用

### 2.1 什么是IO多路复用？

**IO多路复用（I/O Multiplexing）**：

```
问题：
如何同时监听多个Socket，而不需要为每个Socket创建线程？

解决方案：
IO多路复用 = 一个线程监听多个Socket

生活类比：
你是一个老师，要照顾30个学生：

方案1：阻塞（不现实）
- 盯着第1个学生，直到他提问
- 其他29个学生被忽略

方案2：轮询（CPU浪费）
- 不停地问每个学生："有问题吗？"
- 大部分时间都在问，很累

方案3：IO多路复用（最优）
- 坐在讲台上
- 谁举手（有数据），就处理谁
- 没人举手，就休息（阻塞在select上）
```

### 2.2 select

**select原理**：

```python
import select

# 要监听的socket列表
read_list = [sock1, sock2, sock3]
write_list = []
error_list = []

# 阻塞等待，直到有socket就绪
readable, writable, exceptional = select.select(
    read_list,
    write_list,
    error_list,
    timeout  # 超时时间，None表示一直阻塞
)

# 处理就绪的socket
for sock in readable:
    data = sock.recv(1024)
    process(data)
```

**select服务器示例**：

```python
import select
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('0.0.0.0', 8888))
server.listen(5)
server.setblocking(False)

# 要监听的socket列表
inputs = [server]
outputs = []

while inputs:
    # 等待至少一个socket就绪
    readable, writable, exceptional = select.select(
        inputs, outputs, inputs, 1.0
    )

    # 处理可读的socket
    for sock in readable:
        if sock is server:
            # 服务器socket就绪：有新连接
            client, addr = sock.accept()
            client.setblocking(False)
            inputs.append(client)
            print(f"新连接: {addr}")
        else:
            # 客户端socket就绪：有数据
            data = sock.recv(1024)
            if data:
                print(f"收到数据: {data}")
                # 发送响应
                sock.send(data)
            else:
                # 客户端断开
                print("客户端断开")
                inputs.remove(sock)
                sock.close()
```

**select的限制**：

```
1. 监听数量限制
   - Linux默认：1024个socket
   - 由FD_SETSIZE定义
   - 无法修改（编译时确定）

2. 性能问题
   - 每次调用select都要把fd_set从用户空间拷贝到内核
   - 内核需要遍历所有fd，检查是否就绪
   - O(n)复杂度

3. 不支持边缘触发
   - 只支持水平触发（Level Trigger）

4. 返回后需要遍历
   - select只告诉你"有socket就绪"
   - 不告诉你"哪些socket就绪"
   - 需要自己遍历所有socket检查
```

### 2.3 poll

**poll改进**：

```python
import select

# 创建poll对象
poll_obj = select.poll()

# 注册socket
poll_obj.register(sock1, select.POLLIN)
poll_obj.register(sock2, select.POLLIN | select.POLLOUT)

# 等待事件
events = poll_obj.poll(timeout)

# 处理事件
for fd, event in events:
    if event & select.POLLIN:
        # 可读
        data = sockets[fd].recv(1024)
    if event & select.POLLOUT:
        # 可写
        sockets[fd].send(data)
```

**poll vs select**：

```
poll的优点：
✅ 没有最大文件描述符限制
✅ 使用pollfd结构体数组，不需要每次都拷贝整个集合

poll的缺点：
❌ 仍然是O(n)复杂度
❌ 仍需要从用户空间拷贝到内核空间
❌ Windows不支持

共同点：
- 都是水平触发
- 都需要遍历所有fd
```

### 2.4 epoll（Linux专用）

**epoll的优势**：

```
epoll = 高性能的IO多路复用

优点：
✅ 没有最大文件描述符限制
✅ O(1)复杂度（只返回就绪的fd）
✅ 支持边缘触发（Edge Trigger）
✅ 使用内存映射（mmap），减少拷贝

epoll是Linux下高性能网络编程的首选！
```

**epoll使用**：

```python
import select

# 创建epoll对象
epoll = select.epoll()

# 注册socket
epoll.register(sock.fileno(), select.EPOLLIN)

# 等待事件
events = epoll.poll(timeout)

# events是已就绪的fd列表（不需要遍历所有fd）
for fd, event in events:
    if event & select.EPOLLIN:
        # 可读
        data = sockets[fd].recv(1024)
    if event & select.EPOLLOUT:
        # 可写
        sockets[fd].send(data)
```

**epoll服务器示例**：

```python
import select
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('0.0.0.0', 8888))
server.listen(5)
server.setblocking(False)

epoll = select.epoll()
epoll.register(server.fileno(), select.EPOLLIN)

connections = {}
requests = {}
responses = {}

try:
    while True:
        events = epoll.poll(1)

        for fileno, event in events:
            if fileno == server.fileno():
                # 新连接
                client, addr = server.accept()
                client.setblocking(False)
                epoll.register(client.fileno(), select.EPOLLIN)
                connections[client.fileno()] = client
                print(f"新连接: {addr}")

            elif event & select.EPOLLIN:
                # 可读
                data = connections[fileno].recv(1024)
                if data:
                    requests[fileno] = data
                    # 修改为等待可写
                    epoll.modify(fileno, select.EPOLLOUT)
                else:
                    # 断开连接
                    epoll.unregister(fileno)
                    connections[fileno].close()
                    del connections[fileno]

            elif event & select.EPOLLOUT:
                # 可写
                response = requests[fileno]
                connections[fileno].send(response)
                # 修改为等待可读
                epoll.modify(fileno, select.EPOLLIN)
                del requests[fileno]

finally:
    epoll.unregister(server.fileno())
    epoll.close()
    server.close()
```

**epoll触发模式**：

```
水平触发（Level Trigger，LT）：
- 默认模式
- 只要fd就绪，就会一直通知
- 安全但可能有性能损失

示例：
socket缓冲区有100字节数据
你只读了50字节
下次epoll_wait还会通知你（还有50字节未读）

边缘触发（Edge Trigger，ET）：
- 高性能模式
- 只在fd状态变化时通知一次
- 必须一次性处理完所有数据

示例：
socket缓冲区有100字节数据
你只读了50字节
下次epoll_wait不会通知你（直到有新数据到达）
必须循环读取直到EAGAIN错误

选择：
- LT：简单安全，适合大多数场景
- ET：高性能，需要小心处理
```

### 2.5 select/poll/epoll对比

```
┌──────────────┬──────────┬──────────┬──────────┐
│   特性       │  select  │   poll   │  epoll   │
├──────────────┼──────────┼──────────┼──────────┤
│ 最大连接数   │   1024   │   无限制 │  无限制  │
│ 复杂度       │   O(n)   │   O(n)   │   O(1)   │
│ 跨平台       │    ✅    │  部分✅  │    ❌    │
│ 性能         │    低    │    中    │    高    │
│ 边缘触发     │    ❌    │    ❌    │    ✅    │
│ 适用场景     │ 少量连接 │ 中等连接 │ 大量连接 │
└──────────────┴──────────┴──────────┴──────────┘

推荐使用：
- Linux：epoll（最优）
- Windows：IOCP（完成端口，不是select）
- 跨平台：poll（或高级框架如asyncio、Twisted）
```

---

## 3. Socket选项

### 3.1 常用Socket选项

**SO_REUSEADDR**：

```python
# 允许地址重用
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

作用：
1. 允许在TIME_WAIT状态下重新绑定端口
2. 多个socket绑定到同一个端口（不同接口）

场景：
服务器重启时，之前的连接可能还在TIME_WAIT状态
如果不设置此选项，bind()会失败："Address already in use"

推荐：
服务器应该总是设置此选项
```

**SO_REUSEPORT**：

```python
# 允许端口重用（负载均衡）
sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)

作用：
多个进程可以绑定到同一个端口
内核会自动负载均衡（分发连接）

使用场景：
多进程服务器，每个进程监听同一端口
```

**TCP_NODELAY**：

```python
# 禁用Nagle算法
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)

Nagle算法：
- 小数据包会被延迟发送，等待合并
- 减少网络包数量
- 但增加延迟

禁用场景：
- 交互式应用（SSH、Telnet）
- 低延迟要求（游戏、VoIP）
- 已经做了应用层缓冲

保留场景：
- 批量数据传输
- 对延迟不敏感
```

**SO_KEEPALIVE**：

```python
# 启用TCP保活机制
sock.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1)

作用：
定期发送探测包，检测连接是否还活着

参数（Linux）：
TCP_KEEPIDLE: 空闲多久后开始探测（默认2小时）
TCP_KEEPINTVL: 探测间隔（默认75秒）
TCP_KEEPCNT: 探测次数（默认9次）

示例：
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, 60)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPINTVL, 10)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPCNT, 3)
# 60秒后开始探测，每10秒一次，失败3次后关闭连接
```

**SO_RCVBUF / SO_SNDBUF**：

```python
# 设置接收/发送缓冲区大小
sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 128 * 1024)  # 128KB
sock.setsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF, 128 * 1024)

作用：
调整socket缓冲区大小

场景：
- 高带宽网络：增大缓冲区
- 低延迟要求：减小缓冲区
```

**SO_LINGER**：

```python
# 设置关闭时的行为
import struct
linger = struct.pack('ii', 1, 10)  # 启用，等待10秒
sock.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, linger)

参数：
l_onoff: 0=禁用, 1=启用
l_linger: 等待时间（秒）

行为：
l_onoff=0: close()立即返回，后台继续发送
l_onoff=1, l_linger>0: close()等待数据发送完或超时
l_onoff=1, l_linger=0: close()立即返回，发送RST（丢弃数据）

使用场景：
一般不设置，使用默认行为
```

### 3.2 获取Socket选项

```python
# 获取选项值
value = sock.getsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR)
print(f"SO_REUSEADDR: {value}")

# 获取缓冲区大小
rcvbuf = sock.getsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF)
sndbuf = sock.getsockopt(socket.SOL_SOCKET, socket.SO_SNDBUF)
print(f"接收缓冲区: {rcvbuf} 字节")
print(f"发送缓冲区: {sndbuf} 字节")
```

---

## 4. 高性能服务器设计模式

### 4.1 常见架构模式

**1. 单线程阻塞（最简单）**

```python
while True:
    client, addr = server.accept()  # 阻塞
    data = client.recv(1024)        # 阻塞
    client.send(response)
    client.close()

优点：
✅ 代码简单

缺点：
❌ 同一时间只能服务一个客户端
❌ 性能极差

适用：
- 学习demo
- 极少并发的场景
```

**2. 多线程（简单但有限制）**

```python
def handle_client(client_sock):
    data = client_sock.recv(1024)
    client_sock.send(response)
    client_sock.close()

while True:
    client, addr = server.accept()
    thread = threading.Thread(target=handle_client, args=(client,))
    thread.start()

优点：
✅ 可以同时处理多个客户端
✅ 代码相对简单

缺点：
❌ 线程开销大（内存、上下文切换）
❌ 最多几千个线程
❌ C10K问题（10000并发）

适用：
- 中小规模应用
- 每个连接处理时间较长
```

**3. 多进程（进程池）**

```python
from multiprocessing import Process

def worker(server_sock):
    while True:
        client, addr = server_sock.accept()
        handle_client(client)

# 创建多个工作进程
for i in range(4):
    p = Process(target=worker, args=(server,))
    p.start()

优点：
✅ 利用多核CPU
✅ 进程隔离，稳定性好

缺点：
❌ 进程开销更大
❌ 进程数量有限
❌ 进程间通信复杂

适用：
- CPU密集型任务
- 需要进程隔离
```

**4. IO多路复用 + 单线程（高性能）**

```python
epoll = select.epoll()
epoll.register(server.fileno(), select.EPOLLIN)

while True:
    events = epoll.poll()
    for fd, event in events:
        # 处理事件
        pass

优点：
✅ 高性能（C10K、C100K）
✅ 资源占用少
✅ 单线程，无锁

缺点：
❌ 代码复杂
❌ 回调地狱

适用：
- IO密集型
- 大量并发连接
- 高性能要求
```

**5. IO多路复用 + 线程池（推荐）**

```python
from concurrent.futures import ThreadPoolExecutor

executor = ThreadPoolExecutor(max_workers=10)
epoll = select.epoll()

while True:
    events = epoll.poll()
    for fd, event in events:
        # 提交到线程池处理
        executor.submit(handle_event, fd, event)

优点：
✅ 高并发（epoll）
✅ 利用多核（线程池）
✅ 代码相对简单

缺点：
⚠️ 需要处理线程安全

适用：
- 生产环境推荐
- IO + CPU混合负载
```

**6. Reactor模式（事件驱动）**

```
Reactor模式（Twisted、asyncio）：

组件：
1. Reactor: 事件循环
2. Handler: 事件处理器
3. Acceptor: 接受新连接

流程：
1. Reactor监听事件
2. 事件到达 → 调用对应Handler
3. Handler处理 → 返回Reactor

优点：
✅ 高性能
✅ 框架成熟

适用：
- 使用异步框架
- asyncio、Twisted、Tornado
```

### 4.2 C10K问题

```
C10K = 10000个并发连接

历史：
- 2000年前：Apache使用多线程/多进程
- 问题：10000个线程 → 内存耗尽、上下文切换频繁

解决方案：
1. Nginx: IO多路复用 + 事件驱动
2. 使用epoll（Linux）或kqueue（BSD）
3. 非阻塞IO + 状态机

现在：
- C10K已经不是问题
- C100K、C1000K（百万并发）成为新目标
- 需要更多优化：内核参数、零拷贝等
```

---

## 5. 实战项目：多人聊天室

### 5.1 项目需求

```
功能需求：
1. 支持多用户同时在线
2. 用户可以设置昵称
3. 消息广播给所有用户
4. 显示用户上线/下线通知
5. 查看在线用户列表
6. 私聊功能
7. 房间/频道功能

技术要求：
- 使用TCP协议
- 使用IO多路复用（epoll/select）
- 服务器单线程
- 支持至少100个并发连接
```

### 5.2 协议设计

```python
消息格式（JSON）：

1. 加入聊天室
{
    "type": "join",
    "nickname": "Alice"
}

2. 普通消息
{
    "type": "message",
    "content": "Hello everyone!"
}

3. 私聊
{
    "type": "private",
    "to": "Bob",
    "content": "Hi Bob!"
}

4. 列出用户
{
    "type": "list_users"
}

5. 退出
{
    "type": "quit"
}

服务器响应：
{
    "type": "broadcast",
    "from": "Alice",
    "content": "Hello everyone!",
    "timestamp": "2024-01-15 10:30:00"
}
```

### 5.3 工具演示

```bash
# 启动聊天室服务器
python day18_chat_server.py --host 0.0.0.0 --port 8888

# 启动聊天室客户端
python day18_chat_client.py --host 127.0.0.1 --port 8888 --nickname Alice
```

---

## 6. 今日练习

### 练习1：阻塞vs非阻塞

```python
# 编写程序比较阻塞和非阻塞socket的行为
# 1. 创建阻塞socket，测量recv()的等待时间
# 2. 创建非阻塞socket，测量recv()的返回时间
# 3. 对比差异
```

### 练习2：实现select服务器

```python
# 使用select实现简单的回显服务器
# 要求：
# 1. 支持多客户端同时连接
# 2. 每个客户端发送的数据原样返回
# 3. 显示连接/断开信息
```

### 练习3：性能测试

```python
# 编写压力测试工具
# 1. 创建100个并发连接
# 2. 每个连接发送100条消息
# 3. 测量总耗时和QPS
# 4. 对比不同IO模型的性能
```

### 练习4：聊天室扩展

```python
# 为聊天室添加功能：
# 1. 用户认证（登录）
# 2. 消息历史记录
# 3. 文件传输
# 4. 表情支持
```

---

## 7. 常见问题

**Q1：阻塞和非阻塞的本质区别是什么？**

A：
- 阻塞：系统调用会等待，直到条件满足
- 非阻塞：系统调用立即返回，如果未就绪返回错误

**Q2：select/poll/epoll应该选哪个？**

A：
- Linux生产环境：epoll（性能最好）
- 跨平台开发：select或poll
- 少量连接：select够用
- 大量连接：必须epoll

**Q3：为什么epoll比select快？**

A：
1. select需要遍历所有fd（O(n)），epoll只返回就绪的（O(1)）
2. select每次调用都要拷贝fd_set，epoll使用mmap
3. epoll支持边缘触发，减少系统调用

**Q4：什么时候用多线程，什么时候用IO多路复用？**

A：
- IO密集型：IO多路复用
- CPU密集型：多线程/多进程
- 混合型：IO多路复用 + 线程池

**Q5：异步IO和IO多路复用有什么区别？**

A：
- IO多路复用：同步非阻塞，select/epoll通知就绪
- 异步IO：真正的异步，系统完成IO后通知
- Linux的AIO支持不完善，实际多用IO多路复用

---

## 8. 总结

今天我们学习了：

### 核心知识点

1. **阻塞vs非阻塞**：
   - 阻塞：等待直到就绪
   - 非阻塞：立即返回，未就绪返回错误

2. **IO多路复用**：
   - select：跨平台，有限制
   - poll：改进的select
   - epoll：Linux高性能（推荐）

3. **Socket选项**：
   - SO_REUSEADDR：地址重用
   - TCP_NODELAY：禁用Nagle
   - SO_KEEPALIVE：保活机制

4. **服务器架构**：
   - 单线程阻塞：简单但慢
   - 多线程：简单但有限
   - IO多路复用：高性能推荐
   - Reactor模式：事件驱动

5. **实战技巧**：
   - 选择合适的IO模型
   - 配置Socket选项
   - 处理并发连接

### 重点回顾

```
高性能网络编程 = IO多路复用 + 非阻塞Socket

Linux推荐：
epoll + 非阻塞 + 边缘触发

架构选择：
IO密集 → epoll单线程
CPU密集 → 多进程
混合 → epoll + 线程池
```

### 明天预告

**第19天：HTTP协议基础**

内容预览：
- HTTP协议概述
- HTTP请求和响应格式
- HTTP方法详解
- HTTP状态码
- Cookie和Session
- 实现简单的HTTP服务器

---

**继续加油！** 🚀

今天我们学习了高级Socket编程技术，理解了如何构建高性能服务器。这些知识是现代网络编程的核心，也是很多Web框架的底层实现原理。明天我们将学习应用层的HTTP协议！
