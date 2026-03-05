---
title: 4天：TCP/IP四层模型
date: 2025-11-10 14:39:37
categories: Network 
tags: [Network, Book, Python]
---
# 第4天：TCP/IP四层模型

> "理论是OSI，实战是TCP/IP。"

## 📚 今日目标

- 理解TCP/IP四层模型
- 掌握TCP/IP与OSI的区别和联系
- 学会分析真实的网络数据包
- 编写程序捕获和解析数据包

## 🎯 为什么需要TCP/IP模型？

### OSI vs TCP/IP

**OSI七层模型**：
- 📖 理论模型，由ISO制定
- 🎓 教学用途，帮助理解网络概念
- 📚 过于复杂，实际应用较少

**TCP/IP四层模型**：
- 💻 实际使用的模型
- 🌐 互联网的基础
- 🚀 简单实用，广泛应用

### 类比理解

```
OSI模型 = 理想的建筑设计图纸（7层楼）
TCP/IP模型 = 实际建造的房子（4层楼，但功能齐全）
```

## 📊 TCP/IP四层模型

### 层次对应关系

```
┌──────────────────┐
│   OSI七层模型    │            TCP/IP四层模型
├──────────────────┤
│  7. 应用层       │ \
│  6. 表示层       │  \       ┌──────────────────┐
│  5. 会话层       │   } ───> │  4. 应用层       │
└──────────────────┘  /       │  (Application)   │
┌──────────────────┐ /        └──────────────────┘
│  4. 传输层       │ ───────> ┌──────────────────┐
└──────────────────┘          │  3. 传输层       │
┌──────────────────┐          │  (Transport)     │
│  3. 网络层       │ ───────> └──────────────────┘
└──────────────────┘          ┌──────────────────┐
┌──────────────────┐          │  2. 网际层       │
│  2. 数据链路层   │ \        │  (Internet)      │
│  1. 物理层       │  } ───>  └──────────────────┘
└──────────────────┘ /        ┌──────────────────┐
                               │  1. 网络接口层   │
                               │  (Network Access)│
                               └──────────────────┘
```

### TCP/IP各层详解

#### 第4层：应用层（Application Layer）

**对应OSI**：应用层 + 表示层 + 会话层

**主要协议**：
- 🌐 HTTP/HTTPS：网页浏览
- 📧 SMTP/POP3/IMAP：邮件
- 📁 FTP：文件传输
- 🔍 DNS：域名解析
- 🔐 SSH：远程登录
- 💬 MQTT：物联网通信

**作用**：为用户提供各种网络服务

**例子**：
```
浏览器访问 www.baidu.com
↓
应用层：生成HTTP请求
GET / HTTP/1.1
Host: www.baidu.com
```

---

#### 第3层：传输层（Transport Layer）

**对应OSI**：传输层

**主要协议**：
- 🚚 TCP：可靠传输，面向连接
- 📦 UDP：快速传输，无连接

**作用**：提供端到端的数据传输

**关键概念**：
- **端口号**：标识应用程序
  - HTTP: 80
  - HTTPS: 443
  - SSH: 22
  - MySQL: 3306

**TCP vs UDP**：
```
TCP = 打电话（需要接通，可靠）
UDP = 发短信（直接发，不保证送达）
```

---

#### 第2层：网际层（Internet Layer）

**对应OSI**：网络层

**主要协议**：
- 🌐 IP：网际协议（IPv4/IPv6）
- 📡 ICMP：控制消息协议
- 🗺️ ARP：地址解析协议
- 🔀 IGMP：组管理协议

**作用**：路由和寻址，在网络间传输数据

**关键概念**：
- **IP地址**：设备的网络地址
  - 192.168.1.100（私网）
  - 8.8.8.8（公网）

**例子**：
```
从北京的电脑访问上海的服务器
↓
网际层：选择最佳路径
北京 → 路由器1 → 路由器2 → ... → 上海
```

---

#### 第1层：网络接口层（Network Access Layer）

**对应OSI**：数据链路层 + 物理层

**主要协议**：
- 🔗 Ethernet：以太网
- 📶 WiFi：无线局域网
- 🔌 PPP：点对点协议

**作用**：在物理网络上传输数据帧

**关键概念**：
- **MAC地址**：网卡的物理地址
  - AA:BB:CC:DD:EE:FF

**例子**：
```
同一个局域网内的两台电脑通信
↓
网络接口层：通过MAC地址直接传输
电脑A的网卡 → 交换机 → 电脑B的网卡
```

## 📦 TCP/IP数据封装

### 发送数据的封装过程

```
应用层：
[HTTP请求数据]
    ↓
传输层：
[TCP头部 | HTTP请求数据]  ← Segment（段）
    ↓
网际层：
[IP头部 | TCP头部 | HTTP请求数据]  ← Packet（包）
    ↓
网络接口层：
[以太网帧头 | IP头部 | TCP头部 | HTTP请求数据 | 帧尾]  ← Frame（帧）
    ↓
物理层：
01010101...  ← Bits（比特流）
```

