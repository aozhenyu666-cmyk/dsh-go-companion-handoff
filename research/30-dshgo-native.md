# DSH-Go 原生接入 · 事实材料

> 研究方式：**只读源码静态分析 + DSH 安装包（app.asar）静态读取**。
> 仓库：https://github.com/ccch1mneyyy/dsh-Go
> 锁定 commit：`8fe1c1cd804ad763dd357bc1dec4b2cbfdc6f7fa`（4641 个文件，`main`）
> 桥接版本：`dsh-go-bridge` 0.4.0（`runtime/bridge/dsh-go-bridge/package.json:3`）
> App 版本：1.0.9（`app/pubspec.yaml:23` version: 1.0.9+1；`docs/changelog.md` 首条 ## 1.0.9 · 2026-11-02）
> 本机 DSH：0.2.1-rc.2（见 00-SHARED-FACTS.md）；仓库运行时锁文件指向 `@deepseek-ai/dsh` **0.2.1-rc.4**（`runtime/locks/dsh-0.2.1-rc.4/package.json`）

## 判定口径（先读这一节，全文都用这套标记）

| 标记 | 含义 |
|---|---|
| **源码已证实存在** | 我在锁定 commit 的源码/文档里读到实现，给出文件+行号。**不代表跑过** |
| **运行验证通过** | 有人真的跑起来并观察到结果，且证据在本文里给出 |
| **文档声明但未在源码中验证到实现** | 文档写了，我在源码里找不到对应实现或找不到可运行证据 |
| **NOT_TESTED** | 我没有实机/运行证据 |
| **WARN** | 有限制、有条件、或与直觉不符 |
| **FAIL** | 明确不成立 |

**本文没有一条 PASS。** 全部结论都是源码静态分析结论，没有任何一条是实机运行验证。唯一带运行证据的内容来自文档作者自述的实测（会明确标注「文档自述实测，非我实测」）。

---

# 1. 版本与兼容

## 1.1 三个版本号（加两个附带的）

`docs/plugins/versions.md` 「三个版本号」（原文，L7 起）：

> | 版本 | 在哪 | 谁管 | 现在 |
> |---|---|---|---|
> | 协议 `v` | 清单 `dshGo.v` | 主版本，不兼容才升 | `1` |
> | 视图格式 `v` | 视图根节点 `v`（可选） | 样式和组件语言，独立于 NA | `1` 与 `2` 并存；省略为 `1` |
> | App 版本 `minApp` | 清单 `dshGo.minApp` | 你写，插件依赖某个 App 版本才有的东西时 | 不写 |
>
> 另外，`package.json` 的 `engines.dsh` 写插件支持的 DSH 版本（DSH 安装时检查）。

源码侧的协议常量（硬证据）：

- `runtime/bridge/dsh-go-bridge/lib/native/limits.js:3` —— export const PROTOCOL = 1;
- `packages/dsh_client/lib/src/dsh_go_api.dart:40` —— const dshGoProtocol = 1;
- 视图版本只认 `1`/`2`，由 App 渲染器分支（`docs/plugins/versions.md` 「视图 v2：显式启用样式」）。

## 1.2 NA 小版本（不改协议主版本）

`docs/design/native-plugins-protocol.md` 头部（L7–L8）：

> **v1.1**（2026-10-25，桥接 0.3.0）：延后效果（§4.7）……
> **v1.2**（2026-10-26，桥接 0.4.0）：可选能力（§11）——有限音频、开口检测、小字幕通道、悬浮字幕。

v1.3（选择文件）见同文 §12 与 `docs/plugins/files.md`，能力名 `files`，源码 `runtime/bridge/dsh-go-bridge/lib/native/capabilities.js:7`：

```js
export const FEATURES = ["audio", "voice", "micro", "overlay", "files"];
```

**能力靠协商，不靠版本号**：插件不声明 `capabilities` 时行为与 v1.1 逐字节相同（`capabilities.js` `parseCapabilities()`，且文档 §11.1 明写）。

## 1.3 它对应的 DSH 版本

`docs/plugins/versions.md` 「和 DSH 版本的关系」（L86 起，原文）：

> | DSH | 状态 |
> |---|---|
> | 0.2.1-rc.4 | 开发基线：插槽目录（103 个）和网页岛配方按它核对 |
> | 0.2.1-rc.2 | 已验证（工具卡链路、插件安装、桥接） |
> | 0.2.x 其他 | 按 `engines.dsh` 自行声明；`presentationMeta` 自 0.2.0 起存在 |

- 仓库自带的运行时锁：`runtime/locks/dsh-0.2.1-rc.4/package.json` → "@deepseek-ai/dsh": "0.2.1-rc.4"。
- 所以 dsh-Go 1.0.9 的手机本机引擎跑的是 **DSH 0.2.1-rc.4**；本机 Windows 桌面的 DSH Desktop 是 rc.2。

**WARN**：`docs/plugins/versions.md` 说「0.2.1-rc.2 已验证」，但仓库运行时已经升到 rc.4。文档没跟着更新，属文档滞后，不是能力缺失。

## 1.4 破坏性变更说明

- v1 内「只加不改」：`docs/plugins/versions.md` 「v1 内怎么演进」原文表——老 App 遇到不认识的组件画灰占位、属性忽略、插槽忽略、效果丢弃、清单字段忽略。
- 「NA 的破坏性变化需要另升主版本。视图 v2 则是明确选择的新渲染分支：保留 v1，不改变旧视图的含义。」（同文）
- 是否存在历史破坏性变更记录：**仓库里没有**。`docs/changelog.md` 是 App 更新日志，其中「插件开发」类条目不提协议破坏。
- **结论：仓库内没有 NA 协议破坏性变更的说明文档。**

---

# 2. 插件声明（`package.json` 的 `dshGo`）

## 2.1 最重要的前提：不是必须

`docs/plugins/native-for-free.md` 原文表：

> | 插件里写了 | App 里出现在 | 原生程度 | 主机 |
> |---|---|---|---|
> | 斜杠命令（`ctx.commands`） | 输入框 `/` 菜单；执行结果显示在对话里 | 完全原生 | 全部 |
> | 技能（`SKILL.md`） | `/` 菜单的技能组、「技能」页 | 完全原生 | 全部 |
> | 插件配置（导出 `Config`，可在线修改的字段） | 插件详情页、设置里的插件分组：原生表单 | 完全原生 | 本机、SSH；中转视主机而定 |
> | 工具（`defineTool`） | 对话里的工具行：图标 + 名字 + 一行摘要，展开看参数和结果 | 通用样式 | 全部 |
> | 教模型输出 dsh-ui 代码块 | 助手消息里的图表、指标、对比卡、选项 | 完全原生 | 全部 |
> | 工具的 `presentationMeta` 里放 `"dsh-go/view"` | 工具行展开区的原生卡片 | 完全原生 | 全部 |

`docs/plugins/README.md` 原文：

> 插件照常是一个 DSH 插件（npm 包，逻辑跑在引擎里），只是多给 App 一份**声明**和一些 **JSON 视图**；App 用自己的组件画出来，**不执行插件的任何代码**。

所以「写 dshGo 清单」只在需要 **胶囊 / 消息菜单 / 状态条 / 原生面板 / 声音字幕文件能力** 时才需要。

## 2.2 清单结构与必填字段

`docs/design/native-plugins-protocol.md` §1.1 原文表（节选）：

> | 字段 | 类型 | 必填 | 规则 |
> |---|---|---|---|
> | `v` | 整数 | 是 | 协议主版本，v1 写 `1`。桥接或 App 不支持 → 插件状态 `unsupported` |
> | `minApp` | `"x.y.z"` | 否 | App 版本低于它 → 状态 `needs-app` |
> | `title` | Text，≤ 24 字 | 否 | 插件显示名（署名外框、菜单分组）。缺省用包名 |
> | `icon` | IconName | 否 | 插件图标 |
> | `wants` | `("draft" \| "clipboard" \| "message")[]` | 否 | 点按时要附带的内容。不声明就永远拿不到 |
> | `assets` | 相对路径 | 否 | 图片目录 |
> | `contributes` | 对象 | 是 | 键是 surface，值是数组 |
>
> 整个 `dshGo` 序列化后 ≤ 16 KiB，超出整份作废（状态 `invalid`）。

源码校验（`sdk/plugin-kit/lib/bridge/manifest.js` `parseManifest()`，L64 起；该文件与桥接 `lib/native/manifest.js` 逐字节同源）：

- 顶层未知键 → `unknown-field`（warn），白名单 = `TOP_KEYS`（L11）：v, minApp, title, icon, wants, assets, contributes, capabilities
- `v` 不是正整数 → `missing-field`/`invalid-field`（error）并 **return**（L66–70）
- `contributes` 不是对象 → `missing-field`/`invalid-field`（error）并 **return**（L100–103）
- **所以「必填」严格只有两个：`v` 和 `contributes`。**

## 2.3 最小可用 manifest（真实字段表）

最小合法清单（两个必填字段）：

```json
{
  "name": "my-plugin",
  "version": "0.1.0",
  "type": "module",
  "main": "lib/index.js",
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } },
  "dshGo": {
    "v": 1,
    "contributes": {}
  }
}
```

但 contributes: {} 没有任何贡献。**真实可用的最小形态**取仓库示例 `examples/plugins/dsh-go-example-usage/package.json`（原文 `dshGo` 段）：

