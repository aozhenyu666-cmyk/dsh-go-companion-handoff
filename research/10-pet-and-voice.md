# 10 · 桌宠与语音：源码审计报告

> 审计对象与锁定 commit：
> - 桌宠 A：`zhu1090093659/dsh-pet` @ `9ed843bccee3d2604ce0ac31f55cbde9bda97c7f`（下称 zhu 版）
> - 语音主选：`qishuilalala/dsh-voice-mode` @ `3524db40938f16e7c8f0726e30647afe5bfbef25`（下称 voice-mode）
> - 语音备选：`PerryLink/dsh-talk` @ `6a5f768b2607ec1204fc256ede492c2abd39a3e1`（下称 dsh-talk）
> - 对照组（本地已装）：`dsh-pet@0.3.8`（`PC2005-cloud/dsh-pet`，源码在 `~/.dsh/profiles/desktop/node_modules/dsh-pet/`）、surfcore 桌宠 `feiyu-pet@0.2.3`（`~/.dsh/surfcore/pet/app`）、DSH-Go 原生宠物（`ccch1mneyyy/dsh-Go` @ `8fe1c1cd804ad763dd357bc1dec4b2cbfdc6f7fa`）
>
> **性质声明**：本报告全部结论均来自源码静态阅读。没有安装、没有启动、没有一次真机/真麦克风/真模型运行。凡"源码已证实存在"不等于"运行验证通过"，逐条标 PASS/WARN/FAIL/NOT_TESTED 时以静态证据为上限。

---

# 一、桌宠：zhu 版 dsh-pet

## 1.1 版本与血统（先回答"是不是 fork"）

| 项 | zhu 版 | 本地已装 0.3.8 | PC2005 仓库现行 |
|---|---|---|---|
| package name | $linxin666/dsh-pet | dsh-pet | dsh-pet |
| version | **0.4.14** | 0.3.8 | **0.4.1**（`dsh-pet/package.json`） |
| repository | github.com/zhu1090093659/dsh-pet | github.com/PC2005-cloud/dsh-pet | 同左 |
| license | Apache-2.0 | MIT | MIT |
| 宿主门槛 | `dsh.engines.dsh: ">=0.2.1-rc.3"` | 无 engines.dsh，用 `dsh.compatibility.dshReleases` | dshReleases: 0.2.1-rc.2 / rc.3 / 0.2.2-alpha.2 全 compatible |
| 渲染器 | sprite2d / frames2d / live2d | webm `<video>` 双缓冲 | webm + Electron 独立窗 |
| 素材 | 3536 个 `.webp`，0 个 webm | 106 个 `.webm`（约 51.8 MB） | 同左 |

**结论：不是 fork，是两条独立血统。**
- zhu 版自述"宠物插件迁出 dsh-web 主仓后的首次独立发布"（`docs/release-notes/v0.3.25.md`）：它是 linxin666 的 dsh-web 家族卫星包，包名带 scope、Apache-2.0、注册表驱动的多宠 + Live2D 体系。
- 本地 0.3.8 / PC2005 仓库是另一条：包名无 scope、MIT、素材"允许开源使用，禁止商用"（`~/.dsh/profiles/desktop/node_modules/dsh-pet/README.md:305`）、webm 视频播放 + Electron 透明独立窗。
- 两者唯一相同的是"DSH 桌宠"这个定位；代码、素材格式、许可证、宿主接口全不同。**不能把 zhu 版当成 0.3.8 的升级包。**

**红线：zhu 版装不进当前宿主。** `package.json` 声明 `dsh.engines.dsh: ">=0.2.1-rc.3"`、`peerDependencies["@deepseek-ai/dsh"]: ">=0.2.1-rc.3"`，本机 Desktop 是 **0.2.1-rc.2**（00-SHARED-FACTS.md 第 1 节）。因此它只能作研究样本，不能替换现装 0.3.8。

## 1.2 Live2D 与精灵动画的真实实现

三个渲染器统一走一份契约：`src/contracts/renderer.ts:15` 声明 `PET_RENDERER_API_VERSION = 'x-org.linxin666.pet-center/v1alpha1'`，渲染器只能拿到 `{ petId, assetBase, container, phase, interact, onCleanup }`——不许碰 session、registry、网络。注册表 `src/client/renderers/registry.ts` 按 `kind` 分发，未知 kind 渲染诊断卡而非白屏。

清单侧：`src/manifest-v2.ts:34` `PET_RENDERER_KINDS = ['sprite2d','live2d','frames2d']`；v1 清单兼容读为 sprite2d（`:753` 警告 "run scripts/dsh-pet-migrate-v2"）。

### sprite2d（图集）——纯 CSS，不用 canvas
- `src/client/spritesheet.ts`：9 行 × 8 列 × 192×208 单元，`scaledCellSize()` 把单元尺寸取整到整像素，`framePosition()` 返回 `background-position` 偏移（`-col*scaledW, -row*scaledH`）。注释写明非整数单元会导致每帧重采样。
- 行序单一真源在 `src/state.ts:rowOf()`：idle 0 / running-right 1 / running-left 2 / waving 3 / jumping 4 / failed 5 / waiting 6 / running 7 / review 8。
- 挂载方式：`src/client/PetSprite.tsx` 用 React `createPortal` 到 `document.body`，样式来自 CSS Module `src/client/pet.module.css`。没有 canvas、没有 WebGL。

### frames2d（自由帧序列）——canvas 优先、img 兜底
- `src/client/renderers/frames2d.ts` 头注释：默认走 `createImageBitmap` + 单个 `<canvas>`，有 `MAX_RETAINED_BYTES` 预算，稳态零 DOM 变更；jsdom / 缺 API 时退回 `<img>.src` 切换。1.2s 卡顿看门狗会重踢播放链。
- 清单块支持 `tracks / phases / skins`（换肤）+ `gameplayTracks`（玩法态覆写轨道）。`handle.setState(track)` 让玩法层强占轨道，非 loop 轨道播完落回 fallback。

### live2d——是 Cubism Web 运行时，不是自研裁帧
这是 lead 要求确认的点，答案明确：

- `src/client/renderers/live2d/runtime.ts:18-19`：
  ~~~
  const CORE_URL = '/api/pet/runtime/live2dcubismcore.min.js'
  const VENDOR_URL = '/api/pet/runtime/live2d-vendor.js'
  ~~~
  两个脚本都靠 `document.createElement('script')` 注入（`:64-70`），全局是 `window.Live2DCubismCore` 与 `window.__dshPetLive2d`。
- **Cubism Core 由用户自备**：`src/routes.ts` 的 `RUNTIME_FILES` 白名单里 `live2dcubismcore.min.js` 从 `$DSH_HOME/pets/.runtime` 读，缺文件返回 `404 {error:'runtime-file-missing'}`；README.zh.md:149 写死"本插件永不内置、永不代下 Cubism Core 运行时——Live2D 专有许可不允许再分发"。
- vendor 侧是 MIT 栈：`tsdown.live2d-vendor.ts` 把 pixi.js + untitled-pixi-live2d-engine 打成自包含 IIFE `lib/live2d-vendor.js`（`format:'iife'`, `platform:'browser'`, `noExternal:[/.*/]`），`@pixi/sound` 被显式打桩替换为抛错（口型同步/动作音效不在 M3 范围）。
- 渲染侧 `src/client/renderers/live2d.ts`：`vendor.extensions.add(vendor.Live2DPlugin)` + `vendor.configureCubismSDK({ memorySizeMB: 32 })`，`Live2DModel.from(modelUrl)`，相位→动作组 `motions: Partial<Record<ActivityPhase,string>> & {idle:string}`，未映射一律回落 idle，点按命中区播 `TapBody`。纹理 LOD 用 `single-auto`，销毁用 `{removeView:true}`。

**所以 Live2D 这条对 Android 移植是死路**：它 = Cubism Web SDK（专有核心，用户自备）+ WebGL(canvas) + pixi + DOM script 注入。Android 端只能换 Cubism Native SDK 重做，核心授权问题原样存在。

### 渲染载体与 Electron
- 服务端声明 `dsh.client.platform: "web"`（package.json），客户端注入清单：dsh-client-connection / ui-settings / ui-conversation / api-session-controller / ui-workspace。
- 挂载点：`src/client/index.ts:629-633` 自己造 `div[data-dsh-pet-root]` 塞进 `document.body`，再 `createRoot`。注释解释了为什么不用 slot：官方 rc.6 shell 没有根级浮层 slot，dock 是会话级的，挂上去在新会话页会消失（issue #48）。
- **zhu 仓库里没有任何 Electron / standalone 目录**：3831 个 blob 中 `electron|standalone|runtime/` 命中 0。zhu 版是纯 Web 浮层。
- 对照：本地 0.3.8 **有** `runtime/electron-helper/`（14 个文件），`main.js` 51KB，做法是"每只宠物一个透明置顶小窗 + 整窗点击穿透 + `protocol.handle('dsh-pet-bridge://')` 走 stdin/stdout JSON 行协议回宿主"。这才是真正的 Electron 独立窗，zhu 版没有。

## 1.3 宠物状态与情绪表情

- `src/state.ts`：`PetStateMachine` 纯状态机、时钟注入。7 个 `ActivityPhase`（idle/waiting/thinking/tool/review/done/failed）→ 9 个动画；done 播 jumping，`celebrateMs=2400` 后落回 idle；failed 同理 `failureMs=2400`；**settled 态不出气泡**（`render()` 里 `bubble = settled ? undefined : phrase ?? line`）。
- `src/contracts/status-decoration.ts`：M5 装饰契约 `PET_DECORATION_API_VERSION = 'x-org.linxin666.pet-center/status-decoration-v1'`，PNG/WebP 横条按 `phases: Partial<Record<ActivityPhase, {from,to}|'hide'>>` 绑帧段，强制 `aria-hidden`，`prefers-reduced-motion` 降级为无装饰。校验在 `src/decoration.ts`（fail-closed 结构 + warn-and-drop 内容），几何用 `src/image-dimensions.ts`（仅读 PNG/WebP 头，无解码、无依赖）。
- `src/affinity.ts`：见 1.4。
- **情绪表达的真实能力边界**：宠物没有"情绪"字段，能表达的"表情"只有三种——(a) 9 条动画轨道（只有 failed 算负向）；(b) 状态气泡/碎碎念文案池；(c) `gameplay.stats` 里的自定义数值条。**没有 dims（happy/calm/sweet…）这种多维情绪模型。**

