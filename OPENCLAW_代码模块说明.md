# OpenClaw 代码模块说明

本文档基于当前仓库代码结构整理，目标不是做营销式概览，而是帮助开发、排障、二次开发时快速判断每个模块“为什么存在、负责什么、应该去哪里改”。

## 1. 项目整体定位

OpenClaw 不是单一聊天机器人脚本，而是一个完整的个人 AI 助手平台，核心由四层组成：

1. `src/`：核心运行时，负责 CLI、Gateway、Agent、配置、消息路由、模型调用、自动回复、浏览器控制、设备节点、内存、Web 控制面等。
2. `extensions/`：插件层，把不同渠道、外部认证方式、记忆后端、语音/设备能力封装为可插拔模块。
3. `apps/` 与 `ui/`：产品层，分别提供 macOS、iOS、Android 客户端和浏览器控制台。
4. `scripts/`、`docs/`、`skills/`、`test/`：工程化支撑层，负责构建、发布、文档、技能资产和测试。

从入口看，项目的主入口是 [src/index.ts](/Users/wayne/Downloads/Agent/openclaw/src/index.ts)，它先加载环境、校验运行时、接管日志，再构建 CLI 程序；这说明 OpenClaw 的第一入口不是网页，而是本地控制平面。

## 2. 根目录模块地图

| 目录 | 规模 | 作用 |
| --- | ---: | --- |
| `src` | 核心源码主干 | 平台运行时，几乎所有真正的产品逻辑都在这里。 |
| `extensions` | 39 个扩展目录 | 负责渠道接入、外部认证、记忆、语音、线程策略等插件式能力。 |
| `apps/android` | 154 文件 | Android 节点与移动端能力实现。 |
| `apps/ios` | 184 文件 | iOS 节点与客户端实现，当前仍是偏实验/内部使用形态。 |
| `apps/macos` | 363 文件 | macOS 菜单栏客户端、Talk/Voice Wake、节点控制与本地控制中心。 |
| `apps/shared/OpenClawKit` | 118 文件 | iOS/macOS 共享 Swift 库，承载协议、基础能力和聊天 UI 复用。 |
| `ui` | 207 文件 | Gateway 自带的控制台 Web UI，使用 Vite + Lit。 |
| `scripts` | 158 文件 | 构建、打包、发布、诊断、开发辅助脚本。 |
| `skills` | 74 文件 | 内置技能集合，面向最终用户的任务能力封装。 |
| `docs` | 720 文件 | 官方文档与多语言文档资产。 |
| `test` | 46 文件 | 集成测试、脚本测试、辅助测试资源。 |
| `Swabble` | 36 文件 | 独立的唤醒词/语音转写子项目，服务于语音唤醒链路。 |
| `vendor/a2ui` | 173 文件 | 外部引入的 A2UI 画布能力代码，供 Canvas 宿主使用。 |
| `packages` | 空目录 | 当前仓库内未承载实际模块，属于保留结构。 |

## 3. `src/` 核心运行时模块

下面这一节按“真正的源码功能边界”来写。文件数只是帮助判断体量，不代表复杂度上限。

### 3.1 入口、命令与基础设施

