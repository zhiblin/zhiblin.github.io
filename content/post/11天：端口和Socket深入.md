---
title: 11天：端口和Socket深入
date: 2025-11-10 15:21:38
categories: Network 
tags: Network, Book, Python
---
# 第11天：端口和Socket深入

> "端口是应用程序的门牌号，Socket是网络编程的大门。"

## 📚 今日目标

- 理解端口的概念和作用
- 掌握端口的分类和常用端口号
- 深入理解Socket编程
- 区分TCP Socket和UDP Socket
- 编写端口扫描器工具

## 🎯 什么是端口？

### 生活中的例子

```
端口 = 公司大楼的房间号

大楼地址（IP地址）：
北京市朝阳区某某路100号

房间号（端口）：
101室 - 前台
201室 - 财务部
301室 - 技术部

访问方式：
找技术部 = 北京市朝阳区某某路100号301室
访问网站 = 220.181.38.148:80
```

**端口**（Port）是传输层的概念，用于标识主机上的不同应用程序或服务。

### 端口的作用

```
问题：一台电脑同时运行多个网络程序
- 浏览器访问网页
- QQ聊天
- 下载文件
- 播放在线视频

如何区分这些数据是给哪个程序的？
答案：通过端口号！

数据包到达：
IP地址 → 找到正确的电脑
端口号 → 找到正确的程序
```

## 📊 端口的基本特征

### 1. 端口号范围

```
端口号：16位无符号整数
范围：0 - 65535（2^16 - 1）

分为三类：
```

### 2. 端口分类

| 类别 | 范围 | 说明 | 例子 |
|------|------|------|------|
| 知名端口 | 0-1023 | 系统保留，需要管理员权限 | 80(HTTP), 443(HTTPS), 22(SSH) |
| 注册端口 | 1024-49151 | 软件厂商注册使用 | 3306(MySQL), 8080(HTTP代理) |
| 动态/私有端口 | 49152-65535 | 客户端临时使用 | 随机分配 |

### 3. 端口的状态

```
端口的三种状态：

1. LISTENING（监听）
   服务器程序正在监听，等待连接
   例如：Web服务器监听80端口

2. ESTABLISHED（已建立）
   连接已建立，正在通信
   例如：浏览器和服务器之间的连接

3. CLOSED（关闭）
   端口未被使用
```

## 🔢 常用端口号

### Web服务

| 端口 | 协议 | 说明 |
|------|------|------|
| 80 | HTTP | 网页浏览（明文） |
| 443 | HTTPS | 安全网页浏览（加密） |
| 8080 | HTTP | HTTP代理、测试服务器 |
| 8443 | HTTPS | HTTPS备用端口 |

### 文件传输

| 端口 | 协议 | 说明 |
|------|------|------|
| 20 | FTP-DATA | FTP数据传输 |
| 21 | FTP | FTP控制连接 |
| 22 | SFTP/SSH | 安全文件传输 |
| 69 | TFTP | 简单文件传输 |

### 邮件服务

| 端口 | 协议 | 说明 |
|------|------|------|
| 25 | SMTP | 发送邮件 |
| 110 | POP3 | 接收邮件 |
| 143 | IMAP | 邮件访问 |
| 465 | SMTPS | SMTP加密版 |
| 993 | IMAPS | IMAP加密版 |
| 995 | POP3S | POP3加密版 |

### 数据库

| 端口 | 数据库 | 说明 |
|------|--------|------|
| 3306 | MySQL | MySQL数据库 |
| 5432 | PostgreSQL | PostgreSQL数据库 |
| 1521 | Oracle | Oracle数据库 |
| 27017 | MongoDB | MongoDB数据库 |
| 6379 | Redis | Redis缓存数据库 |

### 远程访问

| 端口 | 协议 | 说明 |
|------|------|------|
| 22 | SSH | 安全外壳协议 |
| 23 | Telnet | 远程终端（不安全） |
| 3389 | RDP | Windows远程桌面 |
| 5900 | VNC | 虚拟网络计算 |

### 其他常用端口

| 端口 | 协议 | 说明 |
|------|------|------|
| 53 | DNS | 域名解析 |
| 67/68 | DHCP | 动态主机配置 |
| 161 | SNMP | 简单网络管理 |
| 445 | SMB | Windows文件共享 |
| 3000 | Node.js | 常用开发端口 |
| 5000 | Flask | Python Flask默认 |
| 8000 | Django | Python Django常用 |

## 🔧 Socket编程

### 什么是Socket？

**Socket**（套接字）是网络编程的基础接口，用于实现网络通信。

