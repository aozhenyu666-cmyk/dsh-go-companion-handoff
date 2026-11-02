# 10-20 · dsh-pet（zhu1090093659/dsh-pet）源码级审计

> 审计对象：https://github.com/zhu1090093659/dsh-pet （npm 包名 `@linxin666/dsh-pet`，版本 **0.4.14**）
> 审计时间：2026-11-02
> 克隆位置：`fishy-audit/repos/dsh-pet`（`git clone --filter=blob:none --sparse --depth 1`，sparse 只取 `src contracts shared scripts tests docs .agents assets/whale assets/whale-refined assets/decorations`，未拉 298MB 大素材）
> HEAD：`9ed843bccee3d2604ce0ac31f55cbde9bda97c7f` · `2026-11-02 15:47:53 +0800` · `chore(release): bump to 0.4.14`
> 证据口径：**[已验证]** = 读过源码或跑过命令；**[未验证]** = 仅 README 描述；**NOT_TESTED** = 需要运行环境才能确认（本审计**没有**安装/运行这个插件，遵守只读纪律）

---

## 0 · 一句话结论（先看这段）

| 问题 | 答案 |
|---|---|
| 能否作为「大肥鱼」的**视觉外壳**（Android） | **不能直接用**。它是**纯 Web/浏览器**插件（React + CSS sprite + 可选 PixiJS/WebGL），宿主是 DSH Web GUI，**零 Android/原生渲染代码**。 |
| 有没有可复用的东西 | **有，而且很多**：状态机、事件投影、亲密度账本、小鱼干经济、文案引擎、玩法引擎、manifest 校验器全是**纯 TypeScript、无 DOM、无 React**，合计约 6~7k 行，可**逐文件搬到 Kotlin**（逻辑同构），是这份仓库最大的价值。 |
| 情绪表情能不能接 dsh-feiyu-mood | 仓库里**完全没有** mood/emotion/情绪概念（grep 0 命中）。**唯一的情绪接缝是 Live2D 的 `expressions`（phase→表情名）**，而 phase 是**闭集 7 值**、且没有任何对外 API 能写 phase。**要接 mood/dims 必须自己改代码**；Android 端自己定义映射反而更省事。 |
| 和本地已装的 dsh-pet 0.3.8 是同一个吗 | **不是同一项目，只是同名**。本地是 `PC2005-cloud/dsh-pet`（MIT，5.3k stars，`.webm` 视频桌宠 + Electron 桌面模式）；本仓库是 `@linxin666/dsh-pet`（Apache-2.0，60 stars，sprite2d/Live2D/frames2d 三渲染器）。两者**代码零共享、资产零共享、数据格式不兼容**。 |

---

## 1 · 仓库与获取（验证记录）

### 1.1 元数据 [已验证，GitHub API]

| 字段 | zhu1090093659/dsh-pet | PC2005-cloud/dsh-pet（本地已装） |
|---|---|---|
| stars | **60** | **5326** |
| license（API） | Apache-2.0 | MIT |
| GitHub size | 390,024 KB ≈ 381 MB | 139,138 KB ≈ 136 MB |
| created | 2026-09-23 | 2026-08-14 |
| pushed | 2026-11-02T08:15Z | 2026-11-02T13:38Z |
| 仓库名 / 包名 | `dsh-pet` / `@linxin666/dsh-pet` | `dsh-pet` / `dsh-pet` |

### 1.2 体积构成 [已验证，`git ls-tree` + GitHub tree API]

全仓库 3962 个 blob，**300.8 MB**，其中：

```text
298.3MB  assets/          ← 素材，占 99%
  1.1MB  src/
  0.8MB  docs/
  0.2MB  tests/
  0.1MB  .agents/  pnpm-lock.yaml
```

素材目录明细：

```text
113.9MB assets/jyn              84.5MB assets/jyn-loli
 55.5MB assets/doro             14.9MB assets/jyn-foxtail
 12.9MB assets/miku              3.6MB assets/long-niang
  3.3MB assets/deep-chan         2.9MB assets/ouo-neko
  2.2MB assets/whale-refined     2.0MB assets/whale
  1.8MB assets/blue-throated-bee-eater
  0.8MB assets/starry-doll       0.0MB assets/decorations
```

代码量 [已验证]：`src/` 非测试 TS/TSX **17,002 行**，`src/` 内测试 9,418 行，`tests/` 4,621 行，共 **52 个测试文件**。

> ⚠️ **`package.json` 里没有任何大素材的运行时依赖**：`files` 白名单只发 `assets/whale`、`assets/whale-refined`、`assets/ouo-neko`、`assets/blue-throated-bee-eater`、`assets/decorations`。jyn / jyn-loli / doro / miku / deep-chan / starry-doll / long-niang **只在仓库里，不随 npm 包发布**（README.zh.md 明确写了）。

### 1.3 它到底是什么 [已验证]

`package.json`：

```json
{
  "name": "@linxin666/dsh-pet",
  "version": "0.4.14",
  "description": "Multi-pet companion plugin for the dsh web GUI: a registry-driven floating pet that reacts to model activity, with per-pet naming, petting/feeding interactions and an affinity score",
  "license": "Apache-2.0",
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },
    "client": { "platform": "web", "inject": ["@deepseek-ai/dsh-client-connection", "..."] }
  },
  "peerDependencies": { "@deepseek-ai/dsh": ">=0.2.1-rc.3", "react": "^18.2.0", "react-dom": "^18.2.0" }
}
```

**结论：这是 DSH Web GUI 的 cordis 插件（host 半区 + browser 半区），不是独立应用，不含任何桌面/移动原生壳。** [已验证]

---

## 2 · 渲染方式

### 2.1 三渲染器（manifest v2 显式声明）[已验证]

`src/manifest-v2.ts:34`

```ts
/** Renderer kinds the pet center knows how to dispatch (M1 `2). */
export const PET_RENDERER_KINDS = ['sprite2d', 'live2d', 'frames2d'] as const
export type PetRendererKind = (typeof PET_RENDERER_KINDS)[number]
```

| renderer | 技术栈 | 你的用途 |
|---|---|---|
| `sprite2d` | **DOM + CSS**：一张 8列×9行 图集，用 `background-position` 切帧，`setTimeout` 定时器推进（**不是** rAF，也**不是** canvas） | 最省事，Android 用 `Image`+`srcRect` 或 Compose `Canvas` 直接等价 |
| `frames2d` | **Canvas 2D**（`createImageBitmap` 解码 + 单个 `<canvas>` 绘制），退化路径是 `<img>.src` 换图 | 目录式帧序列（每轨一个目录），最灵活，**推荐大肥鱼用这个** |
| `live2d` | **PixiJS 8 + untitled-pixi-live2d-engine + Cubism Core（WebGL）** | 重，且 Cubism Core 需用户自备、禁止再分发 |

### 2.2 sprite2d 的渲染内核：CSS background-position [已验证]

`src/client/PetSprite.tsx:801-817`

```tsx
<div
  ref={spriteRef}
  className={styles.sprite}
  style={{
    width: spriteWidth,
    height: spriteHeight,
    ...(props.visual === undefined
      ? {
          backgroundImage: imageReady ? 'url(' + definition.atlasUrl + ')' : undefined,
          backgroundSize: (scaled.width * columns) + 'px ' + (scaled.height * (definition.atlasRows ?? rows.length)) + 'px',
          backgroundRepeat: 'no-repeat',
          backgroundPosition: '0 0',
        }
      : {}),
    cursor: dragRef.current === null ? 'grab' : 'grabbing',
    ...(facingRight ? { transform: 'scaleX(-1)' } : {}),
  }}
```

帧循环是 `setTimeout` 自调度（刻意不用 60fps rAF，因为帧时长 300~700ms）：`src/client/PetSprite.tsx:392-502`，关键片段：

```ts
// src/client/PetSprite.tsx:455-498
const schedule = (delay: number): void => {
  timer = window.setTimeout(tick, Math.max(1, delay))
}
const tick = (): void => {
  timer = 0
  const now = performance.now()
  const delta = now - last
  last = now
  if (timeline !== undefined && sequenceItems !== undefined) {
    sequenceElapsed += delta
    const current = timeline.frameAt(sequenceElapsed)
    const item = sequenceItems.get(current.animation)!
    paint(item.row, item.track.frames[current.frameIndex]!)
    schedule(timeline.remainingAt(sequenceElapsed))
    return
  }
  const st = frameRef.current
  if (st.track !== animation) {
    st.track = animation
    st.index = 0
    st.elapsed = 0
  }
  st.elapsed += delta
  const maxIndex = track.frames.length - 1
  while (st.elapsed >= (track.durations[st.index] ?? 0) && st.index < maxIndex) {
    st.elapsed -= track.durations[st.index] ?? 0
    st.index += 1
  }
  if (st.elapsed >= (track.durations[st.index] ?? 0)) {
    if (!track.loop) {
      st.index = maxIndex
      paint(row, track.frames[st.index]!)
      return
    }
    st.elapsed = 0
    st.index = 0
  }
  paint(row, track.frames[st.index]!)
  schedule((track.durations[st.index] ?? 0) - st.elapsed)
}
schedule(0)
```

几何计算（**这段是可直接照搬到 Kotlin 的纯函数**）：`src/client/spritesheet.ts:36-62`

```ts
export function scaledCellSize(cell: PetCell, scale: number): { width: number; height: number } {
  return {
    width: Math.max(1, Math.round(cell.width * scale)),
    height: Math.max(1, Math.round(cell.height * scale)),
  }
}

