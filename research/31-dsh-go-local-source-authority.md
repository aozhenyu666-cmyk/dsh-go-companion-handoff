# DSH-Go 本地源码权威依据：Android 原生插件接入

> 读取对象：**本地真实源码仓库** `D:\code\dshgo`（只读，未修改任何文件）
> 校验结果（`git` 实测）：分支 `feat/r3-candidate`，HEAD `bf9757631e43a2aea6052857d97ca0dbf9fbf0b8`，提交时间 `2026-10-31 14:44:39 +0800`，提交信息 `docs: handover to WorkBuddy - state, regression to fix, build traps, release steps`。
> 标注约定：**〔已读源码验证〕** = 我在本地源码里读到了实现/类型/常量；**〔文档说，代码未找到〕** = 只有文档声称，源码里没找到对应实现或证据。
> 所有代码片段保留原文（含注释），路径写相对仓库根的路径。

---

## 0. 先给结论：三层能力必须分开看

这张表是本报告最重要的一页。任务里提到的"网页插件 / 原生界面 / 系统级能力"三件事，在 dsh-go 里是**三套完全独立的机制**，权限、生命周期、可用 API 都不同。

| | A. DSH 网页端插件（web 半端 React） | B. DSH-Go 原生界面（dshGo 清单 + ctx.dshGo + 原生视图 JSON） | C. Android 系统级能力（浮窗 / 后台 / 无障碍 / 通知） |
|---|---|---|---|
| 代码跑在哪 | DSH 引擎里的 Node（React 渲染在 App 的 WebView / 网页岛里） | 逻辑跑在**引擎里的 Node**；App **只画**声明出来的 JSON 视图，**不执行插件代码** | App 自己的 Kotlin 代码（`app/android/...`） |
| 谁能用 | 所有主机（本机 / SSH / 中转） | 只有装了 `dsh-go-bridge` 的主机（本机引擎自带；SSH 未支持；中转不放行） | **只有 App 自己**。第三方 DSH 插件**完全拿不到** |
| 插件能直接做什么 | 任意网页 UI（React）、任意 JS | 只能返回 JSON 视图 + 一份白名单效果 | 浮窗、无障碍读屏、后台闹钟、通知…… |
| 插件能不能 | 画任意 React 界面 | 只能从 16 个 v1 / +26 个 v2 组件里挑 | **不能**。插件没有入口调用任何 Android API |
| 关键证据 | `docs/plugins/web-panels.md:3-9` | `docs/plugins/README.md:12`、`docs/design/native-plugins-protocol.md:143-190` | `AndroidManifest.xml`、`app/android/.../pet/`、`runtime/proactive/` |

三条最容易犯的错，直接给出否证：

1. **网页插件 ≠ Android 原生桌宠。**
   `docs/plugins/README.md:12`（原文）：
   > 插件照常是一个 DSH 插件（npm 包，逻辑跑在引擎里），只是多给 App 一份**声明**和一些 **JSON 视图**；App 用自己的组件画出来，**不执行插件的任何代码**。

   所以"网页端插件"能做的界面，换成 dshGo 清单后**只能重画成 16/42 种原生组件**，不能自定义颜色、字体、尺寸、动画、定位（`docs/plugins/design-rules.md:5-17`）。桌宠那种"全屏悬浮 + 逐帧动画 + 任意图形"在原生插件层**做不到**。

2. **App 内面板 ≠ 系统悬浮窗。**
   语音字幕（`overlay` 能力）的悬浮字幕是本仓库里最像"浮窗"的东西，但它**明写了不越过 App 边界**。
   `docs/design/native-plugins-protocol.md:557`（原文）：
   > `overlay` | 一行悬浮字幕，内容来自 `micro` 通道或静态文字 | App 画在自己的界面上（**不覆盖其他 App**），× 由 App 画

   实现注释 `app/android/.../nativemedia/MediaOverlays.kt:29-36`（原文）：
   > What a plugin's capabilities show over the app (**never over other apps**): the listening indicator and the one-line caption, stacked at the top centre just under the chat header. Both are the app's own views in the activity, so they are only there while the app is on screen and **need no window permission**.

   真正的跨 App 浮窗（`SYSTEM_ALERT_WINDOW`）只给 **App 自己的桌宠**用：`AndroidManifest.xml:30-31` 与 `pet/PetOverlay.kt`。**插件没有声明入口能拿到它**。

3. **Android 后台不能无限跑。**
   `docs/loop/VALIDATION.md:5`（原文）："本轮所有真机项都是 **NOT_TESTED**"。
   闹钟降级链在 `loop/core/ProactiveScheduler.kt:51-67` 和 `loop/proactive/WakeAlarm.kt:42-45`：系统限制后台时**根本不设闹钟**；强行停止后所有广播都收不到（`WakeAlarm.kt:42-45` 注释原文："After a force stop Android delivers none of these until DSH is opened again"）。

### 0.1 关于任务里的字段名

任务里问的 `composers strip` / `statusbar` / `messageMenu` / `toolCards` 这类键名，**源码里不存在**。真实键名是（`sdk/plugin-kit/schema/manifest.schema.json:43-151`）：

| 任务里的说法 | 源码里的真名 | 位置 |
|---|---|---|
| composers strip / composer.strip | `contributes."composer.strip"` | `manifest.schema.json:72-78` |
| statusbar | 同上（v1 状态条就是 `composer.strip` 的数据通道，不叫 statusbar） | `native-plugins-protocol.md:417-431` |
| messageMenu | `contributes."message.actions"` | `manifest.schema.json:59-71` |
| toolCards | **不占插槽、不写在清单里**，靠工具 `output.presentationMeta` 的 `"dsh-go/view"` 键 | `docs/plugins/quickstart.md:26-46`、`sdk/plugin-kit/lib/card.js:5` |
| panels / pane | `contributes.panels` + `ctx.dshGo.panel(id, …)` | `manifest.schema.json:79-85` |

`contributes` 的真实键全集只有 7 个：`composer.actions`、`message.actions`、`composer.strip`、`panels`、`media`、`micro`、`overlays`。证据 `runtime/bridge/dsh-go-bridge/lib/native/limits.js:64`：
```js
export const SURFACES = ["composer.actions", "message.actions", "composer.strip", "panels"];
```
```js
export const RESOURCE_KINDS = ["media", "micro", "overlays"];   // capabilities.js:18
```

---

## 1. `package.json` 里 `dshGo` 清单的完整字段表

### 1.1 顶层字段（权威：`sdk/plugin-kit/schema/manifest.schema.json`）

**共 8 个顶层键，全部都在 `TOP_KEYS` 白名单里**（`runtime/bridge/dsh-go-bridge/lib/native/manifest.js:10`，原文）：

```js
const TOP_KEYS = new Set(["v", "minApp", "title", "icon", "wants", "assets", "contributes", "capabilities"]);
```

| 字段 | 类型 | 必填 | 取值范围 / schema | 源码位置 |
|---|---|---|---|---|
| `v` | 整数 | **是** | `{"const": 1}`，写 `1` | `manifest.schema.json:12-15` |
| `minApp` | `"x.y.z"` | 否 | `"^\\d+\\.\\d+\\.\\d+$"` | `manifest.schema.json:16-19`，`$defs.semver:167-170` |
| `title` | Text | 否 | ≤ 24 字（按 locale 解析后） | `manifest.schema.json:20-23` |
| `icon` | IconName | 否 | `^[a-z][a-z0-9-]{0,31}$` | `manifest.schema.json:24-26`，`$defs.iconName:176-180` |
| `wants` | 数组 | 否 | `"draft" \| "clipboard" \| "message"`，uniqueItems | `manifest.schema.json:27-38` |
| `assets` | 相对路径 | 否 | `^(\./)?[^/\\][^\\]*$`，必须留在包目录内 | `manifest.schema.json:39-42`，`$defs.relativePath:201-205` |
| `contributes` | 对象 | **是** | 见下 1.2 | `manifest.schema.json:43-151` |
| `capabilities` | 数组 | 否 | `"audio" \| "voice" \| "micro" \| "overlay" \| "files"` | `manifest.schema.json:152-164` |

整个 `dshGo` 序列化后 **≤ 16 KiB**，超出整份作废（`manifest.schema.json:5` 原文 "Serialized size <= 16 KiB."；`runtime/bridge/dsh-go-bridge/lib/native/limits.js:6` `manifestBytes: 16384`；校验 `manifest.js:61-64`）。

### 1.2 `contributes` 下的 7 个键

权威来源 `manifest.schema.json:43-151`，运行时校验 `runtime/bridge/dsh-go-bridge/lib/native/manifest.js:11-16`（原文）：

```js
const ITEM_KEYS = {
  "composer.actions": new Set(["id", "label", "icon", "effect", "requires"]),
  "message.actions": new Set(["id", "label", "icon", "effect", "requires"]),
  "composer.strip": new Set(["id", "icon", "text", "panel"]),
  panels: new Set(["id", "title", "icon", "placement", "view", "micro"]),
};
```

#### (a) `composer.actions` — 输入框上方胶囊
`manifest.schema.json:46-58`：`maxItems: 2`；item 是 `$defs.action`；`label` 描述 "<= 10 characters"。
上限常量 `limits.js:7`：`"composer.actions": 2`，`textChars.composerAction`（`limits.js:8` = `composerAction: 10`）。

#### (b) `message.actions` — 消息长按菜单
`manifest.schema.json:59-71`：`maxItems: 2`；`label` "<= 12 characters"。

#### (c) `composer.strip` — 输入框上方状态条
`manifest.schema.json:72-78`：`maxItems: 1`。`$defs.strip`（`manifest.schema.json:253-274`）原文：

```json
"strip": {
  "type": "object",
  "required": ["id"],
  "properties": {
    "id": { "$ref": "#/$defs/id" },
    "icon": { "$ref": "#/$defs/iconName" },
    "text": { "$ref": "#/$defs/text", "description": "Static fallback, <= 24 characters." },
    "panel": { "$ref": "#/$defs/id", "description": "Own panel opened on tap." }
  }
}
```

#### (d) `panels`
`manifest.schema.json:79-85`：`maxItems: 4`。`$defs.panel`（`manifest.schema.json:275-315`）字段：`id`(必填)、`title`(必填, ≤16 字)、`icon`、`placement`(枚举 `session.tab`/`features`/`chat.panel`，default `["session.tab"]`)、`view`(静态视图文件相对路径, ≤ 32 KiB)、`micro`(本插件的 micro id)。

#### (e) `media`（v1.2，需 `capabilities` 含 `audio`）
`manifest.schema.json:86-100`：`maxItems: 2`，item 仅 `{ id }`。

#### (f) `micro`（v1.2，需 `capabilities` 含 `micro`）
`manifest.schema.json:101-115`：`maxItems: 2`，item 仅 `{ id }`。

#### (g) `overlays`（v1.2，需 `capabilities` 含 `overlay`）
`manifest.schema.json:116-149`：`maxItems: 2`，item `{ id, channel?, text? }`，`channel` 必须是本插件 micro id，`text` `maxLength: 240`；`anyOf` 要求 `channel` 或 `text` 至少一个。

### 1.3 `action` / `id` / `text` 三条公共约定

`$defs.action`（`manifest.schema.json:206-252`）：
```json
"action": {
  "type": "object",
  "required": ["id", "label"],
  "properties": {
    "id": { "$ref": "#/$defs/id" },
    "label": { "$ref": "#/$defs/text" },
    "icon": { "$ref": "#/$defs/iconName" },
    "effect": { ... staticEffect 或 1-4 个 staticEffect 数组 ... },
    "requires": { "items": { "enum": ["audio","voice","micro","overlay","files"] } }
  }
}
```

`$defs.id`（`manifest.schema.json:171-175`）：`"^[A-Za-z0-9][A-Za-z0-9_.-]{0,39}$"`，描述原文："Unique across all contributions of one plugin."

`$defs.text`（`manifest.schema.json:181-200`）原文：
> "A string, or a locale -> string map resolved by the bridge (exact, language, en, zh, first)."

解析顺序文档化为 `docs/plugins/slots.md:46`（原文）："完全匹配 → 语言部分 → `en` → `zh` → 第一个"。

### 1.4 静态 `effect` 只能是 6 种

`sdk/plugin-kit/schema/invoke.schema.json:202-224`（原文 `staticEffect`）只允许 `insert` / `command` / `open` / `link` / `toast` / `copy`。
运行时同样：`runtime/bridge/dsh-go-bridge/lib/native/effects.js:7`：
```js
export const STATIC_TYPES = new Set(["insert", "command", "open", "link", "toast", "copy"]);
```
且**有 `effect` 的条目 App 直接执行、永不 invoke**（`manifest.js:234`：`const handled = it.effect !== undefined || live.handlers.has(it.id);`；`routes.js:274`：`if (item.effect !== undefined) fail(409, "no-handler", ...)`）。

### 1.5 `wants` 的真实约束（源码里的硬编码）

`runtime/bridge/dsh-go-bridge/lib/native/routes.js:309`（原文）：
```js
const allowed = (CONTEXT_FOR[body.surface] ?? []).filter((k) => rec.parsed.wants.includes(k));
```
`CONTEXT_FOR` 的值为：`composer.actions` → `["draft","clipboard"]`，`message.actions` → `["message"]`，其余空。这与文档 `docs/plugins/slots.md:42` 一致。每个 context 字段 ≤ 16 KiB（`limits.js:19` `contextFieldBytes: 16384`；校验 `routes.js:263`）。

---

## 2. `ctx.dshGo` 的完整 API

### 2.1 拿到服务的唯一正确写法

`docs/plugins/slots.md:69`（原文）：
> **写 `['dshGo']`，不要写 `['dshGo?']`**，也不要把 `dshGo` 放进插件顶层的 `inject`：前者回调永远不执行，后者在没有桥接的主机上整个插件起不来。

源码侧对应的报错信息（`runtime/bridge/dsh-go-bridge/lib/native/core.js:353`，原文）：
```js
if (loc === undefined) throw new Error("dshGo: call ctx.dshGo.* as a method from a plugin context (ctx.inject(['dshGo'], ...))");
```

### 2.2 真实类型签名（权威：`sdk/plugin-kit/types/index.d.ts:128-141`）

**这是 `ctx.dshGo` 唯一的权威类型定义**，逐字抄录：

```ts
/** The `dshGo` service the bridge provides; only inside `ctx.inject(['dshGo'], …)`. */
export interface DshGoService {
  readonly v: 1
  readonly bridge: string
  media(id: string, handler: (scope: Scope, event: { key?: string; locale: string; signal: AbortSignal }) => { body: Uint8Array; contentType: 'audio/mpeg' | 'audio/wav' | 'audio/ogg' | 'audio/mp4' | 'audio/aac' } | Promise<{ body: Uint8Array; contentType: 'audio/mpeg' | 'audio/wav' | 'audio/ogg' | 'audio/mp4' | 'audio/aac' }>): Cancel
  micro(id: string, def: { data: DataFunction<MicroData> }): Cancel
  onCapability(handler: (scope: Scope, event: CapabilityEvent) => void | Promise<void>): Cancel
  panel<D = Record<string, unknown>>(id: string, def: PanelDefinition<D>): Cancel
  strip(id: string, data: DataFunction<StreamStripData & {}> | { data: DataFunction<StreamStripData & {}> }): Cancel
  on(id: string, handler: InvokeHandler): Cancel
  refresh(id: string, scope?: Scope): void
  /** Read a copy of selected bytes during the awaited selection callback. Throws after completion/revocation. */
  readPickedFile(handle: string): Uint8Array
}
```

**共 9 个成员：`v`、`bridge` 两个只读属性 + 7 个方法。没有 `tool` / `toolCard` / `input` / `actions` / `statusbar` 这些名字** —— 工具卡、输入、动作、状态条都不是 `ctx.dshGo` 的方法，见第 4/5/7 条。

### 2.3 逐方法对照：签名 / 参数 / 返回 / 同步异步

| 方法 | 参数 | 返回 | 同步/异步 | 运行时实现 |
|---|---|---|---|---|
| `v` | — | `1`（协议主版本） | 只读属性 | `core.js:428` `v: PROTOCOL`（`limits.js:3` `PROTOCOL = 1`） |
| `bridge` | — | 桥接版本字符串 | 只读属性 | `core.js:429` `bridge: this.version` |
| `on(id, handler)` | `id: string`（须在清单声明）、`handler: (scope, event) => HandlerResult \| Promise<HandlerResult>` | `Cancel` | handler **可同步可异步**，但桥接给 **3 秒** | `core.js:435` |
| `strip(id, data)` | `id`、`dataFn(scope, emit, caps) => Cancel \| void \| Promise<…>`；也接受 `{ data }` | `Cancel` | dataFn 可返回 Promise | `core.js:431`，`#register` 分支 `core.js:362-365` |
| `panel(id, def)` | `{ view, data?, actions? }` | `Cancel` | `view` 可为异步函数；`data` 可返回 Promise | `core.js:430`，`#register` 分支 `core.js:357-361` |
| `refresh(id, scope?)` | `id: string`、`scope?: Scope` | `void` | **同步**（抛错即同步抛出） | `core.js:436` → `#refresh` `core.js:413-422` |
| `media(id, handler)` | `id`、`handler(scope, {key, locale, signal}) => { body, contentType }` | `Cancel` | handler 可异步（**30 秒**预算） | `core.js:432` |
| `micro(id, { data })` | `id`、`{ data: dataFn }` | `Cancel` | dataFn 可返回 Promise | `core.js:433` |
| `onCapability(handler)` | `(scope, event: CapabilityEvent) => void \| Promise<void>` | `Cancel` | 可异步；**返回值永不执行** | `core.js:434`（内部 id 固定为 `"events"`） |
| `readPickedFile(handle)` | `handle: string` | `Uint8Array`（Node 里是 `Buffer`） | **同步** | `core.js:437-441` |

**关键实现细节（`core.js:424-445` 原文节选）：**
```js
#createService() {
  const core = this;
  const service = {
    ctx: this.ctx,
    v: PROTOCOL,
    bridge: this.version,
    panel(id, def) { return core.#register(this?.ctx, "panels", id, def); },
    strip(id, def) { return core.#register(this?.ctx, "strips", id, def); },
    media(id, handler) { return core.#register(this?.ctx, "media", id, handler); },
    micro(id, def) { return core.#register(this?.ctx, "micro", id, def); },
    onCapability(handler) { return core.#register(this?.ctx, "capability", "events", handler); },
    on(id, handler) { return core.#register(this?.ctx, "handlers", id, handler); },
    refresh(id, scope) { core.#refresh(this?.ctx, id, scope); },
    readPickedFile(handle) { ... },
  };
  Object.defineProperty(service, TRACKER, { value: { associate: "dshGo", property: "ctx" }, enumerable: false });
  return service;
}
```