```
Socket = 电话机

打电话的过程：
1. 拿起电话（创建Socket）
2. 拨号（连接）
3. 说话（发送数据）
4. 听对方说（接收数据）
5. 挂断（关闭Socket）

网络通信的过程：
1. 创建Socket对象
2. 连接到服务器（TCP）或准备好（UDP）
3. 发送数据
4. 接收数据
5. 关闭Socket
```

### Socket类型

#### 1. TCP Socket（流式套接字）

```python
# 特点：
- 面向连接
- 可靠传输
- 数据按序到达
- 适用于需要可靠传输的场景

# 使用场景：
- 网页浏览（HTTP/HTTPS）
- 文件传输（FTP）
- 邮件发送（SMTP）
- 远程登录（SSH）

# Python创建：
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
```

#### 2. UDP Socket（数据报套接字）

```python
# 特点：
- 无连接
- 不可靠（可能丢包）
- 速度快
- 适用于实时性要求高的场景

# 使用场景：
- 视频直播
- 语音通话
- 在线游戏
- DNS查询

# Python创建：
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
```

### TCP Socket编程流程

#### 服务器端

```
1. 创建Socket
2. 绑定地址和端口（bind）
3. 监听连接（listen）
4. 接受连接（accept）
5. 收发数据（recv/send）
6. 关闭连接（close）

┌─────────────┐
│ 创建Socket  │
└──────┬──────┘
       ↓
┌─────────────┐
│ 绑定端口    │ bind(('0.0.0.0', 8888))
└──────┬──────┘
       ↓
┌─────────────┐
│ 开始监听    │ listen(5)
└──────┬──────┘
       ↓
┌─────────────┐
│ 等待连接    │ accept() ← 阻塞等待
└──────┬──────┘
       ↓
┌─────────────┐
│ 收发数据    │ recv() / send()
└──────┬──────┘
       ↓
┌─────────────┐
│ 关闭连接    │ close()
└─────────────┘
```

#### 客户端

```
1. 创建Socket
2. 连接服务器（connect）
3. 收发数据（recv/send）
4. 关闭连接（close）

┌─────────────┐
│ 创建Socket  │
└──────┬──────┘
       ↓
┌─────────────┐
│ 连接服务器  │ connect(('server_ip', 8888))
└──────┬──────┘
       ↓
┌─────────────┐
│ 收发数据    │ recv() / send()
└──────┬──────┘
       ↓
┌─────────────┐
│ 关闭连接    │ close()
└─────────────┘
```

## 🔧 实战项目：端口扫描器

让我们编写一个功能强大的端口扫描器！

### 代码实现