## 1.4 触摸 / 互动 / 亲密度

- `src/affinity.ts`：`AffinityState = {points,lastPetAt,lastFeedAt,pets,feeds,petRejects,feedRejects,turns}`；`defaultAffinityConfig = { turnReward:1, petReward:1, petCooldownMs:10_000, feedReward:5, feedCooldownMs:30_000 }`；`AFFINITY_MAX = 999_999_999`；9 级阶梯 幼鲸→伙伴→挚友→深海羁绊→心有灵犀→传说羁绊→神话羁绊→永恒之契→鲸生共渡（`AFFINITY_RANKS`），支持宠物自带 `panel.ranks` 改名（`rankOf(points, ladder)`）。
- `src/treats.ts`：小鱼干经济 `{ turnsPerTreat:30, timeTreatMs:300*60_000, maxTreats:20 }`，懒结算（读状态时才结算，无定时器）。
- `src/gameplay.ts`（43KB）：清单声明的养成层——stats（max/initial/decayPerMinute/workingDecayPerMinute/idleDecayPerMinute）、hitBox、touch.zones[].branches[]（概率分支 + 效果 + 状态轨道 + 短语池）、work（tickMs/successProbability/成功失败轨道）、sleep.restore、modes（如洗澡，周期性回 stat）、shop（含抽奖 tiers）、roam、passiveIncome、idleDirector。引擎是纯函数：`settleGameplay`、`applyGameplayEffects`、`rollTouchBranch(zone,rng)`、`rollWorkOutcome`、`touchZoneAt`。所有 verb 显式接时钟与 rng，无定时器。
- 客户端 HUD：`src/client/gameplay-hud.tsx`（数值条 / 商店 / 触摸分区 / work-sleep 循环 / 飘字），通过注入的 verb API 打宿主，2s 轮询兜底。
- 证据样例：`assets/doro/pet.json:46-68` 声明 hunger/mood/energy/affection 四条 stat，mood 是 `{max:100, initial:100, decayPerMinute:0.5}`；`:193-197` `clickBoost: {stat:'mood', min:0, max:3}`；`:232-243` `modes.wash.restore = {stat:'mood', amount:3, intervalMs:1000}`。**宠物内部确实有一个叫 mood 的数值槽，但它是宠物自己的池子，与 feiyu-mood 无关。**

## 1.5 内心独白 / 主动气泡

- `src/chatter.ts`（24KB）两个说话人：
  1. `StatusVoice`：`STATUS_POOLS` 按 11 个 scene（prepare/waiting/thinking/review/toolResult/done/failed/toolFailed/maxTokens/interrupted/blocked）给大池子，`STATUS_ROTATE_MS = 4000` 轮播；`TOOL_POOLS` 按 17 个工具族出文案；`toolArgHint()` 抽真实参数提示（如"跑跑 npm test"）。
  2. `WhisperEngine`（碎碎念）：`WHISPER_COOLDOWN_MS=9000`、`WHISPER_RESULT_COOLDOWN_MS=5000`、`WHISPER_TTL_MS=8000`；`feed(category, nowMs)` / `result(kind, nowMs)`，全部 round-robin，无随机数、无 LLM。头注释明确："The model's own prose is never read, and no whisper ever quotes real content."
- `src/event-projection.ts`：把 DSH 官方会话事件（`turn/start`、`assistant/message`、`tool/call`、`tool/result`、`turn/end`，见 `:103-162`）+ 进程内 `agent/assistant-stream` 帧，投影成 phase/气泡/碎碎念输入，并携带完成回合奖励。`looksLikeTestTool()` 判断是不是测试跑，避免"讨论中提到关键词就开香"。
- `src/announce.ts`：兄弟插件的公告气泡契约，`pet.announce(input)` → 校验成有界 `PetAnnouncement`，`kind` 只接受 `'balance' | 'cost' | 'plan'`，`MAX_ANNOUNCEMENTS = 4`（每 source 一个槽），默认 TTL 10s，上限 `ANNOUNCE_MAX_TTL_MS = 7_200_000`，超界截断，非法丢弃。
- `src/client/RangeBubble.tsx`：**注意——这个文件跟气泡无关**，它是设置面板数字输入的"粗调滑块气泡"（220ms 悬停开启、200ms 关闭、按滚动祖先决定上下方向）。任务清单里说的"主动气泡"实际在 `PetSprite.tsx`（`UsageAnnouncementBubble`，`role="status"` + `aria-live="polite"`，`:219-251`）。
- 分层：`src/voice-pack.ts` 语音包只能覆盖文案池与面板 chrome，纯 JSON，不含音频。

### 与 lead 提供事实的对齐（重要修正）
lead 说"dsh-pet 的碎碎念是自己调 llm，USER_TEXT='随便说一句日常碎碎念'… 一天约 288 次"。**这条对本地 0.3.8 成立，对 zhu 版 0.4.14 不成立**：
- 本地 0.3.8：`src/host/whisper.ts` 头注释"用 DSH 的 LLM 统一抽象层（ctx.llm）"，`generateWhisper()` 走 `llm.stream`；`src/host/chat.ts` 同理（`generateChat()`，chatModel 留空=当前对话模型，回复放宽 256 token，60s 超时）。**会烧模型调用。**
- zhu 版 0.4.14：`src/index.ts` 与 `src/service.ts` 里 `ctx.llm / generate / prompt` 命中 **0**；service 只挂 `session/event`、`agent/assistant-stream`、`session/disposed` 三个监听（`service.ts:641/672/681`）。碎碎念是本地文案池轮播。**不调模型，不烧 token。**

这条差异直接影响选型判断：zhu 版的设计哲学更接近 dsh-Go 的"宠物是状态指示器"，而本地 0.3.8 恰恰是 dsh-Go `docs/design/pets.md` 点名批评的反例。

## 1.6 人格与状态接口（真实 HTTP 路由 + 字段）

路由全部在 `src/routes.ts` 的 `makePetRoutes()`，基址 `PET_API_PREFIX = '/api/pet'`，守卫 `isPetAllowed()`（loopback 恒过，额外允许 paired-device cookie），否则 `403 {error:'forbidden: loopback-only'}`。

| 方法 | 路径 | 请求 | 响应要点 |
|---|---|---|---|
| GET | /api/pet/state?current=<sessionId> | — | PetStateView（见下） |
| GET | /api/pet/pets | — | registry 清单 |
| GET | /api/pet/diagnostics | — | 加载诊断 |
| POST | /api/pet/interact | {kind:'pet'\|'feed'} | LedgerInteractionResult（delta/reaction/accepted） |
| POST | /api/pet/set-visible | {visible:boolean} | 配置 |
| POST | /api/pet/set-config | {size?,right?,bottom?,visible?} | 配置 |
| POST | /api/pet/set-name | {name:string} | 配置 |
| POST | /api/pet/set-skin | {skin?:string} | 配置（frames2d 换肤） |
| POST | /api/pet/set-pet | {petId:string} | 选宠 |
| POST | /api/pet/panel-action | {id:string} | 分发到注册该 action 的兄弟插件 |
| POST | /api/pet/gameplay/touch | {zone?:string} | {hit?,state?,stateMs?,phrase?,view?} |
| POST | /api/pet/gameplay/mode | {mode:string\|null} | 同上 |
| POST | /api/pet/gameplay/work-tick | — | {outcome?:'success'\|'fail', view?} |
| POST | /api/pet/gameplay/buy | {item:string} | {prize?,view?} |
| GET | /pet/<id>/...（prefix） | — | 资产，realpath 包含校验 + 上限（manifest 64KiB / image 20MiB / model 32MiB）+ 弱 ETag 304 |
| GET | /api/pet/runtime/<name>（prefix） | — | 仅白名单 3 个文件：live2dcubismcore.min.js、live2d-vendor.js(.map) |
| GET | /api/pet/decoration/<id>/<file>（prefix） | — | 装饰条，按描述符声明白名单放行 |

`PetStateView`（`src/service.ts:191-275`）：
~~~
animation, bubble?, phase, sessionActive,
sessions?: [{ sessionId, animation, bubble, phase, whisper? }]   // 最多 MAX_SESSION_BUBBLES=12
affinity: { points, rank, rankEn?, rankEmoji, pets, feeds, turns, petCooldown, feedCooldown }
display: { visible, size, right, bottom, bubbleScale }
pet: { id, displayName, description }, name, skin?,
treats: { stocked, max },
decoration?, gameplay?, announcement?, announcements?, statusBubbles?, panelActions?
~~~
另外宿主侧暴露一个 cordis 服务：`declare module '@deepseek-ai/cordis' { interface Context { pet: PetService } }`（`service.ts:303-307`），兄弟插件可 `ctx.pet.registerPanelAction({id,label,onSelect})`（`src/panel-actions.ts`，最多 8 个）。**注意：registerPanelAction 只加按钮，不能写数值、不能读 mood、不能塞气泡文案（announce 的 kind 是枚举）。**

## 1.7 能否消费 dsh-feiyu-mood —— 结论：不能，必须新增桥接

**pet 侧现状（双向确认）**：
- zhu 版：`src/service.ts`、`src/index.ts`、`src/routes.ts`、`src/contracts/*`、`README.zh.md` 里 `mood|feiyu|心情|情绪` 命中 **0**（唯一一处"心情"是 README 描述 doro 宠物洗澡模式 +3 心情，指宠物自己 gameplay 的 mood stat）。
- 本地 0.3.8：`inject` 只有 `['@deepseek-ai/dsh-client-connection']`，`src/` + `runtime/` 全量 grep `mood|feiyu` = **0**。

