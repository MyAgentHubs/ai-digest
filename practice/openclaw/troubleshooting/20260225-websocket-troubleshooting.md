# OpenClaw WebSocket 连接故障排查指南

**适用版本**: v2026.2.3+
**来源**: 官方文档 | Discord 社区 | 实际案例
**验证状态**: Verified
**最后更新**: 2026-02-25

## 概述

OpenClaw 的核心通信机制依赖 WebSocket 连接（默认端口 18789）。本文档汇总常见的 WebSocket 连接问题及排查方法，基于官方文档和社区实际案例整理。

## 常见问题速查表

| 问题现象 | 可能原因 | 快速检查命令 |
|---------|---------|-------------|
| 🔴 连接被拒绝 | Gateway 未启动 | `ps aux \| grep openclaw` |
| 🟡 连接超时 | 防火墙阻止 | `lsof -i :18789` |
| 🟠 频繁断连 | 网络不稳定 | `ping -c 10 localhost` |
| 🔵 认证失败 | Token 配置错误 | 检查 `~/.openclaw/config.yaml` |
| 🟣 消息丢失 | 消息队列满 | 检查日志中的 `queue full` |

---

## 问题 1: WebSocket 连接被拒绝

### 症状
```bash
Error: connect ECONNREFUSED 127.0.0.1:18789
WebSocket connection failed: Connection refused
```

### 排查步骤

**1. 检查 Gateway 进程是否运行**
```bash
ps aux | grep openclaw
# 应该看到类似输出：
# panda  12345  0.5  1.2  openclaw gateway
```

**2. 检查端口占用**
```bash
lsof -i :18789
# 如果没有输出，说明端口未被监听
```

**3. 启动 Gateway**
```bash
openclaw gateway
# 或使用守护进程模式
openclaw gateway --daemon
```

### 解决方案

**方案 A: 手动启动**
```bash
cd ~/.openclaw
openclaw gateway
```

**方案 B: 使用系统服务（推荐）**

创建 systemd 服务（Linux）：
```bash
sudo tee /etc/systemd/system/openclaw.service > /dev/null <<EOF
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=$HOME/.openclaw
ExecStart=$(which openclaw) gateway
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable openclaw
sudo systemctl start openclaw
```

macOS LaunchAgent：
```bash
cat > ~/Library/LaunchAgents/com.openclaw.gateway.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openclaw.gateway</string>
    <key>ProgramArguments</key>
    <array>
        <string>$(which openclaw)</string>
        <string>gateway</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>$HOME/.openclaw/logs/gateway.log</string>
    <key>StandardErrorPath</key>
    <string>$HOME/.openclaw/logs/gateway.error.log</string>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.openclaw.gateway.plist
```

---

## 问题 2: 连接超时或间歇性断连

### 症状
```bash
WebSocket connection timeout after 30s
Connection dropped unexpectedly
Reconnecting... (attempt 3/5)
```

### 排查步骤

**1. 检查网络连通性**
```bash
# 测试本地回环
ping -c 5 127.0.0.1

# 测试 WebSocket 端口
telnet 127.0.0.1 18789
# 或使用 nc
nc -zv 127.0.0.1 18789
```

**2. 检查防火墙规则**
```bash
# macOS
sudo pfctl -sr | grep 18789

# Linux (iptables)
sudo iptables -L -n | grep 18789

# Linux (firewalld)
sudo firewall-cmd --list-ports
```

**3. 查看 Gateway 日志**
```bash
tail -f ~/.openclaw/logs/gateway.log
# 关注以下关键词：
# - "connection timeout"
# - "socket error"
# - "handshake failed"
```

### 解决方案

**方案 A: 调整超时配置**

编辑 `~/.openclaw/config.yaml`：
```yaml
gateway:
  websocket:
    ping_interval: 30        # 心跳间隔（秒）
    ping_timeout: 60         # 心跳超时（秒）
    connection_timeout: 120  # 连接超时（秒）
    reconnect_attempts: 5    # 重连次数
    reconnect_delay: 5       # 重连延迟（秒）
```