```json
"dshGo": {
  "v": 1,
  "title": { "zh": "用量", "en": "Usage" },
  "icon": "chart",
  "contributes": {
    "composer.strip": [
      { "id": "today", "icon": "chart", "panel": "usage" }
    ],
    "panels": [
      { "id": "usage", "title": { "zh": "用量", "en": "Usage" }, "icon": "chart", "placement": ["session.tab", "features"] }
    ]
  }
}
```

顶层字段表（`docs/plugins/slots.md` `清单` + 源码 `ITEM_KEYS`）：

| 字段 | 必填 | 类型 | 约束（源码执行） |
|---|---|---|---|
| `v` | **是** | 整数 | 必须 `1`；否则 `unsupported` |
| `contributes` | **是** | 对象 | 键只能是 `composer.actions` / `message.actions` / `composer.strip` / `panels` / `media` / `micro` / `overlays`（后三者需 `capabilities`） |
| `title` | 否 | 字符串或 {locale: 字符串} | ≤ 24 字，缺省用包名 |
| `icon` | 否 | 字符串 | `^[a-z][a-z0-9-]{0,31}$`；名字表由 App 定（slots.md 列 162 个），不认识的画通用图标 |
| `minApp` | 否 | `"x.y.z"` | 严格三段 |
| `wants` | 否 | 数组 | 元素只能是 `draft`/`clipboard`/`message` |
| `assets` | 否 | 相对路径 | 必须是包内**目录**且存在，否则 `invalid-field` |
| `capabilities` | 否 | 数组 | 元素 ∈ audio voice micro overlay files；不写则 `11/`12 全部按未知忽略 |
| 整个 `dshGo` | — | — | ≤ 16 KiB（`limits.js:6` manifestBytes: 16384），超了整份作废 |

每插槽条目必填（`ITEM_KEYS` manifest.js L12–17 + 解析分支）：

| surface | 条目必填 | 可选 | 每插件上限 | 文字上限 |
|---|---|---|---|---|
| `composer.actions` | `id`、`label` | `icon`、`effect`、`requires` | 2 | 10 字 |
| `message.actions` | `id`、`label` | `icon`、`effect`、`requires` | 2 | 12 字 |
| `composer.strip` | `id` | `icon`、`text`、`panel` | 1 | 24 字 |
| `panels` | `id`、`title` | `icon`、`placement`、`view`、`micro` | 4 | 16 字 |
| `media` / `micro` / `overlays` | `id` | overlays 还有 `channel`/`text` | 各 2 | 240 字 |

`id` 全域唯一（跨插槽也不行）：`^[A-Za-z0-9][A-Za-z0-9_.-]{0,39}$`（`limits.js` `ID`）。

## 2.4 宿主半侧怎么拿到 `ctx.dshGo`

`docs/plugins/slots.md` 原文（加粗在原文里）：

> ctx.inject(['dshGo'], (ctx) => { ... }) —— **写 `['dshGo']`，不要写 `['dshGo?']`**，也不要把 `dshGo` 放进插件顶层的 `inject`：前者回调永远不执行，后者在没有桥接的主机上整个插件起不来。

原因（协议 §2.1）：

> DSH 0.2.1-rc.2 的 cordis 4.0.5 没有可选依赖语法，`'dshGo?'` 会被当成一个永远不存在的服务名，回调永不执行（总设计 3.2 的草案写法作废）。

服务注册处（源码）：`runtime/bridge/dsh-go-bridge/lib/index.js:268` —— ctx.provide("dshGo", native.service);


---

# 3. `ctx.dshGo` API 全清单

**唯一真源**：`runtime/bridge/dsh-go-bridge/lib/native/core.js` `#createService()`（L425–448）。这段是宿主真正 `provide` 出去的对象。TypeScript 类型在 `sdk/plugin-kit/types/index.d.ts` interface DshGoService。

```js
// core.js:427-448（原文，去掉换行）
const service = {
  ctx: this.ctx,
  v: PROTOCOL,
  bridge: this.version,
  panel(id, def)         { return core.#register(this?.ctx, "panels", id, def); },
  strip(id, def)         { return core.#register(this?.ctx, "strips", id, def); },
  media(id, handler)     { return core.#register(this?.ctx, "media", id, handler); },
  micro(id, def)         { return core.#register(this?.ctx, "micro", id, def); },
  onCapability(handler)  { return core.#register(this?.ctx, "capability", "events", handler); },
  on(id, handler)        { return core.#register(this?.ctx, "handlers", id, handler); },
  refresh(id, scope)     { core.#refresh(this?.ctx, id, scope); },
  readPickedFile(handle) { ... },
};
```

| 方法 | 参数 | 返回 | 说明 / 证据 |
|---|---|---|---|
| `v` | — | `1`（常量） | `core.js:428`；只读属性 |
| `bridge` | — | 桥接版本字符串 | `core.js:429`，值来自 `package.json` → `"0.4.0"`（`index.js:36`） |
| `ctx` | — | 桥接的 cordis ctx | 内部用；tracker 关联 `Symbol.for("cordis.tracker")`（`core.js` L449） |
| panel(id, def) | id: string；def: { view, data?, actions? } | dispose: () => void | `view` = 视图对象或 (scope, caps) => 视图｜Promise<视图>（**必填**，非对象非函数会 throw）；`data` = (scope, emit, caps) => cancel；actions: Record<string, handler>。校验在 `#register` L350–372 |
| strip(id, def) | `id`；`def` = 函数或 { data } | `dispose` | data(scope, emit, caps)，emit({text, icon?, tone?})；数据被 `sanitizeStrip()` 洗 |
| on(id, handler) | `id`；handler(scope, event) | `dispose` | 胶囊/菜单/状态条点按；面板没给 `actions` 时也落到这里（用 `event.action` 区分） |
| refresh(id, scope?) | 面板 id，可选 scope | `void` | 给订阅该面板且 scope 匹配的连接发 `stale`，App 重 GET 视图 |
| media(id, handler) | `id`；handler(scope, {key, locale, signal}) => {body, contentType} | `dispose` | v1.2；`body` 是 Buffer/Uint8Array 1B–16MiB；`contentType` 只能 5 种音频类型（`capabilities.js` `AUDIO_TYPES`） |
| micro(id, { data }) | `id`；{ data: (scope, emit, caps) => cancel } | `dispose` | v1.2；emit({text?, progress?})，≤ 240 字，≤ 1 KiB |
| `onCapability(handler)` | handler(scope, {instance, event, reason?, seq, signal}) | `dispose` | v1.2；返回值**永不**当效果执行（协议 §11.2） |
| `readPickedFile(handle)` | 句柄字符串 | `Uint8Array` | v1.3；只在「这一次被等待的选择回调」内有效，其余时刻 throw（`core.js:437`） |

**回调参数形状**（`docs/plugins/slots.md` 「回调参数」 + `index.d.ts`）：

| 参数 | 内容 |
|---|---|
| `scope` | { sessionId?, turnId?, messageId?, callId? }；invoke 时可能多带 `draft`/`clipboard`/`message` |
| `caps`（视图函数） | { protocol, app, locale, surfaces, components }；v1.2 起还有 `features`（`index.d.ts` interface Caps） |
| `caps`（数据函数） | { protocol, locale }（`core.js` `Feed.start()` 传入的是 {protocol: PROTOCOL, locale}） |
| `event` | { surface, id, action?, value?, locale, signal? } |
| `emit(data)` | 每次给**完整**数据；桥接算差量、限 4 Hz、限 32 KiB |

**handler 返回值的四种写法**：`undefined` / 效果对象 / 效果数组 / 简写对象（`{insert}` `{toast}` `{copy}` `{link}` `{open}` `{command}` `{refresh}` `{patch}`），多键按顺序展开为多效果。简写表见 `docs/plugins/actions.md` 「简写」，SDK 构造函数在 `sdk/plugin-kit/lib/effects.js`（insert/command/open/openPanel/link/toast/copy/refresh/patch/pickFile/pickImage/deferred）。

**注册校验（`core.js` `#register` L350–372，硬证据）**：

- loc === undefined → throw "dshGo: call ctx.dshGo.* as a method from a plugin context (ctx.inject(['dshGo'], ...))"。也就是说：**必须从插件自己的 `ctx` 上以方法形式调用**，把函数摘下来单独调用会抛错。
- `panel` 的 `view` 必须是对象或函数，否则 `TypeError`。
- `strip`/`micro` 必须给函数或 `{data}`，否则 `TypeError`。
- 注册与调用方插件生命周期绑定：callerCtx.effect(() => { add; return remove }, ...)——插件卸载/热重载自动撤销。

**「注册了清单里没声明的 id」→ `undeclared`（warn）**，按 kind 分别判定（`core.js` `#add` L376–398）：
- `capability`：需要 parsed.capabilities.length > 0
- `media`/`micro`：必须声明了同名资源
- `handlers`/`panels`/`strips`：必须在该 surface 声明过

---

# 4. 视图 / 卡片 / 胶囊 / 菜单 / 状态条 / 面板

## 4.1 视图 JSON 数据结构

**规格原文**：`docs/design/native-plugins-views.md`；机器可读真源 `sdk/plugin-kit/schema/view.schema.json`（43,732 字节）。

`docs/design/native-plugins-views.md` 1 原文：

> 每个节点是 { "type": 组件名, ...属性 }。属性名大小写敏感。
> **不认识的属性一律忽略**（不报错），方便插件同时兼容新旧版本的 App。
> 文字属性收字符串；写成数字或布尔时按文字显示（`3` → 「3」，`true` → 「是」）。数字一万以上自动加千分位。

节点结构：

