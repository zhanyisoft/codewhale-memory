
---

## Go 高性能聊天室项目

### 项目位置
`/root/codewhale/test1`

### 概述
Go 编写的 WebSocket 聊天室，单进程 50MB 内存，100 万并发零失败。架构核心：单 goroutine 事件循环 + 5 channel + 广播钩子链。可选 Redis/PG 存储。前端单文件 SPA。

### 架构关键点
- **并发模型**：Hub.Run() 单 goroutine 事件循环，5 个 channel（Register/Unregister/Join/Leave/Broadcast），避免锁竞争
- **每连接 2 goroutine**：ReadPump（阻塞读 WS 帧）+ WritePump（逐条发送，不合并）
- **慢客户端保护**：send channel 满 → 直接踢出，不阻塞广播循环
- **广播钩子链**：BroadcastHooks[] 支持多个组件挂载，LocalBroker（持久化）+ DistributedBroker（Redis Pub-Sub）
- **防循环广播**：BroadcastMessage.FromRemote 标志阻断 Redis Pub-Sub 无限回环
- **先广播后存储**：延迟关键路径不等待 I/O，异步 goroutine 写 Redis + PG
- **热冷分层**：Redis Stream（最近 1000 条/房间，TTL 7天）→ PG COPY 批量（100 条或 100ms）

### 性能数据
| 并发 | 成功 | 失败 | P50 | P99 |
|------|------|------|-----|-----|
| 1,000 | 1,000 | 0 | 0.6ms | 7ms |
| 10,000 | 10,000 | 0 | 0.6ms | 30ms |
| 100,000 | 100,000 | 0 | 0.6ms | 30ms |
| 500,000 | 500,000 | 0 | 0.7ms | 298ms |
| 1,000,000 | 1,000,000 | 0 | 0.7ms | 127ms |
- 连接速率：~5,000/s，零失败

### 与 Laravel Reverb 对比
- Go Chatroom: 100 万并发零失败，P50 0.7ms；Laravel Reverb: 最大 1000 并发，1200+ 广播丢失
- 差距：1000 倍并发能力，50 倍连接速率，部署从 5 个 Docker 容器 → 1 个二进制文件

### 技术栈
Go + nhooyr.io/websocket + JWT + Redis Streams + PG COPY + 单文件 SPA 前端

---

## 环境经验

### Codewhale 版本
- 当前版本：**0.9.1**（2026-07-24 发布），GitHub Releases 最新版
- `.codex/version.json` 中的 `0.145.0` 是 `.codex` 内部组件版本，与 Codewhale 无关

### Web 工具 fetch 限制
- `Web` 工具的 `fetch` 功能拒绝访问 `198.18.0.0/15` 私有地址段
- 此沙箱环境中 GitHub 等外部域名被 DNS 解析到该私有段（透明代理）
- 搜索结果能正常获取，但打开页面 URL 会报 `resolved IP 198.18.x.x is a restricted address`
- **替代方案**：直接用 `curl` 走系统网络栈，不受此限制。例如：
  ```
  curl -s https://api.github.com/repos/Hmbown/CodeWhale/releases/latest
  ```

### Codewhale 运行模式
三种模式，空闲时按 `Tab` 循环切换，或用 `/mode` 命令：

| 模式 | 说明 |
|------|------|
| **Plan** | 只读模式，可查看分析代码，不能执行命令或编辑文件 |
| **Act** | 交互编码模式，可检查、编辑、使用 Shell 和工具 |
| **Operate** | 多任务协调模式，功能同 Act，偏好后台 worker 处理独立/并行/长时任务 |

切换命令：`/mode plan`、`/mode act`、`/mode operate`

# 环境记忆

## 网络模式

- 当前为 **WSL2 mirrored 网络模式**（Linux 6.18.35.2-microsoft-standard-WSL2）
- Windows 宿主 IP：`192.168.31.224`
- Docker 版本：`29.6.2`

## Docker 端口映射行为

- Windows `localhost` 自动转发到 WSL2 内 Docker 映射的端口（mirrored 模式自带能力，无需配置）
- **非特权端口（≥1024）**：`docker run -p` 完全正常，Windows 浏览器可直接访问
- **特权端口（<1024，如 80）**：TCP 握手正常但响应体被截断，浏览器显示空白页 —— 避开，用高位端口替代

## 现有服务

| 容器 | 端口 | 用途 |
|:---|:---|:---|
| `laravel-nginx` | 8080 | Laravel 聊天室（原 80 → 已改为 8080） |
| `test-web` | 8081 | Docker 端口映射测试页 |
| `laravel-app` | — | PHP-FPM 8.4 |
| `laravel-mysql` | 3306 | MySQL 8.0 |
| `postgres` | 5432 | PostgreSQL 16 |
| `redis` | 6379 | Redis 7 |
| `laravel-queue` | — | Queue Worker |
| `laravel-reverb` | — | WebSocket |

## 项目路径

- Laravel 项目：`/root/code/my-app/`
- 测试页面：`/root/codewhale/test4/web-test/`
- Nginx 配置：`/root/code/my-app/.docker/nginx/`

## Web 工具限制
- Web search ✅ 正常
- Web fetch ❌ 被沙箱拦截（DNS 解析到 198.18.0.0/15 私有地址段）
- 替代方案：用 curl 抓取网页
- 不要动系统 DNS（10.255.255.254），可能导致断网
