---
title: 19天：HTTP协议基础
date: 2025-11-10 15:24:39
categories: Network 
tags: [Network, Book, Python]
---
# 第19天：HTTP协议基础

## 今日目标

- 理解HTTP协议的工作原理
- 掌握HTTP请求和响应格式
- 学习HTTP方法和状态码
- 了解HTTP头部字段
- 理解Cookie和Session机制
- 学习HTTP版本演进
- 实现简单的HTTP服务器

---

## 1. HTTP协议概述

### 1.1 什么是HTTP？

**HTTP（HyperText Transfer Protocol）**：

```
HTTP = 超文本传输协议

生活类比：
HTTP就像邮寄系统：
1. 你写信（HTTP请求）
2. 邮递员送信（网络传输）
3. 收信人回复（HTTP响应）
4. 邮递员送回回信（网络传输）

特点：
- 文本协议（可读性强）
- 无状态（每次请求独立）
- 客户端-服务器模型
- 基于TCP（可靠传输）
```

**HTTP的位置**：

```
应用层协议栈：

┌────────────────────────────┐
│  应用程序（浏览器、APP）    │
├────────────────────────────┤
│  HTTP（应用层协议）         │
├────────────────────────────┤
│  TCP（传输层协议）          │
├────────────────────────────┤
│  IP（网络层协议）           │
├────────────────────────────┤
│  物理层                     │
└────────────────────────────┘
```

### 1.2 HTTP工作流程

**基本流程**：

```
客户端（浏览器）                    服务器

    1. DNS解析
       www.example.com → 93.184.216.34

    2. 建立TCP连接
       ─────────────────────────>
       三次握手
       <─────────────────────────

    3. 发送HTTP请求
       GET /index.html HTTP/1.1
       Host: www.example.com
       ─────────────────────────>

    4. 服务器处理请求
                                  - 解析请求
                                  - 查找资源
                                  - 生成响应

    5. 返回HTTP响应
       <─────────────────────────
       HTTP/1.1 200 OK
       Content-Type: text/html

       <html>...</html>

    6. 关闭连接（HTTP/1.0）
       或保持连接（HTTP/1.1）
```

**详细步骤**：

```
1. 用户输入URL：http://www.example.com/index.html

2. 浏览器解析URL：
   - 协议：http
   - 主机：www.example.com
   - 端口：80（默认）
   - 路径：/index.html

3. DNS查询：
   - 查询www.example.com的IP地址
   - 得到：93.184.216.34

4. 建立TCP连接：
   - 连接到 93.184.216.34:80
   - 三次握手

5. 发送HTTP请求：
   - 请求行：GET /index.html HTTP/1.1
   - 请求头：Host, User-Agent等
   - 空行
   - 请求体（如果有）

6. 服务器处理：
   - 解析请求
   - 找到/index.html文件
   - 读取文件内容

7. 发送HTTP响应：
   - 状态行：HTTP/1.1 200 OK
   - 响应头：Content-Type, Content-Length等
   - 空行
   - 响应体（HTML内容）

8. 浏览器渲染：
   - 解析HTML
   - 加载CSS、JavaScript、图片等
   - 渲染页面
```

### 1.3 HTTP特点

**无状态（Stateless）**：

```
特点：
服务器不会记住之前的请求

生活类比：
你每次去银行取钱，柜员都不认识你
每次都要重新验证身份

问题：
- 无法识别用户
- 无法保持登录状态

解决方案：
- Cookie
- Session
- Token
```

**明文传输**：

```
优点：
✅ 易于调试
✅ 中间设备可以缓存
✅ 简单直观

缺点：
❌ 不安全（可被窃听）
❌ 敏感信息暴露

解决方案：
使用HTTPS（HTTP + TLS/SSL）
```

**请求-响应模式**：

```
特点：
- 客户端主动发起请求
- 服务器被动响应
- 一个请求对应一个响应

限制：
- 服务器无法主动推送
- 实时性较差

解决方案：
- 长轮询（Long Polling）
- WebSocket
- Server-Sent Events (SSE)
```

