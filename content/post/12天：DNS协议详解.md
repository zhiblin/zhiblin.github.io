---
title: 12天：DNS协议详解
date: 2025-11-10 15:21:39
categories: Network 
tags: [Network, Book, Python]
---
# 第12天：DNS协议详解

## 今日目标

- 理解DNS的工作原理和架构
- 掌握DNS查询过程
- 了解常见的DNS记录类型
- 学会使用Python实现DNS查询工具

---

## 1. 什么是DNS？

### 1.1 DNS的定义

**DNS（Domain Name System，域名系统）** 是互联网的"电话簿"。

**生活类比**：
- 你记得朋友的名字，但不记得他的电话号码
- 你查通讯录，通过名字找到电话号码
- DNS就是这样一个通讯录，把域名翻译成IP地址

**为什么需要DNS？**

```
人类喜欢记忆：www.baidu.com
计算机需要使用：110.242.68.66

DNS的作用：把域名转换为IP地址
```

### 1.2 DNS的重要性

**如果没有DNS会怎样？**

```
访问百度：http://110.242.68.66  ❌ 很难记
访问淘宝：http://106.11.248.146 ❌ 更难记
访问微信：http://101.32.118.25  ❌ 记不住

有了DNS：
访问百度：www.baidu.com  ✅ 容易记
访问淘宝：www.taobao.com ✅ 好记
访问微信：www.weixin.com ✅ 简单
```

---

## 2. DNS架构体系

### 2.1 DNS层级结构

DNS采用**分布式、层级化**的设计，就像一个倒置的树形结构：

```
                         .（根域）
                          |
         +----------------+----------------+
         |                |                |
       com              cn               org
         |                |                |
    +----+----+      +----+----+      +----+----+
    |         |      |         |      |         |
  baidu    google  sina     163    wikipedia  linux
    |         |      |         |        |        |
   www       www    www       www      www      www
```

**域名层级（从右到左）**：

1. **根域（Root Domain）**：`.`
   - 最顶层，通常被省略
   - 全球只有13组根域名服务器

2. **顶级域（TLD - Top Level Domain）**：
   - 通用顶级域：`.com` `.net` `.org` `.edu` `.gov`
   - 国家顶级域：`.cn` `.us` `.uk` `.jp`
   - 新顶级域：`.tech` `.xyz` `.top`

3. **二级域（SLD - Second Level Domain）**：
   - `baidu.com` 中的 `baidu`
   - `google.com` 中的 `google`

4. **三级域（子域）**：
   - `www.baidu.com` 中的 `www`
   - `mail.google.com` 中的 `mail`

**完整域名示例**：

```
www.baidu.com.
 |   |     |  |
 |   |     |  +-- 根域（通常省略）
 |   |     +----- 顶级域
 |   +----------- 二级域
 +--------------- 三级域（子域）
```

### 2.2 DNS服务器类型

```
┌─────────────────────────────────────────┐
│         DNS服务器类型                     │
├─────────────────────────────────────────┤
│ 1. 根DNS服务器（Root DNS Server）        │
│    - 全球13组                            │
│    - 知道所有顶级域服务器的地址          │
│                                          │
│ 2. 顶级域DNS服务器（TLD DNS Server）     │
│    - 管理 .com、.cn 等顶级域             │
│    - 知道所有注册域的权威服务器          │
│                                          │
│ 3. 权威DNS服务器（Authoritative DNS）    │
│    - 存储域名的真实IP地址                │
│    - 由域名所有者管理                    │
│                                          │
│ 4. 本地DNS服务器（Local DNS Server）     │
│    - 通常由ISP提供                       │
│    - 缓存查询结果                        │
│    - 代替客户端执行递归查询              │
└─────────────────────────────────────────┘
```

---

## 3. DNS查询过程

### 3.1 递归查询 vs 迭代查询

**递归查询（Recursive Query）**：
- 客户端发出请求后，DNS服务器负责完成所有查询
- "帮我查到结果再告诉我"

**迭代查询（Iterative Query）**：
- DNS服务器只返回下一步应该查询的服务器地址
- "我不知道，但你可以去问XX服务器"

### 3.2 完整的DNS查询流程

**场景**：你在浏览器输入 `www.example.com`

