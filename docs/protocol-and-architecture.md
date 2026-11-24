# Nuonuo Harmony · 协议接入与架构设计

本文档基于主仓库 **[Nuonuo](https://github.com/cyDione/nuonuo) v0.4.0** 的统一公开协议 `/v1/*`（Protocol 2.0），给出原生鸿蒙（HarmonyOS NEXT / API 12+ / Stage Model）客户端的接口映射、安全存储、实时流处理与工程分层蓝图。

---

## 1. 协议基础约定

| 项目 | 规范 |
| --- | --- |
| Base URL | 用户填写的服务端根地址（如 `https://nuonuo.example.com` 或 `http://192.168.1.10:7420`），客户端规范化时去掉尾部 `/` 与 `/v1` |
| API 前缀 | `<baseUrl>/v1` |
| 协议响应头 | `x-nuonuo-protocol: 2.0` |
| 认证请求头 | `Authorization: Bearer <accessToken>` |
| 错误响应格式 | `{ "error": { "code": string, "message": string } }` |
| 设备元数据 | `{ name: "<设备名>", kind: "phone", platform: "harmonyos", display: "full" }`（折叠屏/平板可用 `kind: "tablet"`） |

---

## 2. 认证、配对与 Token 轮换

### 2.1 探测服务端能力（公开接口）

```http
GET /v1/auth/methods
```

返回：
```json
{
  "protocol": "2.0",
  "server": { "name": "nuonuo", "version": "0.4.0" },
  "webUi": true,
  "methods": {
    "pairing": true,
    "password": false,
    "passkey": false,
    "passkeyConfigured": false
  }
}
```

### 2.2 二维码 / 配对码登录（推荐）

服务端运行 `nuonuo pair --admin --qr` 会生成 10 分钟有效的一次性二维码，内容为标准 URI：

```text
nuonuo://pair?server=https%3A%2F%2Fnuonuo.example.com&code=K7QF-2M9D
```

鸿蒙端使用 **Scan Kit (`@kit.ScanKit`)** 扫码后解析 `server` 与 `code`，提交兑换：

```http
POST /v1/auth/pair/exchange
Content-Type: application/json

{
  "code": "K7QF-2M9D",
  "device": {
    "name": "Mate 70 Pro",
    "kind": "phone",
    "platform": "harmonyos",
    "display": "full"
  }
}
```

成功返回 `201 AuthTokenBundle`：
- `accessToken`（`nna_…`，15 分钟有效）→ **仅存内存**
- `refreshToken`（`nnr_…`，30 天有效，单次轮换）→ **立即存入鸿蒙 Asset Store Kit**
- `device`（含分配的 `scopes`）、`session`

同样支持 Owner 密码登录：`POST /v1/auth/password/login`，请求体 `{ password, device }`。

### 2.3 单飞 Refresh Token 轮换（关键防踩坑）

> [!CAUTION]
> 服务端对 `refreshToken` 实行严格一次性轮换：**若旧 `refreshToken` 被并发发送两次，服务端会判定为令牌重放并立即吊销整条会话家族**。

鸿蒙端 `AuthManager` 必须实现全局互斥刷新：
1. 任意接口收到 `401` 时调用 `refreshSingleFlight()`；
2. 若已有进行中的 `refreshPromise`，所有并发 `401` 请求复用同一个 Promise，绝不重复发 `POST /v1/auth/refresh`；
3. 请求 `POST /v1/auth/refresh`（Body `{ "refreshToken": currentRefreshToken }`）；
4. 收到新 `AuthTokenBundle` 后，**先原子覆盖写入 Asset Store 中的 `refreshToken`**，再更新内存 `accessToken` 并重试原请求；
5. 若刷新本身返回 `401`，清空该服务器凭据并触发 `onAuthLost` 引导重新配对/登录。

### 2.4 鸿蒙安全存储映射

| 数据 | 存储位置 | 说明 |
| --- | --- | --- |
| `accessToken` | 内存（`SessionState`） | 不落盘，进程重启后用 `refreshToken` 静默换取 |
| `refreshToken` | `@kit.AssetStoreKit`（关键资产存储服务） | 硬件级加密保护，按 `serverId` 做 alias 隔离；可选开启生物识别解锁 |
| 服务器列表（`id`, `name`, `baseUrl`, `assistant`） | `@kit.ArkData` Preferences | 非敏感元数据，支持多台服务端一键切换 |

---

## 3. 握手与能力协商（跨版本对齐核心）

登录或切回前台后调用：

```http
GET /v1/hello
Authorization: Bearer <accessToken>
```

返回：
```json
{
  "protocol": "2.0",
  "server": { "name": "nuonuo", "version": "0.4.0", "webUi": true },
  "assistant": { "name": "糯糯", "avatar": "...", "color": "#E11D48", "language": "zh-CN" },
  "device": { "id": "...", "scopes": ["chat", "notices", "approvals", "tasks:read", "tasks:write", "admin"] },
  "sessionId": "...",
  "mainConversationId": "main",
  "features": ["..."],
  "limits": { "maxUpload": 26214400, "maxText": 32000 }
}
```

客户端功能可见性由两部分共同决定：
1. **`device.scopes`**：无 `approvals` 隐藏审批操作、无 `tasks:write` 禁用任务编辑等；
2. **`features`**：新增端云联动特性（如未来离线推送、新产物类型）按 `features.includes(...)` 动态开启，避免手机端升级后连旧服务端报错。

---

## 4. 核心业务接口速查

### 对话与消息（`chat`）
- `GET /v1/conversations?archived=0|1` — 会话列表（主聊 `main: true` + 侧聊）
- `POST /v1/conversations` — 新建侧聊 `{ title? }`
- `GET /v1/conversations/:id/messages?after=<msgId>&limit=50` — 历史与增量拉取（`:id` 支持 `main`）
- `POST /v1/conversations/:id/messages` — 发消息 `{ text, clientId, attachments?: string[] }`（`clientId` 幂等去重）
- `POST /v1/conversations/:id/read` — 标记已读

### 附件与受保护产物（`chat` / `tasks:read`）
- `POST /v1/uploads` — `multipart/form-data`（`file` + 可选 `conversation`）→ 返回 `{ upload: Attachment }`
- `POST /v1/auth/resource-ticket` — `{ path, maxUses?: 4 }` → 返回 30 秒临时票据 `{ ticket, expiresAt }`，供 `ArkWeb` 预览 HTML 产物或文件下载使用

### 高危审批（`approvals`）
- `GET /v1/approvals` — 获取全部待确认卡片（`id`, `task`, `tool`, `action`, `risk`, `reason`, `createdAt`）
- `POST /v1/approvals/:id` — 提交审批 `{ approve: boolean, note?: string, scope?: "once" | "task" | "always" }`（非 `admin` 仅限 `once` / `task`）

### 语音、任务与主动能力（M2+）
- `POST /v1/transcribe` / `POST /v1/speech` — 语音听写与朗读
- `GET /v1/tickets/*`, `/v1/artifacts/*`, `/v1/feed/*`, `/v1/ideas/*`, `/v1/goals/*`, `/v1/memories/*` — 活动步骤、成品、动态、点子、目标与记忆

---

## 5. 实时事件流（SSE）在鸿蒙端的实现要点

### 订阅端点

```http
GET /v1/events
Authorization: Bearer <accessToken>
Accept: text/event-stream
```

可选 `?conversation=main|<id>` 过滤单会话。

### 事件类型（与 `@nuonuo/client` 严格一致）

| SSE `event:` | JSON `data` 关键字段 | 客户端处理 |
| --- | --- | --- |
| `ready` | `{ protocol, device, sessionId }` | 标记在线状态，重置重连退避为 1s |
| `approval` | `{ op: "requested", approval }` / `{ op: "resolved", id }` | 实时弹出或移除审批确认卡 |
| `message` | `type: "message.created" \| "message.updated"` + `{ conversationId, messageId, message? }` | 插入或更新气泡；如无内嵌 `message` 则按 `after` 补拉 |
| `message` | `type: "reply.delta"` + `{ conversationId, callId, text }` | 流式追加当前回复气泡草稿 |
| `message` | `type: "conversation.updated"` + `{ conversationId }` | 刷新会话列表顺序与未读角标 |
| `message` | `type: "notice"` + `{ kind, title, body, silent }` | 非静默时展示应用内横幅或系统通知 |
| 其他未知类型 | 任意 | **静默忽略**，保持前向兼容 |

### 鸿蒙网络层注意事项

1. **流式接收**：使用 `@kit.NetworkKit` 的 `http.createHttp().requestInStream()`，监听 `dataReceive` 回调接收 `ArrayBuffer`。
2. **UTF-8 跨分片解码**：使用 `@kit.ArkTS` 的 `util.TextDecoder.create('utf-8', { ignoreBOM: true })` 并传 `{ stream: true }`，防止中文字符恰好被 TCP 分片截断产生乱码。
3. **重连与退避**：网络波动按 `1s → 2s → 4s → ... → 30s` 指数退避；遇到 `401` 先走单飞刷新再重连；收到 `403` 或刷新 `401` 则停止重连并提示重新认证。
4. **前后台切换补齐**：
   - `onBackground`：记录当前各会话最后一条 `messageId`，允许系统回收长连接；
   - `onForeground`：重建 SSE，并立即并发执行 `GET /v1/conversations/:activeId/messages?after=<lastId>` 与 `GET /v1/approvals` 补齐后台期间遗漏的事件。

---

## 6. 推荐 Stage Model 目录划分

待 DevEco Studio 生成工程后，建议 `entry/src/main/ets/` 按单向分层组织：

```text
entry/src/main/ets/
├── entryability/
│   └── EntryAbility.ets          # 生命周期、深链 nuonuo://pair、前后台状态通知
├── core/
│   ├── auth/
│   │   ├── AssetVault.ets        # @kit.AssetStoreKit 存取 refreshToken
│   │   ├── PairUriParser.ets     # 解析 nuonuo://pair?server=...&code=...
│   │   └── ServerRegistry.ets    # 多服务端配置持久化
│   ├── api/
│   │   ├── Models.ets            # Protocol 2.0 强类型模型（strict ArkTS）
│   │   ├── SseParser.ets         # UTF-8 流式分帧 + 事件分发
│   │   └── NuonuoClient.ets      # 单飞刷新 + REST + SSE 客户端
│   └── store/
│       ├── SessionStore.ets      # 当前连接、hello.features、在线状态
│       ├── ChatStore.ets         # 会话列表、消息流、流式 delta、幂等发送队列
│       └── ApprovalStore.ets     # 待审批队列与一键批复
├── features/
│   ├── connect/                  # 服务器管理、扫码配对页、密码登录页
│   ├── chat/                     # 主聊/侧聊页、多气泡列表、审批卡、输入栏
│   ├── activity/                 # 后台任务进度与步骤检视、产物 ArkWeb 预览
│   └── companion/                # 动态、每日点子、目标、心境与记忆（M2/M3）
└── shared/
    ├── theme/                    # Spatial UI v15 浅/深色陶瓷色板（#E11D48）
    └── components/               # 通用气泡、确认卡、毛玻璃胶囊组件
```

---

## 7. 后续需要主仓配合增强的能力（可选）

当前 `v0.4.0` 服务端已经足以跑通完整的 **M1 + M2** 体验；后续如需体验更接近原生即时通讯，可在主仓库 `cyDione/nuonuo` 增量补充两项能力（通过 `hello.features` 向下兼容）：

1. **离线推送注册**（`POST /v1/devices/:id/push`）：切后台后用 Huawei Push Kit / ntfy / Bark 推送高危审批与重要动态提醒。
2. **机器可读 OpenAPI 导出**：从 `apps/daemon` 导出 `openapi.json`，便于本仓库校验模型字段与主仓零漂移。