```jsonc
{
  "type": "stack",           // 必填，组件名
  "title": "东京 · 未来 3 天", // 可选，只认根节点；工具卡折叠行摘要就是它
  "v": 1,                    // 可选；1 或 2（2 开启样式与输入组件）
  "children": [ ... ]
}
```

**组件目录（v1 = 16 种）**：前 10 与模型输出的 dsh-ui 卡相同，后 6 是插件视图独有（`docs/plugins/components.md` L3）：

| 组 | 组件 | 关键属性 |
|---|---|---|
| 布局/文字 | `stack` `row` `text` `callout` `steps` `compare` | `children` / `text` / `items` / `tone` |
| 数字/图表 | `metric` `chart` `progress` | `label`+`value`；`kind`+`categories`+`series`；`value`+`max` |
| 插件独有 6 | `list` `keyValue` `table` `actions` `image`（+ 已计入的 `progress`） | 见 views.md 2.2（L62 起） |
| 交互 | `options` `confirm` | `id`；点按 → invoke |

**v2 再加 26 个**（输入、结构、空状态、图集）：`docs/plugins/components.md` 「v2 样式」L7 + `docs/plugins/inputs.md` L35 起：textField number stepper toggle slider select segmented chips date time rating form tabs collapsible accordion carousel timeline grid badge tag avatar codeBlock divider spacer emptyState gallery。

**事件绑定（点按 → invoke）**，`docs/design/native-plugins-views.md` 4（原文表）：

| 元素 | invoke 的 `id` | invoke 的 `args` |
|---|---|---|
| `actions` 的按钮 | 按钮的 `action.id` | 按钮的 `action.args`（缺省 `{}`） |
| `list` 的行 | 行的 `action.id` | 行的 `action.args` |
| `options`（单选） | `options.id` | { "value": 选项的 value ?? label } |
| `options`（multi: true） | `options.id` | { "values": [...] } |
| `confirm` | `confirm.id` | { "confirmed": true ｜ false } |

动作引用两种写法："action": "refresh"（简写）或 "action": { "id": "open", "args": { "row": 3 } }；`id` 匹配 `^[A-Za-z0-9][A-Za-z0-9._:-]{0,63}$`，`args` 求值后 ≤ 1 KiB。

**数据绑定**：{ "$": "data.a.b" }，最多 16 段，`type` 不能绑定，取不到显示「—」。工具卡没有数据模型。

**视图上限**（views.md 5，与 `limits.js` 一致）：

| 项 | 工具卡 | 面板 | 状态条 |
|---|---|---|---|
| 视图大小 | 8 KiB | 32 KiB | 2 KiB |
| 组件节点 / 层数 | 40 / 6 | 120 / 8 | 8 / 3 |
| 每个 stack/row 子组件 | 16 | 32 | 4 |
| 文字 | 2000 字 | 2000 字 | 200 字 |

源码对应：limits.js:9 viewBytes: 32768、viewNodes: 120、viewDepth: 8、toolCardBytes: 8192、toolCardNodes: 40。

**渲染器在 App 侧的真实入口**：`app/lib/ui/plugins/view/plugin_view.dart`（`PluginView` widget）；工具卡走 `app/lib/ui/chat/tool_view.dart:16-20`：

```dart
DshUiParse? toolCardParse(ToolChatItem item) {
  ...
  if (meta is! Map || !meta.containsKey(toolCardViewKey)) return null;
  return parsePluginView(meta[toolCardViewKey], budget: PluginViewBudget.toolCard);
}
```

## 4.2 工具结果卡片

**这是唯一不需要桥接、任何主机（含中转）都能用的原生能力。**

`docs/plugins/quickstart.md` 原文：

> ```js
> output: {
>   schema,
>   render: (args, value) => [{ type: 'text', text: summary(value) }],   // 模型看到的
>   presentationMeta: toolCard((args, value) => weatherView(value)),    // App 画的
> },
> ```
>
> `toolCard()` 把视图放进 `{ "dsh-go/view": 视图 }`，并且兜住三件会让**整次工具调用失败**的事：
>
> | `presentationMeta` 抛错 → 工具调用失败 | 捕获，返回 `{}`（不出卡片） |
> | 返回值不是无损 JSON… | 过一遍 JSON |
> | 视图超过 8 KiB | 返回 `{}`。工具卡要写进对话记录，必须小 |

源码：`sdk/plugin-kit/lib/card.js`——VIEW_KEY = 'dsh-go/view'、CARD_MAX_BYTES = 8 * 1024、toolCard(build, {maxBytes, meta, onError})、`toLosslessJson()`、`viewOfMeta()`。

**DSH 侧接口确实存在**（我从 app.asar 里读出的官方包源码）：`dsh/node_modules/@deepseek-ai/dsh-tools/lib/types/schema.js` `defineTool()` 里显式读取并透传 `options.output.presentationMeta`：

```js
const userPresentationMeta = options.output.presentationMeta;
...
...userPresentationMeta !== undefined ? {
  presentationMeta(args, value) { return userPresentationMeta(args, value) }
} : {},
```

→ **工具卡的「引擎侧契约」已在 DSH 官方源码中证实存在**（`dsh-tools`，与本机 0.2.1-rc.2 同版本）。

工具卡的三条硬限制（components.md 「工具卡」+ quickstart.md）：
1. 不能点：按钮和 `confirm` 不显示，`options` 只读（surface: "tool.card" 在桥接里返回 400，见协议 §4.4「保留」一行）。
2. 无数据绑定（没有数据模型）。
3. 写进对话记录，只增不删字段，App 永久兼容老格式；旧对话不会补算。

## 4.3 输入框上方胶囊（`composer.actions`）

- 原生位置：输入框上方的胶囊行，和用户在 `ui.json` 里写的胶囊同一行（slots.md 表首行）。
- 每插件 2 个，标签 ≤ 10 字，胶囊总数由 App 管 ≤ 8（`app/lib/plugins/native/native_model.dart` `NativeLimits`：actionsPerPlugin = 2、capsulesTotal = 8、capsuleLabelMax = 10）。
- 点按两条路：有静态 `effect` → **App 直接执行，永不 invoke**；没有 → POST invoke → 插件 `on(id)`。
- App 侧实现：`app/lib/ui/plugins/native/composer_capsules.dart`（文件头注释原文：// The capsule row above the composer — ui.json's composer.quickActions ... and the plugins' composer.actions ... in one row）。
- `wants` 随点按附带内容：`draft`（输入框文字）/ `clipboard`（剪贴板），各 ≤ 16 KiB，只在 `composer.actions` 有效；App 在插件首次被点按前提示一次，用户拒绝就不附带（协议 §1.4）。

## 4.4 消息菜单（`message.actions`）

- 原生位置：助手消息长按菜单，**按插件分组，组名 = 插件 `title`**。
- 每插件 2 个，标签 ≤ 12 字；wants: ["message"] 附被长按那条回复的正文（≤ 16 KiB）。
- App 侧实现：`app/lib/ui/plugins/native/message_plugin_menu.dart`（`nativeMessageMenuGroups()`），注释原文：// The plugins' items in a reply's long-press menu ... one group per plugin, under its name, after the menu's own rows.
- scope 带 {sessionId, messageId}（源码里构造 DshGoScope(sessionId:..., messageId:...)）。

## 4.5 状态条（`composer.strip`）

- 原生位置：输入框上方**一行**；与活动条互斥，生成回复时让位；多插件轮流，同时 1 条（App 管）。
- 每插件 1 个，`text` ≤ 24 字。
- 数据形状固定：{ text, icon?, tone? }，`tone` ∈ good|warn|bad|info|muted；`text` 空或 null = 这条暂不显示（协议 §4.5，源码 `core.js` `sanitizeStrip()`）。
- 点按：有 `panel` → 打开那个面板；否则走 `on(id)`；都没有就不可点。
- App 侧实现：`app/lib/ui/plugins/native/composer_strip.dart`（注释原文：// A plugin's status line above the composer ... One at a time; several plugins take turns.；StripMetrics.turn = Duration(seconds: 6)）。

## 4.6 原生面板（`panels`）

- 每插件 4 个，标题 ≤ 16 字。
- 三个落点（`placement`）：

| `placement` | 位置 | scope |
|---|---|---|
| `session.tab`（缺省） | 对话详情页「插件」页签 | 带 `sessionId` |
| `features` | 全部功能 | 空 |
| `chat.panel` | 聊天页「插件」弹层 | 带 `sessionId` |

- 视图来源优先级：`panel(id).view` > 清单静态 `view` 文件（≤ 32 KiB 严格 JSON）。都没有 → 409 `no-handler`。
- 面板署名：外框顶部一行「插件图标 + 插件名」（`docs/plugins/design-rules.md` 「署名」；App 侧 `native_panel.dart` 注释 // One panel, signed: the plugin's name over its view in a glass card）。
- App 侧实现：`native_panel.dart`（`NativePanelView`），数据来自 WS 流。

## 4.7 「聊天页插件弹层」与「插件」胶囊

`docs/plugins/web-panels.md`：胶囊行最后一枚「插件」（拼图图标）打开这个对话的插件弹层；只在有内容时出现，不占 8 个名额。可在 `$DSH_HOME/dsh-go/ui.json` 的 `hide` 数组加 `pluginCapsule` 隐藏。

弹层里顺序：**先原生面板（placement: "chat.panel"，按插件名排），再网页插件放在聊天页的部件**（7 个 slot 装在一个面板岛里）。岛在打开弹层时才建，关掉 30 秒后回收。


---

# 5. 语音能力

