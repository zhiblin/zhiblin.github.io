---
title: 31天：WebSocket实时通信
date: 2025-11-10 15:25:46
categories: Network 
tags: [Network, Book, Python]
---
# 第31天：WebSocket实时通信

> "WebSocket让Web应用拥有了真正的实时双向通信能力！"

## 📚 今日目标

- 理解WebSocket协议原理
- 掌握WebSocket握手过程
- 学习WebSocket帧格式
- 实现WebSocket客户端和服务器
- 理解WebSocket vs HTTP轮询

---

## 1. 为什么需要WebSocket？

### 1.1 HTTP的局限性

```
传统HTTP的问题：

1. 单向通信
   客户端 → 请求 → 服务器
   客户端 ← 响应 ← 服务器

   问题：服务器无法主动推送数据

2. 轮询方式效率低

   短轮询（Polling）：
   客户端每隔一段时间发送请求
   ┌─────────────────────────────────┐
   │ 客户端 → 有新消息吗？           │
   │ 服务器 ← 没有                   │
   │ (等待1秒)                       │
   │ 客户端 → 有新消息吗？           │
   │ 服务器 ← 没有                   │
   │ (等待1秒)                       │
   │ 客户端 → 有新消息吗？           │
   │ 服务器 ← 有！这是消息           │
   └─────────────────────────────────┘

   缺点：
   - 大量无用请求
   - 资源浪费
   - 实时性差

   长轮询（Long Polling）：
   客户端发送请求，服务器hold住连接
   有消息时才返回
   ┌─────────────────────────────────┐
   │ 客户端 → 有新消息吗？           │
   │ (服务器等待...)                 │
   │ (有新消息!)                     │
   │ 服务器 ← 有！这是消息           │
   │ 客户端 → 有新消息吗？(立即重连) │
   └─────────────────────────────────┘

   改进：实时性好
   缺点：仍需频繁建立连接

生活化类比：
HTTP轮询 = 不停问："快递到了吗？"
WebSocket = 快递员：" 到了我就告诉你"
```

---

## 2. WebSocket协议详解

### 2.1 WebSocket特点

```
WebSocket协议特性：

1. 全双工通信
   ┌──────────────────────────────┐
   │  客户端 ←──数据──→ 服务器    │
   │         双向实时通信          │
   └──────────────────────────────┘

2. 基于TCP
   - 可靠传输
   - 有序交付
   - 流量控制

3. 低开销
   - 握手后无需HTTP头
   - 帧开销小（2-14字节）
   - 适合高频消息

4. 保持连接
   - 长连接
   - 无需重复建立

5. 支持文本和二进制
   - 文本帧：UTF-8编码
   - 二进制帧：任意数据

协议标识：
ws://example.com/socket   # 非加密
wss://example.com/socket  # TLS加密（推荐）
```

### 2.2 WebSocket握手

```
WebSocket握手过程：

客户端请求（HTTP Upgrade）：
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

关键头部：
- Upgrade: websocket         # 请求升级到WebSocket
- Connection: Upgrade        # 连接升级
- Sec-WebSocket-Key: xxx     # 随机密钥（Base64）
- Sec-WebSocket-Version: 13  # WebSocket版本

服务器响应：
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

关键头部：
- 101 Switching Protocols    # 切换协议
- Sec-WebSocket-Accept: xxx  # 确认密钥

Accept计算方法：
1. 拼接：Sec-WebSocket-Key + "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
2. SHA-1哈希
3. Base64编码

握手完成后：
- HTTP协议升级为WebSocket
- 开始使用WebSocket帧传输数据
```

### 2.3 WebSocket帧格式

```
WebSocket数据帧结构：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                |
+---------------------------------------------------------------+

字段说明：

1. FIN (1 bit)
   - 1: 最后一帧
   - 0: 后续还有帧

2. RSV1, RSV2, RSV3 (各1 bit)
   - 保留位，通常为0

3. Opcode (4 bits)
   - 0x0: 继续帧
   - 0x1: 文本帧
   - 0x2: 二进制帧
   - 0x8: 关闭连接
   - 0x9: Ping
   - 0xA: Pong

4. MASK (1 bit)
   - 1: 负载已掩码（客户端必须设置）
   - 0: 负载未掩码

5. Payload length (7/7+16/7+64 bits)
   - 0-125: 实际长度
   - 126: 后续16位表示长度
   - 127: 后续64位表示长度

6. Masking-key (0 or 4 bytes)
   - 掩码密钥（仅MASK=1时存在）

7. Payload Data
   - 实际数据
```