注意 `onCapability` 的 id 是**硬编码的 `"events"`**——插件的所有能力事件都注册在这个固定槽上。

### 2.4 注册校验：没在清单里声明就只记 warn、不生效

`core.js:388-397`（原文）：
```js
const rec = this.plugin(pkg);
const declared = rec?.parsed.ids.get(id);
const ok = kind === "capability" ? (rec?.parsed.capabilities?.length ?? 0) > 0
  : kind === "media" || kind === "micro" ? declared === kind
  : kind === "handlers" ? declared !== undefined
  : kind === "panels" ? declared === "panels" : declared === "composer.strip";
if (!ok) {
  const what = kind === "capability" ? "onCapability() needs dshGo.capabilities" : `${REG_NAMES[kind]}("${id}") is not declared in dshGo.contributes`;
  this.runtimeIssue(pkg, `undeclared:${kind}:${id}`, { path: `/runtime/${id}`, code: "undeclared", severity: "warn", message: `${what}; ignored` });
}
```
`on(id)` 必须对应**任意一个插槽里声明过的 id**（`declared !== undefined`），但 `panel(id)` 必须声明在 `panels` 里，`strip(id)` 必须声明在 `composer.strip` 里。

### 2.5 回调参数的真实类型

`sdk/plugin-kit/types/index.d.ts` 原文：

```ts
/** `{ sessionId?, turnId?, messageId?, callId? }` — where the tap or the panel is. */
export type Scope = InvokeScope
```
（`index.d.ts:35-36`；`InvokeScope` 定义在 `schema.d.ts:588-593`：`sessionId?`、`turnId?`、`messageId?`、`callId?`，每个都是 string）

```ts
export interface InvokeScopeWithContext extends Scope {
  /** composer.actions, with `wants: ["draft"]`: the composer text. */
  draft?: string
  /** composer.actions, with `wants: ["clipboard"]`: the clipboard text. */
  clipboard?: string
  /** message.actions, with `wants: ["message"]`: the long-pressed reply's text. */
  message?: string
}
```
（`index.d.ts:38-46`）

```ts
export interface Caps {
  features?: Capability[]
  protocol: number
  /** App version, "x.y.z". */
  app: string
  /** BCP 47, e.g. "zh-CN". */
  locale: string
  surfaces: string[]
  components: string[]
}
```
（`index.d.ts:53-62`）

```ts
export interface InvokeEvent {
  surface: 'composer.actions' | 'message.actions' | 'composer.strip' | 'panel'
  /** Contribution id (the panel id for taps inside a panel). */
  id: string
  /** Panel taps: the view's action id (button, list row, options, confirm). */
  action?: string
  /** Panel taps: the action's args; options → { value } / { values }, confirm → { confirmed }. */
  value?: unknown
  locale: string
  signal?: AbortSignal
}
```
（`index.d.ts:66-76`）

```ts
export type Emit<D = Record<string, unknown>> = (data: D | null) => void
export type Cancel = () => void
export type DataFunction<D = Record<string, unknown>> = (
  scope: Scope,
  emit: Emit<D>,
  /** `{ protocol, locale }` of the app that subscribed: localize pushed text with it. */
  caps: Pick<Caps, 'protocol' | 'locale'>,
) => Cancel | void | Promise<Cancel | void>

export interface PanelDefinition<D = Record<string, unknown>> {
  view: View | ((scope: Scope, caps: Caps) => View | Promise<View>)
  data?: DataFunction<D>
  actions?: Record<string, InvokeHandler>
}
```
（`index.d.ts:112-126`）

```ts
export type InvokeHandler = (scope: InvokeScopeWithContext, event: InvokeEvent) => HandlerResult | Promise<HandlerResult>
```
（`index.d.ts:110`）

**注意一个易错点**：视图函数拿到的 `caps` 是**完整 5 字段**（`protocol, app, locale, surfaces, components` + 可选 `features`），但数据函数（`strip`/`panel.data`/`micro`）只拿 `{ protocol, locale }`。运行时证据 `core.js:534`：
```js
result = call(this.dataFn, { ...this.scope }, emit, { protocol: PROTOCOL, locale: this.locale });
```

### 2.6 handler 返回值的形状

`index.d.ts:90-108` 原文：
```ts
export type ImmediateResult = void | undefined | null | Effect | EffectShorthand | Array<Effect | EffectShorthand>

/** What a deferred result may finally carry: insert, copy, toast, patch only. */
export type DeferredEffect = InvokeInsert | InvokeCopy | InvokeToast | InvokePatch
export type DeferredValue =
  | void | undefined | null | DeferredEffect | Pick<EffectShorthand, 'insert' | 'copy' | 'toast' | 'patch'>
  | Array<DeferredEffect | Pick<EffectShorthand, 'insert' | 'copy' | 'toast' | 'patch'>>

export interface Deferred {
  pending: true | { label?: string; timeoutMs?: number }
  result: Promise<DeferredValue> | ((signal: AbortSignal) => DeferredValue | Promise<DeferredValue>)
}

export type HandlerResult = ImmediateResult | Deferred
```

简写对象 `EffectShorthand`（`index.d.ts:79-88`）：
```ts
export interface EffectShorthand {
  insert?: string
  toast?: string
  copy?: string
  link?: string
  open?: string
  command?: string | { name: string; args?: string }
  refresh?: true | string
  patch?: Record<string, unknown>
}
```

### 2.7 每次 invoke 的效果上限与时限（真实常量）

`runtime/bridge/dsh-go-bridge/lib/native/limits.js:23` `effectsPerInvoke: 4`
`limits.js:28` `handlerTimeoutMs: 3000`
`limits.js:31` `effectWindowMs: 3000`（App 侧执行窗口）
`limits.js:43-48` 延后效果：`pendingPerPlugin: 2`、`pendingDefaultMs: 30000`、`pendingMinMs: 1000`、`pendingMaxMs: 60000`、`pendingLabelChars: 16`、`pendingEarlyResultMs: 5000`
`limits.js:20-22` invoke 频率：`invokeBurst: 8`、`invokePerSecond: 4`、`invokeConcurrent: 4`
`limits.js:17-19`：`invokeBytes: 65536`、`invokeValueBytes: 4096`、`contextFieldBytes: 16384`

App 侧二次校验（`app/lib/plugins/native/native_model.dart:40-67` 原文）：
```dart
abstract final class NativeLimits {
  /// Composer buttons and message-menu items, per plugin.
  static const actionsPerPlugin = 2;
  /// Capsules above the composer, ui.json's included.
  static const capsulesTotal = 8;
  /// A capsule's label.
  static const capsuleLabelMax = 10;
  /// A message-menu item's label.
  static const menuLabelMax = 12;
  /// A panel's title.
  static const panelTitleMax = 16;
  /// Panels per plugin.
  static const panelsPerPlugin = 4;
  /// The status line's text.
  static const stripTextMax = 24;
  /// A toast.
  static const toastMax = 40;
  /// Effects run only this soon after the tap.
  static const tapWindow = Duration(seconds: 3);
}
```

### 2.8 `refresh` 的真实语义（源码级）

`core.js:413-422`（原文）：
```js
#refresh(callerCtx, id, partial) {
  const entry = entryOf(callerCtx?.fiber);
  const loc = entry === undefined ? undefined : this.locate(entry);
  if (loc === undefined) throw new Error("dshGo: call ctx.dshGo.refresh as a method from a plugin context");
  const scope = normalizeScope(partial ?? {});
  if (scope === undefined) throw new TypeError("dshGo.refresh: invalid scope");
  for (const feed of this.feeds.values()) {
    if (feed.pkg === loc.pkg && feed.id === id && scopeMatches(feed.scope, scope.scope)) feed.notifyStale();
  }
}
```
→ 它**只给订阅了该面板且 scope 匹配的连接发 `stale`**（`Feed.notifyStale` `core.js:638-640`），App 收到后重新 GET 视图（`app/lib/plugins/native/native_surface.dart:225-226`：`case DshGoStale(): unawaited(_read());`）。**它不发数据、不改视图、不推送。**

---

## 3. 原生视图 JSON 的完整 DSL + 预算

权威机器可读规格：`sdk/plugin-kit/schema/view.schema.json`（1192 行）。文档：`docs/plugins/components.md`、`docs/plugins/inputs.md`、`docs/plugins/styles.md`、`docs/design/native-plugins-views.md`。

### 3.1 节点形状

每个节点是 `{ "type": 组件名, ...属性 }`。根节点可多带 `title`（≤80 字，只读根节点）、`v`（1 或 2，只读根节点）、`accent`（v2 only，只读根节点）。

`view.schema.json` 的 `$defs.node`（读源码验证）里 `type` 枚举**共 42 个**：

```
stack, row, text, metric, chart, compare, steps, callout, options, confirm, list, keyValue,
progress, table, actions, image,
textField, number, stepper, toggle, slider, select, segmented, chips, date, time, rating,
form, tabs, collapsible, accordion, carousel, timeline, grid, badge, tag, avatar, codeBlock,
divider, spacer, emptyState, gallery
```

前 16 个是 **v1**（`sdk/plugin-kit/lib/view.js:115-118` `V1_COMPONENTS`），后 26 个是 **v2 专属**（`view.schema.json` 的 `x-dsh-go.version2Only`，我读到的完整数组）：

```json
"version2Only": [
  "textField","number","stepper","toggle","slider","select","segmented","chips","date","time",
  "rating","form","tabs","collapsible","accordion","carousel","timeline","grid","badge","tag",
  "avatar","codeBlock","divider","spacer","emptyState","gallery"
]
```

**v2 还要显式开**：根节点写 `v: 2`。`view.schema.json` 原文（`v` 字段的 description）：
> "Read on the root only. Omitted or 1 preserves the v1 rendering path; 2 enables bounded styles and controls."

### 3.2 每种节点的必填字段（从 schema `required` 直接读出）

| type | required | 可选 props | 上限 |
|---|---|---|---|
| `stack` | `children` | `style` | 每 stack 子节点：工具卡 16 / 面板 32 / 状态条 4 |
| `row` | `children` | `style` | 同上 |
| `text` | `text` | `tone`(仅 muted)、`style`、`spans` | 文字 2000 字 / 标签 200 字 |
| `metric` | `label`, `value` | `tone`, `caption`, `style`, `trend`(v2) | trend 2–30 点 |
| `chart` | `categories`, `series` | `kind`(bar/hbar/line/donut), `unit`, `caption` | ≤ 60 点、≤ 6 组 |
| `compare` | `items` | — | 2–4 项，每项 ≤ 8 行 |
| `steps` | `items` | — | ≤ 12 步 |
| `callout` | `text` | `tone`(info/good/warn/bad), `style` | 文字上限 |
| `options` | `id`, `options` | `multi`, `selected` | ≤ 8 项 |
| `confirm` | `id` | `confirmLabel`, `cancelLabel` | 标签 ≤ 12 字 |
| `list` | `items` | — | ≤ 50 行 |
| `keyValue` | `items` | — | ≤ 24 项 |
| `progress` | `value` | `max`, `label`, `caption`, `tone`, `shape`(bar/ring, v2) | — |
| `table` | `columns`, `rows` | — | ≤ 8 列、≤ 50 行 |
| `actions` | `items` | — | ≤ 4 个，至多 1 个 primary |
| `image` | `src` | `alt`, `aspect`(0.25–4), `caption` | data: ≤ 64 KiB；单边 ≤ 4096；总像素 ≤ 400 万 |
| `textField`/`number`/`stepper`/`toggle`/`slider`/`select`/`segmented`/`chips`/`date`/`time`/`rating` | `label` | 见 `view.schema.json` `$defs.inputV2`：`bind, value, readOnly, placeholder, required, minLength, maxLength, multiline, maxLines, min, max, step, range, multi, options, pattern, onChange, minDate, maxDate` | 每视图 ≤ 32 字段 |
| `form` | `children`(+必填 `submit`) | `label`, `submitLabel` | 不许嵌套 |
| `tabs`/`carousel` | `items` | — | ≤ 8 |
| `collapsible` | `children` | `label`, `expanded` | 单个 |
| `accordion` | `items` | `expanded` | ≤ 8 组 |
| `timeline` | `items` | — | ≤ 24 项 |
| `grid` | `children` | `columns` | 1–4 列 |
| `badge`/`tag` | `text` | `tone` | ≤ 200 字 |
| `avatar` | `src` | `alt` | 沿用 image 限制，48dp |
| `codeBlock` | `code` | `language` | ≤ 8000 UTF-16 单元 |
| `divider` | — | — | — |
| `spacer` | — | `size`: S/M/L | 8/16/24 dp |
| `emptyState` | `title` | `message`, `icon`, `action` | 标题 40 / 说明 120 / 按钮 12 字 |
| `gallery` | `items` | `columns`(2–4), `aspect`(0.5–2) | 面板 24 张、工具卡 8 张 |

### 3.3 绑定与动作

**绑定**（`view.schema.json` `$defs.binding`，原文）：
```json
{
 "description": "A value read from the view's data model. Missing values show as an em dash; arrays that are missing or empty show as 'no data yet'.",
 "type": "object",
 "required": ["$"],
 "properties": {
  "$": { "type": "string", "pattern": "^data((\\.[^.\\[\\]]+)|(\\[[0-9]+\\])){0,16}$" }
 },
 "additionalProperties": false
}
```
→ 路径必须 `data` 开头，最多 16 段。`type` **不能**绑定（`docs/design/native-plugins-views.md:97`："`type` 不能绑定（绑了按缺少 type 处理）"）。

**动作**（`$defs.action`，原文描述）：
> "Tapping sends an invoke back to the plugin that owns the view. args must stay within 1 KiB after bindings are resolved."

三种写法：`"refresh"`（动作 id 字符串）、`{ "id": "...", "args": { … } }`、或一个绑定。
`actionId` 正则（协议 §4 / `effects.js`）：`^[A-Za-z0-9][A-Za-z0-9._:-]{0,63}$`。args 求值后 ≤ 1 KiB（`view.schema.json` `limits.actionArgsBytesMax: 1024`；App 侧 `dsh_ui_plugin_model.dart:47` `actionArgsBytes = 1024`）。

**v2 有界公式**（`$defs.formula`，原文）：
```json
{
 "type": "object",
 "required": ["="],
 "properties": {
  "=": {
   "type": "string",
   "maxLength": 256,
   "description": "Bounded arithmetic: numbers, data paths, + - * / %, parentheses, abs/min/max/clamp. At most 128 tokens and depth 16. No code."
  }
 },
 "additionalProperties": false
}
```

**v2 样式**（`$defs.textStyle` / `layoutStyle`）：`size`(display/title/subtitle/body/caption/mono)、`weight`(regular/medium/semibold/bold)、`italic`/`underline`/`strike`、`align`、`maxLines`(1–20)、`color`；容器 `card`(plain/surface/tinted/outlined)、`gap`/`padding`(S=8/M=12/L=16)、`align`、`background`、`columns`(equal 或 1–8 个权重)、`dividers`。
字号真值（`x-dsh-go.v2Styles.fontPixels`）：display 28 / title 24 / subtitle 20 / body 16 / caption 13 / mono 14。

**保留字段（v2 会忽略并记诊断）**（`x-dsh-go.v2Styles.reservedStyleKeys`）：
```
position, left, right, top, bottom, width, height, radius, borderRadius, elevation,
shadow, fontFamily, fontSize, lineHeight, opacity, gradient, transform, zIndex
```
**保留 role（会让该节点失效）**：`system, approval, authorization, dialog, permission`。

### 3.4 预算限制 —— 逐条核实

**这是任务里问的 32KiB / 节点数 / 层数 / 4Hz / 3s / 连续违规降级，全部在源码里找到了。**

权威常量（`sdk/plugin-kit/schema/view.schema.json` 的 `x-dsh-go.budgets`，原文）：

```json
"budgets": {
  "toolCard": { "maxBytes": 8192, "maxNodes": 40, "maxDepth": 6, "maxChildren": 16 },
  "panel":    { "maxBytes": 32768, "maxNodes": 120, "maxDepth": 8, "maxChildren": 32 },
  "strip":    { "maxBytes": 2048, "maxNodes": 8, "maxDepth": 3, "maxChildren": 4 }
}
```

**运行时同一组数字**（`runtime/bridge/dsh-go-bridge/lib/native/limits.js:9-13`，原文）：
```js
viewBytes: 32768,
viewNodes: 120,
viewDepth: 8,
toolCardBytes: 8192,
toolCardNodes: 40,
```

**App 渲染器同一组数字**（`app/lib/ui/chat/dsh_ui/dsh_ui_plugin_model.dart:63-82`，原文）：
```dart
static const toolCard = DshUiProfile._(
  name: 'toolCard',
  plugin: true,
  bytes: 8192,
  nodes: 40,
  depth: 6,
  children: 16,
);

static const panel = DshUiProfile._(name: 'panel', plugin: true, bytes: 32768, nodes: 120, depth: 8, children: 32);

static const strip = DshUiProfile._(
  name: 'strip',
  plugin: true,
  bytes: 2048,
  nodes: 8,
  depth: 3,
  children: 4,
  text: 200,
);
```

**SDK 检查器同一组数字**（`sdk/plugin-kit/lib/view-check.js:30-33`，原文）：
```js
export const BUDGETS = Object.freeze({
  toolCard: Object.freeze({ name: 'toolCard', bytes: 8192, nodes: 40, depth: 6, children: 16, text: L.text }),
  panel: Object.freeze({ name: 'panel', bytes: 32768, nodes: 120, depth: 8, children: 32, text: L.text }),
  strip: Object.freeze({ name: 'strip', bytes: 2048, nodes: 8, depth: 3, children: 4, text: 200 }),
});
```

**"32 KiB"是面板预算，不是工具卡预算。** 工具卡是 8 KiB（因为它写进对话记录）。这一点 `docs/plugins/components.md:132-133` 与源码完全一致。