---

## 2. HTTP请求格式

### 2.1 请求格式概览

**完整的HTTP请求**：

```
GET /index.html HTTP/1.1              ← 请求行
Host: www.example.com                 ← 请求头
User-Agent: Mozilla/5.0               ← 请求头
Accept: text/html                     ← 请求头
Connection: keep-alive                ← 请求头
                                      ← 空行（CRLF）
[请求体，如果有的话]                   ← 请求体
```

**结构**：

```
┌────────────────────────────────┐
│  请求行（Request Line）         │
├────────────────────────────────┤
│  请求头（Request Headers）      │
│  ...                           │
├────────────────────────────────┤
│  空行（CRLF）                   │
├────────────────────────────────┤
│  请求体（Request Body）         │
│  （可选，GET请求通常没有）       │
└────────────────────────────────┘
```

### 2.2 请求行

**格式**：

```
方法 URI HTTP版本

示例：
GET /index.html HTTP/1.1
POST /api/login HTTP/1.1
PUT /api/users/123 HTTP/1.1
```

**组成部分**：

```
1. 方法（Method）
   - GET：获取资源
   - POST：提交数据
   - PUT：更新资源
   - DELETE：删除资源
   - HEAD：获取头部
   - OPTIONS：查询支持的方法
   - TRACE：追踪请求
   - CONNECT：建立隧道

2. URI（统一资源标识符）
   - 路径：/index.html
   - 查询参数：?name=Alice&age=20
   - 片段：#section1（不发送到服务器）

3. HTTP版本
   - HTTP/1.0：早期版本
   - HTTP/1.1：当前主流
   - HTTP/2：二进制协议
   - HTTP/3：基于QUIC
```

### 2.3 请求头

**常用请求头**：

```
Host: www.example.com
  - 指定服务器域名（必需）
  - HTTP/1.1要求必须有

User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
  - 客户端信息（浏览器、操作系统）

Accept: text/html,application/json
  - 客户端可接受的内容类型

Accept-Encoding: gzip, deflate, br
  - 客户端支持的压缩格式

Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
  - 客户端首选语言

Connection: keep-alive
  - 连接管理
  - keep-alive：保持连接
  - close：关闭连接

Content-Type: application/json
  - 请求体的MIME类型

Content-Length: 123
  - 请求体的长度（字节）

Cookie: session_id=abc123; user=Alice
  - 发送Cookie给服务器

Authorization: Bearer eyJhbGc...
  - 身份认证信息

Referer: https://www.google.com/
  - 来源页面（从哪个页面跳转）

Cache-Control: no-cache
  - 缓存控制

If-Modified-Since: Wed, 21 Oct 2015 07:28:00 GMT
  - 条件请求（只在修改后才返回）
```

### 2.4 请求体

**何时有请求体**：

```
有请求体的方法：
- POST：提交表单、上传文件
- PUT：更新资源
- PATCH：部分更新

没有请求体的方法：
- GET：通过URL传参
- DELETE：通常不需要
- HEAD：只要头部
```

**常见Content-Type**：

```
1. application/x-www-form-urlencoded（表单默认）
   Content-Type: application/x-www-form-urlencoded

   请求体：
   name=Alice&age=20&email=alice@example.com

2. application/json（API常用）
   Content-Type: application/json

   请求体：
   {
       "name": "Alice",
       "age": 20,
       "email": "alice@example.com"
   }

3. multipart/form-data（文件上传）
   Content-Type: multipart/form-data; boundary=----WebKitFormBoundary

   请求体：
   ------WebKitFormBoundary
   Content-Disposition: form-data; name="file"; filename="photo.jpg"
   Content-Type: image/jpeg

   [二进制文件内容]
   ------WebKitFormBoundary--

4. text/plain（纯文本）
   Content-Type: text/plain

   请求体：
   Hello, this is plain text.

5. application/xml（XML数据）
   Content-Type: application/xml

   请求体：
   <?xml version="1.0"?>
   <user>
       <name>Alice</name>
       <age>20</age>
   </user>
```

