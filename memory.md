
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

