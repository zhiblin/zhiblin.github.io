---
title: 35天：DNS深入 - 递归与迭代查询
date: 2025-11-10 15:25:56
categories: Network 
tags: Network, Book, Python
---
# 第35天：DNS深入 - 递归与迭代查询

> "DNS就像互联网的电话簿，将域名翻译成IP地址！"

## 📚 今日目标

- 深入理解DNS工作原理
- 掌握递归查询和迭代查询
- 学习DNS记录类型
- 理解DNS缓存机制
- 实现DNS查询工具

---

## 1. DNS基础回顾

### 1.1 DNS是什么？

```
DNS (Domain Name System) - 域名系统

作用：将人类可读的域名转换为机器可读的IP地址

示例：
www.example.com  →  93.184.216.34

生活化类比：
DNS = 电话簿
域名 = 人名（张三）
IP地址 = 电话号码（138-xxxx-xxxx）

你记得住"张三"，但记不住他的电话号码
DNS帮你把"张三"翻译成具体的电话号码
```

### 1.2 为什么需要DNS？

```
问题：如果没有DNS

访问网站需要记住IP地址：
- 访问百度：39.156.66.10
- 访问谷歌：172.217.160.68
- 访问GitHub：20.205.243.166

缺点：
❌ IP地址难记
❌ IP地址会变化
❌ 无法使用有意义的名称

有了DNS：
✅ 记住域名即可：www.baidu.com
✅ IP变化对用户透明
✅ 域名有意义且易记
```

---

## 2. DNS层级结构

### 2.1 DNS树状结构

```
DNS命名空间是一个树状层级结构：

                    根域 (.)
                      |
    ┌─────────────────┼─────────────────┐
    |                 |                 |
   com               org               cn
    |                 |                 |
┌───┴───┐         ┌───┴───┐         ┌───┴───┐
|       |         |       |         |       |
example google   wikipedia ietf    baidu  163
|
www

完整域名（FQDN）：www.example.com.
- www: 主机名
- example: 二级域名
- com: 顶级域名（TLD）
- .: 根域（通常省略）

域名层级（从右到左）：
1. 根域 (Root): .
2. 顶级域 (TLD): com, org, cn, net
3. 二级域 (SLD): example, google, baidu
4. 子域 (Subdomain): www, mail, ftp

常见顶级域名：
通用顶级域（gTLD）：
- .com: 商业
- .org: 组织
- .net: 网络
- .edu: 教育
- .gov: 政府

国家顶级域（ccTLD）：
- .cn: 中国
- .us: 美国
- .uk: 英国
- .jp: 日本
```

### 2.2 DNS服务器层级

```
DNS服务器架构：

┌─────────────────────────────────────────┐
│ 根DNS服务器 (Root DNS Servers)          │
│ - 全球13个根服务器集群                   │
│ - 知道所有TLD服务器的位置                │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 顶级域DNS服务器 (TLD DNS Servers)       │
│ - .com, .org, .cn等的权威服务器         │
│ - 知道各自域下的权威服务器位置           │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 权威DNS服务器 (Authoritative Servers)   │
│ - 存储域名的实际DNS记录                  │
│ - example.com的权威服务器                │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 本地DNS服务器 (Local/Recursive Servers) │
│ - ISP提供或自己配置                      │
│ - 执行实际的DNS查询                      │
│ - 缓存查询结果                           │
└─────────────────────────────────────────┘
```

---

## 3. DNS查询类型

### 3.1 递归查询（Recursive Query）

```
递归查询流程：

客户端只需要问一次，DNS服务器负责完成所有查询

┌──────────┐
│  客户端  │
└────┬─────┘
     │ 1. 查询 www.example.com
     ↓
┌────────────────┐
│ 本地DNS服务器  │ ← 递归查询的核心
└────┬───────────┘
     │ 2. 查询根服务器
     ↓
┌────────────────┐
│  根DNS服务器   │
└────┬───────────┘
     │ 3. 返回.com服务器地址
     ↓
┌────────────────┐
│ 本地DNS服务器  │
└────┬───────────┘
     │ 4. 查询.com服务器
     ↓
┌────────────────┐
│ .com DNS服务器 │
└────┬───────────┘
     │ 5. 返回example.com服务器地址
     ↓
┌────────────────┐
│ 本地DNS服务器  │
└────┬───────────┘
     │ 6. 查询example.com服务器
     ↓
┌─────────────────────┐
│ example.com服务器   │
└────┬────────────────┘
     │ 7. 返回www的IP地址
     ↓
┌────────────────┐
│ 本地DNS服务器  │
└────┬───────────┘
     │ 8. 返回最终答案
     ↓
┌──────────┐
│  客户端  │ ← 得到IP地址
└──────────┘

特点：
✅ 客户端简单，只问一次
✅ 服务器负责所有查询工作
✅ 结果可以被缓存
❌ 服务器负担重

生活化类比：
递归查询 = 找秘书帮忙
你："帮我查一下张三的电话"
秘书："好的，我帮你查"
秘书查遍所有通讯录，然后告诉你结果
```