| 模块 | 文件数 | 主要职责 | 关键说明 |
| --- | ---: | --- | --- |
| `src/cli` | 288 | CLI 参数层与命令装配层 | 负责参数解析、帮助信息、浏览器/渠道/安全等子命令接口，是所有 `openclaw ...` 命令的命令面。 |
| `src/commands` | 370 | 具体命令实现 | 真正执行 `agent`、`gateway`、`onboard`、`models`、`status`、`channels` 等业务命令。 |
| `src/config` | 233 | 配置系统 | 负责配置文件结构、默认值、兼容迁移、校验、会话配置、路由和 agent 目录规则。 |
| `src/infra` | 378 | 通用基础设施 | 处理端口、环境变量、Bonjour、归档、设备身份、二进制探测、路径安全、控制台资源等底层能力。 |
| `src/process` | 28 | 子进程执行封装 | 对系统命令执行、超时、中断、shell 进程生命周期做统一封装。 |
| `src/logging` | 29 | 日志设施 | 统一日志采集、结构化输出、控制台捕获，是排障时的重要落点。 |
| `src/daemon` | 49 | 守护进程支持 | 负责后台常驻、进程管理、服务化运行的支撑逻辑。 |
| `src/terminal` | 19 | 终端相关能力 | 封装终端交互、TTY 行为与命令行展示细节。 |
| `src/tui` | 45 | 终端 UI | 提供文本交互界面能力，适合无图形环境或快速运维。 |
| `src/wizard` | 16 | 向导式初始化 | 负责安装、配置、接入前的交互式引导。 |
| `src/shared` | 51 | 跨模块共享代码 | 放置多处都会复用的中间层逻辑，减少横向重复。 |
| `src/types` | 9 | 通用类型 | 统一类型定义，降低跨模块接口漂移。 |
| `src/utils` | 29 | 通用工具函数 | 放置轻量级工具逻辑，不适合单独形成业务模块。 |
| `src/compat` | 1 | 兼容层 | 用于处理历史行为或运行环境兼容问题，体量小但通常很关键。 |
| `src/scripts` | 2 | 源码内辅助脚本入口 | 不是根目录 `scripts/` 的替代，更偏源码内部调用。 |
| `src/version.ts` 等单文件入口 | 少量 | 版本、运行时辅助元数据 | 用于发布版本、运行环境检测和入口导出。 |

### 3.2 Agent 运行时与模型能力

| 模块 | 文件数 | 主要职责 | 关键说明 |
| --- | ---: | --- | --- |
| `src/agents` | 850 | Agent 核心运行时 | 项目最大模块，负责模型调用、上下文压缩、工具调用、auth profile、CLI backend、sandbox、技能装载、模型选择、失败切换、命令执行批准等。 |
| `src/acp` | 55 | ACP 协议与运行时桥接 | 负责 ACP 客户端/服务端、会话映射、事件翻译、持久绑定、运行时注册，是 Agent 控制协议层。 |
| `src/providers` | 11 | Provider 级能力 | 针对 GitHub Copilot、Qwen Portal 等上游提供商的专用认证与模型发现逻辑。 |
| `src/memory` | 101 | 长短期记忆与向量化 | 负责 embedding、批处理、远程/本地记忆后端接入、召回、分块与模型限制适配。 |
| `src/hooks` | 48 | 钩子系统 | 在 agent 前后、模型切换、外部事件触发等节点插入扩展逻辑。 |
| `src/secrets` | 50 | 密钥管理 | 统一处理 secret 引用、存储、分发、解析与安全约束。 |
| `src/security` | 29 | 安全策略 | 负责权限边界、输入约束、访问控制与安全默认值。 |
| `src/sessions` | 12 | 会话模型 | 维护会话标识、状态与路由绑定，是多会话 Agent 的基础。 |
| `src/routing` | 11 | 账户/会话路由 | 负责按账户、会话 key、路由规则选择正确的 agent 或目标会话。 |
| `src/context-engine` | 6 | 上下文构造引擎 | 用于在消息进入 Agent 前组织上下文信息。 |
| `src/link-understanding` | 6 | 链接理解 | 对 URL 进行解析和补充理解，为 Agent 提供更高质量上下文。 |
| `src/markdown` | 14 | Markdown 处理 | 统一格式化、渲染前后处理与渠道兼容。 |
| `src/media` | 40 | 媒体基础处理 | 图像、音频、视频的统一输入输出与临时资源管理。 |
| `src/media-understanding` | 65 | 媒体理解链路 | 将多模态内容接入模型，处理转写、识别、摘要等语义层逻辑。 |
| `src/tts` | 4 | 语音合成 | 为 Talk/Voice 相关功能提供 TTS 能力接入。 |

### 3.3 Gateway、Web 与自动回复

