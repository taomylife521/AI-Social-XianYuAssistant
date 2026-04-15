# 闲鱼助手凭证体系说明

## 概述

本项目涉及 3 种凭证，它们之间存在层级依赖关系。Cookie 是所有凭证的基础，如果 Cookie 失效，其他凭证将无法刷新，最终导致 WebSocket 连接掉线。

## 凭证列表

| # | 凭证 | 数据库字段 | 用途 | 有效期 | 刷新方式 |
|---|------|-----------|------|--------|----------|
| 1 | **Cookie** | `xianyu_cookie.cookie_text` | 整体登录态，包含 `_m_h5_tk`、`unb`、`cookie2`、`cna` 等子项 | 约1天（依赖登录态） | `hasLogin` 接口保活（每30分钟一次） |
| 2 | **_m_h5_tk** | `xianyu_cookie.m_h5_tk` | API 签名令牌，用于 mtop 接口的签名计算 | 约2小时 | 调用 H5 API，从响应 `Set-Cookie` 中提取（每2小时定时刷新） |
| 3 | **WebSocket Token (accessToken)** | `xianyu_cookie.websocket_token` | WebSocket 连接的鉴权令牌，建连时通过 URL 参数传入 | 20小时 | 调用 `mtop.taobao.idlemessage.pc.login.token` API 获取（提前1小时刷新） |

## 依赖关系

```
Cookie（登录态，最底层）
  └── _m_h5_tk（从 Cookie 中提取，用于签名）
       └── WebSocket Token（用 _m_h5_tk 签名后请求获取，用于 WebSocket 连接）
```

**核心原则：Cookie 失效 → 一切失效。**

## 各凭证详细说明

### 1. Cookie

- **来源**：用户通过扫码登录获取，存储为完整的 Cookie 字符串
- **关键子项**：
  - `_m_h5_tk`：签名令牌（见下方）
  - `unb`：用户ID，用于构建 hasLogin 请求参数
  - `cookie2`：用于 hasLogin 请求的 `hsiz` 参数
  - `cna`：设备标识，用于 hasLogin 请求的 `deviceId` 参数
  - `XSRF-TOKEN`：CSRF 令牌
- **保活机制**：每30分钟调用 `hasLogin` 接口（`passport.goofish.com/newlogin/hasLogin.do`），保持登录态活跃
- **失效表现**：`hasLogin` 返回 `success: false`
- **相关代码**：
  - 刷新服务：`CookieRefreshServiceImpl`
  - 保活定时任务：`TokenRefreshServiceImpl.scheduledCookieKeepAlive()`

### 2. _m_h5_tk

- **来源**：包含在 Cookie 中，格式为 `{token}_{timestamp}_{sign}_{version}`
- **用途**：mtop API 签名计算，`sign = md5(token + t + appKey + data)`
  - 其中 `token` 是 `_m_h5_tk` 的前半部分（`_` 分割后取第一段）
- **刷新机制**：
  1. 调用 H5 API（`h5api.m.goofish.com/h5/mtop.gaia.nodejs.gaia.idle.data.gw.v2.index.get/1.0/`）
  2. 从响应的 `Set-Cookie` 头中提取新的 `_m_h5_tk`
  3. 失败后重试2次，仍失败则调用 `hasLogin` 刷新 Cookie 后再试
- **定时刷新**：每2小时一次
- **相关代码**：
  - 签名工具：`XianyuSignUtils.generateSign()`
  - 刷新服务：`TokenRefreshServiceImpl.refreshMh5tkToken()`
  - 定时任务：`TokenRefreshServiceImpl.scheduledRefreshMh5tk()`

### 3. WebSocket Token (accessToken)

- **来源**：调用 mtop token API 获取
- **用途**：WebSocket 连接的鉴权令牌，在连接 URL 中以参数形式传入
- **获取流程**：
  1. 从数据库读取最新 Cookie
  2. 解析 `_m_h5_tk` 提取签名 token
  3. 生成签名 `sign = md5(token + timestamp + appKey + data)`
  4. 调用 `mtop.taobao.idlemessage.pc.login.token` API
  5. 从响应 `data.accessToken` 中获取 token
- **刷新机制**：
  - 获取失败时先重试2次
  - 仍失败则调用 `hasLogin` 刷新 Cookie 后重试
  - 响应 `Set-Cookie` 中的新 `_m_h5_tk` 会更新到数据库，供下次签名使用
- **定时刷新**：每分钟检查一次，Token 剩余有效期不足1小时时触发刷新
- **相关代码**：
  - 获取服务：`WebSocketTokenServiceImpl.getAccessToken()`
  - 刷新服务：`TokenRefreshServiceImpl.refreshWebSocketToken()`
  - 定时任务：`TokenRefreshServiceImpl.scheduledRefreshWebSocketToken()`

## 自动重连机制

当 WebSocket 连接断开时，系统会自动尝试重连：

1. **触发条件**：
   - 服务器主动关闭连接（`onClose` 事件）
   - 心跳超时（`heartbeat_interval + heartbeat_timeout` 秒无响应）
   - Token 失效（收到 401 响应）

2. **重连策略**（参考 Python XianyuAutoAgent 的 `while True` 循环）：
   - 无限重连，使用指数退避：5s → 10s → 20s → 40s → 60s（最大60秒）
   - 重连前先通过 `hasLogin` 检查 Cookie 状态
   - 同一账号同时只有一个重连任务（防止重复）

3. **相关代码**：
   - WebSocket 客户端：`XianyuWebSocketClient`
   - 重连调度：`WebSocketServiceImpl.scheduleReconnect()`
   - 心跳检测：`WebSocketServiceImpl.startHeartbeat()`

## 常见问题

### WebSocket 一天左右掉线

**根因**：Cookie 登录态过期，导致 WebSocket Token 无法刷新。

**解决**：确保 `hasLogin` 保活定时任务正常运行（每30分钟一次）。

### Token 获取失败（FAIL_SYS_SESSION_EXPIRED）

**原因**：`_m_h5_tk` 过期，签名无效。

**解决**：系统会自动从响应 `Set-Cookie` 中获取新的 `_m_h5_tk` 并重试，如果重试2次仍失败会自动调用 `hasLogin` 刷新 Cookie。

### 触发风控（RGV587_ERROR）

**原因**：请求频率过高或异常行为被检测。

**解决**：需要用户手动进入闲鱼网页版过滑块验证后更新 Cookie。