---

## 3. HTTP响应格式

### 3.1 响应格式概览

**完整的HTTP响应**：

```
HTTP/1.1 200 OK                       ← 状态行
Content-Type: text/html               ← 响应头
Content-Length: 1234                  ← 响应头
Date: Mon, 15 Jan 2024 10:00:00 GMT  ← 响应头
Server: nginx/1.18.0                  ← 响应头
                                      ← 空行（CRLF）
<html>                                ← 响应体
  <body>Hello World</body>
</html>
```

**结构**：

```
┌────────────────────────────────┐
│  状态行（Status Line）          │
├────────────────────────────────┤
│  响应头（Response Headers）     │
│  ...                           │
├────────────────────────────────┤
│  空行（CRLF）                   │
├────────────────────────────────┤
│  响应体（Response Body）        │
└────────────────────────────────┘
```

### 3.2 状态行

**格式**：

```
HTTP版本 状态码 状态描述

示例：
HTTP/1.1 200 OK
HTTP/1.1 404 Not Found
HTTP/1.1 500 Internal Server Error
```

### 3.3 状态码

**状态码分类**：

```
┌─────────┬──────────────────┬────────────────────┐
│  类别   │      范围        │      含义          │
├─────────┼──────────────────┼────────────────────┤
│  1xx    │  100-199         │  信息性响应        │
│  2xx    │  200-299         │  成功              │
│  3xx    │  300-399         │  重定向            │
│  4xx    │  400-499         │  客户端错误        │
│  5xx    │  500-599         │  服务器错误        │
└─────────┴──────────────────┴────────────────────┘
```

**1xx 信息性响应**：

```
100 Continue
  - 客户端应继续发送请求体
  - 用于大文件上传前的确认

101 Switching Protocols
  - 切换协议（如WebSocket升级）
```

**2xx 成功**：

```
200 OK
  - 请求成功
  - 最常见的成功状态码

201 Created
  - 资源已创建
  - 常用于POST请求创建新资源

202 Accepted
  - 请求已接受，但处理未完成
  - 用于异步处理

204 No Content
  - 成功，但无内容返回
  - 常用于DELETE请求

206 Partial Content
  - 部分内容
  - 用于断点续传
```

**3xx 重定向**：

```
301 Moved Permanently
  - 永久重定向
  - 搜索引擎会更新索引
  - 示例：http → https

302 Found（临时重定向）
  - 临时重定向
  - 原URL仍然有效

303 See Other
  - 查看其他位置
  - POST请求后重定向到GET

304 Not Modified
  - 资源未修改
  - 可使用缓存

307 Temporary Redirect
  - 临时重定向（保持请求方法）

308 Permanent Redirect
  - 永久重定向（保持请求方法）
```

**4xx 客户端错误**：

```
400 Bad Request
  - 请求格式错误
  - 参数错误

401 Unauthorized
  - 未认证
  - 需要登录

403 Forbidden
  - 禁止访问
  - 已认证但无权限

404 Not Found
  - 资源不存在
  - 最常见的错误

405 Method Not Allowed
  - 不支持的HTTP方法

408 Request Timeout
  - 请求超时

409 Conflict
  - 资源冲突
  - 如重复创建

413 Payload Too Large
  - 请求体过大

414 URI Too Long
  - URL过长

415 Unsupported Media Type
  - 不支持的Content-Type

429 Too Many Requests
  - 请求过于频繁
  - 限流
```

**5xx 服务器错误**：

```
500 Internal Server Error
  - 服务器内部错误
  - 最常见的服务器错误

501 Not Implemented
  - 功能未实现

502 Bad Gateway
  - 网关错误
  - 上游服务器返回无效响应

503 Service Unavailable
  - 服务不可用
  - 服务器过载或维护

504 Gateway Timeout
  - 网关超时
  - 上游服务器超时

505 HTTP Version Not Supported
  - HTTP版本不支持
```