| 模块 | 文件数 | 主要职责 | 关键说明 |
| --- | ---: | --- | --- |
| `src/gateway` | 359 | 平台控制面核心 | 负责 WebSocket 控制平面、客户端连接、鉴权、聊天请求、能力发现、配置热重载、节点与渠道状态管理。 |
| `src/web` | 77 | Web 入口与 Web 渠道处理 | 包括登录、二维码、Web inbound/outbound、监控 inbox、媒体转发，是浏览器侧/网页侧能力承接层。 |
| `src/auto-reply` | 287 | 自动回复总线 | 负责消息分块、自动回复调度、命令检测、节流、heartbeat、回复模板、队列与回传行为。 |
| `src/browser` | 155 | 浏览器控制 | 管理 Chromium/Chrome、CDP、桥接服务、动作执行、状态读取、鉴权和扩展中继。 |
| `src/canvas-host` | 6 | Canvas 宿主 | 负责 A2UI 画布能力接入，体量不大，但定位非常明确。 |
| `src/cron` | 108 | 定时任务系统 | 负责 cron 作业、唤醒逻辑、隔离 agent 调度与服务运行。 |
| `src/node-host` | 16 | 设备节点主机能力 | 承接节点调用、设备侧能力暴露与主控交互。 |
| `src/pairing` | 9 | 配对机制 | 处理设备、用户或渠道的配对批准逻辑。 |

### 3.4 渠道抽象与消息接入

这里要特别区分两层：

1. `src/channels`：渠道无关的统一抽象层，处理 allowlist、mention gating、会话规则、送达与节流策略。
2. `src/<channel>`：渠道专属运行时实现，处理各平台协议差异和行为细节。

| 模块 | 文件数 | 主要职责 | 关键说明 |
| --- | ---: | --- | --- |
| `src/channels` | 175 | 渠道抽象层 | 统一处理发送者身份、会话标签、allow-from、命令 gating、group/mention 规则、路由辅助。 |
| `src/telegram` | 141 | Telegram 核心实现 | 处理 Telegram 入站、发送、媒体和平台差异。 |
| `src/slack` | 122 | Slack 核心实现 | 承接 Slack 事件、消息格式、回写与鉴权边界。 |
| `src/discord` | 172 | Discord 核心实现 | 涵盖 DM/群组策略、交互事件、消息读写等。 |
| `src/signal` | 32 | Signal 核心实现 | 处理 signal-cli 相关接入逻辑。 |
| `src/imessage` | 31 | iMessage 核心实现 | 提供遗留 iMessage 通道支持，与 BlueBubbles 路线并存。 |
| `src/line` | 48 | LINE 核心实现 | 处理 LINE 接入与渠道适配。 |
| `src/whatsapp` | 4 | WhatsApp 薄层入口 | 体量很小，说明更多 WhatsApp 能力被抽到了 `src/web`、`src/channels` 或扩展插件中。 |

### 3.5 文档与国际化辅助

| 模块 | 文件数 | 主要职责 | 关键说明 |
| --- | ---: | --- | --- |
| `src/docs` | 1 | 源码内文档桥接 | 通常用于源码层面的文档索引或导出，而不是承载全部站点文档。 |
| `src/i18n` | 1 | 源码级国际化入口 | 真正的多语言内容更多在 `ui/`、`docs/` 和原生 App 中。 |

## 4. `extensions/` 插件模块

`extensions/` 体现了 OpenClaw 的一个关键设计：核心平台不把所有渠道和上游能力硬编码进主干，而是通过插件边界扩展。这让功能可独立启停、迭代和打包。

### 4.1 渠道插件