### 各层添加的信息

| 层次 | 添加的信息 | 目的 |
|------|------------|------|
| 应用层 | 应用数据 | 用户数据 |
| 传输层 | 源端口、目标端口 | 标识应用程序 |
| 网际层 | 源IP、目标IP | 标识主机位置 |
| 网络接口层 | 源MAC、目标MAC | 标识网卡 |

## 🔧 实战项目：数据包分析器

让我们编写一个程序，捕获和分析真实的网络数据包！

### 代码实现

```python
# day04_packet_analyzer.py
# 功能：捕获并分析网络数据包（TCP/IP各层信息）

import socket
import struct
import textwrap

def ethernet_frame(data):
    """
    解析以太网帧（网络接口层）

    以太网帧格式：
    [目标MAC(6字节) | 源MAC(6字节) | 类型(2字节) | 数据 | CRC(4字节)]
    """
    # 提取MAC地址和类型
    dest_mac, src_mac, eth_type = struct.unpack('! 6s 6s H', data[:14])

    # 格式化MAC地址
    dest_mac = ':'.join('{:02x}'.format(b) for b in dest_mac)
    src_mac = ':'.join('{:02x}'.format(b) for b in src_mac)

    # 以太网类型
    # 0x0800 = IPv4, 0x86DD = IPv6, 0x0806 = ARP
    eth_type = socket.htons(eth_type)

    # 返回MAC地址、类型和剩余数据
    return dest_mac, src_mac, eth_type, data[14:]

def ipv4_packet(data):
    """
    解析IPv4数据包（网际层）

    IPv4头部格式（20字节）：
    [版本+头长(1) | 服务类型(1) | 总长度(2) | 标识(2) | 标志+偏移(2) |
     TTL(1) | 协议(1) | 校验和(2) | 源IP(4) | 目标IP(4)]
    """
    # 解析IPv4头部的前20字节
    version_header_length = data[0]
    version = version_header_length >> 4  # 高4位是版本
    header_length = (version_header_length & 15) * 4  # 低4位是头长（单位：4字节）

    # 提取TTL、协议、源IP、目标IP
    ttl, proto, src, target = struct.unpack('! 8x B B 2x 4s 4s', data[:20])

    # 格式化IP地址
    src_ip = '.'.join(map(str, src))
    target_ip = '.'.join(map(str, target))

    # 协议号对应的协议名
    protocols = {1: 'ICMP', 6: 'TCP', 17: 'UDP'}
    proto_name = protocols.get(proto, f'Other({proto})')

    return version, header_length, ttl, proto_name, src_ip, target_ip, data[header_length:]

def tcp_segment(data):
    """
    解析TCP段（传输层）

    TCP头部格式（至少20字节）：
    [源端口(2) | 目标端口(2) | 序号(4) | 确认号(4) |
     头长+标志(2) | 窗口(2) | 校验和(2) | 紧急指针(2)]
    """
    src_port, dest_port, sequence, acknowledgment, offset_reserved_flags = struct.unpack(
        '! H H L L H', data[:14]
    )

    # 提取头部长度（高4位）
    offset = (offset_reserved_flags >> 12) * 4

    # 提取TCP标志位
    flag_urg = (offset_reserved_flags & 32) >> 5
    flag_ack = (offset_reserved_flags & 16) >> 4
    flag_psh = (offset_reserved_flags & 8) >> 3
    flag_rst = (offset_reserved_flags & 4) >> 2
    flag_syn = (offset_reserved_flags & 2) >> 1
    flag_fin = offset_reserved_flags & 1

    return src_port, dest_port, sequence, acknowledgment, flag_urg, flag_ack, \
           flag_psh, flag_rst, flag_syn, flag_fin, data[offset:]

def udp_segment(data):
    """
    解析UDP段（传输层）

    UDP头部格式（8字节）：
    [源端口(2) | 目标端口(2) | 长度(2) | 校验和(2)]
    """
    src_port, dest_port, length = struct.unpack('! H H 2x H', data[:8])
    return src_port, dest_port, length, data[8:]

def format_multi_line(prefix, string, size=80):
    """格式化多行输出"""
    size -= len(prefix)
    if isinstance(string, bytes):
        string = ''.join(r'\x{:02x}'.format(byte) for byte in string)
        if size % 2:
            size -= 1

    return '\n'.join([prefix + line for line in textwrap.wrap(string, size)])

def main():
    """
    主函数：捕获并分析数据包
    """
    print("=" * 80)
    print("TCP/IP数据包分析器".center(80))
    print("=" * 80)
    print("\n📡 正在监听网络接口...")
    print("💡 提示：访问一个网站或ping一个地址来生成流量")
    print("⚠️  按Ctrl+C停止捕获\n")
    print("=" * 80)

    # 创建原始套接字
    # 注意：在某些系统上可能需要root权限
    try:
        # Windows
        conn = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_IP)
        conn.bind(('', 0))
        conn.setsockopt(socket.IPPROTO_IP, socket.IP_HDRINCL, 1)
        conn.ioctl(socket.SIO_RCVALL, socket.RCVALL_ON)
    except AttributeError:
        # Linux/Mac
        conn = socket.socket(socket.AF_PACKET, socket.SOCK_RAW, socket.ntohs(3))

    packet_count = 0

    try:
        while True:
            # 接收数据
            raw_data, addr = conn.recvfrom(65535)
            packet_count += 1

            print(f"\n{'▼' * 40}")
            print(f"📦 数据包 #{packet_count}")
            print(f"{'▼' * 40}\n")

            # 1. 解析以太网帧（网络接口层）
            try:
                dest_mac, src_mac, eth_type, data = ethernet_frame(raw_data)
                print(f"🔗 网络接口层（以太网帧）")
                print(f"   源MAC地址: {src_mac}")
                print(f"   目标MAC地址: {dest_mac}")
                print(f"   协议类型: 0x{eth_type:04x}")
            except:
                # 某些系统可能不包含以太网头
                eth_type = 0x0800
                data = raw_data

            # 2. 解析IP数据包（网际层）
            if eth_type == 0x0800:  # IPv4
                version, header_length, ttl, proto, src_ip, target_ip, data = ipv4_packet(data)
                print(f"\n🌐 网际层（IP数据包）")
                print(f"   IP版本: IPv{version}")
                print(f"   头部长度: {header_length}字节")
                print(f"   TTL: {ttl}")
                print(f"   协议: {proto}")
                print(f"   源IP地址: {src_ip}")
                print(f"   目标IP地址: {target_ip}")

                # 3. 解析传输层
                if proto == 'TCP':
                    src_port, dest_port, sequence, acknowledgment, flag_urg, flag_ack, \
                    flag_psh, flag_rst, flag_syn, flag_fin, data = tcp_segment(data)

                    print(f"\n🚚 传输层（TCP段）")
                    print(f"   源端口: {src_port}")
                    print(f"   目标端口: {dest_port}")
                    print(f"   序列号: {sequence}")
                    print(f"   确认号: {acknowledgment}")
                    print(f"   标志位: ", end="")

                    flags = []
                    if flag_syn: flags.append("SYN")
                    if flag_ack: flags.append("ACK")
                    if flag_fin: flags.append("FIN")
                    if flag_rst: flags.append("RST")
                    if flag_psh: flags.append("PSH")
                    if flag_urg: flags.append("URG")

                    print(", ".join(flags) if flags else "无")

                    # 4. 应用层数据
                    if len(data) > 0:
                        print(f"\n📱 应用层数据")
                        print(f"   数据长度: {len(data)}字节")

                        # 尝试解析HTTP
                        try:
                            data_str = data.decode('utf-8', errors='ignore')
                            if data_str.startswith('HTTP') or data_str.startswith('GET') or \
                               data_str.startswith('POST'):
                                print(f"   协议: HTTP")
                                lines = data_str.split('\\r\\n')[:5]  # 只显示前5行
                                for line in lines:
                                    if line:
                                        print(f"   {line}")
                        except:
                            print(f"   数据预览: {data[:50]}")

                elif proto == 'UDP':
                    src_port, dest_port, length, data = udp_segment(data)

                    print(f"\n📦 传输层（UDP段）")
                    print(f"   源端口: {src_port}")
                    print(f"   目标端口: {dest_port}")
                    print(f"   长度: {length}字节")

                    # DNS查询/响应
                    if src_port == 53 or dest_port == 53:
                        print(f"   应用协议: DNS")

                elif proto == 'ICMP':
                    print(f"\n📡 传输层（ICMP）")
                    print(f"   这是一个ping或traceroute数据包")

            print("\n" + "=" * 80)

            # 只捕获前10个包作为演示
            if packet_count >= 10:
                print(f"\n✅ 已捕获{packet_count}个数据包，停止监听")
                break

    except KeyboardInterrupt:
        print(f"\n\n✋ 用户停止捕获")
        print(f"📊 总共捕获了 {packet_count} 个数据包")

    finally:
        # Windows需要关闭混杂模式
        try:
            conn.ioctl(socket.SIO_RCVALL, socket.RCVALL_OFF)
        except:
            pass
        conn.close()

    print("=" * 80)

if __name__ == "__main__":
    import sys
    if sys.platform == 'win32' or sys.platform == 'linux' or sys.platform == 'darwin':
        main()
    else:
        print("⚠️  此程序可能需要管理员/root权限")
        print("💡 Linux/Mac: sudo python day04_packet_analyzer.py")
        print("💡 Windows: 以管理员身份运行")
```