**feiyu-mood 侧的实际接口（本地 0.8.1 读盘）**：
- 状态文件 `~/.dsh/feiyu-mood/state.json`：`{version:2, moods:{<slotId>:{mood:0..100, dims:{happy,calm,sweet,heartache,worry,sad,jealous}, lastNotedMood, lastReason, lastConfidence, updatedAt, timeline[], saidAt[]}}, lastActive}`（`lib/index.js:776-806`；七维 DIMS 见 `:302`）。
- **它已经开了一个只读 HTTP 路由** `/dsh-feiyu-mood/state?key=<slot>`，返回 `{ok, key, mode, slots[], label, name, ownerName, mood, moodLabel, emoji, dims:[{key,label,value}], lastReason, lastConfidence, updatedAt, injectMode, saidToday, injectQuota{said,cap,left}, decayHalfHours, sigMood, sigDim, minor, kind, injectable, lastInjectedText, lastInjectedAt}`（`lib/index.js:1619-1700`）。
- 也就是说：**心情侧早就把数据摆出来了，缺的是宠物侧的消费者。**

### 最小桥接改造（按代价从小到大）

**方案 A（推荐，宿主侧只读，约 30 行）**：zhu 版新增 `src/mood-bridge.ts`，在 `PetService.state()` 里多返回一个字段。
1. 新文件 `src/mood-bridge.ts`：`readFeiyuMood(home = dshHome())` → 直接读 `$DSH_HOME/feiyu-mood/state.json`（纯 fs + JSON，无依赖），按 `lastActive` 取当前 slot，输出 `{ mood: 0..100, label, dims: Record<7维, number>, updatedAt }`，文件缺失/损坏返回 `undefined`（fail-open，宠物不掉线）。
2. `src/service.ts:191` 的 `PetStateView` 增加 `mood?: PetMoodView`；在 `state()` 的返回对象（约 `:1120`）里 `...(mood === undefined ? {} : { mood })`。读盘要加 1-2s 进程内缓存，否则 2s 轮询会把磁盘读穿。
3. `src/client/PetSprite.tsx` 取 `snapshot.mood` 决定表情/轨道：mood<30 → failed 轨 + 负向文案池，30..70 → 常规，>70 → jumping/waving；dims 里 sad/worry 高 → 套 sad 类表情图。**这一步是真正需要产品决策的地方**：现有 9 条轨道没有"难过但仍空闲"这一档，要么复用 failed（语义错），要么在 `manifest-v2.ts` 的轨道表里新增 mood 分段（改动大）。
4. 或者复用宠物自己的 stat 槽（更省事但语义绕）：给 `PetService` 加外部写入口 `setStat(petId, stat, value)` + 路由 `POST /api/pet/gameplay/stat`，让 feiyu-mood 或一个 5 行适配插件把 mood 写进 `gameplay.stats.mood`。改动面：`src/routes.ts`（加 1 个 postRoute）+ `src/service.ts`（1 个方法，内部调 `applyGameplayEffects`），客户端 `gameplay-hud.tsx` 免费得到一条情绪条。**代价最低，但只能表达单值，7 个 dims 进不来。**

**方案 B（不推荐）**：扩展 `src/announce.ts` 的 `kind` 枚举，让 feiyu-mood 主动 push 一句"我现在有点难过"。缺点：announce 是 TTL 气泡不是状态，会把情绪降格成一次性通知，且要改 3 处枚举（announce.ts 的 type、parse、service 的聚合）。

**为什么必须新增、不能"已经有"**：两边数据形状完全不同——feiyu-mood 是"1 个 0-100 基调 + 7 个维度偏移 + 时间衰减"，pet 是"9 条动画轨 + 若干文案池 + 可选自定义 stat"。映射表本身要人写，不是接线就完事。

## 1.8 三个问题

### 问题 1：它能不能当大肥鱼的视觉外壳？

**能当"外壳"，但不能当"大肥鱼的外壳"，而且当前形态不建议直接上。**

1. **技术上能接管视觉**（源码已证实）：zhu 版完全注册表驱动——加一只宠物 = 一个目录 `$DSH_HOME/pets/<id>/` 里放 `pet.json` + 图集/帧序列，宿主零改动（`src/registry.ts` 头注释，四来源扫描：包内 assets/ → ~/.codex/pets → $DSH_HOME/pets → PetConfig.pets）。所以"大肥鱼"可以作为一只新宠物装进去，改 pet.json 的 petId 即可。
2. **但"大肥鱼"的素材不在 zhu 版里**：它的内置宠物是鲸鱼娘（whale/whale-refined）、deep-chan、doro、miku、jyn 系列、蓝喉蜂虎（blue-throated-bee-eater）等 12 套，全是 `.webp` 图集/帧序列。真正的大肥鱼素材在另外两处：本地 0.3.8 的 106 段 `.webm`（51.8MB，VP9-alpha），和 surfcore 的 `feiyu-01..05` 五套分层 PNG rig（19-21 层骨骼，带 blink/mouth/tail/headPivot）。三种素材格式互不兼容（webm ↔ 图集 ↔ 分层 rig）。
3. **它表达的"情绪"太薄**：只有 9 条轨道，其中只有 failed 是负向。feiyu-mood 有 7 个情绪维度。直接把大肥鱼塞进去，结果就是"心情 20 分的鱼还在跳庆祝舞"——**这不是外壳问题，是表达力不够**。

**判定：WARN**。作为视觉容器成立（注册表 + 三渲染器 + 公告气泡 + 亲密度都是现成的），作为大肥鱼的表达载体缺少情绪层；本地 0.3.8 反而在"表情/动作丰富度"上更强（106 段视频 + 碎碎念 + 余额气泡 + 桌面独立窗）。且 zhu 版装不进当前 0.2.1-rc.2 宿主（engines 门槛 rc.3）。

### 问题 2：哪些逻辑可以直接复用（文件级清单）

判定口径：**无 DOM、无 cordis、无 Electron、无浏览器 API 的纯函数模块**才算"可直接复用"。全部逐文件读过 import 头。

| 文件 | 复用度 | 说明 |
|---|---|---|
| src/state.ts | **直接复用** | 纯状态机，时钟注入，7 phase → 9 animation，零依赖 |
| src/affinity.ts | **直接复用** | 纯，接 remarks.ts（同为纯）；9 级阶梯 + 冷却 |
| src/treats.ts | **直接复用** | 纯懒结算经济 |
| src/gameplay.ts | **直接复用** | 43KB 全纯：解析/结算/概率/抽奖，rng 与时钟显式注入 |
| src/remarks.ts | **直接复用** | 纯文案池 + round-robin |
| src/chatter.ts | **直接复用** | StatusVoice + WhisperEngine 全纯，时钟注入，无 LLM |
| src/voice-pack.ts | **直接复用** | 纯 JSON 归一化/合并 |
| src/announce.ts | **直接复用** | 纯校验，有界 payload |
| src/panel-actions.ts | **直接复用** | 纯注册表 + id 文法校验 |
| src/contracts/renderer.ts、status-decoration.ts | **直接复用** | 纯类型 + 常量 |
| src/client/sequences.ts | **直接复用** | 纯时序（只 import type） |
| src/client/spritesheet.ts | **直接复用** | 纯几何（scaledCellSize/framePosition/trimTrack），输出数字不是 CSS |
| src/client/phase-stream.ts | **直接复用** | 纯 pub/sub |
| src/client/work-tick-gate.ts | **直接复用** | 纯节流门 |
| src/event-projection.ts | **复用（需换类型源）** | 逻辑纯，但 import 了 dsh-agent 的 AssistantStreamFrame 与 dsh-session 的 SessionEvent 类型，Android 端要按自家事件类型重写这两个 type 别名 |
| src/ledger.ts | **复用（需换持久化）** | 逻辑纯（拼 affinity+treats+remarks），但 import persist.ts 的类型；换存储层即可 |
| src/persist.ts | **半复用** | 形状/校验/迁移逻辑可抄，`node:fs` + renameSync 必须换成平台文件 API |
| src/decoration.ts | **半复用** | 校验逻辑纯，但 import { isAbsolute } from 'node:path' 要替换 |
| src/image-dimensions.ts | **半复用** | 仅读 PNG/WebP 头，约 40 行算法可直接搬；Buffer 换 Uint8Array |
| src/model3.ts | **半复用** | Live2D 引用闭包校验，依赖 manifest-v2.safeManifestPath |
| src/manifest-v2.ts | **半复用** | fail-closed 清单校验，纯逻辑可搬，是宠物包格式的真源 |
| src/registry.ts | **半复用（只搬解析）** | 54KB，混了 node:fs 扫描 + 尺寸探测 + 清单解析 + voice/decoration 组装；Android 端只抄"解析 + 组装"，扫描换平台目录遍历 |

一句话：**"宠物是什么、怎么算亲密度、怎么出文案、怎么算玩法数值"这一整套（约 15 个纯模块）可以整包搬到 Kotlin/Dart，一行逻辑都不用重想。**

### 问题 3：哪些必须在 Android 端重写（文件级清单）