#### 其它硬上限（`x-dsh-go.limits`，原文）
```json
"limits": {
  "textMax": 2000, "labelMax": 200, "titleMax": 80,
  "chartPointsMax": 60, "chartSeriesMax": 6,
  "compareItemsMax": 4, "compareRowsMax": 8, "stepsMax": 12, "optionsMax": 8,
  "confirmLabelMax": 12, "listItemsMax": 50, "keyValueItemsMax": 24,
  "tableColumnsMax": 8, "tableRowsMax": 50,
  "actionsMax": 4, "primaryActionsMax": 1, "actionArgsBytesMax": 1024,
  "bindingSegmentsMax": 16, "imageDataBytesMax": 65536,
  "imageSideMax": 4096, "imagePixelsMax": 4000000
}
```

#### 原始 JSON 嵌套 64 层
`x-dsh-go` 里 `"maxJsonNesting": 64`；App 侧 `DshUiProfile.jsonNesting = 64`（`dsh_ui_plugin_model.dart:44`）。
注意这跟"组件层数"（6/8/3）是**两个不同的限制**：JSON 嵌套算上数组与对象，组件层数只算带 `type` 的节点（`docs/design/native-plugins-protocol.md:326`："节点 = 带字符串 `type` 字段的 JSON 对象，层数 = 这种对象的嵌套深度"）。

### 3.5 超限后的行为（区分"整块降级" vs "截断"）

`docs/design/native-plugins-views.md:158`（原文）：
> 视图大小、JSON 嵌套超限：整块降级（§6）。其余超限一律截断，在被截的组件下方标一行小字（「已截断：还有 3 项没有显示」）。

运行时：桥接**只卡字节**（32 KiB），节点数与层数由 App 截断并标注，桥接把数出来的值放 `stats` 另记一条 `too-many`（warn）。证据 `routes.js:198-212`（原文）：
```js
if (json.bytes > L.viewBytes) {
  core.violation(pkg, "invalid-view", `view(${id}) is ${json.bytes} bytes (max ${L.viewBytes})`);
  fail(500, "invalid-view", `view(${id}) exceeds ${L.viewBytes} bytes`);
}
...
if (stats.nodes > L.viewNodes || stats.depth > L.viewDepth) {
  issues.push({ path: `/runtime/${id}`, code: "too-many", severity: "warn", message: `view has ${stats.nodes} nodes / depth ${stats.depth} (max ${L.viewNodes} / ${L.viewDepth}); the app truncates` });
}
```

### 3.6 「4 Hz」的出处与真实语义

`limits.js:16`：`dataHz: 4`
`core.js:567-575`（原文）—— 这才是实现：
```js
/** Queue a new full value; flushed at most `dataHz` times a second. */
push(value) {
  if (this.stopped) return;
  this.pending = { value };
  if (this.timer !== undefined) return;
  const wait = Math.max(0, this.lastFlush + 1000 / LIMITS.dataHz - Date.now());
  this.timer = setTimeout(() => this.#flush(), wait);
  this.timer.unref?.();
}
```
→ **是"每个数据源最多每 250 ms 发一次最新一份"**，不是丢数据；同值不发（`core.js:584` `if (prev !== undefined && deepEqual(prev, next)) return;`）。
**注意 `micro` 通道不是 4 Hz，而是 10 Hz**（`capabilities.js:22` `microMs: 100`）。两者常被混为一谈。

### 3.7 「3 秒」的三个不同含义（容易搞错）

| 3 秒 | 是什么 | 源码 |
|---|---|---|
| `handlerTimeoutMs: 3000` | 插件 handler（`on`/`panel.actions`）必须在这之内返回，否则 504 + 计违规 | `limits.js:28`；`routes.js:318-324` |
| `viewTimeoutMs: 3000` | 视图函数必须在这之内返回，否则 504 + 计违规 | `limits.js:29`；`routes.js:186-192` |
| `effectWindowMs: 3000` + `NativeLimits.tapWindow` | **App 侧**：从用户点按算起 3 秒内收到的效果才执行 | `limits.js:31`；`native_model.dart:65-66` |

另有 `firstDataWaitMs: 1000`（GET 视图时等首份数据的时长，`limits.js:30`；`routes.js:221` `await feed.waitFirst(L.firstDataWaitMs);`）。

### 3.8 连续违规降级 = **5 次**

`limits.js:42`：`violationsToDegrade: 5`
实现（`core.js:295-309`，原文节选）：
```js
violation(pkg, code, message) {
  const h = this.#health(pkg);
  h.consecutive++;
  h.total++;
  ...
  if (h.consecutive >= LIMITS.violationsToDegrade && !h.degraded) {
    h.degraded = true;
    this.capabilityChanged?.();
    h.issues.set("degraded", { path: "/runtime/degraded", code: "degraded", severity: "error", message: `${h.consecutive} consecutive violations; native UI off until the plugin reloads` });
    for (const feed of [...this.feeds.values()]) if (feed.pkg === pkg) feed.stop("degraded");
  }
  ...
}
```
`success(pkg)`（`core.js:311-314`）把 `consecutive` 清零 → **中间任一成功即清零**，与文档 `docs/plugins/hosts.md:49`（"连续 5 次违规，本次运行降级成网页版"）一致。

**哪些算违规、哪些不算**（`docs/design/native-plugins-protocol.md:487-489`，与 `effects.js:117-139` 的 `violation` 标记实现一致）：
- 算：视图函数抛错/超时/超 32 KiB/非对象；数据函数抛错；`emit` 非对象/不可序列化/超 32 KiB；handler 抛错/超时/返回结构不对/超上限效果；`pending` 形状不对；延后结果含非法效果。
- 不算：文字超长截断、条目超数截断、**未知效果类型**、App 侧 429、插件主动返回空效果、延后结果过期或出错、`no-stream`、`too-many-pending`。

### 3.9 出错时的显示分档

`docs/plugins/components.md:103-111`（原文）：
| 情况 | 显示 |
|---|---|
| 子组件写错（不认识的 `type`、缺必填属性） | 这一个换成灰色占位「这部分无法显示」，其余照画 |
| 根节点坏了、视图超大小、嵌套太深 | 整块一行「插件界面有误」 |
| 需要数组的组件拿到空数组或绑定取不到 | 一行「暂无数据」，不算错 |
| 绑定的文字取不到 | 「—」 |
| 图片地址不合规或解不开 | 浅底占位「图片无法显示」 |

---

## 4. 工具结果卡片（toolCards）

### 4.1 声明方式：**不写在 `dshGo` 里**，写在工具的 `output.presentationMeta` 里

`docs/plugins/quickstart.md:26-46` 的接法，加上 SDK 的真实实现 `sdk/plugin-kit/lib/card.js`。

关键常量（`card.js:5-8`，原文）：
```js
/** The key the dsh-go app reads from `tool/result.meta`. */
export const VIEW_KEY = 'dsh-go/view'

/** A tool card is written into the conversation log, so it must stay small. */
export const CARD_MAX_BYTES = 8 * 1024
```

`toolCard()` 的完整实现（`card.js:48-72`，原文）：
```js
export function toolCard(build, options = {}) {
  const maxBytes = options.maxBytes ?? CARD_MAX_BYTES
  return (args, value) => {
    let extra = {}
    try {
      if (options.meta) extra = toLosslessJson(options.meta(args, value)) ?? {}
      if (typeof extra !== 'object' || extra === null || Array.isArray(extra)) extra = {}
    } catch (error) {
      options.onError?.(error)
      extra = {}
    }
    try {
      const view = toLosslessJson(build(args, value))
      if (view === undefined || view === null) return extra
      if (jsonBytes(view) > maxBytes) {
        options.onError?.(new RangeError(`tool card is ${jsonBytes(view)} bytes, over ${maxBytes}`))
        return extra
      }
      return { ...extra, [VIEW_KEY]: view }
    } catch (error) {
      options.onError?.(error)
      return extra
    }
  }
}
```

它兜住三件**会让整次工具调用失败**的事（`docs/plugins/quickstart.md:36-41`，与 `card.js:18-26` `toLosslessJson` 实现一致）：
| DSH 的规则 | `toolCard()` 怎么处理 |
|---|---|
| `presentationMeta` 抛错 → 工具调用失败 | 捕获，返回 `{}`（不出卡片） |
| 返回值不是无损 JSON（`undefined`/`NaN`/`Infinity`/`-0`/`Date`/`Map`）→ 工具调用失败 | 过一遍 JSON |
| 视图超过 8 KiB | 返回 `{}` |

`toLosslessJson`（`card.js:18-26`，原文）：
```js
export function toLosslessJson(value) {
  let text
  try {
    text = JSON.stringify(value)
  } catch {
    return undefined
  }
  return text === undefined ? undefined : JSON.parse(text)
}
```

### 4.2 完整链路（源码级）

`docs/design/native-plugins-views.md:280-301` 给出了逐跳位置（标注"已查源码，DSH 0.2.1-rc.2"）：
```
defineTool output.presentationMeta(args, value)
  → ToolRegistry.createSuccessResult：成功后调用，快照成无损 JSON，作为 result.meta
  → appendToolResult：session.append("tool/result", { turn, step, message, error?, meta })
  → 会话日志（jsonl，持久化；重进对话、重放、导出都带着）
  → App：SessionLogEvent.fromJson → ToolResultEvent.meta = data["meta"]
  → ChatTranscript：ToolChatItem.meta = result.meta
  → ToolView.pluginView = parsePluginView(meta["dsh-go/view"], budget: toolCard)
```

**App 侧读取（我在源码里找到了，这是本报告里"已读源码验证"最硬的一条）** `app/lib/ui/chat/tool_view.dart:13-21`（原文）：
```dart
/// 插件工具的原生卡片(结果 meta 里的 `dsh-go/view`,见
/// `docs/design/native-plugins-views.md` §8):null = 没有;
/// `DshUiInvalid` = 有但画不了。只在成功的调用上读;解析按 meta 对象缓存。
DshUiParse? toolCardParse(ToolChatItem item) {
  if (item.status != ToolStatus.succeeded) return null;
  final meta = item.meta;
  if (meta is! Map || !meta.containsKey(toolCardViewKey)) return null;
  return parsePluginView(meta[toolCardViewKey], budget: PluginViewBudget.toolCard);
}
```
`toolCardViewKey` 的真值（`app/lib/ui/chat/dsh_ui/dsh_ui_plugin.dart:11`）：`const toolCardViewKey = 'dsh-go/view';`

### 4.3 渲染成什么样

`docs/design/native-plugins-views.md:305-313`（原文）：
| 位置 | 有 `dsh-go/view` 且能画 | 有但画不了 | 没有 |
|---|---|---|---|
| 工具行 | **默认展开**：原生卡片；参数和原始结果收在卡片下的「详情」里，默认收起。行收起时摘要换成视图 `title` | 默认收起；展开是一行「插件界面有误」 | 照旧 |
| 工具组 | 带卡片的调用不进组：它把连续的调用切开 | 照旧 | 照旧 |
| 一轮结束、折叠后 | 工具行收进「已完成 · N 次工具调用」，**卡片本身留在消息流里** | 照旧折叠 | 照旧折叠 |

**工具卡不能点**（`docs/plugins/components.md:97` 原文）：
> 工具卡：工具行展开区里的卡片；折叠时摘要是视图的 `title`。**第 0 期不能点**：按钮和确认不显示，选项只读。

（`docs/design/native-plugins-views.md:134` 给出完整清单："`actions`、`confirm` 不显示；`options` 显示成只读清单（`selected` 打勾）；`list` 行不带箭头"。）

**工具卡没有数据模型**：`docs/plugins/components.md:125`（"工具卡没有数据"）；`docs/plugins/versions.md:76`（"工具卡没有 `caps`（它写进记录，以后的 App 也要能画），所以工具卡只用最稳的组件：`stack` `row` `text` `metric` `chart` `compare` `steps` `callout`"）。

**工具卡不能用 `asset:` 图片**：`docs/design/native-plugins-views.md:278`（原文）："图片只能用 `data:`：对话记录里只有工具名，App 不知道它属于哪个插件包，`asset:` 取不到。"

### 4.4 「失败的工具调用不出卡片」是真的

上面 `toolCardParse` 的第一行就是 `if (item.status != ToolStatus.succeeded) return null;`，源码级确证。文档侧同：`docs/plugins/quickstart.md:83`（"失败的工具调用不出卡片"）。

### 4.5 版本兼容承诺

`docs/plugins/versions.md:80-83`（原文）：
> 工具卡写进对话记录，一年后打开老对话也要能画。App 对 `meta["dsh-go/view"]` **只增不删**，老格式永久能画。

协议 §8 同（`docs/design/native-plugins-protocol.md:529`）："工具卡（`meta["dsh-go/view"]`）| 写进对话记录，只增不删字段，App 永久兼容老格式"。

### 4.6 主机覆盖（这是工具卡的独特优势）

`docs/plugins/hosts.md:5-9`（原文）：

| | 工具卡 | 胶囊 · 菜单 · 状态条 · 面板 | 网页版插件（网页岛） |
|---|---|---|---|
| 本机引擎 | ✅ | ✅ | ✅ |
| SSH 主机 | ✅ | 第二期 | ✅ |
| 中转主机 | ✅（随对话记录走） | ❌ | ❌ |

→ **工具卡是唯一"三种主机都可用"的原生能力**，因为它走对话记录、不走桥接。

---

## 5. 输入框上方胶囊 / `composer.strip`

任务里把这两件事写成一件事，**源码里它们是两个完全不同的插槽**，必须分开。

### 5.1 `composer.actions` —— 胶囊（按钮）

**清单**（`docs/plugins/slots.md:24-28` 示例；schema `manifest.schema.json:46-58`）：
- 每插件 ≤ 2（`limits.js:7`）
- `label` ≤ 10 字（`limits.js:8` `composerAction: 10`）
- 字段：`id`、`label`（必填）、`icon?`、`effect?`、`requires?`

**两种点按行为**（这是关键设计）：
1. 有静态 `effect` → **App 直接执行，永不 invoke**（`manifest.js:234`、`routes.js:274`）
2. 没有 `effect` → 必须注册 `ctx.dshGo.on(id, handler)`，点按时 invoke

App 侧真实上限（`app/lib/plugins/native/native_model.dart:42-52`）：
```dart
/// Composer buttons and message-menu items, per plugin.
static const actionsPerPlugin = 2;

/// Capsules above the composer, ui.json's included.
static const capsulesTotal = 8;

/// A capsule's label.
static const capsuleLabelMax = 10;
```

**「8 个」是全 App 总数**，含 ui.json 的 `quickActions`。`app/lib/ui/plugins/native/composer_capsules.dart:1-8`（原文）：
```dart
// The capsule row above the composer — ui.json's `composer.quickActions`
// (docs/design/plugin-seams.md §10.2) and the plugins' `composer.actions`
// (docs/design/native-plugins.md §3.3) in one row. ui.json's come first and in
// its order; plugins add up to two each, up to eight in all. A tap puts text in
// the composer or does what the capsule says; nothing is ever sent by it. The
// row steps aside while a reply is generating (the activity strip takes the
// place). Last, outside the eight, 「插件」 opens the conversation's plugin
// sheet (chat_plugins_sheet.dart) while it has something.
```
截断实现（`composer_capsules.dart:215-228` 附近，读源码）：
```dart
for (final a in plugins) {
  if (items.length >= NativeLimits.capsulesTotal) break;
  ...
}
```

### 5.2 `composer.strip` —— 状态条（一行文字）

**清单**（`manifest.schema.json:72-78` + `$defs.strip:253-274`）：
- 每插件 **≤ 1**
- 字段：`id`（必填）、`icon?`、`text?`（静态兜底，≤ 24 字）、`panel?`（点按打开本插件哪个面板）

**注册方式**（`ctx.dshGo.strip`，两种等价写法，`core.js:362-365` 原文）：
```js
} else if (kind === "strips" || kind === "micro") {
  const data = typeof def === "function" ? def : def?.data;
  if (typeof data !== "function") throw new TypeError(`dshGo.${kind === "micro" ? "micro" : "strip"}: pass (scope, emit) => cancel or { data }`);
  value = { data };
}
```

**数据 shape —— 这是任务问的"数据 shape"，源码权威在 `core.js:485-496`（原文）**：
```js
/** Strip data -> {text, icon?, tone?} | null, or undefined when malformed. */
function sanitizeStrip(d) {
  if (d === null || d === undefined) return null;
  if (!isPlainObject(d)) return undefined;
  if (d.text !== undefined && d.text !== null && typeof d.text !== "string") return undefined;
  const out = {};
  const text = typeof d.text === "string" ? cleanLine(d.text) : "";
  out.text = text === "" ? null : chars(text) > LIMITS.textChars.stripText ? truncateChars(text, LIMITS.textChars.stripText) : text;
  if (typeof d.icon === "string" && ICON.test(d.icon)) out.icon = d.icon;
  if (TONES.includes(d.tone)) out.tone = d.tone;
  return out;
}
```
→ **只有三个字段**：`text`（≤ 24 字，空/null = 这条暂不显示）、`icon`（IconName）、`tone`（`good|warn|bad|info|muted`，真值见 `limits.js:67` `TONES = ["good","warn","bad","info","muted"]`）。**其它字段直接丢弃。**

类型定义（`sdk/plugin-kit/types/schema.d.ts:1072-1077` 原文）：
```ts
/** Data model of a composer.strip subscription (§4.5). */
export type StreamStripData = {
  text?: string | null
  icon?: string
  tone?: "good" | "warn" | "bad" | "info" | "muted"
} | null
```

**点按行为**：有 `panel` → 打开那个面板；否则若有 `on(id)` → invoke；都没有 → 不可点。证据 `manifest.js:249-252`（原文）：
```js
const liveStrip = live.strips.has(it.id);
const tappable = it.panel !== undefined || live.handlers.has(it.id);
const ready = liveStrip || tappable || it.text !== undefined;
```
App 侧 `app/lib/ui/plugins/native/composer_strip.dart:230-247`（原文）：
```dart
void _tap(NativeStrip strip) {
  final plugin = ref.read(nativeContributionsProvider).plugin(strip.package);
  final panel = plugin?.panels.where((p) => p.id == strip.panel).firstOrNull;
  if (panel != null) {
    unawaited(Navigator.of(context).push(NativePanelPage.route(panel: panel, sessionId: widget.sessionId)));
  } else if (strip.tappable) {
    unawaited(runNativeTap(..., surface: NativeSurface.composerStrip, id: strip.id, ...));
  }
}
```

**多插件轮流**：同时只显示 1 条，每 6 秒换下一个（`composer_strip.dart:42-46`，原文）：
```dart
/// How long each plugin's line stays before the next one's.
static const turn = Duration(seconds: 6);

/// Lines subscribed at once (the rest wait their turn).
static const live = 4;
```

