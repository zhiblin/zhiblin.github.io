---
title: 10天：MAC地址和ARP协议
date: 2025-11-10 15:21:37
categories: Network 
tags: Network, Book, Python
---
# 第10天：MAC地址和ARP协议

> "IP地址是逻辑地址，MAC地址是物理地址。"

## 📚 今日目标

- 理解MAC地址的概念和格式
- 掌握MAC地址和IP地址的区别
- 理解ARP协议的工作原理
- 学会查看和管理ARP缓存
- 编写ARP扫描工具

## 🎯 什么是MAC地址？

### 生活中的例子

```
MAC地址 = 身份证号

身份证号：终身不变，唯一标识一个人
MAC地址：出厂固化，唯一标识一块网卡

IP地址 = 家庭住址

家庭住址：可以搬家更换
IP地址：可以更换网络而改变
```

**MAC地址**（Media Access Control Address，媒体访问控制地址）是网卡（网络接口卡）的物理地址，用于在数据链路层标识网络设备。

### MAC地址的特点

1. **全球唯一**：理论上世界上没有两块网卡有相同的MAC地址
2. **固化在硬件**：出厂时写入ROM，一般不可更改
3. **48位（6字节）**：通常用12位十六进制数表示
4. **数据链路层**：工作在OSI模型的第2层

## 📊 MAC地址格式

### 标准格式

```
常见表示方法：

1. 冒号分隔（Linux/Unix）
   AA:BB:CC:DD:EE:FF

2. 横线分隔（Windows）
   AA-BB-CC-DD-EE-FF

3. 点分隔（Cisco）
   AABB.CCDD.EEFF

4. 连续（无分隔符）
   AABBCCDDEEFF

都是同一个MAC地址！
```

### MAC地址结构

```
MAC地址分为两部分：

┌─────────────────┬─────────────────┐
│  OUI (24位)     │  设备标识(24位) │
│  厂商标识       │  序列号         │
└─────────────────┴─────────────────┘
  AA:BB:CC         DD:EE:FF

OUI（Organizationally Unique Identifier）：
- 前3个字节（24位）
- IEEE分配给厂商的唯一标识
- 例如：
  - 00:50:56 → VMware虚拟网卡
  - 00:1C:42 → Parallels虚拟网卡
  - DC:A6:32 → Raspberry Pi
  - 3C:A9:F4 → 小米设备

设备标识：
- 后3个字节（24位）
- 厂商自行分配的序列号
- 理论上可产生 2^24 = 16,777,216 个地址
```

### 特殊MAC地址

```
1. 广播MAC地址
   FF:FF:FF:FF:FF:FF
   用于向局域网内所有设备发送数据

2. 组播MAC地址
   第一个字节的最低位为1
   例如：01:00:5E:XX:XX:XX（IPv4组播）

3. 单播MAC地址
   第一个字节的最低位为0
   普通的设备地址
```

## 🔍 MAC地址 vs IP地址

### 对比表

| 特性 | MAC地址 | IP地址 |
|------|---------|--------|
| 层次 | 数据链路层（第2层） | 网络层（第3层） |
| 长度 | 48位（6字节） | 32位（IPv4）或128位（IPv6） |
| 表示 | 十六进制 | 十进制（点分） |
| 分配 | 厂商分配，固化在硬件 | 网络管理员或DHCP分配 |
| 唯一性 | 全球唯一 | 网络内唯一 |
| 可变性 | 理论上不可变 | 可以更换 |
| 作用范围 | 局域网内 | 可跨越多个网络 |
| 路由 | 不能路由 | 可以路由 |

### 为什么需要两个地址？

```
类比：快递系统

MAC地址 = 楼层和房间号
- 在一栋楼内找到具体房间
- 只在本楼有效
- 固定不变

IP地址 = 城市和街道地址
- 跨城市寻找目标
- 可以搬家（更换网络）
- 可变

过程：
1. 快递先通过城市地址找到目标城市（IP路由）
2. 到达后通过楼层房间号找到具体位置（MAC寻址）
```

### 数据传输过程

```
北京的电脑A → 上海的电脑B

1. 应用层：生成数据
2. 传输层：添加端口号
3. 网络层：添加IP地址
   源IP：北京A的IP
   目标IP：上海B的IP
4. 数据链路层：添加MAC地址
   源MAC：当前路由器的MAC
   目标MAC：下一跳路由器的MAC

重点：
- IP地址从头到尾不变（北京A → 上海B）
- MAC地址每一跳都会变化（每段链路使用不同的MAC）
```