| 文件 / 目录 | 为什么必须重写 |
|---|---|
| src/client/index.ts | cordis 客户端插件体：inject = ['slots','locale','connection','configForms','remote','sessions','uiWorkspace']，造 DOM 容器、createRoot 挂 document.body |
| src/client/PetSprite.tsx（49KB） | React DOM + createPortal + CSS Module + 指针/拖拽/悬停；Android 要 Flutter Widget / Compose 重画 |
| src/client/gameplay-hud.tsx | 同上（React + CSS Module + useSyncExternalStore） |
| src/client/pet.module.css、settings-card.module.css、settings-section.module.css | CSS Module |
| PetDockEntry.tsx、PetPreview.tsx、PetSettingsCard.tsx、PluginSettingsCard.tsx、SegmentGroup.tsx、RangeBubble.tsx、settings-form.ts、settings-registration.ts、locales.ts | DSH 设置面板 seat（settings.section）+ React；Android 是原生设置页 |
| renderers/live2d.ts + renderers/live2d/runtime.ts + Live2dVisualMount.tsx + vendor-entry.ts | **Cubism Web 运行时**（用户自备 live2dcubismcore.min.js）+ pixi.js + document.createElement('script') + WebGL。Android 只能用 Cubism Native SDK 重做，专有权问题原样存在 |
| renderers/frames2d.ts（25KB） | createImageBitmap + canvas 2D + img 兜底 + 1.2s 看门狗；Android 换 ImageDecoder/Bitmap + 自绘 |
| renderers/registry.ts | 契约本身可留，实现要按平台重写（HTMLElement container 类型） |
| src/routes.ts + http.ts + loopback.ts + access.ts + pair-access.ts | node:http + ctx.webServer.register + loopback/配对 cookie 守卫；Android 端没有这些 |
| src/service.ts（51KB） | cordis Service 子类，ctx.on('session/event'/'agent/assistant-stream'/'session/disposed')、settings 行、ctx.effect。状态视图可抄，宿主接线必须换 |
| src/index.ts | cordis 插件入口（name='pet', inject=['webServer'], apply） |
| dsh-home.ts、host/run-guarded.ts、mount-once.ts、telemetry.ts、client/telemetry.ts、client/drag-stream.ts、client/main-session.ts、client/web-extension-*.ts、client/ui-teardown.ts | DSH 宿主 / DSH Web 扩展 / 热重载接管专用 |
| （本地 0.3.8 独有）runtime/electron-helper/* 14 个文件 | Electron 主进程透明置顶窗。Android 对应物是 SYSTEM_ALERT_WINDOW 悬浮窗（dsh-Go 已经做了：app/android/.../pet/PetOverlay*.kt） |

**移植判定：FAIL（不可直接移植）+ WARN（逻辑可搬）**。客户端 15 个 TSX + 3 个 CSS Module + 2 个渲染器实现 + 1 套 HTTP 路由全部作废；纯逻辑 15 个模块可整包复用。若目标是大肥鱼 + Android，正确做法不是移植 zhu 版，而是**把纯逻辑模块搬到 dsh-Go 已有的原生宠物骨架上**（见 1.9）。

## 1.9 三个桌宠并列对比（lead 要求的判断依据）

| 维度 | ① dsh-pet（本地 0.3.8，Web 浮层） | ② surfcore pet（Electron 独立进程） | ③ DSH-Go 原生宠物 |
|---|---|---|---|
| 代码位置 | ~/.dsh/profiles/desktop/node_modules/dsh-pet/ | ~/.dsh/surfcore/pet/app/（feiyu-pet@0.2.3，来源 pet-pack-v0.2.3.zip） | ccch1mneyyy/dsh-Go app/lib/pets/* + app/android/.../pet/* |
| 渲染 | DOM <video> A/B 双缓冲播 106 段 webm（VP9-alpha，51.8MB） | Electron BrowserWindow + **PixiJS** 分层 PNG rig（5 套皮肤，19-21 层骨骼） | Flutter Widget + Kotlin 悬浮窗；APNG/序列图集；像素对齐 |
| 形象 | 蓝毛小女仆大肥鱼（视频） | 小肥鱼 ×5 套穿搭（feiyu-01..05，带 persona.md） | Deepy 小鲸鱼 / 鲸娘 + 可导入 Clawd/Codex 包 |
| 状态表达 | 会话事件聚合出的工作态 + 情绪池文案 | 人格卡 + 衰减记忆卡 + 目的机制 | Clawd 12 态 + 优先级表（error 8 / notification 7 / sweeping 6 / attention 5 / carrying,juggling 4 / working 3 / thinking 2 / idle,roam 1 / sleeping 0，app/lib/pets/pet_mood.dart:346-354） |
| 会不会自己调模型 | **会**：src/host/whisper.ts 走 ctx.llm，whisperEnabled:true → 一天约 288 次 | **会**：桌宠对话/主动搭话直接调 DSH llm 服务（docs/PET.md §3："不会走 DSH 的完整 agent 回合…而是插件直接调 llm"） | **不会**：docs/design/pets.md §0 "不主动调模型、不独立联网"，并把 dsh-pet 每 5 分钟调模型列为明确不做的反例 |
| 与 feiyu-mood 关系 | 无（1.7 已证） | 无（走自己的记忆卡 + purpose-outreach.json） | 无（pet_mood.dart 是 App 内信号归约，不是情绪持久化） |
| 跑在哪 | DSH Web GUI 里（document.body 浮层） | 独立 Electron 进程（--no-sandbox，pid 写 pet.pid） | 手机 App 内 + 可选 Android 悬浮窗 |
| 当前状态 | 在装、在用（main-config.jsonc：id=main，size=462，display=web） | **死的**：pet.pid=31524，tasklist /FI "PID eq 31524" 无结果；start.log 末行 "[chat] surfcore offline: fetch failed" ×2 | 设计已定稿（29KB），Kotlin/Dart 代码已入库（含 12 个测试文件） |
| Android 可行性 | 低：Web UI 依赖，无原生悬浮窗 | 零：Electron 跑不到 Android | **就是为 Android 写的** |
| 素材授权 | MIT 代码 / 素材禁止商用 | 自有（人设/美术自绘，skins/feiyu-*） | ASSETS.md 登记（Deepy 鲸鱼、鲸娘序列帧） |

**这张表的结论**：
- 想要"大肥鱼的脸" → 素材在 ② 和 ①，不在 ③。
- 想要"Android 上能跑" → 只有 ③。
- 想要"不烧 token、不打扰" → 只有 ③ 的设计哲学；zhu 版 0.4.14 也符合（本地文案池）；① 和 ② 都不符合。
- **正确路线不是三选一，而是"③ 的骨架 + ② 的素材 + zhu 版的纯逻辑"**。② 的 `skins/feiyu-02/manifest.json` 已经把人设、骨骼枢轴、口型、命中区分层写成 JSON（rigOptions / interaction.regions / persona.prompt），这套数据可以直接当 dsh-Go 宠物包的资产来源；只是那套 rig 播放器是 PixiJS，得重写成 Flutter/Kotlin。

---

# 二、语音主选：dsh-voice-mode

## 2.1 清单与兼容声明

`plugin/dsh-voice-mode/package.json`：
- `name: dsh-voice-mode`，`version: 0.7.34`，`license: MIT`
- `engines: { node: ">=18.0.0", dsh: ">=0.1.1-rc.2 <0.3.0-0" }`
- `peerDependencies: { "@deepseek-ai/cordis": "^4.0.1", react: "^18.2.0" }`
- `dependencies: { "@deepseek-ai/schemastery": "^3.18.5", "sherpa-onnx": "^1.13.6", "sherpa-onnx-node": "1.13.6" }`
- `dsh.client.inject`：9 个锚点（dsh-client-connection / dsh-cordis-client-runner / dsh-api-remotes / dsh-client-locale / dsh-client-ui-renderer / dsh-client-ui-conversation / dsh-client-ui-layout / dsh-client-ui-settings / dsh-client-ui-settings-plugins），`platform: "web"`

**"有没有声明兼容 0.2.1-rc.2"——准确回答：没有 `dsh.compatibility.dshReleases` 这种逐版本清单，它用 `engines.dsh` 区间声明。** 0.2.1-rc.2 落在该区间内。实证记录在 `docs/compat-contract.md`：
- §19（2026-10-19）：0.2.1-rc.2 四层验证——L1 锚点 9/9 通过、L2 host+client 双 typecheck 通过、L3 runtime 冒烟"host 三端点 200、mic 中英渲染、console 0 error、设置卡片入口/迁移/写入/刷新保留/双语/语言热切换/覆盖层落盘/引擎热切换/非法值 400/未知键 400/跨源 403 全断言通过"、**L4 真 LLM 端到端（语音闭环）未跑**、**桌面端 Electron 冒烟未跑**（原文"未跑层（如实声明缺口）"）。
- §2.2 规则（v0.7.29 落地）：`engines.dsh` 上界收紧为 `<0.3.0-0`，因为 semver 的 `<0.3.0` 不排除 `0.3.0-alpha.1`。
- CHANGELOG 侧最新维护：0.7.34（2026-10-28）+ Unreleased "补验 dsh 0.2.1-rc.5"。

**判定：源码/声明层面 PASS（区间覆盖 + 文档记录了 rc.2 的三层实测通过）；运行层面 NOT_TESTED（本轮没跑，L4 官方自认未验收）。**

## 2.2 中文语音识别（ASR）的真实实现

三层，全在 **host 侧**跑（不是浏览器）：

1. **流式识别：zipformer2-zh int8（sherpa-onnx Node）**
   `src/asr-host.ts:52` `MODEL_REPO = 'csukuangfj/sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30'`；文件与 SHA256 写死（`:53-58`）：encoder.int8.onnx（约 153.7MB）/ decoder.onnx / joiner.int8.onnx / tokens.txt。引擎 `import sherpa_onnx from 'sherpa-onnx'`（`:20`），用 `createOnlineRecognizer`。
2. **定稿重译：SenseVoice（离线整段，带标点 + ITN）**
   `:67` `SENSE_REPO = 'csukuangfj/sherpa-onnx-sense-voice-zh-en-ja-ko-yue-2024-07-17'`，model.int8.onnx 约 228MB（`:66` 注释："模型总体积 160+228=388MB ≤500MB 约束"）。跑在 **worker_threads**（`src/sense-worker.ts`，`createOfflineRecognizer`），20s 超时后降级回 zipformer 定稿（`:639-663`）。
3. **VAD：Silero**
   `:60-64` 官方 csukuangfj/vad 的 silero_vad.onnx（约 2MB）；`vad.acceptWaveform(inc)` → `vad.isDetected()`（`:591-594`）。

模型下载：`src/models.ts:22` 严格白名单 `['huggingface.co','hf.co','hf-mirror.com']`，仅 https，重定向逃逸也拦（`redirectHostAllowed`），每文件 SHA256 校验 + 断点续传（.part）+ 头阶段 60s 超时 + body 空闲 120s 看门狗 + 校验结果缓存。

**中文质量判定：WARN**。方案本身是当前开源里最强的一档（zipformer2-zh 流式 + SenseVoice 定稿重译），但**质量数字我没有、仓库也没有给出字错率基线**——`docs/qa/must-verify-manually.md` 明确写"在固定语料上记录实际字错率与冷启动等待，**不凭主观印象宣称 ≥95%**"。所以我只能写"设计正确"，不能写"识别准确率多少"。NOT_TESTED。

## 2.3 VAD 与打断

- **VAD 在服务端**（ADR-0003 已改名 `0003-server-side-vad.md` 纠正错误暗示）：链路 = 采集帧 → `POST /voice-mode/asr?vadOnly=1`（100ms 节拍）→ host Silero → 下行 isSpeech → 客户端连续 `INT_CONFIRM_FRAMES = {0:3, 1:2, 2:1}`（`src/client.tsx:61`）次为真 → 回声门控 → `hardBreak`。
- **真机标定延迟约 525ms**（`docs/adr/0003-server-side-vad.md` 拆解表：轮询量化约 64ms 均/128ms 最坏 + 上行推理下行数十 ms + 确认窗 384/256/128ms + Silero 固有约 32ms）。
- **barge-in 三档**（ADR-0006，`bargeInMode` 默认 `detect`）：detect 自动探测降级 / auto 强制自动 / manual 长按。两级探测：第一级 `getUserMedia` 后读 `track.getSettings().echoCancellation`（false → 直接落 manual）；第二级首播前约 2s 窗口标定（`echoGateDb` 默认 6dB）。
- **用户开口自动停止播报**：`src/client.tsx:2623` `hardBreak()` → 升级 epoch、清字幕队列、`POST /cancel`。另有一层"让位语义"（ADR-0008，`backchannelYield` 默认 true）：只说"嗯"时跳当前句 + 1.5s 静音，不算一次打断。
- **打断抢救**（ADR-0012，`bargeInSalvage` 默认 true）：把开口前约 250ms 预录环并入转写，修"打断吃句"。
- **AEC**：原生优先（ADR-0001，AEC3），自研 NLMS 仅在 `echoCancellation===false` 时主路径；实测"朗读期 Silero 判回声为语音 0/937 帧"（`docs/findings/2026-09-14-fixture-verdict.md`）。

## 2.4 流式 TTS / 本地引擎 / 依赖

- **分句 → 队列 → SSE 下发**：`src/segmenter.ts`（markdown 剥离 + 中英日终止标点 + 强制上限 + 单字符消毒，注意 `<break>/<whisper>` 等 emotion 标签被白名单保留）；`src/tts-queue.ts`（逐会话串行合成，协议="单 chunk + final 帧"，`cancel()` 提 epoch 弃积压，真正静音）。
- **三个引擎**：
  - `edge`（默认，云端）：msedge-tts 被 esbuild **内联**成 `lib/msedge-tts.cjs`（ADR-0011，因为 pnpm11 拒安装脚本 + `buffer/index` 被 ResolutionRouter 拒）。**第三方非官方接口**，2026-10-26 微软改握手导致整体无声（CHANGELOG 0.7.31），修法是升内联版到 2.1.1 换端点。
  - `vits`（本地中文）：`csukuangfj/sherpa-onnx-vits-zh-ll`，5 说话人，约 130MB，WASM，跑 `fork()` 子进程（`lib/tts-vits-worker.cjs`，`src/tts-local.ts:363`）。
  - `kokoro`（本地中英）：`kokoro-int8-multi-lang-v1_1`（int8 约 109MB 主模型）/ `kokoro-multi-lang-v1_1` fp32 双档 + espeak-ng-data 整树约 355 文件；走 `sherpa-onnx-node` 原生 addon。
- **Electron 坑（对 Android 极重要）**：`src/tts-runtime.ts` 头注释——Kokoro 的 N-API 外部缓冲区在 Electron V8 被拒（`External buffers are not allowed`），所以 Electron 宿主下必须另找一个**真 Node ≥18**（`DSHVM_NODE` 或 PATH）来跑子进程。
- **依赖重量**：`sherpa-onnx` + `sherpa-onnx-node` 两个运行时依赖（WASM/原生），模型合计 160(ASR)+228(Sense)+2(VAD)+130(VITS)+约310(Kokoro fp32) → **全下满约 800MB+**，常用档也在 390-500MB。`docs/dependency-policy.md` 自己写明"单测用桩件替代 sherpa-onnx，CI 绿灯不能证明真实识别行为未变"。

## 2.5 字幕 / 长会话 / 手机麦克风 / 前后台

- **字幕**：`showCaptions` 默认 true；`captionFontSize` 4 档（0/1/2/3 → 12/14/18/24px，`client.tsx:4333`）；`captionMaxWidth` 3 档（50/70/90vw，`:4350`）；`.dshvm-caption { word-break: break-word; overflow-wrap: anywhere }`（`:1850`）；`maxHeight:30vh + overflow:hidden`。无障碍：状态条 `role=status aria-atomic`，波形/计时/partial/角标全部 `aria-hidden`（`:4065` 注释），只播报状态文本变化（v0.7.29 实测"识别 27 次更新期间播报 1 次"）。
- **长会话**：`idleTimeoutMinutes` 默认 5、范围 1-120；空闲续期口径——朗读/回合运行中只续期不刷新基准，**2h 绝对上限**；9:30 出预警；隐藏标签页仍按活动续期。
- **手机麦克风权限 / 前台后台**：README 列了三条硬约束——(a) 必须安全上下文（https 或 localhost/127.0.0.1），LAN IP 的 http:// 不暴露麦克风；(b) iOS 后台/锁屏暂停识别与朗读，回前台自动恢复（可能丢句）；(c) `visibilitychange` 隐藏 tab 主动关麦（`client.tsx:3359-3380`，注释"M2：隐藏 tab 暂停收音（隐私）"），恢复失败则完整退出语音模式。
- **权限与数据流**（README 权限章）：network = Edge 云端会收到待朗读文本；shell = 本地 TTS `child_process.fork`；识别全程在宿主，不经独立云 ASR。

## 2.6 voice-mode 功能点结论表

| 功能点 | 判定 | 依据 |
|---|---|---|
| 版本/许可证 | PASS（静态） | 0.7.34 / MIT |
| 声明兼容 0.2.1-rc.2 | PASS（静态，区间覆盖 + §19 记录三层实测） | engines.dsh + compat-contract §19 |
| 真语音端到端 | NOT_TESTED | §19 自认 L4 未跑；rc.3/rc.4 上因凭据失败（full-e2e.sh 0 帧，rc.2 对照组同样） |
| 桌面 Electron 冒烟 | NOT_TESTED | must-verify-manually.md 登记"用户侧复跑" |
| 中文 ASR 字错率 | NOT_TESTED | 仓库明确拒绝给 ≥95% 这种数字 |
| 流式识别 | PASS（静态） | zipformer2 online recognizer |
| 定稿重译 + ITN | PASS（静态） | SenseVoice worker，20s 超时降级 |
| 服务端 VAD | PASS（静态） | Silero，ADR-0003 |
| 打断（barge-in） | PASS（静态）；延迟 525ms 为历史实测值，本轮未复测 | ADR-0006 / 0003 拆解表 |
| 打断抢救 | PASS（静态） | ADR-0012，bargeInSalvage |
| 流式 TTS | PASS（静态）→ 实为"逐句串行 + 单 chunk"，非 token 级流 | tts-queue.ts P1-1 协议 |
| 本地 TTS | PASS（静态）；Kokoro fp32 官方自认未完成完整真实推理 | CHANGELOG 0.7.34「验证边界」 |
| 字幕 | PASS（静态） | 字号/宽度/aria |
| 长会话 | PASS（静态） | idleTimeout + 2h 上限 |
| 手机麦克风/前后台 | WARN | 有代码与文档，但只有 Web/Electron 语义，无 Android 原生实现 |
| 需联网 | **是（默认）** | Edge TTS 云端；模型首次下载需 HF/hf-mirror |

---

# 三、语音备选：dsh-talk

## 3.1 清单与兼容声明

`package.json`：
- `name: dsh-talk`，`version: 0.3.26`，`license: Apache-2.0`
- `engines: { node: "^22.19.0 || >=24.0.0", dsh: ">=0.2.1-0 <0.3.0" }`
- `peerDependencies`：11 个包，全是多段区间（`>=0.1.2-rc.1 <0.2.0 || >=0.1.5-alpha.1 <0.2.0 || >=0.1.6-0 <0.2.0 || >=0.1.7-0 <0.2.0 || >=0.2.0-0 <0.3.0 || >=0.2.1-0 <0.3.0`），且 7 个标 optional
- `dependencies`：**只有 `zod 4.4.3`**（0.3.26 的修复就是把它钉住，防宿主 zod 双实例破坏 ZodType 同一性）
- `dshWorkshop.compatibility.dshVersions`：`["0.1.2-rc.1","0.1.5-rc.2","0.1.7-alpha.2","0.1.7-rc.1","0.1.7-rc.2","0.2.1-alpha.1"]` —— **清单里没有 0.2.1-rc.2**，最新只到 0.2.1-alpha.1
- `dshWorkshop.evidence`：`{install:null, failureIsolation:null, hotReload:null, remove:null}` —— **四项证据全空**
- `dsh.client.inject`：只有 3 个锚点（connection / locale / ui-settings）

**兼容判定：WARN**。engines.dsh 区间覆盖 0.2.1-rc.2，README 兼容表写"DeepSeek Harness dsh-v0.2.1-alpha.1（2026-09-24 适配…月度 Compat workflow 已锚定该线）"，但**逐版本清单停在 alpha.1**，evidence 四项 null。声明上"能装"，证据上"没验过 rc.2"。

## 3.2 源码结构与能力

- `src/engine.ts`：本地引擎**全部靠外部可执行文件**。
  - edge-tts：`edge-tts --voice <v> --rate=<±N%> --text <t> --write-media <out.mp3>`（`:53-64`）
  - piper：`piper --model <onnx> --output_file <wav> --length_scale <s>`，文本走 stdin（`:66-82`）
  - whisper.cpp：`whisper-cli --model <ggml> --file <wav> --output-txt --output-file <base> [--language]`，读 `<base>.txt`（`:84-105`）
  - FunASR：multipart POST `{url}/v1/audio/transcriptions` → `{text}`（超时 120s）
  - 全部经 `ctx.subprocess` 执行，产物 `MAX_SYNTH_BYTES = 16MiB` 上限
- `src/config.ts`：命令默认值全是外部二进制名 `'edge-tts' / 'piper' / 'whisper-cli'`（`:439, 453-457`），`stt.engine=funasr` 时**必须**配 `url`，否则 resolve 抛错（`:478-479`）。**即：想在本地用中文 ASR，用户必须自己跑一个 FunASR 服务或装 whisper.cpp + 模型。**
- `src/speech.ts`：把每句话 append 成 `dsh-talk/speech` 会话事件，用 `KNOWN_SESSION_EVENT_TYPES.has(...)` 做门禁（宿主不认识这个类型就跳过并把降级报给调用方）—— 设计很克制。
- `src/tool.ts`：`speak` 工具，模型只看到"says aloud with engine"，音频在浏览器播，本地引擎还会落盘并把绝对路径写进 `audioPath`（可以喂给别的工具）。
- `src/service.ts`：`TypertRemoteService`，`interrupt()` 是 `AbortController.abort()` 掉在途合成（`:518-526`）。**没有 VAD、没有 barge-in 判定。**
- `src/client/vad.ts`：**纯 RMS 能量**（`rmsOf` / `isSpeechFrame(rms>=threshold)` / `advanceSilence` / `silenceTrips`），无模型、无 Silero。
- `src/client/TalkMicButton.tsx`：`MediaRecorder` 录音 → `talk/transcribe`；只有 `stt.resolved === 'web'` 时走 `webkitSpeechRecognition`（0.3.22 修的 issue #25）。"说话即打断" = **按下麦克风时** `void interrupt()`（`:241-242`），不是检测到人声。
- 播放：`talk:speech` 投影 → browser 走 `speechSynthesis`，本地引擎走 `talk/audio` 取 base64 → `new Audio(url).play()`（`:437-462`）。**整段播放，无字幕、无 overlay、无进度。**
- `src/provider.ts` 只是"未来注册表"的接口占位（注释自认 reserved）。
- `src/client/present.ts` / `browserVoice.ts` / `src/audio-export.ts` / `src/sanitize.ts` 都是纯函数 + 有界文件目录。

**字幕：0 命中。** `TalkMicButton.tsx` 全文件 grep `caption|字幕` = 0；client 只有麦克风按钮 + 设置页签（`conversation.input.left`、`settings.plugins.tab`）。

## 3.3 dsh-talk 功能点结论表

| 功能点 | 判定 | 依据 |
|---|---|---|
| 版本/许可证 | PASS（静态） | 0.3.26 / Apache-2.0 |
| 声明兼容 0.2.1-rc.2 | WARN | engines 覆盖，但 dshVersions 只到 0.2.1-alpha.1，evidence 四项 null |
| 中文 ASR | WARN | 非自带：Web Speech（Chrome 走云端厂商服务，国产 ROM/无 GMS 不可用）或自建 FunASR / whisper.cpp |
| VAD | WARN | 纯 RMS 能量门限（energyThreshold 0.01），无模型 |
| 流式 TTS | FAIL | 整段合成 → 整段播放（engine.ts 读完整文件；client new Audio()） |
| 打断 | WARN | 只有"按下麦克风即停播"（interrupt()），无 VAD 驱动的自动打断 |
| 字幕 | FAIL | 源码零实现 |
| 长会话 | WARN | 无空闲超时；单次录音上限 60s（可配到 600s），非持续会话 |
| 依赖重量 | PASS（很轻） | 运行时只 zod；但功能靠外部二进制/服务 |
| 纯前端 | 否 | STT/TTS 都在宿主或外部进程 |
| Android 可移植性 | WARN | 逻辑薄（易重写），但能力本身依赖宿主二进制 + 浏览器 API |
| 维护活跃度 | PASS（静态） | 2026-11-01 有推送；0.3.26 于 2026-10-24；5 种语言 README；dsh-plugin-doctor 徽章 |

---

# 四、对比表

| 维度 | dsh-voice-mode 0.7.34 | dsh-talk 0.3.26 |
|---|---|---|
| **中文 ASR 质量方案** | **自带两段式**：zipformer2-zh int8 流式（154MB）+ SenseVoice int8 定稿重译（228MB，带标点+ITN）；模型 SHA256 钉死、可镜像 | **不自带**：Web Speech（Chrome 系走厂商云，国产 ROM 常不可用）或外部 FunASR HTTP 服务 / 本地 whisper.cpp 二进制 |
| **VAD** | 服务端 **Silero**（silero_vad.onnx 约 2MB）+ 客户端确认帧（3/2/1），确认窗 128/256/384ms，整体约 525ms | 客户端 **纯 RMS 能量**（energyThreshold 默认 0.01）+ 静音计时，仅用于"自动结束录音" |
| **流式 TTS** | 逐句串行队列 + SSE 单 chunk/final 帧；边生成边播（句级流），三引擎（Edge 云 / VITS 本地 / Kokoro 本地） | **无**：整段合成到临时文件 → 读全量字节 → 浏览器整段播 |
| **打断（barge-in）** | **有，三档** auto/manual/detect（默认 detect 自动探测降级）+ 回声门控 dB + 确认帧档位 + 打断抢救（预录 250ms 并入）+ 让位语义（"嗯"不算打断） | **只到"开口即停"的一半**：按下麦克风按钮时 abort 在途合成；无 VAD 驱动、无回声门控、无抢救 |
| **字幕** | 有：句级字幕 + 4 档字号 + 3 档宽度 + aria-live 只播状态 + 中文长 URL 换行 | **无**（源码 0 命中） |
| **长会话** | 有：idleTimeout 1-120 分钟（默认 5）、2h 绝对上限、朗读/回合中续期、隐藏 tab 关麦、回前台恢复 | 无空闲超时；单次录音 60s（可配 ≤600s）；无会话态 |
| **依赖重量** | 重：sherpa-onnx + sherpa-onnx-node（WASM/原生），模型 160+228+2(+130/+310) MB，全下满约 800MB；内联 msedge-tts 约 0.7MB | 极轻：运行时仅 zod；但要另装 edge-tts/piper/whisper-cli 二进制或自建 FunASR |
| **是否纯前端** | 否（ASR/TTS 全在 host） | 否（STT/TTS 在 host 或外部进程） |
| **Android 可移植性** | 低-中：核心是 sherpa-onnx-node（已有 arm64 预编译，见 dsh-Go docs/design/voice-input.md §2.1 实测 sherpa-onnx.node 在 arm64 构建报告里）；但客户端 4409 行 DOM/React/AudioWorklet/SSE 全废，且 Electron 下 Kokoro 要外部真 Node（手机上不可能） | 低：逻辑薄但能力不在自己手里——手机上跑不了 edge-tts/piper/whisper-cli 二进制 |
| **DSH 版本兼容** | engines.dsh >=0.1.1-rc.2 <0.3.0-0；文档记录 rc.2/rc.3/rc.4/0.2.2-alpha.2 多层验证；无 dshReleases 清单 | engines.dsh >=0.2.1-0 <0.3.0；dshVersions 只到 0.2.1-alpha.1；evidence 四项 null |
| **许可证** | MIT | Apache-2.0 |
| **维护活跃度** | 高：0.7.34（2026-10-28），CHANGELOG 逐版记；20 版锚点矩阵 + 17 内核冒烟 + CI 每日上游巡检 + HOL 扫描 100 分；文档密度极高（40+ ADR/findings/QA） | 中：0.3.26（2026-10-24），2026-11-01 有推送；CI/Compat/Scorecard 工作流齐；文档偏 README，无 ADR |
| **安全姿态** | 强：回环 + 同源 + 每会话 60 req/s 限流 + 模型 SHA256 + 域名白名单 + 重定向逃逸拦截 | 中：subprocess:spawn + network:outbound 权限声明；sanitize.ts 脱敏；无模型校验（不下载模型） |

---

# 五、选型结论

## 5.1 桌面端（当前 DSH Desktop 0.2.1-rc.2）

- **主方案：dsh-voice-mode**。理由只有三条，但每条都硬：
  1. **它是唯一自带中文 ASR 的**。dsh-talk 的 ASR 是"把 Web Speech 或外部服务接进来"，在国产 ROM/无 GMS 场景直接失效（dsh-Go 自己的 docs/design/voice-input.md §1 用大篇幅证明这条链路不可靠）。
  2. **它是唯一有真 barge-in 的**。服务端 Silero + 确认帧 + 回声门控 + 三档模式 + 打断抢救，是 8 个 ADR 拧出来的工程结果；dsh-talk 只有"按麦克风时停播"。
  3. **它有字幕**。dsh-talk 没有。
- **备选：dsh-talk**，仅在这些情况用：不想要 800MB 模型 / 只想让 agent 有个 speak 工具朗读 / 已有自建 FunASR 服务 / 需要"朗读落盘成文件喂给别的工具"这个独有能力（audioPath 是 dsh-talk 独有的，voice-mode 没有）。
- **不许两个都装。** 根本原因不是端口冲突，是同一条输入链被两套抢：
  - 两者都往 conversation.input.* 注入麦克风/renderer（voice-mode 用 right+dock，dsh-talk 用 left）；
  - 两者都挂 llm/stream 或 speak 工具去截/发朗读文本（voice-mode 是 llm/stream tap + 自动分句；dsh-talk 是模型主动调 speak 工具）→ 装两个 = 同一句回复被读两遍，或 voice-mode 自动读 + 模型又调 speak；
  - 两者都要抢麦：voice-mode 持续采音（AudioWorklet 常开），dsh-talk 按需 MediaRecorder → 蓝牙/内置麦的独占会导致其中一个拿不到流；
  - 两者都写自己的设置覆盖层（voice-mode 走 ctx.settings.register + 自己的覆盖文档，dsh-talk 走 cordis.patch.yml append-only patch）→ 设置面板出现两个"语音"页签。
  - 结论：**二选一，且选 voice-mode。**

## 5.2 落到 DSH-Go（Android）—— 这段是最关键的判断

**先给结论：voice-mode 和 dsh-talk 都不能原样装进 DSH-Go，必须重写成 dsh-Go 原生插件。** 理由有规格 + 源码双重证据：

1. **DSH-Go 的语音能力面只有四样，且明确不支持流式**。`docs/plugins/voice.md`（协议 v1.2）：
   - `audio`：一段声音，`media` 只回**完整的一段**（audio/mpeg|wav|ogg|mp4|aac），原文"**不支持流式**"，上限 **16MiB**，超时 30s；
   - `voice`：只给 speechStart / speechEnd 事件，"App 不会把声音或文字给你"（拿不到 PCM、拿不到识别文本）；
   - `micro`：一条 ≤240 字、≤10Hz 的数据通道（字幕）；
   - `overlay`：一行悬浮字幕。
   → voice-mode 的整套链路（浏览器采 PCM → POST /voice-mode/asr 每 128ms 上行 → host 流式识别 → SSE 推逐句音频帧 → 浏览器拼接播放）**在这套能力上一条都落不下去**：没有上行 PCM 通道、没有下行流、没有 SSE。
2. **App 只在前台听**："退到后台、锁屏、换对话、换主机都立刻停，不会自动恢复"；"全 App 同时只有一个声音会话"；麦克风权限由 App 代为询问（第一次问一次，记住答案 + 插件设置里有"听你开口"开关）。
3. **App 侧已经有自己的回声处理**："用系统的通话录音通道和回声消除（设备有的话），朗读时把开口门槛调高、开头 0.4 秒不算"，并明确承认"外放很响、设备没回声消除时，朗读可能被自己的声音打断" → voice-mode 那套"服务端 Silero + 确认帧 + echoGateDb"在 Android 上是重复建设，且拿不到 PCM 无法实施。
4. **纯 Web 手段在手机上不可行**：voice-mode 的客户端是 DSH Web 插件（dsh.client.platform: "web"），DSH-Go 只能在"岛屿"里跑 DSH Web 页面（docs/plugins/web-panels.md：整页岛"只在这台手机自己的引擎上、主机有桥接占用表、路由守卫生效、资源预算够时打开"，12s 加载不出来就降级成"在网页版打开"；聊天页部件岛"关掉 30 秒后回收"）。岛屿里的麦克风权限、AudioWorklet、SSE 长连接、后台保活全部没有规格保证 → NOT_TESTED（本轮无法验证，仓库文档也没有承诺）。所以"靠 Web 岛跑 voice-mode 实现流式语音对话"不能作为方案。

**唯一正确的落点**（与 voice-mode 的能力可对齐，但代码要重写）：
- 用 **media 回整段音频**：插件在**主机侧**调 TTS（可以是 voice-mode 的 sherpa-onnx 引擎，或 surfcore 的本地 8765 TTS，或 Edge）→ 返回 `{body: Buffer, contentType:'audio/mpeg'}`。这天然规避"不支持流式"：**句级流式改成"整段回复一次合成"**，代价是首字延迟变成长度相关。若必须逐句流，只能 App 侧排队 `audio.play` 多次（协议未定义"play 后追加"，属未支持用法）。
- 用 **overlay + micro 做字幕**：插件合成时按字数/时长切段，每 100ms 推一次 `{text, progress}` → App 画悬浮字幕。这正是 `examples/plugins/dsh-go-example-readaloud/lib/index.js` 的写法（`ctx.dshGo.media('speech', ...)` + `ctx.dshGo.micro('caption', {data})` + `ctx.dshGo.onCapability` 接 `voice.speechStart`）。
- 用 **voice 做打断**：不要自己判 VAD，**App 已经把"用户开口先自己停掉声音"做掉了**，插件只需在 onCapability 收到 voice.speechStart 后停掉字幕推送、记一笔（示例代码里就是 `readings.end(scope)`）。
- 用 **requires: ["audio"]** 让不支持声音的宿主（中转主机）自动不显示按钮。
- 识别方向反过来想：**DSH-Go 不需要插件做 ASR**——它自带官方语音套件（docs/design/voice-input.md：sherpa-onnx-node 跑 SenseVoiceSmall INT8 241MB，speech/catalog、speech/prepare、speech/transcribe），且 arm64 构建报告里 sherpa-onnx.node 存在（官方实测）。→ **voice-mode 的 ASR 那 388MB 模型在 Android 上是重复投资，只有它的 TTS（本地 VITS/Kokoro）与"句子切分/sanitize/emotion 标签"这几个纯模块有迁移价值。**

**可迁移清单（voice-mode → dsh-Go）**：
- 直接搬（纯函数，零 DOM）：`src/segmenter.ts`（分句 + markdown 剥离 + 单字符消毒）、`src/emotion.ts`（`<break 300ms>` → 插静音、`<whisper>` → PCM ×0.5、`<laugh>/<sigh>/<emphasis>` 剥离）、`src/draft-compose.ts`、`src/mic-gesture.ts`（手势/节流判定，去掉 DOM 部分）、`src/voice-catalog.ts`（音色表数据）、`src/models.ts`（SHA256 + 白名单 + 断点续传下载器）。
- 需改造：`src/tts-local.ts`（fork 子进程 → Android 没有 fork；要么在 proot 里的 Node 侧保留，要么改成 App 内嵌 sherpa-onnx 原生库）、`src/tts-queue.ts`（SSE 广播改 micro/media）。
- 必须放弃：`src/client.tsx`（4409 行）、`src/asr.ts`（1744 行，AudioWorklet + 128ms 轮询）、`src/asr-host.ts`（服务端 Silero 打断判定）、`src/aec.ts`、`src/index.ts`（/voice-mode/* 路由 + SSE + llm/stream tap）、`src/settings-form.tsx`。

---

# 六、证据清单（仓库 + SHA + 路径 + 关键片段）

**zhu1090093659/dsh-pet @ 9ed843bccee3d2604ce0ac31f55cbde9bda97c7f**
1. `package.json` — version 0.4.14、name $linxin666/dsh-pet、license Apache-2.0、dsh.engines.dsh ">=0.2.1-rc.3"
2. `src/contracts/renderer.ts:15` — PET_RENDERER_API_VERSION = 'x-org.linxin666.pet-center/v1alpha1'
3. `src/manifest-v2.ts:34` — PET_RENDERER_KINDS = ['sprite2d','live2d','frames2d']
4. `src/client/renderers/live2d/runtime.ts:18-19` — CORE_URL='/api/pet/runtime/live2dcubismcore.min.js'、VENDOR_URL='/api/pet/runtime/live2d-vendor.js'
5. `src/client/renderers/live2d.ts` — vendor.extensions.add(vendor.Live2DPlugin) + vendor.configureCubismSDK({memorySizeMB:32})
6. `tsdown.live2d-vendor.ts` — format:'iife'、platform:'browser'、noExternal:[/.*/]，@pixi/sound 打桩抛错
7. `src/client/spritesheet.ts` — scaledCellSize() / framePosition()（-col*scaled.width, -row*scaled.height）
8. `src/state.ts` — rowOf() 9 行表；celebrateMs:2400、failureMs:2400
9. `src/routes.ts` — PET_API_PREFIX='/api/pet'；makePetRoutes() 14 条 API + 3 条 prefix；PET_ASSET_CAPS={manifest:64KB,image:20MB,model:32MB}；RUNTIME_FILES 白名单 3 项
10. `src/service.ts:191-275` — PetStateView 全字段；:303-307 interface Context { pet: PetService }；:641/672/681 三个 ctx.on
11. `src/announce.ts` — kind: 'balance'|'cost'|'plan'；MAX_ANNOUNCEMENTS=4；ANNOUNCE_MAX_TTL_MS=7_200_000
12. `src/affinity.ts` — defaultAffinityConfig={turnReward:1,petReward:1,petCooldownMs:10_000,feedReward:5,feedCooldownMs:30_000}；AFFINITY_MAX=999_999_999
13. `src/treats.ts` — {turnsPerTreat:30,timeTreatMs:300*60_000,maxTreats:20}
14. `src/chatter.ts` — STATUS_ROTATE_MS=4000；WHISPER_COOLDOWN_MS=9000；WHISPER_TTL_MS=8000；"The model's own prose is never read"
15. `src/gameplay.ts` — PET_ROAM_DIRECTIONS、parseGameplayManifest、settleGameplay、applyGameplayEffects、rollTouchBranch、touchZoneAt
16. `assets/doro/pet.json:46-68, 193-197, 232-243` — stats.mood{max:100,decayPerMinute:0.5}、clickBoost{stat:'mood'}、modes.wash.restore{stat:'mood',amount:3}
17. `src/client/index.ts:629-633` — document.createElement('div') + container.dataset.dshPetRoot + createRoot(container)
18. `README.zh.md:149` — "本插件永不内置、永不代下 Cubism Core 运行时"
19. `docs/release-notes/v0.3.25.md` — "宠物插件迁出 dsh-web 主仓后的首次独立发布"
20. trees 统计：3831 blob；.webp 3536 / .webm 0；electron|standalone|runtime/ 命中 0

