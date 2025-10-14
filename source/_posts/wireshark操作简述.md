---
title: wireshark操作简述
date: 2025-10-13 13:52:10
categories: cn
---

Wireshark是一个网络协议分析工具，可以捕获并分析数据包

# 过滤器

## 基于协议的过滤

```plain
# 显示所有TCP数据包
tcp

# 显示所有UDP数据包
udp

# 显示所有HTTP数据包
http

# 显示所有DNS数据包
dns

# 显示所有HTTPS数据包
ssl or tls

# 显示所有FTP数据包
ftp
```

## 基于IP地址的过滤

```plain
# 显示来自或发送到特定IP的数据包
ip.addr == 192.168.1.100

# 显示来自特定源IP的数据包
ip.src == 192.168.1.100

# 显示发送到特定目标IP的数据包
ip.dst == 8.8.8.8

# 显示源IP和目标IP之间的通信
ip.src == 192.168.1.100 and ip.dst == 8.8.8.8

# 显示与特定网络段相关的数据包
ip.addr >= 192.168.1.0 and ip.addr <= 192.168.1.255
```

## 基于端口号的过滤

```plain
# 显示特定端口的数据包
tcp.port == 80

# 显示特定源端口的数据包
tcp.srcport == 80

# 显示特定目标端口的数据包
tcp.dstport == 443

# 显示多个端口的数据包
tcp.port == 80 or tcp.port == 443 or tcp.port == 8080

# 显示端口范围
tcp.port >= 80 and tcp.port <= 100
```

## 基于HTTP的过滤

```plain
# 显示所有HTTP请求
http.request

# 显示所有HTTP响应
http.response

# 显示特定HTTP方法的请求
http.request.method == "GET"
http.request.method == "POST"
http.request.method == "PUT"
http.request.method == "DELETE"

# 显示包含特定字符串的HTTP数据包
http contains "login"

# 显示特定User-Agent的请求
http.user_agent contains "Mozilla"

# 显示特定主机的HTTP流量
http.host == "www.example.com"

# 显示特定状态码的HTTP响应
http.response.code == 200
http.response.code == 404
http.response.code == 500
```

## 组合过滤

```plain
# AND操作：同时满足两个条件
tcp.port == 80 and ip.src == 192.168.1.100

# OR操作：满足其中一个条件
tcp.port == 80 or tcp.port == 443

# NOT操作：不满足条件
not tcp.port == 80

# 复杂组合
(tcp.port == 80 or tcp.port == 443) and ip.addr == 192.168.1.100

# 使用括号控制优先级
(tcp.srcport == 80 and ip.dst == 192.168.1.1) or (tcp.dstport == 80 and ip.src == 192.168.1.1)
```