```python
# day11_port_scanner.py
# 功能：端口扫描器

import socket
import threading
from concurrent.futures import ThreadPoolExecutor, as_completed
import time

class PortScanner:
    """端口扫描器"""

    def __init__(self):
        # 常用端口和服务的映射
        self.common_ports = {
            20: 'FTP-DATA',
            21: 'FTP',
            22: 'SSH',
            23: 'Telnet',
            25: 'SMTP',
            53: 'DNS',
            80: 'HTTP',
            110: 'POP3',
            143: 'IMAP',
            443: 'HTTPS',
            445: 'SMB',
            3306: 'MySQL',
            3389: 'RDP',
            5432: 'PostgreSQL',
            6379: 'Redis',
            8080: 'HTTP-Proxy',
            27017: 'MongoDB',
        }

    def scan_port(self, host, port, timeout=1):
        """
        扫描单个端口

        参数：
            host: 目标主机
            port: 端口号
            timeout: 超时时间（秒）

        返回：
            (port, is_open, service_name) 元组
        """
        try:
            # 创建TCP Socket
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(timeout)

            # 尝试连接
            result = sock.connect_ex((host, port))

            # 关闭连接
            sock.close()

            # result = 0 表示连接成功（端口开放）
            if result == 0:
                service = self.common_ports.get(port, 'Unknown')
                return (port, True, service)
            else:
                return (port, False, None)

        except socket.timeout:
            return (port, False, None)
        except socket.error:
            return (port, False, None)
        except Exception as e:
            return (port, False, None)

    def scan_port_with_banner(self, host, port, timeout=1):
        """
        扫描端口并尝试获取服务banner

        参数：
            host: 目标主机
            port: 端口号
            timeout: 超时时间

        返回：
            (port, is_open, service_name, banner)
        """
        port_result = self.scan_port(host, port, timeout)
        port_num, is_open, service = port_result

        if not is_open:
            return (port_num, False, None, None)

        # 尝试获取banner
        banner = None
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(2)
            sock.connect((host, port))

            # 发送简单的请求
            if port == 80 or port == 8080:
                sock.send(b'GET / HTTP/1.0\r\n\r\n')

            # 尝试接收响应
            banner = sock.recv(1024).decode('utf-8', errors='ignore').strip()
            sock.close()

            # 只保留第一行
            if banner:
                banner = banner.split('\n')[0][:50]

        except:
            pass

        return (port_num, is_open, service, banner)

    def scan_range(self, host, start_port, end_port, timeout=1, show_closed=False):
        """
        扫描端口范围

        参数：
            host: 目标主机
            start_port: 起始端口
            end_port: 结束端口
            timeout: 超时时间
            show_closed: 是否显示关闭的端口

        返回：
            开放端口列表
        """
        print(f"🔍 正在扫描 {host}:{start_port}-{end_port}")
        print(f"⏳ 请稍候...\n")

        open_ports = []
        total_ports = end_port - start_port + 1
        scanned = 0

        # 使用线程池并发扫描
        with ThreadPoolExecutor(max_workers=100) as executor:
            # 提交所有扫描任务
            futures = {
                executor.submit(self.scan_port, host, port, timeout): port
                for port in range(start_port, end_port + 1)
            }

            # 处理完成的任务
            for future in as_completed(futures):
                scanned += 1
                port_num, is_open, service = future.result()

                if is_open:
                    open_ports.append((port_num, service))
                    print(f"✅ 端口 {port_num:<6} 开放 - {service}")
                elif show_closed:
                    print(f"❌ 端口 {port_num:<6} 关闭")

                # 显示进度
                if scanned % 50 == 0:
                    progress = (scanned / total_ports) * 100
                    print(f"   进度: {progress:.1f}% ({scanned}/{total_ports})", end='\r')

        return open_ports

    def scan_common_ports(self, host, timeout=1):
        """
        扫描常用端口

        参数：
            host: 目标主机
            timeout: 超时时间

        返回：
            开放端口列表
        """
        print(f"🔍 正在扫描 {host} 的常用端口")
        print(f"📋 共 {len(self.common_ports)} 个常用端口\n")

        open_ports = []

        with ThreadPoolExecutor(max_workers=50) as executor:
            futures = {
                executor.submit(self.scan_port_with_banner, host, port, timeout): port
                for port in self.common_ports.keys()
            }

            for future in as_completed(futures):
                port_num, is_open, service, banner = future.result()

                if is_open:
                    open_ports.append((port_num, service, banner))
                    if banner:
                        print(f"✅ 端口 {port_num:<6} 开放 - {service:<15} | {banner}")
                    else:
                        print(f"✅ 端口 {port_num:<6} 开放 - {service}")

        return open_ports

    def display_results(self, host, open_ports):
        """
        显示扫描结果

        参数：
            host: 主机地址
            open_ports: 开放端口列表
        """
        print("\n" + "=" * 70)
        print(f"扫描结果 - {host}".center(70))
        print("=" * 70)

        if open_ports:
            print(f"\n发现 {len(open_ports)} 个开放端口:\n")
            print(f"{'端口':<10} {'服务':<20} {'备注'}")
            print("-" * 70)

            for port_info in sorted(open_ports):
                if len(port_info) == 2:
                    port, service = port_info
                    print(f"{port:<10} {service:<20}")
                elif len(port_info) == 3:
                    port, service, banner = port_info
                    banner_str = banner[:40] if banner else ""
                    print(f"{port:<10} {service:<20} {banner_str}")
        else:
            print("\n❌ 未发现开放端口")

        print("\n" + "=" * 70)

def main():
    """
    主函数
    """
    scanner = PortScanner()

    print("=" * 70)
    print("端口扫描器".center(70))
    print("=" * 70)
    print()

    print("请选择扫描模式:")
    print("1. 扫描常用端口（推荐）")
    print("2. 扫描端口范围")
    print("3. 扫描单个端口")
    print("0. 退出")

    choice = input("\n请输入选项: ").strip()

    if choice == '0':
        print("\n再见！")
        return

    # 获取目标主机
    host = input("\n请输入目标主机（IP或域名）: ").strip()
    if not host:
        print("❌ 主机不能为空")
        return

    # 验证主机是否可达
    print(f"\n🔍 正在检查主机 {host} 是否可达...")
    try:
        socket.gethostbyname(host)
        print("✅ 主机可达")
    except socket.gaierror:
        print("❌ 无法解析主机名")
        return

    print()
    start_time = time.time()

    if choice == '1':
        # 扫描常用端口
        open_ports = scanner.scan_common_ports(host)
        scanner.display_results(host, open_ports)

    elif choice == '2':
        # 扫描端口范围
        start_port = int(input("请输入起始端口: ").strip())
        end_port = int(input("请输入结束端口: ").strip())

        if start_port < 1 or end_port > 65535 or start_port > end_port:
            print("❌ 端口范围无效")
            return

        open_ports = scanner.scan_range(host, start_port, end_port)
        scanner.display_results(host, open_ports)

    elif choice == '3':
        # 扫描单个端口
        port = int(input("请输入端口号: ").strip())

        if port < 1 or port > 65535:
            print("❌ 端口号无效")
            return

        port_num, is_open, service, banner = scanner.scan_port_with_banner(host, port)

        if is_open:
            print(f"\n✅ 端口 {port} 开放")
            print(f"   服务: {service}")
            if banner:
                print(f"   Banner: {banner}")
        else:
            print(f"\n❌ 端口 {port} 关闭")

    # 显示耗时
    elapsed_time = time.time() - start_time
    print(f"\n⏱️  扫描耗时: {elapsed_time:.2f} 秒")

    # 安全提示
    print("\n" + "=" * 70)
    print("⚠️  安全提示:")
    print("   - 仅扫描您有权限扫描的主机")
    print("   - 未经授权的端口扫描可能违法")
    print("   - 请遵守网络安全法律法规")
    print("=" * 70)

if __name__ == "__main__":
    main()
```