**本地 dsh-pet 0.3.8（~/.dsh/profiles/desktop/node_modules/dsh-pet/）**
21. `package.json` — version 0.3.8、license MIT、dsh.compatibility.dshReleases{"0.2.1-rc.2":"compatible"}
22. `src/host/whisper.ts` 头注释 — "碎碎念生成（host 半侧）：用 DSH 的 LLM 统一抽象层（ctx.llm）"
23. `src/host/chat.ts` — "对话生成（host 半侧）：用 DSH 的 LLM 统一抽象层（ctx.llm）…回复放宽到 256 token，超时放宽到 60s"
24. `src/client/pet.ts` — 双缓冲 video：videoARef/videoBRef、.dsh-pet-video{...object-fit:contain;opacity:0;transition:opacity .18s}
25. `runtime/electron-helper/main.js` — "为每只桌面宠物开一个独立的局部小窗口（透明、置顶、不可激活）"
26. `assets/webm/` — 106 个 .webm，合计约 51.8 MB
27. `README.md:305` — "素材（动画/提示词/源视频）：允许开源使用，禁止商用"
28. grep：src/ + runtime/ 全量 mood|feiyu = 0

**PC2005-cloud/dsh-pet（main，2026-11-02T14:36Z 推送）**
29. `dsh-pet/package.json` — version 0.4.1、license MIT、dshReleases{0.2.1-rc.2:compatible, 0.2.1-rc.3:compatible, 0.2.2-alpha.2:compatible}