## 📡 ARP协议

### 什么是ARP？

**ARP**（Address Resolution Protocol，地址解析协议）用于将IP地址转换为MAC地址。

```
问题：我知道目标的IP地址，但要发送数据需要MAC地址

ARP的作用：
IP地址（192.168.1.100） → ARP查询 → MAC地址（AA:BB:CC:DD:EE:FF）
```

### ARP工作原理

#### 场景：主机A（192.168.1.10）要发送数据给主机B（192.168.1.20）

```
步骤1：检查ARP缓存
主机A首先查看本地ARP缓存表
- 如果有192.168.1.20的MAC地址 → 直接使用
- 如果没有 → 发送ARP请求

步骤2：发送ARP请求（广播）
主机A发送广播包：
┌─────────────────────────────────────┐
│ 谁是192.168.1.20？                  │
│ 请告诉192.168.1.10                  │
│ 我的MAC是AA:AA:AA:AA:AA:AA          │
└─────────────────────────────────────┘
目标MAC：FF:FF:FF:FF:FF:FF（广播）

步骤3：所有主机接收
局域网内所有主机都收到这个广播包
- 主机B：这是问我的！
- 其他主机：不是问我的，丢弃

步骤4：主机B发送ARP响应（单播）
主机B回复：
┌─────────────────────────────────────┐
│ 我是192.168.1.20                    │
│ 我的MAC是BB:BB:BB:BB:BB:BB          │
└─────────────────────────────────────┘
目标MAC：AA:AA:AA:AA:AA:AA（单播给A）

步骤5：主机A更新ARP缓存
主机A记录：
192.168.1.20 → BB:BB:BB:BB:BB:BB

步骤6：开始通信
现在主机A知道了主机B的MAC地址，可以发送数据了
```

### ARP报文格式

```
ARP请求/响应报文：

┌────────────────────────────────────┐
│ 硬件类型（2字节）                  │ 1 = 以太网
├────────────────────────────────────┤
│ 协议类型（2字节）                  │ 0x0800 = IP
├────────────────────────────────────┤
│ 硬件地址长度（1字节）              │ 6 = MAC地址6字节
├────────────────────────────────────┤
│ 协议地址长度（1字节）              │ 4 = IP地址4字节
├────────────────────────────────────┤
│ 操作码（2字节）                    │ 1=请求，2=响应
├────────────────────────────────────┤
│ 发送方MAC地址（6字节）             │
├────────────────────────────────────┤
│ 发送方IP地址（4字节）              │
├────────────────────────────────────┤
│ 目标MAC地址（6字节）               │ 请求时为全0
├────────────────────────────────────┤
│ 目标IP地址（4字节）                │
└────────────────────────────────────┘
```

### ARP缓存

为了提高效率，系统会缓存ARP解析的结果。

```
ARP缓存表示例：

IP地址           MAC地址              类型     过期时间
192.168.1.1     AA:BB:CC:DD:EE:FF    动态     120秒
192.168.1.20    11:22:33:44:55:66    动态     120秒
192.168.1.100   FF:EE:DD:CC:BB:AA    静态     永久

类型：
- 动态：通过ARP协议自动学习，有过期时间
- 静态：手动配置，永久有效
```

### 查看ARP缓存

```bash
# Windows
arp -a

# Linux/Mac
arp -n

# 查看特定IP的MAC
arp -n 192.168.1.1

# 清空ARP缓存（需要管理员权限）
# Windows
arp -d

# Linux
sudo ip -s -s neigh flush all

# Mac
sudo arp -d -a
```

## 🔧 实战项目：ARP扫描工具

让我们编写一个ARP扫描工具，用于发现局域网内的所有设备！

### 代码实现