## 5.1 三分：谁有什么

这一步最容易搞混，先摆清楚。

| 层 | 有什么 | 谁提供 | 证据 |
|---|---|---|---|
| **DSH 引擎（官方）** | **ASR（语音转文字）**：`ctx.speechToText` 服务 + `sensevoice-local` provider（SenseVoiceSmall INT8，sherpa-onnx）+ `speech/*` RPC | @deepseek-ai/dsh-experimental-{speech-to-text, speech-to-text-sensevoice, api-speech-to-text, voice-input-bundle} | 4 个包都在 app.asar 的 `dsh/node_modules/@deepseek-ai` 下（用 original-fs 枚举确认）；包自带源码与 README |
| **DSH 引擎（官方）** | **TTS（文字转语音）**：**没有** | — | `dsh-experimental-speech-to-text/README.zh.md`「已知限制」原文：**「仅支持完整录音转写。服务没有流式识别或语音合成方法。」**；我在 306 个 @deepseek-ai 包里 grep tts\|speak\|audio\|sound 只有 3 个 speech 包命中，**无 TTS 包** |
| **dsh-Go App（安卓原生）** | 麦克风采集 + **本地回声消除 + 能量门限开口检测**；系统 `SpeechRecognizer` 封装；`MediaPlayer` 播放桥接给的音频；一行字幕 + 聆听指示条 | `app/android/.../nativemedia/`、`voice/` | `MicArbiter.kt`、`SpeechGate.kt`、`VoiceCapture.kt`、`ClipPlayer.kt`、`MediaOverlays.kt`、`VoicePlugin.kt` |
| **dsh-Go App（Dart）** | 语音输入模式切换（自动/模型/系统）、模型下载、`speech/*` RPC 调用 | `app/lib/state/voice_*.dart`、`app/lib/voice/`、`packages/dsh_client/lib/src/speech_api.dart` | `speech_api.dart` 有 `speechCatalog/prepareSpeech/cancelSpeechPreparation/configureSpeech/transcribe` |
| **NA 插件（v1.2 capabilities）** | `media`（播放主机给的音频）`voice`（只听「开口/说完」事件）`micro`（字幕小通道）`overlay`（一行悬浮字幕） | 桥接 + App | 协议 §11，`capabilities.js` |

## 5.2 `docs/plugins/voice.md` 说了什么（原文摘录）

表头（L8 起）：

> | 能力 | 你得到 | 你做 | App 做 |
> |---|---|---|---|
> | `audio` | 一段声音：播、暂停、继续、停，播完有事件 | 在主机上生成音频字节（比如问主机上的 TTS） | 从桥接取回、在手机上播 |
> | `voice` | 用户开口 / 说完的事件 | 什么都不用做 | 在手机上听；一开口**先自己停掉声音**，再告诉你 |
> | `micro` | 一条小数据通道，最快每秒 10 次 | 推 { text, progress } | 画成一行字幕 |
> | `overlay` | 一行悬浮字幕 | 指定它跟哪条 `micro` 通道 | 浮在对话顶栏下，带关闭键 |

「在哪能用」（L125）：

> 本机引擎可用。SSH 主机要等桥接能装上（见[主机覆盖](hosts.md)）；中转主机没有这些能力，带 `requires` 的按钮不会出现。

App 替用户守着的（原文表）：

> | 麦克风 | 每个插件第一次要听时 App 问一次，记住答案；插件设置里有「听你开口」开关。之后才是系统录音权限… |
> | 只在前台 | 退到后台、锁屏、换对话、换主机都立刻停，不会自动恢复 |
> | 一次一个 | 全 App 同时只有一个声音会话，新的点按换下旧的 |
> | 跟着声音走 | 声音结束后 App 自动停止聆听（等用户说完）、关掉字幕 |
> | 语音输入优先 | 用户按输入框的语音输入时，你的聆听和朗读先停 |

## 5.3 插件能不能调用 TTS / ASR / 打断？

| 问题 | 答案 | 依据 |
|---|---|---|
| 插件能调 TTS 吗？ | **能，但 TTS 得插件自己在主机上做**。桥接只负责把**完整音频字节**送到手机播放，不提供 TTS 服务 | `voice.md`：`audio` 行「在主机上生成音频字节（比如问主机上的 TTS）」；`media` 只回完整一段，不支持流式 |
| 插件能调 ASR 吗？ | **NA 协议里没有 ASR**。`voice` 只给「开口/说完」两个事件，不给文字：原文「`voice` `start`/`stop`：只检测开口，App 不会把声音或文字给你」。插件要文字只能自己调主机上的服务（**注意：官方 `speech/*` 是 DSH 引擎接口，不是 NA 能力，插件可以直接用——见 5.4**） | `docs/plugins/voice.md` 「效果」；协议 §11.3；`speech_api.dart` |
| 插件能打断播放吗？ | **能，三层**：① 插件返回 {type:'audio', op:'stop'}；② 用户开口 → **App 在本地先停播**，再发 `voice.speechStart` 给插件（插件是被通知，不是被征询）；③ 用户在指示条点「停止」→ App 撤销整个租约 | 协议 §11.4 原文：「`voice.speechStart`｜voice｜用户开口。**同一实例的声音已在 App 内停掉**，所以它同时结束 audio 角色，后面不再跟 `audio.stopped`」 |
| App 已内置什么？ | 内置：**ASR 全链路（模型下载 + 引擎侧 SenseVoice + 系统 SpeechRecognizer 双路）**、麦克风采集与回声消除、开口检测、播放器、字幕展示。**不内置 TTS** | `docs/design/voice-input.md`；`speech_api.dart`；`SpeechGate.kt` |

## 5.4 官方语音套件的真实接口（我从 app.asar 读到的）

`dsh-experimental-speech-to-text/lib/types/index.js`（class SpeechToText extends Service，服务名 `'speechToText'`）：

```js
static Config = z.object({
    defaultProvider: z.string().min(1).required().volatile(),
    language: z.string().min(1).default('auto').volatile(),
});
...
register(provider)                // 注册识别器，重复 id 直接 throw
listProviders()
follow(caller)                    // 异步生成器，推完整就绪快照
snapshot()                        // { providers: [...], selection: { providerId, language } }
configure(patch)                  // 写回 profile 条目
async transcribe(spec, signal)    // 只做完整录音转写
```

`dsh-experimental-voice-input-bundle/cordis.patch.yml`（原文，说明它是「可选 bundle」）：

```yaml
- insert:
    - id: speech-to-text
      name: '@deepseek-ai/dsh-experimental-speech-to-text'
      config:
        defaultProvider: sensevoice-local
    - id: speech-to-text-sensevoice
      name: '@deepseek-ai/dsh-experimental-speech-to-text-sensevoice'
      config:
        dataRoot: !!js dshHomePath('speech-to-text', 'sensevoice')
    - id: api-speech-to-text
      name: '@deepseek-ai/dsh-experimental-api-speech-to-text'
    - id: ui-voice-input
      name: '@deepseek-ai/dsh-experimental-client-ui-voice-input'
```

`dsh-experimental-api-speech-to-text/README.zh.md` 原文：

> `catalog()` 暴露 Provider 标识、默认选择与录音限制。`transcribe()` 在解析指定 Provider 前校验规范 base64 与 16 kHz 单声道 PCM16 WAV。…音频是临时数据，不成为 Session 事件或附件；只有用户之后的普通提交才记录识别文字。

**关键判断**：这条 ASR 链路是 **DSH 官方能力，不是 dsh-Go 原生插件协议的一部分**。dsh-Go 只是「消费者」：App 从 ModelScope 预置模型文件到引擎缓存，打开 bundle，`speech/prepare`，然后 `speech/transcribe`（`docs/design/voice-input.md` 2.3–2.4、`4.1`）。

**所以语音方案放哪一侧，取决于要什么**：

- 要「用户说话 → 得到文字」：**放在 DSH 引擎侧（官方 ASR）**，插件可以直接用 `speech/*` 或 `ctx.speechToText`；不需要 NA 能力。
- 要「插件朗读一段话 + 用户一开口就停 + 字幕」：**必须用 NA v1.2 capabilities（audio/voice/micro/overlay）+ 插件自己在主机上做 TTS**。这条链不经过官方 speech 套件。

---

# 6. 媒体播放与用户打断

## 6.1 API

**声明**（清单，`docs/plugins/voice.md` `清单`）：

```jsonc
"dshGo": {
  "v": 1,
  "wants": ["message"],
  "capabilities": ["audio", "voice", "micro", "overlay"],
  "contributes": {
    "message.actions": [{ "id": "read", "label": "朗读", "icon": "play", "requires": ["audio"] }],
    "media":    [{ "id": "speech" }],
    "micro":    [{ "id": "caption" }],
    "overlays": [{ "id": "subtitle", "channel": "caption" }]
  }
}
```

**效果**（只在「点按的直接返回」里有效；静态 effect / 延后结果里出现算 `bad-effect` 违规）：

| 效果 | 字段 | 证据 |
|---|---|---|
| {type:'audio', op:'play', instance, src:'media:<id>', key?} | `key` 可选，`^[A-Za-z0-9_-]{1,128}$`，原样传给 `media` handler | 协议 §11.3 |
| {type:'audio', op:'pause'｜'resume'｜'stop', instance} | 不带 `src`/`key` | 同上 |
| {type:'voice', op:'start'｜'stop', instance} | 只检测开口，不上传任何声音或文字 | 同上 |
| {type:'overlay', op:'show'｜'hide', instance, id} | `id` 是本插件 overlay | 同上 |

