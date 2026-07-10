# 网络基础

## OSI 七层模型

| 层 | 名称 | 协议 |
|----|------|------|
| 7 | 应用层 | HTTP, FTP, SMTP |
| 6 | 表示层 | SSL/TLS, JPEG |
| 5 | 会话层 | NetBIOS, RPC |
| 4 | 传输层 | TCP, UDP |
| 3 | 网络层 | IP, ICMP |
| 2 | 数据链路层 | Ethernet, WiFi |
| 1 | 物理层 | 光纤, 双绞线 |

## TCP vs UDP

| 特性 | TCP | UDP |
|------|-----|-----|
| 连接 | 面向连接 | 无连接 |
| 可靠性 | 可靠 | 不可靠 |
| 顺序 | 保证顺序 | 不保证 |
| 速度 | 较慢 | 较快 |
| 用途 | HTTP, FTP | DNS, 视频 |

## HTTP 协议

### 请求方法

| 方法 | 用途 | 幂等 |
|------|------|------|
| GET | 获取资源 | 是 |
| POST | 创建资源 | 否 |
| PUT | 更新资源 | 是 |
| DELETE | 删除资源 | 是 |

### 状态码

| 范围 | 含义 |
|------|------|
| 2xx | 成功 |
| 3xx | 重定向 |
| 4xx | 客户端错误 |
| 5xx | 服务端错误 |

## DNS

```bash
# DNS 查询
nslookup example.com
dig example.com

# 本地 DNS 缓存
cat /etc/hosts
```

## Socket 编程

```python
# TCP 服务端
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.bind(('localhost', 8080))
server.listen(5)

while True:
    client, addr = server.accept()
    data = client.recv(1024)
    client.send(b"Hello!")
    client.close()

# TCP 客户端
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(('localhost', 8080))
client.send(b"Hello Server!")
data = client.recv(1024)
client.close()
```