| 扩展 | 文件数 | 作用 |
| --- | ---: | --- |
| `bluebubbles` | 47 | BlueBubbles 渠道插件，承担推荐的 iMessage 接入路线。 |
| `discord` | 8 | Discord 渠道插件，对外暴露插件接口，底层大量逻辑仍在 `src/discord`。 |
| `feishu` | 98 | Feishu/Lark 渠道插件，社区维护量较大。 |
| `googlechat` | 25 | Google Chat 渠道插件。 |
| `imessage` | 6 | iMessage 渠道插件，偏旧路径的插件包装。 |
| `irc` | 30 | IRC 渠道插件。 |
| `line` | 9 | LINE 渠道插件。 |
| `matrix` | 95 | Matrix 渠道插件，体量较大，说明协议适配较重。 |
| `mattermost` | 53 | Mattermost 渠道插件。 |
| `msteams` | 83 | Microsoft Teams 渠道插件。 |
| `nextcloud-talk` | 32 | Nextcloud Talk 渠道插件。 |
| `nostr` | 28 | Nostr 渠道插件，重点支持 NIP-04 加密私信。 |
| `signal` | 7 | Signal 渠道插件。 |
| `slack` | 6 | Slack 渠道插件。 |
| `synology-chat` | 17 | Synology Chat 渠道插件。 |
| `telegram` | 6 | Telegram 渠道插件。 |
| `tlon` | 38 | Tlon/Urbit 渠道插件。 |
| `twitch` | 36 | Twitch 渠道插件。 |
| `whatsapp` | 8 | WhatsApp 渠道插件。 |
| `zalo` | 32 | Zalo 官方账号渠道插件。 |
| `zalouser` | 37 | Zalo 个人账号插件，基于原生 `zca-js` 集成。 |

### 4.2 协议、能力与工具插件

| 扩展 | 文件数 | 作用 |
| --- | ---: | --- |
| `acpx` | 26 | ACP runtime backend，通过 acpx 暴露运行时能力。 |
| `diffs` | 29 | 差异对比查看插件，适合把文件/文本 diff 以工具形式暴露给 Agent。 |
| `llm-task` | 6 | JSON-only 的 LLM task 插件，适合固定结构任务。 |
| `lobster` | 10 | 工作流工具插件，强调 typed pipeline 与可恢复审批。 |
| `open-prose` | 90 | OpenProse VM 技能包插件，包含斜杠命令和 telemetry。 |
| `voice-call` | 68 | 语音通话插件，是实时语音产品能力的关键组成。 |
| `device-pair` | 3 | 设备配对插件，用来把设备接入配对流程。 |
| `phone-control` | 3 | 手机控制插件，暴露设备控制类能力。 |
| `talk-voice` | 2 | Talk 语音相关插件，体量小但聚焦。 |
| `thread-ownership` | 3 | 线程归属策略插件，用于多线程/多会话边界判定。 |

### 4.3 Provider / OAuth / 记忆插件

| 扩展 | 文件数 | 作用 |
| --- | ---: | --- |
| `copilot-proxy` | 4 | GitHub Copilot Proxy provider 插件。 |
| `google-gemini-cli-auth` | 6 | Gemini CLI OAuth provider 插件。 |
| `minimax-portal-auth` | 5 | MiniMax Portal OAuth provider 插件。 |
| `qwen-portal-auth` | 4 | Qwen Portal OAuth provider 插件。 |
| `memory-core` | 3 | 核心记忆搜索插件，定义记忆能力接口。 |
| `memory-lancedb` | 5 | 基于 LanceDB 的长期记忆插件，支持召回与自动捕获。 |

### 4.4 观测、共享与测试插件

| 扩展 | 文件数 | 作用 |
| --- | ---: | --- |
| `diagnostics-otel` | 5 | OpenTelemetry 诊断输出插件。 |
| `shared` | 2 | 扩展间共享辅助代码。 |
| `test-utils` | 3 | 扩展测试运行时和 mock 支持。 |

## 5. `apps/` 与 `ui/` 产品端模块

这些模块不是演示页，而是最终用户实际会接触的产品形态。

### 5.1 Web 控制台：`ui/`

| 模块 | 文件数 | 说明 |
| --- | ---: | --- |
| `ui` | 207 | Gateway 控制台前端，使用 Vite + Lit 构建。主要负责聊天视图、配置表单、使用量面板、节点视图、渠道视图、调试页面和国际化。 |

`ui/src/ui` 下的实现重点包括：

