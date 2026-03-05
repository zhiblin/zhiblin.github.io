---
title: 6天：网络命令进阶 - traceroute
date: 2025-11-10 14:39:39
categories: Network 
tags: [Network, Book, Python]
---
# 第6天：网络命令进阶 - traceroute

> "看看数据包是怎样环游世界的。"

## 📚 今日目标

- 理解traceroute的工作原理
- 掌握如何追踪网络路径
- 深入理解TTL的作用机制
- 编写自己的traceroute工具

## 🎯 什么是traceroute？

### 生活中的例子

```
traceroute = 追踪快递路线

你的包裹从北京到上海：
北京 → 石家庄中转站 → 济南中转站 → 南京中转站 → 上海

traceroute就是显示数据包经过的所有"中转站"（路由器）
```

**traceroute**（Windows上叫tracert）是用来显示数据包从源到目的地所经过的路由路径的工具。

### traceroute的作用

1. **诊断网络故障**：找出哪个路由器出问题了
2. **分析网络路径**：了解数据包的传输路线
3. **测量延迟**：看每一跳的延迟
4. **发现网络瓶颈**：找出哪里最慢

## 🔍 traceroute的工作原理

### 核心思想：利用TTL

还记得昨天学的TTL吗？
- 每个数据包都有TTL（Time To Live）
- 每经过一个路由器，TTL减1
- 当TTL=0时，路由器丢弃数据包，并返回"超时"错误（ICMP Time Exceeded）

**traceroute的巧妙之处**：故意设置小的TTL值！

### 完整流程

```
第1步：发送TTL=1的包
┌──────┐  TTL=1   ┌────────┐
│ 你的 │ ───────> │ 路由器1 │
│ 电脑 │          └────────┘
└──────┘          TTL变成0！
                  返回"超时"消息
                  (ICMP Time Exceeded)

→ 得知第1跳是路由器1

第2步：发送TTL=2的包
┌──────┐  TTL=2   ┌────────┐  TTL=1   ┌────────┐
│ 你的 │ ───────> │ 路由器1 │ ───────> │ 路由器2 │
│ 电脑 │          └────────┘          └────────┘
└──────┘                              TTL变成0！
                                      返回"超时"消息

→ 得知第2跳是路由器2

第3步：发送TTL=3的包
（依此类推，直到到达目的地）
```

### 图解示意

```
你的电脑 (192.168.1.100)
    |
    | TTL=1
    v
路由器1 (192.168.1.1)     → 返回: Time Exceeded
    |
    | TTL=2
    v
路由器2 (10.0.0.1)        → 返回: Time Exceeded
    |
    | TTL=3
    v
路由器3 (61.135.169.1)    → 返回: Time Exceeded
    |
    | TTL=4
    v
目标服务器 (220.181.38.148) → 返回: Echo Reply
```

## 🔧 使用traceroute命令

### 基本用法

**Linux/Mac**：
```bash
# 基本traceroute
traceroute baidu.com

# 限制最大跳数
traceroute -m 20 baidu.com

# 不解析域名（更快）
traceroute -n baidu.com

# 使用ICMP而不是UDP
traceroute -I baidu.com
```

**Windows**：
```bash
# 基本tracert
tracert baidu.com

# 限制最大跳数
tracert -h 20 baidu.com

# 不解析域名
tracert -d baidu.com
```

### 输出解析

```bash
$ traceroute baidu.com

traceroute to baidu.com (220.181.38.148), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)          1.234 ms  1.123 ms  1.089 ms
 2  10.0.0.1 (10.0.0.1)                5.678 ms  5.432 ms  5.345 ms
 3  61.135.169.1 (61.135.169.1)        12.345 ms 12.234 ms 12.123 ms
 4  61.135.169.125 (61.135.169.125)    15.678 ms 15.567 ms 15.456 ms
 5  * * *
 6  220.181.38.148 (220.181.38.148)    28.901 ms 28.789 ms 28.678 ms
```

**字段含义**：
- 第1列：跳数（hop number）
- 第2列：路由器的IP地址和主机名
- 后3列：三次测试的RTT（往返时间）
- `* * *`：该路由器没有回应（可能禁用了ICMP）

## 🔧 实战项目：实现简易traceroute工具

让我们用Python实现traceroute！

### 代码实现

