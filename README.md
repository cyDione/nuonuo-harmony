# Nuonuo Harmony（糯糯 · 原生鸿蒙客户端）

原生 **HarmonyOS NEXT（ArkTS / ArkUI Stage Model）** 个人助理客户端，基于统一 `/v1/*` 协议 2.0 直连你自托管的 **[Nuonuo（糯糯）](https://github.com/cyDione/nuonuo)** 独立服务端（`v0.4.0+`）。

> [!NOTE]
> 当前仓库处于初始化阶段，已完成协议接入设计与工程护栏文档，等待在 **DevEco Studio** 中按本机 SDK 生成原生工程骨架后开始 MVP 开发。

---

## 定位与设计原则

- **纯轻量原生端**：零内置服务端、零 Web 套壳，专注鸿蒙手机（及后续平板/折叠屏）日常随身交互。
- **端服彻底解耦**：仅通过 [`/v1/*` Protocol 2.0](docs/protocol-and-architecture.md) 与自托管糯糯服务端通信，支持多服务器配置、扫码配对（`nuonuo://pair?...`）与 Owner 密码登录。
- **能力协商而非版本硬编码**：通过 `GET /v1/hello` 返回的 `features` 动态适配服务端能力，手机端与服务端独立升级不互相阻塞。
- **视觉延续 Spatial UI v15**：浆果绯红（`#E11D48`）主题点缀，天光雾白浅色 / 午夜黑曜深色自动跟随系统，保持干净多气泡对话与大白话审批确认体验。

---

## 开发路线（MVP → 完整陪伴）

- [ ] **M0 · 工程初始化**：在 DevEco Studio 中创建 HarmonyOS NEXT（API 12+ Stage Model）工程并合入本仓库。
- [ ] **M1 · 核心可用（自用首版）**
  - Scan Kit 扫码配对（`nuonuo://pair?server=...&code=...`）与手动输入地址/配对码/密码登录
  - 单飞（Single-Flight）Refresh Token 轮换 + `Asset Store Kit` 硬件级加密持久化
  - 主聊 / 侧聊切换、实时 SSE 流式消息与切回前台增量补齐（`?after=`）
  - 高危动作**大白话审批确认卡**（花钱 / 外发 / 删改一键批准或拒绝）
- [ ] **M2 · 日常高频体验**
  - 附件上传（相册 / 文件）与受保护产物预览（30s `resource-ticket` + ArkWeb）
  - 语音输入听写（`/v1/transcribe`）与语音播报（`/v1/speech`）
  - 右侧/底部抽屉查看后台任务进度与步骤详情、动态（Feed）、每日点子与长期目标
- [ ] **M3 · 深度陪伴与系统集成**
  - 二维语调盘、四维心境展示、`SOUL.md` / `MEMORY.md` 浏览与待确认记忆审核
  - 离线推送通知（待服务端补充 `/v1/devices/:id/push`）、桌面卡片与快捷提问入口

---

## 使用 DevEco Studio 接续建工程

1. 安装 **DevEco Studio**（建议最新 HarmonyOS NEXT 稳定版，API 12+）。
2. 在 DevEco Studio 中新建一个 **Empty Ability（Stage Model / ArkTS）** 工程，或将生成的工程目录内容（`AppScope/`、`entry/`、`build-profile.json5`、`hvigor/`、`oh-package.json5` 等）放入本仓库根目录。
   - **重要**：不要将签名密钥密码写入已提交的 `build-profile.json5`；`.gitignore` 已屏蔽 `*.p12`、`*.cer`、`*.p7b`、`local.properties` 等敏感文件。
3. 在本机或 VPS 启动糯糯服务端（`v0.4.0+`）并签发一次性配对二维码：
   ```sh
   nuonuo start -d
   nuonuo pair --admin --qr
   ```
4. 参考 [协议接入与架构设计](docs/protocol-and-architecture.md) 开始实现核心网络层与页面。

---

## 相关文档

- **[协议接入与架构设计](docs/protocol-and-architecture.md)**：`/v1/*` 认证、Token 轮换、SSE 流式消费、模块划分与鸿蒙系统能力映射
- **[AGENTS.md](AGENTS.md)**：AI 编程助手与协作者开发护栏
- **[Nuonuo 主仓库](https://github.com/cyDione/nuonuo)**：独立 Headless 服务端、WebUI、macOS 桌面端与 `/v1/*` 协议规范

---

## 开源协议

基于 **[MIT License](LICENSE)** 开源。