- `app.ts` / `app-render.ts`：应用主渲染逻辑。
- `app-chat.ts`：聊天展示与消息流。
- `app-gateway.ts` / `gateway.ts`：与 Gateway 的连接和状态同步。
- `views/*`：配置、渠道、节点、会话、用量等具体页面。
- `chat/*`：消息归并、工具卡片、Markdown 展示等聊天细节。

### 5.2 macOS 客户端：`apps/macos`

| 模块 | 文件数 | 说明 |
| --- | ---: | --- |
| `apps/macos` | 363 | 当前最完整的原生客户端，承担菜单栏入口、设置页、Voice Wake、Talk Mode、Canvas 窗口、设备节点控制、远程 Gateway 发现与本地系统能力暴露。 |

从源码命名看，macOS 端包含这些重点子系统：

- 菜单与设置：`SettingsRootView`、`GeneralSettings`、`PermissionsSettings`、`InstancesSettings`、`ChannelsSettings`。
- 语音相关：`VoiceWake*`、`TalkMode*`、`TalkOverlay*`、`TalkAudioPlayer`。
- Canvas 相关：`CanvasWindowController*`、`CanvasScheme*`、`CanvasA2UIActionMessageHandler`。
- 节点与网关：`GatewayConnection`、`GatewayConnectivityCoordinator`、`NodeServiceManager`、`RemotePortTunnel`。
- 系统能力：`PermissionManager`、`CameraCaptureService`、`ShellExecutor`、`LaunchAgentManager`。

这说明 macOS 端不是简单壳层，而是 OpenClaw 很多“本地感”能力的主要承载者。

### 5.3 iOS 客户端：`apps/ios`

| 模块 | 文件数 | 说明 |
| --- | ---: | --- |
| `apps/ios` | 184 | iPhone 节点/客户端实现，当前仍偏 super-alpha，但已具备 Gateway 连接、Canvas、Share Extension、Activity Widget、屏幕/相机能力和节点调用能力。 |

重要组成包括：

- `Sources/`：主 App 源码。
- `ShareExtension/`：把系统分享内容导入 OpenClaw。
- `ActivityWidget/`：Live Activity / Widget。
- `Tests/`：覆盖连接、安全、相机、屏幕录制、Deep Link、Gateway 发现等。

### 5.4 Android 客户端：`apps/android`

| 模块 | 文件数 | 说明 |
| --- | ---: | --- |
| `apps/android` | 154 | Android 节点和移动客户端，负责连接 Gateway、聊天、语音、Canvas、相机/录屏以及 Android 设备命令。 |

从构建脚本可以确认：

- 包名为 `ai.openclaw.app`。
- 共享资源来自 `apps/shared/OpenClawKit/Sources/OpenClawKit/Resources`。
- 工程同时包含 `benchmark` 模块，说明团队在关注启动性能和真实设备表现。

### 5.5 共享 Swift 库：`apps/shared/OpenClawKit`

| 模块 | 文件数 | 说明 |
| --- | ---: | --- |
| `apps/shared/OpenClawKit` | 118 | iOS/macOS 共享库，拆成 `OpenClawProtocol`、`OpenClawKit`、`OpenClawChatUI` 三个产物，用于统一协议、基础能力和聊天 UI。 |

这是移动端与桌面原生端共享代码的关键模块，避免协议与 UI 行为在多端分叉。

## 6. 语音与可视化相关外部子项目

| 模块 | 文件数 | 说明 |
| --- | ---: | --- |
| `Swabble` | 36 | 独立的 Swift 语音唤醒/转写守护项目，提供本地 Speech.framework 唤醒词能力，可被 OpenClaw 复用。 |
| `vendor/a2ui` | 173 | 外部引入的 A2UI 代码库，OpenClaw 用它构建受 Agent 驱动的 Canvas 交互界面。 |

这两个目录的重要性很高：

- `Swabble` 解决“语音入口”问题。
- `A2UI` 解决“可视化输出与可操控工作区”问题。

它们都不是主仓核心逻辑，但都直接影响产品体验。