```python
# day06_simple_traceroute.py
# 功能：实现简单的traceroute工具

import socket
import struct
import time
import sys

def calculate_checksum(data):
    """
    计算ICMP校验和（与ping工具中的相同）
    """
    if len(data) % 2 != 0:
        data += b'\x00'

    checksum = 0
    for i in range(0, len(data), 2):
        word = (data[i] << 8) + data[i + 1]
        checksum += word

    while checksum >> 16:
        checksum = (checksum & 0xFFFF) + (checksum >> 16)

    checksum = ~checksum & 0xFFFF
    return checksum

def create_icmp_packet(packet_id, sequence):
    """
    创建ICMP Echo Request数据包
    """
    icmp_type = 8  # Echo Request
    icmp_code = 0
    icmp_checksum = 0

    header = struct.pack('!BBHHH', icmp_type, icmp_code, icmp_checksum,
                         packet_id, sequence)
    data = struct.pack('!d', time.time())

    icmp_checksum = calculate_checksum(header + data)
    header = struct.pack('!BBHHH', icmp_type, icmp_code, icmp_checksum,
                         packet_id, sequence)

    return header + data

def parse_icmp_reply(packet):
    """
    解析ICMP回复

    返回：
        (icmp_type, source_ip)
    """
    # 解析IP头部，获取源IP
    ip_header = packet[:20]
    iph = struct.unpack('!BBHHHBBH4s4s', ip_header)
    source_ip = socket.inet_ntoa(iph[8])

    # 解析ICMP头部
    icmp_header = packet[20:28]
    icmp_type, icmp_code = struct.unpack('!BB', icmp_header[:2])

    return icmp_type, source_ip

def traceroute(host, max_hops=30, timeout=2, tries=3):
    """
    追踪到目标主机的路由路径

    参数：
        host: 目标主机名或IP
        max_hops: 最大跳数
        timeout: 超时时间（秒）
        tries: 每一跳尝试次数

    工作原理：
        1. 从TTL=1开始，逐步增加TTL
        2. 发送ICMP Echo Request
        3. 收到Time Exceeded回复时，记录该路由器
        4. 收到Echo Reply时，说明到达目的地
    """
    print(f"traceroute to {host}, {max_hops} hops max")
    print()

    # 解析目标主机
    try:
        dest_ip = socket.gethostbyname(host)
        print(f"目标IP: {dest_ip}")
        print("-" * 70)
    except socket.gaierror:
        print(f"❌ 无法解析主机名: {host}")
        return

    # 创建原始socket（需要root权限）
    try:
        send_sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_ICMP)
        recv_sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_ICMP)
        recv_sock.settimeout(timeout)
    except PermissionError:
        print("❌ 需要管理员权限运行此程序")
        print("💡 Linux/Mac: sudo python day06_simple_traceroute.py")
        print("💡 Windows: 以管理员身份运行")
        return

    packet_id = 12345
    reached = False

    # 从TTL=1开始，逐步增加
    for ttl in range(1, max_hops + 1):
        # 设置TTL
        send_sock.setsockopt(socket.IPPROTO_IP, socket.IP_TTL, ttl)

        # 尝试3次
        hop_info = {
            'ttl': ttl,
            'ip': None,
            'hostname': None,
            'rtts': []
        }

        print(f" {ttl:2d}  ", end="", flush=True)

        for attempt in range(tries):
            try:
                # 创建并发送ICMP包
                packet = create_icmp_packet(packet_id, ttl)
                send_time = time.time()
                send_sock.sendto(packet, (dest_ip, 0))

                # 接收回复
                try:
                    reply_packet, addr = recv_sock.recvfrom(1024)
                    recv_time = time.time()
                    rtt = (recv_time - send_time) * 1000  # 转换为毫秒

                    # 解析回复
                    icmp_type, source_ip = parse_icmp_reply(reply_packet)

                    # 记录信息
                    if hop_info['ip'] is None:
                        hop_info['ip'] = source_ip

                        # 尝试获取主机名
                        try:
                            hostname = socket.gethostbyaddr(source_ip)[0]
                            hop_info['hostname'] = hostname
                        except:
                            hop_info['hostname'] = source_ip

                    hop_info['rtts'].append(rtt)

                    # ICMP类型11 = Time Exceeded（中间路由器）
                    # ICMP类型0 = Echo Reply（到达目的地）
                    if icmp_type == 0:
                        reached = True

                except socket.timeout:
                    hop_info['rtts'].append(None)

            except Exception as e:
                hop_info['rtts'].append(None)

        # 打印这一跳的信息
        if hop_info['ip']:
            if hop_info['hostname'] and hop_info['hostname'] != hop_info['ip']:
                print(f"{hop_info['hostname']} ({hop_info['ip']})  ", end="")
            else:
                print(f"{hop_info['ip']}  ", end="")

            # 打印RTT
            for rtt in hop_info['rtts']:
                if rtt is not None:
                    print(f"{rtt:.2f} ms  ", end="")
                else:
                    print("*  ", end="")
        else:
            print("* * *", end="")

        print()  # 换行

        # 如果到达目的地，停止
        if reached:
            print()
            print(f"✅ 已到达目的地 {dest_ip}")
            break

    if not reached:
        print()
        print(f"⚠️  未能在{max_hops}跳内到达目的地")

    send_sock.close()
    recv_sock.close()

def main():
    """
    主函数
    """
    print("=" * 70)
    print("简易Traceroute工具".center(70))
    print("=" * 70)
    print()

    # 获取目标主机
    if len(sys.argv) > 1:
        host = sys.argv[1]
    else:
        host = input("请输入要追踪的主机名或IP: ").strip()
        if not host:
            host = "baidu.com"  # 默认

    print(f"\n🔍 开始追踪路由路径...\n")

    # 执行traceroute
    traceroute(host, max_hops=30, timeout=2, tries=3)

    print()
    print("=" * 70)
    print("\n💡 说明:")
    print("   - 每一行代表一跳（一个路由器）")
    print("   - 显示3次测试的RTT时间")
    print("   - * 表示该路由器没有响应")
    print("   - 某些路由器可能禁用了ICMP响应")
    print("=" * 70)

if __name__ == "__main__":
    main()
```