```python
# day10_arp_scanner.py
# 功能：ARP扫描工具

import socket
import struct
import time
from concurrent.futures import ThreadPoolExecutor

class ARPScanner:
    """ARP扫描器"""

    def __init__(self):
        # OUI数据库（部分常见厂商）
        self.oui_database = {
            '00:50:56': 'VMware',
            '00:0C:29': 'VMware',
            '00:1C:42': 'Parallels',
            '08:00:27': 'VirtualBox',
            'DC:A6:32': 'Raspberry Pi',
            '3C:A9:F4': '小米',
            '28:6C:07': '小米',
            'F8:A4:5F': '华为',
            '00:E0:4C': '华为',
            'AC:DE:48': 'Apple',
            '00:03:93': 'Apple',
        }

    def get_mac_vendor(self, mac):
        """
        根据MAC地址获取厂商信息

        参数：
            mac: MAC地址字符串

        返回：
            厂商名称
        """
        # 提取OUI（前3个字节）
        oui = ':'.join(mac.split(':')[:3]).upper()

        return self.oui_database.get(oui, '未知厂商')

    def get_local_ip(self):
        """
        获取本机IP地址
        """
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
            sock.connect(("8.8.8.8", 80))
            local_ip = sock.getsockname()[0]
            sock.close()
            return local_ip
        except:
            return None

    def get_mac_address(self, ip):
        """
        通过ARP获取指定IP的MAC地址

        参数：
            ip: 目标IP地址

        返回：
            MAC地址字符串，如果失败返回None

        工作原理：
            发送ARP请求，等待响应
        """
        try:
            # 在Windows上使用arp命令
            import subprocess
            import platform

            system = platform.system()

            if system == 'Windows':
                # Windows: arp -a IP
                result = subprocess.check_output(
                    f'arp -a {ip}',
                    shell=True,
                    stderr=subprocess.DEVNULL
                ).decode('gbk', errors='ignore')

                # 解析输出
                for line in result.split('\n'):
                    if ip in line:
                        parts = line.split()
                        if len(parts) >= 2:
                            mac = parts[1].replace('-', ':')
                            return mac

            else:
                # Linux/Mac: arp -n IP
                result = subprocess.check_output(
                    f'arp -n {ip}',
                    shell=True,
                    stderr=subprocess.DEVNULL
                ).decode('utf-8', errors='ignore')

                for line in result.split('\n'):
                    if ip in line:
                        parts = line.split()
                        if len(parts) >= 3:
                            return parts[2]

        except:
            pass

        return None

    def ping_and_get_mac(self, ip):
        """
        先ping一下IP（触发ARP），然后获取MAC地址

        参数：
            ip: 目标IP地址

        返回：
            (ip, mac, vendor) 元组，失败返回None
        """
        try:
            import subprocess
            import platform

            system = platform.system()

            # 先ping一下，触发ARP
            if system == 'Windows':
                ping_cmd = f'ping -n 1 -w 500 {ip}'
            else:
                ping_cmd = f'ping -c 1 -W 1 {ip}'

            subprocess.run(
                ping_cmd,
                shell=True,
                stdout=subprocess.DEVNULL,
                stderr=subprocess.DEVNULL,
                timeout=2
            )

            # 获取MAC地址
            mac = self.get_mac_address(ip)

            if mac and mac != '(incomplete)' and 'ff:ff:ff:ff:ff:ff' not in mac.lower():
                vendor = self.get_mac_vendor(mac)
                return (ip, mac, vendor)

        except:
            pass

        return None

    def scan_network(self, network, max_workers=50):
        """
        扫描整个网络

        参数：
            network: 网络地址（如 "192.168.1.0/24"）
            max_workers: 最大线程数

        返回：
            发现的设备列表
        """
        import ipaddress

        print(f"🔍 开始扫描网络: {network}")
        print(f"⏳ 请稍候...\n")

        # 解析网络
        net = ipaddress.ip_network(network, strict=False)
        total = net.num_addresses - 2

        devices = []
        scanned = 0

        # 使用线程池并发扫描
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            futures = {
                executor.submit(self.ping_and_get_mac, str(ip)): ip
                for ip in net.hosts()
            }

            from concurrent.futures import as_completed

            for future in as_completed(futures):
                scanned += 1
                result = future.result()

                if result:
                    devices.append(result)
                    ip, mac, vendor = result
                    print(f"✅ {ip:<15} {mac:<17} {vendor}")

                # 显示进度
                if scanned % 20 == 0:
                    progress = (scanned / total) * 100
                    print(f"   进度: {progress:.1f}% ({scanned}/{total})", end='\r')

        return devices

    def display_results(self, devices):
        """
        显示扫描结果

        参数：
            devices: 设备列表
        """
        print("\n" + "=" * 70)
        print(f"扫描完成！共发现 {len(devices)} 台设备")
        print("=" * 70)

        if devices:
            print(f"\n{'IP地址':<15} {'MAC地址':<17} {'厂商'}")
            print("-" * 70)

            for ip, mac, vendor in sorted(devices):
                print(f"{ip:<15} {mac:<17} {vendor}")

        print("\n" + "=" * 70)

def view_arp_cache():
    """
    查看系统ARP缓存
    """
    import subprocess
    import platform

    print("=" * 70)
    print("系统ARP缓存表".center(70))
    print("=" * 70)
    print()

    try:
        system = platform.system()

        if system == 'Windows':
            result = subprocess.check_output('arp -a', shell=True).decode('gbk', errors='ignore')
        else:
            result = subprocess.check_output('arp -n', shell=True).decode('utf-8', errors='ignore')

        print(result)

    except Exception as e:
        print(f"❌ 无法获取ARP缓存: {e}")

    print("=" * 70)

def main():
    """
    主函数
    """
    scanner = ARPScanner()

    print("=" * 70)
    print("ARP扫描工具".center(70))
    print("=" * 70)
    print()

    # 获取本机IP
    local_ip = scanner.get_local_ip()
    if local_ip:
        print(f"💻 本机IP: {local_ip}")

        # 推断网络
        ip_parts = local_ip.split('.')
        network = f"{ip_parts[0]}.{ip_parts[1]}.{ip_parts[2]}.0/24"
        print(f"🌐 推断网络: {network}")
    else:
        network = "192.168.1.0/24"
        print(f"🌐 使用默认网络: {network}")

    print()
    print("请选择功能:")
    print("1. 扫描局域网设备（推荐）")
    print("2. 扫描自定义网络")
    print("3. 查看ARP缓存表")
    print("0. 退出")

    choice = input("\n请输入选项: ").strip()

    if choice == '1':
        devices = scanner.scan_network(network)
        scanner.display_results(devices)

    elif choice == '2':
        custom_network = input("\n请输入网络地址（如192.168.1.0/24）: ").strip()
        if custom_network:
            devices = scanner.scan_network(custom_network)
            scanner.display_results(devices)

    elif choice == '3':
        view_arp_cache()

    elif choice == '0':
        print("\n再见！")

if __name__ == "__main__":
    main()
```