```
步骤1：检查本地缓存
─────────────────────────
[你的电脑]
  ↓
先查看本地DNS缓存
  ↓
如果有缓存 → 直接返回IP ✅
如果没有 → 继续查询

步骤2：查询本地DNS服务器
─────────────────────────
[你的电脑] → [本地DNS服务器]
               （通常是运营商的DNS）
  ↓
检查缓存
  ↓
如果有 → 返回IP ✅
如果没有 → 开始递归查询

步骤3：查询根DNS服务器
─────────────────────────
[本地DNS] → [根DNS服务器]
问：www.example.com 的IP是多少？
答：我不知道，但 .com 的地址我知道，去问它

步骤4：查询顶级域DNS服务器
─────────────────────────
[本地DNS] → [.com DNS服务器]
问：www.example.com 的IP是多少？
答：我不知道，但 example.com 的权威DNS我知道，去问它

步骤5：查询权威DNS服务器
─────────────────────────
[本地DNS] → [example.com 权威DNS]
问：www.example.com 的IP是多少？
答：93.184.216.34 ✅

步骤6：返回结果并缓存
─────────────────────────
[本地DNS] → [你的电脑]
返回：93.184.216.34
并且缓存结果（TTL时间内有效）
```

**时间流程图**：

```
时间轴：
─────────────────────────────────────────────────
t0: 用户输入 www.example.com
    ↓
t1: 查询本地缓存（1ms）
    ↓
t2: 查询本地DNS服务器（5ms）
    ↓
t3: 本地DNS查询根服务器（50ms）
    ↓
t4: 本地DNS查询.com服务器（30ms）
    ↓
t5: 本地DNS查询权威服务器（20ms）
    ↓
t6: 返回结果给用户（5ms）
    ↓
总耗时：约 111ms
```

### 3.3 DNS缓存机制

**多级缓存**：

```
1. 浏览器缓存
   ├── Chrome: chrome://net-internals/#dns
   ├── Firefox: about:networking#dns
   └── 缓存时间：通常几分钟

2. 操作系统缓存
   ├── Windows: ipconfig /displaydns
   ├── Linux: nscd
   └── 缓存时间：根据TTL

3. 路由器缓存
   └── 家庭网关会缓存DNS结果

4. 本地DNS服务器缓存
   └── ISP的DNS服务器
   └── 缓存时间：根据TTL

5. 权威DNS服务器
   └── 最终的真实数据源
```

**TTL（Time To Live）**：

```
DNS记录都有TTL值，表示缓存有效期

示例：
www.example.com.  300  IN  A  93.184.216.34
                   ↑
                  TTL = 300秒（5分钟）

含义：
- 这条记录可以被缓存5分钟
- 5分钟后需要重新查询
- TTL短：更新快，但查询频繁
- TTL长：查询少，但更新慢
```

---

## 4. DNS记录类型

### 4.1 常见DNS记录类型

| 记录类型 | 全称 | 作用 | 示例 |
|---------|------|------|------|
| **A** | Address | IPv4地址 | `www.example.com → 93.184.216.34` |
| **AAAA** | IPv6 Address | IPv6地址 | `www.example.com → 2606:2800:220:1:...` |
| **CNAME** | Canonical Name | 别名 | `blog.example.com → www.example.com` |
| **MX** | Mail Exchange | 邮件服务器 | `example.com → mail.example.com` |
| **NS** | Name Server | 域名服务器 | `example.com → ns1.example.com` |
| **TXT** | Text | 文本信息 | `SPF、DKIM等验证信息` |
| **PTR** | Pointer | 反向解析 | `34.216.184.93.in-addr.arpa → www.example.com` |
| **SOA** | Start of Authority | 授权起始 | 域的管理信息 |
| **SRV** | Service | 服务位置 | 特定服务的主机和端口 |

### 4.2 记录类型详解

**A记录（最常用）**：

```
用途：将域名解析为IPv4地址

配置示例：
www.example.com.    IN  A  93.184.216.34
api.example.com.    IN  A  93.184.216.35
cdn.example.com.    IN  A  104.16.132.229

一个域名可以有多个A记录（负载均衡）：
www.example.com.    IN  A  93.184.216.34
www.example.com.    IN  A  93.184.216.35
www.example.com.    IN  A  93.184.216.36
```

**AAAA记录**：

```
用途：将域名解析为IPv6地址

配置示例：
www.example.com.    IN  AAAA  2606:2800:220:1:248:1893:25c8:1946
```

**CNAME记录**：

```
用途：创建域名别名

配置示例：
www.example.com.     IN  A      93.184.216.34
blog.example.com.    IN  CNAME  www.example.com.
forum.example.com.   IN  CNAME  www.example.com.

解析过程：
用户访问 blog.example.com
  ↓ CNAME
指向 www.example.com
  ↓ A记录
解析为 93.184.216.34

注意：CNAME不能与其他记录共存
```