### 3.4 响应头

**常用响应头**：

```
Content-Type: text/html; charset=utf-8
  - 响应体的MIME类型

Content-Length: 1234
  - 响应体的长度（字节）

Content-Encoding: gzip
  - 响应体的压缩方式

Date: Mon, 15 Jan 2024 10:00:00 GMT
  - 响应时间

Server: nginx/1.18.0
  - 服务器软件信息

Set-Cookie: session_id=abc123; Path=/; HttpOnly
  - 设置Cookie

Location: https://www.example.com/new-url
  - 重定向目标URL

Cache-Control: max-age=3600
  - 缓存控制

Expires: Wed, 21 Oct 2025 07:28:00 GMT
  - 过期时间

ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
  - 资源版本标识

Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
  - 最后修改时间

Access-Control-Allow-Origin: *
  - CORS跨域控制

Connection: keep-alive
  - 连接管理

Transfer-Encoding: chunked
  - 分块传输编码

Vary: Accept-Encoding
  - 缓存变化因素
```

### 3.5 响应体

**常见Content-Type**：

```
text/html        - HTML页面
text/plain       - 纯文本
text/css         - CSS样式表
text/javascript  - JavaScript代码

application/json        - JSON数据
application/xml         - XML数据
application/pdf         - PDF文件
application/zip         - ZIP压缩包
application/octet-stream - 二进制流（下载）

image/jpeg       - JPEG图片
image/png        - PNG图片
image/gif        - GIF图片
image/svg+xml    - SVG矢量图

video/mp4        - MP4视频
audio/mpeg       - MP3音频
```

---

## 4. HTTP方法详解

### 4.1 GET方法

```
作用：
获取资源（只读）

特点：
- 参数在URL中（查询字符串）
- 可缓存
- 可添加书签
- 幂等性（多次请求结果相同）
- 安全性（不修改服务器状态）

示例：
GET /api/users?id=123&name=Alice HTTP/1.1
Host: api.example.com

使用场景：
- 获取网页
- 查询数据
- 下载文件
- API查询
```

### 4.2 POST方法

```
作用：
提交数据（创建资源）

特点：
- 参数在请求体中
- 不可缓存
- 不可添加书签
- 非幂等（多次请求可能产生不同结果）
- 非安全（会修改服务器状态）

示例：
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
    "name": "Alice",
    "email": "alice@example.com"
}

使用场景：
- 提交表单
- 上传文件
- 创建资源
- API提交
```

### 4.3 PUT方法

```
作用：
更新资源（完整更新）

特点：
- 幂等（多次请求结果相同）
- 替换整个资源

示例：
PUT /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
    "id": 123,
    "name": "Alice Updated",
    "email": "alice@example.com"
}

使用场景：
- 完整更新资源
- 替换文件
```

### 4.4 DELETE方法

```
作用：
删除资源

特点：
- 幂等（多次删除同一资源结果相同）

示例：
DELETE /api/users/123 HTTP/1.1
Host: api.example.com

使用场景：
- 删除用户
- 删除文件
- 注销账号
```

### 4.5 PATCH方法

```
作用：
部分更新资源

特点：
- 只更新部分字段
- 比PUT更节省带宽

示例：
PATCH /api/users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
    "name": "Alice Updated"
}

使用场景：
- 只更新某些字段
- 修改配置
```

### 4.6 HEAD方法

```
作用：
获取资源头部信息

特点：
- 不返回响应体
- 用于检查资源是否存在
- 用于获取元数据

示例：
HEAD /api/users/123 HTTP/1.1
Host: api.example.com

使用场景：
- 检查资源是否存在
- 获取文件大小
- 检查最后修改时间
```

### 4.7 OPTIONS方法