**媒体路由**：GET /dsh-go/v1/media/<pkg>/<id>?lease=…&key=…（协议 §11.4；路由分发在 `runtime/bridge/dsh-go-bridge/lib/native/routes.js:609`）。全桥接同时 2 个、每插件 1 个；cache-control: no-store、`nosniff`；超时 504、取消 410、不合格 502/413。**App 整段下载到私有缓存再播放**，手机永远不直接访问插件给的地址。

**实例与租约**：同一次返回里同一 `instance` 的多个效果绑**同一个租约**（桥接加 `lease` 字段）；每连接 ≤ 8 个、每插件每连接 ≤ 2 个。

**示例（`examples/plugins/dsh-go-example-readaloud/lib/index.js` 原文）**：

```js
ctx.dshGo.on('read', (scope, event) => {
  const key = readings.begin(scope, scope.message)
  ...
  return [
    { type: 'audio', op: 'play', instance: 'reading', src: 'media:speech', key },
    { type: 'voice', op: 'start', instance: 'reading' },
    { type: 'overlay', op: 'show', instance: 'reading', id: 'subtitle' },
  ]
})
ctx.dshGo.media('speech', (scope, { key }) => ({ body: readings.clip(scope, key), contentType: 'audio/wav' }))
ctx.dshGo.micro('caption', { data: (scope, emit) => readings.watch(scope, emit) })
ctx.dshGo.onCapability((scope, event) => {
  if (event.event === 'voice.speechStart' || event.event.startsWith('audio.')) readings.end(scope)
})
```

## 6.2 事件（`onCapability`）

协议 §11.4 表 + `capabilities.js` `CAP_EVENTS`：

| event | 角色 | 含义 |
|---|---|---|
| `audio.ended` / `audio.stopped` / `audio.error` | audio | 自然播完 / 被停 / 出错 |
| `voice.speechStart` | voice | 用户开口；**声音已在 App 内停掉**，之后不再跟 `audio.stopped` |
| `voice.speechEnd` | voice | 说完（约 0.7 秒静音） |
| `voice.stopped` / `voice.error` | voice | 不再检测 / 出错（reason: "declined" / `"permission"`） |
| `overlay.closed` | overlay | 字幕关了（reason: "user" 是点了 ×） |

终态每角色一次；开口/说完这类边沿每租约最多 10 次/秒（突发 4）。**返回值永不执行**（协议 §11.2）。

## 6.3 打断的真实实现路径（源码）

1. `VoiceCapture.kt` 用 `AudioRecord`，音频源 **`MediaRecorder.AudioSource.VOICE_COMMUNICATION`**，并尝试挂 `AcousticEchoCanceler`：

   ```kotlin
   AudioRecord(..., MediaRecorder.AudioSource.VOICE_COMMUNICATION, ..., AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT, ...)
   if (runCatching { AcousticEchoCanceler.isAvailable() }.getOrDefault(false)) { aec = ...; echoCancelled = ... }
   ```

2. `SpeechGate.kt` 做能量门限：20 ms 一帧，噪声地板自适应（下降快、上升慢，播放时不升），播放时门槛从 15 dB 提高到 24 dB（有 AEC）/ 30 dB（无 AEC），开头 400 ms 的帧不算（`SETTLE_MS`）。文件头注释原文：

   > That only lowers the risk: on a loud speaker without a working echo canceller the clip can still trip it (and stop itself), and quiet speech over the clip can be missed. **Nothing here keeps or sends audio.**

3. `NativeMediaPlugin.kt` 文件头注释（原文）：

   > Speech while a clip plays: the capture thread's edge reaches the main thread, the clip is released here, and only then does `voice.speechStart` go to Dart, with localStopped: true.

→ **打断是「App 本地先停播、再通知插件」**，网络不在停止路径上。这是源码已证实的设计；**听力效果是否真准我没有实机验证 → NOT_TESTED**。

## 6.4 播放器

`ClipPlayer.kt`：`android.media.MediaPlayer`，主 looper 上单实例，`setAudioAttributes(ATTRIBUTES)` + 音频焦点请求/释放。**不支持流式**（协议明写「media 只回完整的一段」）。

---

# 7. 字幕与悬浮

## 7.1 两条字幕通道，不要混

| 通道 | 位置 | 谁能画 | 证据 |
|---|---|---|---|
| `micro` + 面板 `micro` 字段 | 面板视图**下面多一行**字幕 | 插件推数据，App 画 | 协议 §11.5；`app/lib/ui/plugins/native/native_micro_caption.dart` |
| `overlays` + {type:'overlay', op:'show'} | 一行**浮在 App 界面上**的字幕（对话顶栏下） | 插件指定跟哪条 micro 通道，App 画 | 协议 `11.3/`11.4；`MediaOverlays.kt` |

`micro` 数据模型（`capabilities.js` `microModel()`）：只能 { text?, progress? }，`text` ≤ 240 字，`progress` 0–1 或 null，序列化 ≤ 1 KiB，最多 10 Hz（100 ms 内只发最新一份）。

## 7.2 字幕能力的边界（原文）

协议 §11.6「App 里的规矩（插件改变不了）」原文表：

> | 字幕 | 一行纯文字，App 的字体和颜色，没有链接、没有输入；文字只是文字 |
> | 前台 | 只在 App 在屏幕上时有：退到后台、锁屏、换对话、换主机、断流都立刻在本地停掉，不会自动恢复 |

## 7.3 「悬浮」到底是什么：三种完全不同的东西

这是整份材料里最容易被糊成一团的地方。

| 名字 | 是什么 | 覆盖范围 | 权限 | 源码证据 |
|---|---|---|---|---|
| **NA `overlay` 能力（插件字幕）** | App **自己 Activity 里的一个 View**，加在 `android.R.id.content` 上 | **只在 dsh-Go App 内可见** | **不需要任何窗口权限** | `MediaOverlays.kt:1-40`：* What a plugin's capabilities show over the app (never over other apps)... Both are the app's own views in the activity, so they are only there while the app is on screen and need no window permission. |
| **手机操控悬浮条**（`docs/design/phone-overlay.md`） | DSH 用无障碍服务操作手机时，浮在**其他应用上面**的控制条（胶囊/卡片/小球） | 跨 App | 无障碍服务 + `TYPE_ACCESSIBILITY_OVERLAY` | `docs/design/phone-overlay.md:3`：代码在 `phone/a11y/`；同文 L127 提到 `TYPE_ACCESSIBILITY_OVERLAY` |
| **宠物 App 外悬浮窗**（`docs/design/pets.md` 2.7） | 桌宠浮在**其他应用上面** | 跨 App | `SYSTEM_ALERT_WINDOW`（`AndroidManifest.xml:34`），或借用无障碍 overlay（`OverlayAccess.kt` 四态：`ACCESSIBILITY`/`GRANTED`/`BLOCKED`/`DENIED`；窗口类型 `TYPE_APPLICATION_OVERLAY`，`OverlayAccess.kt:23`） | `OverlayAccess.kt`；`PetOverlayService.kt` |

**结论（点名回答）**：

- `docs/design/phone-overlay.md` 讲的是**手机操控**的悬浮条（`phone/a11y/`），**不是**插件字幕，**也不是**宠物；它只在 DSH 驱动手机时出现，形态是胶囊/卡片/小球三档，核心约束是「停止在任何形态下一步可达」（小球按住 1 秒停止）。
- 插件的 `overlay` **不是 Android 系统级悬浮窗**，是 App 内的 View。所以：**不能把 App 内面板等同于 Android 系统级悬浮窗，也不能把 NA `overlay` 能力当成系统悬浮窗。**
- 插件**没有任何 API** 可以在其他 App 上画东西。要跨 App 显示只能靠 App 自己的两个系统级通道（手机操控悬浮条、宠物悬浮窗），这两个**都不由插件控制**。

## 7.4 `phone-overlay.md` 受什么限制（原文要点）

- 三档形态同一物体：胶囊（默认，一行状态 + 暂停/停止）→ 卡片（点文字展开）→ 小球（贴边、可拖、闲置变淡）。
- 停止一步可达：胶囊/卡片右端永远停止；小球按住 1 秒停止（按下 0.3 s 起画进度环，满 1 s 震动并停止；拖动永远不会变成停止）。
- 「智能体注入手势时所有悬浮窗都不可触摸」——防注入。
- 视觉与动效规定极细（实心表面、1dp 发丝描边、elevation 6/5、圆角 24、340 ms emphasized 等）。
- 11「需要真机确认」：国产 ROM（vivo OriginOS 等）上 `TYPE_ACCESSIBILITY_OVERLAY` 的阴影/圆角裁切/窗口尺寸变化、小球贴边与系统边缘返回手势冲突、按住停止手感、ROM 是否拦截悬浮窗上的振动 —— **作者自己列了四项待真机确认**。

→ 这些**全部是 NOT_TESTED**（我没有任何 Android 真机/模拟器运行）。


---

# 8. 插件与 Android 本机引擎的通信路径（实际实现）

**不是 JS bridge，不是 WebView，不是 HTTP 直连原生，也不是 intent。真正路径是四段：**

