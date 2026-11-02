# 能力边界矩阵（三分法）· 可直接搬

> 来源：ccch1mneyyy/dsh-Go @ `8fe1c1cd804ad763dd357bc1dec4b2cbfdc6f7fa` + DSH 0.2.1-rc.2 app.asar 只读。
> 全部为**源码静态分析**结论，没有一条是实机验证（见 30-dshgo-native.md 12 未验证清单）。
> 三列定义：**A = DSH 网页端插件**（纯 DSH API，跑在引擎里）；**B = DSH-Go 原生界面**（额外写 `dshGo` 清单 + `ctx.dshGo`）；**C = Android 系统级**（App 自己实现，插件不可达）。

## 0. 一句话

- **A 是「引擎里能做什么」**：随便写，跑在 Node，能读写文件、联网、调模型、调官方 ASR。但它看不见手机。
- **B 是「App 界面能长什么样」**：只能交 JSON 组件树 + 效果白名单，App 用自己的组件画。它是**窄的、有界的、被 App 审核过的**。
- **C 是「Android 能给什么」**：前台服务、无障碍、跨 App 悬浮窗、麦克风、通知、系统选择器。**这些没有任何一条暴露给插件。**

## 1. 主表

| # | 能力 | A · DSH 网页端插件 | B · DSH-Go 原生界面 | C · Android 系统级 | 证据 |
|---|---|---|---|---|---|
| 1 | 注册工具 | ✅ `defineTool` | ✅ 工具卡：`presentationMeta` 放 `"dsh-go/view"` | — | `docs/plugins/native-for-free.md`；`sdk/plugin-kit/lib/card.js`；asar `dsh-tools/lib/types/schema.js` |
| 2 | 斜杠命令 | ✅ `ctx.commands` | ✅ App 原生画 `/` 菜单（不需写任何 App 代码） | — | `native-for-free.md` 表 |
| 3 | 技能 | ✅ `SKILL.md` | ✅ App 原生画「技能」页 | — | 同上 |
| 4 | 插件配置表单 | ✅ 导出 schemastery `Config` | ✅ App 原生表单（`plugin-config` 路由补全非 volatile 字段） | — | 同上；`runtime/bridge/dsh-go-bridge/lib/plugin-config.js:6` |
| 5 | 让模型画图表/卡片 | ✅ 教模型输出 `dsh-ui` 代码块 | ✅ App 原生画（v1 前 10 组件） | — | `native-for-free.md` 「dsh-ui 代码块」 |
| 6 | 工具结果画成原生卡片 | ❌（网页版只有通用卡） | ✅ `presentationMeta` + `toolCard()`，≤ 8 KiB | — | `docs/plugins/quickstart.md`；`app/lib/ui/chat/tool_view.dart:16` |
| 7 | 输入框上方胶囊 | ❌ | ✅ `composer.actions`，每插件 2 个，标签 ≤ 10 字，胶囊总数 ≤ 8 | — | `docs/plugins/slots.md`；`native_model.dart` `NativeLimits` |
| 8 | 消息长按菜单项 | ❌ | ✅ `message.actions`，每插件 2 个，标签 ≤ 12 字，按插件名分组 | — | 同上；`message_plugin_menu.dart` |
| 9 | 输入框上方状态条 | ❌ | ✅ `composer.strip`，每插件 1 个，`text` ≤ 24 字，多插件轮流 | — | 同上；`composer_strip.dart` |
| 10 | 原生面板（任意组件树） | ❌ | ✅ `panels`，每插件 4 个，视图 ≤ 32 KiB / 120 节点 / 8 层 | — | 同上；`native_panel.dart` |
| 11 | 面板落点 | ❌ | ✅ 三个：`session.tab` / `features` / `chat.panel` | — | 协议 §1.2 |
| 12 | 实时推数据到面板 | ❌ | ✅ `panel({data})\` → WS `patch`，≤ 4 Hz，≤ 32 KiB | — | 协议 §4.3；`core.js` `Feed` |
| 13 | 视图里放输入控件/表单 | ❌ | ✅ 仅视图 **v2**：24 个输入/结构组件，`form.submit` 快照 ≤ 4 KiB | — | `docs/plugins/inputs.md` |
| 14 | 点按 → 让 App 做事 | ❌ | ✅ 8 种效果：`insert` `command` `open` `link` `toast` `copy` `refresh` `patch`；3 秒窗口；每次 ≤ 4 个 | — | `docs/plugins/actions.md`；协议 §5 |
| 15 | 慢活（模型/网络）→ 延后效果 | ❌ | ✅ {pending, result}，1–60 s（缺省 30 s），每插件同时 ≤ 2；只能用 `insert/copy/toast/patch` | — | 协议 §4.7；`effects.deferred()` |
| 16 | 读输入框 / 剪贴板 / 被长按的回复 | ❌ | ✅ wants: ["draft"｜"clipboard"｜"message"]，各 ≤ 16 KiB；App 首次点按前问一次 | — | 协议 §1.4 |
| 17 | 把文字塞进输入框（不发送） | ❌ | ✅ `insert` 效果，1–2000 字，**永不自动发送** | — | 协议 §5 |
| 18 | 直接发送消息 / 批准工具调用 / 改权限模式 | ❌ | ❌ **永不开放** | — | `docs/plugins/actions.md` 「永远不开放」；`native_effects.dart` 里 `permission` 命令被显式拒绝（L199–202） |
| 19 | 手机操控（读屏、点按） | ❌ | ❌ **永不开放** | ✅ 无障碍服务 + 特权通道（root/Sui/Shizuku/自连 ADB） | 协议 §5；`AndroidManifest.xml:246` `PhoneA11yService`；`docs/design/privileged-paths.md` |
| 20 | 装卸插件 / 切主机 / 读凭据 | ❌ | ❌ **永不开放** | — | 协议 §5 |
| 21 | 自定义绘制 / 动画 / 字体 / 圆角 / 几何 | ❌ | ❌ 全部不开放；只有 v2 有界样式（6 字号 / 4 字重 / 命名色 / accent / 间距档 / 比例列），且颜色由 App 校正对比度 | — | `docs/plugins/design-rules.md`；`docs/plugins/styles.md` |
| 22 | 外部图片 / SVG / 文字里的链接 | ❌ | ❌；只能 `asset:`（清单 `assets` 目录内 png/jpg/webp/gif）或 ≤ 64 KiB 的 `data:` | — | 同上；协议 §4.6 |
| 23 | 审批卡 / 授权按钮 / 密码框 | ❌ | ❌ 永远没有 | ✅ App 自己的审批 UI | `design-rules.md` 「不能自定义的」 |
| 24 | 播放一段音频 | ❌ | ✅ v1.2 `audio` 能力：插件在**主机上**生成字节（≤ 16 MiB），App 从同源媒体路由取回并在手机上播；不支持流式 | ✅ `MediaPlayer`（App 实现，插件不能指定除音频内容外的任何 Android 行为） | 协议 `11.3/`11.4；`ClipPlayer.kt` |
| 25 | 让 AI 朗读（TTS） | ⚠️ DSH 官方**没有 TTS**（`speechToText` 只做转写）；要 TTS 得插件自己在主机上接一个（例：主机上的 `127.0.0.1:8765`） | ✅ 播放这条链路（⏵ 见 #24）；**不提供 TTS 服务** | — | asar `dsh-experimental-speech-to-text/README.zh.md`：「服务没有流式识别或语音合成方法」 |
| 26 | 语音转文字（ASR） | ✅ **DSH 官方能力**：`ctx.speechToText` + `sensevoice-local`（SenseVoiceSmall INT8 / sherpa-onnx）+ `speech/catalog｜prepare｜transcribe` | ❌ NA 协议**没有 ASR**；`voice` 能力只给「开口/说完」事件，**不给文字** | ✅ 系统 `SpeechRecognizer`（App 封装，国产 ROM 多半不可用）；麦克风 `RECORD_AUDIO` | asar `dsh-experimental-{speech-to-text,-api-speech-to-text,voice-input-bundle}`；`speech_api.dart`；`docs/design/voice-input.md`；`VoicePlugin.kt` |
| 27 | 「用户一开口就停播」（打断） | ❌ | ✅ v1.2 `voice` 能力：**App 在本地先停掉同一实例的声音，再**发 `voice.speechStart`；插件是被通知 | ✅ 实现：`AudioRecord(VOICE_COMMUNICATION)` + `AcousticEchoCanceler` + 能量门限 `SpeechGate` | 协议 §11.4；`VoiceCapture.kt`、`SpeechGate.kt`、`NativeMediaPlugin.kt` |
| 28 | 一行字幕（面板内） | ❌ | ✅ v1.2 `micro` 通道：{text?, progress?} ≤ 240 字，≤ 10 Hz；面板加 `micro` 字段就在视图下多一行 | — | 协议 §11.5；`native_micro_caption.dart` |
| 29 | 一行「悬浮字幕」 | ❌ | ✅ v1.2 `overlay`：**App 自己 Activity 里的 View**，只在 App 内、对话顶栏下；**不需要窗口权限**；退到后台/锁屏立刻停 | — | `MediaOverlays.kt` 文件头：「show over the app (never over other apps) … it is the app's own view in the activity … needs no window permission」 |
| 30 | 在其他 App 上面显示任何东西 | ❌ | ❌ **插件完全没有这个 API** | ✅ 两条 App 自有通道：① 手机操控悬浮条（`TYPE_ACCESSIBILITY_OVERLAY`，`phone/a11y/`）② 宠物 App 外悬浮窗（`SYSTEM_ALERT_WINDOW` / 借用无障碍 overlay） | `docs/design/phone-overlay.md:3`、L127；`AndroidManifest.xml:34`；`OverlayAccess.kt:23` |
| 31 | 桌宠 / 宠物皮肤 | ❌（`dsh-pet` 是网页插件，只能走网页岛） | ❌ **现在不行**：宠物原生适配器是空壳，protocols = <String>{}，`canRender` 恒 false；宠物包是 P3 计划 | ✅ 内置两只（Deepy 小鲸鱼 / 鲸娘）+ 站台/气泡/App 外悬浮窗，全部由 App 实现 | `app/lib/plugins/seams/adapters/pet_adapter.dart:15`；`docs/design/pets.md` |
| 32 | 系统通知 | ❌ | ❌（v1 没有 `notify` 效果，属计划） | ✅ `POST_NOTIFICATIONS` + 通知 channel + 引擎常驻通知 | `AndroidManifest.xml:9`；`notify/NotifyPlugin.kt` |
| 33 | 系统文件选择器（读用户选的文件） | ❌ | ✅ v1.3 `files` 能力：`pickFile`/`pickImage`，单文件 ≤ 8 MiB、单次 ≤ 16 MiB、1–4 个；一次性只读句柄 | ✅ `ACTION_OPEN_DOCUMENT` / `MediaStore.ACTION_PICK_IMAGES`（API 33+），只请求临时读授权 | 协议 §12；`docs/plugins/files.md`；`NativeFilesPlugin.kt:90,114` |
| 34 | 后台继续跑 | ⚠️ 引擎在后台**可能被系统结束** | ❌ 插件的声音/字幕在后台**一定停**（`revoke("background")`） | ⚠️ 前台服务 `specialUse` + `PARTIAL_WAKE_LOCK`，但 **`START_NOT_STICKY`**、开机不自启；FAQ 明说「各品牌限制不同，做完这些仍可能被系统结束」 | `DshService.kt:118,157`；`NativeMediaPlugin.kt:97`；`docs/guide/zh/11-faq.md:20` |
| 35 | 麦克风 | ❌ | 只能**间接**：`voice.start` 让 App 听「开口」，App 自己不拿音频内容，插件也拿不到 | ✅ `RECORD_AUDIO`（点麦克风时才申请）；App 内单麦克风仲裁 `MicArbiter`（语音输入优先） | `AndroidManifest.xml:17`；`MicArbiter.kt` |
| 36 | 常驻引擎（不停机） | — | — | ✅ 前台服务 `DshService` 跑 proot 里的 dsh web；`FOREGROUND_SERVICE_SPECIAL_USE` | `AndroidManifest.xml:226`；`docs/architecture.md` |
| 37 | 网页界面（整页/面板 WebView） | ✅（就是网页版） | ⚠️ 网页插件在 App 里只能做成**网页岛**（WebView），第一个岛约 200 MB、就绪 0.8–3.4 s（**模拟器自述实测，非真机**）；**不是原生** | ✅ `webview_flutter` | `island_view.dart:9`；`docs/design/native-plugins.md` 2.3 |
| 38 | 跨主机能力 | ✅ 任何主机 | ⚠️ 只有**装了桥接**的主机：本机引擎 ✅；SSH 主机「第二期」（桥接还不能装）；**中转（DSH Remote）主机 ❌，只有工具卡能用** | — | `docs/plugins/hosts.md` 表；`dsh_go_api.dart:640` canReachDshGo => carrier is DshHttpCarrier |

## 2. 项目上最容易踩的四个等号（都是错的）

| 错误等号 | 为什么错 | 正确说法 |
|---|---|---|
| 网页插件 = Android 原生桌宠 | dsh-pet 是**网页插件**，在 App 里只能走网页岛（WebView）；原生桌宠只有 App 内置的两只；第三方宠物包是 P3 计划、适配器是空壳 | 「网页桌宠在 App 里是嵌在 WebView 里的网页；想原生显示目前只能选内置那两只」 |
| App 内面板 = Android 系统级悬浮窗 | 面板是 App 页面里的一个 widget；NA `overlay` 字幕也是 App Activity 里的一个 View（need no window permission）；系统级悬浮窗是另一套（`SYSTEM_ALERT_WINDOW` / `TYPE_ACCESSIBILITY_OVERLAY`），**插件碰不到** | 「插件只能在 App 内部画；跨 App 显示是 App 自己的两个通道，插件无权限」 |
| 插件能调 TTS / ASR | NA 协议里既没有 TTS 也没有 ASR。TTS 得插件自己在主机上做；ASR 是 DSH 官方 `speechToText` 服务（插件可以用，但那是 A 列不是 B 列） | 「朗读 = 插件主机侧 TTS + NA `audio` 播放；转写 = DSH 官方 ASR，与 NA 无关」 |
| Android 后台能无限跑 | 引擎是 `START_NOT_STICKY` 前台服务、开机不自启；FAQ 明说「仍可能被系统结束」；插件音频在后台被强制 `revoke("background")` | 「后台不稳：厂商会杀；插件的声音和字幕在后台一定停」 |

## 3. 一图流

```
┌─ A · DSH 网页端插件（引擎内，Node）────────────────────────────┐
│ 工具 / 命令 / 技能 / Config / dsh-ui 围栏 / 任意 I/O / 官方 ASR │
│ 看不见手机                                                  │
└───────────────┬────────────────────────────┬────────────────┘
                │ ① 进程内方法调用             │ presentationMeta
                │    ctx.dshGo.*              │ （只要工具卡就不用桥接）