```
作用：
查询支持的方法

特点：
- 用于CORS预检请求
- 返回Allow头部

示例：
OPTIONS /api/users HTTP/1.1
Host: api.example.com

响应：
HTTP/1.1 200 OK
Allow: GET, POST, PUT, DELETE, OPTIONS

使用场景：
- CORS预检
- 服务发现
```

### 4.8 方法对比

```
┌────────┬────────┬────────┬────────┬──────────────┐
│  方法  │  幂等  │  安全  │ 可缓存 │    用途      │
├────────┼────────┼────────┼────────┼──────────────┤
│  GET   │   ✅   │   ✅   │   ✅   │  查询        │
│  POST  │   ❌   │   ❌   │   ❌   │  创建        │
│  PUT   │   ✅   │   ❌   │   ❌   │  完整更新    │
│  PATCH │   ❌   │   ❌   │   ❌   │  部分更新    │
│  DELETE│   ✅   │   ❌   │   ❌   │  删除        │
│  HEAD  │   ✅   │   ✅   │   ✅   │  获取头部    │
│ OPTIONS│   ✅   │   ✅   │   ❌   │  查询方法    │
└────────┴────────┴────────┴────────┴──────────────┘

幂等性：多次请求结果相同
安全性：不修改服务器状态
```

---

## 5. Cookie和Session

### 5.1 为什么需要Cookie？

```
问题：
HTTP是无状态的
→ 服务器无法识别用户
→ 无法保持登录状态

解决方案：
Cookie = 客户端存储的小数据

工作流程：
1. 用户登录成功
2. 服务器返回Set-Cookie头
3. 浏览器保存Cookie
4. 后续请求自动带上Cookie
5. 服务器识别用户
```

### 5.2 Cookie详解

**设置Cookie**：

```
服务器响应：
HTTP/1.1 200 OK
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure
Set-Cookie: user=Alice; Max-Age=86400

客户端后续请求：
GET /api/profile HTTP/1.1
Cookie: session_id=abc123; user=Alice
```

**Cookie属性**：

```
Name=Value
  - Cookie的键值对

Expires=Wed, 21 Oct 2025 07:28:00 GMT
  - 过期时间（绝对时间）

Max-Age=3600
  - 有效期（秒数，相对时间）

Domain=.example.com
  - 生效域名
  - .example.com：所有子域名都生效

Path=/
  - 生效路径
  - /：所有路径
  - /api：只在/api路径下生效

Secure
  - 只在HTTPS下传输

HttpOnly
  - 禁止JavaScript访问
  - 防止XSS攻击

SameSite=Strict
  - Strict：完全禁止跨站发送
  - Lax：大部分情况禁止
  - None：允许跨站（需要Secure）
  - 防止CSRF攻击
```

**Cookie示例**：

```python
# 设置普通Cookie
Set-Cookie: user=Alice

# 设置过期Cookie（30天）
Set-Cookie: user=Alice; Max-Age=2592000

# 设置安全Cookie
Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict
```

### 5.3 Session详解

**Session工作原理**：

```
流程：

1. 用户登录
   ─────────────>
   POST /api/login
   {"username": "Alice", "password": "***"}

2. 服务器验证成功
   - 创建Session（服务器内存/数据库）
   - Session ID: abc123
   - Session数据: {user_id: 1, username: "Alice"}

3. 返回Session ID
   <─────────────
   Set-Cookie: session_id=abc123; HttpOnly

4. 后续请求带上Cookie
   ─────────────>
   GET /api/profile
   Cookie: session_id=abc123

5. 服务器查找Session
   - 根据session_id查找Session数据
   - 获取用户信息
   - 返回用户资料
```

**Cookie vs Session**：