**~/.dsh/feiyu-mood 0.8.1（读盘）**
30. `lib/index.js:776-806` — {version:STATE_VERSION, moods:{<key>:normState(v)}, lastActive}；:302 DIMS 七维
31. `lib/index.js:1619-1700` — GET /dsh-feiyu-mood/state 返回 {mood, moodLabel, emoji, dims[], lastReason, lastConfidence, injectQuota, injectable, ...}

**surfcore pet（~/.dsh/surfcore/pet/app）**
32. `package.json` — name:"feiyu-pet", version:"0.2.3", dependencies:{pixi.js:"^8.22.0"}
33. `.surfcore-pet.json` — source 指向 pet-pack-v0.2.3.zip，installedAt 2026-10-30T01:43:32.610Z
34. `pet.pid` = 31524；tasklist /FI "PID eq 31524" → 无匹配；start.log 末两行 "[chat] surfcore offline: fetch failed"
35. `skins/feiyu-02/manifest.json` — rig:"layered"、rigOptions{headPivot,mouthPivot,eyeCenters,motion,tailPivot,mouth{rest:"open",squashMin:0.28}}、interaction.regions[{id:"head",gesture:"pat"},{id:"tail",gesture:"tailpet"}]
36. `~/.dsh/plugins/surfcore/docs/PET.md` §3 — "对话不会走 DSH 的完整 agent 回合…而是插件直接调 DSH 的 llm 服务"；§7 列出 /dsh-surfcore/pet-api/* 全部路由与鉴权（x-pet-token，仅 127.0.0.1）

**qishuilalala/dsh-voice-mode @ 3524db40938f16e7c8f0726e30647afe5bfbef25**
37. `plugin/dsh-voice-mode/package.json` — 0.7.34、MIT、engines.dsh ">=0.1.1-rc.2 <0.3.0-0"、deps sherpa-onnx ^1.13.6 + sherpa-onnx-node 1.13.6
38. `src/asr-host.ts:52-58` — MODEL_REPO='csukuangfj/sherpa-onnx-streaming-zipformer-zh-int8-2025-06-30' + 4 个文件 SHA256；:60-64 Silero silero_vad.onnx；:67-70 SENSE_REPO + model.int8.onnx；:591-594 vad.acceptWaveform → vad.isDetected()
39. `src/sense-worker.ts` — worker_threads，createOfflineRecognizer
40. `src/models.ts:22,225` — ALLOWED_MODEL_HOSTNAMES=['huggingface.co','hf.co','hf-mirror.com']；下载 URL 形如 <host>/<repo>/resolve/<revision>/<file>
41. `src/client.tsx:61` — INT_CONFIRM_FRAMES={0:3,1:2,2:1}；:2623 hardBreak；:3359-3380 visibilitychange 关麦/恢复；:4333/4350 字幕字号/宽度
42. `src/segmenter.ts` — plainText() + sanitizeForTts()（emotion 标签白名单保留）
43. `src/tts-queue.ts` — 单 chunk + final 帧协议；cancel() 提 epoch
44. `src/tts-local.ts:24-26` — TTS_MODEL_REPO='csukuangfj/sherpa-onnx-vits-zh-ll'、kokoro-int8-multi-lang-v1_1、kokoro-multi-lang-v1_1；:363 fork(workerPath)
45. `src/tts-runtime.ts` — Electron 下 Kokoro "External buffers are not allowed"，需外部真 Node ≥18
46. `src/index.ts:1222` — const BASE_PATH='/voice-mode'；路由 19 条（含 /stream SSE、/asr、/toggle、/models/*）
47. `docs/compat-contract.md` §19 — 0.2.1-rc.2 四层验证（L1/L2/L3 通过，L4 真语音与桌面 Electron 未跑）
48. `docs/adr/0003-server-side-vad.md` — 打断延迟约 525ms 拆解表
49. `docs/adr/0006-barge-in-auto-degrade.md` — bargeInMode: auto/manual/detect，detect 为默认
50. `docs/adr/0011-bundle-msedge-tts-inline.md` — msedge-tts 内联为 lib/msedge-tts.cjs，移入 devDependencies
51. `docs/qa/must-verify-manually.md` — 4 项真机必过；"不凭主观印象宣称 ≥95%"
52. `CHANGELOG.md` 0.7.34「验证边界」— "fp32 的固定版本与摘要已经核对及下载回归，尚未完成完整真实推理"
53. `docs/dependency-policy.md` — "单测用桩件替代 sherpa-onnx…CI 绿灯不能证明真实识别行为未变"

**PerryLink/dsh-talk @ 6a5f768b2607ec1204fc256ede492c2abd39a3e1**
54. `package.json` — 0.3.26、Apache-2.0、engines.dsh ">=0.2.1-0 <0.3.0"、dependencies{zod:4.4.3}、dshWorkshop.compatibility.dshVersions 末项 0.2.1-alpha.1、evidence 四项 null
55. `src/engine.ts:53-105` — edgeTtsArgv / piperSpec / whisperArgv 全为外部可执行文件
56. `src/config.ts:439,453-457` — 默认命令名 'whisper-cli'/'edge-tts'/'piper'；:478-479 funasr 必须有 url
57. `src/client/vad.ts` — rmsOf / isSpeechFrame / advanceSilence / silenceTrips（纯能量）
58. `src/client/TalkMicButton.tsx:241-242` — 按下即 interrupt()；:437-462 整段音频 new Audio(url).play()
59. `src/service.ts:518-526` — interrupt() = controller.abort()
60. `src/speech.ts` — appendSpeechEvent 受 KNOWN_SESSION_EVENT_TYPES 门禁
61. `src/tool.ts` — speak 工具，audioPath 回传落盘路径
62. grep：TalkMicButton.tsx 中 caption|字幕 = 0；src/client/index.ts 仅注册 conversation.input.left + settings.plugins.tab

**ccch1mneyyy/dsh-Go @ 8fe1c1cd804ad763dd357bc1dec4b2cbfdc6f7fa**
63. `docs/plugins/voice.md` — 四能力表；"media 只回完整的一段…不支持流式"；16MiB；"voice…App 不会把声音或文字给你"；"退到后台、锁屏、换对话、换主机都立刻停"
64. `docs/design/pets.md` §0 — "宠物是状态指示器，不是养成游戏…dsh-pet 每 5 分钟调一次模型说话，是我们明确不做的反例"；§1.1 dsh-pet 批判条目（碎碎念 288 次/天、/chat 不鉴权、462px、素材不可商用）
65. `app/lib/pets/pet_mood.dart:346-354` — 优先级表 error 8 / notification 7 / sweeping 6 / attention 5 / carrying,juggling 4 / working 3 / thinking 2 / idle,waking 1 / sleeping 0
66. `app/lib/pets/pet_state.dart:14-25` — 12 态枚举
67. `docs/design/voice-input.md` §1/§2.1 — SpeechRecognizer 在国产 ROM 不可用；sherpa-onnx-node 1.13.8 + SenseVoiceSmall INT8 241MB 已在运行时；arm64 构建报告含 sherpa-onnx.node
68. `docs/plugins/web-panels.md` — 岛屿降级规则（本机引擎 + 桥接占用表 + 路由守卫 + 预算）、12s 超时降级、面板岛 30s 回收
69. `examples/plugins/dsh-go-example-readaloud/lib/index.js` — ctx.dshGo.media('speech') + micro('caption') + onCapability 接 voice.speechStart

---

# 七、NOT_TESTED 清单（本轮完全没跑的）

**桌宠**
1. 未安装、未激活 zhu 版 dsh-pet（且 engines.dsh ">=0.2.1-rc.3" 与宿主 0.2.1-rc.2 冲突，按纪律也不能装）。
2. Live2D 渲染未运行：Cubism Core 需用户自备，本机 $DSH_HOME/pets/.runtime/ 有没有该文件未检查，即使有也没加载。
3. 未验证任何 /api/pet/* 路由真实响应（全部结论来自 routes.ts 源码 + PetStateView 类型定义）。
4. 未验证亲密度/小鱼干结算、玩法 verb、装饰条渲染、碎碎念节奏。
5. 未跑 surfcore 桌宠（进程已死，pid 31524 不存在；只读了 docs/PET.md 与 skins/*/manifest.json）。
6. 未验证 feiyu-mood /dsh-feiyu-mood/state 的实际 HTTP 返回（只读 lib/index.js 源码）；"本地 state.json 当前 mood=70、dims 全 0、timeline=[]" 来自 00-SHARED-FACTS.md，本报告未重新解析。
7. 三个桌宠的视觉一致性/帧率/耗电完全没有测量。
8. 未验证 PC2005 仓库 0.4.1 与本地 0.3.8 的实际行为差异（只比了 package.json 字段与目录结构）。