### 运行程序

```bash
python code/day10/day10_arp_scanner.py
```

## 🔍 代码详解

### 1. ARP工作流程

```python
# 1. Ping触发ARP
subprocess.run(f'ping -c 1 {ip}', ...)

# 2. 系统自动发送ARP请求
# 3. 目标主机响应
# 4. 系统缓存ARP表项

# 5. 读取ARP缓存
subprocess.check_output('arp -n', ...)
```

### 2. 并发扫描提速

```python
# 使用线程池并发扫描多个IP
with ThreadPoolExecutor(max_workers=50) as executor:
    # 50个线程同时工作
    futures = {executor.submit(ping_and_get_mac, ip): ip
               for ip in net.hosts()}
```

### 3. MAC地址厂商识别

```python
# 通过MAC地址前3个字节（OUI）识别厂商
oui = mac[:8]  # 如 "00:50:56"
vendor = oui_database.get(oui, '未知厂商')
```

## 🎓 知识小结

今天学习了：

1. ✅ MAC地址是网卡的物理地址，全球唯一
2. ✅ MAC地址工作在数据链路层
3. ✅ ARP协议用于IP地址到MAC地址的转换
4. ✅ ARP使用广播请求，单播响应
5. ✅ 系统会缓存ARP解析结果

### 重要概念

| 概念 | 说明 |
|------|------|
| MAC地址 | 48位物理地址，固化在网卡 |
| OUI | MAC地址前24位，厂商标识 |
| ARP | 地址解析协议，IP→MAC |
| ARP缓存 | 系统缓存的IP-MAC映射表 |
| 广播MAC | FF:FF:FF:FF:FF:FF |

## 💪 练习题

### 练习1：查看本机信息
1. 查看本机的MAC地址
2. 查看本机的ARP缓存
3. 找出路由器的MAC地址

### 练习2：理解ARP
1. 画出ARP请求和响应的完整过程
2. 解释为什么ARP请求使用广播
3. ARP缓存的作用是什么？

### 练习3：实践操作
1. 清空ARP缓存
2. Ping一个局域网内的设备
3. 再次查看ARP缓存，观察变化

### 练习4：安全思考
1. 什么是ARP欺骗（ARP Spoofing）？
2. 如何防范ARP攻击？
3. 为什么ARP协议不安全？

---

💡 **今日金句**：MAC地址是网络世界的身份证！

👉 [上一天：子网划分实战](day09.md) | [返回目录](../../README.md) | [下一天：端口和Socket深入](day11.md)