```
┌────────────┬────────────────┬────────────────┐
│   特性     │     Cookie     │     Session    │
├────────────┼────────────────┼────────────────┤
│  存储位置  │   客户端       │   服务器       │
│  安全性    │   较低         │   较高         │
│  容量限制  │   4KB          │   无限制       │
│  性能      │   无压力       │   占用内存     │
│  生命周期  │   可持久化     │   关闭浏览器失效│
└────────────┴────────────────┴────────────────┘

实际应用：
Cookie存储Session ID
Session存储用户数据
```

---

## 6. HTTP版本演进

### 6.1 HTTP/1.0

```
特点：
- 每个请求都要建立新的TCP连接
- 请求完成后立即关闭连接

问题：
- 性能差（TCP三次握手开销）
- 无法复用连接

示例：
客户端 → 服务器：建立连接
客户端 → 服务器：GET /page1.html
服务器 → 客户端：返回page1.html
断开连接

客户端 → 服务器：建立连接
客户端 → 服务器：GET /page2.html
服务器 → 客户端：返回page2.html
断开连接
```

### 6.2 HTTP/1.1

```
改进：
✅ 持久连接（Connection: keep-alive）
✅ 管道化（Pipelining）
✅ 分块传输（Chunked Transfer）
✅ 缓存控制增强
✅ Host头部（支持虚拟主机）

持久连接：
客户端 → 服务器：建立连接
客户端 → 服务器：GET /page1.html
服务器 → 客户端：返回page1.html
客户端 → 服务器：GET /page2.html
服务器 → 客户端：返回page2.html
... 保持连接 ...

问题：
- 队头阻塞（Head-of-Line Blocking）
- 一个请求慢会阻塞后续请求
```

### 6.3 HTTP/2

```
改进：
✅ 二进制协议（不再是文本）
✅ 多路复用（Multiplexing）
✅ 头部压缩（HPACK）
✅ 服务器推送（Server Push）
✅ 流优先级

多路复用：
一个TCP连接，多个请求并行

┌─────────────────────────┐
│      TCP连接             │
├─────────────────────────┤
│ Stream 1: GET /page1    │
│ Stream 2: GET /style.css│
│ Stream 3: GET /script.js│
│ Stream 4: GET /image.jpg│
└─────────────────────────┘

优点：
- 解决队头阻塞
- 减少延迟
- 提高性能
```

### 6.4 HTTP/3

```
改进：
✅ 基于QUIC（UDP）
✅ 更快的连接建立
✅ 更好的拥塞控制
✅ 连接迁移（IP变化不断线）

为什么用UDP？
- TCP队头阻塞无法彻底解决
- UDP更灵活
- QUIC在应用层实现可靠传输

对比：
HTTP/1.1：文本 + TCP
HTTP/2：二进制 + TCP
HTTP/3：二进制 + QUIC(UDP)
```

### 6.5 版本对比

```
┌────────────┬──────────┬──────────┬──────────┐
│   特性     │ HTTP/1.1 │  HTTP/2  │  HTTP/3  │
├────────────┼──────────┼──────────┼──────────┤
│  协议      │   文本   │  二进制  │  二进制  │
│  传输层    │   TCP    │   TCP    │  QUIC    │
│  多路复用  │    ❌    │    ✅    │    ✅    │
│  头部压缩  │    ❌    │    ✅    │    ✅    │
│  服务器推送│    ❌    │    ✅    │    ✅    │
│  连接建立  │   慢     │   中等   │   快     │
└────────────┴──────────┴──────────┴──────────┘
```

---

## 7. 实战项目：简单HTTP服务器

### 7.1 项目需求

```
功能需求：
1. 支持GET/POST请求
2. 解析HTTP请求
3. 返回HTTP响应
4. 支持静态文件服务
5. 支持JSON API
6. 日志记录

技术要求：
- 使用Socket实现
- 遵循HTTP/1.1协议
- 支持Content-Type识别
- 支持基本的错误处理
```

### 7.2 工具演示

```bash
# 启动HTTP服务器
python day19_http_server.py --host 0.0.0.0 --port 8080

# 测试HTTP客户端
python day19_http_client.py --url http://127.0.0.1:8080/

# 浏览器访问
open http://127.0.0.1:8080/
```