**状态条 v1 不走组件树**（`docs/design/native-plugins-protocol.md:431` 原文）：
> 订阅方式与面板相同（`sub {pkg, id: <strip id>, scope: {sessionId}}`），**没有视图，不需要 GET**：App 直接画 `icon` + `text`（v1 状态条不走组件树，视图 schema 的 `strip` 预算档留给以后）。

→ **所以"statusbar 的视图 JSON"在 v1 里根本不会被读取**；`strip` 那档 2 KiB/8 节点/3 层的预算目前是**预留档**。

---

## 6. 消息菜单 `message.actions`

**清单**（`manifest.schema.json:59-71`）：每插件 ≤ 2，`label` ≤ 12 字。字段与胶囊完全相同（`id`/`label`/`icon?`/`effect?`/`requires?`）。

**位置**：助手消息**长按菜单**，**按插件分组，组名是插件 `title`**（`docs/plugins/slots.md:8`；`docs/plugins/design-rules.md:26`）。

**App 侧真实实现 `app/lib/ui/plugins/native/message_plugin_menu.dart:1-49`（原文，完整）：**
```dart
// The plugins' items in a reply's long-press menu
// (docs/design/native-plugins.md §3.3 `message.actions`): one group per plugin,
// under its name, after the menu's own rows. A tap goes to the plugin with the
// reply's id; what comes back runs within 3 s.

List<MenuGroup> nativeMessageMenuGroups(BuildContext context, AssistantChatItem item, {String? text}) {
  final scope = NativeComposerScope.of(context);
  final sessionId = scope?.host.sessionId;
  if (scope == null || sessionId == null) return const [];
  final ref = scope.ref;
  final plugins = ref.read(nativeMessageActionsProvider);
  if (plugins.isEmpty) return const [];
  final messageId = ref.read(chatProvider(sessionId).notifier).replyMessageId(item);
  if (messageId == null) return const [];
  final target = DshGoScope(sessionId: sessionId, messageId: messageId);
  return [
    for (final plugin in plugins)
      MenuGroup(
        title: ref.read(nativePluginTitleProvider(plugin.package)),
        items: [
          for (final action in plugin.messageActions)
            MenuItem(
              icon: action.icon ?? AppIcons.plugin,
              label: action.label,
              onTap: () {
                final host = scope.host.context;
                if (host == null || !host.mounted) return;
                unawaited(runNativeAction(host, ref, action, scope: target, message: text, composer: scope.host));
              },
            ),
        ],
      ),
  ];
}
```
→ **scope 带 `sessionId` + `messageId`**（这是唯一会自动带上 `messageId` 的插槽）。插件可用 `wants: ["message"]` 直接拿到正文，不用自己按 id 查（`docs/design/native-plugins-protocol.md:123`："插件在宿主上本来就能读对话（与网页插件同权），`message` 只是省掉插件按 id 查消息的麻烦，不新增暴露"）。

**永远拿不到 `draft`/`clipboard`**（`routes.js:309` 的 `CONTEXT_FOR` 限制）。

**调用点**：`app/lib/ui/chat/widgets/assistant_message.dart:155-159`（`showMessageMenu(..., extra: nativeMessageMenuGroups(context, widget.item, text: markdown))`）。注意 `user_bubble.dart:63` 也调 `showMessageMenu` 但**没有 `extra`** → 插件菜单项**只在助手消息上出现**，用户消息上没有。

---

## 7. 状态条 `statusbar`

**源码里没有 `statusbar` 这个插槽名。**"状态条"就是 `composer.strip`（第 5.2 节）。我全仓库 grep `statusbar` 只命中 Android 的 `window.statusBarColor`、`TerminalStatusBar`、`ShareStatusBar` 等 App 内部 UI，**没有一个是插件插槽**。

唯一与"状态条"相关但**不是插件能力**的东西：`app/lib/ui/plugins/native/composer_strip.dart:1-5` 提到"生成回复时让位给活动条（activity strip）"——那是 App 自己的"正在生成"指示条，插件无法写。

因此第 7 条答案 = **第 5.2 节**。不重复。

---

## 8. 面板 pane/panel：打开、scope、actions 回传、refresh

### 8.1 声明 + 注册

清单（`manifest.schema.json:275-315`）：`{ id, title(≤16字), icon?, placement?, view?, micro? }`，每插件 ≤ 4。
placement 三个值（默认 `["session.tab"]`）：
| placement | 位置 | scope | 源码 |
|---|---|---|---|
| `session.tab` | 对话详情页「插件」页签 | **带 `sessionId`** | `manifest.schema.json:296-305`；`docs/plugins/slots.md:104` |
| `features` | 全部功能 | **空** | `docs/plugins/slots.md:105` |
| `chat.panel` | 聊天页「插件」弹层 | **带 `sessionId`** | `docs/plugins/slots.md:106` |

视图来源优先级：`panel(id).view` > 清单静态 `view` 文件（`routes.js:182-207`；`docs/design/native-plugins-protocol.md:324`"视图来源优先级：`panel(id).view` > 清单静态 `view` 文件。都没有 → 409 `no-handler`"）。静态 `view` 文件必须 ≤ 32 KiB（`manifest.js:182` `else if (st.size > LIMITS.viewBytes) issues.push(...)`）。

### 8.2 打开流程（App → 桥接，源码级）

`app/lib/plugins/native/native_surface.dart:174-196`（原文，`_read`）：
```dart
/// Reads the panel's view (and initial data); then follows the stream.
Future<void> _read() async {
  if (!ref.mounted) return;
  final client = ref.read(nativeBridgeProvider);
  if (client == null) return;
  final read = ++_reads;
  try {
    final view = await client.view(key.package, key.id, scope: key.scope, caps: ref.read(nativeCapsProvider));
    if (!ref.mounted || read != _reads) return;
    _diagnostics.view(key.package, view.bytes);
    _diagnostics.report(key.package, '', [for (final i in view.issues) NativeIssue.fromBridge(i)]);
    _rev = view.rev;
    _set(NativeSurfaceState(view: view.view, data: view.data, title: view.title, loading: false));
    final stream = ref.read(nativeStreamProvider);
    if ((view.live || key.live) && stream != null && _sub == null) _follow(stream, data: view.data);
  } on DshException catch (e) { ... }
}
```

HTTP 请求（`packages/dsh_client/lib/src/dsh_go_api.dart:668-683` 原文）：
```dart
Future<DshGoView> readDshGoView(
  String pkg,
  String id, {
  DshGoScope scope = DshGoScope.none,
  DshGoCaps caps = const DshGoCaps(),
  Duration timeout = const Duration(seconds: 8),
}) async {
  final path = dshGoViewPath(pkg, id);
  final response = await _dshGoConnection(
    path,
  ).get(path, query: {...caps.query, ...scope.toJson()}, timeout: timeout);
  ...
}
```

### 8.3 `scope` 里到底有什么

**四个字段，全部可选**（`sdk/plugin-kit/schema/invoke.schema.json:23-39` `$defs.scope`，原文）：
```json
"scope": {
  "type": "object",
  "properties": {
    "sessionId": { "$ref": "#/$defs/scopeValue" },
    "turnId":    { "$ref": "#/$defs/scopeValue" },
    "messageId": { "$ref": "#/$defs/scopeValue" },
    "callId":    { "$ref": "#/$defs/scopeValue" }
  }
}
```
每个值（`$defs.scopeValue`，原文）：`minLength: 1`、`maxLength: 256`、`"^[^\\u0000-\\u001f\\u007f]+$"`（不含控制字符）。

**scope 规范化**（`docs/design/native-plugins-protocol.md:200`，与 `core.js` `normalizeScope` 一致）：
> 去掉空值后按键名排序；两个 scope 相等当且仅当规范化后相等。订阅按 `{pkg, id, 规范化 scope}` 共享同一份数据源

**各插槽实际带什么**（`docs/plugins/slots.md:82` + `invoke.schema.json`）：
| 插槽 | scope |
|---|---|
| `composer.actions` 点按 | `sessionId`（有当前对话时） |
| `message.actions` 点按 | `sessionId` + `messageId` |
| `composer.strip` 点按/订阅 | `sessionId` |
| 面板（`session.tab` / `chat.panel`） | `sessionId` |
| 面板（`features`） | 空对象 |

→ **插件判断"有没有具体对话"只需看 `scope.sessionId` 有没有值**（`docs/plugins/slots.md:108`："视图函数按 `scope.sessionId` 有没有来区分（`session.tab` 和 `chat.panel` 都带）"）。

### 8.4 actions 怎么回传

**统一走一次 invoke**：`surface: "panel"`，`id` = 面板 id，`action` = 视图里动作的 id，`value` = 动作的 args。

路由实现（`routes.js:276-280`，原文）：
```js
const panel = isPanel ? core.registration(body.pkg, "panels", body.id) : undefined;
const handler = core.registration(body.pkg, "handlers", body.id);
let kind = "panels", registration = panel;
let fn = isPanel && body.action !== undefined ? panel?.def.actions?.[body.action] : undefined;
if (fn == null) { kind = "handlers"; registration = handler; fn = handler?.def; }
```
→ **先找 `panel.actions[action]`，找不到再回落到 `on(面板id)`**。这正是 `docs/plugins/actions.md:46` 说的"没有 `actions` 表时落到 `on(面板 id)`，用 `event.action` 区分"。

**四种可点元素与 `event.value` 的形状**（`docs/plugins/actions.md:48-53`，与 `view.schema.json` 一致）：

| 元素 | 视图里写 | handler 收到 |
|---|---|---|
| 按钮 | `actions` 组件：`{ label, action: 'refresh' }` | `action: 'refresh'`，`value: {}` |
| 列表行 | `list` 行：`{ title, action: { id: 'open', args: { row: 3 } } }` | `action: 'open'`，`value: { row: 3 }` |
| 单选/多选 | `options`：`{ id: 'range', options: [...] }` | `action: 'range'`，`value: { value }` / `{ values }` |
| 确认 | `confirm`：`{ id: 'clear' }` | `action: 'clear'`，`value: { confirmed: true \| false }` |

`value` 的 JSON 上限：**4 KiB**（`limits.js:18` `invokeValueBytes: 4096`；`routes.js:256` 校验）。

**v2 表单提交的形状不同**（`docs/plugins/inputs.md:72-77`）：
| 触发 | handler 的 `event.value` |
|---|---|
| `onChange: 'changed'` | `{ bind: 'data.form.name', value: '新值' }` |
| `form.submit: 'save'` | `{ values: { 'data.form.name': '新值', 'data.form.count': 2 } }` |
完整 `event.value` 仍限 **4 KiB UTF-8 JSON**。

### 8.5 `refresh` 怎么用

