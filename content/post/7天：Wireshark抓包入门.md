---
title: 7天：Wireshark抓包入门
date: 2025-11-10 14:49:32
categories: Network 
tags: [Network, Book, Python]
---
# 第7天：Wireshark抓包入门

> "看见网络数据包的流动，就像看见了互联网的血液循环。"

## 📚 今日目标

- 学会安装和使用Wireshark
- 掌握基本的抓包和过滤技巧
- 能够分析HTTP请求过程
- 理解各层协议在实际数据包中的呈现

## 🎯 什么是Wireshark？

**Wireshark**是世界上最流行的网络协议分析工具（也叫抓包工具）。

### 生活中的例子

```
Wireshark = 网络显微镜

就像医生用显微镜观察血液：
- 看到红细胞、白细胞
- 分析血液成分
- 诊断健康问题

Wireshark让你看到网络数据包：
- 看到TCP、HTTP、DNS包
- 分析数据内容
- 诊断网络问题
```

### Wire

shark的特点

- 🔍 **可视化**：图形界面，易于使用
- 🎨 **彩色显示**：不同协议不同颜色
- 🔎 **强大过滤**：快速找到需要的包
- 📊 **统计分析**：流量统计、协议分布
- 💾 **保存分析**：可以保存抓包文件
- 🌐 **跨平台**：支持Windows、Mac、Linux

## 🔧 安装Wireshark

### 下载

访问官网：https://www.wireshark.org/

### 各平台安装

**Windows**：
```
1. 下载.exe安装包
2. 双击安装
3. 安装过程中选择安装WinPcap或Npcap（必需）
4. 完成后重启计算机
```

**Mac**：
```
1. 下载.dmg文件
2. 拖动到Applications文件夹
3. 首次运行可能需要在系统偏好设置中允许
```

**Linux（Ubuntu/Debian）**：
```bash
sudo apt update
sudo apt install wireshark

# 添加当前用户到wireshark组
sudo usermod -aG wireshark $USER

# 重新登录生效
```

## 📖 Wireshark界面介绍

### 主界面布局

```
┌─────────────────────────────────────────────────────────┐
│  菜单栏: 文件 编辑 查看 捕获 分析 统计 帮助         │
├─────────────────────────────────────────────────────────┤
│  工具栏: [开始] [停止] [重启] [保存] ...                │
├─────────────────────────────────────────────────────────┤
│  过滤器: [___________________________] [应用]           │
├─────────────────────────────────────────────────────────┤
│  ① 数据包列表区（Packet List）                          │
│  No.  Time     Source      Destination  Protocol Info   │
│  1    0.000    192.168.1.1 192.168.1.2  TCP      SYN    │
│  2    0.001    192.168.1.2 192.168.1.1  TCP      ACK    │
├─────────────────────────────────────────────────────────┤
│  ② 数据包详情区（Packet Details）                       │
│  ▶ Frame 1: 74 bytes on wire                            │
│  ▶ Ethernet II, Src: aa:bb:cc:dd:ee:ff                  │
│  ▶ Internet Protocol Version 4                          │
│  ▶ Transmission Control Protocol                        │
├─────────────────────────────────────────────────────────┤
│  ③ 十六进制数据区（Packet Bytes）                       │
│  0000  00 0c 29 6f 6c 92 00 0c 29 5b 3f e8 08 00 45 00 │
│  0010  00 3c f9 68 40 00 40 06 38 57 c0 a8 01 64 c0 a8 │
└─────────────────────────────────────────────────────────┘
```

### 三大区域

1. **数据包列表区**：显示捕获的所有包
2. **数据包详情区**：显示选中包的协议层次结构
3. **十六进制数据区**：显示原始字节数据

## 🔍 开始第一次抓包

### 步骤1：选择网络接口

1. 启动Wireshark
2. 在主界面看到网络接口列表
3. 选择正在使用的接口（有流量波形的）
   - WiFi：通常是 `en0`（Mac）或 `wlan0`（Linux）
   - 有线：通常是 `eth0` 或 `en0`

### 步骤2：开始捕获