export function framePosition(cell: PetCell, row: number, col: number, scale = 1): { x: number; y: number } {
  const scaled = scaledCellSize(cell, scale)
  return { x: -col * scaled.width, y: -row * scaled.height }
}
```

> **注意**：整数化单元格是有意为之（issue #17）——非整数 cell 会让合成器每帧重采样整张图集。Android 上同理，务必用整数 Rect。

### 2.3 图集行序契约（9 行固定）[已验证]

`src/state.ts:17-26` + `src/state.ts:84-97`

```ts
export type PetAnimation =
  | 'idle' | 'running-right' | 'running-left' | 'waving' | 'jumping'
  | 'failed' | 'waiting' | 'running' | 'review'

export function rowOf(animation: PetAnimation): number {
  const rows: Record<PetAnimation, number> = {
    'idle': 0, 'running-right': 1, 'running-left': 2, 'waving': 3, 'jumping': 4,
    'failed': 5, 'waiting': 6, 'running': 7, 'review': 8,
  }
  return rows[animation]
}
```

默认帧数表 / 默认节奏表：`src/registry.ts:76`（`[6,8,8,4,5,8,6,6,6]`）、`src/registry.ts:124-138`（每轨 durations/loop/fallback）。

### 2.4 frames2d：Canvas 位图缓冲 [已验证]

`src/client/renderers/frames2d.ts:1-27` 模块头（原样）：

```text
Frame presentation has two modes picked once at mount by capability
probing:
- Canvas bitmap buffer (default where createImageBitmap/fetch/2D context
  exist): frames are decoded once into an ImageBitmap on demand, with a
  bounded look-ahead window over the playing track, and drawn onto one
  <canvas>. Steady-state playback issues zero DOM mutations and re-decodes
  only what the retention budget evicted: decoded frames are held under
  MAX_RETAINED_BYTES, ...
- Classic <img> fallback (jsdom/tests or missing APIs): identical to the
  historical behavior - cache-warm Image elements plus guarded src swaps, ...
```

接口（`src/client/renderers/frames2d.ts:67-78`）：

```ts
export interface Frames2dRendererHandle extends PetRendererHandle {
  /** Force a track id (gameplay override); undefined returns to phase mapping. */
  setState(track: string | undefined): void
  /** Swap the base idle target ... to a declared looping track — a skin. */
  setIdleTrack(track: string | undefined): void
  /** The track currently playing (diagnostics and tests). */
  currentTrack(): string
}
```

### 2.5 Live2D：PixiJS + 懒加载 vendor [已验证]

`src/client/renderers/live2d/runtime.ts:16-18`

```ts
/** Runtime file URLs the host serves ('/api/pet/runtime/<name>', M3-2). */
const CORE_URL = '/api/pet/runtime/live2dcubismcore.min.js'
const VENDOR_URL = '/api/pet/runtime/live2d-vendor.js'
```

`src/client/renderers/live2d.ts:234-247` 建 Pixi Application：

```ts
const pixiApp = new vendor.Application()
const initialSize = normalizeRendererSize(ctx.container.clientWidth, ctx.container.clientHeight) ?? {
  width: 160, height: 174,
}
try {
  await pixiApp.init({
    width: initialSize.width, height: initialSize.height,
    backgroundAlpha: 0, antialias: true, autoDensity: true,
    preference: 'webgl',
  })
}
```

vendor 打包配置（`tsdown.live2d-vendor.ts`）：`pixi.js` + `untitled-pixi-live2d-engine` 打成 **IIFE**，全局名 `window.__dshPetLive2d`，`external: []`（全内联）。

**关键限制 [已验证]**：仓库里 **0 个 `model3.json`**（GitHub tree API 全量扫过）。Live2D 通道只有代码没有模型——README 说 Cubism Core 与模型都要用户自备，属实。

### 2.6 挂载点：全局 React root 挂 `document.body`（不是 session slot）[已验证]

`src/client/index.ts:629-637`

```ts
const container = document.createElement('div')
container.dataset.dshPetRoot = ''
container.dataset.dshPlugin = 'pet'
document.body.appendChild(container)
const petRoot = createRoot(container)
```

### 2.7 渲染层结论

- **没有任何原生/移动渲染代码**，也没有 PWA/WebView 封装。 [已验证]
- 渲染是「DOM 精灵（CSS background-position）+ 可选 Canvas 2D + 可选 WebGL/Live2D」三选一。 [已验证]
- 若要搬到 Android：**Live2D 是最不划算的一条**（Cubism Core 有专有许可、仓库不带模型、WebGL 在 Android 需要 SurfaceView/GLSurfaceView 重做）。**frames2d 的目录式帧序列是最合适大肥鱼的形态**。 [推断，基于源码]

---
## 3 · 宠物状态机

### 3.1 状态集合：活动相位 7 个 → 动画轨道 9 条 [已验证]

`src/state.ts:14-26`

```ts
/** Activity phases understood by the pet host. */
export type ActivityPhase = 'idle' | 'waiting' | 'thinking' | 'tool' | 'review' | 'done' | 'failed'

/** The Codex-compatible 9-state animation contract (spritesheet rows). */
export type PetAnimation =
  | 'idle' | 'running-right' | 'running-left' | 'waving' | 'jumping'
  | 'failed' | 'waiting' | 'running' | 'review'
```

相位→动画的映射是**写死的 switch**（`src/state.ts:71-81`）：

```ts
export function animationForPhase(phase: ActivityPhase): PetAnimation {
  switch (phase) {
    case 'thinking': return 'running'
    case 'tool': return 'running-right'
    case 'review': return 'review'
    case 'waiting': return 'waiting'
    case 'done': return 'jumping'
    case 'failed': return 'failed'
    case 'idle': return 'idle'
  }
}
```

状态机本体（`src/state.ts:103-165`，纯函数、注入时钟、无副作用）：

```ts
export class PetStateMachine {
  private phase: ActivityPhase = 'idle'
  private line: string | undefined
  private phrase: string | undefined
  private sessionActive = false
  private doneAt: number | undefined
  private failedAt: number | undefined

  onActivityStatus(input: PetStateInput): void {
    this.phase = input.phase
    this.line = input.line
    this.phrase = input.phrase
    this.doneAt = input.phase === 'done' ? this.now() : undefined
    this.failedAt = input.phase === 'failed' ? this.now() : undefined
  }