### 3.2 迭代查询（Iterative Query）

```
迭代查询流程：

每次查询返回下一步应该问谁

┌──────────┐
│  客户端  │ （本地DNS服务器对客户端是递归）
└────┬─────┘
     │ 1. 查询 www.example.com
     ↓
┌────────────────┐
│ 本地DNS服务器  │ ← 从这里开始是迭代查询
└────┬───────────┘
     │ 2. 查询根服务器
     ↓
┌────────────────┐
│  根DNS服务器   │
└────┬───────────┘
     │ 3. 返回："我不知道，但你可以问.com服务器(192.x.x.x)"
     ↓
┌────────────────┐
│ 本地DNS服务器  │
└────┬───────────┘
     │ 4. 查询.com服务器(192.x.x.x)
     ↓
┌────────────────┐
│ .com DNS服务器 │
└────┬───────────┘
     │ 5. 返回："我不知道，但你可以问example.com服务器(93.x.x.x)"
     ↓
┌────────────────┐
│ 本地DNS服务器  │
└────┬───────────┘
     │ 6. 查询example.com服务器(93.x.x.x)
     ↓
┌─────────────────────┐
│ example.com服务器   │
└────┬────────────────┘
     │ 7. 返回："www.example.com的IP是93.184.216.34"
     ↓
┌────────────────┐
│ 本地DNS服务器  │
└────┬───────────┘
     │ 8. 返回最终答案
     ↓
┌──────────┐
│  客户端  │
└──────────┘

特点：
✅ 每个服务器只负责回答或指引
✅ 服务器负担轻
❌ 需要多次查询
❌ 客户端需要多次请求

生活化类比：
迭代查询 = 问路
你："请问图书馆怎么走？"
路人甲："我不知道，但你可以问前面的警察"
你走到警察处："请问图书馆怎么走？"
警察："直走500米左转"
```

### 3.3 查询对比

```
实际DNS查询过程：

客户端 → 本地DNS: 递归查询
本地DNS → 其他DNS: 迭代查询

为什么这样设计？
1. 简化客户端
2. 减轻根服务器负担
3. 利用本地DNS缓存

完整查询示例：

时间线：
T0: 客户端请求 www.example.com
    ├─ 客户端 → 本地DNS [递归查询]

T1: 本地DNS开始工作
    ├─ 检查缓存（未命中）
    ├─ 本地DNS → 根服务器 [迭代查询]
    └─ 根返回：请查询.com服务器(a.gtld-servers.net)

T2: 本地DNS继续
    ├─ 本地DNS → .com服务器 [迭代查询]
    └─ .com返回：请查询ns1.example.com

T3: 本地DNS最后查询
    ├─ 本地DNS → ns1.example.com [迭代查询]
    └─ ns1返回：93.184.216.34

T4: 结果返回
    ├─ 本地DNS缓存结果（TTL=300秒）
    └─ 本地DNS → 客户端 [返回答案]

总耗时：约50-100ms（首次查询）
```

---

## 4. DNS记录类型

### 4.1 常见DNS记录