### 2.4 心跳机制

```
WebSocket心跳（Ping/Pong）：

作用：
1. 保持连接活跃
2. 检测连接是否断开
3. 防止中间代理超时关闭

Ping/Pong机制：
客户端 → Ping → 服务器
客户端 ← Pong ← 服务器

实现方式：
# 客户端
setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({type: 'ping'}));
    }
}, 30000);  // 每30秒发送一次

# 服务器
if (message.type === 'ping') {
    ws.send(JSON.stringify({type: 'pong'}));
}

超时检测：
let pongReceived = true;

setInterval(() => {
    if (!pongReceived) {
        // 超时，关闭连接
        ws.close();
    }
    pongReceived = false;
    ws.ping();
}, 30000);

ws.on('pong', () => {
    pongReceived = true;
});
```

---

## 3. 实战项目：WebSocket服务器

### 3.1 代码实现

```python
# day31_websocket_server.py
# 功能：WebSocket服务器实现
# 需要安装：pip install websockets

import asyncio
import websockets
import json
from datetime import datetime

# 存储所有连接的客户端
connected_clients = set()


async def handle_client(websocket, path):
    """
    处理客户端连接

    参数：
        websocket: WebSocket连接对象
        path: 连接路径
    """
    # 添加到连接集合
    connected_clients.add(websocket)
    client_id = id(websocket)

    print(f"[{datetime.now()}] 客户端连接: {client_id}")
    print(f"当前在线: {len(connected_clients)} 人")

    # 发送欢迎消息
    welcome_msg = {
        "type": "system",
        "message": f"欢迎加入聊天室！当前在线 {len(connected_clients)} 人",
        "timestamp": datetime.now().isoformat()
    }
    await websocket.send(json.dumps(welcome_msg, ensure_ascii=False))

    # 广播用户加入消息
    join_msg = {
        "type": "user_join",
        "message": f"用户 {client_id} 加入了聊天室",
        "online_count": len(connected_clients),
        "timestamp": datetime.now().isoformat()
    }
    await broadcast(json.dumps(join_msg, ensure_ascii=False), exclude=websocket)

    try:
        # 持续接收消息
        async for message in websocket:
            print(f"[{datetime.now()}] 收到消息: {message}")

            try:
                # 解析JSON消息
                data = json.loads(message)

                # 处理不同类型的消息
                if data.get('type') == 'ping':
                    # 心跳响应
                    pong_msg = {
                        "type": "pong",
                        "timestamp": datetime.now().isoformat()
                    }
                    await websocket.send(json.dumps(pong_msg))

                elif data.get('type') == 'chat':
                    # 聊天消息，广播给所有人
                    chat_msg = {
                        "type": "chat",
                        "from": client_id,
                        "message": data.get('message', ''),
                        "timestamp": datetime.now().isoformat()
                    }
                    await broadcast(json.dumps(chat_msg, ensure_ascii=False))

            except json.JSONDecodeError:
                # 非JSON消息，直接广播
                text_msg = {
                    "type": "chat",
                    "from": client_id,
                    "message": message,
                    "timestamp": datetime.now().isoformat()
                }
                await broadcast(json.dumps(text_msg, ensure_ascii=False))

    except websockets.exceptions.ConnectionClosed:
        print(f"[{datetime.now()}] 客户端 {client_id} 连接关闭")

    finally:
        # 移除连接
        connected_clients.remove(websocket)
        print(f"当前在线: {len(connected_clients)} 人")

        # 广播用户离开消息
        leave_msg = {
            "type": "user_leave",
            "message": f"用户 {client_id} 离开了聊天室",
            "online_count": len(connected_clients),
            "timestamp": datetime.now().isoformat()
        }
        await broadcast(json.dumps(leave_msg, ensure_ascii=False))


async def broadcast(message, exclude=None):
    """
    广播消息给所有客户端

    参数：
        message: 要广播的消息
        exclude: 排除的WebSocket连接（可选）
    """
    if connected_clients:
        # 过滤掉exclude的连接
        clients = connected_clients - {exclude} if exclude else connected_clients

        # 并发发送给所有客户端
        await asyncio.gather(
            *[client.send(message) for client in clients],
            return_exceptions=True
        )


async def main():
    """启动WebSocket服务器"""
    print("""
    ╔══════════════════════════════════════════════════════════╗
    ║       WebSocket 服务器                                    ║
    ║       WebSocket Server                                   ║
    ╚══════════════════════════════════════════════════════════╝

    服务器地址: ws://localhost:8765
    等待客户端连接...
    """)

    # 启动服务器
    server = await websockets.serve(
        handle_client,
        "localhost",
        8765
    )

    print("WebSocket服务器已启动")
    print("按 Ctrl+C 停止服务器\n")

    # 保持服务器运行
    await asyncio.Future()  # 永久运行


if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\n服务器已停止")
```