## 7. 配套工程模块

### 7.1 `scripts/`

`scripts/` 有 158 个文件，是工程化生命线，覆盖：

- TypeScript 构建与产物复制。
- macOS/iOS 签名与打包。
- 文档检查、国际化处理。
- e2e、复现脚本、Docker/Podman、systemd 支持。
- 开发期重启和本地运行辅助。

当你要改构建链、发布链、CI 表现或平台打包时，应优先看这里，而不是直接猜测 `package.json`。

### 7.2 `skills/`

`skills/` 有 74 个文件，包含大量开箱即用技能，例如 GitHub、Discord、Notion、weather、canvas、voice-call 等。这一层更接近“可交付给用户的任务能力资产”，不是底层协议模块。

### 7.3 `docs/`

`docs/` 有 720 个文件，覆盖：

- 渠道接入文档。
- CLI 子命令文档。
- 模型、会话、上下文、流式输出等概念文档。
- 自动化、Web、Gateway、节点、平台安装说明。
- 多语言文档与术语表。

如果某个模块职责不够明确，先查 `docs/` 常常比直接读全量源码更快。

### 7.4 `test/` 与测试辅助目录

除了根目录 `test/` 外，源码里还有大量 `*.test.ts`、`apps/*/Tests`、`src/test-utils`、`src/test-helpers`、`extensions/test-utils`。这说明项目测试分布式存在于各模块内部，而不是集中在一个地方。

这类结构的优点是测试贴近实现，缺点是新人第一次找测试入口会比较分散。

## 8. 读代码时最重要的几个认知

### 8.1 先分清“平台主干”和“插件包装”

很多同名模块会在两个地方同时出现：

- `src/telegram` 与 `extensions/telegram`
- `src/slack` 与 `extensions/slack`
- `src/discord` 与 `extensions/discord`

前者通常是协议/运行时核心实现，后者通常是给插件系统、装载系统或打包系统暴露的插件边界。改行为逻辑时大多要看 `src/`；改启用方式、插件清单或插件打包时要看 `extensions/`。

### 8.2 这是“控制平面 + 多终端”的产品，不是单端 App

最关键的执行路径通常是：

1. CLI 或客户端发起操作。
2. `src/gateway` 作为控制平面调度。
3. `src/agents` 调模型、工具、记忆、权限。
4. `src/channels` 和渠道模块把结果发回外部世界。
5. `apps/*` 或 `ui/` 负责可视化与设备侧能力。

所以很多问题不能只在某个前端或某个插件里看，必须沿调用链向上回溯。

### 8.3 体量最大的模块值得优先建立局部地图

按文件数看，最值得优先单独建立更细文档的模块是：

1. `src/agents`
2. `src/infra`
3. `src/commands`
4. `src/gateway`
5. `src/cli`
6. `src/config`

这些模块决定了系统行为上限、调试成本和后续扩展成本。

## 9. 建议的阅读顺序

如果你的目标是快速理解 OpenClaw，建议按下面顺序读：

1. [src/index.ts](/Users/wayne/Downloads/Agent/openclaw/src/index.ts)：确认程序入口和初始化流程。
2. `src/cli` + `src/commands`：理解命令如何映射到具体能力。
3. `src/config`：理解配置模型、agent 目录和渠道声明方式。
4. `src/gateway`：理解控制平面和连接模型。
5. `src/agents`：理解模型调用、工具调用、审批、上下文和失败切换。
6. `src/channels` + 一个具体渠道模块：理解消息如何进入系统。
7. `extensions/*`：理解插件如何把能力挂到平台上。
8. `ui` 或 `apps/macos`：理解最终产品界面如何消费 Gateway。

## 10. 一句话结论

OpenClaw 的代码组织核心思想是：以 `src/` 维护统一控制平面与 Agent 运行时，以 `extensions/` 做能力插件化，以 `apps/` 和 `ui/` 提供真实产品终端，再用 `scripts`、`docs`、`skills` 和大量测试把整个系统拉到可持续演进的工程状态。