```
DNS资源记录（Resource Records, RR）格式：
名称  TTL  类  类型  数据

主要记录类型：

1. A记录 (Address)
   - 域名 → IPv4地址
   - 示例：www.example.com. 300 IN A 93.184.216.34
   - 用途：最常用，将域名指向IPv4

2. AAAA记录
   - 域名 → IPv6地址
   - 示例：www.example.com. 300 IN AAAA 2606:2800:220:1:248:1893:25c8:1946
   - 用途：将域名指向IPv6

3. CNAME记录 (Canonical Name)
   - 域名 → 另一个域名（别名）
   - 示例：www.example.com. 300 IN CNAME example.com.
   - 用途：创建域名别名
   - 注意：CNAME不能与其他记录共存

4. MX记录 (Mail Exchange)
   - 域名 → 邮件服务器
   - 示例：example.com. 300 IN MX 10 mail.example.com.
   - 优先级：数字越小优先级越高
   - 用途：指定邮件服务器

5. NS记录 (Name Server)
   - 域名 → 权威DNS服务器
   - 示例：example.com. 300 IN NS ns1.example.com.
   - 用途：指定域名的DNS服务器

6. TXT记录
   - 域名 → 文本信息
   - 示例：example.com. 300 IN TXT "v=spf1 include:_spf.example.com ~all"
   - 用途：SPF、DKIM、域名验证等

7. PTR记录 (Pointer)
   - IP地址 → 域名（反向解析）
   - 示例：34.216.184.93.in-addr.arpa. 300 IN PTR www.example.com.
   - 用途：IP反向查询

8. SOA记录 (Start of Authority)
   - 域的权威信息
   - 包含：主DNS服务器、管理员邮箱、序列号、刷新时间等
   - 示例：
     example.com. 3600 IN SOA ns1.example.com. admin.example.com. (
       2024010601 ; 序列号
       3600       ; 刷新时间
       1800       ; 重试时间
       604800     ; 过期时间
       86400      ; 最小TTL
     )

9. SRV记录 (Service)
   - 服务位置记录
   - 示例：_http._tcp.example.com. 300 IN SRV 10 5 80 www.example.com.
   - 用途：指定服务的位置
```

### 4.2 DNS记录示例

```
完整的DNS区域文件示例：

; example.com DNS区域文件

; SOA记录
example.com.    3600    IN  SOA ns1.example.com. admin.example.com. (
                            2024010601  ; Serial
                            3600        ; Refresh
                            1800        ; Retry
                            604800      ; Expire
                            86400 )     ; Minimum TTL

; NS记录
example.com.    3600    IN  NS  ns1.example.com.
example.com.    3600    IN  NS  ns2.example.com.

; A记录
example.com.    300     IN  A   93.184.216.34
www.example.com. 300    IN  A   93.184.216.34
mail.example.com. 300   IN  A   93.184.216.35
ftp.example.com.  300   IN  A   93.184.216.36

; AAAA记录
www.example.com. 300    IN  AAAA 2606:2800:220:1:248:1893:25c8:1946

; MX记录
example.com.    300     IN  MX  10 mail.example.com.
example.com.    300     IN  MX  20 mail2.example.com.

; CNAME记录
blog.example.com. 300   IN  CNAME www.example.com.
shop.example.com. 300   IN  CNAME www.example.com.

; TXT记录
example.com.    300     IN  TXT "v=spf1 mx ~all"
_dmarc.example.com. 300 IN  TXT "v=DMARC1; p=none; rua=mailto:dmarc@example.com"

记录解读：
- TTL 300: 缓存5分钟
- TTL 3600: 缓存1小时
- IN: Internet类
- 优先级10 < 20: mail.example.com优先
```

---

## 5. DNS缓存机制

### 5.1 缓存层级

```
DNS缓存的多个层级：

1. 浏览器缓存
   - 最快，但容量小
   - Chrome: chrome://net-internals/#dns
   - TTL: 通常60秒

2. 操作系统缓存
   - Windows: ipconfig /displaydns
   - Linux: systemd-resolved
   - macOS: dscacheutil -cachedump

3. 本地DNS服务器缓存
   - ISP的DNS服务器
   - 或自己配置的DNS（如8.8.8.8）
   - TTL: 由记录的TTL决定

4. 其他DNS服务器缓存
   - 中间DNS服务器
   - 根据TTL缓存

缓存工作流程：
查询 www.example.com

Level 1: 浏览器缓存
  ├─ 命中 → 直接返回 ✅
  └─ 未命中 → 继续

Level 2: OS缓存
  ├─ 命中 → 返回 ✅
  └─ 未命中 → 继续

Level 3: 本地DNS缓存
  ├─ 命中 → 返回 ✅
  └─ 未命中 → 开始递归/迭代查询

Level 4: 实际DNS查询
  └─ 查询权威服务器 → 获取答案 → 各层缓存
```

### 5.2 TTL（生存时间）

```
TTL (Time To Live) - 缓存有效期

示例：
www.example.com. 300 IN A 93.184.216.34
                 ↑
                TTL=300秒（5分钟）

TTL的作用：
1. 控制缓存时间
2. 平衡性能和新鲜度
3. 减轻DNS服务器负担

TTL设置策略：

短TTL（60-300秒）：
✅ 变更生效快
✅ 适合频繁变更的记录
❌ 查询多，服务器压力大

长TTL（3600-86400秒）：
✅ 减少查询，降低延迟
✅ 减轻服务器负担
❌ 变更生效慢

推荐设置：
- 常用记录（www）: 300-3600秒
- 稳定记录（NS）: 3600-86400秒
- 邮件记录（MX）: 3600秒
- 变更前临时降低TTL
```