三种用法：
1. **插件主动**：`ctx.dshGo.refresh(id, scope?)` → 只发 `stale`（§2.8）
2. **handler 返回**：`{ refresh: true }`（缺省 = 发起点按的面板）或 `{ refresh: 'panelId' }` → App 重新 GET 视图（`effects.js:96-100`）
3. **App 内建**：面板加载失败时给「重试」按钮（`native_panel.dart:89-92: `DshUiControlButton(label: l10n.commonRetry, onPressed: () => ref.read(nativeSurfaceProvider(key).notifier).refresh())`）

App 侧 `refresh()` 实现（`native_surface.dart:242-258` 原文）：
```dart
/// A `refresh` effect, or 重试: read the view again (a panel), or follow
/// the stream again (a strip).
void refresh() {
  if (!ref.mounted) return;
  if (ref.read(nativeBridgeProvider) == null) { _set(state.copyWith(loading: false, offline: true)); return; }
  _set(state.copyWith(loading: !state.hasView, offline: false, clearFailure: true));
  if (_panel) { unawaited(_read()); return; }
  final stream = ref.read(nativeStreamProvider);
  if (stream == null) return;
  _stop();
  _follow(stream);
  stream.foreground();
}
```

**`refresh` 不能用于延后结果**（`effects.js:10`：`DEFERRED_TYPES = new Set(["insert", "copy", "toast", "patch"])`）；**不能出现在清单静态 effect 里**（`STATIC_TYPES` 不含 `refresh`）。

### 8.6 `patch` 与数据流的关系

`patch` 不只是 App 本地合并——桥接**同时并入数据源**，其他订阅者会收到（`core.js:467-476` 原文）：
```js
/** Merges a `patch` effect into a running feed (other subscribers see it through the stream). */
patchFeed(pkg, id, scope, data) {
  for (const feed of this.feeds.values()) {
    if (feed.pkg !== pkg || feed.id !== id || feed.dataFn === undefined || feed.data === undefined) continue;
    if (JSON.stringify(feed.scope) !== scope.key) continue;
    // Merge over the newest value, including an emit still waiting for its 4 Hz slot.
    const current = feed.pending !== undefined ? feed.pending.value : feed.data;
    feed.push(mergeApply(current ?? {}, data));
  }
}
```
调用点 `routes.js:342`：`for (const e of norm.effects) if (e.type === "patch") core.patchFeed(body.pkg, e.id, scope, e.data);`

**注意**：`patchFeed` 要求 `feed.dataFn !== undefined`，即**只有注册了 `data` 函数的面板才能收到 patch**。纯静态视图面板的 `patch` 只影响 App 本地。

### 8.7 面板的署名与「插件内容」边界

`docs/plugins/design-rules.md:25`（原文）：
> 面板 | 外框顶部一行：插件图标 + 插件名

App 侧 `native_panel.dart:113-121` `PluginFrame(pluginName: title, icon: icon, child: ...)`。v2 面板与工具卡还带不能被作者样式覆盖的固定「插件内容」标记（`docs/plugins/styles.md:111`）。

---

## 9. 语音能力（最高优先级）

### 9.1 全节结论速查

| 问题 | 答案 | 证据等级 |
|---|---|---|
| 插件能不能调 TTS？ | **能，但不是"调 TTS 接口"，而是"在主机上自己生成音频字节"。** 插件在 `ctx.dshGo.media(id, handler)` 里返回 `{ body, contentType }` | 〔已读源码验证〕 |
| 接口叫什么？ | `ctx.dshGo.media(id, handler)` + 效果 `{ type:'audio', op:'play', instance, src:'media:<id>', key? }` | 〔已读源码验证〕 |
| 本地还是在线引擎？ | **插件自己决定。** dsh-go 只搬运字节。**仓库内没有任何内置 TTS 引擎** | 〔已读源码验证〕 |
| 能不能调 ASR？ | **插件不能。** 原生插件只有"检测你有没有开口"（`voice`），**不给你任何声音或文字** | 〔已读源码验证〕 |
| 流式吗？ | **不流式。** `media` 只回完整一段 | 〔已读源码验证〕 |
| 能不能播音频？ | 能（≤ 16 MiB，5 种 MIME） | 〔已读源码验证〕 |
| 能不能被用户打断（开口即停）？ | **能，而且是系统级保证**：App 先在本地停掉播放，**再**通知插件 | 〔已读源码验证〕 |
| 字幕（含悬浮字幕）支持吗？ | 支持，但**"悬浮"仅限 App 内**，不覆盖其他 App | 〔已读源码验证〕 |
| 后台/前台限制？ | **只在前台**。退后台 / 锁屏 / 换对话 / 换主机 / 断流 → 立刻本地停 | 〔已读源码验证〕 |
| **有没有"让 agent 主动开一轮对话 / 定时唤醒 / 通知后唤醒"的官方能力？** | **有，而且它是本仓库里最完整的子系统。但它不是插件能力**，是 App 自己的 proactive kernel + 一个独立主机插件 `dsh-go-proactive`。详见 §9.10 | 〔已读源码验证〕 |

### 9.2 `docs/plugins/voice.md` 全文要点（逐条对齐代码）

`docs/plugins/voice.md:7-12` 的四能力表（原文）：

| 能力 | 你得到 | 你做 | App 做 |
|---|---|---|---|
| `audio` | 一段声音：播、暂停、继续、停，播完有事件 | 在主机上生成音频字节（比如问主机上的 TTS） | 从桥接取回、在手机上播 |
| `voice` | 用户开口 / 说完的事件 | 什么都不用做 | 在手机上听；一开口**先自己停掉声音**，再告诉你 |
| `micro` | 一条小数据通道，最快每秒 10 次 | 推 `{ text, progress }` | 画成一行字幕 |
| `overlay` | 一行悬浮字幕 | 指定它跟哪条 `micro` 通道 | 浮在对话顶栏下，带关闭键 |

**这四条我全部在源码里找到了对应实现。**

#### (1) `audio` —— 插件做字节，App 播

协议（`sdk/plugin-kit/types/index.d.ts:132` 原文）：

```ts
media(id: string, handler: (scope: Scope, event: { key?: string; locale: string; signal: AbortSignal }) => { body: Uint8Array; contentType: 'audio/mpeg' | 'audio/wav' | 'audio/ogg' | 'audio/mp4' | 'audio/aac' } | Promise<{ body: Uint8Array; contentType: 'audio/mpeg' | 'audio/wav' | 'audio/ogg' | 'audio/mp4' | 'audio/aac' }>): Cancel
```

**MIME 白名单**（`runtime/bridge/dsh-go-bridge/lib/native/capabilities.js:24` 原文）：

```js
export const AUDIO_TYPES = new Set(["audio/mpeg", "audio/wav", "audio/ogg", "audio/mp4", "audio/aac"]);
```

App 侧同一集合（`packages/dsh_client/lib/src/dsh_go_api.dart:915-918`）：

```dart
const dshGoAudioTypes = {'audio/mpeg', 'audio/wav', 'audio/ogg', 'audio/mp4', 'audio/aac'};
const dshGoMediaMaxBytes = 16 * 1024 * 1024;
```

**上限**（`capabilities.js:19-23` 原文）：

```js
export const CAP_LIMITS = Object.freeze({
  mediaBytes: 16 * 1024 * 1024, mediaMs: 30000, mediaConcurrent: 2,
  leases: 8, pluginLeases: 2, eventBurst: 4, eventsPerSecond: 10,
  microBytes: 1024, microMs: 100, microSubs: 8, pluginMicroSubs: 2, textChars: 240, perKind: 2,
});
```

→ 单段音频 **1 字节 – 16 MiB**，生成 + 发送共 **30 秒**，全桥接并发 2、每插件 1。

**媒体路由**：`GET /dsh-go/v1/media/<pkg>/<id>?lease=…&key=…`（`routes.js:598-602`）。**手机永远不直接访问插件给的地址**（`docs/plugins/voice.md:25` 原文："手机从不直接访问你给的地址。主机上的 `127.0.0.1:8765` 由你在 `media` 里去取，App 只认桥接的同源媒体路由"）。

**不流式——源码级确证**（`capability-runtime.js:152-153` 原文）：

```js
if (!result || !(result.body instanceof Uint8Array) || !AUDIO_TYPES.has(result.contentType)) fail(502, "invalid-media", "expected bounded audio bytes and supported MIME");
if (result.body.byteLength === 0 || result.body.byteLength > L.mediaBytes) fail(413, "too-large", "audio must be 1..16 MiB");
```

→ 必须先有完整 `Uint8Array` 才能写响应，**没有 chunked 路径**。

#### (2) `voice` —— 只检测开口，**不上传任何声音或文字**

协议（`docs/design/native-plugins-protocol.md:601` 原文）：

> `{ type: "voice", op: "start" | "stop", instance }` | 只检测开口，**不上传任何声音或文字**

源码证据（`app/android/.../nativemedia/VoiceCapture.kt:88-111` 原文关键部分）：

```kotlin
private fun capture(r: AudioRecord, g: SpeechGate, mine: Int) {
    val frame = ShortArray((RATE * SpeechGate.FRAME_MS / 1000).toInt())
    var filled = 0
    try {
        while (running) {
            val n = r.read(frame, filled, frame.size - filled)
            if (!running) break
            if (n < 0) { post(mine) { onError("capture") }; break }
            filled += n
            if (filled < frame.size) continue
            filled = 0
            val level = SpeechGate.levelOf(frame, frame.size)
            val edge = synchronized(g) { g.consume(level, SystemClock.elapsedRealtime()) }
            if (edge != SpeechGate.Edge.NONE) post(mine) { onEdge(edge) }
        }
    }
}
```

→ **只算 RMS 电平，然后把 PCM 帧丢掉**（`SpeechGate.kt:87-97` 的 `levelOf` 只返回 dBFS）。注释也明说（`SpeechGate.kt:15`）："Nothing here keeps or sends audio."

#### (3) `micro` —— 10 Hz 小字幕通道

数据模型（`capabilities.js:36-46` 原文）：

```js
/** A micro value -> the complete small model `{text?, progress?}`, or undefined. */
export function microModel(value) {
  if (!isPlainObject(value)) return undefined;
  const keys = Object.keys(value);
  if (keys.length === 0 || keys.some((k) => k !== "text" && k !== "progress")) return undefined;
  if (value.text !== undefined && (typeof value.text !== "string" || chars(value.text) > CAP_LIMITS.textChars)) return undefined;
  if (value.progress !== undefined && value.progress !== null
    && (typeof value.progress !== "number" || !Number.isFinite(value.progress) || value.progress < 0 || value.progress > 1)) return undefined;
  const json = toJson(value);
  return json && json.bytes <= CAP_LIMITS.microBytes && Object.keys(json.value).length > 0 ? json.value : undefined;
}
```

→ **只允许 `text`（≤240 字）和 `progress`（0–1 或 null）两个键**，序列化 ≤ 1 KiB，多一个键就整个丢弃并计违规。节流 100 ms（`capabilities.js:22` `microMs: 100`；`capability-runtime.js:234-236`）。

#### (4) `overlay` —— App 内悬浮字幕

**这是任务问的"悬浮字幕"，源码级确认边界**（`MediaOverlays.kt:29-36` 已引）。它是 `android.R.id.content` 里加的一个 `LinearLayout`（`MediaOverlays.kt:135-151`），**不是 WindowManager 窗口**，所以：只在 App 在前台时存在；不需要 `SYSTEM_ALERT_WINDOW`；**不覆盖其他 App**。

字幕文字上限 240 字 / 1 KiB（`NativeMediaPlugin.kt:379-380` 原文）：

```kotlin
const val MAX_CAPTION_CHARS = 240
const val MAX_CAPTION_BYTES = 1024
```

### 9.3 「开口即停」是怎么做到的（源码级）

**这是整个语音能力里最硬的一条保证，而且是在 Kotlin 层做的，不依赖网络。**

`NativeMediaPlugin.kt:284-302`（原文，完整）：

```kotlin
private fun onSpeechEdge(edge: SpeechGate.Edge) {
    if (owner == null || capture?.active != true) return
    when (edge) {
        SpeechGate.Edge.START -> {
            // Stop the clip here first; only a released player is proof enough to report.
            val t0 = SystemClock.elapsedRealtimeNanos()
            val hadClip = player?.active == true
            capture?.setPlaying(false)
            if (player?.release() == false) {
                revoke("player")
                return
            }
            Log.i(TAG, "speech: ${if (hadClip) "clip released" else "no clip"} in ${(SystemClock.elapsedRealtimeNanos() - t0) / 1000} µs, then notify")
            emitCurrent("voice.speechStart", localStopped = true)
        }
        SpeechGate.Edge.END -> emitCurrent("voice.speechEnd")
        SpeechGate.Edge.NONE -> {}
    }
}
```

→ **顺序被硬编码**：先 `player.release()`，**只有确认释放成功**才发 `voice.speechStart`（带 `localStopped: true`）。如果释放失败 → `revoke("player")`，**根本不报事件**。

Dart 侧再校验一次（`app/lib/plugins/native/native_capabilities.dart:454-461` 原文）：

```dart
if (kind == 'voice.speechStart') {
  // Only with proof that this owner's playback is already stopped.
  if (!o.granted || !o.roles.contains('voice') || o.speaking || value['localStopped'] != true) return;
  o.speaking = true;
  ++o.audioGeneration;
  // The bridge ends the audio role on speechStart; no audio.stopped follows.
  if (o.roles.remove('audio')) o.ended.add('audio');
  _queue(o, 'voice.speechStart');
  _afterAudio(o);
}
```

→ 没有 `localStopped: true` 就**直接丢弃**。

**"不会再跟 audio.stopped"** 也是源码保证（`capability-runtime.js:116` 原文）：

```js
if (msg.event === "voice.speechStart") { e.audioStopping = true; e.roles.delete('audio'); }
```

**回声处理**（`SpeechGate.kt:7-16` + `VoiceCapture.kt:42-79`）：
- 录音源是 `MediaRecorder.AudioSource.VOICE_COMMUNICATION`（Android 会做回声消除的通道）
- 开了 `AcousticEchoCanceler`（有就用）
- 播放时门槛抬高（`PLAY_MARGIN_DB = 24.0` / 无 AEC 时 `PLAY_MARGIN_NO_AEC_DB = 30.0`，`SpeechGate.kt:76-77`）
- 播放开头 `SETTLE_MS = 400L` 不算（`SpeechGate.kt:85`）
- 播完后 `HANGOVER_MS = 700L`（`SpeechGate.kt:84`）——这对应协议 §11.4 的"说完（约 0.7 秒静音）"

**诚实的局限**（`docs/plugins/voice.md:107` 原文）：

> 外放很响、设备没有回声消除时，朗读可能被自己的声音打断，小声说话也可能听不出来。戴耳机最稳。

### 9.4 效果与事件表（源码原文）

**效果**（`invoke.schema.json` 的 `$defs.audio` / `voice` / `overlay`）：

```json
{ "type": "audio", "op": "play", "instance": "<id>", "src": "media:<id>", "key": "<可选>" }
{ "type": "audio", "op": "pause" | "resume" | "stop", "instance": "<id>" }
{ "type": "voice", "op": "start" | "stop", "instance": "<id>" }
{ "type": "overlay", "op": "show" | "hide", "instance": "<id>", "id": "<overlay id>" }
```

（`lease` 由桥接自己加，插件写的忽略——`invoke.schema.json:400-404` 的 description："Bridge-added opaque connection-bound lease, never supplied as authority by the plugin."）

**事件**（`capabilities.js:25-31` 原文）：

```js
export const CAP_EVENTS = Object.freeze({
  "audio.ended": "audio", "audio.stopped": "audio", "audio.error": "audio",
  "voice.speechStart": "voice", "voice.speechEnd": "voice", "voice.stopped": "voice", "voice.error": "voice",
  "overlay.closed": "overlay",
});
/** Events that end their role (each role ends at most once per lease). */
export const TERMINAL_EVENTS = new Set(["audio.ended", "audio.stopped", "audio.error", "voice.stopped", "voice.error", "overlay.closed"]);
```

`onCapability` 收到的 event 类型（`sdk/plugin-kit/types/index.d.ts:52` 原文）：

```ts
export interface CapabilityEvent { instance: string; event: 'audio.ended' | 'audio.stopped' | 'audio.error' | 'voice.speechStart' | 'voice.speechEnd' | 'voice.stopped' | 'voice.error' | 'overlay.closed'; reason?: string; seq: number; signal: AbortSignal }
```

**`reason` 的真实取值**（源码级）：
- `voice.error` 的 reason：`'declined'`（用户不同意）/ `'permission'`（系统没给录音权限）—— `native_capabilities.dart:330`、`338`
- `owner.revoked` 的 reason：`'user'`（点指示条停止）/ `'dictation'`（输入框语音输入抢走麦克风）/ `'background'`（离开屏幕）/ `'detached'` / `'player'` —— `NativeMediaPlugin.kt:63-64`、`97`、`130`、`292-294`
- `overlay.closed` 的 reason：`'user'`（点了 ×）/ `'unsupported'` / `'error'` / `'audio-ended'` —— `NativeMediaPlugin.kt:63`、`native_capabilities.dart:370`、`501`
- `voice.stopped` 的 reason：`'audio-ended'` / `'speech-ended'` / `'declined'` —— `native_capabilities.dart:467`、`490`、`660`

### 9.5 后台/前台限制（源码级）

`NativeMediaPlugin.kt:93-98`（原文）：

```kotlin
override fun onActivityStopped(a: Activity) {
    if (a !== activity) return
    stopped = true
    // Off the screen (another app, the lock screen): stop here and now.
    revoke("background")
}
```

Dart 侧（`native_capabilities.dart:220-224` 原文）：

```dart
/// The app is on screen (resumed or inactive) or not; leaving stops all.
void foreground(bool value) {
  _foreground = value;
  if (!value) unawaited(revoke());
}
```

`_current()` 同时校验 stream 连接与 session 一致（`native_capabilities.dart:213-218` 原文）：

```dart
bool _current(_Owner o) {
  if (_disposed || _nativeFailed || !_foreground || !identical(_owner, o)) return false;
  if (stream.conn != o.conn || session() != o.scope.sessionId) return false;
  final p = plugin(o.pkg);
  return p != null && p.native && o.claimed.every(p.capabilities.contains);
}
```

**"一次一个"**：`NativeMediaPlugin.kt:186-189`（原文注释："A new owner replaces the old one, which Dart has disposed (or is disposing) already."）+ `disposeLocal()`。

**麦克风仲裁（输入框语音输入优先）**：`MicArbiter`（`nativemedia/MicArbiter.kt`，1513 B）。`NativeMediaPlugin.kt:256`：

```kotlin
if (!MicArbiter.acquire(micHolder)) return result.error("busy", "dictation is using the microphone", null)
```

反向也成立：`VoicePlugin.kt:186` 开始听写时 `MicArbiter.beginDictation()`，会触发 `micHolder` 的 `revoke("dictation")`（`NativeMediaPlugin.kt:64`）。

**必须有 App 指示条才能听**（`NativeMediaPlugin.kt:257-261` 原文）：

```kotlin
// The indicator first: no listening without it.
if (!overlays.showIndicator(o.pluginTitle)) {
    MicArbiter.release(micHolder)
    return result.error("indicator", "nothing to show the indicator in", null)
}
```

### 9.6 麦克风权限的两级同意

**第一级（App 自己的同意，Dart）**：`app/lib/ui/plugins/native/native_voice_consent.dart:14-38` —— 按 **host + package** 记住，弹窗文案 `plugslotsVoiceAsk(title)`。
**第二级（Android RECORD_AUDIO）**：`NativeMediaPlugin.kt:234-250`，`host.requestPermissions(arrayOf(Manifest.permission.RECORD_AUDIO), REQUEST_CODE)`。

顺序在 `native_capabilities.dart:326-340`（原文）：

```dart
if (o.denied || o.roles.contains('voice') || o.ended.contains('voice')) return false;
final allowed = consent == null ? false : await consent(pkg);
if (!_current(o)) return false;
if (!allowed) { _deny(o, 'declined'); return false; }
// The system permission (asked once by Android, then remembered).
final granted = await platform.call('requestVoiceGrant', o.args);
if (!_current(o)) return false;
if (granted != true) { _deny(o, 'permission'); return false; }
```

### 9.7 清单写法（v1.2）—— 完整可跑示例

`examples/plugins/dsh-go-example-readaloud/package.json:31-45`（原文）：

```json
"dshGo": {
  "v": 1,
  "title": { "zh": "朗读", "en": "Read aloud" },
  "icon": "play",
  "wants": ["message"],
  "capabilities": ["audio", "voice", "micro", "overlay"],
  "contributes": {
    "message.actions": [
      { "id": "read", "label": { "zh": "朗读", "en": "Read aloud" }, "icon": "play", "requires": ["audio"] }
    ],
    "media": [{ "id": "speech" }],
    "micro": [{ "id": "caption" }],
    "overlays": [{ "id": "subtitle", "channel": "caption" }]
  }
}
```

插件半侧（`dsh-go-example-readaloud/lib/index.js:19-47` 原文，完整）：

```js
export function apply(ctx) {
  const readings = createReadings()

  ctx.inject(['dshGo'], (ctx) => {
    ctx.dshGo.on('read', (scope, event) => {
      const key = readings.begin(scope, scope.message)
      if (!key) return { toast: event?.locale?.startsWith('zh') === false ? 'Nothing to read in this reply' : '这条回复没有可读的文字' }
      // One instance for the three: the bridge gives them one lease, the app one owner.
      return [
        { type: 'audio', op: 'play', instance: 'reading', src: 'media:speech', key },
        { type: 'voice', op: 'start', instance: 'reading' },
        { type: 'overlay', op: 'show', instance: 'reading', id: 'subtitle' },
      ]
    })

    ctx.dshGo.media('speech', (scope, { key }) => {
      const body = readings.clip(scope, key)
      if (!body) throw new Error('no such reading')
      return { body, contentType: 'audio/wav' }
    })

    ctx.dshGo.micro('caption', { data: (scope, emit) => readings.watch(scope, emit) })

    ctx.dshGo.onCapability((scope, event) => {
      // Record synchronously: the event's signal is aborted once its lease is over.
      if (event.event === 'voice.speechStart' || event.event.startsWith('audio.')) readings.end(scope)
    })
  })
}
```

**这个示例里没有真 TTS** —— 它合成占位音（`dsh-go-example-readaloud/lib/speech.js:1-4` 原文："A reply as something to listen to, without any speech service: ... A stand-in for TTS: a real plugin asks its own engine on the Host for the clip"）。

### 9.8 TTS / ASR 到底能不能用：把边界说清楚

**TTS：**
- dsh-go **不提供任何 TTS 引擎、TTS 插件、TTS HTTP 接口**。我全仓库 grep `TextToSpeech` = **0 命中**；`tts` 只出现在文档/示例里作为"你做的那部分"。
- 唯一路径：**插件自己在主机上生成音频字节**。可以是调装在主机上的本地 TTS CLI、可以是请求 `http://127.0.0.1:xxxx`（`docs/plugins/voice.md:25` 与 `dsh-go-example-readaloud/lib/index.js:12-14` 都以此举例），可以是任何办法。
- **不是"插件调 Android TTS"**：App 不做 TTS，`NativeMediaPlugin` 只做播放（`ClipPlayer`）。Android 的 `android.speech.tts.TextToSpeech` **在本仓库里完全没被使用**。

**ASR：分三个完全不同的东西，不要混。**

| | 谁用 | 插件能用吗 | 源码 |
|---|---|---|---|
| 原生插件的 `voice` 能力 | 插件 | **只能检测开口/说完，拿不到声音也拿不到文字** | `capabilities.js:25-31`、`VoiceCapture.kt` |
| App 自己的输入框语音输入 | App 用户 | **不能**。插件无入口 | `app/android/.../voice/VoicePlugin.kt`（MethodChannel `dsh/voice`，只给 App 自己的 Dart 用） |
| 引擎侧 SenseVoice 模型 | App 的语音输入 | **不能**。那是 DSH 官方语音套件的 `speech/transcribe` | `docs/design/voice-input.md:73-81`、`runtime/SpeechGuest.kt` |

`VoicePlugin.kt:28-37`（原文）：

> - `startRecognizer`: the system `SpeechRecognizer` (Google's or the vendor's service) with partial results; `stop` asks for the final text, which arrives as an event.
> - `startRecorder`: 16 kHz mono PCM16 for the engine's own recognizer ([WavRecorder]); `stop` returns the WAV bytes.

**两者都是 `dsh/voice` MethodChannel，App 内部通道，插件拿不到。** 插件能拿到的 ASR 只有"用户开口了/说完了"这两个信号。

**引擎侧 ASR 是整段、不流式的**（`docs/design/voice-input.md:80` 原文）：

> `speech/transcribe` | 整段 16 kHz WAV，返回最终文字（**不流式**）

### 9.9 语音能力的成熟度与主机覆盖

| 项 | 状态 | 证据 |
|---|---|---|
| App 侧能力运行时（Dart） | 已实现 | `native_capabilities.dart` 726 行 |
| Android 侧播放/麦克风/字幕 | 已实现 | `nativemedia/` 五个文件 |
| 桥接侧租约/媒体路由/micro | 已实现 | `capability-runtime.js` 256 行 |
| 协议/文档/示例 | 齐全 | `docs/plugins/voice.md` + readaloud 示例 |
| 真机验证 | **NOT_TESTED** | 全仓库未见真机证据 |
| 哪里能用 | **本机引擎可用。SSH 主机要等桥接能装上；中转主机没有这些能力**（`docs/plugins/voice.md:125`） | 〔文档说的〕 |

`docs/plugins/voice.md:125` 原文：
> 本机引擎可用。SSH 主机要等桥接能装上（见主机覆盖）；中转主机没有这些能力，带 `requires` 的按钮不会出现。

### 9.10 ⭐ 「让 agent 主动开一轮对话 / 定时唤醒 / 通知后唤醒」—— 有，但要分清是谁的能力

**明确答案：有。而且是本仓库里最完整、测试最扎实的子系统。但它有两条硬约束，直接决定你们能不能用。**