  render(): PetStateSnapshot {
    const nowMs = this.now()
    let animation = animationForPhase(this.phase)
    const doneSettled = this.phase === 'done' && this.doneAt !== undefined
      && nowMs - this.doneAt >= this.config.celebrateMs
    const failedSettled = this.phase === 'failed' && this.failedAt !== undefined
      && nowMs - this.failedAt >= this.config.failureMs
    if (doneSettled || failedSettled) animation = 'idle'
    const settled = this.phase === 'idle' || doneSettled || failedSettled
    const bubble = settled ? undefined : this.phrase ?? this.line
    return { animation, ...(bubble === undefined ? {} : { bubble }), animationStartedAt: nowMs, phase: this.phase, sessionActive: this.sessionActive }
  }
}
```

**时间常量**（`src/state.ts` 的 `defaultPetStateConfig`）：

```ts
export const defaultPetStateConfig: PetStateConfig = { celebrateMs: 2400, failureMs: 2400 }
```

### 3.2 切换触发：hook 了哪些事件 [已验证]

`src/service.ts:633-702`（`syncActivity()`）注册了 **3 个监听器**：

| # | 事件名 | 载荷形状 | 作用 |
|---|---|---|---|
| 1 | `session/event` | `(session: Session, event: SessionEvent)` | 官方会话事件投影；兼容 `activity/status` |
| 2 | `agent/assistant-stream` | `{ agent: { session }, frame: AssistantStreamFrame }` | 流式增量（推理/正文 delta） |
| 3 | `session/disposed` | `(session: Session)` | 会话销毁，回落或归 idle |

原文（`src/service.ts:640-701`，节选）：

```ts
const disposers = [
  this.ctx.on('session/event', (session: Session, event: SessionEvent) => {
    const runtime = this.activityOf(session).runtime
    if ((event.type as string) === 'activity/status') {
      const payload = ((event as unknown as { data?: unknown }).data ?? {}) as ActivityStatusEventLike
      if (typeof payload.phase !== 'string' || !isActivityPhase(payload.phase)) return
      this.applyActivity(session, {
        phase: payload.phase,
        ...(typeof payload.line === 'string' ? { line: payload.line } : {}),
        ...(typeof payload.phrase === 'string' ? { phrase: payload.phrase } : {}),
      })
      if (payload.phase === 'done' && !runtime.officialEventsSeen) this.rewardLegacyTurn()
      return
    }
    const transition = projectOfficialEvent(event, runtime)
    if (transition === undefined) return
    runtime.officialEventsSeen = true
    this.officialEventSessions.add(session)
    this.applyActivity(session, transition.input, transition.whisper)
    if (transition.completedTurn !== undefined) this.rewardTurn(String(session.id), transition.completedTurn)
  }),
  this.ctx.on('agent/assistant-stream', ({ agent, frame }) => {
    const session = agent.session
    const runtime = this.activityOf(session).runtime
    const transition = projectAssistantStreamFrame(frame, runtime)
    if (transition === undefined) return
    ...
  }),
  this.ctx.on('session/disposed', (session: Session) => { ... }),
]
```

### 3.3 「模型活动」→ 相位的完整投影表 [已验证]

`src/event-projection.ts:97-201`（`projectOfficialEvent`）：

| SessionEvent.type | 命中条件 | 相位 | 气泡文案来源 |
|---|---|---|---|
| `turn/start` | — | `waiting` | `voice.scene('prepare')` |
| `step/start` | — | `waiting` | `voice.scene('waiting')` |
| `assistant/message` | — | `review` | `voice.scene('review')` |
| `tool/call` | — | `tool` | `voice.tool(name, displayName, argHint)` |
| `tool/result` | 还有活跃工具 | `tool` | `voice.toolRemaining(n)` |
| `tool/result` | 本步有失败 | `failed` | `voice.scene('toolFailed')` |
| `tool/result` | 无失败 | `thinking` | `voice.scene('toolResult')` |
| `turn/end` | `reason.kind==='completed'` | `done` | `voice.scene('done')` |
| `turn/end` | `'error'` | `failed` | `voice.scene('failed')` |
| `turn/end` | `'max-tokens'` | `failed` | `voice.scene('maxTokens')` |
| `turn/end` | `'interrupted'` | `failed` | `voice.scene('interrupted')` |
| `turn/end` | `'blocked'` | `waiting` | `voice.scene('blocked')` |
| `turn/end` | `'aborted'` / 未知 | `idle` | 无（静默） |

代码（`src/event-projection.ts:114-133`，`tool/call` 分支，含碎碎念触发）：

```ts
case 'tool/call': {
  const callId = String(event.data.callId)
  runtime.activeTools.add(callId)
  // A test-looking call is remembered so its result (the only place the
  // outcome is known) can wake the test-green mood later.
  if (looksLikeTestTool(event.data.name, event.data.arguments)) runtime.testCalls.add(callId)
  const whisper = runtime.whispers.feed(whisperCategoryOf(toolCategory(event.data.name)), nowMs)
  return {
    input: {
      phase: 'tool',
      line: runtime.voice.tool(
        event.data.name,
        displayToolName(event.data.name),
        toolArgHint(event.data.name, event.data.arguments),
        nowMs,
      ),
    },
    ...(whisper === undefined ? {} : { whisper }),
  }
}
```

流式增量投影（`src/event-projection.ts:208-230`）：

```ts
export function projectAssistantStreamFrame(frame, runtime, nowMs = Date.now()) {
  if (frame.type !== 'chunk') return undefined
  const { chunk } = frame
  if (chunk.type === 'reasoning-delta' && chunk.text.length > 0) {
    const whisper = runtime.whispers.feed('thinking', nowMs)
    return { input: { phase: 'thinking', line: runtime.voice.scene('thinking', nowMs) }, ...(whisper === undefined ? {} : { whisper }) }
  }
  if (chunk.type === 'text-delta' && chunk.text.length > 0) {
    const whisper = runtime.whispers.feed('writing', nowMs)
    return { input: { phase: 'review', line: runtime.voice.scene('review', nowMs) }, ...(whisper === undefined ? {} : { whisper }) }
  }
  return undefined
}
```

### 3.4 多会话语义 [已验证]

- 每个会话一个 `SessionActivity`（`runtime` 投影器 + 自己的 `PetStateMachine`），`src/service.ts:308-321`。
- **最后到达有意义事件的会话**决定精灵动画（`displaySession`）；每个活跃顶层会话各占一条气泡，上限 `MAX_SESSION_BUBBLES = 12`（`src/service.ts:188`）。
- 子代理会话（`session.header.origin === 'subagent'`）**不占气泡位**，仍参与动画与计奖（`src/service.ts:1072`）。

### 3.5 场景序列（可选，让动画更像"戏"）[已验证]

manifest 可声明 `sequences`：场景→至少 5 条动画轨的循环序列，比如内置鲸鱼娘（`assets/whale/pet.json`）：

```json
"sequences": {
  "thinking": ["running", "running-right", "running", "running-left", "running", "waiting", "running"],
  "done": ["jumping", "waving", "jumping", "waving", "jumping", "idle"]
}
```

计时器是纯函数：`src/client/sequences.ts:36-72`（`createSequenceTimeline`，含 `frameAt(elapsedMs)` / `remainingAt(elapsedMs)`）。

---

## 4 · 情绪表情（**关键问题**）

### 4.1 结论 [已验证]

1. **sprite2d / frames2d 通道根本没有"表情"这个概念。** 只有 9 条固定动画轨（sprite2d）或任意命名轨道（frames2d），没有任何表情参数化。
2. **Live2D 通道有 `expressions`，但它是 phase→表情名的静态映射**，phase 是**闭集 7 值**。
3. **没有任何 API 能让外部写 phase 或写表情。** 全仓库 grep `setPhase|forcePhase|injectPhase|setEmotion|setAnimation` = **0 命中**。
4. **仓库里 grep `mood`/`emotion`/`情绪` 在 `src/` 下 = 0 命中**（唯一 `mood` 是 gameplay 的可选属性名 `stats.mood`，纯 HUD 血条，见 4.4 节）。

### 4.2 Live2D expressions 的真实定义 [已验证]

类型（`src/manifest-v2.ts:59-74`）：

```ts
export interface PetManifestLive2d {
  model: string
  scale?: number
  translate?: { x?: number; y?: number }
  /** ActivityPhase -> motion group name; idle is required, unmapped phases fall back to idle. */
  motions: Partial<Record<ActivityPhase, string>> & { idle: string }
  /** Optional ActivityPhase -> expression name layered over the motion. */
  expressions?: Partial<Record<ActivityPhase, string>>
  /** Hit area names triggering pet.interact; defaults to every model HitArea. */
  hitAreas?: string[]
  /** Whisper-paced lip sync (post-M3). */
  lipSync?: boolean
}
```

应用点（`src/client/renderers/live2d.ts:216-221`）：

```ts
const applyPhase = (phase: ActivityPhase): void => {
  phaseGroup = config.motions[phase] ?? config.motions.idle
  playGroup(phaseGroup)
  const expression = config.expressions?.[phase]
  if (expression !== undefined && model !== undefined) void model.expression(expression)
}
```

**也就是说：表情只在相位切换时被套一次，且相位只有 7 个。** 想做「开心/失落/害羞/吃醋」这类情绪表情，7 个相位不够用，而 `ActivityPhase` 是被引用了 **60+ 处**的闭集并集类型——**改它要动整条链路**（`state.ts` 的 union + `animationForPhase` + `rowOf` + `registry.ts` 的 9 轨表 + 所有 manifest 校验 + 前端帧循环）。**不值得。**

### 4.3 能不能"外部注入情绪"——三条可行路径（按代价排序）

#### 路径 A（**推荐给 Android**）：完全绕开这个仓库，自己定义情绪层
在 Android 端自己造：`mood:Int` + `dims:Map<String,Int>` → `Emotion` 枚举 → 你的帧轨道/表情资源。**参考 dsh-feiyu-mood 现成的映射算法**（见 4.5 节）。零改动、零风险。

#### 路径 B（Web 端可行、Android 无关）：用 `activity/status` 兼容事件硬塞相位
`src/service.ts:646-661` 表明该事件**没有白名单**：只要有插件 `ctx.emit('session/event', fakeSession, { type: 'activity/status', data: { phase, line, phrase } })`，service 就会把它当成真实活动应用（`isActivityPhase` 只校验 7 个合法值）。

- 优点：不用改 pet 源码，能驱动 7 个相位里的任意一个 + 自定义气泡文案。
- 代价：相位仍是 7 值；会与真实会话事件抢 `displaySession`；类型上需 `declare module '@deepseek-ai/dsh-session'` 扩展 `SessionEventMap`（仓库自己的测试就这么干的，`tests/service-enabled.spec.ts:45-53`）。
- **状态：可行但 hacky。未实测。**

#### 路径 C（改源码）：把 phase 换成开放字符串
把 `ActivityPhase` 从 union 改成 `string`，`animationForPhase` 改成表格查 + fallback。**这等于重写状态机契约**，会连锁破坏 manifest v2 的 fail-closed 校验。**不推荐。**

### 4.4 `stats.mood` 是假的情绪 [已验证]

`src/gameplay.ts:47-55`

```ts
export interface PetGameplayStatDef {
  max: number
  initial?: number
  decayPerMinute?: number
  /** Decay rate while the work mode is active (defaults to decayPerMinute). */
  workingDecayPerMinute?: number
  /** Extra decay rate while no session is active. */
  idleDecayPerMinute?: number
}
```

`stats` 是**任意命名数值条**（jyn-loli 声明了 hunger/mood/energy/affection 四条），按分钟线性衰减，只用于：

1. HUD 菜单卡的血条（`src/client/gameplay-hud.tsx:546-559`）；
2. 触摸分支/打工/商店的加成效果。

**它不驱动任何动画轨道**（已验证：`src/client/renderers/Frames2dVisualMount.tsx` 对 `stats` 0 命中，`PetSprite.tsx` 里 `stats` 只出现在面板文案格式化 `panel.stats`）。

衰减实现（`src/gameplay.ts:844-894`）：

```ts
export function settleGameplay(
  state: PetGameplayState,
  manifest: PetGameplayManifest,
  now: number,
  options: { sessionActive: boolean },
): boolean {
  const elapsedMs = now - state.settledAt
  if (elapsedMs <= 0) return false
  const minutes = elapsedMs / 60_000
  let changed = false
  for (const [name, def] of Object.entries(manifest.stats ?? {})) {
    const current = state.stats[name]
    if (current === undefined || current <= 0) continue
    let rate = def.decayPerMinute ?? 0
    if (state.mode === 'work' && def.workingDecayPerMinute !== undefined) rate = def.workingDecayPerMinute
    if (!options.sessionActive) rate += def.idleDecayPerMinute ?? 0
    if (rate <= 0) continue
    const next = Math.max(0, current - rate * minutes)
    if (next !== current) { state.stats[name] = next; changed = true }
  }
  ...
}
```

> 这是**惰性结算**（lazy settle）：没有定时器，靠 `settledAt` 时间差算，读取时算但不写盘（`src/service.ts:1098-1106` 注释确认）。这个模式很适合 Android（后台被杀了也不丢时间）。

### 4.5 dsh-feiyu-mood 的实际数据接口（Android 直接可读）[已验证，读 lib/index.js]

它本地装在 `~/.dsh/profiles/desktop/node_modules/dsh-feiyu-mood`（v0.8.1，MIT，无 src，只有打包后的 `lib/index.js` 1900 行 + 独立前端 `assets/widget.js`）。

**HTTP 端点**（`lib/index.js:1617-1704`）：`GET /dsh-feiyu-mood/state?key=<份id>`

返回（`lib/index.js:1646-1696`，原样字段名）：

```js
const out = {
  ok: true,
  key: 这份, mode: cfg.模式, slots: 份列表, multi: 份列表.length > 1,
  label: String(这份份.label || ''),
  name: String(这份份.selfName || cfg.selfName),
  ownerName: 对方名,
  mood: Math.round(s.mood),           // 0..100
  moodLabel: moodLabel(Math.round(s.mood)),
  emoji: moodEmoji(Math.round(s.mood)),
  dims: DIMS.map(([k, label]) => ({ key: k, label, value: Math.round(s.dims[k]) })),
  lastReason: s.lastReason, lastConfidence: s.lastConfidence, updatedAt: s.updatedAt,
  injectMode: cfg.injectMode, injectConfidence: cfg.injectConfidence, injectPerDay: cfg.injectPerDay,
  saidToday: 今天说过,
  injectQuota: { said: 今天说过, cap: 每天上限, left: 还剩 },
  decayHalfHours: cfg.decayHalfHours, sigMood: cfg.sigMood, sigDim: cfg.sigDim,
  minor: Boolean(s.minor), kind: String(s.kind || ''),
  injectable: ...,
  lastInjectedText: s.lastInjectedText, lastInjectedAt: s.lastInjectedAt,
  injectPreview: { state: 'off'|'minor'|'blocked'|'pending'|'said', text, ... },
  timeline: s.timeline,
}
```

**7 个 dims 键**（`lib/index.js:50-58`）：

```js
const DIMS = [
  ['happy', '开心'], ['calm', '踏实'], ['sweet', '甜蜜'], ['heartache', '心疼'],
  ['worry', '担心'], ['sad', '难过'], ['jealous', '吃醋'],
]
```

**mood→标签/表情的现成分档**（`lib/index.js:870-884`）：

```js
function moodLabel(v) {
  if (v >= 80) return '欢喜'
  if (v >= 62) return '开心'
  if (v >= 42) return '一般'
  if (v >= 25) return '心疼'
  return '难过'
}
function moodEmoji(v) {
  if (v >= 80) return '😊'
  if (v >= 62) return '🙂'
  if (v >= 42) return '😌'
  if (v >= 25) return '🥺'
  return '😢'
}
```

**它自己已有的 5 张情绪脸素材**（`assets/face-great.png`…，映射见 `assets/widget.js:17-21`）：

```js
if (v >= 80) return { img: B + '/face-great.png', label: '欢喜' }
if (v >= 62) return { img: B + '/face-good.png',  label: '开心' }
if (v >= 42) return { img: B + '/face-ok.png',    label: '一般' }
if (v >= 25) return { img: B + '/face-worry.png', label: '心疼' }
return         { img: B + '/face-sad.png',   label: '难过' }
```

落盘（`~/.dsh/feiyu-mood/state.json`，本机实测存在）：

```json
{"version":2,"moods":{"__default__":{"mood":70,"dims":{"happy":0,"calm":0,"sweet":0,"heartache":0,"worry":0,"sad":0,"jealous":0},
 "lastReason":"","lastConfidence":0,"injectedFor":0,"lastInjectedText":"","lastInjectedAt":0,"saidAt":[],
 "lastNotedMood":70,"minor":false,"kind":"","updatedAt":1793535279201,"lastDecayAt":1793629364455,"timeline":[]}},
 "lastActive":"__default__"}