### 运行程序

```bash
# Linux/Mac（需要root权限）
sudo python code/day06/day06_simple_traceroute.py baidu.com

# Windows（以管理员身份运行）
python code\day06\day06_simple_traceroute.py baidu.com
```

### 预期输出

```
======================================================================
                      简易Traceroute工具
======================================================================

🔍 开始追踪路由路径...

traceroute to baidu.com, 30 hops max

目标IP: 220.181.38.148
----------------------------------------------------------------------
  1  192.168.1.1  1.23 ms  1.12 ms  1.09 ms
  2  10.0.0.1  5.67 ms  5.43 ms  5.34 ms
  3  61.135.169.1  12.34 ms  12.23 ms  12.12 ms
  4  61.135.169.125  15.67 ms  15.56 ms  15.45 ms
  5  * * *
  6  220.181.38.148  28.90 ms  28.78 ms  28.67 ms

✅ 已到达目的地 220.181.38.148

======================================================================

💡 说明:
   - 每一行代表一跳（一个路由器）
   - 显示3次测试的RTT时间
   - * 表示该路由器没有响应
   - 某些路由器可能禁用了ICMP响应
======================================================================
```

## 🔍 代码详解

### 1. 设置TTL

```python
send_sock.setsockopt(socket.IPPROTO_IP, socket.IP_TTL, ttl)
```

这是关键！通过设置socket的TTL选项，我们可以控制数据包能经过多少个路由器。

### 2. ICMP消息类型

```python
# 收到的ICMP类型：
- 类型11（Time Exceeded）：中间路由器返回，说明TTL耗尽
- 类型0（Echo Reply）：目标主机返回，说明到达目的地
```

### 3. 为什么要尝试3次？

```python
for attempt in range(tries):  # 尝试3次
```

**原因**：
1. 网络可能不稳定，单次测试不准确
2. 可以计算平均延迟
3. 可以检测丢包情况
4. 与标准traceroute行为一致

### 4. 为什么有些路由器显示 * * * ？

可能的原因：
1. **防火墙阻止**：该路由器配置了防火墙，不响应ICMP
2. **安全策略**：运营商出于安全考虑，禁用了ICMP响应
3. **负载均衡**：每次请求可能走不同路径
4. **超时**：路由器太忙，来不及响应

## 🎓 知识小结

### 今天学到了什么？

1. ✅ traceroute通过逐步增加TTL来追踪路径
2. ✅ TTL=0时路由器返回"Time Exceeded"消息
3. ✅ 每一跳通常测试3次以获得稳定数据
4. ✅ 某些路由器可能不响应ICMP

### traceroute vs ping

| 特性 | ping | traceroute |
|------|------|------------|
| 目的 | 测试连通性 | 追踪路径 |
| 显示内容 | 目标主机信息 | 所有中间路由器 |
| TTL设置 | 固定（通常64） | 递增（1, 2, 3...） |
| 数据包数量 | 少 | 多 |
| 执行速度 | 快 | 慢 |