### 运行程序

```bash
python code/day11/day11_port_scanner.py
```

## 🔍 代码详解

### 1. Socket连接测试

```python
# connect_ex() 方法
result = sock.connect_ex((host, port))

# 返回值：
# 0 = 连接成功（端口开放）
# 其他 = 连接失败（端口关闭或被过滤）
```

### 2. 并发扫描提速

```python
# 使用线程池同时扫描多个端口
with ThreadPoolExecutor(max_workers=100) as executor:
    # 100个线程并发工作
    futures = {executor.submit(scan_port, host, port): port
               for port in range(start, end)}
```

### 3. Banner抓取

```python
# 连接成功后，尝试接收服务器的响应
sock.connect((host, port))
sock.send(b'GET / HTTP/1.0\r\n\r\n')  # 发送HTTP请求
banner = sock.recv(1024)  # 接收响应

# Banner可以帮助识别服务类型和版本
```

## 🎓 知识小结

今天学习了：

1. ✅ 端口是传输层的概念，用于标识应用程序
2. ✅ 端口号范围：0-65535
3. ✅ Socket是网络编程的基础接口
4. ✅ TCP Socket面向连接，UDP Socket无连接
5. ✅ 端口扫描是网络安全审计的重要手段

### 重要概念

| 概念 | 说明 |
|------|------|
| 端口 | 16位数字，标识应用程序 |
| Socket | 网络编程接口 |
| TCP Socket | 面向连接，可靠传输 |
| UDP Socket | 无连接，快速但不可靠 |
| 端口扫描 | 检测端口开放状态 |

### 常用命令

```bash
# 查看本机监听的端口
netstat -an | grep LISTEN    # Linux/Mac
netstat -an | findstr LISTEN # Windows

# 查看特定端口
lsof -i :8080               # Mac/Linux
netstat -ano | findstr :8080 # Windows

# 测试端口连通性
telnet host port
nc -zv host port            # Linux/Mac
```

## 💪 练习题

### 练习1：查看本机端口
1. 查看本机正在监听的所有端口
2. 找出80和443端口的状态
3. 查看哪些程序占用了端口

### 练习2：理解Socket
1. 画出TCP Socket通信的完整流程
2. 解释TCP三次握手在Socket中的体现
3. 说明服务器端和客户端的区别

### 练习3：端口扫描
1. 扫描本机的常用端口
2. 扫描局域网网关的端口
3. 分析扫描结果，识别运行的服务

### 练习4：代码扩展
扩展端口扫描器：
1. 添加扫描结果保存功能
2. 实现服务版本检测
3. 添加操作系统指纹识别
4. 支持多主机批量扫描

### 练习5：安全思考
1. 什么是端口扫描？合法吗？
2. 如何防御端口扫描？
3. 防火墙如何过滤端口？

---

💡 **今日金句**：端口是网络世界的大门，Socket是打开这扇门的钥匙！

👉 [上一天：MAC地址和ARP](day10.md) | [返回目录](../../README.md) | [下一天：DNS协议详解](day12.md)