### 运行说明

**Linux/Mac**：
```bash
sudo python code/day04/day04_packet_analyzer.py
```

**Windows**：
以管理员身份运行命令提示符，然后：
```bash
python code\day04\day04_packet_analyzer.py
```

### 预期输出

程序会实时显示捕获到的数据包的各层信息。

## 🎓 知识小结

### 今天学到了什么？

1. ✅ TCP/IP四层模型是实际使用的网络模型
2. ✅ TCP/IP比OSI更简洁实用
3. ✅ 每一层都有特定的协议和作用
4. ✅ 数据包包含多层的头部信息

### TCP/IP四层对应表

| TCP/IP层 | 主要协议 | 关键信息 | 设备 |
|----------|----------|----------|------|
| 应用层 | HTTP、DNS、FTP | 应用数据 | - |
| 传输层 | TCP、UDP | 端口号 | - |
| 网际层 | IP、ICMP | IP地址 | 路由器 |
| 网络接口层 | Ethernet、WiFi | MAC地址 | 交换机、网卡 |

### 重要协议

**应用层**：
- HTTP（80）：网页
- HTTPS（443）：加密网页
- DNS（53）：域名解析
- FTP（21）：文件传输
- SSH（22）：远程登录
- SMTP（25）：发送邮件

**传输层**：
- TCP：可靠传输
- UDP：快速传输