---

## 8. 今日练习

### 练习1：手动构建HTTP请求

```python
# 使用socket发送原始HTTP请求
# 1. 连接到www.baidu.com:80
# 2. 发送GET请求
# 3. 解析响应
# 4. 打印状态码和头部
```

### 练习2：实现HTTP客户端

```python
# 实现一个简单的HTTP客户端
# 支持：
# 1. GET请求
# 2. POST请求（JSON）
# 3. 自动处理重定向
# 4. Cookie管理
```

### 练习3：分析HTTP流量

```python
# 使用Wireshark或tcpdump
# 1. 捕获HTTP流量
# 2. 分析请求和响应
# 3. 识别状态码和头部
```

### 练习4：扩展HTTP服务器

```python
# 为HTTP服务器添加功能：
# 1. 支持PUT/DELETE方法
# 2. Session管理
# 3. 文件上传
# 4. 访问控制（Basic Auth）
```

---

## 9. 常见问题

**Q1：GET和POST的本质区别是什么？**

A：
- 语义：GET用于获取，POST用于提交
- 参数：GET在URL，POST在请求体
- 幂等性：GET幂等，POST不幂等
- 缓存：GET可缓存，POST不可
- 安全性：GET参数可见，POST相对隐藏

**Q2：为什么有状态码304？**

A：
- 节省带宽
- 利用浏览器缓存
- 服务器返回"资源未修改"
- 浏览器使用本地缓存

**Q3：Cookie和Session哪个更安全？**

A：
- Session更安全（数据在服务器）
- Cookie易被窃取和篡改
- 实际使用：Cookie存Session ID

**Q4：HTTP/2比HTTP/1.1快多少？**

A：
- 理论上：30%-50%
- 实际：取决于网络和资源数量
- 资源越多，优势越明显

**Q5：什么时候用POST，什么时候用PUT？**

A：
- POST：创建新资源（ID由服务器生成）
- PUT：更新已存在资源（ID已知）
- POST非幂等，PUT幂等

---

## 10. 总结

今天我们学习了：

### 核心知识点

1. **HTTP基础**：
   - 应用层协议
   - 无状态、明文传输
   - 请求-响应模式

2. **请求格式**：
   - 请求行（方法 URI 版本）
   - 请求头
   - 空行
   - 请求体（可选）

3. **响应格式**：
   - 状态行（版本 状态码 描述）
   - 响应头
   - 空行
   - 响应体

4. **HTTP方法**：
   - GET：查询
   - POST：创建
   - PUT：更新
   - DELETE：删除

5. **状态码**：
   - 2xx：成功
   - 3xx：重定向
   - 4xx：客户端错误
   - 5xx：服务器错误

6. **Cookie/Session**：
   - Cookie：客户端存储
   - Session：服务器存储
   - 配合使用

7. **版本演进**：
   - HTTP/1.1：持久连接
   - HTTP/2：多路复用
   - HTTP/3：QUIC

### 重点回顾

```
HTTP请求 = 请求行 + 请求头 + 空行 + 请求体
HTTP响应 = 状态行 + 响应头 + 空行 + 响应体

常用状态码：
200 OK
301 永久重定向
302 临时重定向
304 未修改
400 请求错误
401 未认证
403 禁止
404 未找到
500 服务器错误

HTTP/2核心：
多路复用 + 头部压缩 + 二进制协议
```

### 明天预告

**第20天：HTTPS和安全**

内容预览：
- TLS/SSL协议
- 数字证书
- 对称加密和非对称加密
- HTTPS握手过程
- 证书验证
- 常见攻击和防御

---

**继续加油！** 🚀

今天我们学习了HTTP协议的基础知识，这是Web开发的核心。理解HTTP协议对于开发Web应用、调试网络问题都至关重要。明天我们将学习HTTPS，了解如何保护HTTP通信的安全！