---

## 6. 实战项目：DNS查询工具

### 6.1 代码实现

```python
# day35_dns_query_tool.py
# 功能：DNS查询工具

import socket
import struct
from typing import List, Tuple, Dict
import time


class DNSQuery:
    """
    DNS查询工具

    功能：
        1. 构造DNS查询报文
        2. 发送查询
        3. 解析响应
        4. 支持多种记录类型
    """

    # DNS记录类型
    QTYPE = {
        'A': 1,      # IPv4地址
        'NS': 2,     # 域名服务器
        'CNAME': 5,  # 规范名称
        'SOA': 6,    # 授权开始
        'PTR': 12,   # 指针记录
        'MX': 15,    # 邮件交换
        'TXT': 16,   # 文本
        'AAAA': 28,  # IPv6地址
        'ANY': 255   # 所有记录
    }

    def __init__(self, dns_server='8.8.8.8', port=53):
        """
        初始化DNS查询器

        参数：
            dns_server: DNS服务器地址
            port: DNS端口（默认53）
        """
        self.dns_server = dns_server
        self.port = port

    def build_query(self, domain: str, qtype: str = 'A') -> bytes:
        """
        构造DNS查询报文

        参数：
            domain: 要查询的域名
            qtype: 查询类型（A, AAAA, MX等）

        返回：
            DNS查询报文（字节）

        DNS报文格式：
        +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
        |                    Header                     |
        +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
        |                   Question                    |
        +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
        |                    Answer                     |
        +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
        |                  Authority                    |
        +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
        |                  Additional                   |
        +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
        """
        # 1. 构造Header（12字节）
        transaction_id = 0x1234  # 事务ID
        flags = 0x0100  # 标准查询，期望递归
        questions = 1   # 问题数
        answers = 0     # 回答数
        authority = 0   # 权威记录数
        additional = 0  # 附加记录数

        header = struct.pack(
            '!HHHHHH',
            transaction_id, flags,
            questions, answers, authority, additional
        )

        # 2. 构造Question
        # 域名编码：将 www.example.com 编码为 \x03www\x07example\x03com\x00
        qname = b''
        for part in domain.split('.'):
            qname += bytes([len(part)]) + part.encode()
        qname += b'\x00'  # 结束标记

        qtype_value = self.QTYPE.get(qtype.upper(), 1)
        qclass = 1  # IN (Internet)

        question = qname + struct.pack('!HH', qtype_value, qclass)

        return header + question

    def parse_response(self, response: bytes) -> Dict:
        """
        解析DNS响应

        参数：
            response: DNS响应报文

        返回：
            解析后的结果字典
        """
        result = {
            'header': {},
            'questions': [],
            'answers': [],
            'authority': [],
            'additional': []
        }

        # 解析Header
        header = struct.unpack('!HHHHHH', response[:12])
        result['header'] = {
            'id': header[0],
            'flags': header[1],
            'questions': header[2],
            'answers': header[3],
            'authority': header[4],
            'additional': header[5]
        }

        # 解析Answer部分（简化版本）
        offset = 12

        # 跳过Question部分
        while response[offset] != 0:
            offset += 1
        offset += 5  # 跳过结束符和QTYPE、QCLASS

        # 解析Answer
        for _ in range(result['header']['answers']):
            # 简化：直接提取IP地址（假设是A记录）
            if offset + 12 <= len(response):
                # 跳过名称（指针）
                offset += 2

                # 读取类型、类、TTL、数据长度
                rtype, rclass, ttl, rdlength = struct.unpack(
                    '!HHIH',
                    response[offset:offset+10]
                )
                offset += 10

                # 读取数据
                rdata = response[offset:offset+rdlength]
                offset += rdlength

                if rtype == 1 and rdlength == 4:  # A记录
                    ip = '.'.join(str(b) for b in rdata)
                    result['answers'].append({
                        'type': 'A',
                        'ttl': ttl,
                        'data': ip
                    })

        return result

    def query(self, domain: str, qtype: str = 'A', verbose: bool = True) -> Dict:
        """
        执行DNS查询

        参数：
            domain: 域名
            qtype: 查询类型
            verbose: 是否显示详细信息

        返回：
            查询结果
        """
        if verbose:
            print(f"\n正在查询: {domain} ({qtype}记录)")
            print(f"DNS服务器: {self.dns_server}:{self.port}")

        # 构造查询
        query_data = self.build_query(domain, qtype)

        # 创建UDP socket
        sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        sock.settimeout(3)

        try:
            # 发送查询
            start_time = time.time()
            sock.sendto(query_data, (self.dns_server, self.port))

            # 接收响应
            response, _ = sock.recvfrom(512)
            query_time = (time.time() - start_time) * 1000

            # 解析响应
            result = self.parse_response(response)
            result['query_time'] = query_time

            if verbose:
                print(f"查询时间: {query_time:.2f}ms")
                print(f"回答数量: {result['header']['answers']}")

                if result['answers']:
                    print("\n结果:")
                    for answer in result['answers']:
                        print(f"  类型: {answer['type']}")
                        print(f"  TTL: {answer['ttl']}秒")
                        print(f"  数据: {answer['data']}")

            return result

        except socket.timeout:
            if verbose:
                print("❌ 查询超时")
            return {'error': 'timeout'}
        except Exception as e:
            if verbose:
                print(f"❌ 查询失败: {e}")
            return {'error': str(e)}
        finally:
            sock.close()

    def query_with_system(self, domain: str, verbose: bool = True):
        """
        使用系统DNS查询（对比）

        参数：
            domain: 域名
            verbose: 是否显示详细信息
        """
        if verbose:
            print(f"\n使用系统DNS查询: {domain}")

        try:
            start_time = time.time()
            result = socket.getaddrinfo(domain, None)
            query_time = (time.time() - start_time) * 1000

            if verbose:
                print(f"查询时间: {query_time:.2f}ms")
                print("\n结果:")
                for item in result:
                    if item[0] == socket.AF_INET:  # IPv4
                        print(f"  IPv4: {item[4][0]}")
                    elif item[0] == socket.AF_INET6:  # IPv6
                        print(f"  IPv6: {item[4][0]}")

            return {'results': result, 'query_time': query_time}

        except Exception as e:
            if verbose:
                print(f"❌ 查询失败: {e}")
            return {'error': str(e)}


def compare_dns_servers():
    """对比不同DNS服务器的性能"""
    print("\n" + "="*60)
    print(" DNS服务器性能对比")
    print("="*60)

    dns_servers = [
        ('Google DNS', '8.8.8.8'),
        ('Cloudflare DNS', '1.1.1.1'),
        ('114 DNS', '114.114.114.114'),
    ]

    test_domains = [
        'www.baidu.com',
        'www.google.com',
        'www.github.com'
    ]

    print(f"\n测试域名: {', '.join(test_domains)}\n")

    for name, server in dns_servers:
        print(f"\n{name} ({server}):")
        print("-" * 60)

        query_tool = DNSQuery(server)
        total_time = 0
        success_count = 0

        for domain in test_domains:
            result = query_tool.query(domain, verbose=False)
            if 'query_time' in result:
                total_time += result['query_time']
                success_count += 1
                print(f"  {domain}: {result['query_time']:.2f}ms")

        if success_count > 0:
            avg_time = total_time / success_count
            print(f"\n  平均查询时间: {avg_time:.2f}ms")


def main():
    """主程序"""
    print("""
    ╔══════════════════════════════════════════════════════════╗
    ║       DNS查询工具 v1.0                                    ║
    ║       DNS Query Tool                                     ║
    ╚══════════════════════════════════════════════════════════╝
    """)

    # 创建DNS查询器
    dns_query = DNSQuery(dns_server='8.8.8.8')

    # 演示1：基本查询
    print("\n【演示1：基本DNS查询】")
    dns_query.query('www.baidu.com', 'A')

    input("\n按Enter继续...")

    # 演示2：系统查询对比
    print("\n【演示2：自定义vs系统DNS查询对比】")
    dns_query.query_with_system('www.baidu.com')

    input("\n按Enter继续...")

    # 演示3：DNS服务器性能对比
    print("\n【演示3：不同DNS服务器性能对比】")
    compare_dns_servers()

    print("""
\n总结：
DNS查询过程包括：
1. 构造查询报文
2. 发送UDP查询（端口53）
3. 接收响应
4. 解析结果

优化建议：
- 使用快速DNS服务器
- 合理设置TTL
- 启用DNS缓存
- 考虑使用DNS over HTTPS (DoH)

明天我们将学习DNS安全（DNSSEC）！
    """)


if __name__ == "__main__":
    main()