```
[插件 JS，跑在 Node 里]
      │  ① 进程内方法调用（cordis 服务 ctx.dshGo.*）
      ▼
[dsh-go-bridge（同进程的 cordis 插件）]      runtime/bridge/dsh-go-bridge/lib/index.js:268  ctx.provide("dshGo", ...)
      │  ② HTTP / WebSocket，本机回环 http://127.0.0.1:<port>
      ▼
[Flutter/Dart 侧 dsh_client]                  packages/dsh_client/lib/src/dsh_go_api.dart:42-45
      │  ③ Flutter MethodChannel / EventChannel（仅 v1.2/v1.3 需要原生的部分）
      ▼
[Kotlin 原生]                                 nativemedia/NativeMediaPlugin.kt:25  "dsh/native-media"
```

**分段证据**：

| 段 | 机制 | 证据 |
|---|---|---|
| ① 插件 → 桥接 | 进程内 cordis 服务方法调用，**没有序列化、没有 IPC** | `core.js` `#createService()`；`index.js:268` ctx.provide("dshGo", native.service) |
| ② 桥接 → App | HTTP：`/dsh-go/v1/{contributions,views,invoke,media,assets,files,plugin-config}` + WS `/dsh-go/v1/stream` | `routes.js:602-614` 显式分发；`dsh_go_api.dart:42-45` 四个常量路径 |
| 传输载体 | 本机 `http://127.0.0.1:<port>`；**canReachDshGo => carrier is DshHttpCarrier** | `dsh_go_api.dart:640`；`carrier.dart` abstract class DshCarrier / class DshHttpCarrier。端口来自 `DshProcess.kt:322` val base = "http://127.0.0.1:$port" |
| ③ App → Android | Flutter MethodChannel + EventChannel | 我穷举了 `app/android/app/src/main/kotlin/**` 下全部 channel 名（见下表） |
| WebView 用在哪 | **只用于网页岛**（网页插件整页/面板）和 Mermaid 预览，**与原生插件无关** | `app/lib/ui/plugins/islands/island_view.dart:9` import 'package:webview_flutter/webview_flutter.dart';；`app/pubspec.yaml:48` `webview_flutter` |
| 原生 intent 用在哪 | 只用于系统选择器（`ACTION_OPEN_DOCUMENT` / `MediaStore.ACTION_PICK_IMAGES`）和分享/深链，**不用于插件通信** | `nativefiles/NativeFilesPlugin.kt:90` `Intent(Intent.ACTION_OPEN_DOCUMENT)`；`NativeFilesPlugin.kt:114` `MediaStore.ACTION_PICK_IMAGES` |

**我穷举出的全部 Android channel 名**（源码 `grep`，位于 `app/android/app/src/main/kotlin/io/github/ccch1mneyyy/dsh_app/*`）：

| channel | Kotlin 文件 |
|---|---|
| `dsh/native-media` + `dsh/native-media/events` | `nativemedia/NativeMediaPlugin.kt:25` |
| `dsh/native-files` | `nativefiles/NativeFilesPlugin.kt:52` |
| `dsh/voice` + `dsh/voice/events` | `voice/VoicePlugin.kt` |
| `dsh/runtime` + `dsh/runtime/events` | `runtime/RuntimePlugin.kt` |
| `dsh/phone` + `dsh/phone/events` | `phone/PhonePlugin.kt` |
| `dsh/notify` + `dsh/notify/taps` + `dsh/notify/actions` | `notify/NotifyPlugin.kt` |
| `dsh/pet_overlay` | `pet/PetOverlayPlugin.kt` |
| `dsh/islands` | `islands/IslandsPlugin.kt` |
| `dsh/files` | `files/FilesPlugin.kt` |
| `dsh/usage` `dsh/share` `dsh/links` `dsh/update` `dsh/diagnostics` `dsh/performance`(+`/events`) `dsh/theme`(`dsh/dynamic_color`) `dsh/shortcuts` `dsh/launcher_icon` `dsh/picture_shortcut` `dsh/launch` `dsh/launch_clip` `dsh/web-files` | 各自对应目录 |

**所以「插件里写的 JS 永远不跑在 Android 上」**——它跑在 Node（手机本机引擎是 proot 里的 Ubuntu + Node；或远程主机）。App 只消费 JSON + 通过 channel 拿到 Android 的播放/麦克风/选择器等**有限**能力。

**「本机引擎」到底是什么（点名回答）**：

- 是 **Android 上一个前台服务起的 proot 进程，里面跑真正的 DSH 本体（Node）**，不是 App 内嵌 JS 解释器。
- `docs/architecture.md` 原文：本地运行时 | Ubuntu 24.04 base + Node + @deepseek-ai/dsh，经 proot 运行 | App 自带，免 Root、免 Termux；原生层 | Kotlin（Flutter 插件形式，写在 app 内） | 解压运行时、管理 proot 进程、前台服务。
- 启动：原生层起 dsh web --host 127.0.0.1 --port 3080 --no-open，从 stdout 解析 token；Dart 用 token 换 cookie，之后所有 HTTP/WS 带 cookie（`docs/architecture.md` 「本地连接」）。
- `DshProcess.kt:20` 注释原文：Runs dsh web through proot or the optional direct glibc planner；`DshProcess.kt:162` 打日志 launch method: proot / `direct-glibc`。
- 可选「直接启动」（`runtime/DirectLaunch.kt`）：默认**关闭**，要在开发者选项手动打开；部分机型（Android 16、x86_64）会失败并自动回退 proot（`docs/design/runtime-cold-start.md` 0 与 `docs/guide/zh/11-faq.md` 「引擎反复启动失败」）。
- 桥接是**运行时装进本机引擎的 npm 包**：`docs/plugins/hosts.md` 本机引擎 ✅ 桥接随 App 的运行时安装。
- 冷启动实测（**文档自述实测，非我实测**）：模拟器 x86_64 上「点图标 → 对话列表」3.016 s → 优化后 2.462 s（`docs/design/runtime-cold-start.md` `2.1`）；其中「DSH 加载插件 → HTTP 监听」占 2,017 ms，是最大一段。**ARM 真机绝对值未测，文档自己说「真机 ARM 的绝对值会大得多」。**

---

# 9. 能力边界三分法

完整表格见独立文件 **`40b-capability-matrix.md`**（Lead 要求单独一份，可直接搬）。核心三分：

| 侧 | 能做什么 | 不能做什么 |
|---|---|---|
| **A. DSH 网页端插件**（跑在引擎里，纯 DSH API） | 工具、斜杠命令、技能、schemastery Config、`presentationMeta` 工具卡、dsh-ui 围栏；引擎侧任意 I/O（文件、网络、子进程）、官方 ASR（`speech/*`、`ctx.speechToText`） | 不改 App 界面；不知道手机存在；原样搬进 App 只能做成网页岛（WebView），**不是原生** |
| **B. DSH-Go 原生界面**（额外写 `dshGo` 清单 + `ctx.dshGo`） | 输入框胶囊、消息菜单、状态条、原生面板、工具卡；视图 v1 16 种 / v2 +26 种；8 种效果 + 延后效果；v1.2 音频播放/开口检测/字幕/悬浮字幕；v1.3 系统选择器读文件 | 自定义绘制/动画/字体/圆角/几何；外部图片与 SVG；链接直达；审批卡/密码框；替换官方界面；跨 App 显示；后台无限运行；调用 TTS 或 ASR（协议层面没有） |
| **C. Android 系统级**（App 自己实现，插件不可达） | 前台服务常驻引擎（`FOREGROUND_SERVICE_SPECIAL_USE` + WakeLock）、无障碍读屏与点按、跨 App 悬浮窗（`SYSTEM_ALERT_WINDOW` / `TYPE_ACCESSIBILITY_OVERLAY`）、系统通知、麦克风（`RECORD_AUDIO`）、系统 `SpeechRecognizer`、系统文件选择器、Shizuku/root/ADB 特权通道、开机广播 Job | 这些能力**没有任何一条暴露给 NA 插件**；插件只能通过 App 已实现的窄接口间接触发 |

---

# 10. 权限与后台运行限制

**结论先说：仓库明确说「可能被系统结束」，不存在「后台无限跑」的承诺。**

## 10.1 仓库里关于后台的原文（照抄）

`docs/guide/zh/11-faq.md` 「切到后台后引擎被关掉，或收不到通知」（L20 起，**原文照抄**）：

```
## 切到后台后引擎被关掉，或收不到通知

<!-- nav: commonSettings,phoneEngineKeepAlive -->
设置 › 后台运行 · [打开](dshgo://open/background)

按指南关闭电池优化、保留「引擎运行中」通知、在最近任务里锁定 App。各品牌限制不同，做完这些仍可能被系统结束。

通知还要在 App 的**通知**里开启对应提醒，并在系统设置里允许 DSH Go 通知。远程主机的提醒需要 App 在运行、连接在线。
```

关键词是最后那句：**「各品牌限制不同，做完这些仍可能被系统结束。」**

`docs/design/privileged-keep.md` 5「开机与系统限制」原文：

> 强行停止 App、设备未解锁、系统/厂商后台限制、管理器尚未启动都可能阻止恢复。

`docs/design/runtime-cold-start.md` 4 明确对比：参考项目「开机即起、常驻保活」，而 dsh-Go **「引擎也用前台服务保活，但开机不自启。是否开机自启是产品取舍（耗电、厂商后台策略）」**。

## 10.2 App 实际声明了什么（AndroidManifest.xml 硬证据）