**语音**
9. voice-mode 未安装、未启动、未进过语音模式。中文识别字错率、真实延迟、打断体感全部 NOT_TESTED；525ms 是仓库 9 月的历史实测值，本轮未复现。
10. 模型未下载，未验证 sherpa-onnx 在本机（Windows x64 / Node 24）能否加载、fork 子进程能否起。
11. Edge TTS 云端未验证（CHANGELOG 记录 2026-10-26 微软改握手导致整体无声，0.7.31 修，本轮无法确认当前是否仍可用）。
12. dsh-talk 未安装；未验证 edge-tts/piper/whisper-cli 二进制是否存在（PATH 上大概率没有）。
13. 两者都没有做"同时装会不会冲突"的实测——"不许两个都装"这条结论来自源码级冲突分析（输入槽 / llm stream / 麦克风独占 / 设置覆盖层），非实测。
14. Android 侧：DSH-Go App 未构建、未在模拟器/真机运行；media 16MiB / micro 10Hz / voice.speechStart / 悬浮字幕全部只读规格，零实测。
15. DSH-Go 语音套件（speech/catalog 等）未启用、模型未下载；arm64 proot 下 sherpa-onnx.node 能否加载没有任何实测证据（voice-input.md 自认"需要真机"）。
16. "在 DSH-Go 的 Web 岛屿里跑 voice-mode"这条路：岛屿能否拿到麦克风权限、能否维持 SSE、后台是否被回收 —— 全部未知，未测，且仓库文档无承诺。

**跨模块**
17. 本报告没有任何"性能/耗电/内存"数字是实测的；引用的 360-396MB 内存、0.26-0.49s 推理等均来自 dsh-Go voice-input.md 的 x86_64 自测，不是本机数据。
18. 未验证 dsh-pet 与 dsh-feiyu-mood 的桥接改造是否能编译/运行（方案 A/B 均为设计，未写代码）。
19. 未验证 dsh-Go 宠物包格式（pack.json / 12 态 / 图集契约）能否直接吃下 surfcore 的分层 PNG rig —— 只读了 manifest.json 与 pets.md，没有做格式映射验证。