**MX记录**：

```
用途：指定邮件服务器

配置示例：
example.com.    IN  MX  10  mail1.example.com.
example.com.    IN  MX  20  mail2.example.com.
                      ↑
                   优先级（越小越优先）

邮件发送过程：
发送到 user@example.com
  ↓
查询 example.com 的 MX记录
  ↓
找到 mail1.example.com（优先级10）
  ↓
查询 mail1.example.com 的 A记录
  ↓
获得邮件服务器IP，发送邮件
```

**NS记录**：

```
用途：指定域名服务器

配置示例：
example.com.    IN  NS  ns1.example.com.
example.com.    IN  NS  ns2.example.com.

通常配置多个NS记录（冗余备份）
```

**TXT记录**：

```
用途：存储文本信息（常用于验证）

配置示例：

1. SPF记录（防止邮件伪造）：
   example.com.  IN  TXT  "v=spf1 mx include:_spf.google.com ~all"

2. DKIM（邮件签名）：
   selector._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA..."

3. 域名验证：
   example.com.  IN  TXT  "google-site-verification=rXOxyZounnZasA8Z7oaD3c14JdjS9aKSWvsR1EbUSIQ"
```

**PTR记录（反向解析）**：

```
用途：从IP地址查询域名

配置示例：
34.216.184.93.in-addr.arpa.  IN  PTR  www.example.com.

应用场景：
- 邮件服务器验证
- 安全审计
- 日志分析
```

---

## 5. 实战项目：DNS查询工具

### 5.1 项目功能

我们将实现一个DNS查询工具，支持：

1. 查询各种DNS记录类型
2. 指定DNS服务器
3. 显示详细的查询结果
4. 性能分析（查询时间）

### 5.2 工具演示

```bash
# 查询A记录
python day12_dns_tool.py www.baidu.com

# 查询指定类型
python day12_dns_tool.py www.baidu.com MX

# 指定DNS服务器
python day12_dns_tool.py www.baidu.com A 8.8.8.8

# 查询所有记录
python day12_dns_tool.py example.com ALL
```

---

## 6. 知识拓展

### 6.1 公共DNS服务器

**国内公共DNS**：

```
阿里DNS：
- 223.5.5.5
- 223.6.6.6

腾讯DNS：
- 119.29.29.29

百度DNS：
- 180.76.76.76

114DNS：
- 114.114.114.114
```

**国际公共DNS**：

```
Google DNS：
- 8.8.8.8
- 8.8.4.4

Cloudflare DNS：
- 1.1.1.1
- 1.0.0.1

OpenDNS：
- 208.67.222.222
- 208.67.220.220
```

### 6.2 DNS安全问题

**1. DNS劫持**：

```
正常流程：
用户 → www.bank.com → 真实银行IP

被劫持：
用户 → www.bank.com → 假冒钓鱼网站IP

防护措施：
- 使用可信DNS服务器
- 启用DNSSEC
- 使用HTTPS
```

**2. DNS污染**：

```
什么是DNS污染？
- DNS服务器返回错误的IP地址
- 通常是某些地区的网络审查手段

现象：
- 某些网站无法访问
- 访问被重定向到其他页面

解决方法：
- 更换DNS服务器
- 使用VPN
- 使用加密DNS（DoH/DoT）
```

**3. DNS放大攻击**：

```
攻击原理：
1. 攻击者伪造受害者IP
2. 向DNS服务器发送查询请求
3. DNS服务器将大量响应发送给受害者
4. 造成受害者网络瘫痪

特点：
- 小请求 → 大响应（放大效应）
- 典型放大倍数：28-54倍
```

### 6.3 新一代DNS技术

**DoH（DNS over HTTPS）**：

```
传统DNS：
明文传输，端口53，易被监听和劫持

DoH：
通过HTTPS加密传输，端口443
- 隐私保护
- 防止劫持
- 绕过某些限制

支持DoH的浏览器：
- Firefox
- Chrome
- Edge
```

**DoT（DNS over TLS）**：

```
类似DoH，但使用TLS协议
端口：853

区别：
DoH：使用HTTPS，融入正常网页流量
DoT：专用端口，易被识别和过滤
```

**DNSSEC（DNS Security Extensions）**：