#### 9.10.1 它是什么

**两层：**

**层 1（App 侧，Kotlin）：proactive kernel** —— `app/android/.../loop/core/`（纯 Kotlin，无 Android 依赖）
- `ProactiveScheduler.kt`（373 行）：随机排期
- `Coordinator.kt`（477 行）：仲裁状态机
- `CoreModel.kt`（216 行）：数据模型
- `AgentWire.kt`（76 行）：协议编解码
- 适配层 `app/android/.../loop/proactive/`：`ProactiveRuntime.kt`（417 行）、`WakeAlarm.kt`（56 行）、`NotificationPortImpl.kt`（152 行）、`ProactivePageActivity.kt`（183 行）、`ProactivePlugin.kt`（134 行）

**层 2（引擎侧，Node）：主机插件 `dsh-go-proactive`** —— `runtime/proactive/dsh-go-proactive/`
- 路由前缀 `/dsh-go-proactive/v1`
- 随 runtime 打包到 `/opt/dsh-go/dsh-go-proactive`（`runtime/build-runtime.sh:395-431`、`runtime/lib/rtbuild.py:906-913`）〔已读源码验证〕

#### 9.10.2 关键接口（原文）

`runtime/proactive/dsh-go-proactive/README.md:2-3`（原文）：

> DSH 宿主插件。把手机端的一条主动事件交给**一个已经活着、正空闲**的 DSH 会话，作为一次 `followup` 回合，再把这回合的结果如实报回 App。

**五条路由**（`lib/index.js:4-8` 原文）：

```
POST /dsh-go-proactive/v1/request   {v, requestId, eventId, kind, payload, sessionId?, deadlineMs}
GET  /dsh-go-proactive/v1/result?requestId=…&waitMs=…   long poll (≤ 25 s)
POST /dsh-go-proactive/v1/cancel    {requestId}
GET  /dsh-go-proactive/v1/status    configured session, live agents, counters
POST /dsh-go-proactive/v1/config    {sessionId}  the session proactive turns go to
```

**状态机**（`lib/core.js:3-9` 原文）：

```
accepted   the message is in the agent's inbox (agent.followup returned without throwing)
claimed    the loop took it: its user/message is in the session log and a turn is running
generated  that turn ended `completed` (or `max-tokens`) with visible assistant text
failed     the turn ended otherwise, the agent went away, or the deadline passed
cancelled  the App withdrew it; any later text is never handed out
```

**409 的八种拒绝**（`README.md:39-46` 原文）：

```
409 no-session          没指定、也没配置目标会话
409 no-agent            会话不存在或没活着。插件从不新建或唤醒会话
409 archived            会话已归档
409 agent-busy          agent 正在跑（包括用户自己正在对话、等审批）或收件箱有待处理消息
409 desk-busy           插件已有一个在途请求（一次只处理一个）
409 event-in-flight     同一个 eventId 已有在途请求
409 request-id-conflict requestId 已用于另一个 eventId
409 followup-failed     followup 当场抛错
```

#### 9.10.3 ⚠️ 两条硬约束（判断"DSH 侧主动调度能不能落地手机端"的关键）

**约束 1：它从不新建、也从不唤醒会话。**

`lib/core.js:130-136`（原文）：

```js
const sessionId = req.sessionId ?? this.deps.defaultSession()
if (!sessionId) return this.refuse(409, 'no-session')
if (this.deps.isArchived(sessionId)) return this.refuse(409, 'archived')
const agent = this.deps.findAgent(sessionId)
if (agent === undefined) return this.refuse(409, 'no-agent') // never woken or created here
if (agent.status !== undefined && agent.status !== 'idle') return this.refuse(409, 'agent-busy')
if (agent.inbox?.hasPending) return this.refuse(409, 'agent-busy')
```

`README.md:18-19` 补充前提（原文）：

> `ctx.agents.get(id)` 只返回**活着的** agent；`status` 只有 `idle` / `running`（等审批、等回答也算 running）

→ **必须同时满足：①一个已经存在的会话 ②该 agent 还活着且在 idle ③收件箱空。** 用户自己正在聊时**永远推不进去**（`agent-busy`）。

**约束 2：主动回合里工具被硬性禁用。**

`lib/index.js:86-93`（原文）：

```js
// Hard limit at the tool layer (review r3, E): the prompt asks for no tools; this makes sure.
// global: the registry dispatches with the agent as carrier, which would otherwise filter us out.
// prepend: run before any other policy, so nothing can approve a call we deny.
ctx.on('tools/pre-execute', (exec, next) => {
  let denied
  try { denied = desk.guardTool(exec?.agent?.id ?? exec?.agent?.session?.id) } catch { denied = undefined }
  return denied ?? next()
}, { global: true, prepend: true })
```

`lib/core.js:243-247`（原文）：

```js
guardTool(sessionId) {
  if (sessionId == null || !this.proactiveTurns.has(sessionId)) return undefined
  this.counters.toolDenied++
  return { kind: 'deny', reason: 'dsh-go-proactive: tools are disabled in a proactive turn (no phone control, no files, no commands)' }
}
```

→ **主动回合里模型只能说话，不能调任何工具**（不能操作手机、不能读文件、不能跑命令）。

#### 9.10.4 定时唤醒的真实能力（Kotlin 侧）

`ProactiveScheduler.kt:6-27`（原文，`SchedulerConfig` 默认值）：

```kotlin
data class SchedulerConfig(
    val enabled: Boolean = false,
    /** A random gap in [minIntervalMs, maxIntervalMs] between attempts. */
    val minIntervalMs: Long = 45 * 60_000L,
    val maxIntervalMs: Long = 3 * 3_600_000L,
    /** Quiet hours as minutes of the local day; equal values switch them off. Crossing midnight is fine. */
    val quietStartMinute: Int = 23 * 60,
    val quietEndMinute: Int = 8 * 60,
    /** Deliveries per local day (posted surfaces, seen or not). 0 = no limit. */
    val dailyMax: Int = 6,
    /** Right after the person used DSH or answered, wait at least this long. */
    val userActiveDeferMs: Long = 20 * 60_000L,
    /** Each confirmed-but-ignored message multiplies the next gap by this… */
    val ignoredBackoff: Double = 2.0,
    /** …up to this factor. */
    val maxBackoffFactor: Double = 8.0,
    /** After this many confirmed-and-ignored in a row, stop until the person shows up. 0 = never. */
    val pauseAfterIgnored: Int = 3,
    val priority: Priority = Priority.NORMAL,
    /** How long a scheduled attempt may wait in the coordinator. */
    val eventTtlMs: Long = 15 * 60_000L,
)
```

→ **默认：45 分钟 – 3 小时的随机间隔、23:00–08:00 免打扰、每天最多 6 次、用户刚用过就推后 20 分钟、连续 3 次已读不回就暂停。**

判定顺序（`ProactiveScheduler.kt:162-182` 原文 `evaluate`）：

```kotlin
val d = when {
    !c.enabled -> Decision.Wait(0, "disabled")
    s.pausedForIgnored -> Decision.Wait(0, "paused_ignored")
    s.pendingEventId != null -> Decision.Wait(now.wall + RECHECK_MS, "pending")
    inQuiet(now.wall) -> Decision.Wait(quietEndAfter(now.wall), "quiet")
    c.dailyMax > 0 && state.countToday >= c.dailyMax -> Decision.Wait(nextDayStart(now.wall), "daily_max")
    !due(now) -> Decision.Wait(dueWall(now), "not_due")
    userRecentlyActive(now) -> Decision.Wait(userActiveUntil(now), "user_active")
    else -> Decision.Fire(EventRequest(
        eventId = ids(), source = "SCHEDULER", kind = "random_chat", dedupeKey = "scheduler:random",
        priority = c.priority, createdAt = now.wall, ttlMs = c.eventTtlMs, needsAgent = true,
        payload = mapOf("streak" to s.ignoredStreak.toString(), "countToday" to state.countToday.toString()),
    ))
}
```

**`WakeMode` 三级降级**（`ProactiveScheduler.kt:51-67` 原文）：

```kotlin
enum class WakeMode {
    /** One exact alarm at the next decision time. */
    EXACT,
    /** One inexact alarm; the attempt may come late, never early. */
    INEXACT,
    /** No alarms: checks happen only when DSH is opened or its accessibility service sees a window change. */
    OPPORTUNISTIC,
    ;

    companion object {
        fun of(caps: BackgroundCaps): WakeMode = when {
            caps.backgroundRestricted || !caps.alarmsAvailable -> OPPORTUNISTIC
            caps.exactAlarms -> EXACT
            else -> INEXACT
        }
    }
}
```

`caps()` 实测（`ProactiveRuntime.kt:390-398` 原文）：

```kotlin
fun caps(): BackgroundCaps {
    val am = app.getSystemService(ActivityManager::class.java)
    val alarms = app.getSystemService(AlarmManager::class.java)
    return BackgroundCaps(
        alarmsAvailable = alarms != null,
        exactAlarms = Build.VERSION.SDK_INT < 31 || alarms?.canScheduleExactAlarms() == true,
        backgroundRestricted = Build.VERSION.SDK_INT >= 28 && am?.isBackgroundRestricted == true,
    )
}
```

#### 9.10.5 「通知后唤醒」—— 有，但语义要说准

`NotificationPortImpl.kt:16-33`（原文）：

```kotlin
/**
 * D2: the notification surface of the proactive kernel ([NotificationPort]).
 *
 * Same discipline as `NotificationGate.showPrompt`, and one rule that is *stricter* than it:
 * **a notification never reports [io.github.ccch1mneyyy.dsh_app.loop.core.SurfaceReport.Shown]**.
 *
 * Posting proves only that the system accepted the notification — the person may never look at it,
 * or may look and swipe it away. So:
 *  - after [post] succeeds, the port reports nothing on its own; [ProactiveRuntime] counts the
 *    delivery from the kernel's own `Posted` report;
 *  - the 回应 button reports `Answered("reply")`, i.e. the person really did respond;
 *  - swiping it away reports `Dismissed`, which is *not* evidence of having read it;
 *  - the deadline reports nothing: the kernel's own timer closes the event as TIMEOUT.
 */
```

→ **通知是"展示面"，不是"唤醒源"。** 它由 scheduler / alarm 触发后发出，点了「回应」按钮才会 `Answered`。**点了之后不会自动开一轮对话** —— `Answered` 只是给调度器一个 `Engagement.RESPONDED` 回执（`ProactiveScheduler.kt:198-218`），并在 `ProactiveRuntime.onSurface` 里调 `scheduler?.onUserActivity(clocks())`（`ProactiveRuntime.kt:194`）。

#### 9.10.6 因此：DSH 侧主动调度能不能真落地手机端？

**能落地，但有五条必须接受的现实：**

1. **必须由 App 的 proactive kernel 发起**，不能由"DSH 侧某个插件/定时任务主动调手机"。方向是 **App → 引擎**，不是引擎 → App。理由：`dsh-go-proactive` 暴露的是 HTTP 路由，**由 App 的 Dart 层带着 cookie 转发**（`app/lib/proactive/proactive_channel.dart:1-16` 原文："The kernel (loop/core) never touches the network itself: it asks Dart, which is the side that holds the engine's authenticated cookie"）。引擎侧没有任何主动连 App 的通道。

2. **必须有已经活着且 idle 的会话**（§9.10.3 约束 1）。目标会话由 `POST /config {sessionId}` 设（`ProactivePlugin.kt:112-114` 的 `agentSessionSet`）。

3. **主动回合里模型不能调任何工具**（§9.10.3 约束 2）。

4. **后台可靠性没有任何真机证据。** `docs/loop/VALIDATION.md:5`（原文）："**本轮所有真机项都是 NOT_TESTED**"。`docs/loop/ROUND2.md:167`（原文）："**NOT_TESTED（没有真机证据）**：所有 Android 行为——无障碍服务启动页面是否被国产 ROM 拦截、闹钟在省电策略下是否送达、开机广播、强行停止后的恢复…"。

5. **插件无法自己用这套能力。** 它**不是 `ctx.dshGo` 的成员**（§2.2 的 9 个成员里没有），也**不在 `dshGo` 清单里**（§1.1 的 8 个顶层键里没有）。它是 App 的一个设置页开关（入口：设置 → 手机操控 → 主动搭话，`phone_control_screen.dart:359-371`）。

**协议文档里的明确禁止项**（`docs/design/native-plugins-protocol.md:482` 原文）：

> **永不开放**：批准工具调用、改权限模式、手机操控、装卸插件、切主机、读 App 设置或凭据、**无点按自动触发（总设计 §3.5）**

→ 插件**永远不可能**"自己开一轮对话"。

#### 9.10.7 这套东西的成熟度（诚实评级）

| 项 | 状态 | 证据 |
|---|---|---|
| 内核纯逻辑（仲裁/调度/编解码） | **已实现 + 66 个 JVM 测试通过** | `docs/loop/ROUND2.md:138` |
| 主机插件 `dsh-go-proactive` | **已实现 + 10/10 纯逻辑 + 26/26 真实引擎 e2e（文档声称）** | `docs/loop/ROUND2.md:140-141` |
| Android 适配层 | **已实现，只做了编译检查** | `docs/loop/ROUND2.md:163` |
| 真机行为 | **NOT_TESTED** | `docs/loop/VALIDATION.md:5`、`ROUND2.md:167` |
| 是否随 runtime 打包 | **是** | `runtime/build-runtime.sh:395-431`、`runtime/lib/rtbuild.py:906-913`〔已读源码验证〕 |


---

## 10. Android 权限模型

### 10.1 AndroidManifest 里声明了什么（`app/android/app/src/main/AndroidManifest.xml`，逐条）

| 行 | 权限 | 用途（manifest 注释原文） |
|---|---|---|
| 4 | `INTERNET` | — |
| 5 | `ACCESS_NETWORK_STATE` | — |
| 6 | `FOREGROUND_SERVICE` | 前台服务（`DshService`、`PetOverlayService`） |
| 7 | `FOREGROUND_SERVICE_SPECIAL_USE` | 同上，类型 `specialUse` |
| 8 | `POST_NOTIFICATIONS` | 通知 |
| 9 | `WAKE_LOCK` | `DshService` 持锁 |
| 11 | `RECEIVE_BOOT_COMPLETED` | "Proactive scheduler (loop/proactive/WakeAlarm.kt): re-plan the single alarm after reboot." |
| 12 | `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | 电池优化豁免申请 |
| 14 | `RECORD_AUDIO` | "Composer voice input (voice/VoicePlugin.kt), asked for on the first tap of the mic." |
| 19 | `com.android.alarm.permission.SET_ALARM` | `intent kind=alarm/timer` |
| 21-22 | `READ_EXTERNAL_STORAGE` / `WRITE_EXTERNAL_STORAGE` | "可选「访问手机文件」模式" |
| 23-25 | `MANAGE_EXTERNAL_STORAGE`（`tools:ignore="ScopedStorage"`） | 同上 |
| 28 | `moe.shizuku.manager.permission.API_V23` | 不打扰模式（Shizuku shell-uid） |
| 31 | `SYSTEM_ALERT_WINDOW` | 注释原文："App 外悬浮窗 (pet/PetOverlay.kt): the floating pet over other apps, off until the person turns it on." |
| 34 | `REQUEST_INSTALL_PACKAGES` | 应用内更新 |
| 38 | `READ_PHONE_STATE`（`tools:node="remove"`） | 显式移除 |

**关键：`AndroidManifest.xml:15-17` 有一条 TODO 明确说了 `SpeechRecognizer` 在 targetSdk≥30 会有包可见性问题**（当前 targetSdk=28 所以没问题）：

```xml
<!-- TODO(targetSdk >= 30): package visibility hides recognition services from
     SpeechRecognizer.isRecognitionAvailable unless declared; add to <queries>:
     <intent><action android:name="android.speech.RecognitionService" /></intent> -->
```

**没有声明的**（我逐个 grep 确认 0 命中）：
- `SCHEDULE_EXACT_ALARM` / `USE_EXACT_ALARM` → **0 命中**
- `QUERY_ALL_PACKAGES` → 0
- `BIND_ACCESSIBILITY_SERVICE` 只在 `<service android:permission=...>` 上（`AndroidManifest.xml:241`），不是 `uses-permission`

### 10.2 逐项：怎么声明 / 怎么申请 / 怎么检测

#### 浮窗 `SYSTEM_ALERT_WINDOW`
- 声明：`AndroidManifest.xml:31`
- **申请**：跳系统设置页（`pet/PetOverlayPlugin.kt:87-89` 原文）：

```kotlin
"requestPermission" -> result.success(
    start(Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION, Uri.parse("package:${context.packageName}"))) || openAppDetails(),
)
```

- **检测**：**不信 `canDrawOverlays`**，以"能不能真的 addView 成功"为准（`pet/OverlayAccess.kt:22-46` 原文）：

```kotlin
/**
 * The last attempt to add a TYPE_APPLICATION_OVERLAY window: null when none was made since the
 * permission last changed. The attempt is what counts — `Settings.canDrawOverlays` is wrong on
 * some vendor builds in both directions.
 */
enum class AddResult { OK, FAILED }