### Windows vs Linux/Mac

| 特性 | Windows (tracert) | Linux/Mac (traceroute) |
|------|-------------------|------------------------|
| 协议 | ICMP | UDP（默认）或ICMP |
| 命令 | tracert | traceroute |
| ICMP选项 | 默认 | -I 参数 |

## 💪 练习题

### 练习1：实际操作
使用traceroute追踪以下网站，记录经过的跳数：
1. baidu.com
2. google.com（如果能访问）
3. 你的学校或公司网站
4. 国外的网站（观察延迟变化）

### 练习2：路径分析
观察traceroute的输出，回答：
1. 第一跳通常是什么设备？
2. 为什么有些跳的延迟突然增大？
3. 国内访问国外网站，通常在第几跳出国？

### 练习3：故障诊断
如果traceroute显示以下情况，说明什么问题？
```
1  192.168.1.1  1 ms
2  * * *
3  * * *
4  * * *
...
```

### 练习4：代码扩展
扩展traceroute工具：
1. 添加路径可视化（绘制路由图）
2. 支持同时追踪多个目标
3. 记录历史路径，对比路径变化
4. 添加AS号查询（自治系统号）

### 练习5：思考题
1. 为什么去程和回程路径可能不同？
2. traceroute能用来攻击吗？
3. 如何阻止别人traceroute你的服务器？

## 📚 扩展阅读

### traceroute的不同实现方式

#### 1. UDP方式（Linux默认）

```bash
# 使用UDP包，目标端口从33434开始递增
traceroute baidu.com

工作原理：
- 发送UDP包到不常用的端口
- 中间路由器：返回ICMP Time Exceeded
- 目标主机：返回ICMP Port Unreachable
```

#### 2. ICMP方式（Windows默认）

```bash
# 使用ICMP Echo Request
traceroute -I baidu.com  # Linux
tracert baidu.com        # Windows

工作原理：
- 发送ICMP Echo Request
- 中间路由器：返回ICMP Time Exceeded
- 目标主机：返回ICMP Echo Reply
```

#### 3. TCP方式

```bash
# 使用TCP SYN包（需要指定端口）
traceroute -T -p 80 baidu.com

工作原理：
- 发送TCP SYN包到指定端口
- 中间路由器：返回ICMP Time Exceeded
- 目标主机：返回TCP SYN+ACK
```

### MTR：增强版traceroute

MTR（My Traceroute）结合了ping和traceroute的功能：

```bash
# 安装
sudo apt install mtr      # Ubuntu/Debian
brew install mtr          # Mac

# 使用
sudo mtr baidu.com
```

**MTR的优势**：
- 实时更新
- 显示丢包率
- 显示每一跳的统计信息
- 更直观的界面

### 路由的变化

互联网的路由不是固定的，会因为以下原因变化：

1. **负载均衡**：分散流量到不同路径
2. **路由策略**：运营商动态调整路由
3. **故障切换**：某条线路故障时自动切换
4. **时间因素**：不同时段可能走不同路由

### 实际应用场景

1. **网络故障诊断**：
   ```
   情况：网站访问很慢
   步骤：traceroute 查看哪一跳延迟高
   结果：定位瓶颈所在
   ```

2. **CDN检测**：
   ```
   情况：想知道访问的是哪个CDN节点
   步骤：traceroute 查看最后几跳
   结果：发现CDN在本地，延迟低
   ```

3. **网络拓扑发现**：
   ```
   情况：想了解公司网络结构
   步骤：traceroute 到不同内网地址
   结果：绘制网络拓扑图
   ```

### 有趣的发现

**地理位置与延迟**：
```
本地路由器：    1-2 ms
同城服务器：    10-30 ms
相邻省份：      30-60 ms
跨国连接：      100-300 ms
全球另一端：    300-500 ms
```

**光速的限制**：
- 光在光纤中的速度约为 200,000 km/s
- 地球周长约 40,000 km
- 理论上环绕地球一圈至少需要 200ms
- 实际延迟会更高（路由器处理时间）

## 🎯 明天预告

**第7天：Wireshark抓包入门**

我们将学习：
- Wireshark的基本使用
- 如何捕获和分析数据包
- 过滤器的使用技巧
- 实战：分析HTTP请求过程

---

💡 **今日金句**：追踪路由，就像在互联网上寻找宝藏的地图！

👉 [上一天：ping命令详解](day05.md) | [返回目录](../../README.md) | [下一天：Wireshark抓包入门](day07.md)