点击接口名称或点击工具栏的"开始捕获"按钮（鲨鱼鳍图标）

### 步骤3：生成流量

在浏览器中访问：http://www.example.com

### 步骤4：停止捕获

点击工具栏的"停止捕获"按钮（红色方块）

## 🎨 认识颜色代码

Wireshark用颜色区分不同类型的包：

| 颜色 | 协议/类型 | 含义 |
|------|-----------|------|
| 浅紫色 | TCP | TCP协议 |
| 浅蓝色 | UDP | UDP协议 |
| 黑色 | 有问题的包 | 校验和错误、重传等 |
| 浅绿色 | HTTP | HTTP协议 |
| 浅黄色 | Windows特定 | NetBIOS、SMB等 |
| 深灰色 | TCP重传 | 数据包重传 |

## 🔎 过滤器使用

### 显示过滤器（Display Filter）

**在工具栏的过滤器框中输入**，用于过滤已捕获的包。

#### 常用过滤器

```
# 只显示HTTP
http

# 只显示TCP
tcp

# 只显示特定IP
ip.addr == 192.168.1.100

# 只显示源IP
ip.src == 192.168.1.100

# 只显示目标IP
ip.dst == 8.8.8.8

# 只显示特定端口
tcp.port == 80

# 组合条件（AND）
ip.addr == 192.168.1.100 && tcp.port == 80

# 组合条件（OR）
http || dns

# 排除（NOT）
!arp

# HTTP GET请求
http.request.method == "GET"

# DNS查询
dns.flags.response == 0
```

### 捕获过滤器（Capture Filter）

**在开始捕获前设置**，只捕获符合条件的包。

```
# 只捕获80端口
port 80

# 只捕获特定主机
host 192.168.1.100

# 只捕获TCP
tcp

# 组合条件
host 192.168.1.100 and port 80
```

## 🔬 实战：分析HTTP请求

让我们实际抓取并分析一个HTTP请求！

### 步骤1：设置过滤器

在开始前，设置显示过滤器：`http`

### 步骤2：开始捕获

点击开始按钮

### 步骤3：访问网站

在浏览器访问：http://www.example.com
（注意：必须是http，不是https）

### 步骤4：停止并分析

停止捕获，你会看到HTTP请求和响应。

### 分析HTTP GET请求

找到一个HTTP GET请求包，点击它：

```
在数据包详情区，你会看到：

▶ Frame 123: 502 bytes on wire
  ├─ 物理层信息

▶ Ethernet II
  ├─ Destination: xx:xx:xx:xx:xx:xx
  ├─ Source: yy:yy:yy:yy:yy:yy
  └─ Type: IPv4 (0x0800)
    # 数据链路层：MAC地址

▶ Internet Protocol Version 4
  ├─ Source: 192.168.1.100
  ├─ Destination: 93.184.216.34
  ├─ Protocol: TCP (6)
  └─ TTL: 64
    # 网际层：IP地址

▶ Transmission Control Protocol
  ├─ Source Port: 54321
  ├─ Destination Port: 80
  ├─ Sequence number: 1
  └─ Flags: 0x018 (PSH, ACK)
    # 传输层：端口号

▶ Hypertext Transfer Protocol
  ├─ GET / HTTP/1.1\r\n
  ├─ Host: www.example.com\r\n
  ├─ User-Agent: Mozilla/5.0...\r\n
  ├─ Accept: text/html...\r\n
  └─ \r\n
    # 应用层：HTTP协议
```

**看到了什么？**
- ✅ 完整的TCP/IP四层结构！
- ✅ 每一层的头部信息
- ✅ HTTP请求的完整内容

### 分析HTTP响应

找到对应的HTTP响应包：

```
▶ Hypertext Transfer Protocol
  ├─ HTTP/1.1 200 OK\r\n
  ├─ Content-Type: text/html\r\n
  ├─ Content-Length: 1256\r\n
  ├─ Server: Apache/2.4.41\r\n
  └─ \r\n
  └─ [HTML content]
```

## 💡 实用技巧

### 1. 跟踪TCP流

右键点击一个TCP包 → Follow → TCP Stream