object OverlayAccess {
    fun permission(a11yConnected: Boolean, canDraw: Boolean, lastAdd: AddResult?): OverlayPermission = when {
        a11yConnected -> OverlayPermission.ACCESSIBILITY
        lastAdd == AddResult.OK -> OverlayPermission.GRANTED
        lastAdd == AddResult.FAILED -> if (canDraw) OverlayPermission.BLOCKED else OverlayPermission.DENIED
        canDraw -> OverlayPermission.GRANTED
        else -> OverlayPermission.DENIED
    }
```

四种状态 `OverlayPermission`（`OverlayAccess.kt:15-20`）：`ACCESSIBILITY` / `GRANTED` / `BLOCKED`（厂商开关，`canDrawOverlays` 看不见）/ `DENIED`。

**谁用**：只有桌宠（`pet/`）。**插件拿不到。**

#### 麦克风 `RECORD_AUDIO`
- 声明：`AndroidManifest.xml:14`
- **申请**：两个入口都要 `host.requestPermissions(arrayOf(Manifest.permission.RECORD_AUDIO), …)`：
  - `nativemedia/NativeMediaPlugin.kt:239`（插件 voice 能力）
  - `voice/VoicePlugin.kt:132`（App 输入框语音）
- **检测**：`appContext.checkSelfPermission(Manifest.permission.RECORD_AUDIO) == PackageManager.PERMISSION_GRANTED`
- **三态返回**（`VoicePlugin.kt:145-153` 原文）：`"granted"` / `"blocked"`（`!host.shouldShowRequestPermissionRationale(...)`）/ `"denied"`

#### 通知
- 声明 `POST_NOTIFICATIONS`（`AndroidManifest.xml:8`）
- **不主动申请**：全仓库 `POST_NOTIFICATIONS` 在 Kotlin 里 **0 命中**。原因在 `build.gradle.kts:44`：`targetSdk = 28`，而 `POST_NOTIFICATIONS` 是 API 33 才要运行时申请。**targetSdk 28 的应用不受该限制。**
- **检测**：`NotificationManager.areNotificationsEnabled()`（9 处调用，见 `ProactiveRuntime.kt:377`、`NotificationPortImpl.kt:129` 等）
- **打开设置**：各插件把用户指引到系统页（`SystemActions.kt:318` 附近）

#### 前台服务
- 声明两个 service，都是 `foregroundServiceType="specialUse"`（`AndroidManifest.xml:218-235`）：

```xml
<!-- Keeps the local DSH process alive while the app is in the background. -->
<service
    android:name=".runtime.DshService"
    android:exported="false"
    android:foregroundServiceType="specialUse">
    <property
        android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
        android:value="Runs the user's local DeepSeek Harness agent server" />
</service>

<!-- Keeps the floating pet's process up when only remote hosts are used (pet/PetOverlayService.kt). -->
<service
    android:name=".pet.PetOverlayService"
    android:exported="false"
    android:foregroundServiceType="specialUse">
    <property
        android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
        android:value="Shows the user's floating assistant pet while their remote agent session runs" />
</service>
```

- **启动**（`runtime/RuntimeController.kt:224`）：`if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) app.startForegroundService(intent) else app.startService(intent)`
- **进前台**（`runtime/DshService.kt:116-122` 原文）：

```kotlin
private fun foreground(notification: Notification) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.UPSIDE_DOWN_CAKE) {
        startForeground(NOTIFICATION_ID, notification, ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE)
    } else {
        startForeground(NOTIFICATION_ID, notification)
    }
}
```

- `DshService` 是 `START_NOT_STICKY`（`docs/loop/AUDIT.md:69`："前台服务，START_NOT_STICKY（被杀不自动重启）"）

#### 电池优化
- 声明 `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`（`AndroidManifest.xml:12`）
- **三级回退申请**（`runtime/RuntimePlugin.kt:177-193` 原文）：

```kotlin
/** The system's "ignore battery optimizations" dialog, else the closest settings page that opens. */
private fun requestBatteryExempt(): Boolean {
    val packageUri = Uri.parse("package:${appContext.packageName}")
    val host = activity?.activity
    for (intent in listOf(
        Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS, packageUri),
        Intent(Settings.ACTION_IGNORE_BATTERY_OPTIMIZATION_SETTINGS),
        Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS, packageUri),
    )) {
        val started = runCatching {
            if (host != null) host.startActivity(intent)
            else appContext.startActivity(intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK))
        }.isSuccess
        if (started) return true
    }
    return false
}
```

- 另有**厂商设置页**跳转（`RuntimePlugin.kt:199-215`，`VendorSettings.pages(vendor, switch)`）——国产 ROM 的后台策略各不相同。

#### 无障碍
- 声明为 service（`AndroidManifest.xml:237-249`），配置在 `res/xml/phone_accessibility_service.xml`
- **关键：`android:isAccessibilityTool="false"`**（xml 第 5 行注释原文："Not an accessibility tool (isAccessibilityTool=false): Play / Play Protect rules, and accessibilityDataSensitive views stay hidden from us, which is the safe side."）
- 权限是 `BIND_ACCESSIBILITY_SERVICE`（系统签名级），用户手动在系统设置里开启
- **谁用**：DSH 自己的手机操控（`phone/a11y/`）。**插件拿不到。**

#### targetSdk = 28（这个数字很重要）

`app/android/app/build.gradle.kts:38-47`（原文）：

```kotlin
defaultConfig {
    applicationId = "io.github.ccch1mneyyy.dsh_app"
    // java.nio.file (symlinks while unpacking the rootfs) needs API 26.
    minSdk = 26
    // Target 28 like Termux, so the API 29+ exec restriction on app-private
    // files does not apply to binaries inside the rootfs.
    targetSdk = 28
    versionCode = flutter.versionCode.takeIf { it > 1 } ?: buildMinuteVersionCode()
    versionName = flutter.versionName
}
```

影响：`POST_NOTIFICATIONS` 不用申请；包可见性无限制；Android 10+ 的后台启动限制与 targetSdk 无关，但**系统绑定的无障碍服务是官方豁免项**（`docs/loop/AUDIT.md:76`）。

### 10.3 权限 → 谁能用（第三列最重要）

| 权限 | App 自己用在哪 | 原生插件能用吗 |
|---|---|---|
| `SYSTEM_ALERT_WINDOW` | 桌宠悬浮窗 | **不能** |
| `RECORD_AUDIO` | 输入框语音 + 插件 voice 能力（经 App 转手） | **能，但只能检测开口**，拿不到音频 |
| `POST_NOTIFICATIONS` | 对话通知、审批通知、主动搭话 | **不能** |
| `FOREGROUND_SERVICE` | 保活引擎 / 桌宠 | **不能** |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | 引擎保活 | **不能** |
| `BIND_ACCESSIBILITY_SERVICE` | 手机操控 | **不能** |
| `READ/WRITE/MANAGE_EXTERNAL_STORAGE` | 访问手机文件 | **不能**（插件用 `files` 能力走 SAF，见 `docs/plugins/files.md`） |

---

## 11. 后台保活：WorkManager / 前台服务 / WakeAlarm

### 11.1 直接回答三个问题

| 机制 | 有没有 | 证据 |
|---|---|---|
| **WorkManager** | **没有**。全仓库 Kotlin/Dart/Manifest 里 `WorkManager` **0 命中** | grep 实测 |
| **JobScheduler** | **没有**。`JobScheduler` **0 命中** | grep 实测 |
| **AlarmManager（WakeAlarm）** | **有**，单一闹钟 | `loop/proactive/WakeAlarm.kt` |
| 前台服务 | **有**，两个 `specialUse` | §10.2 |
| 无障碍事件当 opportunistic 时钟 | **有** | `ProactiveRuntime.kt:162-168` |

`docs/loop/AUDIT.md:71-72`（改造前的审计，原文）：

> | 开机/升级恢复 | **没有** `BOOT_COMPLETED`、`MY_PACKAGE_REPLACED` 接收器 |
> | 定时 | **没有** AlarmManager / WorkManager / JobScheduler |

改造后：`BOOT_COMPLETED` + `MY_PACKAGE_REPLACED` 加了（`AndroidManifest.xml:273-280`），AlarmManager 加了（`WakeAlarm.kt`），**WorkManager / JobScheduler 仍然没有**。

### 11.2 `WakeAlarm` 的真实接口（`loop/proactive/WakeAlarm.kt` 全文 56 行）

```kotlin
package io.github.ccch1mneyyy.dsh_app.loop.proactive

import android.app.AlarmManager
import android.app.PendingIntent
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.util.Log
import io.github.ccch1mneyyy.dsh_app.loop.core.WakeMode

/**
 * The scheduler's single alarm. An alarm is only a reminder to evaluate again; the decision is
 * always made from the persisted state, so a late, lost or duplicate alarm cannot cause a wrong one.
 */
object WakeAlarm {
    const val ACTION = "io.github.ccch1mneyyy.dsh_app.loop.PROACTIVE_WAKE"
    @Volatile var lastError: String? = null
        private set

    fun set(context: Context, atWall: Long?, mode: WakeMode) {
        val am = context.getSystemService(AlarmManager::class.java) ?: return
        val pi = pending(context)
        am.cancel(pi)
        if (atWall == null || mode == WakeMode.OPPORTUNISTIC) return
        lastError = try {
            if (mode == WakeMode.EXACT) am.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, atWall, pi)
            else am.setAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, atWall, pi)
            null
        } catch (t: Throwable) {
            // e.g. SecurityException on a ROM that withdrew exact alarms: degrade, never crash.
            runCatching { am.setAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, atWall, pi) }
            "${t.javaClass.simpleName}: ${t.message}"
        }
    }

    private fun pending(context: Context): PendingIntent = PendingIntent.getBroadcast(
        context, 0, Intent(context, ProactiveWakeReceiver::class.java).setAction(ACTION),
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE,
    )
}

/**
 * Alarm, boot and app update. After a force stop Android delivers none of these until DSH is
 * opened again; the settings page says so instead of promising background recovery.
 */
class ProactiveWakeReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent?) {
        val action = intent?.action ?: return
        if (action != WakeAlarm.ACTION && action != Intent.ACTION_BOOT_COMPLETED && action != Intent.ACTION_MY_PACKAGE_REPLACED) return
        runCatching {
            ProactiveRuntime.ensure(context)
            ProactiveRuntime.tick(action.substringAfterLast('.'))
        }.onFailure { Log.w("DshProactive", "wake", it) }
    }
}
```

**四条设计纪律（都在注释里，也都是真的）：**
1. **只有一个闹钟**（`am.cancel(pi)` 先取消旧的）
2. **闹钟只是"去读盘比对"的提醒**——决策永远来自持久化状态，迟到/丢失/重复的闹钟不会造成错误决策
3. **降级不崩**：`setExactAndAllowWhileIdle` 抛错（厂商收回精确闹钟）→ 退回 `setAndAllowWhileIdle`
4. **强行停止后什么都收不到**（注释原文："After a force stop Android delivers none of these until DSH is opened again"）

### 11.3 `ProactiveScheduler` 的真实接口（`loop/core/ProactiveScheduler.kt`）

**公开 API（签名逐条从源码抄）：**

| 方法 | 签名 | 语义 |
|---|---|---|
| `recover` | `fun recover(now: Clocks): SchedulerState` | 进程启动：读盘、修时钟跳变、丢掉过期的 pending |
| `setConfig` | `fun setConfig(next: SchedulerConfig, now: Clocks)` | 改配置；启用状态或间隔变了就重排 |
| `evaluate` | `fun evaluate(now: Clocks): Decision` | **确定性**判定：`Decision.Fire(request)` 或 `Decision.Wait(untilWall, reason)` |
| `onSubmitted` | `fun onSubmitted(eventId: String, now: Clocks)` | 协调器收下了，等回执 |
| `onRejected` | `fun onRejected(reason: String, now: Clocks)` | 被直接拒（不计次），稍后重试 |
| `onOutcome` | `fun onOutcome(eventId: String, engagement: Engagement, now: Clocks): Boolean` | 回执；只有 pending 那条算 |
| `onUserActivity` | `fun onUserActivity(now: Clocks)` | 用户用过 DSH：清忽略计数 + 推后 |
| `nextWake` | `fun nextWake(now: Clocks, mode: WakeMode): Long?` | 该设哪个时刻的闹钟；`OPPORTUNISTIC` 时返回 null |

**`Decision` 类型**（`ProactiveScheduler.kt:108-111` 原文）：

```kotlin
sealed interface Decision {
    data class Fire(val request: EventRequest) : Decision
    data class Wait(val untilWall: Long, val reason: String) : Decision
}
```

**`Clocks` 双时钟**（`ProactiveScheduler.kt:40-41` 原文）：

```kotlin
/** Both clocks: wall for the person's day, elapsed (since boot) for intervals the wall cannot bend. */
data class Clocks(val wall: Long, val elapsed: Long, val boot: Int = 0)
```

→ 防护点：**改系统时间既不会提前也不会推迟触发**（`due` 用 elapsed，`ProactiveScheduler.kt:267`），免打扰和"今天"用 wall；日期回拨不会清零今日次数（`rollDay`，`ProactiveScheduler.kt:287-290`）。

**回执 → 下一次的完整规则**（`ProactiveScheduler.kt:198-218` 原文）：

```kotlin
/** A receipt from the coordinator. Only the pending attempt's receipt counts; repeats are ignored. */
fun onOutcome(eventId: String, engagement: Engagement, now: Clocks): Boolean = synchronized(lock) {
    if (eventId != state.pendingEventId) return@synchronized false
    rollDay(now)
    val s = state.copy(pendingEventId = null)
    val delivered = engagement != Engagement.NOT_DELIVERED
    state = when (engagement) {
        Engagement.RESPONDED, Engagement.DISMISSED -> s.copy(ignoredStreak = 0, pausedForIgnored = false, notDeliveredStreak = 0)
        Engagement.IGNORED_SEEN -> {
            val streak = s.ignoredStreak + 1
            s.copy(ignoredStreak = streak, notDeliveredStreak = 0,
                pausedForIgnored = config.pauseAfterIgnored in 1..streak)
        }
        // Posted but never confirmed seen: no penalty, no reward.
        Engagement.UNCONFIRMED -> s.copy(notDeliveredStreak = 0)
        Engagement.NOT_DELIVERED -> s.copy(notDeliveredStreak = s.notDeliveredStreak + 1)
    }.let { if (delivered) it.copy(countToday = it.countToday + 1) else it }
    if (engagement == Engagement.NOT_DELIVERED && state.notDeliveredStreak <= MAX_QUICK_RETRIES) retrySoon(now, "not_delivered")
    else plan(now, "after_${engagement.name.lowercase()}")
    persist()
    true
}
```

**`Engagement` 五态**（`loop/core/CoreModel.kt:51-55` 原文）：

```kotlin
/**
 * What the person's side of an event amounted to — the only thing the scheduler learns from.
 * A posted notification that timed out is [UNCONFIRMED]: posting is not reading.
 */