**方案 B: 开放防火墙端口**
```bash
# macOS
# 系统偏好设置 > 安全性与隐私 > 防火墙 > 防火墙选项
# 允许 openclaw 接受传入连接

# Linux (iptables)
sudo iptables -A INPUT -p tcp --dport 18789 -j ACCEPT
sudo iptables-save > /etc/iptables/rules.v4

# Linux (firewalld)
sudo firewall-cmd --permanent --add-port=18789/tcp
sudo firewall-cmd --reload
```

**方案 C: 使用 Unix Socket（高级）**

如果 TCP 连接不稳定，可切换到 Unix Socket：
```yaml
gateway:
  transport: unix
  socket_path: ~/.openclaw/gateway.sock
```

客户端配置：
```yaml
client:
  gateway_url: unix:///Users/panda/.openclaw/gateway.sock
```

---

## 问题 3: 认证失败

### 症状
```bash
WebSocket authentication failed: Invalid token
Connection rejected: 401 Unauthorized
```

### 排查步骤

**1. 检查 Token 配置**
```bash
# 查看 Gateway Token
grep 'gateway_token' ~/.openclaw/config.yaml

# 查看客户端配置
grep 'auth_token' ~/.openclaw/clients/*/config.yaml
```

**2. 验证 Token 格式**
```bash
# Token 应该是 32 字符的十六进制字符串
# 示例：a1b2c3d4e5f6789012345678901234567890abcd
```

### 解决方案

**方案 A: 重新生成 Token**
```bash
openclaw gateway --reset-token
# 输出新 Token：
# New gateway token: a1b2c3d4e5f6789012345678901234567890abcd
```

**方案 B: 同步客户端配置**
```bash
# 获取当前 Token
TOKEN=$(grep 'gateway_token:' ~/.openclaw/config.yaml | awk '{print $2}')

# 更新所有客户端
for client in ~/.openclaw/clients/*/config.yaml; do
    sed -i.bak "s/auth_token:.*/auth_token: $TOKEN/" "$client"
done
```

**方案 C: 禁用认证（仅本地开发）**
```yaml
gateway:
  require_auth: false  # ⚠️ 仅用于本地测试，生产环境必须启用
```

---

## 问题 4: 消息丢失或延迟

### 症状
```bash
Message sent but no response
Response time > 10s (expected < 1s)
Gateway log: "message queue full, dropping message"
```

### 排查步骤

**1. 检查消息队列状态**
```bash
openclaw gateway status --verbose
# 输出：
# Message Queue: 985/1000 (98.5% full)
# Pending Messages: 127
# Avg Response Time: 8.3s
```

**2. 查看资源使用**
```bash
# CPU 和内存
top -p $(pgrep openclaw)

# 磁盘 I/O
iostat -x 1 5
```

### 解决方案

**方案 A: 调整队列大小**

编辑 `~/.openclaw/config.yaml`：
```yaml
gateway:
  message_queue:
    max_size: 5000          # 增加队列容量
    worker_threads: 4       # 增加工作线程
    batch_size: 10          # 批处理大小
    timeout: 30             # 消息超时（秒）
```

**方案 B: 启用消息持久化**
```yaml
gateway:
  persistence:
    enabled: true
    storage: sqlite         # 或 redis
    db_path: ~/.openclaw/messages.db
    retention_days: 7       # 消息保留天数
```

**方案 C: 优化 Agent 响应**
```bash
# 检查 Agent 性能
openclaw agent list --with-stats

# 输出示例：
# agent-1  Active   Avg: 2.3s  Max: 45s  Timeout: 3
# agent-2  Active   Avg: 0.8s  Max: 5s   Timeout: 0
```

如果某个 Agent 响应慢，考虑：
- 增加 Agent 超时时间
- 优化 Agent 代码
- 使用异步处理

---

## 问题 5: SSL/TLS 错误（远程连接）

### 症状
```bash
WebSocket secure connection failed: CERT_HAS_EXPIRED
SSL handshake error: self-signed certificate
```

### 排查步骤

**1. 检查证书有效期**
```bash
openssl x509 -in ~/.openclaw/certs/gateway.crt -noout -dates
# 输出：
# notBefore=Jan 1 00:00:00 2026 GMT
# notAfter=Dec 31 23:59:59 2026 GMT
```

**2. 验证证书链**
```bash
openssl verify -CAfile ~/.openclaw/certs/ca.crt ~/.openclaw/certs/gateway.crt
```

### 解决方案