```
功能：
- 防止DNS欺骗
- 验证DNS响应的真实性
- 使用数字签名

工作原理：
1. 权威DNS用私钥签名记录
2. 查询时验证签名
3. 确保数据未被篡改
```

---

## 7. 实用命令

### 7.1 Windows DNS命令

```bash
# 查看DNS缓存
ipconfig /displaydns

# 清除DNS缓存
ipconfig /flushdns

# 查看DNS配置
ipconfig /all

# 使用nslookup查询
nslookup www.baidu.com
nslookup www.baidu.com 8.8.8.8
```

### 7.2 Linux/Mac DNS命令

```bash
# 使用dig查询（推荐）
dig www.baidu.com
dig www.baidu.com @8.8.8.8
dig www.baidu.com MX
dig www.baidu.com +trace  # 追踪完整查询过程

# 使用host命令
host www.baidu.com
host -t MX baidu.com

# 使用nslookup
nslookup www.baidu.com

# 清除DNS缓存（Mac）
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder

# 查看DNS配置
cat /etc/resolv.conf
```

---

## 8. 今日练习

### 练习1：DNS查询分析

使用我们的工具查询以下域名，分析结果：

```
1. www.baidu.com（A记录）
2. baidu.com（MX记录）
3. www.taobao.com（CNAME记录）
4. qq.com（NS记录）
```

### 练习2：DNS性能对比

对比不同DNS服务器的查询速度：

```
测试域名：www.google.com
测试DNS：
- 本地DNS
- 8.8.8.8（Google）
- 223.5.5.5（阿里）
- 1.1.1.1（Cloudflare）

记录每个DNS的响应时间
```

### 练习3：DNS追踪

手动追踪一个域名的完整解析过程：

```
域名：www.example.com

步骤：
1. 查询根DNS服务器
2. 查询.com顶级域服务器
3. 查询example.com权威服务器
4. 记录每一步的结果
```

### 练习4：DNS记录配置

如果你有自己的域名，尝试配置：

```
1. A记录：指向你的服务器
2. CNAME记录：创建子域名别名
3. MX记录：配置邮件服务器
4. TXT记录：添加网站验证信息
```

---

## 9. 常见问题

**Q1：为什么有时候改了DNS记录，但访问还是旧的IP？**

A：这是因为DNS缓存。解决方法：
- 清除本地DNS缓存
- 等待TTL过期
- 设置较短的TTL值（修改前）

**Q2：什么时候用A记录，什么时候用CNAME？**

A：
- A记录：直接指向IP，速度快
- CNAME：指向另一个域名，灵活但多一次查询
- 根域名（example.com）只能用A记录
- 子域名（www.example.com）两者都可以

**Q3：如何选择DNS服务器？**

A：考虑因素：
- 速度：选择离你近的服务器
- 稳定性：选择大厂的公共DNS
- 隐私：避免记录查询日志的DNS
- 功能：是否支持DoH、广告过滤等

**Q4：DNS查询走TCP还是UDP？**

A：
- 通常走UDP（端口53）
- 响应超过512字节时走TCP
- DNSSEC、区域传输走TCP
- DoH走TCP（HTTPS）

---

## 10. 总结

今天我们学习了：

### 核心知识点

1. **DNS基础**：
   - DNS是域名与IP地址的映射系统
   - 分布式、层级化架构

2. **DNS架构**：
   - 根域、顶级域、二级域、子域
   - 根服务器、TLD服务器、权威服务器、本地服务器

3. **查询过程**：
   - 递归查询 vs 迭代查询
   - 多级缓存机制
   - TTL机制

4. **记录类型**：
   - A/AAAA：地址记录
   - CNAME：别名记录
   - MX：邮件服务器
   - NS：域名服务器
   - TXT：文本信息

5. **DNS安全**：
   - DNS劫持、DNS污染
   - DoH、DoT、DNSSEC

### 重点回顾

```
DNS = 互联网的"通讯录"
作用 = 域名 → IP地址

架构：分布式 + 层级化
查询：缓存优先 + 递归查询
安全：加密 + 验证
```

### 明天预告

**第13天：DHCP协议与网络配置**

内容预览：
- DHCP工作原理
- IP地址自动分配
- DHCP四次握手过程
- 实战：DHCP服务器搭建

---

**继续加油！🚀**

今天我们深入学习了DNS协议，理解了互联网域名解析的完整流程。DNS是互联网的基础设施，几乎所有的网络访问都离不开它。掌握DNS不仅能帮你理解网络工作原理，还能在遇到网络问题时快速定位和解决。