┌───────────────▼────────────────────┐       │
│  dsh-go-bridge（同进程 cordis 插件）│       │
│  HTTP + WS  →  /dsh-go/v1/*        │       │
└───────────────┬────────────────────┘       │
                │ ② 本机回环 127.0.0.1:port   │
┌───────────────▼────────────────────┐       │
│ B · DSH-Go App（Flutter）           │◀──────┘
│ 胶囊/菜单/状态条/面板/工具卡         │
│ 视图 = JSON 组件树（v1 16 / v2 +26） │
│ 效果白名单 8 种 + 3 秒窗口          │
└───────────────┬────────────────────┘
                │ ③ MethodChannel / EventChannel
┌───────────────▼────────────────────┐
│ C · Android 系统级（Kotlin）        │
│ FGS / WakeLock / 无障碍 / 跨 App 悬浮窗 │
│ 麦克风+AEC / 通知 / 系统选择器        │
│ 插件对这些一条都碰不到               │
└────────────────────────────────────┘
```

## 4. 三条「不能假设」

1. **不能假设网页插件搬进 App 就是原生**——它要么是网页岛（WebView），要么得按 NA 协议重写一份声明 + 视图。
2. **不能假设 App 内面板/字幕是系统级悬浮窗**——只有 `phone/a11y/` 的操控悬浮条和宠物的 App 外悬浮窗是跨 App 的，两者都归 App，不归插件。
3. **不能假设 Android 后台能无限跑**——`START_NOT_STICKY`、开机不自启、FAQ 明说会被厂商杀，插件音频在后台强制停。

## 5. 证据索引（本文件用到的）

| 断言 | 文件 / 行 |
|---|---|
| `ctx.dshGo` 只有 10 个方法、无 `turnNote`/`notify` | `runtime/bridge/dsh-go-bridge/lib/native/core.js:425-448` |
| 4 个 surface + 3 个 v1.2 资源 + 3 个 v1.3 能力 | `lib/native/limits.js:52-60` `SURFACES/PLACEMENTS/WANTS`；`lib/native/capabilities.js:7,9,20-23` |
| 8 种效果 + 上限 | `lib/native/effects.js`；协议 §5 |
| 永不开放清单 | `docs/plugins/actions.md` 「永远不开放」；`app/lib/plugins/native/native_effects.dart:199-202`（`permission` 被拒） |
| 插件 overlay 是 App 内 View、无需窗口权限 | `app/android/.../nativemedia/MediaOverlays.kt:1-40` |
| 跨 App 悬浮窗两条通道 | `AndroidManifest.xml:34` `SYSTEM_ALERT_WINDOW`；`pet/OverlayAccess.kt:23` `TYPE_APPLICATION_OVERLAY`；`docs/design/phone-overlay.md:3,127` |
| 宠物适配器是空壳 | `app/lib/plugins/seams/adapters/pet_adapter.dart:2,15` |
| TTS 不存在 | asar `dsh-experimental-speech-to-text/README.zh.md`「已知问题」；306 个 @deepseek-ai 包 grep 无 TTS |
| ASR 接口 | asar `dsh-experimental-speech-to-text/lib/types/index.js`；`packages/dsh_client/lib/src/speech_api.dart` |
| 后台不 sticky + 插件后台停 | `runtime/DshService.kt:118,157`；`nativemedia/NativeMediaPlugin.kt:97`；`docs/guide/zh/11-faq.md:20` |
| 中转主机不支持 | `packages/dsh_client/lib/src/dsh_go_api.dart:640`；`docs/plugins/hosts.md` |
| WebView 只用于网页岛 | `app/lib/ui/plugins/islands/island_view.dart:9` |
