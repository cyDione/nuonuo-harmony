# AGENTS.md — Nuonuo Harmony 开发与架构护栏

本文件是 AI 编程助手与协作者进入 `nuonuo-harmony` 仓库的入口约束；协议与模块细节见 [`docs/protocol-and-architecture.md`](docs/protocol-and-architecture.md)。

## 1. 仓库定位与边界

- 本仓库是 **[Nuonuo（糯糯）](https://github.com/cyDione/nuonuo)** 的**原生鸿蒙客户端**（HarmonyOS NEXT / API 12+ / ArkTS / ArkUI Stage Model）。
- **只消费公开协议**：严禁假设服务端源码存在于本仓库，所有数据与能力严格经由统一 `/v1/*` Protocol 2.0（响应头 `x-nuonuo-protocol: 2.0`）获取。
- **能力协商优先**：涉及服务端可选或后续新增能力时，必须检查 `GET /v1/hello` 返回的 `features` 列表并做优雅降级，禁止硬编码判断服务端版本号来开关功能。

## 2. 不可突破的安全与认证红线

1. **Token 生命周期与存储隔离**：
   - `accessToken`（`nna_…`，15 分钟短效）**仅驻留内存**，绝不落盘、绝不写入明文 Preferences、绝不打印到 `hilog`。
   - `refreshToken`（`nnr_…`，30 天单次轮换）**必须使用鸿蒙系统安全存储**（优先 `@kit.AssetStoreKit` / `@kit.UniversalKeystoreKit` HUKS 加密后存盘），禁止明文写入普通 Preferences 或沙箱文本文件。
2. **严格单飞刷新（Single-Flight Refresh）**：
   - 服务端启用了 Refresh Token 重放检测，同一旧 `refreshToken` 被并发发送两次会**立即吊销整条会话家族**。
   - 网络层必须维护全局串行/单飞刷新 Promise（复用同一次 `POST /v1/auth/refresh`），拿到新令牌对后**先原子持久化新 `refreshToken`**，再放行等待中的请求。
3. **严禁 URL 携带长期令牌**：
   - 受保护的文件下载、图片加载或 `ArkWeb` 产物预览，禁止将 `accessToken` / `refreshToken` 拼进 URL Query；必须请求 `POST /v1/auth/resource-ticket` 获取 30 秒单路径临时票据。
4. **严禁提交签名与密钥资产**：
   - 禁止提交 `*.p12`、`*.cer`、`*.p7b`、`*.csr`、`local.properties` 或包含明文签名口令的 `build-profile.json5`；本地自动/手动签名配置需与提交配置分离。

## 3. ArkTS 与网络层约束

- **遵守 ArkTS 严格类型规范**：禁用 `any` / `unknown` 隐式推断、禁用动态增删对象属性、明确定义接口/类模型，避免触发 ArkTS 编译期静态检查错误（`arkts-no-any-unknown`、`arkts-no-untyped-obj-literals` 等）。
- **SSE 实时流规范**：
  - 使用 `@kit.NetworkKit` 的 `http.createHttp().requestInStream()` 订阅 `GET /v1/events`（带 `Authorization: Bearer <accessToken>` 与 `Accept: text/event-stream`）。
  - 正确处理跨 TCP chunk 截断的 UTF-8 多字节字符（中文）与 `\r\n` / `\n\n` 事件分隔。
  - **静默忽略未知事件类型**：保障前向兼容，切勿因服务端新增 SSE 事件导致解析抛错断流。
  - 鸿蒙应用进入后台（`onBackground`）时应妥善处理连接挂起；回到前台（`onForeground`）时立即重连并按 `GET /v1/conversations/:id/messages?after=<lastId>` 增量拉取断线期间漏掉的消息，同时刷新 `/v1/approvals`。
- **发送消息幂等**：调用 `POST /v1/conversations/:id/messages` 必须携带稳定的 `clientId`（UUID），弱网重试复用同一 `clientId` 避免重复发消息。

## 4. 模块分层方向

推荐在 `entry/src/main/ets/`（或拆分 HAR 模块）中严格单向依赖：

```text
features/* (UI 页面与视图状态)
   ↓
core/store (会话、消息、审批、多服状态管理)
   ↓
core/api   (NuonuoClient、SSE 解析、单飞刷新、数据模型)
   ↓
core/auth  (Asset Store / HUKS 凭据库、配对 URI 解析)
```

禁止 UI 页面直接拼接 `/v1/*` URL 或直接读取底层凭据存储。

## 5. 提交前检查

- 确认未包含签名证书、明文 Token、本机局域网真实凭据或 `local.properties`。
- 工程接入 DevEco Studio 后，确保 `hvigorw assembleHap`（及后续单元测试）通过后再提交。