enum class Engagement { RESPONDED, DISMISSED, IGNORED_SEEN, UNCONFIRMED, NOT_DELIVERED }
```

派生（`CoreModel.kt:122-128` 原文）：

```kotlin
val engagement: Engagement get() = when {
    outcome == Outcome.RESPONDED -> Engagement.RESPONDED
    outcome == Outcome.DISMISSED -> Engagement.DISMISSED
    shownAt != null -> Engagement.IGNORED_SEEN
    deliveredAt != null -> Engagement.UNCONFIRMED
    else -> Engagement.NOT_DELIVERED
}
```

→ **"通知已发" ≠ "已读"**：`deliveredAt` 有值只算 `UNCONFIRMED`，`shownAt` 有值才算 `IGNORED_SEEN`。

**`plan()` 是唯一的随机点**（`ProactiveScheduler.kt:242-250` 原文）：

```kotlin
private fun plan(now: Clocks, reason: String) {
    val c = config
    val base = c.minIntervalMs + (rng.nextDouble() * (c.maxIntervalMs - c.minIntervalMs)).toLong()
    val factor = c.ignoredBackoff.pow(state.ignoredStreak).coerceAtMost(c.maxBackoffFactor)
    var delay = (base * factor).toLong()
    val target = now.wall + delay
    if (inQuiet(target)) delay = quietEndAfter(target) - now.wall + (rng.nextDouble() * QUIET_JITTER_MS).toLong()
    setDue(now, delay, reason)
}
```

**常量**（`ProactiveScheduler.kt:324-330` 原文）：

```kotlin
companion object {
    const val MINUTE = 60_000L
    const val DAY = 86_400_000L
    const val RECHECK_MS = 5 * MINUTE
    const val QUIET_JITTER_MS = 30 * MINUTE
    const val PENDING_GRACE_MS = 15 * MINUTE
    const val MAX_QUICK_RETRIES = 2
}
```

### 11.4 `Coordinator` 的真实接口与状态机（`loop/core/Coordinator.kt`）

**`CoordinatorConfig` 默认值**（`Coordinator.kt:3-19` 原文）：

```kotlin
data class CoordinatorConfig(
    /** Events waiting behind the active one. */
    val maxQueue: Int = 4,
    val agentTimeoutMs: Long = 45_000,
    /** A started page must report Posted/Shown within this, or it counts as not shown. */
    val presentAckMs: Long = 3_000,
    /** How long a confirmed page waits for an answer. */
    val interactionTimeoutMs: Long = 60_000,
    /** How long a posted notification waits for an answer (it may never be seen). */
    val notificationTimeoutMs: Long = 10 * 60_000,
    /** An agent reply older than this is not shown any more. */
    val replyFreshMs: Long = 2 * 60_000,
    /** How often an event may give way to DSH and come back. */
    val maxYields: Int = 2,
    val receiptCap: Int = 200,
    val seenIdCap: Int = 512,
)
```

**公开方法：**

| 方法 | 签名 |
|---|---|
| `recover` | `fun recover(gateNow: Gate): Step` |
| `setGate` | `fun setGate(next: Gate): Step` |
| `tick` | `fun tick(): Step` |
| `submit` | `fun submit(req: EventRequest): Pair<Submit, Step>` |
| `onAgent` | `fun onAgent(requestId: String, reply: AgentReply): Step` |
| `onSurface` | `fun onSurface(token: String, report: SurfaceReport): Step` |
| `cancel` | `fun cancel(eventId: String, reason: String): Step` |
| `nextDeadline` | `fun nextDeadline(): Long?` |
| `cancelSource` | `fun cancelSource(source: String, reason: String): Step` |
| 只读 | `active()` / `queued()` / `receipts(limit)` / `find(eventId)` |

**状态机（`CoreModel.kt` + `Coordinator.kt` 联合读出的真值）**

`Stage` 七态（`CoreModel.kt:18` 原文）：

```kotlin
enum class Stage { QUEUED, AWAITING_AGENT, READY, PRESENTING, DELIVERED, SHOWN, CLOSED }
```

`CoreModel.kt:8-10` 的流水线注释（原文）：

```
//   QUEUED → AWAITING_AGENT → READY → PRESENTING → SHOWN → CLOSED
//                (optional)              │  (notification: DELIVERED, never "seen")
//                                        └→ withdrawn when DSH needs the screen → READY again
```

`docs/loop/ROUND2.md:27-48` 给出的完整状态图（与 `Coordinator.advance()` 逻辑我逐条对过，一致）：

```
[*] --> QUEUED: submit（去重/合并/满了丢最弱）
QUEUED --> AWAITING_AGENT: 轮到它且此刻能展示（needsAgent）
QUEUED --> READY: 轮到它（本地文案）
QUEUED --> CLOSED: EXPIRED / CANCELLED / DROPPED / MERGED
AWAITING_AGENT --> READY: Generated（requestId 必须是当前这次）
AWAITING_AGENT --> READY: 失败或超时且有本地兜底（content=FALLBACK）
AWAITING_AGENT --> CLOSED: 失败且无兜底 FAILED / 取消 CANCELLED
READY --> PRESENTING: 闸门打开，发新令牌
READY --> CLOSED: EXPIRED（TTL 或回复放太久）
PRESENTING --> SHOWN: 页面 onResume（令牌匹配）
PRESENTING --> DELIVERED: 通知已发出（令牌匹配）
PRESENTING --> READY: 3 秒没确认：撤下，改用通知一次
PRESENTING --> CLOSED: 通知也失败 UNAVAILABLE
SHOWN --> CLOSED: 回应 RESPONDED / 关掉 DISMISSED / 超时 TIMEOUT
DELIVERED --> CLOSED: 点了 RESPONDED / 划掉 DISMISSED / 超时 TIMEOUT
PRESENTING --> READY: DSH 要用屏幕 → 撤下（yield，最多 2 次）
SHOWN --> READY: 同上
DELIVERED --> READY: 同上
READY --> CLOSED: 让路超过 2 次 YIELDED
```

**`Outcome` 十一态**（`CoreModel.kt:26-49` 原文枚举值）：
`RESPONDED, DISMISSED, TIMEOUT, CANCELLED, YIELDED, EXPIRED, MERGED, DROPPED, UNAVAILABLE, FAILED, ABANDONED`

**`Gate` —— 仲裁的全部输入**（`CoreModel.kt:135-170` 原文，完整）：

```kotlin
data class Gate(
    val featureEnabled: Boolean = true,
    /** PhoneTasks: a task is active, or a tool call is in flight. */
    val dshTaskActive: Boolean = false,
    /** PromptArbiter has a question open (approval, review, native confirm, handoff). */
    val dshPromptOpen: Boolean = false,
    /** A system permission / install / security page is in front (ProtectedPages). */
    val protectedForeground: Boolean = false,
    val screenOn: Boolean = true,
    val unlocked: Boolean = true,
    /** DSH's accessibility service is bound: the only way to start a page from the background. */
    val a11yConnected: Boolean = true,
    val notificationsAllowed: Boolean = true,
) {
    /** DSH needs the screen or the person is answering DSH: proactive surfaces must give way. */
    val dshNeedsScreen: Boolean get() = dshTaskActive || dshPromptOpen || protectedForeground

    /** Nothing new may appear now. */
    fun holdReason(): String? = when {
        !featureEnabled -> "disabled"
        dshTaskActive -> "dsh_task"
        dshPromptOpen -> "dsh_prompt"
        protectedForeground -> "protected_page"
        !screenOn -> "screen_off"
        !unlocked -> "locked"
        else -> null
    }

    /** Where a page could appear now; null when none can. */
    fun surface(): Surface? = when {
        holdReason() != null -> null
        a11yConnected -> Surface.ACTIVITY
        notificationsAllowed -> Surface.NOTIFICATION
        else -> null
    }
}
```

**这是整个系统里最重要的一段"让路"逻辑**：DSH 在操作手机 / 有审批待答 / 权限页在前 / 灭屏 / 锁屏 → 主动事件**一律不展示**（已在屏上的会被撤下回 READY）。

**`Command` 六种**（`CoreModel.kt:173-182` 原文）：

```kotlin
sealed interface Command {
    data class RequestAgent(val eventId: String, val requestId: String, val deadline: Long, val request: EventRequest) : Command
    /** After a restart: the request may still be running in the engine; ask for its result again. */
    data class ResumeAgent(val eventId: String, val requestId: String, val deadline: Long) : Command
    data class CancelAgent(val eventId: String, val requestId: String, val reason: String) : Command
    data class Present(val eventId: String, val token: String, val surface: Surface, val content: Content, val deadline: Long) : Command
    data class Withdraw(val eventId: String, val token: String, val surface: Surface, val reason: String) : Command
    /** After a restart: remove any page or notification a dead process left behind. */
    data object ClearLeftovers : Command
}
```

**令牌纪律（防"旧页面回答新事件"的核心）**（`Coordinator.kt:146-160` 原文）：

```kotlin
/** A report from a surface, matched by token only. Old pages and old buttons are refused. */
fun onSurface(token: String, report: SurfaceReport): Step = synchronized(lock) {
    ...
    val live = a != null && a.token == token && a.stage in SURFACE_STAGES
    if (!live) {
        staleReports++
        // A late page from a closed or replaced presentation must not stay on screen.
        val owner = retired[token]
        if (owner != null && (report == SurfaceReport.Posted || report == SurfaceReport.Shown)) {
            out.commands += Command.Withdraw(owner.first, token, owner.second, "stale")
        }
        return@synchronized out.step()
    }
```

**`ProactivePageActivity` 的「Shown 必须有证据」纪律**（`ProactivePageActivity.kt:26-33` 原文）：

> **`onResume` alone is not "Shown".** The coordinator treats Shown as "the person had this on screen" and feeds it into the scheduler's ignored-message backoff, so claiming it for a page nobody saw would corrupt the schedule. An activity can be resumed without being drawn — started behind the lock screen, covered at once by another activity, or on a ROM that lets it be created but never brought forward … Shown is therefore only reported once the decor view is attached, shown, laid out *and* focused

### 11.5 `ProactiveRuntime` 的胶水（`loop/proactive/ProactiveRuntime.kt`）

它做的 5 件事（`ProactiveRuntime.kt:65-72` 原文注释）：

> Owns one [Coordinator] and one [ProactiveScheduler] per process, builds the [Gate] from DSH's own state (**read only**), runs the kernel's commands, and keeps exactly one timer: a main-thread callback at the coordinator's next deadline, plus one alarm at the scheduler's next decision (none when the system forbids it).

**`Gate` 的真实取值**（`ProactiveRuntime.kt:362-379` 原文）：

```kotlin
fun gate(): Gate {
    val core = PhoneControl.peek()
    val power = app.getSystemService(PowerManager::class.java)
    val keyguard = app.getSystemService(KeyguardManager::class.java)
    val nm = app.getSystemService(NotificationManager::class.java)
    val fg = LoopRuntime.lastForeground
    return Gate(
        // Either feature keeps arbitration on; each one's events are cancelled by applySwitches.
        featureEnabled = Switches(LoopRuntime.triggerEnabled(), proactiveEnabled()).arbitrate,
        dshTaskActive = core != null && (core.tasks.isActive || core.tasks.callsInFlight > 0),
        dshPromptOpen = core?.prompts?.busy() == true,
        protectedForeground = io.github.ccch1mneyyy.dsh_app.loop.ProtectedForeground.isProtected(fg),
        screenOn = power?.isInteractive != false,
        unlocked = keyguard?.isKeyguardLocked != true,
        a11yConnected = PhoneA11yService.isConnected(),
        notificationsAllowed = nm?.areNotificationsEnabled() == true && notificationPort != null,
    )
}
```

**两个开关**（`Switches.kt:17-37` 原文）：

```kotlin
data class Switches(val triggerTest: Boolean, val proactive: Boolean) {
    val observeForeground: Boolean get() = triggerTest || proactive
    val arbitrate: Boolean get() = triggerTest || proactive

    /** Whether events from [source] may be accepted / kept. */
    fun accepts(source: String): Boolean = when (source) {
        SOURCE_APP_ENTER -> triggerTest
        SOURCE_SCHEDULER -> proactive
        else -> arbitrate
    }

    /** Sources whose events must be cancelled now. */
    fun disabledSources(): List<Pair<String, String>> = buildList {
        if (!triggerTest) add(SOURCE_APP_ENTER to "trigger_test_off")
        if (!proactive) add(SOURCE_SCHEDULER to "proactive_off")
    }

    companion object {
        const val SOURCE_APP_ENTER = "A11Y_WINDOW"
        const val SOURCE_SCHEDULER = "SCHEDULER"
    }
}
```

→ 两个开关：**触发测试**（打开某个 App 时触发）和**主动搭话**（随机定时）。关掉一个只取消它自己的事件。

**App 侧设置页**：`app/lib/ui/phone/proactive_page.dart`，入口在 `phone_control_screen.dart:359-371`（设置 → 手机操控 → 主动搭话）。
**Dart 侧通道**：`MethodChannel('dsh/proactive')` + `EventChannel('dsh/proactive/agent')`（`ProactivePlugin.kt:36-37`）。

### 11.6 保活能力的真实边界（必须承认的）

| 情形 | 会发生什么 | 证据 |
|---|---|---|
| App 在后台，无前台服务 | 进程可能被杀 → 主动事件全停 | `docs/loop/AUDIT.md:69` |
| 有 `DshService` 前台服务 | 进程活着，但 `START_NOT_STICKY`，被杀不自动重启 | 同上 |
| 系统开"限制后台活动"/ OEM 省电杀 | `WakeMode` 直接降为 `OPPORTUNISTIC`：**不设闹钟**，只在打开 DSH 或无障碍看到窗口变化时顺带检查（最多每 60 秒一次） | `ProactiveScheduler.kt:62`、`ProactiveRuntime.kt:162-168` |
| 厂商收回精确闹钟权限 | 从 `EXACT` 降为 `INEXACT`（只会晚、不会早） | `ProactiveScheduler.kt:61-65`、`WakeAlarm.kt:29-33` |
| 用户强行停止 App | **所有广播都收不到**，直到用户再次打开 DSH | `WakeAlarm.kt:42-45` |
| 手机重启 | `BOOT_COMPLETED` 重排（如果没被强行停止过） | `AndroidManifest.xml:273-280`、`WakeAlarm.kt:49` |
| 灭屏 / 锁屏 | 主动表面**一律不出** | `Gate.holdReason()`、`CoreModel.kt:153-161` |

`docs/loop/VALIDATION.md:5`（原文）："**本轮所有真机项都是 NOT_TESTED。**"

---

## 12. 附录：证据索引

### 12.1 已读的关键源码文件（全部本地读取，未改动）

| 文件 | 行数 / 大小 | 用途 |
|---|---|---|
| `sdk/plugin-kit/schema/manifest.schema.json` | 318 行 | `dshGo` 清单权威 schema |
| `sdk/plugin-kit/schema/view.schema.json` | 1192 行 | 视图 DSL 权威 schema + `x-dsh-go` 预算 |
| `sdk/plugin-kit/schema/invoke.schema.json` | 736 行 | invoke / 效果 / scope |
| `sdk/plugin-kit/schema/contributions.schema.json` | 683 行 | `contributions` 响应 + `limits` 全表 |
| `sdk/plugin-kit/types/index.d.ts` | 282 行 | **`DshGoService` 唯一权威类型** |
| `sdk/plugin-kit/types/schema.d.ts` | 1090 行 | 生成类型（Manifest/Invoke/Stream） |
| `sdk/plugin-kit/lib/card.js` | 81 行 | `toolCard()` 真实现 |
| `sdk/plugin-kit/lib/view-check.js` | 1090 行 | `BUDGETS` 三档 |
| `runtime/bridge/dsh-go-bridge/lib/native/limits.js` | 72 行 | 全部运行时上限常量 |
| `runtime/bridge/dsh-go-bridge/lib/native/core.js` | 671 行 | `ctx.dshGo` 服务实现 + Feed |
| `runtime/bridge/dsh-go-bridge/lib/native/manifest.js` | 338 行 | 清单解析与渲染 |
| `runtime/bridge/dsh-go-bridge/lib/native/routes.js` | 629 行 | 全部 HTTP 路由 |
| `runtime/bridge/dsh-go-bridge/lib/native/effects.js` | ~250 行 | 效果白名单 |
| `runtime/bridge/dsh-go-bridge/lib/native/capabilities.js` | 171 行 | v1.2 能力 + `CAP_LIMITS` |
| `runtime/bridge/dsh-go-bridge/lib/native/capability-runtime.js` | 256 行 | 租约、媒体路由、micro |
| `app/android/.../nativemedia/NativeMediaPlugin.kt` | 382 行 | 音频播放 / 麦克风 / 字幕 |
| `app/android/.../nativemedia/VoiceCapture.kt` | 139 行 | 开口检测采集 |
| `app/android/.../nativemedia/SpeechGate.kt` | 99 行 | 能量门限算法 |
| `app/android/.../nativemedia/MediaOverlays.kt` | 346 行 | App 内字幕 / 指示条 |
| `app/android/.../voice/VoicePlugin.kt` | 361 行 | App 输入框语音（插件拿不到） |
| `app/android/.../loop/core/ProactiveScheduler.kt` | 373 行 | 随机调度 |
| `app/android/.../loop/core/Coordinator.kt` | 477 行 | 仲裁状态机 |
| `app/android/.../loop/core/CoreModel.kt` | 216 行 | 数据模型 + `Gate` |
| `app/android/.../loop/core/AgentWire.kt` | 76 行 | 协议编解码 |
| `app/android/.../loop/core/Switches.kt` | 39 行 | 两个开关 |
| `app/android/.../loop/proactive/WakeAlarm.kt` | 56 行 | 单一闹钟 |
| `app/android/.../loop/proactive/ProactiveRuntime.kt` | 417 行 | Android 适配 |
| `app/android/.../loop/proactive/NotificationPortImpl.kt` | 152 行 | 通知表面 |
| `app/android/.../loop/proactive/ProactivePageActivity.kt` | 183 行 | 令牌化页面 |
| `app/android/.../loop/proactive/ProactivePlugin.kt` | 134 行 | Flutter 通道 |
| `app/android/app/src/main/AndroidManifest.xml` | 330 行 | 全部权限声明 |
| `app/lib/plugins/native/native_capabilities.dart` | 726 行 | 能力运行时（Dart） |
| `app/lib/plugins/native/native_model.dart` | 501 行 | 上限 + 模型 |
| `app/lib/plugins/native/native_surface.dart` | 263 行 | 面板/状态条读与订阅 |
| `app/lib/ui/chat/dsh_ui/dsh_ui_plugin_model.dart` | 分片 | `DshUiProfile` 三档预算 |
| `app/lib/ui/chat/tool_view.dart` | 424 行 | 工具卡读取 |
| `app/lib/ui/plugins/native/composer_strip.dart` | 311 行 | 状态条 UI |
| `app/lib/ui/plugins/native/composer_capsules.dart` | 278 行 | 胶囊 UI（8 个上限） |
| `app/lib/ui/plugins/native/message_plugin_menu.dart` | 50 行 | 消息菜单 |
| `app/lib/ui/plugins/native/native_panel.dart` | 154 行 | 面板 UI |
| `app/lib/proactive/proactive_channel.dart` | 195 行 | 主动事件 HTTP 转发 |
| `runtime/proactive/dsh-go-proactive/lib/core.js` | 284 行 | 主动回合状态机 |
| `runtime/proactive/dsh-go-proactive/lib/index.js` | 143 行 | 五条路由 |
| `examples/plugins/dsh-go-example-readaloud/*` | 全套 | 语音能力可跑示例 |

### 12.2 已读的文档（全部本地读取）

`docs/plugins/` 下全部 19 个文件：README / quickstart / from-zero / checklist / components / actions / files / hosts / inputs / local-development / native-for-free / preview / slots / styles / diagnostics / versions / voice / web-panels / design-rules（另含 `en/` 英文副本）。

设计文档：`docs/design/native-plugins-protocol.md`（704 行）、`docs/design/native-plugins-views.md`（346 行）、`docs/design/voice-input.md`（262 行）。
loop 目录：`docs/loop/ARCHITECTURE.md`、`AUDIT.md`、`ROUND2.md`、`TASKS.md`、`VALIDATION.md`。

### 12.3 明确标注为「文档说，代码未找到」的项

这份报告里**只有以下 4 条**属于此类，其余都在源码里找到了：

1. **`docs/plugins/diagnostics.md` 全篇**（插件详情页「加载状态」区的三种证据）—— 文档描述的 UI 呈现，我没有逐个去 App 的 Dart UI 里验证渲染结果。文档本身说这是"文件描述"，未涉及插件 API。
2. **`docs/plugins/preview.md` 的 `bin/preview.mjs`** —— 我列出了 `sdk/plugin-kit/bin/` 目录但没有逐行读 `preview.mjs` 的 Flutter 调用实现；文档说它跑 `flutter test --no-pub`。
3. **`docs/plugins/local-development.md` 的「重启内核以重载」入口** —— 文档描述的 App 按钮流程，我没有逐行读该文件 UI 条件判断。
4. **`dsh-go-proactive` 的 e2e 26/26 结果** —— 文档 `ROUND2.md:132` 声称在真实引擎上跑过；我读了 `test/e2e-live.mjs` 的存在（11836 B）但**没有运行它**，因此这是"文档说的"，不是"我验证过的"。

### 12.4 三条"一定要提醒"的落差

这些是任务方最容易踩的坑，都是源码级差异：

1. **`32KiB` 是面板预算；工具卡只有 `8KiB`。** 而工具卡恰恰是最通用的那一种（三种主机都能用）。如果按 32 KiB 设计卡片，上线就会看到"整张卡只剩一行『插件界面有误』"。
2. **`4Hz` 是面板/状态条数据通道；`micro` 字幕通道是 `10Hz`。** 两者不能互相套用。想推高频字幕必须走 `micro`，用 `emit` 面板数据永远只有 4 Hz。
3. **`statusbar` 不是插槽名，`composer.strip` 在 v1 里根本不读视图 JSON**（只推 `{text, icon, tone}`）。视图 schema 里那个 2 KiB / 8 节点 / 3 层的 `strip` 预算是**预留档**，现在写了也不会画。所以"给状态条写一个复杂视图"是无效功。

另外两条容易忽略的：

4. **插件图标的 162 个名字分散在两张表里**：`app/lib/plugins/native/native_icons.dart:18-148` 的 `_more` 有 119 项，`app/lib/ui_config/ui_config_icons.dart:10-53` 的 `uiConfigIcons` 有 43 项，两者**不重叠**，合起来正是 `nativeIconNames`（`native_icons.dart:152`：`List<String>.unmodifiable([...uiConfigIcons.keys, ..._more.keys])`）= 162，与文档 `docs/plugins/slots.md:51` 声明的"162 个"一致。**但如果你只 grep `native_icons.dart` 会漏掉 43 个（含 `bolt`、`chart`、`settings` 这些最常用的），误以为不支持。**
5. **工具卡不能用 `asset:` 图片，只能用 `data:`**（`docs/design/native-plugins-views.md:278`），因为对话记录里只有工具名，App 不知道它属于哪个包。

---

*报告完。所有源文件均只读，未修改 `D:\code\dshgo` 下任何文件。*