| 能力 | 源码 | 说明 |
|---|---|---|
| `INTERNET` `ACCESS_NETWORK_STATE` | `AndroidManifest.xml` 前段 | 回环 + 远程 |
| `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_SPECIAL_USE` | L8 | 跑引擎/宠物服务 |
| `POST_NOTIFICATIONS` | L9 | 通知（Android 13+ 需用户授予） |
| `WAKE_LOCK` | L10 | `DshService.kt:157` newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "dsh-app:runtime") |
| `RECEIVE_BOOT_COMPLETED` | L5 | 但只给 `PrivilegedKeepJob` 用（手机操控通道保持），**引擎不开机自启** |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | L15 | 设置页引导用户关电池优化 |
| `RECORD_AUDIO` | L17 | 用户点麦克风时才申请 |
| `SYSTEM_ALERT_WINDOW` | L34 | **宠物 App 外悬浮窗**，默认关闭 |
| `WRITE_SECURE_SETTINGS` | L14 | 特权通道（需 root/Shizuku 等） |
| `MANAGE_EXTERNAL_STORAGE` | 中段 | 可选「访问手机文件」模式 |

**两个前台服务**（同一 manifest）：

- `DshService`（L226）：`android:foregroundServiceType="specialUse"`，PROPERTY_SPECIAL_USE_FGS_SUBTYPE = "Runs the user's local DeepSeek Harness agent server"。`DshService.kt:118` startForeground(NOTIFICATION_ID, notification, ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE)。返回 **`START_NOT_STICKY`**（源文件多处）——**系统杀掉后不会自动拉活**。
- `PetOverlayService`（L234）：同样 `specialUse`；文件头注释原文：Not sticky: without the app's Dart side there is nothing to show, so it isn't brought back after the process dies.

## 10.3 前台限制（插件侧明确写死）

协议 §11.6 原文：

> | 前台 | 只在 App 在屏幕上时有：退到后台、锁屏、换对话、换主机、断流都立刻在本地停掉，不会自动恢复 |

源码印证：`NativeMediaPlugin.kt:97` —— `Application.ActivityLifecycleCallbacks.onActivityStopped` 里直接 `revoke("background")`，注释原文：// Off the screen (another app, the lock screen): stop here and now.

WS 流也一样：`dsh_go_api.dart` 注释 It closes 60 s after the app goes to the background and comes back with every subscription's rev（协议 §4.3：App 进后台 60 秒后关，code 1000）。

## 10.4 「不能假设」的三条

1. **后台不无限跑**：`START_NOT_STICKY` + FAQ 明说「仍可能被系统结束」+ 厂商限制。
2. **插件的声音/字幕在后台一定停**：源码强制 `revoke("background")`，没有「后台继续朗读」这个能力。
3. **中转（DSH Remote）主机不支持原生插件**：`dsh_go_api.dart:640` canReachDshGo => carrier is DshHttpCarrier；`docs/plugins/hosts.md` 表：中转主机 胶囊·菜单·状态条·面板 ❌、网页岛 ❌、只有工具卡 ✅。

---

# 11. 证据清单

所有路径都相对 `ccch1mneyyy/dsh-Go@8fe1c1cd804ad763dd357bc1dec4b2cbfdc6f7fa`，除非标注 app.asar。

## 11.1 文档（我逐字读过的）

| 路径 | 关键内容 |
|---|---|
| `docs/plugins/README.md` | 一张图；四插槽表；工具表 |
| `docs/plugins/quickstart.md` | 5 分钟工具卡；`toolCard()` 兜的三件事；排错表 |
| `docs/plugins/from-zero.md` | 从空目录到市场；`engines.dsh` 写法；预览命令 |
| `docs/plugins/slots.md` | 四插槽表 + 清单 + `ctx.dshGo` + 回调参数 |
| `docs/plugins/components.md` | v1 16 / v2 +26 组件；上限表 |
| `docs/plugins/actions.md` | 8 种效果 + 简写 + 3 秒窗口 + 延后效果 |
| `docs/plugins/inputs.md` | v2 输入组件、绑定路径、表单校验、4 KiB 动态值 |
| `docs/plugins/hosts.md` | 三主机覆盖表（本机/SSH/中转）；安装与降级 |
| `docs/plugins/files.md` | v1.3 files；预算表；Android API 依据 |
| `docs/plugins/styles.md` | v2 字号/字重/spans/accent/对比度校正 |
| `docs/plugins/versions.md` | 三版本号；v1 只加不改；与 DSH 版本关系 |
| `docs/plugins/voice.md` | v1.2 四能力；事件表；App 守的规矩；回声 |
| `docs/plugins/web-panels.md` | 网页插件入口；聊天页弹层；`pluginCapsule` 隐藏 |
| `docs/plugins/native-for-free.md` | 不写 App 代码就原生显示的六样 |
| `docs/plugins/design-rules.md` | 不能自定义的清单；署名；文案 |
| `docs/plugins/checklist.md` | 发布前自查 |
| `docs/plugins/diagnostics.md` | 三类证据；不拿配置当运行 |
| `docs/plugins/preview.md` | 本地 PNG 预览；边界 |
| `docs/plugins/local-development.md` | 重载 = 重启内核；没有插件级 HMR |
| `docs/design/native-plugins.md` | 接缝盘点；路线 c；能力阶梯 L0–L4 |
| `docs/design/native-plugins-protocol.md` | 协议 v1 全文（36 KB）：清单/路由/WS/效果/上限/11 capabilities/12 files |
| `docs/design/native-plugins-views.md` | 视图 v1/v2 规格（17 KB） |
| `docs/design/plugin-ui-v3.md` | 网页插件 slot 统计；P1–P3 路线 |
| `docs/design/plugin-seams.md` | 接缝规格；网页岛配方 |
| `docs/design/pets.md` | 宠物系统（29 KB）：协议、站位、气泡、2.7 App 外悬浮窗 |
| `docs/design/phone-overlay.md` | 手机操控悬浮条（胶囊/卡片/小球） |
| `docs/design/voice-input.md` | 本机语音模型（16 KB）：现状为什么用不了、官方套件、实测数字 |
| `docs/design/rendering.md` | Markdown/公式/Mermaid 渲染取舍 |
| `docs/design/runtime-cold-start.md` | 冷启动测量；proot 占比；路线 A/B/C |
| `docs/design/privileged-keep.md` | 特权通道保持；5 系统限制 |
| `docs/architecture.md` | 技术栈；平台通道；打包；本地连接 |
| `docs/guide/zh/06-pets.md` | 宠物用户指南（含「问一句」与离线语音） |
| `docs/guide/zh/08-plugins.md` | 插件用户指南（四位置 + 桥接要求） |
| `docs/guide/zh/11-faq.md` | 后台运行原文 |
| `docs/changelog.md` | 1.0.9（2026-11-02）等版本记录 |

## 11.2 源码（我逐字读过的）

| 路径 | 用途 |
|---|---|
| `runtime/bridge/dsh-go-bridge/lib/native/core.js` | `#createService()`（L425–448，`ctx.dshGo` 真身）；`#register` L350；`Feed` 类；`sanitizeStrip` |
| `.../lib/native/limits.js` | `PROTOCOL=1`(L3)、`manifestBytes`(L6)、`viewBytes`(L9)、`handlerTimeoutMs`(L28)、`effectWindowMs`(L31)、`violationsToDegrade=5`(L42) |
| `.../lib/native/capabilities.js` | `FEATURES`(L7)、`EFFECT_FEATURES`(L9)、`CAP_LIMITS`(L19–23)、`microModel`、`capabilityEffect`、`pickerEffect` |
| `.../lib/native/manifest.js` | `parseManifest`、`renderPlugin`、`checkManifest`、`TOP_KEYS`/`ITEM_KEYS` |
| `.../lib/native/routes.js` | 路由分发 L602–614；`/media/`、`/assets/`、`/files/`、`/views/` |
| `.../lib/native/ws.js` | WS 流（读过头注释与路径常量） |
| `.../lib/index.js` | ctx.provide("dshGo", …) L268；`VERSION` L36；路由守卫 `installFallbackGuard` |
| `.../package.json` | "version": "0.4.0" L3 |
| `sdk/plugin-kit/lib/{card,caps,effects,index,check}.js` | SDK 实现 |
| `sdk/plugin-kit/lib/bridge/{manifest,capabilities,effects,limits,util}.js` | 桥接源码在 SDK 里的逐字节镜像（`scripts/sync-bridge.mjs` 维护） |
| `sdk/plugin-kit/types/index.d.ts` | interface DshGoService、`CapabilityEvent`、`MicroData`、`Caps` |
| `sdk/plugin-kit/schema/{manifest,view,invoke,stream,contributions}.schema.json` | 机器可读真源 |
| `examples/plugins/dsh-go-example-{weather,usage,quick,readaloud}/package.json` + `readaloud/lib/index.js` | 真实清单与宿主半侧写法 |
| `app/android/app/src/main/AndroidManifest.xml` | 全部权限；`DshService`(L226) `PetOverlayService`(L234) `PhoneA11yService`(L246) |
| `app/android/.../runtime/DshService.kt` | `startForeground`(L118) `PARTIAL_WAKE_LOCK`(L157) `START_NOT_STICKY` |
| `app/android/.../runtime/DshProcess.kt` | base = "http://127.0.0.1:$port"(L322)；proot/direct 启动(L162) |
| `app/android/.../nativemedia/NativeMediaPlugin.kt` | `dsh/native-media`(L25)；`revoke("background")`(L97)；`FEATURE_MICROPHONE`(L139) |
| app/android/.../nativemedia/{ClipPlayer,VoiceCapture,SpeechGate,MicArbiter,MediaOverlays}.kt | MediaPlayer / AudioRecord+VOICE_COMMUNICATION+AEC / 能量门限 / 单麦克风仲裁 / App 内字幕 View |
| `app/android/.../nativefiles/NativeFilesPlugin.kt` | `dsh/native-files`(L52)；`ACTION_OPEN_DOCUMENT`(L90)；`ACTION_PICK_IMAGES`(L114) |
| `app/android/.../voice/VoicePlugin.kt` | `dsh/voice`；`SpeechRecognizer`(L176)；`startRecognizer`/`startRecorder` |
| app/android/.../pet/{OverlayAccess,PetOverlayService,OverlayPolicy,OverlayWindowOwner}.kt | 悬浮窗权限四态；`TYPE_APPLICATION_OVERLAY`(L23)；非 sticky 前台服务；可见性状态机 |
| `packages/dsh_client/lib/src/dsh_go_api.dart` | 4 条路由常量(L42–45)；`canReachDshGo`(L640)；`dshGoFeatures`(L912) |
| `packages/dsh_client/lib/src/speech_api.dart` | `speech/catalog/prepare/cancelPreparation/configure/transcribe` |
| `packages/dsh_client/lib/src/carrier.dart` | `DshHttpCarrier`(L108) |
| app/lib/plugins/native/{native_model,native_effects,native_surface,native_capabilities,native_platform,native_file_platform}.dart | App 侧限制、效果白名单（`permission` 命令被拒，L199–202）、能力解析、channel |
| app/lib/ui/plugins/native/{composer_capsules,composer_strip,message_plugin_menu,native_panel,native_micro_caption}.dart | 四插槽的真实 widget |
| `app/lib/ui/chat/tool_view.dart` | `toolCardParse`(L16) 用 `PluginViewBudget.toolCard`(L20) |
| `app/lib/ui/plugins/islands/island_view.dart` | WebView 只用于网页岛(L9) |
| `app/lib/plugins/seams/adapters/pet_adapter.dart` | **宠物原生适配器是空壳**：static const protocols = <String>{};(L15)，`canRender` 恒 false |
| `app/pubspec.yaml` | version: 1.0.9+1(L23)；`webview_flutter`(L48) |
| `app/android/app/build.gradle.kts` | minSdk = 26(L41)；targetSdk = 28(L44) |