**网际层**：
- IP：寻址和路由
- ICMP：错误报告和诊断

**网络接口层**：
- Ethernet：有线网络
- WiFi：无线网络

## 💪 练习题

### 练习1：协议识别
以下端口号对应什么协议？
1. 80
2. 443
3. 22
4. 3306
5. 27017

### 练习2：层次判断
以下信息分别属于哪一层？
1. 192.168.1.1
2. AA:BB:CC:DD:EE:FF
3. 端口8080
4. HTTP/1.1
5. TCP序列号12345

### 练习3：绘制数据包结构
画出一个完整的HTTP请求数据包的结构，标注每一层的头部。

### 练习4：代码扩展
扩展数据包分析器：
1. 添加数据包过滤功能（只显示特定协议）
2. 统计各协议的数据包数量
3. 将捕获的数据包保存到文件

### 练习5：思考题
1. 为什么TCP/IP模型只有4层而OSI有7层？
2. 路由器工作在哪一层？交换机呢？
3. 防火墙可能工作在哪几层？

## 📚 扩展阅读

### TCP/IP协议族

TCP/IP不仅仅是TCP和IP两个协议，而是一整个协议族：

```
应用层：
┌────────────────────────────────────┐
│ HTTP  FTP   DNS   SMTP  SSH  等... │
└────────────────────────────────────┘

传输层：
┌──────────┬──────────┐
│   TCP    │   UDP    │
└──────────┴──────────┘

网际层：
┌────────────────────────────┐
│   IP   ICMP   ARP   IGMP   │
└────────────────────────────┘

网络接口层：
┌────────────────────────────┐
│  Ethernet  WiFi  PPP  等... │
└────────────────────────────┘
```

### 常见端口号记忆

**HTTP家族**：
- 80：HTTP
- 443：HTTPS
- 8080：HTTP代理

**文件传输**：
- 20/21：FTP
- 22：SFTP（SSH）
- 69：TFTP

**邮件**：
- 25：SMTP（发送）
- 110：POP3（接收）
- 143：IMAP（接收）

**数据库**：
- 3306：MySQL
- 5432：PostgreSQL
- 27017：MongoDB
- 6379：Redis

**远程**：
- 22：SSH
- 23：Telnet
- 3389：RDP（Windows远程桌面）

### 实际应用场景

1. **浏览网页**：
   ```
   应用层：HTTP请求 www.baidu.com
   传输层：TCP 端口80
   网际层：IP地址 220.181.38.148
   网络接口层：通过以太网发送
   ```

2. **发送邮件**：
   ```
   应用层：SMTP协议
   传输层：TCP 端口25
   网际层：IP地址（邮件服务器）
   网络接口层：WiFi
   ```

3. **ping命令**：
   ```
   应用层：无（ICMP是网际层协议）
   传输层：无
   网际层：ICMP echo request
   网络接口层：以太网
   ```

## 🎯 明天预告

**第5天：网络命令入门 - ping**

我们将学习：
- ping命令的工作原理
- ICMP协议详解
- RTT（往返时间）的计算
- 编写自己的ping工具

---

💡 **今日金句**：掌握TCP/IP，就掌握了互联网的语言！

👉 [上一天：OSI七层模型](day03.md) | [返回目录](../../README.md) | [下一天：ping命令详解](day05.md)