```

> **给 Android 的落地建议**：不要在 dsh-pet 里接，**在 Android 端读 `GET /dsh-feiyu-mood/state`**（或直接读 `state.json`），拿到 `mood` + `dims`，用自己的映射表转成：
> - **表情帧组**（frames2d 风格目录，如 `face/happy/`）
> - **叠加配件**（眼泪/爱心/问号贴图，见 10.3 节）
> - **可选**：把 `mood` 当作 `affection` 的另一个输入
>
> 这套映射**本来就要重写**，所以「接不进 dsh-pet」对 Android 不是损失。

---
## 5 · 触摸 / 互动 / 亲密度

### 5.1 交互只有两种：`pet`（摸头）和 `feed`（喂食）[已验证]

`src/affinity.ts:14-16`

```ts
/** One interaction the user can perform on the pet. */
export type PetInteraction = 'pet' | 'feed'
```

### 5.2 数据结构（持久化落 $DSH_HOME/pet.json）[已验证]

`src/affinity.ts:17-37`

```ts
export interface AffinityState {
  points: number        // 总亲密度，上限 AFFINITY_MAX
  lastPetAt: number     // 上次摸头 epoch ms
  lastFeedAt: number    // 上次喂食 epoch ms
  pets: number          // 累计摸头次数
  feeds: number         // 累计喂食次数
  petRejects: number    // 被冷却拒绝的摸头次数
  feedRejects: number   // 被冷却拒绝的喂食次数
  turns: number         // 见证的完成轮数
}
export const AFFINITY_MAX = 999_999_999
```

9 级等级阶梯（`src/affinity.ts:47-57`）：

```ts
export const AFFINITY_RANKS = [
  { min: 0, name: '幼鲸', emoji: '*' },
  { min: 25, name: '伙伴', emoji: '**' },
  { min: 50, name: '挚友', emoji: '***' },
  { min: 80, name: '深海羁绊', emoji: '****' },
  { min: 200, name: '心有灵犀', emoji: '*****' },
  { min: 500, name: '传说羁绊', emoji: '******' },
  { min: 2_000, name: '神话羁绊', emoji: '*******' },
  { min: 10_000, name: '永恒之契', emoji: '********' },
  { min: 100_000, name: '鲸生共渡', emoji: '*********' },
] as const
```

可被 voice.json 的 `panel.ranks` **只改显示名**（`src/affinity.ts:139-166` 的 `rankOf`；`src/voice-pack.ts:241` 的 `normalizeRankLadder`，限制 1–16 条、首条 min=0、名字 1–24 字符）。

### 5.3 落盘文件：$DSH_HOME/pet.json（单文件原子写）[已验证]

`src/persist.ts:87-119`

```ts
export interface PetPersist {
  petId: string                        // 选中的宠物 id
  names: Record<string, string>        // 每只宠物的改名（按 id）
  skins: Record<string, string>        // 每只宠物选中的 frames2d 皮肤 id
  affinity: AffinityState
  treats: TreatLedger                  // 小鱼干库存账本
  display: PetDisplayConfig
  gameplay: Record<string, PetGameplayState>   // 每只宠物的玩法状态（stats/currencies/mode）
}
```

`src/persist.ts:275-282`

```ts
/** Atomically persist state (write temp + rename). */
export function savePetPersist(data: PetPersist, dir: string = petHomeDir()): void {
  mkdirSync(dir, { recursive: true })
  const target = join(dir, 'pet.json')
  const tmp = target + '.tmp'
  writeFileSync(tmp, JSON.stringify(data, null, 2), 'utf8')
  renameSync(tmp, target)
}
```

- 路径解析 `dshHome()`：`$DSH_HOME` 环境变量优先，否则 `~/.dsh`（`src/dsh-home.ts:31-42`）。**本机 `~/.dsh/pet.json` 目前不存在**（只有 `~/.dsh/dsh-pet/main-config.jsonc`，那是另一个插件的）——说明这个插件在本机**从未真正跑起来过**。 [已验证]
- 容错读取：文件缺失/损坏/JSON 坏 → 全默认（`src/persist.ts:217` 起 `loadPetPersist`，整体 try/catch）。旧版扁平 `name` 字段会迁移到 `names[petId]`。

### 5.4 冷却与奖励公式 [已验证]

`src/affinity.ts:96-110`

```ts
export const defaultAffinityConfig: AffinityConfig = {
  turnReward: 1,
  petReward: 1,
  petCooldownMs: 10_000,
  feedReward: 5,
  feedCooldownMs: 30_000,
}
```

`applyInteraction`（`src/affinity.ts:220-268`）语义：**首次交互（last*At === 0）永不冷却**；冷却内 delta=0 且 petRejects/feedRejects +1；文案按**持久化计数器**轮询（`countedRemark`），不是随机。

### 5.5 小鱼干经济 [已验证]

`src/treats.ts:16-30`

```ts
export interface TreatConfig {
  turnsPerTreat: number   // 每 30 轮完成给 1 条
  timeTreatMs: number     // 每 5 小时给 1 条
  maxTreats: number       // 库存上限 20
}
export const defaultTreatConfig: TreatConfig = {
  turnsPerTreat: 30,
  timeTreatMs: 300 * 60_000,
  maxTreats: 20,
}
```

结算函数 `settleTreatGrants`（`src/treats.ts:71-113`）双源独立：work 源按 `turns` delta，time 源按 `lastTreatGrantAt`；**首次零收益也会写时间锚**（代码注释点名这是"锚点死锁"修复）。

### 5.6 交互的完整调用链 [已验证]

```text
点击精灵 (PetSprite.tsx:821 onClick)
  → props.onPet()                        // PetDockEntry → PetInjected.pet
  → petApi.interact('pet')               // client/index.ts:88  POST /api/pet/interact {kind}
  → postRoute('.../interact')            // routes.ts:585 校验 kind ∈ {pet,feed}
  → service.interact(kind)               // service.ts:753
  → ledger.interact(kind, nowMs)         // ledger.ts:254  settleTreats → applyInteraction → picker.pickAt
  → takeDirty() ? flush() : -            // service.ts:756  写 pet.json
  → 返回 LedgerInteractionResult {reaction, delta, affinity}
  → setFeedback({text: result.reaction, kind:'pet', at})   // client/index.ts:522-531
  → feedback 气泡 2600ms 后自动清（PetSprite.tsx:519-523）