### 3.2 客户端代码

```python
# day31_websocket_client.py
# 功能：WebSocket客户端

import asyncio
import websockets
import json
from datetime import datetime


async def client():
    """WebSocket客户端"""
    uri = "ws://localhost:8765"

    print("""
    ╔══════════════════════════════════════════════════════════╗
    ║       WebSocket 客户端                                    ║
    ║       WebSocket Client                                   ║
    ╚══════════════════════════════════════════════════════════╝
    """)

    async with websockets.connect(uri) as websocket:
        print(f"已连接到服务器: {uri}\n")

        # 启动接收消息的任务
        receive_task = asyncio.create_task(receive_messages(websocket))

        # 启动心跳任务
        heartbeat_task = asyncio.create_task(send_heartbeat(websocket))

        # 发送消息
        print("输入消息并按Enter发送（输入'quit'退出）：\n")
        try:
            while True:
                message = await asyncio.get_event_loop().run_in_executor(
                    None, input, "你: "
                )

                if message.lower() == 'quit':
                    break

                # 发送聊天消息
                chat_msg = {
                    "type": "chat",
                    "message": message,
                    "timestamp": datetime.now().isoformat()
                }
                await websocket.send(json.dumps(chat_msg, ensure_ascii=False))

        except KeyboardInterrupt:
            pass
        finally:
            receive_task.cancel()
            heartbeat_task.cancel()
            print("\n已断开连接")


async def receive_messages(websocket):
    """接收服务器消息"""
    try:
        async for message in websocket:
            data = json.loads(message)

            if data['type'] == 'system':
                print(f"\n[系统] {data['message']}")
            elif data['type'] == 'user_join':
                print(f"\n[通知] {data['message']} (在线: {data['online_count']})")
            elif data['type'] == 'user_leave':
                print(f"\n[通知] {data['message']} (在线: {data['online_count']})")
            elif data['type'] == 'chat':
                print(f"\n[{data['from']}]: {data['message']}")
            elif data['type'] == 'pong':
                # 心跳响应，不显示
                pass

            print("你: ", end='', flush=True)

    except asyncio.CancelledError:
        pass


async def send_heartbeat(websocket):
    """发送心跳"""
    try:
        while True:
            await asyncio.sleep(30)  # 每30秒
            ping_msg = {
                "type": "ping",
                "timestamp": datetime.now().isoformat()
            }
            await websocket.send(json.dumps(ping_msg))
    except asyncio.CancelledError:
        pass


if __name__ == "__main__":
    asyncio.run(client())
```

---

## 4. WebSocket vs HTTP对比

```
对比总结：

┌────────────┬───────────────────┬───────────────────┐
│ 特性       │ HTTP              │ WebSocket         │
├────────────┼───────────────────┼───────────────────┤
│ 通信方向   │ 单向（请求-响应） │ 双向（全双工）    │
│ 连接       │ 短连接            │ 长连接            │
│ 开销       │ 每次请求都有头部  │ 握手后开销小      │
│ 实时性     │ 差（需轮询）      │ 好（推送）        │
│ 服务器推送 │ 不支持            │ 支持              │
│ 适用场景   │ 传统Web应用       │ 实时应用          │
└────────────┴───────────────────┴───────────────────┘

适用场景：

WebSocket适合：
✅ 实时聊天
✅ 在线游戏
✅ 实时协作（文档编辑）
✅ 实时数据展示（股票、监控）
✅ 推送通知

HTTP适合：
✅ 传统网页浏览
✅ RESTful API
✅ 文件上传/下载
✅ 不需要实时性的应用
```

---

## 5. 今日练习

### 练习1：实现简单聊天室
运行WebSocket服务器和多个客户端，测试：
1. 消息广播
2. 用户加入/离开通知
3. 在线人数统计

### 练习2：添加功能
扩展聊天室功能：
1. 用户命名
2. 私聊功能
3. 消息历史记录
4. 断线重连

---

**进度：31/120天（25.8%）**

明天我们将进行实战项目：构建完整的Web API服务！