## 11.3 DSH 侧（app.asar 只读）

方法：用 Electron 的 `original-fs` 打头读 asar header（baseOffset = 16 + jsonSize = 4779796），按 `dsh/node_modules/@deepseek-ai/<pkg>/…` 取节点读内容。

| 事实 | 证据 |
|---|---|
| `@deepseek-ai` 下 **306 个包** | 目录枚举 |
| 语音相关只有 5 个包，**无 TTS 包** | `dsh-experimental-api-speech-to-text` / `-client-ui-voice-input` / `-speech-to-text-sensevoice` / `-speech-to-text` / `-voice-input-bundle`；grep tts\|speak\|audio\|sound 无命中 |
| ASR 服务接口 | `dsh-experimental-speech-to-text/lib/types/index.js`：class SpeechToText extends Service（`ctx.speechToText`）、`register/listProviders/follow/snapshot/configure/transcribe` |
| 「没有语音合成」原文 | `dsh-experimental-speech-to-text/README.zh.md` 已知限制：「仅支持完整录音转写。服务没有流式识别或语音合成方法。」 |
| 语音 bundle 是可选、四包一组 | `dsh-experimental-voice-input-bundle/cordis.patch.yml`（原文见 5.4） |
| `presentationMeta` 是官方契约 | `dsh-tools/lib/types/schema.js` `defineTool()` 里 const userPresentationMeta = options.output.presentationMeta; 并透传 |

---

# 12. 未验证清单（全部 NOT_TESTED）

下面每一条我都没有运行证据，只是读到了源码或文档。**不要当成 PASS。**

## 12.1 完全未验证（NOT_TESTED）

| # | 项 | 为什么没验证 |
|---|---|---|
| 1 | 任何插件在真机上装起来、胶囊/菜单/状态条/面板真的出现 | 只读研究，禁止装插件 |
| 2 | 桥接在本机引擎里真的被加载并 `provide("dshGo")` | 需要跑起引擎 + 有声明 `dshGo` 的插件 |
| 3 | `presentationMeta` 工具卡在 App 里真的画出卡片 | 需要一次真实工具调用 |
| 4 | `contributions/views/invoke/stream` 四条路由的端到端握手、ETag、补丁合并、重连补拉 | 需要真 App + 真引擎 |
| 5 | v1.2 `audio.play` → `media` 路由 → `MediaPlayer` 真的出声 | 需要真机音频 |
| 6 | `voice.speechStart` 打断是否真的准（外放/AEC/音量） | 需要真机 + 喇叭 + 人说话；源码注释自己承认可能误判 |
| 7 | `SpeechGate` 的参数（15/24/30 dB、120/200 ms、700 ms hangover、400 ms settle）在实际设备上是否合适 | 只有 `SpeechGateTest.kt` 的 JVM 单测，**不是设备测量** |
| 8 | `micro` 字幕 10 Hz、`overlay` 悬浮字幕在 App 内的实际观感 | 需要真机 |
| 9 | v1.3 系统文件选择器（`ACTION_OPEN_DOCUMENT` / `ACTION_PICK_IMAGES`）在国产 ROM 上的行为 | `docs/plugins/files.md` 自己写「系统 UI、厂商提供方、进程重建及 SSH 实机链路仍需设备验证」 |
| 10 | 手机操控悬浮条三档形态、按住停止手感、`TYPE_ACCESSIBILITY_OVERLAY` 在 vivo 上的阴影/圆角 | `docs/design/phone-overlay.md` 11 作者自己列了四项待真机确认 |
| 11 | 宠物 App 外悬浮窗在国产 ROM 上的行为 | `docs/design/pets.md`；`guide/zh/06-pets.md` 提到「开关点不动」的厂商限制 |
| 12 | 官方语音套件（SenseVoice）在 **proot arm64** 上能否加载 `sherpa-onnx.node` | `docs/design/voice-input.md` 2.1 原文：「arm64 上没有实测证据，参考项目也写着「还没在 proot arm64 上验证」」 |
| 13 | 官方语音套件在国产 ROM 上的下载（ModelScope → 引擎缓存）与内存/发热 | 文档 6「需要 vivo 真机确认」8 项 |
| 14 | 后台保活：各厂商关电池优化后引擎能活多久 | FAQ 明说「仍可能被系统结束」；**无任何量化数据** |
| 15 | `direct-glibc` 直接启动在真机上是否可用 | FAQ：默认关闭，部分机型（Android 16、x86_64）会失败回退 |
| 16 | 冷启动数字（2.462 s 等）在 ARM 真机上是否成立 | 文档明写是 **x86_64 模拟器 KVM** 的数字，「真机 ARM 的绝对值会大得多」 |
| 17 | 网页岛（WebView）内存 ~200 MB、就绪 0.8–3.4 s | `docs/design/native-plugins.md` 2.3 是「模拟器实测」，非真机 |
| 18 | 插件连续 5 次违规真的会降级 | 只读到 `core.js` `#violation` 分支（L302），没跑过降级 |

## 12.2 「文档声明但未在源码中验证到实现」

| 声明 | 出处 | 我在源码里看到什么 | 结论 |
|---|---|---|---|
| 宠物协议支持第三方宠物包 | `plugin-seams.md` L179 \| pet \| shell.overlay \| 包声明 Codex / linxin manifest \| 以后做（pets.md）\|；`plugin-ui-v3.md` L311 pets … P3 | `app/lib/plugins/seams/adapters/pet_adapter.dart`：**空壳**，static const protocols = <String>{};（L15），文件头注释「Skeleton: no pet protocol is wired to the native layer yet, so the adapter never matches」 | **文档声明但未在源码中验证到实现**（宠物包是 P3 计划，不是现成能力）。**所以「网页插件 ≠ Android 原生桌宠」要更狠一点：现在连「原生桌宠插件」都不存在。** |
| `ctx.dshGo.turnNote()` / `ctx.dshGo.notify()` | `plugin-ui-v3.md` L351 ctx.dshGo.turnNote('cost', ...)、L370 ctx.dshGo.notify('done', {...}) | `core.js` `#createService()` **没有这两个方法**；`limits.js` `SURFACES` 只有 4 个；协议 §1.2 明写「未知 surface（如第二期的 `pets`、`turn.note`）：忽略」 | **文档声明但未在源码中验证到实现**（属 P2/P3 计划） |
| panels[].presentation: "sheet" | `plugin-ui-v3.md` L305 | `ITEM_KEYS.panels` 无 `presentation`；`PLACEMENTS` 只有 3 个 | 同上，计划项 |
| `ctx.dshGoPanels.register(...)` | `plugin-seams.md` L186 | 桥接没有 `dshGoPanels` 服务（grep 无）；只有 `dshGo` | 同上 |
| 网页插件「同一份 `dshGo` 声明在网页版也显示」 | `native-plugins.md` 7 与 `plugin-ui-v3.md` 多次提及 | `hosts.md` 明写：「`dshGo` 清单和 `ctx.dshGo` 只对 App 生效，DSH 网页版看不到它们」 | **未实现**（文档两处自相矛盾，以 hosts.md 与源码为准） |

## 12.3 我明确没做的事

- 没有装/卸任何插件，没有改 `~/.dsh` 下任何文件，没有动 desktop profile，没有改 dsh-Go 仓库。
- 没有跑 `check.mjs` / `preview.mjs` / 任何 Flutter 或 Gradle 构建。
- 没有连 App、没有连引擎、没有发任何外部消息。
- 没有跑 Android 模拟器或真机。