```

返回结构（`src/ledger.ts:43-50`）：

```ts
export interface LedgerInteractionResult {
  reaction: string          // 反应气泡文案
  delta: number             // 实际加点（冷却内为 0）
  affinity: PetAffinityView // 完整亲密度快照
}
```

只读视图（`src/affinity.ts:168-183`）：

```ts
export interface PetAffinityView {
  points: number
  rank: string
  rankEn?: string
  rankEmoji: string
  pets: number
  feeds: number
  turns: number
  petCooldown: boolean
  feedCooldown: boolean
}
```

### 5.7 拖拽 / 位置持久化 [已验证]

`src/client/PetSprite.tsx:541-572`：pointerdown 记录起点，移动超过 **4px** 才算拖（避免拖完 trailing click 又触发摸头，`draggedRef`）；pointerup 通过 `props.onDragEnd(right, bottom)` → `POST /api/pet/set-config {right, bottom}`。

位置字段在 `src/persist.ts:19-45`：`{ visible, size, right, bottom, bubbleScale }`，范围 `32..1024`（size）、`0..10000`（inset）。

---

## 6 · 内心独白 / 主动气泡

### 6.1 结论：**全部是插件自己生成的文案，一次模型都不调** [已验证]

全仓库 grep `ctx.llm` / `llm.stream` / `agentDefaultModel` / `systemPrompt` = **0 命中**（`dsh-llm` 只在测试文件里作为类型 import 出现一次）。`package.json` 的 `dependencies` 只有 `@deepseek-ai/schemastery` 和 `clsx`。

> **这和本地那个 dsh-pet（PC2005 版）是完全相反的设计** —— 那个版本碎碎念/对话**会调 LLM**（见 9.2 节）。

### 6.2 文案引擎结构 [已验证]

`src/chatter.ts` 847 行，两个扬声器：

| 扬声器 | 用途 | 触发 |
|---|---|---|
| `StatusVoice` | 状态气泡文案（"正在用 bash"等） | 相位/工具事件，**同场景 4s 内稳定、超过 4s 换一句** |
| `WhisperEngine` | 碎碎念（内心独白） | 情境 + 结构化结果，**有冷却** |

池表：
- 状态池 11 个场景：`prepare / waiting / thinking / review / toolResult / done / failed / toolFailed / maxTokens / interrupted / blocked`（`src/chatter.ts:31-43`），每池 8 句左右。
- 工具池 17 个族：`read / write / edit / shell / grep / find / ls / webSearch / webFetch / mcp / memory / subagent / todo / browser / git / ask / generic`（`src/chatter.ts:184-234`），支持 `{tool}` / `{hint}` 占位符。
- 碎碎念 10 个分类：`thinking / writing / reading / editing / running / searching / git / delegating / browsing / generic`（`src/chatter.ts:524-540`）。
- 碎碎念 3 个结果：`pass / fail / done`（`src/chatter.ts:521`）。

### 6.3 气泡触发条件（精确数字）[已验证]

`src/chatter.ts:544-550`

```ts
/** Murmur pacing: the cooldown between category whispers. */
export const WHISPER_COOLDOWN_MS = 9000
/** Outcome whispers get their own shorter cooldown so a real moment still speaks. */
export const WHISPER_RESULT_COOLDOWN_MS = 5000
/** How long a whisper stays on screen (host-side expiry). */
export const WHISPER_TTL_MS = 8000
```

`src/chatter.ts:45`：`export const STATUS_ROTATE_MS = 4000`（同场景文案轮换周期）。

碎碎念引擎（`src/chatter.ts:784-846`）：

```ts
export class WhisperEngine {
  feed(category: WhisperCategory, nowMs: number): string | undefined {
    if (nowMs - this.lastWhisperAt < this.categoryCooldownMs) return undefined
    const pool = this.categoryPool(category)
    if (pool.length === 0) return undefined
    const index = (this.categoryCursor.get(category) ?? 0) % pool.length
    this.categoryCursor.set(category, index + 1)
    return this.speak(pool[index]!, nowMs)
  }
  result(kind: WhisperResult, nowMs: number): string | undefined {
    if (nowMs - this.lastWhisperAt < this.resultCooldownMs) return undefined
    ...
  }
  private speak(line: string, nowMs: number): string {
    this.lastWhisperAt = nowMs
    return line
  }
}
```

**冷却计时器是共享的**：`lastWhisperAt` 只有一个，所以 category 与 result 互相压制（先到先得）。

### 6.4 触发点（谁喂它）[已验证]

| 事件 | 喂的分类/结果 | 出处 |
|---|---|---|
| `tool/call` | `whisperCategoryOf(toolCategory(name))` | event-projection.ts:118 |
| `tool/result` 失败 | `result('fail')` | event-projection.ts:146 |
| `tool/result` 且是测试工具且通过 | `result('pass')` | event-projection.ts:148 |
| `turn/end completed` | `result('done')` | event-projection.ts:167 |
| `turn/end error` | `result('fail')` | event-projection.ts:178 |
| 流式 `reasoning-delta` | `feed('thinking')` | event-projection.ts:214 |
| 流式 `text-delta` | `feed('writing')` | event-projection.ts:222 |

**「是测试工具」的判定**（`src/chatter.ts:590-617`）：工具名正则命中 `test|spec|vitest|jest|pytest|mocha|playwright|cypress|karma`，或参数里的 `command` / `code` 命中 `(pnpm|npm|yarn|npx|bun|python) (run )?tests?`。

**碎碎念"绝不读模型原文"**（`src/chatter.ts:13-15` 注释）：永远只输出池子里的固定句，不拼接工具名/路径/模型文本。

### 6.5 主动气泡的另外两条通道 [已验证]

**（1）`pet.announce()`** —— 兄弟插件推结构化公告（余额/消费/套餐），载荷 `src/announce.ts:18-40`：

```ts
export interface PetAnnouncement {
  source: string                   // 槽位键（发布方）
  kind: 'balance' | 'cost' | 'plan'
  title: string
  amount?: string                  // balance / cost 必填
  percent?: number                 // plan 必填，0-100
  resetAt?: string                 // ISO 8601
  note?: string
  tone: 'ok' | 'warn' | 'low'
  ttlMs: number
  at: number
}
```

约束：`MAX_ANNOUNCEMENTS = 4`（`src/announce.ts:54`）、TTL 收 1s–2h（`ANNOUNCE_MAX_TTL_MS = 7_200_000`，`src/announce.ts:62`）、TITLE_MAX=80 / TEXT_MAX=120 / SOURCE_MAX=64。**纯内存，不落盘。**

**（2）`statusBubbles` 开关** —— `'auto' | 'off'`（`src/service.ts:146`），`off` 只关宠物自带的状态气泡，公告与互动反馈照常。**这正是"插件要自己画气泡面"时该用的开关。**

---

## 7 · 人格与状态接口（对外暴露的 API / 事件 / ctx 服务）

### 7.1 Cordis 服务：`ctx.pet` [已验证]

`src/service.ts:303-307`

```ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    pet: PetService
  }
}
```

构造函数 `super(ctx, 'pet')`（`src/service.ts:390`）。

`PetService` 的**全部公开成员**（`src/service.ts`，行号已验证）：

| 成员 | 行号 | 签名 |
|---|---|---|
| `isEnabled` | 440 | `(): boolean` |
| `state` | 478 | `async (currentSessionId?: string): Promise<PetStateView>` |
| `announce` | 495 | `(input: unknown): { ok: boolean }` |
| `registerPanelAction` | 546 | `(action: PetPanelActionRegistration): () => void` |
| `selectPanelAction` | 559 | `async (id: string): Promise<{ok:boolean; error?:string}>` |
| `display` | 569 | `(): PetDisplayConfig` |
| `pets` | 574 | `async (): Promise<PetDefinition[]>` |
| `registrySnapshot` | 579 | `(): PetRegistry` |
| `diagnostics` | 584 | `async (): Promise<{diagnostics: PetRegistryDiagnostic[]}>` |
| `activeEntry` | 599 | `(): PetEntry` |
| `selectedPetId` | 604 | `(): string` |
| `petName` | 609 | `(petId?: string): string` |
| `setPetId` | 616 | `async (petId): Promise<{ok:true;petId:string} | {ok:false;error:string}>` |
| `setEnabled` | 627 | `(enabled: boolean): void` |
| `interact` | 753 | `async (kind: PetInteraction): Promise<PetInteractResult>` |
| `gameplayTouch` | 824 | `async (zone?: string): Promise<PetGameplayVerbResult>` |
| `gameplaySetMode` | 868 | `async (mode: string | null): Promise<PetGameplayVerbResult>` |
| `gameplayWorkTick` | 887 | `async (): Promise<PetGameplayVerbResult>` |
| `gameplayBuy` | 903 | `async (itemId: string): Promise<PetGameplayVerbResult>` |
| `setVisible` | 937 | `async (visible: boolean): Promise<{ok:true; display}>` |
| `setSkin` | 960 | `async (skin?: string): Promise<...>` |
| `setConfig` | 975 | `async (patch: Partial<PetDisplayConfig>): Promise<...>` |
| `setName` | 988 | `async (name: string): Promise<...>` |
| `applySettingsSection` | 1003 | `(section: PetSettingsSection): void` |
| `settingsNamespaceProvider` | 387 | `(() => string | undefined) | undefined`（属性） |

> ⚠️ **没有任何 write-phase / write-emotion / force-animation 的接口。** 这是 4 节 结论的源码级依据。

`registerPanelAction` 的注册契约（`src/panel-actions.ts:65-80`）：

```ts
export interface PetPanelActionRegistration {
  id: string      // ^[a-z0-9][a-z0-9._-]{0,63}$
  label: string   // ≤24
  title?: string  // ≤80
  order?: number
  onSelect: PetPanelActionHandler   // (ctx: { petId: string }) => void | Promise<void>
}
export const MAX_PANEL_ACTIONS = 8
```

### 7.2 HTTP API（同源 JSON，默认仅 loopback）[已验证]

`src/routes.ts:29` `PET_API_PREFIX = '/api/pet'`；`src/routes.ts:574-670` `makePetRoutes`：

| 方法 | 路径 | 请求体 | 返回 |
|---|---|---|---|
| GET | `/api/pet/state?current=<sessionId>` | — | `PetStateView` |
| GET | `/api/pet/pets` | — | `PetDefinition[]` |
| GET | `/api/pet/diagnostics` | — | `{diagnostics}` |
| POST | `/api/pet/interact` | `{kind:'pet'|'feed'}` | `LedgerInteractionResult` |
| POST | `/api/pet/set-visible` | `{visible:boolean}` | `{ok, display}` |
| POST | `/api/pet/set-config` | `{size?,right?,bottom?,visible?}` | `{ok, display}` |
| POST | `/api/pet/set-name` | `{name:string}` | `{ok, name}` |
| POST | `/api/pet/set-skin` | `{skin?:string}` | `{ok, skin?}` |
| POST | `/api/pet/set-pet` | `{petId:string}` | `{ok, petId}` |
| POST | `/api/pet/panel-action` | `{id:string}` | `{ok, error?}` |
| POST | `/api/pet/gameplay/touch` | `{zone?:string}` | `PetGameplayVerbResult` |
| POST | `/api/pet/gameplay/mode` | `{mode:string|null}` | 同上 |
| POST | `/api/pet/gameplay/work-tick` | — | 同上 |
| POST | `/api/pet/gameplay/buy` | `{item:string}` | 同上 |
| GET/HEAD | `/pet/<id>/<file>` | — | 素材（realpath 包含性 + 大小上限） |
| GET | `/api/pet/runtime/<name>` | — | Cubism Core / vendor bundle |
| GET | `/api/pet/decoration/<id>/<file>` | — | 装饰条 |

安全围栏（`src/routes.ts:137-142` + `src/access.ts` + `src/loopback.ts`）：socket 必须是 loopback **且** Host 头是 loopback，`sec-fetch-site: cross-site` 直接拒，Origin 必须同 host；否则 `403 {"ok":false,"error":"forbidden: loopback-only"}`。

大小上限（`src/routes.ts:43` `PET_ASSET_CAPS`）：manifest 64KB / image 20MB / model 32MB。

### 7.3 状态快照 `PetStateView`（这是"外部读宠物状态"的唯一出口）[已验证]

`src/service.ts:191-274`，完整字段：

```ts
export interface PetStateView {
  animation: PetStateSnapshot['animation']   // 9 值之一
  bubble?: string
  phase: PetStateSnapshot['phase']           // 7 值之一
  sessionActive: boolean
  sessions?: PetSessionView[]                // 每会话一条气泡，≤12
  affinity: PetAffinityView
  display: PetDisplayConfig
  pet: { id: string; displayName: string; description: string }
  name: string
  skin?: string
  treats: { stocked: number; max: number }
  decoration?: DecorationView
  gameplay?: PetGameplayStateView            // { stats: Record<string,number>; mode: string|null }
  announcement?: PetAnnouncement             // 兼容单槽
  announcements?: PetAnnouncement[]          // 每 source 一槽
  statusBubbles?: PetStatusBubbleMode
  panelActions?: PetPanelActionView[]
}
```

`PetSessionView`（`src/service.ts:174-185`）：`{ sessionId, animation, bubble, phase, whisper? }`。

### 7.4 hook 的宿主事件（只读消费）[已验证]

| 事件 | 出处 | 用途 |
|---|---|---|
| `session/event` | service.ts:641 | 主投影源 |
| `agent/assistant-stream` | service.ts:672 | 流式相位 |
| `session/disposed` | service.ts:681 | 清理/回落 |
| `loader/volatile-update` | index.ts:432 | 设置项热更新 |
| 兼容输入 `activity/status` | service.ts:646 | 外部塞相位的口子（4.3 节 路径 B） |

插件**自己派发**的事件：未发现 `ctx.emit`；对外全部走 `ctx.pet` 方法与 HTTP 端点。

---
## 8 · 资产格式与授权

### 8.1 三种资产格式 [已验证]

| renderer | 目录结构 | 格式 |
|---|---|---|
| `sprite2d` | `<pet>/pet.json` + `spritesheet.webp`（+ `previews/*.gif`） | 8 列 × 9 行（v1）/ 11 行（v2 带视线行）；默认 cell **192×208**；deep-chan 用 256×288 |
| `frames2d` | `<pet>/pet.json` + `thumb/<track>/<frame>.webp` | 每帧 webp/png/gif/jpg；帧时长可编进文件名 `<base>_<index>_<ms>.webp`，默认 200ms（jyn 系列 40ms） |
| `live2d` | `<pet>/pet.json` + `model/*.model3.json` + 引用闭包 | Cubism `.moc3` + 贴图 + motion/physics/pose/expression JSON |

内置鲸鱼娘图集实测 [已验证，手工解析 VP8L 头]：`assets/whale/spritesheet.webp` = **1536×1872**，1,544,154 bytes，正好 8×192 × 9×208。与 README 一致。

decoration 条带（`assets/decorations/whale/decoration.json`，原样）：

```json
{
  "decorationManifestVersion": 1,
  "id": "whale",
  "displayName": "喷水鲸鱼",
  "license": "MIT",
  "entry": "whale-frames.png",
  "cell": { "width": 64, "height": 48 },
  "columns": 4,
  "frameMs": 160,
  "loop": true,
  "phases": {
    "idle": "hide",
    "waiting": { "from": 0, "to": 1 },
    "thinking": { "from": 0, "to": 3 },
    "tool": { "from": 0, "to": 3 },
    "review": { "from": 0, "to": 1 },
    "done": { "from": 2, "to": 3 },
    "failed": { "from": 3, "to": 3 }
  }
}
```

### 8.2 授权清单（逐只宠物，读各自 pet.json 的 `license` 字段）[已验证]

| 资产目录 | renderer | `license` 字段实际值 | author | 随 npm 包发布 |
|---|---|---|---|---|
| `whale`（鲸鱼娘原版） | sprite2d | **BSD-3-Clause** | — | 是 |
| `whale-refined`（精致版） | sprite2d | **MIT** | — | 是 |
| `ouo-neko` | sprite2d | **MIT** | Pessimist0906 | 是 |
| `blue-throated-bee-eater` | sprite2d | **Apache-2.0** | dsh-web | 是 |
| `decorations/whale` | — | **MIT** | — | 是 |
| `jyn` / `jyn-foxtail` / `jyn-loli` | frames2d | MIT | 11726 | **否**（只在仓库） |
| `doro` | frames2d | MIT | 11726 | **否** |
| `miku` | frames2d | MIT（角色受 Piapro 许可） | stushansusu | **否** |
| `deep-chan` | sprite2d | **Personal-Use-Only** | Daizhdd | **否** |
| `starry-doll` | sprite2d | **CC-BY-NC-SA-4.0** | Theater-ahyeon | **否** |
| `long-niang` | sprite2d | **CC-BY-NC-SA-4.0** | ZipZipPipe / zhangyue311 | **否** |

### 8.3 授权上的三个坑（重要）

**(a) 仓库自己的 license 声明自相矛盾** [已验证]

- `LICENSE` 文件正文 = **Apache License 2.0**
- `package.json` 写 `"license": "Apache-2.0"`
- `README.zh.md` 末尾写的是 `[BSD-3-Clause](LICENSE)` ← **错**
- 但 `assets/whale/pet.json` 自己写 `"license": "BSD-3-Clause"`

→ 若要用，**以 Apache-2.0 为准**（LICENSE 与 package.json 一致），并保留 NOTICE。

**(b) 内置素材里混着"仅个人非商业"** [已验证，THIRD_PARTY_NOTICES.md]

- `deep-chan`：DeepSeek 鲸鱼娘**非官方同人**，"素材仅限个人非商业使用"
- `doro`：SHIFT UP《胜利女神：妮姬》二创，"个人非商业使用"
- `jyn` 系列：DeepSeek 鲸鱼 mark 的二创，"个人非商业使用"
- `miku`：Crypton / Piapro Character License

→ **这些一个都不能用于商业发布**。若「大肥鱼」要上线，**只能自己画或买素材**。

**(c) Live2D 的法律雷区** [已验证，README.zh.md]

- 插件**永不内置、永不代下 Cubism Core**（专有许可禁止再分发）——用户须自行从 Live2D 官方获取，放 `$DSH_HOME/pets/.runtime/live2dcubismcore.min.js`
- 官方示例模型（Hiyori / Haru）**仅供评估、禁止再分发**
- README 自己警告：本插件属 Live2D 术语下的「可扩展性 APP」，公开发布衍生作品**可能不论规模都须签发行许可**

→ **如果大肥鱼要 Live2D，Android 端要自己搞定 Cubism SDK for Native 的商业授权。这是法律成本，不只是技术成本。**

### 8.4 遥测 [已验证，`src/client/telemetry.ts`]

客户端每个 UTC 日向 `https://dsh-market.com/api/telemetry/event` 发一次心跳：`{kind:'heartbeat', visitor:<localStorage UUID>, items:[{name, version?, channel?}]}`。无会话内容；`navigator.webdriver` 为真时跳过。**复用代码时要把这个文件删掉**（Android 端不该向第三方发任何请求）。

---

## 9 · 与本地已装的 dsh-pet（PC2005-cloud v0.3.8）对比

### 9.1 结论：**不是同一个项目，只是同名** [已验证]

| 维度 | zhu1090093659/dsh-pet（本审计对象） | PC2005-cloud/dsh-pet v0.3.8（本地已装） |
|---|---|---|
| npm 包名 | `@linxin666/dsh-pet` | `dsh-pet` |
| 版本 | 0.4.14 | 0.3.8 |
| stars | 60 | 5326 |
| 许可证 | Apache-2.0（README 误写 BSD-3） | **MIT** |
| 体积 | 381 MB（仓库） | 136 MB（仓库）/ 本机安装 63.5 MB |
| 渲染 | CSS background-position 精灵 / Canvas 2D 帧序列 / PixiJS-Live2D | **`<video>` 播 `.webm`（VP9-alpha）/ `.mov`（HEVC-alpha）** |
| 动画素材 | 图集切帧 / 目录帧序列 | **106 个透明视频**（`assets/webm/`，本机实测） |
| 桌面模式 | 无（纯 Web） | **有**：Electron 透明置顶窗（自带 helper，会下载 Electron 到 `~/.dsh/electron`）+ 独立 CLI `dsh-pet-standalone` |
| 碎碎念 | **不调模型**，纯文案池 | **调 LLM**（`ctx.llm.stream`，30s 超时，人设 `whisperPrompt`） |
| 对话 | 无 | 有：右键「对话」+ `/chat` 命令，记忆存 `memory.json` |
| 亲密度 | **有**（points / 9 级 / 冷却 / 小鱼干） | **没有**（grep `affinity` / `亲密度` / `intimacy` / `好感` = 0 命中） |
| 物理 | 无（拖动只改 right/bottom） | **有**：弹簧跟手 + 甩抛 + 重力反弹 + 碰撞 + Q 弹（`src/shared/physics.ts` 587 行） |
| 宠物种类 | manifest 驱动（`pet.json` v1/v2） | `pet/<名>-animation/` 目录 + `<名>-config.json` 的 pet pack |
| 数据落盘 | `$DSH_HOME/pet.json`（单文件） | `~/.dsh/dsh-pet/main-config.jsonc`（配置）+ `memory.json`（对话）+ 内存态 S |
| 对外接口 | `ctx.pet` 服务 + `/api/pet/*` | `ctx.provide('petApi7340', {balance:{register,list}})` + `/dsh-pet-7340/*` |
| 路由前缀 | `/api/pet/` | `/dsh-pet-7340/` |

**没有一行代码是共享的**：目录结构、模块名、类型名、事件契约全部不同。

### 9.2 本地那份（PC2005 v0.3.8）的关键事实（供对比参考）[已验证]

- 渲染：`src/client/pet.ts:2236-2237` 两个 `<video>` 元素交替播放（A/B 双缓冲无缝衔接）。
- 扩展名选择：`src/client/pet.ts:117-135` `prefersMov()` —— 非 Chromium/Firefox 且 macOS 才要 `.mov`，否则 `.webm`。
- 工作状态档位：`src/host/work-status.ts:16` `['thinking','working','result','waiting','success','error']` **6 档**（本仓库是 7 相位）。
- 展示优先级：`src/host/work-status.ts:152` `waiting 60 > error 50 > working 40 > thinking 30 > result 25 > success 20`。
- **有外部写状态的端点**：`POST /dsh-pet-7340/work-status`（`src/host/index.ts:1523`），body `{state, task?}`，用固定外部槽 `EXTERNAL_WORK_SESSION = 'api'`，参与同一套优先级。`DELETE` = 清空回空闲。
- **有外部塞气泡的端点**：`POST /dsh-pet-7340/broadcast?pet=<id>`（`src/host/index.ts:1699`），body `{text, image?, url?, duration?, anim?}` ——**不生成，调用方自己给文本**。
- **有点播动画**：`POST /dsh-pet-7340/anim`（`src/host/index.ts:1808`），body `{name}` 或 `{url, facing?}`。
- **有占用锁**：`/lock` / `/lock/release`（`src/host/index.ts:1604`）。
- LLM 人设：`assets/config.jsonc` 的 `whisperPrompt` 字段（条目级，实例可用 `pets[].whisperPrompt` 覆盖）。

> **对 Android 的启示**：本地这个 v0.3.8 的**接口设计**（外部写 work-status / broadcast / anim / lock）比 zhu 版更适合"外部程序驱动桌宠"；而且**它的素材是 webm，Android 的 `MediaPlayer`/`ExoPlayer` 可以直接播**，zhu 版是图集必须自己切。反过来 zhu 版的**心智模型（状态机 / 亲密度 / 玩法）更完整**。两者可以取长补短，但不能混用代码。

---

## 10 · 结论：能否作为「大肥鱼」的视觉外壳

### 10.1 总判定

| 用法 | 判定 |
|---|---|
| 直接把 dsh-pet 装进 Android App 当外壳 | **不行**。它是 DSH Web GUI 的 cordis 插件，依赖 `@deepseek-ai/dsh-client-*` 模块表、React 18、`window.__ModuleLoader__`、同源 `/api/pet/*`；Android 端没有这套宿主。 |
| 用 WebView 包一层跑它 | **理论可行但代价高**：需要自建一个假 DSH Web 宿主（模块表 + 路由 + 事件桥），且 300MB 素材、Live2D 许可、遥测上报、loopback 围栏（WebView 自定义 scheme 会 403）全要处理。**不建议。** |
| **把它的「逻辑层」搬成 Kotlin** | **强烈推荐**。下面 10.2 列的 12 个纯 TS 文件（约 5,000 行，零 DOM 零 React）可**逐函数翻译**，语义完全确定。 |
| **把它的「渲染层」搬到 Android 原生** | 推荐 **frames2d 模型**（目录帧序列 + Canvas/Compose 绘制），**放弃 Live2D**。 |
| **接 dsh-feiyu-mood** | 可以，但**不经过 dsh-pet**。Android 端直接读 `/dsh-feiyu-mood/state`（或 `state.json`），自己映射。 |

### 10.2 可直接复用的逻辑（清单 + 证据）

**全部为纯 TS / 零 DOM / 零 React**（已逐文件验证 import 语句）：

| # | 文件 | 行数 | 内容 | 搬运价值 |
|---|---|---|---|---|
| 1 | `src/state.ts` | 167 | `ActivityPhase` / `PetAnimation` / `animationForPhase` / `rowOf` / `PetStateMachine` | 高 —— 状态机内核，直接照抄（含 2400ms 庆祝窗口） |
| 2 | `src/event-projection.ts` | 231 | `projectOfficialEvent` / `projectAssistantStreamFrame` | 高 —— 事件→相位映射表，Android 换成自己的事件源即可 |
| 3 | `src/affinity.ts` | 280 | 亲密度账本 + 9 级阶梯 + 冷却 + `applyInteraction` | 高 —— 纯函数，含所有边界（首次免冷却、上限 999,999,999） |
| 4 | `src/treats.ts` | 117 | 小鱼干双源结算（含锚点死锁修复） | 高 —— 惰性结算模式，适合移动端 |
| 5 | `src/chatter.ts` | 847 | `StatusVoice` + `WhisperEngine` + 全部文案池 + `toolCategory` + `toolArgHint` + `looksLikeTestTool` | 高 —— 中文文案资产 + 节流算法，**Android 可整段抄池子** |
| 6 | `src/remarks.ts` | 221 | 互动反应库（5 槽 × 16 句）+ `RemarkPicker` 轮询 | 中 —— 纯数据 + 20 行逻辑 |
| 7 | `src/gameplay.ts` | 942 | stats 衰减 / touch 分区 / work 循环 / sleep / modes / roam / passiveIncome / shop / lottery | 高 —— 整套小游戏引擎（manifest 驱动、fail-closed 解析） |
| 8 | `src/manifest-v2.ts` | 760 | manifest v2 校验（结构 fail-closed + 内容 warn-and-drop） | 中 —— 需先自定义 Android 端 manifest |
| 9 | `src/client/spritesheet.ts` | 73 | `scaledCellSize` / `framePosition` / `trimTrack` | 高 —— 图集几何，20 行 Kotlin 能重写但要保证整数化语义一致 |
| 10 | `src/client/sequences.ts` | 81 | `createSequenceTimeline` / `sequenceFrameAt` / `remainingAt` | 高 —— 场景序列计时，纯算术 |
| 11 | `src/client/phase-stream.ts` | 40 | 相位流（`get` / `subscribe` / `push`，仅变更时派发） | 中 —— Android 用 `StateFlow` 天然等价 |
| 12 | `src/announce.ts` | 121 | 公告载荷校验 + TTL | 中 —— 若要做"余额气泡"可抄边界值 |

额外值得抄的**设计模式**（不是代码）：

- **惰性结算**（`settledAt` 时间差，无定时器）：`gameplay.ts:844`、`treats.ts:71` —— 移动端后台被杀不丢进度。
- **整数化单元格**（`spritesheet.ts:36`，issue #17）：避免合成器每帧重采样整张图集。
- **setTimeout 自调度帧循环**（`PetSprite.tsx:455`，不用 60fps rAF）：帧时长 300–700ms 时省电。
- **状态池 + 4s 轮换 + 9s / 5s 冷却**（`chatter.ts:45`、`chatter.ts:546`）：让气泡"活着"但不刷屏。
- **文案按持久化计数器轮询而非随机**（`affinity.ts:189` `countedRemark`）：可测试、可复现。

### 10.3 必须改成 Android 原生组件的部分

| 原组件 | 原实现 | Android 必须重写 | 建议方案 |
|---|---|---|---|
| **精灵渲染** | `PetSprite.tsx` 1058 行，CSS `background-position` | 全量重写 | Compose `Canvas` 画 `ImageBitmap` 的 srcRect，或 `ImageView` + `BitmapShader` matrix。**保留 `framePosition` 的整数化公式** |
| **帧序列渲染** | `frames2d.ts` 601 行，Canvas 2D + `createImageBitmap` + 保留预算 | 全量重写 | `ImageDecoder` / `BitmapFactory` + `LruCache`（等价 `MAX_RETAINED_BYTES`）+ `Choreographer` 自调度 |
| **Live2D** | PixiJS 8 + untitled-pixi-live2d-engine + Cubism Web | **建议放弃** | 若一定要：Cubism SDK for Native（**商业授权**）+ `GLSurfaceView`。先算法律成本 |
| **浮层 / 拖动 / portal** | react-dom `createPortal` + pointer events + `document.body` | 全量重写 | `WindowManager` 悬浮窗（需 `SYSTEM_ALERT_WINDOW`）或 App 内 `Box` + `Modifier.draggable`（含 **4px 阈值**防误触） |
| **气泡 DOM** | `pet.module.css` + `aria-live` | 全量重写 | `Popup` / `Box` + `semantics { liveRegion }`；玻璃质感用 `RenderEffect` blur（API 31+）或半透明降级 |
| **设置页** | `PetSettingsCard.tsx` 640 行 + `PluginSettingsCard.tsx` 817 行 | 重写 | Compose 设置页 |
| **轮询** | `setInterval(pollNow, 2000)` + 页面可见性暂停（`index.ts:465-497`） | 重写 | `StateFlow` + `repeatOnLifecycle`；或直接接 `dsh-feiyu-app` 的 `/feiyu/*` |
| **持久化** | 单文件 `pet.json` + 原子 rename | 重写 | `DataStore`(Proto) 或 Room；**保留"临时文件 + rename"的原子语义** |
| **HTTP 围栏** | loopback + Host + Origin 校验 | 按需 | 若与 host 同机，用明文 loopback + 自定义 token |
| **遥测** | `telemetry.ts` 上报 dsh-market.com | **删除** | 不该有 |
| **减少动画偏好** | `prefers-reduced-motion` 停在首帧 | 重写 | 读 `Settings.Global.ANIMATOR_DURATION_SCALE == 0` 时停在首帧 |
| **React / cordis 依赖** | `ctx.pet` 服务、`ctx.on`、`ctx.effect`、`__ModuleLoader__` | 全丢 | 换成 Kotlin 的 `CoroutineScope` + `Flow` |

### 10.4 给 Android 开发者的建议架构（基于本审计的事实）

```text
┌──────────────────────────── Android App ────────────────────────────┐
│                                                                      │
│  [数据源]                                                            │
│   · GET /dsh-feiyu-mood/state   → mood:Int(0..100) + dims:Map(7键)   │
│   · （可选）/feiyu/history      → 会话活动流（dsh-feiyu-app v0.1.0）  │
│   · 本地 DataStore              → affinity / treats / gameplay /     │
│                                   display                            │
│                          │                                           │
│                          ▼                                           │
│  [纯逻辑层 · 直译自 TS]                                               │
│   PhaseMachine   ←  state.ts:103  (7 相位 + 2400ms 庆祝窗口)          │
│   AffinityLedger ←  affinity.ts:220 (pet +1/10s, feed +5/30s, 9 级)  │
│   TreatLedger    ←  treats.ts:71  (30 轮 / 5 小时, 上限 20, 惰性锚点) │
│   WhisperEngine  ←  chatter.ts:784 (9s / 5s 冷却, 8s TTL, 轮询池)     │
│   MoodMapper     ←  ★新写★  ← dsh-feiyu-mood 的 5 档阈值            │
│   GameplayEngine ←  gameplay.ts:844 (stats 衰减 / touch / work /     │
│                                      sleep)                          │
│                          │                                           │
│                          ▼                                           │
│  [渲染层 · 原生重写]                                                  │
│   FramePlayer    : ImageDecoder + LruCache + Choreographer           │
│   SpriteCanvas   : Canvas.drawBitmap(srcRect, dstRect, paint)        │
│   BubbleLayer    : Compose + liveRegion semantics                    │
│   FloatWindow    : WindowManager（或 App 内 Box）                     │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

**MoodMapper 的具体建议**（复用 4.5 节已验证的阈值 + 7 dims）：

```kotlin
// 阈值来源：dsh-feiyu-mood lib/index.js:870-884（已验证）
fun moodBand(mood: Int) = when {
    mood >= 80 -> Band.GREAT    // 欢喜 (😊 / face-great.png)
    mood >= 62 -> Band.GOOD     // 开心 (🙂 / face-good.png)
    mood >= 42 -> Band.OK       // 一般 (😌 / face-ok.png)
    mood >= 25 -> Band.WORRY    // 心疼 (🥺 / face-worry.png)
    else        -> Band.SAD     // 难过 (😢 / face-sad.png)
}
// 主导 dim（dims 值 >= 15 才参与，取 top1）→ 叠加配件
// dims 键固定 7 个：happy / calm / sweet / heartache / worry / sad / jealous
```

**核心原则**：`mood` 决定**底面表情帧组**（5 档），top-dim 决定**叠加配件**（7 种，如 sweet→爱心、worry→问号、sad→泪滴、jealous→黑线），`phase`（活动状态）决定**身体动作轨**。三个维度正交，不需要动 dsh-pet 的任何代码。

---

## 11 · 未验证 / 风险清单

| 项 | 状态 | 说明 |
|---|---|---|
| 插件实际运行效果 | **NOT_TESTED** | 全过程只读；未 `pnpm install`、未启动 `dsh web`、未渲染一帧。所有"渲染效果"结论来自源码阅读。 |
| `activity/status` 注入是否真的生效 | **未验证（读源码推断）** | `isActivityPhase` 只查 7 值合法性，`applyActivity` 无条件执行；但 `SessionEventMap` 未声明该事件，运行时会否被上游过滤**未测**。 |
| Live2D 渲染质量 / 性能 | **未验证** | 仓库 0 个 `model3.json`，无素材可跑。 |
| 300MB 素材能否直接转 Android | **未验证** | 只读了图集头部尺寸；未逐个解码 webp/gif 到 Android。 |
| `frames2d` 的 `MAX_RETAINED_BYTES` 具体值 | **未读全** | `frames2d.ts` 601 行，只读了头部注释与 handle 接口。 |
| 官方 DSH 版本兼容性 | **未验证** | `package.json` 要求 `@deepseek-ai/dsh >= 0.2.1-rc.3`；本机宿主是 `dsh-desktop-runtime 0.2.1-rc.2`（`BASELINE-latest.json` 记 `0.2.1-rc.2`）——**看起来不满足该下限**，可能装不上。 |
| README 的 `license: BSD-3-Clause` | **已验证是错的** | LICENSE 文件与 package.json 都是 Apache-2.0。 |
| 大肥鱼的商业发布 | **法律风险** | 内置素材 `deep-chan` / `doro` / `jyn` 系全部"仅个人非商业"；Live2D 需发行许可。 |

---

## 附录 A · 审计过程中跑过的命令（可复现）

```powershell
# 通过本机代理 7897 克隆（直连 github 超时：curl 28 / Recv failure）
git -c http.proxy=http://127.0.0.1:<proxy> clone --filter=blob:none --sparse --depth 1 ^
    https://github.com/zhu1090093659/dsh-pet.git repos/dsh-pet
git -c http.proxy=http://127.0.0.1:<proxy> sparse-checkout set src contracts shared scripts tests docs .agents
git -c http.proxy=http://127.0.0.1:<proxy> sparse-checkout add assets/whale assets/whale-refined assets/decorations

# 元数据 + 全量树（算体积 / 找 model3.json）
curl --proxy http://127.0.0.1:<proxy> https://api.github.com/repos/zhu1090093659/dsh-pet
curl --proxy http://127.0.0.1:<proxy> "https://api.github.com/repos/zhu1090093659/dsh-pet/git/trees/main?recursive=1"
```

## 附录 B · `src/` 逐文件行数（非测试）

```text
affinity.ts 280    announce.ts 121    chatter.ts 847    decoration.ts 220
defaults.ts 8      dsh-home.ts 43      event-projection.ts 231   gameplay.ts 942
http.ts 128        image-dimensions.ts 77   index.ts 455   ledger.ts 299
loopback.ts 64     manifest-v2.ts 760  model3.ts 92       mount-once.ts 143
pair-access.ts 73  panel-actions.ts 183 persist.ts 282    registry.ts 1268
remarks.ts 221     routes.ts 674       service.ts 1164    state.ts 167
treats.ts 117      voice-pack.ts 550
client/PetSprite.tsx 1058   client/index.ts 704   client/gameplay-hud.tsx 663
client/PluginSettingsCard.tsx 817   client/PetSettingsCard.tsx 640
client/renderers/frames2d.ts 601    client/renderers/live2d.ts 332
client/settings-form.ts 510 client/locales.ts 216  client/telemetry.ts 106
```

---

**报告结束。** 一句话给决策者：**这不是一个可以直接用的外壳，而是一套写得很干净的「桌宠逻辑引擎 + Web 渲染参考实现」。把逻辑引擎翻成 Kotlin，渲染按 frames2d 的目录帧序列用 Android 原生重画，情绪层自己接 dsh-feiyu-mood —— 这条路可行、成本可控、法律清爽。**