**方案 A: 使用 Let's Encrypt（推荐）**
```bash
# 安装 certbot
sudo apt install certbot  # Debian/Ubuntu
# 或
brew install certbot       # macOS

# 申请证书
sudo certbot certonly --standalone -d your-domain.com

# 配置 Gateway
openclaw gateway --tls \
  --cert /etc/letsencrypt/live/your-domain.com/fullchain.pem \
  --key /etc/letsencrypt/live/your-domain.com/privkey.pem
```

**方案 B: 生成自签名证书**
```bash
# 生成 CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -days 365 -key ca.key -out ca.crt

# 生成服务器证书
openssl genrsa -out gateway.key 4096
openssl req -new -key gateway.key -out gateway.csr
openssl x509 -req -days 365 -in gateway.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out gateway.crt

# 移动到配置目录
mv ca.crt gateway.crt gateway.key ~/.openclaw/certs/
```

**方案 C: 客户端信任自签名证书**
```yaml
client:
  gateway_url: wss://your-server:18789
  tls:
    verify: false           # ⚠️ 仅用于测试
    # 或指定 CA 证书
    ca_cert: ~/.openclaw/certs/ca.crt
```

---

## 调试技巧

### 1. 启用详细日志
```bash
# 设置日志级别为 DEBUG
openclaw gateway --log-level debug

# 或在配置文件中
gateway:
  log_level: debug
  log_format: json        # 便于解析
```

### 2. 使用 WebSocket 测试工具
```bash
# wscat（需要安装：npm install -g wscat）
wscat -c ws://127.0.0.1:18789

# 发送测试消息
> {"type":"ping","timestamp":1234567890}

# 预期响应
< {"type":"pong","timestamp":1234567890}
```

### 3. 抓包分析
```bash
# 使用 tcpdump
sudo tcpdump -i lo0 -A 'tcp port 18789'

# 使用 Wireshark
# 捕获过滤器：tcp.port == 18789
# 显示过滤器：websocket
```

### 4. 监控 WebSocket 连接
```bash
# 实时监控
watch -n 1 'lsof -i :18789 | tail -n +2 | wc -l'

# 输出连接数
# Every 1.0s: lsof -i :18789 | tail -n +2 | wc -l
# 3
```

---

## 性能优化建议

### 1. 调整系统参数（Linux）
```bash
# 增加文件描述符限制
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# 优化 TCP 参数
sudo sysctl -w net.ipv4.tcp_tw_reuse=1
sudo sysctl -w net.core.somaxconn=4096
```

### 2. 使用连接池
```yaml
client:
  connection_pool:
    min_size: 5
    max_size: 20
    idle_timeout: 300       # 空闲连接超时（秒）
```

### 3. 启用压缩（大消息场景）
```yaml
gateway:
  websocket:
    compression: true
    compression_level: 6    # 1-9，数字越大压缩率越高
```

---

## 参考架构图

![WebSocket 通信流程图](../images/WebSocket通信流程图.png)

---

## 常见错误代码

| 错误码 | 含义 | 解决方法 |
|-------|------|---------|
| 1000 | 正常关闭 | 无需处理 |
| 1001 | 端点离开 | 检查 Gateway 是否重启 |
| 1002 | 协议错误 | 检查客户端版本兼容性 |
| 1003 | 不支持的数据类型 | 检查消息格式 |
| 1006 | 异常关闭 | 检查网络连接 |
| 1008 | 违反策略 | 检查认证配置 |
| 1011 | 内部错误 | 查看 Gateway 日志 |

---

## 相关文档

- [OpenClaw 快速部署指南](../getting-started/20260225-quick-deploy-guide.md)
- [安全配置最佳实践](../best-practices/20260225-security-guide.md)
- [版本演进总结](../changelog/20260225-version-evolution.md)

---

## 社区支持

遇到无法解决的问题？

- 📖 官方文档：https://docs.openclaw.ai/troubleshooting/websocket
- 💬 Discord 社区：https://discord.gg/clawd（#troubleshooting 频道）
- 🐛 GitHub Issues：https://github.com/openclaw/openclaw/issues

**提问时请附带**：
- OpenClaw 版本（`openclaw --version`）
- 操作系统和版本
- 完整错误日志
- 配置文件（脱敏处理）