**作用**：查看完整的TCP会话内容（所有来回的数据）

### 2. 统计信息

菜单：Statistics → Protocol Hierarchy

**作用**：查看各种协议的流量占比

### 3. 导出对象

菜单：File → Export Objects → HTTP

**作用**：提取HTTP传输的文件（图片、HTML等）

### 4. 时间列显示

右键点击Time列 → Display Format → 选择格式

**推荐**：选择"Seconds Since Previous Displayed Packet"查看包间隔

### 5. 查找包

Ctrl+F（Windows/Linux）或 Cmd+F（Mac）

可以搜索：
- 显示过滤器
- 十六进制值
- 字符串

## 🎓 知识小结

### 今天学到了什么？

1. ✅ Wireshark是最强大的网络协议分析工具
2. ✅ 可以看到每个数据包的完整结构
3. ✅ 过滤器可以快速定位需要的包
4. ✅ 可以分析HTTP、TCP、IP等各层协议
5. ✅ 实际验证了前几天学习的协议知识

### 重要概念

| 概念 | 含义 | 用途 |
|------|------|------|
| 抓包 | 捕获网络数据包 | 分析网络行为 |
| 显示过滤器 | 过滤已捕获的包 | 查看特定包 |
| 捕获过滤器 | 只捕获特定的包 | 减少文件大小 |
| 跟踪流 | 查看完整会话 | 理解通信过程 |

### 常用过滤器速查

```bash
# 协议过滤
http, tcp, udp, dns, icmp

# IP过滤
ip.addr == 192.168.1.1
ip.src == 192.168.1.1
ip.dst == 8.8.8.8

# 端口过滤
tcp.port == 80
tcp.srcport == 80
tcp.dstport == 443

# HTTP过滤
http.request
http.response
http.request.method == "POST"

# 逻辑运算
&& (AND)
|| (OR)
! (NOT)
```

## 💪 练习题

### 练习1：基础抓包
1. 抓取访问baidu.com的数据包
2. 找到DNS查询包
3. 找到TCP三次握手的三个包
4. 找到HTTP请求和响应

### 练习2：过滤器练习
使用过滤器完成以下任务：
1. 只显示你的电脑发送的包
2. 只显示HTTP POST请求
3. 显示所有到8.8.8.8的包
4. 显示DNS查询但不显示响应

### 练习3：协议分析
抓取一个完整的HTTP请求，分析：
1. TCP三次握手过程
2. HTTP请求头部内容
3. HTTP响应状态码
4. TCP四次挥手过程

### 练习4：进阶任务
1. 导出一个网页传输的图片
2. 统计5分钟内各协议的流量占比
3. 找出延迟最大的数据包
4. 分析一个TCP重传的包

## 📚 扩展阅读

### 为什么HTTPS不能抓包？

HTTPS是加密的：
```
HTTP:  明文传输，Wireshark可以看到内容
HTTPS: 加密传输，Wireshark只能看到加密数据

要抓取HTTPS内容需要：
1. 安装证书
2. 配置浏览器导出密钥
3. 在Wireshark中配置SSL密钥日志
```

### Wireshark的替代工具

- **tcpdump**：命令行抓包工具
- **Charles**：专注HTTP/HTTPS的工具
- **Fiddler**：Windows上的HTTP调试工具
- **mitmproxy**：命令行的中间人代理

### 实际应用场景

1. **Web开发调试**：查看HTTP请求和响应
2. **网络故障排查**：分析丢包、延迟
3. **安全分析**：检测异常流量
4. **学习协议**：深入理解网络协议
5. **性能优化**：分析网络瓶颈

## 🎯 下一步：第一周总结

**明天我们将总结第一周的所有内容！**

回顾：
- 网络基础概念
- OSI和TCP/IP模型
- ping和traceroute命令
- Wireshark抓包分析

准备好进行第一周的综合测验了吗？

---

💡 **今日金句**：Wireshark让不可见的网络数据包变得可见！

👉 [上一天：traceroute详解](day06.md) | [返回目录](../../README.md) | [下一天：第一周总结](week_summary.md)
