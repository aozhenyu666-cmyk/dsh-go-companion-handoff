# DSH-GO 大肥鱼 AI 伴侣 · 技术交接文档

> **文档版本**：1.0
> **生成时间**：2026-11-02 23:00 (+08:00)
> **交接方向**：电脑端 DSH → DSH-Go（Android 原生）
> **执行者**：DSH Lead（多 agent 协同审计）
> **纪律**：全程**只读**。未安装任何新插件、未修改生产 profile、未改动 DSH-Go 主仓库、未向 QQ / B站 等外部渠道发送任何消息。

---

## 阅读方式（先看这 5 行）

1. 本文所有结论带**证据路径**。凡标 `NOT_TESTED` 的，是**没有运行验证**的，不要当已完成。
2. 本文**没有一条 PASS 是"猜的"**。每一条 PASS 都有实测输出或源码行号。
3. 核心结论先给你：**DSH 侧的问题不是缺插件，是三处真实断链**（主会话死了 / 记忆召回率≈0 / 三个调度器互不知情）。
4. **不要照本文去装新插件**。本文的选型结论是"绝大多数新插件不该装"。
5. 需要你（用户）在手机上做的事，见 [§18](#18-需要用户在手机上完成的验证步骤)。

---

## 目录

- [0. 一句话总结](#0-一句话总结)
- [1. 项目最终目标](#1-项目最终目标)
- [2. 当前 DSH 生态清单](#2-当前-dsh-生态清单)
- [3. 各插件实际版本与测试状态](#3-各插件实际版本与测试状态)
- [4. 新插件采用与淘汰结论](#4-新插件采用与淘汰结论)
- [5. 整体系统架构](#5-整体系统架构)
- [6. 模块调用关系](#6-模块调用关系)
- [7. 人格 / 记忆 / 情绪数据结构](#7-人格--记忆--情绪数据结构)
- [8. 主动交流事件流](#8-主动交流事件流)
- [9. Agent 会话唤醒方案](#9-agent-会话唤醒方案)
- [10. DSH-Go 原生插件接入方案](#10-dsh-go-原生插件接入方案)
- [11. Android 语音交互方案](#11-android-语音交互方案)
- [12. 桌宠与视觉交互方案](#12-桌宠与视觉交互方案)
- [13. 权限与后台运行限制](#13-权限与后台运行限制)
- [14. API / 事件 / 数据协议](#14-api--事件--数据协议)
- [15. 失败处理与降级](#15-失败处理与降级)
- [16. 测试用例](#16-测试用例)
- [17. 版本兼容矩阵](#17-版本兼容矩阵)
- [18. 需要用户在手机上完成的验证步骤](#18-需要用户在手机上完成的验证步骤)
- [19. 安装与部署步骤 / 回滚步骤](#19-安装与部署步骤--回滚步骤)
- [20. 未解决的问题](#20-未解决的问题)
- [附录 A：证据索引](#附录-a证据索引)
- [附录 B：本次审计没有做的事](#附录-b本次审计没有做的事)
- [附录 C：诚实声明](#附录-c诚实声明)
- [附录 D：审计残留物](#附录-d审计残留物必须知道)
- [附录 E：审计过程中的自我纠正记录](#附录-e审计过程中的自我纠正记录)
- [附录 F：三条最容易被误传的"事实"](#附录-f三条最容易被误传的事实请勿沿用)

---

## 0. 一句话总结

**大肥鱼现在不是"功能不够"，是"三条主链路都断在同一个点上"。**

`session-429facb2-ccc8-49dc-b99d-cf44d77d8dca` —— 这个被 feiyu-mood、speak-up、qq-bridge 三方同时指着的「大肥鱼主会话」——**没有活着的 agent**。

- 情绪判定挂在它的 `agent/turn-stopping` 上 → 没有回合 → **最后一次真实情绪变化是 2026-10-30T06:12（3.5 天前）**
- 主动发言指着它 → `agents.get()` 返回 undefined → **按设计不唤醒**，一轮都不做
- QQ 桥指着它 → NapCat 本身也没起 → **整条通道不通**
- 记忆写着它 → 但**它的库文件根本不存在**（`/list` → `count:0, lastTrace:null`）；全系统 2 个库共 2 条记忆，`diag` 里 **434 次召回有 426 次「注入 0 条」**（98.2% 空转）

**所以第一优先级不是装插件，是让这个会话活着 + 修记忆召回。**

而 DSH-Go 侧的真实能力边界也跟直觉相反：

- DSH-Go 的「本机引擎」**就是在一台 Android 手机上用 proot 跑真正的 DSH** —— 大肥鱼的插件可以原样装到手机上跑，不需要移植。
- 但插件的**原生界面**只能交 JSON 组件树（16/42 种白名单组件）+ 8 种白名单效果，**不能自定义绘制、动画、字体**。
- 插件的**语音**只有四样：播一段主机给的音频、听「开口/说完」、一行字幕、一行 App 内悬浮字幕。**NA 协议既没有 TTS 也没有 ASR**。
- **第三方原生桌宠目前不存在** —— `pet_adapter.dart:15` 是空壳（`protocols = <String>{}`）。
- **后台不能假设能活** —— `DshService` 是 `START_NOT_STICKY`，开机不自启，FAQ 原文写「做完这些仍可能被系统结束」。

---

## 1. 项目最终目标

打造一个真正具有**持续记忆、独立人格、情绪变化、主动交流、语音交互、视觉陪伴**的 Android AI 伴侣。

拆成可验收的六条：

| # | 目标 | 验收标准（可测量） | 当前状态 |
|---|---|---|---|
| G1 | 持续记忆 | 一周前的对话能被自动召回进当前语境 ≥ 1 次/天 | **FAIL**（召回 100/100 全空） |
| G2 | 独立人格 | 人格只有一份真源，DSH 与 DSH-Go 读到的是同一份 | **WARN**（现有两份：surfcore persona.md + qq-bridge 配置内嵌人设） |
| G3 | 情绪变化 | 情绪能改变语气、话题选择、主动频率，且每天至少变化一次 | **FAIL**（最后一次真实变化 3.5 天前；且只有一条 systemPrompt 文字通道，源码 0 处与 speak-up 耦合） |
| G4 | 主动交流 | 小时级、不固定、由内容价值驱动；且**只有一个调度中心** | **FAIL**（当前两个开启的调度器 + 一个关闭的） |
| G5 | 语音交互 | 中文 ASR + TTS + 打断 + 字幕，真机可用 | **NOT_TESTED**（DSH 侧未装语音插件；Android 侧未测） |
| G6 | 视觉陪伴 | 大肥鱼的形象在手机上可见，且表情跟情绪走 | **FAIL**（当前没有任何一条链路把 mood 接到形象上） |

---

## 2. 当前 DSH 生态清单

### 2.1 环境基线（全部实测）

| 项 | 值 | 证据 |
|---|---|---|
| DSH Desktop 版本 | **0.2.1-rc.2** | `resources/runtime/primary-runtime/runtime.json` → `desktopVersion` |
| 构建 commit | `332895e0f7c9d4dbfb29729f5eb367fa263ff986` | `app.asar` 内 root `package.json` → `dshBuildCommit` |
| Electron | 44.7.0 | crash log `crash-2026-10-31T21-20-50-079Z-host.log` |
| Node（运行时自带） | **24.21.0** | `runtime/versions.json` |
| Node（PATH 上） | 24.19.0 | `node -v` |
| pnpm（自带） | **11.28.5** | `runtime/versions.json` |
| **pnpm / npm / npx（PATH）** | **都不存在** | `where dsh` / `npm -v` → 命令未找到 |
| **Cordis** | **4.0.5（精确）** | `app.asar → dsh/package.json` → `"@deepseek-ai/cordis": "4.0.5"` |
| @deepseek-ai/dsh-tools / dsh-llm / dsh-skill | 0.2.1-rc.2 | profile `node_modules/@deepseek-ai/*/package.json` |
| schemastery | 3.18.5 | 同上 |
| 宿主协议版本 | `hostProtocolVersion: 4` | `desktop-runtime.json` |
| 平台 | win32 x64 | — |
| DSH 用户目录 | `C:\Users\敖震宇\.dsh` | — |
| Web GUI | `http://127.0.0.1:57009`（根路径 401，要凭据） | 实测 |

> ⚠️ **工具链缺口（影响所有验收）**：本机没有 `npm` / `pnpm` / `npx` / `adb` 在 PATH。
> 唯一可用的 pnpm 在 `%USERPROFILE%\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\pnpm\bin\pnpm.cjs`。
> **后果**：`dsh-clinic` 的 R2（`npm pack`）对所有插件假红；D 组沙箱冒烟跑不了；**本次所有 Android 项一律 NOT_TESTED**。

### 2.2 Profile

| Profile | 状态 | 说明 |
|---|---|---|
| **desktop** | **生产，正在用** | 27 个依赖 / **31 个 bundle**；`cordis.patch.yml` 只覆盖 agent-default-model / ui-* / llm-pi-ai / android-emulator-helper |
| headless | 空 patch | `[]` |
| abtest-zseven | 空 patch | `[]` ← **天然适合作影子模式试验田** |

**本次审计全程未改动 desktop。**

### 2.3 已装插件（31 bundle，按性质分组）

**大肥鱼核心（自研，Yuuyuko-uu 系）**

| 包 | 版本 | 挂载 id | 仓库 |
|---|---|---|---|
| dsh-fish-memory | **1.4.3** | `memory` | Yuuyuko-uu/dsh-fish-memory |
| dsh-feiyu-mood | **0.8.1** | `mood` | Yuuyuko-uu/dsh-feiyu-mood |
| dsh-speak-up | **0.2.7** | `speak-up` | Yuuyuko-uu/dsh-speak-up |
| dsh-bili-subagent | **0.8.7** | `bili-subagent` | Yuuyuko-uu/dsh-bili-subagent |
| dsh-qq-bridge | **1.4.0** | `qq-bridge` | Yuuyuko-uu/dsh-qq-bridge |
| dsh-feiyu-app | **0.1.0** | `feiyu-app` | **本地** `C:\Users\敖震宇\feiyu\plugin`（link） |
| surfcore | **0.7.10** | `surfcore`（3 个 patch） | **本地** `~/.dsh/plugins/surfcore`（link） |

> **更正一个历史记录错误**：任务书里写的 `dsh-qq-link` **不存在**。真实包名是 `dsh-qq-bridge@1.4.0`。
> 上游 npm 上 `dsh-qq-bridge` 的 latest 是 **0.1.3** 且 repository 指向**另一个作者** `xiaomengxinbb/dsh-qq-bridge`。
> ⇒ **绝不能按 npm 包名升级**，否则会装成别人的另一套实现。本机装的是 `Yuuyuko-uu/dsh-qq-bridge`。

**桌宠 / 视觉**

| 包 | 版本 | 状态 |
|---|---|---|
| dsh-pet | **0.3.8**（`PC2005-cloud/dsh-pet`） | 加载正常；与 feiyu-mood **零桥接** |
| dsh-whale-widget | 0.4.2 | 余额挂件，与宠物视觉场景重叠 |
| dsh-plugin-wallpaper-engine | 1.4.0 | 6 个子包 |

**平台 / 基础设施（与本任务无直接关系）**
dshmarket 1.66.47、dsh-context 0.73.3、dsh-cost-meter 1.9.22、cc-safety-net 2.6.16、dsh-better-sidebar 0.28.0、
@wenaixi/dsh-superpower 8.0.0、@maci0/dsh-caveman 0.18.0、dsh-flash-presets 0.2.2、dsh-startup-screen 1.1.2、
dsh-emil-skills 1.0.0、dsh-i-have-adhd 1.0.2、@nanmicoder/dsh-agent-teams 0.1.25、
@michengai/dsh-skills-manager 1.2.9、@liustack/modsearch 5.10.10、@deepseek-ai/dsh-subagent-codex 0.2.1-rc.2

**Android / 开发工具**
@zhz8888/dsh-android-emulator-helper 1.2.0（提供本会话全部 `android_*` 工具）、@zseven-w/dsh-android 0.1.0-rc.12

**官方 bundle（8 个）**
dsh-base、dsh-web-app、**dsh-experimental-voice-input-bundle**、dsh-experimental-auto-review、
dsh-experimental-inspector-profile、dsh-experimental-tool-worktree、dsh-experimental-plugin-market-bundle、
dsh-experimental-agent-profile

> ⚠️ `dsh-experimental-voice-input-bundle` **已经装了** —— 语音选型时必须先和它对齐，否则会抢同一个槽位。

### 2.4 关键状态文件（本次审计全部实测读过）

```
~/.dsh/
├── speak-up-config.json         总开关 = false            ← 主动发言是关的
├── qq-bridge-config.json        总开关 = true, NapCat 127.0.0.1:3000
├── qq-bridge-state.json         上次私聊 1793535276
├── fish-memory-config.json      {"sessions":{},"switchMigrated":true}
├── fish-memory/<sid>.json       version 2, memories 只有 1 条, traces 100 条已满
├── fish-memory-diag.log         240KB，含 remember 可见性自检
├── feiyu-mood/config.json       7 个 slot 字段 + judgeModel deepseek-flash
├── feiyu-mood/state.json        version 2, mood=70, dims 全 0, timeline 空
├── feiyu-mood/diag.log          最后一条「心情更新」= 2026-10-30T06:12
├── dsh-pet/main-config.jsonc    pets[0]: name="蓝毛小女仆", whisperEnabled=true, display=web
├── surfcore/config.json         platforms.start.qq=false, bili=false, bridge.autoStart=false
├── surfcore/characters/char-mvuaxf54/persona.md   11803 字节（大肥鱼人格卡）
├── surfcore/characters/char-mvuaxf54/purpose-outreach.json  lastOutcome=sent, lastSentAt=2026-10-30T01:55Z
├── surfcore/pet/app/            feiyu-pet 0.2.3（Electron+PixiJS）→ pet.pid=31524 已死
└── bili-duty/                   asr.py, mcp_server.py(62KB), duty.log, seen.md（seen.md 空）
```


---

## 3. 各插件实际版本与测试状态

### 3.1 状态口径

| 标记 | 含义 |
|---|---|
| **PASS** | 实际跑过并通过，有输出/日志证据 |
| **WARN** | 有限制、部分通过、或声明与实际不符 |
| **FAIL** | 跑过且失败，或静态可证永不成立 |
| **NOT_TESTED** | 没跑过（读源码不算 PASS） |

### 3.2 大肥鱼核心链路实测

| 组件 | 版本 | 加载 | 功能 | 依据 |
|---|---|---|---|---|
| dsh-fish-memory | 1.4.3 | PASS | **写入 PASS / 落盘 PASS / 删除 PASS / 生产召回 FAIL** | 五步实测：remember 返回 `{"ok":true,"message":"记下了（今天 22:49）。"}` → 文件出现 `id=mmvzdk3h4894` → traces picked=1 → 下一轮 system prompt 出现「【你记得的事】」→ delete 返回 `{"ok":true}` 且进 `deleted` 回收站 |
| dsh-feiyu-mood | 0.8.1 | PASS | **衰减 PASS / 判定 FAIL（停更 3 天）/ 注入机制源码已读** | `GET /dsh-feiyu-mood/state` 200，mood=70，7 dims 全 0，timeline 空；`lastDecayAt` = 刚刚（衰减在跑）；`updatedAt` = 2026-11-01T12:14:39Z；最后一次「心情更新」日志 = **2026-10-30T06:12** |
| dsh-speak-up | 0.2.7 | PASS | **FAIL：当前配置永不开口** | 22:28 调 `POST /dsh-speak-up/roll {模式:"演练"}` → 日志原文「勾的目标会话现在都不能说话（d77d8dca(没活着)）—— 按设计**不自动唤醒**，这轮什么都不做」；`GET /dsh-speak-up/state` → `目标活着: false` |
| dsh-qq-bridge | 1.4.0 | PASS | **FAIL（链路）：NapCat 不通** | `GET /dsh-qq-bridge/state` 200 且 `总开关=true`；但 NapCat 探活 `{"错":"fetch failed"}`、`Test-NetConnection 127.0.0.1:3000` = False、进程表无 NapCat |
| dsh-bili-subagent | 0.8.7 | PASS | **NOT_TESTED**（`autoMode: manual`，`seen.md` 空，从未刷过） | — |
| dsh-pet（PC2005） | 0.3.8 | PASS | **状态路由 PASS** | `GET /dsh-pet-7340/state` 200，返回实时 workStatus |
| surfcore | 0.7.10 | PASS | **WARN** | `[pet-purpose]` 已启动；但 `pet-api` 令牌 401、Electron 宠物进程 pid 31524 **已死**、start.log 末行 `[chat] surfcore offline: fetch failed` ×2 |
| dsh-feiyu-app | 0.1.0 | PASS | **PASS（接口在线）** | `GET /feiyu/ping` → `{"ok":true,"版本":"0.1.0"}`；`/feiyu/sessions` 返回 11 个会话 |

### 3.3 三条链路的共同断点（**本次审计最重要的发现**）

```
session-429facb2-ccc8-49dc-b99d-cf44d77d8dca   ← 三方同时指向的「大肥鱼主会话」
        │
        ├── feiyu-mood  config.slots[0].sessions = [这个会话]
        │      → 判定钩子挂在它的 agent 上（agent/turn-stopping）
        │      → 会话没有活 agent → 没有 turn → 永不判定
        │
        ├── speak-up    目标会话 = [这个会话]
        │      → agents.get(id) 返回 undefined
        │      → 源码 L693-696 明确「按设计不自动唤醒，这轮什么都不做」
        │
        └── qq-bridge   bindings[0].会话 = 这个会话
               → NapCat 本身没起 → 更上游就断了
```

**证据**：`GET /feiyu/debug?sid=429facb2-...` 返回 `有活agent:false`。
（同一路由对 `resolveAgent` 的探测返回 `agent ... followup=function` —— 说明**会话本身可以被唤醒**，只是没人去唤。）

### 3.4 记忆召回率≈0 的根因（源码级）

`dsh-fish-memory@1.4.3 lib/index.js:2968-2979`：召回门槛卡的是 **C（证据分）**，`C <= 0` 直接 `hard_gate`。
而 C 的计算依赖**库内稀有度统计**。库里只有 1 条 → 稀有度整批算 0 → C 恒为 0 → **永远 hard_gate**。

实测：`traces` 100 条历史召回记录，`picked` **全为 0**，`omission` 全是 `hard_gate`。
唯一一条生产记忆（`mmvw95m9j274`，2026-10-31 DSH Go R4 那条设计约束）**从来没被召回进任何一轮**。

⇒ **这是要修的第一优先级，比换插件重要得多。**

### 3.5 三个「假活」陷阱（运行时验证的额外发现，必须看）

| # | 陷阱 | 表现 | 真相 |
|---|---|---|---|
| F1 | **mood 的 `state.json` mtime 一直很新** | 看起来"插件很活跃" | 插件每 3 秒 tick 衰减并重写 `lastDecayAt`，**mtime 新 ≠ 有情绪变化**。要判定活跃度得看 `diag.log` 里的「心情更新」行，或 `timeline.length` |
| F2 | **speak-up 与 feiyu-app 对同一个会话给出相反答案** | speak-up `目标情况[0].活着 = true`；feiyu-app `有活agent = false` | 两者判据不同（前者用 `agents.get` 的双前缀尝试 + 额外 state，后者只用 `agents.get`），**这个判据本身不稳**。任何依赖它的结论都要标 WARN |
| F3 | **`/feiyu/history` 对盘上有 288 个事件的会话返回 0 条** | 手机端"读不到历史" | feiyu-app 的历史是**进程内内存环形缓冲**，不是从会话日志读。进程重启即空，且只记录"它自己见过的"消息 |

### 3.6 大肥鱼主会话的记忆实况 = 死的

`session-429facb2-…`（大肥鱼生产会话）：
- `~/.dsh/fish-memory/` 下**根本没有它的库文件**（只有 `session-2c93cd4f` 和 `session-6489a146` 两个）
- `GET /dsh-fish-memory/list` → `count: 0, lastTrace: null`
- 全系统只有 **2 个库 / 共 2 条记忆**，`diag` 里 **434 次召回中有 426 次是「注入 0 条」**

⇒ **大肥鱼的记忆是彻底的 0：写入 0 / 持久化 0（文件不存在）/ 召回 0。**

### 3.7 情绪状态的一处内部矛盾（未查明）

- `state.json` 的 `timeline` 是**空数组**
- 但 `diag.log` 里明确有 **2 条「心情更新」记录**（2026-10-30T06:11:55 mood=68 / 06:12:12 mood=69）
- `state.json` 的 `lastNotedMood = 70`（基线），与那两条都不符

⇒ **两者互相矛盾，原因未查明 → 标 NOT_TESTED。** 这也解释了为什么 `updatedAt` 停在 2026-11-01T12:14。

### 3.8 PTC 模式下 remember 不可见 —— 是误报，但暴露一个真问题

`fish-memory-diag.log` 里有 `assemble 的工具（1 个）：run_code / ↑ 里面有 remember 吗：没有 ✗`。
统计 120 次快照：**103 次「有 ✓」/ 17 次「没有 ✗」**，17 次"没有"全部是"只有 1 个工具 = run_code"那类。

根因（`@deepseek-ai/dsh-tools` README + 源码）：`mode: native | ptc`。
- `native`：每个工具 schema 都进提示词 → remember 可见
- `ptc`：只给模型 `run_code` + 一份生成的 SDK → **单个工具名不出现在工具表里**

在 `run_code` 沙箱里实测 `Object.keys(tools)` = **96 个工具，含 remember**。
⇒ **工具在，只是 ptc 模式下模型看不到名字。** 但后果是真的：**ptc 模式的会话里，模型不会主动调 remember。**

**真正的 bug 是另一个**：remember 的 per-agent 那份靠 `ctx.interval(tick, 500)` 补挂，新会话首轮可能拿不到（源码注释自承）；现由插件级全局注册兜住。

---

## 4. 新插件采用与淘汰结论

**一句话：7 个待研究仓库里，建议装 0 个到生产，建议装 1 个到测试 profile。**

| 仓库 | 版本/commit | 采用结论 | 理由（一句话） |
|---|---|---|---|
| zhu1090093659/dsh-pet | 0.4.14 @ `9ed843bc` | **淘汰（不装）** | 不是本地 0.3.8 的升级，是另一条血统（`@linxin666/dsh-pet`、Apache-2.0）；`engines.dsh >= 0.2.1-rc.3` **装不进当前 rc.2**；且无 dims 情绪模型 |
| qishuilalala/dsh-voice-mode | 0.7.34 @ `3524db40` | **采用为主方案（但只在隔离 profile 验证）** | 唯一自带中文 ASR（zipformer2-zh 流式 + SenseVoice 定稿）+ 唯一真 barge-in + 唯一有字幕；engines 声明 `>=0.1.1-rc.2 <0.3.0-0` 覆盖当前版本，且 `verify-dual.sh` 的 21 版矩阵显式含 0.2.1-rc.2 |
| PerryLink/dsh-talk | 0.3.26 @ `6a5f768b` | **淘汰（当备选参考，不装）** | 默认 `stt.engine=auto` 在 Android WebView 上**必然死路**（Web Speech 在 WebView 不暴露）；无真打断、无字幕、TTS 整段不流式 |
| Aik358/dsh-auto-memory | @ `a244cf03` | **不装进生产；只在 abtest-zseven 跑影子模式** | 默认带 `autoArchiveDays:2` + `autoDeleteDays:7` + `sessionArchiveEnabled:true`，源码里真的 `rmSync` 删 `~/.dsh/sessions` —— **这是"会删我数据"级别的变更** |
| ljsysfurryACE/dsh-aura-scheduler | 0.2.0 @ `f9048cd5` | **淘汰（只吸收 3 个机制）** | 全仓零 `followup`、零 `webServer` —— **它没有投递能力**，连"说一句话"都做不到；`onProactive` 为空时唯一结果是一行日志；价值网络是装饰（`_interruptionCost()` 硬编码 0.3）；**GPL-3.0** + 定时器真泄漏 |
| PerryLink/dsh-clinic | 0.8.0 @ `bd1b3e84` | **采用（测试 profile 的验收门）** | 结构/契约/运行时/沙箱冒烟四组检查；零依赖；**不是安全认证**（grep `CVE|vulnerab|security|exfiltrat` 零命中真检查） |
| ccch1mneyyy/dsh-Go | 1.0.9 @ `8fe1c1cd` | **只读研究，不改代码** | 见 §10–§13 |

### 4.1 为什么"装 0 个"是对的

用户的核心诉求是**不要破坏已有的记忆、情绪和主动发言**。而 7 个仓库里：

- 2 个（pet / talk）在当前宿主或当前平台**装不上或用不了**
- 1 个（auto-memory）**默认会删数据**
- 1 个（aura）**根本没有投递能力**
- 1 个（voice-mode）很强但会**抢已装的 voice-input-bundle 槽位**
- 1 个（clinic）是工具不是功能
- 1 个（dsh-Go）是 App 侧

⇒ **"先盘点、验证，再实施"这条纪律直接推出的结论就是：一个都不装。**

### 4.2 aura-scheduler 值得吸收的 3 个机制（不装它）

| 机制 | aura 的位置 | 落到哪 | 价值 |
|---|---|---|---|
| **传感器触发**（file / process / http / port 状态翻转） | `src/sensors.ts` `SensorEngine.probe()`，只在 `prev.state !== current.state` 时 fire | surfcore `pet-purpose` 的 `collectMaterials()` 加一路素材源 | 让"主动说话"有真实外部事件驱动，而不是纯计时。**注意 Windows 上 `process`（pgrep）不可用**，只吸收 file / http / port |
| **off-peak 窗口** | `src/index.ts` `_inOffPeak()` + `[2,7]` | surfcore 决策前加一道 | 省钱的正确形态；surfcore 现在只有 quiet/daily/gap 三档，没有"便宜时段才开口" |
| **HMAC webhook 限流** | `src/webhook.ts` 全文（`createHmac('sha256',secret).update(name+':'+body)` + `timingSafeEqual` + 60/min + 401/429/404） | 若要给 DSH 开"外部事件唤醒"面 | 现成的正确做法。但**必须补上 `ctx.webServer.register` 那一层**——aura 缺的就是这层 |

**不吸收**：价值网络（`V = α·idleRatio + β·relevance − δ·0.3`）、`onProactive` emit 机制、class + `ctx.provide('aura')` 写法。


---

## 5. 整体系统架构

### 5.1 目标架构（六子系统 + 唯一真源）

~~~
┌──────────────────────────────────────────────────────────────────────┐
│                        人格系统（唯一真源）                            │
│  surfcore/characters/<id>/persona.md  ← 只此一份，DSH 与 DSH-Go 共读   │
│  （现在有两份：persona.md + qq-bridge 配置内嵌「人设」—— 必须收敛）      │
└───────────────┬──────────────────────────────────────────────────────┘
                │ 每轮注入
    ┌───────────┴───────────┬──────────────────┬───────────────────┐
    ▼                       ▼                  ▼                   ▼
┌─────────┐         ┌─────────────┐     ┌────────────┐     ┌─────────────┐
│ 记忆系统 │         │  情绪系统    │     │ 主动交流    │     │  信息获取    │
│fish-mem │◀───────▶│ feiyu-mood  │────▶│  调度中心   │◀────│ bili-duty   │
│1.4.3    │ 情绪影响 │ 0.8.1       │情绪 │ (唯一一个)  │候选 │bili-subagent│
│memories │ 记忆权重 │ mood+dims   │影响 │ speak-up    │话题 │surfcore B站 │
│traces   │         │ +timeline   │频率 │ 0.2.7(改造) │     │ (关闭中)     │
└────┬────┘         └──────┬──────┘     └─────┬──────┘     └──────┬──────┘
     │                     │                  │                   │
     │  systemPrompt.context 注入             │ 唯一出口：            │
     │                     │                  │ agent.followup()      │
     └─────────────────────┴──────────────────┴───────────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   交互系统         │
                    │ DSH: dsh-pet(web) │
                    │ DSH-Go: App原生宠物│
                    │ + 语音 + 字幕      │
                    └───────────────────┘
~~~

### 5.2 三层职责切分（**这条是本次交接的核心**）

| 层 | 谁做 | 具体内容 | 能不能被第三方插件调用 |
|---|---|---|---|
| **A · DSH 引擎** | 主机（电脑或手机本机引擎） | 人格 / 记忆 / 情绪 / 调度 / 采集 / Agent 回合 / 官方 ASR | — |
| **B · DSH-Go 原生界面** | App（Flutter + Kotlin） | 画 JSON 组件树、执行 8 种白名单效果、播音频、听开口、画字幕 | ✅ 通过 `ctx.dshGo` |
| **C · Android 系统级** | App（仅自己） | 前台服务、无障碍、跨 App 悬浮窗、麦克风原始流、通知、系统选择器 | ❌ **插件完全没有入口** |

### 5.3 DSH-Go 本机引擎的真实形态（决定「要不要移植」）

`docs/architecture.md` 原文：

> | 本地运行时 | Ubuntu 24.04 base + Node + `@deepseek-ai/dsh`，经 proot 运行 | App 自带，免 Root、免 Termux |
>
> 1. 原生层启动 `dsh web --host 127.0.0.1 --port 3080 --no-open`，从 stdout 解析 token。
> 2. Dart 侧 `dsh_client` 用 token 换 cookie，之后所有 HTTP / WebSocket 带 cookie，不带 `Origin`。

**⇒ 结论：手机上跑的就是真 DSH。大肥鱼的 31 个 bundle 可以原样装到手机上，不需要移植任何插件。**
**⇒ 移植压力全部集中在「界面 + 语音 + 系统权限」这三块，而不是业务逻辑。**

---

## 6. 模块调用关系

### 6.1 记忆（实测链路，源码级）

~~~
模型回合
  │
  ├─[写] agent 调 remember 工具
  │      → makeRememberTool(ctx).execute
  │      → 洗人称 / 去重 / 事实键 / 重要度
  │      → 写 ~/.dsh/fish-memory/<sid>.json 的 memories[]
  │      → 顺手生成「检索词」(related) 补进该条
  │
  └─[读] ctx.on('system-prompt/assemble')
         → 取「这一轮该拿哪句去检索」（lib/index.js:1804-1820）
         → recall(data, query, now, sid)  （:2877）
         → 打分 C(证据) P(先验) + 时间/重要度/情绪权重
         → C <= 0 → hard_gate（:2968-2979）★ 当前恒为 0
         → 命中则 agent.ctx.systemPrompt.context({name:'memory', text})
         → 注入段进 system prompt
         → traces 记录 picked/rows（最多 100 条环形）
~~~

**关键 API（给 DSH-Go 用，全部只读）**

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/dsh-fish-memory/list` | 列会话与记忆 |
| GET | `/dsh-fish-memory/page` | 分页读 |
| POST | `/dsh-fish-memory/add` | 手工加一条 |
| POST | `/dsh-fish-memory/delete` | 软删（进 deleted 回收站） |
| POST | `/dsh-fish-memory/edit` | 改 |
| POST | `/dsh-fish-memory/policy` | 读/改召回策略（**修召回率就改这里**） |
| POST | `/dsh-fish-memory/llm-check` | 测模型可用性 |

### 6.2 情绪（实测链路，源码级）

~~~
每个绑定会话的 agent
  ├─ ctx.on('agent/inbox/inserted')     → 记「她说了什么」
  ├─ ctx.on('agent/turn-stopping')      → 触发判定（异步，不阻塞对话）
  │     → 调 judgeModel（deepseek-flash）输出 JSON {moodDelta, dims, reason, confidence}
  │     → mood 只吃 1/3（大心情慢），dims 直接加
  │     → 显著才记 timeline，小波动只记账
  └─ agent.ctx.systemPrompt.context({name:'feiyu-mood', order:900, text:()=>…})
        → 每轮现查 cfg.injectMode（改设置立刻生效）
        → injectedFor === updatedAt 就返回 ''（一次变化只说一次）
        → minor / confidence < 55 / 超日上限 → 不注入
~~~

### 6.3 主动发言（**现状：三条通道，互不知情**）

~~~
① dsh-speak-up 0.2.7        开关=false             ctx.interval(10min) 掷骰 → agent.followup()  [开真回合]
② surfcore [pet-purpose]    开（lastOutcome=sent）  setInterval(60s) → 推气泡/语音              [不开回合]
③ dsh-pet 碎碎念 0.3.8      开（whisperEnabled）   每 300s 调一次 llm 生成一句话                [不开回合]
~~~

**现状**：桌宠「蓝毛小女仆」每 5 分钟用模型说一句；surfcore「肥鱼」按驱力在气泡里说一句；
而真正该说话的「大肥鱼」主会话被 speak-up 指着但开关关着、会话还是死的。**三个角色、三套节奏、一屏两个气泡源。**

---

## 7. 人格 / 记忆 / 情绪数据结构

### 7.1 人格（**当前有两份，必须收敛**）

| 来源 | 位置 | 大小 | 用途 |
|---|---|---|---|
| **surfcore 角色卡** | `~/.dsh/surfcore/characters/char-mvuaxf54/persona.md` | 11803 字节 | surfcore 桌宠 / QQ 用 |
| **qq-bridge 内嵌人设** | `~/.dsh/qq-bridge-config.json` → 「人设」字段 | ~3KB | QQ 桥每轮注入 system prompt |
| dsh-pet 宠物人设 | `~/.dsh/dsh-pet/main-config.jsonc` → `whisperPrompt` | 未设 | 桌宠碎碎念 |

**建议收敛方案**：把 persona.md 作为唯一真源；qq-bridge 的「人设」字段改为从 persona.md 读（小改或加同步脚本）；dsh-pet 的 `whisperPrompt` 设为空。

### 7.2 记忆数据结构（v2，实测样本）

```jsonc
// ~/.dsh/fish-memory/<sessionId>.json
{
  "version": 2,
  "policy": {
    "baseHalfLifeDays": 30, "emotionHalfLifeDays": 7,
    "baselineWeight": 0.55, "activationWeight": 0.25,
    "importanceWeight": 0.15, "emotionWeight": 0.05,
    "threshold": 0.3,        // ★ 召回门槛，卡的是 C（证据分）
    "maxItems": 6, "maxChars": 800, "dailyQuota": 8,
    "rarityK": 0.8,          // ★ 稀有度系数 —— 库太小会让 C 恒为 0
    "queryGate": 0.8, "factsMax": 2, "unknownActivation": 0.5
  },
  "memories": [{
    "id": "mmvw95m9j274",
    "text": "在 DSH Go R4 的主动干预系统里…",
    "at": "2026-10-31T10:22:38.839Z",
    "category": "event",          // fact | event | preference | promise
    "importance": 0.7,            // 0..1
    "emotion": null,
    "tags": ["DSH Go R4", "主动干预"],
    "source": "她自己记的",
    "事实键": "dshgor4.拦截实现方式",   // 用于「取代」旧事实
    "related": ["…检索词…"],            // 扩词结果，可为空
    "activation0": 1.0                  // 活跃度，衰减用
  }],
  "traces": [{                    // 召回轨迹，环形最多 100 条
    "at": 1793459432128,
    "session": "session-…",
    "query": "…这一轮用的检索句…",
    "gateLimit": 0.8,
    "picked": [],                 // ★ 实测全为 0 长度
    "rows": [{ "id": "…", "C": 0, "P": 0, "omission": "hard_gate" }]
  }],
  "deleted": [],                  // 软删回收站
  "purgeEpoch": 0
}
```

**实测规模**：`memories = 1`（2026-10-31 那条）、`traces = 100`（已满）、`deleted = 0`。

### 7.3 情绪数据结构（v2，实测样本）

```jsonc
// ~/.dsh/feiyu-mood/state.json
{
  "version": 2,
  "moods": {
    "__default__": {                          // slot id（稳定键，可绑多个会话）
      "mood": 70,                             // 0..100，基线 70，无底线
      "dims": {                               // 七维情绪
        "happy": 0, "calm": 0, "sweet": 0,
        "heartache": 0, "worry": 0, "sad": 0, "jealous": 0
      },
      "lastReason": "", "lastConfidence": 0,
      "injectedFor": 0, "lastInjectedText": "", "lastInjectedAt": 0,
      "saidAt": [],                           // 今天已说过的次数
      "lastNotedMood": 70, "minor": false, "kind": "",
      "updatedAt": 1793535279201,             // 2026-11-01T12:14:39Z
      "lastDecayAt": 1793629686246,           // 衰减在跑
      "timeline": []                          // ★ 实测为空
    }
  },
  "lastActive": "__default__"
}
```

**五档标签映射（源码 `lib/index.js:870-884`，Android 端可直接抄）**

```js
moodLabel(v): v>=80 欢喜 | v>=62 开心 | v>=42 一般 | v>=25 心疼 | <25 难过
moodEmoji(v): v>=80 😊 | v>=62 🙂 | v>=42 一般 | v>=1 😟 | <1 😢
```

**配套表情素材（已随插件发布）**：`assets/face-great.png` / `face-good.png` / `face-ok.png` / `face-worry.png` / `face-sad.png`

### 7.4 情绪 → 视觉的桥接（**当前完全缺失，这是 G6 的根因**）

**已证事实**：
- zhu 版 dsh-pet：`grep 'mood|feiyu|心情|情绪'` 在 `src/` → **0 命中**
- 本地 0.3.8：`inject = ['@deepseek-ai/dsh-client-connection']`，`src/` + `runtime/` grep → **0 命中**
- 但 feiyu-mood **已经开好只读路由** `GET /dsh-feiyu-mood/state?key=<slot>`

**最小桥接方案（两种，按代价排序）**

| 方案 | 改哪 | 代码量 | 局限 |
|---|---|---|---|
| **A（给 web 侧）** | zhu 版新增 `src/mood-bridge.ts` 读 state；`src/service.ts` 的 `PetStateView` 加 `mood` 字段 | ~30 行 | 只能驱动 `expressions`（phase→表情名，闭集 7 值） |
| **B（推荐，给 Android 侧）** | **完全绕开桌宠插件**，Android 端直接读 `GET /dsh-feiyu-mood/state`，自己定义 emotion→帧轨道映射 | 0（不改任何插件） | 无 —— **这是 Android 侧的正确做法** |

> **结论：Android 端不需要任何插件桥接。情绪→表情的映射在 App 侧做，数据源就是 mood 的 HTTP 路由。**

---

## 8. 主动交流事件流

### 8.1 唯一调度中心的设计（收敛后）

~~~
        ┌──────────────────────────────────────────────────────────┐
        │              唯一调度中心（dsh-speak-up 改造版）           │
        │                                                          │
        │  输入信号（全部只读）：                                   │
        │   ① 用户是否正在交流    ← ctx.agents.get(id).status       │
        │   ② 用户上次回应时间    ← session/event (user/message)    │
        │   ③ 过去的话题          ← fish-memory 的 memories+traces  │
        │   ④ 当前情绪            ← GET /dsh-feiyu-mood/state       │
        │   ⑤ 值得分享的信息      ← bili-duty / bili-subagent 候选   │
        │   ⑥ 上次主动发言时间    ← 自己记账                        │
        │   ⑦ 话题是否重复        ← 自己记账 + 记忆里的事实键         │
        │   ⑧ 打扰成本            ← 安静时段 / 峰谷 / 冷却          │
        │                                                          │
        │  决策：命中线 = 基础(峰25/谷85) × 时机因子 + 连没中加成     │
        │        (+15/次，上限 50) ± 情绪修正（建议新增）            │
        │                                                          │
        │  出口：唯一 agent.followup(message)                       │
        │        message.source.kind = 'speak-up'（心情插件按此分）  │
        └──────────────────────────────────────────────────────────┘

 ❌ 禁止：surfcore pet-purpose 自己发 / dsh-pet whisper 自己说 / 任何插件直接调 followup
 ✅ 允许：它们只提供「候选话题」，由调度中心决定是否打扰
~~~

### 8.2 现状事件流（三条并行 = 发言权分裂）

~~~
T+0    surfcore [pet-purpose] tick（每 60s）
         drives = {play:0.27, learn:0.15, install:0.38, connect:0.95, care:0.36, …}
         → 选 goal → 出文案 → 推给 Electron 桌宠气泡（不走 agent）
         → purpose-outreach.json 记 lastOutcome=sent

T+300  dsh-pet whisper（每 300s）
         → ctx.llm 让模型「随便说一句日常碎碎念，20 字以内」
         → 弹气泡（不进任何会话历史）

T+n    dsh-speak-up（每 600s，当前关闭）
         → 掷骰 → 命中则 agent.followup
         → 但目标会话没活 agent → 记账「按设计不自动唤醒」→ 什么都不做
~~~

### 8.3 收敛顺序（四步，每步带回滚）

| 步 | 动作 | 回滚 |
|---|---|---|
| **1** | 关掉纯烧钱的碎碎念：`main-config.jsonc` → `pets[0].whisperEnabled: true → false` | 改回 `true` |
| **2** | 认下唯一提示类说话人 = surfcore `[pet-purpose]`：确认 `purpose.carrier = 'pet'`，并确认 qq-bridge 那条自动 tick 确实跳过（`lib/pet-purpose.js:8-11` 说 `carrier=pet` 时 `bridge.js` 会跳过）。**这一步只读确认。** | `carrier` 改回 `'qq'` |
| **3** | speak-up 保持关（`总开关=false`），**先改它的唤醒能力**（见 §9）；改完先 `试运行=true` 观察 2–3 天 | `总开关=false`（就是当前值） |
| **4** | 从 aura 只吸收 3 个机制到 surfcore（sensors / off-peak / HMAC webhook），**不装 aura** | `git checkout` surfcore 对应文件 + 重启插件 |

### 8.4 AI 如何主动发起新话题（而不是问「今天有什么计划」）

现在 speak-up 的机制是：`触发文案` 固定一句 + `框住内心话()` 包一层「这是你自己心里冒出来的念头」。
**它只能让 AI「开口」，不能让它「有话说」。**

要解决「有话可说」，候选话题必须来自真实内容源：

| 来源 | 现有能力 | 需要什么 |
|---|---|---|
| B站 | dsh-bili-subagent 0.8.7 已装（`autoMode: manual`，从未刷过） | 打开自动扫描 + 把候选写进「候选池」 |
| 记忆 | fish-memory 里有共同经历 / 偏好 / 承诺 | 用「昨天说过的事」当话题（需要召回先修好） |
| 情绪 | feiyu-mood 的 `lastReason` | 情绪高时说开心的事，情绪低时说需要陪的事 |
| 时间 / 天气 / 日程 | **无** | 需要新增（低优先级） |

**建议的候选池协议（新增一个文件，不装任何插件）**

```jsonc
// ~/.dsh/companion/topic-candidates.json
{
  "version": 1,
  "candidates": [{
    "id": "c-20261102-001",
    "source": "bili",                    // bili | memory | mood | manual
    "title": "BV1xx411c7mD 一个关于…的视频",
    "why": "鱼片上周提过想学做饭，这个视频正好讲…",
    "value": 0.72,                       // 话题价值 0..1
    "expiresAt": 1793700000000,
    "usedAt": null
  }]
}
```

调度中心每轮挑 `value` 最高、未过期、未用过的一条作为开口理由。**这样才不会每次都说「今天有什么计划」。**

---

## 9. Agent 会话唤醒方案

### 9.1 结论（推翻一个常见假设）

**DSH 本身已经有完整的冷会话唤醒能力。DSH-Go 不需要提供「唤醒」原语。**

### 9.2 源码证据

`app.asar → dsh/node_modules/@deepseek-ai/dsh-api-session-controller/lib/index.js`

```js
// L210-212
async resolveAgent(sessionId) { return this.resolve(sessionId) }

// L221-246  resolve()
const live = this.liveAgent(sessionId);
if (live !== void 0) return live;                 // 活着 → 直接返回
...
resume = this.resume(sessionId, observation)      // 不活着 → resume

// L370-404  resume() → resumeObserved()
const observation = await this.ctx.sessionQuery.observeSession(sessionId);
return (await this.ctx.agents.resume({
  resumeSessionId: sessionId,
  agentOptions: this.agentOptions(),
  setup: composition.setup
})).agent;

// L925-950  prompt() —— 官方就是这么做的
const agent = await this.resolveAgent(request.sessionId);
...
if (request.mode === "steer") agent.steer(message);
else agent.followup(message);
```

### 9.3 实测证据（含一处必须澄清的误读）

利用本机已装的 `dsh-feiyu-app@0.1.0` 的只读诊断路由：

~~~
GET http://127.0.0.1:57009/feiyu/debug?sid=429facb2-ccc8-49dc-b99d-cf44d77d8dca
→ {"ok":true, "sid":"429facb2-…", "有活agent":false,
   "唤醒":"取不到可用 agent",
   "形状":"object :: agent | agent 键: loopCtx,id,options,session,inbox,phase,…,followup=function | agents 有: 无"}
~~~

**必须澄清**：`唤醒: "取不到可用 agent"` 这个标签是 **feiyu-app 自己的判定 bug**，不是 resolveAgent 失败。

读 `C:\Users\敖震宇\feiyu\plugin\index.mjs` 的 `/feiyu/debug` 代码：

```js
形状 = r === null ? 'null' : typeof r + ' :: ' + Object.keys(r || {}).slice(0, 12).join(',')
// → "object :: agent"   说明 r = { agent: {...} }，是包装对象

if (r && r.agent) {
  形状 += ' | agent 键: ' + Object.keys(r.agent).slice(0, 20).join(',')
  形状 += ' | followup=' + typeof r.agent.followup     // → "followup=function"
}

唤醒 = 找agent(sid) ? '唤醒后能取到 agent'
      : r && typeof r.followup === 'function' ? '返回值本身是 agent'   // ← 只查 r.followup
      : '取不到可用 agent'                                            // ← 没查 r.agent.followup
```

**真实情况**：
- `有活agent: false` → `ctx.agents.get()` 返回 undefined（会话是冷的）✅
- `resolveAgent()` 返回值 `r = { agent: … }`，且 `typeof r.agent.followup === "function"` ✅
- 但 `唤醒` 那一行只检查了 `r.followup`（包装对象上没有），于是打了「取不到可用 agent」

⇒ **结论仍然是：冷会话被 resolveAgent 成功解析成了可用 agent（只是 feiyu-app 的诊断标签写错了）。**

⚠️ **但必须同时承认**：`~/.dsh/fish-memory-diag.log` 与 `/dsh-speak-up/state` 里，
对同一个 `session-429facb2-…`，speak-up 报「目标情况[0].活着 = true」而 feiyu-app 报「有活agent = false」。
**两个插件对「是否活着」给出相反答案**，说明这个判据本身不稳。
→ 因此本节的结论**只能算「源码支持 + 单次探测支持」，不能算「稳定可依赖」**。标 **WARN**，不是 PASS。
→ DSH-Go 侧若依赖这一点，必须自己再验一遍。

### 9.4 所以 speak-up 的根本缺陷是「实现选择」，不是「能力缺失」

```js
// dsh-speak-up@0.2.7 lib/host.js L460-475
function 找agent(ctx, id) {
  const 候选 = [原]; if (原.startsWith('session-')) 候选.push(原.slice(8)); else 候选.push('session-' + 原)
  for (const 一个 of 候选) {
    const a = ctx.agents.get(一个)      // ★ 只查活着的，冷会话 → undefined
    if (a !== undefined) return a
  }
  return undefined
}

// L676-697
for (const id of 配置.目标会话) {
  const a = 找agent(ctx, id)
  if (a === undefined) { 不可用.push(短id(id) + '(没活着)'); continue }
  ...
}
if (可用.length === 0) {
  return 记账('勾的目标会话现在都不能说话（…）—— 按设计**不自动唤醒**，这轮什么都不做')
}
```

### 9.5 修复方案（约 20 行，需用户授权）

```js
// 把 找agent 改为异步，并在不可用时尝试 resolveAgent
async function 找agent_或唤醒(ctx, id) {
  const 候选 = [String(id).trim()]
  候选.push(候选[0].startsWith('session-') ? 候选[0].slice(8) : 'session-' + 候选[0])

  // ① 先试活着的（原逻辑，零成本）
  for (const 一个 of 候选) {
    try { const a = ctx.agents.get(一个); if (a !== undefined) return a } catch {}
  }

  // ② 冷会话：交给 sessionController 唤醒
  const sc = (() => { try { return ctx.get('sessionController') } catch { return undefined } })()
  if (!sc || typeof sc.resolveAgent !== 'function') return undefined
  for (const 一个 of 候选) {
    let r = null
    try { r = await sc.resolveAgent(一个) } catch { continue }
    if (r && r.error) continue                       // not-found / writer-held
    const a2 = (() => { try { return ctx.agents.get(一个) } catch { return undefined } })()
    if (a2 !== undefined) return a2
    if (r && typeof r.followup === 'function') return r
    if (r && r.agent && typeof r.agent.followup === 'function') return r.agent
  }
  return undefined
}
```

**必须保留的原判定**：`状态 !== undefined && 状态 !== 'idle'` → 记为「忙」，不插话。

**风险**：
1. 唤醒冷会话会真的开一轮，等于花一次模型调用。所以必须配合 `搭话冷却分钟` 与 `试运行` 先用。
2. 会话被别的写者持有时（`session/writer-held`），`resolveAgent` 返回 error，按「没活着」处理即可。

### 9.6 DSH-Go 需要提供什么

**不需要提供「唤醒」。** 但需要知道这几件事：

| 需求 | 谁负责 | 说明 |
|---|---|---|
| 目标会话在手机上也要能醒 | DSH-Go 本机引擎 | 只要会话存在且没被写者持有，`resolveAgent` 就能醒 |
| 手机端触发主动回合 | 见 §8 调度中心 | 走 speak-up 的 `/dsh-speak-up/roll` 或等它自己定时 |
| **主动回合里禁止工具** | **DSH-Go 已实现** | `dsh-go-proactive` 有 `tools/pre-execute` 硬拒 |
| 手机端提醒（通知） | DSH-Go | `ProactiveScheduler` + `WakeAlarm`（单闹钟，精确→非精确降级） |

---

## 10. DSH-Go 原生插件接入方案

### 10.1 通信路径（真实实现，不是 JS bridge 也不是 WebView）

~~~
插件 JS（跑在 Node）
   ↓ 同进程 cordis 服务
ctx.dshGo   （runtime/bridge/dsh-go-bridge/lib/native/core.js:425-448 的 #createService）
   ↓ HTTP / WS 到 127.0.0.1:<port>（dsh_go_api.dart:42-45, 640）
Flutter（packages/dsh_client）
   ↓ MethodChannel
Kotlin（NativeMediaPlugin.kt:25 等）
   ↓
Android（MediaPlayer / AudioRecord / View）
~~~

**App 只消费 JSON 视图 + 8 种白名单效果，从不执行插件代码。**
（`docs/plugins/README.md:12` 原文：「插件照常是一个 DSH 插件（npm 包，逻辑跑在引擎里），只是多给 App 一份**声明**和一些 **JSON 视图**；App 用自己的组件画出来，**不执行插件的任何代码**。」）

### 10.2 不写 App 代码就能原生显示的六样（`docs/plugins/native-for-free.md`）

| 插件里写了 | App 里出现在 | 原生程度 |
|---|---|---|
| 斜杠命令 `ctx.commands` | 输入框 `/` 菜单；结果显示在对话里 | 完全原生 |
| 技能 `SKILL.md` | `/` 菜单技能组、「技能」页 | 完全原生 |
| 插件配置（schemastery `Config`） | 插件详情页、设置里的插件分组：原生表单 | 完全原生 |
| 工具 `defineTool` | 对话里的工具行：图标 + 名字 + 一行摘要 | 通用样式 |
| 模型输出 `dsh-ui` 代码块 | 助手消息里的图表 / 指标 / 对比卡 / 选项 | 完全原生 |
| 工具 `presentationMeta` 放 `"dsh-go/view"` | 工具行展开区的原生卡片 | 完全原生 |

**工具行摘要取值顺序**（`native-for-free.md` 原文）：
`command` → `file_path` → `path` → `pattern` → `query` → `url` → `description` → `prompt` → `name` → 其余参数里第一个字符串

### 10.3 需要 `dshGo` 清单的四个插槽（`docs/plugins/slots.md`）

| 插槽（surface） | 原生位置 | 每插件 | 文字上限 | 点按 |
|---|---|---|---|---|
| `composer.actions` | 输入框上方的胶囊行 | 2 | `label` 10 字 | 有 `effect` → App 直接执行；没有 → invoke `on(id)` |
| `message.actions` | 助手消息长按菜单，按插件分组 | 2 | `label` 12 字 | 同上 |
| `composer.strip` | 输入框上方一行；生成回复时让位 | 1 | `text` 24 字 | 有 `panel` → 打开；否则 invoke `on(id)` |
| `panels` | 对话详情页「插件」页签 / 全部功能 / 聊天页面板 | 4 | `title` 16 字 | 视图里的按钮、行、选项 → invoke |

⚠️ `contributes` 的真实键全集只有 **7 个**：`composer.actions`、`message.actions`、`composer.strip`、`panels`、`media`、`micro`、`overlays`
（`runtime/bridge/dsh-go-bridge/lib/native/limits.js:64` + `capabilities.js:18`）

### 10.4 `ctx.dshGo` 完整 API（源码 `core.js:425-448`，共 10 个成员）

```js
const service = {
  ctx, v: PROTOCOL, bridge: this.version,
  panel(id, def)     { return core.#register(this?.ctx, "panels",  id, def); },
  strip(id, def)     { return core.#register(this?.ctx, "strips",  id, def); },
  media(id, handler) { return core.#register(this?.ctx, "media",   id, handler); },
  micro(id, def)     { return core.#register(this?.ctx, "micro",   id, def); },
  on(id, handler)    { /* 点按处理 */ },
  refresh(id, scope) { /* 让 App 重新 GET 视图 */ },
  onCapability(fn)   { /* v1.2 能力事件 */ },
  readPickedFile(...) { /* v1.3 一次性只读句柄 */ },
}
```

**唯一正确写法**：

```js
export function apply(ctx) {
  ctx.inject(['dshGo'], (ctx) => {          // 只有装了 dsh-go 桥接的主机才会进来
    ctx.dshGo.on('read', (scope, event) => ({ insert: '…' }))
  })
}
```

⚠️ **写 `['dshGo']`，不要写 `['dshGo?']`，也不要把 `dshGo` 放进插件顶层的 `inject`**：
前者回调永远不执行，后者在没有桥接的主机上整个插件起不来。

### 10.5 回调参数与限额

| 名称 | 内容 |
|---|---|
| `scope` | `{ sessionId?, turnId?, messageId?, callId? }`，点按时再带 `draft` / `clipboard` / `message` |
| `caps` | 视图函数：`{ protocol, app, locale, surfaces, components }`；数据函数：`{ protocol, locale }` |
| `event` | `{ surface, id, action?, value?, locale }` |
| `emit(data)` | 每次给完整数据，桥接算差量、限 **4 Hz**、限 **32 KiB** |

**限额总表**：
- `dshGo` 整份 ≤ **16 KiB**
- 视图 ≤ **32 KiB** / **120 节点** / **8 层**
- handler 超时 **3 秒**；媒体 `media` 超时 **30 秒**
- 每次 invoke ≤ **4 个效果**
- `invoke.value` ≤ **4 KiB**
- 每个面板的数据源 ≤ **4 Hz**
- 连续 **5 次**违规 → 该插件本次运行降级成网页版

### 10.6 8 种白名单效果（`docs/plugins/actions.md` + 协议 §5）

| type | 字段 | 约束 | App 行为 |
|---|---|---|---|
| `insert` | `text` | 1–2000 字 | 插到输入框光标处，**不发送** |
| `command` | `name`, `args?` | `name` 匹配 `^[A-Za-z0-9][A-Za-z0-9._:-]{0,63}$` | 执行斜杠命令 |
| `open` | `page` | `panel:<id>` 或 App 页面 id | 打开页面 / 本插件面板 |
| `link` | `url` | 只收 `https://`，≤ 2048 字 | 每次弹确认，外部浏览器打开 |
| `toast` | `text` | ≤ 40 字 | 一行提示 |
| `copy` | `text` | 1–8000 字 | 复制到剪贴板 |
| `refresh` | `id?` | 本插件面板 id | 重新 GET 视图 |
| `patch` | `data`, `id?` | Merge Patch，≤ 16 KiB | 立即合并到面板数据 |

**永不开放**：批准工具调用、改权限模式、手机操控、装卸插件、切主机、读 App 设置或凭据、无点按自动触发。

### 10.7 能力边界三分法（完整 38 行表见 `research/40b-capability-matrix.md`）

| | A · DSH 网页端插件 | B · DSH-Go 原生界面 | C · Android 系统级 |
|---|---|---|---|
| 代码跑在哪 | Node（React 渲染在 WebView / 网页岛） | 逻辑跑在 Node，App **只画 JSON** | App 的 Kotlin |
| 谁能用 | 所有主机 | 只有装了 `dsh-go-bridge` 的主机 | **只有 App 自己** |
| 自定义绘制 / 动画 / 字体 | ✅ 任意 | ❌ 只能从白名单组件挑 | ✅ |
| 跨 App 悬浮窗 | ❌ | ❌ | ✅（仅 App 自己的桌宠） |
| 麦克风原始流 | ❌ | ❌（只能听「开口 / 说完」） | ✅ |
| 系统通知 | ❌ | ❌（v1 无 `notify` 效果） | ✅ |
| 后台保活 | ⚠️ 引擎可能被结束 | ❌ 声音/字幕后台**一定停** | ⚠️ 前台服务 + 厂商限制 |

### 10.8 主机覆盖（`docs/plugins/hosts.md`）

| | 工具卡 | 胶囊 · 菜单 · 状态条 · 面板 | 网页版插件（网页岛） |
|---|---|---|---|
| **本机引擎（手机）** | ✅ | ✅（桥接随 App 运行时安装） | ✅ |
| SSH 主机 | ✅ | ❌ 第二期（桥接还装不上） | ✅ |
| 中转主机（DSH Remote） | ✅ | ❌（中转不放行 `/dsh-go/*`） | ❌ |

---

## 11. Android 语音交互方案

### 11.1 三层拆解（这是最容易搞混的地方）

| 层 | 有什么 | 谁提供 | 插件能用吗 |
|---|---|---|---|
| **DSH 引擎（官方）** | **ASR**：`ctx.speechToText` + `sensevoice-local`（SenseVoiceSmall INT8 + sherpa-onnx）+ `speech/catalog｜prepare｜transcribe` | `dsh-experimental-speech-to-text` 等 4 个官方包 | ✅ **能用** |
| **DSH 引擎（官方）** | **TTS**：**没有** | — | ❌ 官方 README 原文：「仅支持完整录音转写。服务没有流式识别或语音合成方法。」 |
| **DSH-Go NA（v1.2）** | `audio` 播一段主机给的字节 / `voice` 只听开口 / `micro` 一行字幕 / `overlay` App 内悬浮字幕 | 桥接 | ✅ 只能用这四样 |
| **DSH-Go NA** | ASR / TTS | **协议里都没有** | ❌ |
| **Android 系统** | `SpeechRecognizer` / `TextToSpeech` / `AudioRecord` | App 自己 | ❌ 插件无入口 |

### 11.2 NA v1.2 四样能力的真实约束（`docs/plugins/voice.md`，全部原文核对过）

| 能力 | 你得到 | 你做 | App 做 |
|---|---|---|---|
| `audio` | 一段声音：播、暂停、继续、停，播完有事件 | 在主机上生成音频字节 | 从桥接取回、在手机上播 |
| `voice` | 用户开口 / 说完的事件 | 什么都不用做 | 在手机上听；一开口**先自己停掉声音**，再告诉你 |
| `micro` | 一条小数据通道，最快每秒 10 次 | 推 `{ text, progress }` | 画成一行字幕 |
| `overlay` | 一行悬浮字幕 | 指定它跟哪条 `micro` 通道 | 浮在对话顶栏下，带关闭键 |

**硬约束**：
- `media` **只回完整的一段**（`audio/mpeg|wav|ogg|mp4|aac`），**不支持流式**；≤ **16 MiB**
- `micro` 的 `text` ≤ **240 字**，App 最多每秒收 **10 次**（只要最新那份）
- `overlay` 是 **App 自己 Activity 里的 View，永不覆盖其他 App**，不需要窗口权限
- **只在前台**：退到后台、锁屏、换对话、换主机都立刻停，不会自动恢复
- 全 App 同时只有**一个**声音会话
- 用户按输入框语音输入时，插件的聆听和朗读**先停**

### 11.3 因此语音方案必须这样切

~~~
ASR：用 DSH 官方 ctx.speechToText（sensevoice-local）—— 不要自己接 ASR
     ⚠ 官方是整段转写，不流式（docs/design/voice-input.md:80 原文）

TTS：插件在主机侧自己接（当前主机上是 Edge 云端 或 本地 8765/8767）
     → 生成整段音频字节 → ctx.dshGo.media('speech', …) 返回 Buffer
     → App 用 audio.play 播（≤16 MiB，不支持流式）

打断：靠 NA v1.2 voice 能力。用户开口 → App 本地先停声音 → 再发 voice.speechStart
     → 插件在 onCapability 里收到，停止后续字幕推送

字幕：ctx.dshGo.micro('caption', …) 每秒推 ≤4 次 {text} + overlays 挂上去
~~~

### 11.4 语音插件选型（DSH 侧）

| | dsh-voice-mode 0.7.34 | dsh-talk 0.3.26 |
|---|---|---|
| 中文 ASR | sherpa-onnx zipformer2-zh 流式 + SenseVoice 定稿（本地、零 API Key） | Web Speech / 自备 FunASR / 自备 whisper.cpp |
| 实时流式识别 | ✅ 64ms 帧 / 100ms 增量 POST / partial | ❌ 只有浏览器 interim |
| 流式 TTS 按句朗读 | ✅ 分句器 + 逐句入队 + SSE | ❌ 整段一次性合成 |
| 语音打断（VAD） | ✅ Silero VAD 帧级 + 三档确认 + 回声门控 + 硬停播 | ❌ 按按钮才「请求」停，且**不会真停播** |
| 语音字幕 | ✅ `shell.overlay` 浮层 | ❌ 无 |
| 手机 WebView 麦克风 | ✅ 错误分类 + worklet 回退 | ⚠️ 默认 `stt.engine=auto` 在 Android WebView 上**必然死路** |
| 前后台切换 | ✅ hidden 停麦 + 可见恢复 | ❌ 完全没有 |
| 依赖体积 | ⚠️ 15MB WASM + 平台原生包 + ~390MB 模型 | ✅ 只有 zod |
| 与已装 voice-input-bundle 冲突 | ⚠️ 抢 `conversation.input.activity`（single） | ✅ 用 `conversation.input.left`（list） |

**结论：主选 `dsh-voice-mode`，备选 `dsh-talk`（只当架构参考，不装）。两者不许同时装。**

**但落到 DSH-Go 上两者都不能原样用**（voice-mode 的 host 侧 ASR 依赖 `sherpa-onnx-node` 原生 addon，**没有 android-arm64 平台包**）：

| 层 | 能否复用 | 说明 |
|---|---|---|
| 浏览器采集（getUserMedia + AudioWorklet） | ✅ 基本可复用 | 需 App 实现 `onPermissionRequest` 授 `RESOURCE_AUDIO_CAPTURE` 且页面在安全上下文 |
| HTTP/SSE 传输 | ✅ 可复用 | 同源 `127.0.0.1` |
| **host 侧 ASR/TTS 引擎** | ❌ **必须重写** | 换成 Android 原生 `sherpa-onnx` AAR，或把 ASR 留在主机上 |
| 字幕渲染 | ⚠️ 两选一 | 留在 WebView 里（零成本）或接 NA `overlays`（需重画） |
| 播放 / 打断静音 | ✅ 可复用 | WebView 里的 AudioContext |

**给 DSH-Go 的三条硬提醒**：
1. **Web Speech API 在 Android WebView 里不可用**（`webkitSpeechRecognition` 不暴露）→ 任何依赖它的方案在手机上必死。
2. **`speechSynthesis` 在 WebView 里不可靠**（`getVoices()` 常返回空，`onend` 有时不触发）。
3. **NA 的 `overlay` 不是系统悬浮窗**，它只是 App Activity 里的一个 View。想要跨 App 的字幕**做不到**。

---

## 12. 桌宠与视觉交互方案

### 12.1 三个桌宠并列对比（这是决策的核心表）

| 维度 | ① dsh-pet（本地 0.3.8，Web 浮层） | ② surfcore pet（Electron 独立进程） | ③ DSH-Go 原生宠物 |
|---|---|---|---|
| 代码位置 | `~/.dsh/profiles/desktop/node_modules/dsh-pet/` | `~/.dsh/surfcore/pet/app/`（feiyu-pet 0.2.3） | `dsh-Go app/lib/pets/*` + `app/android/.../pet/*` |
| 渲染 | DOM `<video>` A/B 双缓冲播 106 段 webm（VP9-alpha，51.8MB） | Electron BrowserWindow + **PixiJS** 分层 PNG rig（5 套皮肤，19–21 层） | Flutter Widget + Kotlin 悬浮窗；APNG / 序列图集 |
| 形象 | 蓝毛小女仆大肥鱼（视频） | 小肥鱼 ×5 套穿搭（feiyu-01..05，带 persona.md） | Deepy 小鲸鱼 / 鲸娘 + 可导入 Clawd / Codex 包 |
| 状态表达 | 会话事件聚合的工作态 + 情绪池文案 | 人格卡 + 衰减记忆卡 + 目的机制 | Clawd 12 态 + 优先级表（error 8 / notification 7 / … / sleeping 0） |
| 会不会自己调模型 | **会**：`whisperEnabled:true` → 一天约 288 次 | **会**：直接调 DSH llm 服务 | **不会**：`docs/design/pets.md` §0 明确「不主动调模型、不独立联网」 |
| 与 feiyu-mood 关系 | **无**（双侧 grep 0 命中） | **无**（走自己的记忆卡） | **无**（`pet_mood.dart` 是 App 内信号归约） |
| 当前状态 | 在装、在用 | **死的**（pid 31524 已不存在，start.log 报 offline） | 设计已定稿（29KB），代码已入库 |
| Android 可行性 | 低（Web UI 依赖） | 零（Electron 跑不到 Android） | **就是为 Android 写的** |
| 素材授权 | MIT 代码 / **素材禁止商用** | 自有（自绘） | ASSETS.md 登记 |

### 12.2 结论：正确路线不是三选一

> **③ 的骨架 + ② 的素材 + zhu 版（0.4.14）的纯逻辑**

- 想要「大肥鱼的脸」→ 素材在 ② 和 ①，不在 ③
- 想要「Android 上能跑」→ 只有 ③
- 想要「不烧 token、不打扰」→ 只有 ③ 的设计哲学（zhu 版 0.4.14 也符合：本地文案池，`ctx.llm` 命中 0）
- 想要「状态机 / 亲密度 / 玩法 / 文案引擎」→ zhu 版有约 **15 个纯模块**可整包搬到 Kotlin/Dart

### 12.3 zhu1090093659/dsh-pet 0.4.14 的真实情况

| 项 | 事实 |
|---|---|
| 是不是本地 0.3.8 的 fork | **不是**。包名 `@linxin666/dsh-pet`、Apache-2.0、`engines.dsh >= 0.2.1-rc.3`（**装不进当前 rc.2**） |
| Live2D 实现 | **Cubism Web 运行时**：用户自备 `live2dcubismcore.min.js` + pixi.js iife vendor + WebGL，不是自研裁帧 |
| 情绪表达 | 只有 9 条动画轨（其中只有 `failed` 是负向），**无 dims 情绪模型** |
| 主动说话 | **不调 LLM**（本地文案池轮播）—— 这点比本地 0.3.8 省钱 |
| 素材 | 3536 个 `.webp`，0 个 webm；**大肥鱼素材不在里面** |

### 12.4 可直接复用的纯逻辑（文件级，共 15 个模块）

| 文件 | 说明 |
|---|---|
| `src/state.ts` | 纯状态机，7 phase → 9 animation，时钟注入，零依赖 |
| `src/affinity.ts` | 亲密度：9 级阶梯 + 冷却 |
| `src/treats.ts` | 纯懒结算经济（小鱼干） |
| `src/gameplay.ts` | 43KB 全纯：解析 / 结算 / 概率 / 抽奖 |
| `src/remarks.ts` | 纯文案池 + round-robin |
| `src/chatter.ts` | StatusVoice + WhisperEngine 全纯，无 LLM |
| `src/voice-pack.ts` | 纯 JSON 归一化 / 合并 |
| `src/announce.ts` | 纯校验，有界 payload |
| `src/panel-actions.ts` | 纯注册表 + id 文法校验 |
| `src/contracts/renderer.ts`、`status-decoration.ts` | 纯类型 + 常量 |
| `src/client/sequences.ts` | 纯时序 |
| `src/client/spritesheet.ts` | 纯几何（输出数字不是 CSS） |
| `src/client/phase-stream.ts` | 纯 pub/sub |
| `src/client/work-tick-gate.ts` | 纯节流门 |
| `src/event-projection.ts` / `ledger.ts` | 逻辑纯，需换类型源 / 持久化层 |

### 12.5 必须在 Android 重写的（文件级）

- 全部 `src/client/*.tsx`（15 个）+ 3 个 `.module.css` —— React DOM / CSS Module
- `renderers/live2d.ts` + `live2d/runtime.ts` + `vendor-entry.ts` —— Cubism Web + pixi + WebGL
- `renderers/frames2d.ts` —— `createImageBitmap` + canvas 2D
- `src/routes.ts` + `http.ts` + `loopback.ts` + `access.ts` —— node:http 路由
- `src/service.ts`（51KB）+ `src/index.ts` —— cordis 插件入口与 session 接线
- 本地 0.3.8 独有：`runtime/electron-helper/*`（14 文件）—— Electron 主进程窗

### 12.6 情绪 → 表情的具体映射建议（给 Android 端）

```kotlin
// 数据源：GET /dsh-feiyu-mood/state → { mood: 0..100, dims: {...}, moodLabel, emoji }
fun emotionOf(mood: Int, dims: Map<String, Int>): Emotion = when {
  dims["sad"]!! > 20 || mood < 25      -> Emotion.SAD        // 😢
  dims["heartache"]!! > 20 || mood < 42 -> Emotion.WORRIED    // 😟
  dims["jealous"]!! > 20              -> Emotion.JEALOUS
  dims["sweet"]!! > 20                -> Emotion.SWEET
  dims["happy"]!! > 20 || mood >= 80   -> Emotion.HAPPY      // 😊
  mood >= 62                          -> Emotion.PLEASED
  else                                -> Emotion.CALM
}
```

这套映射**不需要任何插件配合**，直接读 HTTP 路由即可。表情素材可用 feiyu-mood 自带的 5 张 `assets/face-*.png`。

---

## 13. 权限与后台运行限制

### 13.1 AndroidManifest 里真实声明了什么（App 自己用，插件拿不到）

| 权限 | 用途 |
|---|---|
| `INTERNET` / `ACCESS_NETWORK_STATE` | — |
| `FOREGROUND_SERVICE` + `FOREGROUND_SERVICE_SPECIAL_USE` | 两个 `specialUse` 前台服务：`DshService`（跑本地 DSH）、`PetOverlayService`（桌宠） |
| `POST_NOTIFICATIONS` | 通知（targetSdk=28，不需要运行时申请） |
| `WAKE_LOCK` | `DshService` 持锁 |
| `RECEIVE_BOOT_COMPLETED` | 开机重排主动触发的单闹钟 |
| `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` | 电池优化豁免 |
| `RECORD_AUDIO` | 输入框语音输入 + 插件 `voice` 能力 |
| `SYSTEM_ALERT_WINDOW` | **仅 App 自己的桌宠**（`pet/PetOverlay.kt`），插件无入口 |
| `MANAGE_EXTERNAL_STORAGE` | 可选「访问手机文件」模式 |
| `moe.shizuku.manager.permission.API_V23` | 不打扰模式（Shizuku shell-uid） |
| `REQUEST_INSTALL_PACKAGES` | 应用内更新 |
| `READ_PHONE_STATE` | `tools:node="remove"` 显式移除 |

**没有声明的**：`SCHEDULE_EXACT_ALARM` / `USE_EXACT_ALARM`（0 命中）、`QUERY_ALL_PACKAGES`（0）

### 13.2 后台保活的真实能力

| 机制 | 有没有 | 证据 |
|---|---|---|
| WorkManager | **没有**（0 命中） | grep 实测 |
| JobScheduler | **没有**（0 命中） | grep 实测 |
| AlarmManager（WakeAlarm） | **有，单一闹钟** | `loop/proactive/WakeAlarm.kt` |
| 前台服务 | **有，两个 specialUse** | `AndroidManifest.xml:218-235` |
| 无障碍事件当 opportunistic 时钟 | **有** | `ProactiveRuntime.kt:162-168` |

`WakeAlarm.kt` 四条设计纪律（注释原文）：
1. **只有一个闹钟**（先 `am.cancel(pi)`）
2. **闹钟只是「去读盘比对」的提醒** —— 决策永远来自持久化状态
3. **降级不崩**：`setExactAndAllowWhileIdle` 抛错 → 退回 `setAndAllowWhileIdle`
4. **强行停止后什么都收不到**（原文：`After a force stop Android delivers none of these until DSH is opened again`）

### 13.3 不能假设的三条（用户点名要求）

1. **不能假设 Android 后台能无限跑**
   - `DshService` 是 **`START_NOT_STICKY`**（被杀不自动重启）
   - 开机不自启（只有 `BOOT_COMPLETED` 重排闹钟，不是重启引擎）
   - FAQ 原文：「各品牌限制不同，做完这些仍可能被系统结束」
2. **不能把 App 内面板当成系统级悬浮窗**
   - `overlay` 是 App Activity 里的 View（源码注释 `never over other apps`、`needs no window permission`）
   - 真正的跨 App 浮窗只给 App 自己的桌宠，**插件没有声明入口**
3. **不能把网页插件等同于 Android 原生桌宠**
   - `app/lib/plugins/seams/adapters/pet_adapter.dart:15` 是**空壳**：`protocols = <String>{}`，`canRender` 恒 false
   - 第三方原生宠物包是 P3 计划，**现在不存在**

### 13.4 插件侧的前台限制（协议写死）

- 插件的声音 / 字幕在**退到后台时被 App 强制 `revoke("background")`**
- 锁屏、换对话、换主机同样立刻停，**不会自动恢复**
- 全 App 同时只有**一个**声音会话

---

## 14. API / 事件 / 数据协议

### 14.1 DSH 侧已存在的只读接口（DSH-Go 可直接用，全部实测 200）

| 服务 | 方法 | 路径 | 返回要点 |
|---|---|---|---|
| feiyu-mood | GET | `/dsh-feiyu-mood/state?key=<slot>` | `{ok, key, mode, slots[], label, name, ownerName, mood, moodLabel, emoji, dims[{key,label,value}], lastReason, lastConfidence, updatedAt, injectMode, saidToday, injectQuota, decayHalfHours, timeline[]}` |
| feiyu-mood | GET | `/dsh-feiyu-mood/sessions` | `{ok, sessions:[{id,title}]}` |
| feiyu-mood | GET/POST | `/dsh-feiyu-mood/config` | 读 / 改配置 |
| fish-memory | GET | `/dsh-fish-memory/list` | 会话与记忆清单 |
| fish-memory | POST | `/dsh-fish-memory/add` | 加一条 |
| fish-memory | POST | `/dsh-fish-memory/delete` | 软删 |
| fish-memory | POST | `/dsh-fish-memory/policy` | 读 / 改召回策略 |
| speak-up | GET | `/dsh-speak-up/state` | `{配置, 目标活着, 目标情况[], 日志}` |
| speak-up | POST | `/dsh-speak-up/roll` | `{模式: 演练 | 真发 | 自动}` 手动触发一次 |
| qq-bridge | GET | `/dsh-qq-bridge/state` | `{版本, 总开关, 绑定[], NapCat, 错}` |
| dsh-pet | GET | `/dsh-pet-7340/state` | 宠物实时状态 |
| feiyu-app | GET | `/feiyu/ping` | `{ok, 版本}` |
| feiyu-app | GET | `/feiyu/sessions` | `{ok, 会话:[{sid, 消息数}]}` |
| feiyu-app | GET | `/feiyu/history?sid=&after=&limit=` | `{ok, sid, 总数, 消息:[{id,role,text,turn,at,kind}]}` |
| feiyu-app | POST | `/feiyu/send` | `{sid, text} → {ok, 说}` |

### 14.2 feiyu-app 作为移动端 API 底子的三个缺陷（DSH-Go 接手时要修）

1. **无鉴权 + CORS 通配**：`access-control-allow-origin: *` → 手机上任何 App / 网页都能往大肥鱼会话里塞话
2. **历史只在内存**（环形 300 条，重启即丢）；且实测 11 个会话里**没有** `429facb2`（大肥鱼生产会话）→ 手机端读不到主会话历史
3. **它自己判活用的是 `ctx.agents.get`**（保守）→ `/feiyu/send` 对冷会话会**误报失败**，虽然 `resolveAgent` 其实能唤醒

### 14.3 建议新增的协议（DSH 侧，不装插件）

```jsonc
// 1) 情绪快照（给 Android 端喂表情）—— 直接复用 feiyu-mood 的 /state，不新增
// 2) 候选话题池（给调度中心用）
// ~/.dsh/companion/topic-candidates.json  （见 §8.4）
// 3) 人格真源
// ~/.dsh/companion/persona.md  （软链或同步自 surfcore/characters/<id>/persona.md）
// 4) 调度快照（给 Android 端显示「大肥鱼现在想说话吗」）
// ~/.dsh/companion/scheduler-state.json
{
  "lastSpoke": 1793600000000,
  "nextRollAt": 1793600600000,
  "lastOutcome": "not_hit",      // hit | not_hit | failed | skipped
  "cooldownUntil": 1793600600000,
  "mute": { "from": 60, "to": 480 }   // 安静时段（分钟）
}
```

### 14.4 NA 协议要点速查（给写插件的 DSH-Go 开发者）

| 项 | 值 |
|---|---|
| 协议主版本 | `dshGo.v = 1` |
| 视图版本 | 根节点 `v: 1` 或 `2`（v2 才有样式 / 表单） |
| 桥接版本 | `dsh-go-bridge 0.4.0` |
| 路由 | `GET /dsh-go/v1/contributions`、`GET /dsh-go/v1/views/<pkg>/<id>`、`WS /dsh-go/v1/stream`、`POST /dsh-go/v1/invoke`、`GET /dsh-go/v1/assets/<pkg>/<path>` |
| 流消息（App→桥） | `hello` / `sub` / `unsub` / `ping` / `cancel` |
| 流消息（桥→App） | `welcome` / `data` / `patch` / `sync` / `stale` / `end` / `changed` / `error` / `pong` / `done` / `void` |
| 心跳 | 桥接每 25 秒 ping；60 秒无任何帧 → 关 1001 |
| 重连退避 | 1, 2, 4, 8, 16, 30, 30… 秒 |
| 数据补丁 | JSON Merge Patch（RFC 7386）；`null` 等于缺失 |
| v1.2 能力 | `audio` / `voice` / `micro` / `overlay`（靠协商，不靠版本号） |
| v1.3 能力 | `files`（`pickFile` / `pickImage`） |

---

## 15. 失败处理与降级

### 15.1 DSH 侧

| 场景 | 降级行为 | 证据 |
|---|---|---|
| 目标会话没活 agent | **当前：什么都不做**（不改的话） | speak-up L693-696 |
| 目标会话被写者持有 | `resolveAgent` 返回 error → 按「没活着」处理 | session-controller L243 |
| 记忆库太小导致召回失败 | `hard_gate`，注入空字符串（不报错） | fish-memory L2968-2979 |
| 判定模型不可用 | diag 记「判断返回空」，心情不变 | feiyu-mood diag.log 实例 |
| NapCat 掉线 | QQ 通道静默失效，插件本身不崩 | qq-bridge 探活 fetch failed |
| 插件未声明 compatibility | 只能实跑判断 | 见 §17 |

### 15.2 DSH-Go 侧（协议内建的降级链）

| 情形 | App 显示 |
|---|---|
| 主机没有桥接（SSH / 中转） | 只有工具卡；其余按网页版插件处理 |
| 插件 `minApp` 高于 App | 原生部分不显示，提示更新 |
| 某一项写错 | 只丢那一项，插件详情页「原生界面 · N 项有误」 |
| 插件运行中连续 **5 次**违规 | **本次运行降级成网页版**；重启引擎或重新启用后恢复 |
| 宿主半侧没注册处理 | 那一项置灰（`ready: false`） |
| 视图组件 App 不认识 | 画成灰色占位，其余照画 |
| 声音 / 字幕在后台 | 强制 `revoke"(background)"` |
| 麦克风被拒 | `voice.stopped { reason: declined }` |
| 录音权限没给 | `voice.stopped { reason: permission }` |
| 闹钟精确权限被收回 | 退回 `setAndAllowWhileIdle`（会迟到，设置页如实说明） |
| 强行停止 App | 所有闹钟与广播都收不到，直到用户再次打开 |

### 15.3 回滚优先级（万一出事按这个顺序退）

1. `speak-up` 总开关 → `false`
2. 卸载 / 禁用新装的语音插件（不影响 DSH 启动）
3. `git checkout` surfcore 对应文件 + 重启插件
4. 从 profile 的 `dsh.profile.bundles` 删掉新增行
5. 从 `~/.dsh/backups/dsh-ecosystem-audit-2026-11-02T2230/` 恢复配置（含 SHA256 清单）

---

## 16. 测试用例

### 16.1 已在本次完成的只读实测（有输出）

| # | 用例 | 方法 | 结果 |
|---|---|---|---|
| T01 | DSH 版本与 Cordis 版本 | 读 `runtime.json` + 解包 `app.asar` | PASS |
| T02 | 记忆写入 | 调 `remember` 工具 | PASS（"记下了（今天 22:49）。"） |
| T03 | 记忆落盘 | 读 `fish-memory/session-6489a146….json` | PASS（出现 `id=mmvzdk3h4894`） |
| T04 | 记忆召回 | 看 `traces` 最后一条 | PASS（`picked` 长度 1） |
| T05 | 记忆注入 | 下一轮 system prompt | PASS（出现【你记得的事】） |
| T06 | 记忆删除 | `POST /dsh-fish-memory/delete` | PASS（进 `deleted` 回收站） |
| T07 | 情绪状态读取 | `GET /dsh-feiyu-mood/state` | PASS（200，字段齐全） |
| T08 | 情绪判定活跃度 | 看 `diag.log` | **FAIL**（最后一次判定 2026-10-30T06:12） |
| T09 | 主动发言演练 | `POST /dsh-speak-up/roll` | **FAIL**（日志「按设计不自动唤醒」） |
| T10 | NapCat 探活 | `GET http://127.0.0.1:3000/` | **FAIL**（fetch failed） |
| T11 | 会话唤醒能力 | `GET /feiyu/debug?sid=429facb2-…` | **PASS**（resolveAgent 返回 `followup=function`） |
| T12 | 移动端 API 在线 | `GET /feiyu/ping` | PASS（`版本 0.1.0`） |
| T13 | 记忆库规模 | 读 `session-2c93cd4f….json` | **FAIL**（memories=1，召回 0/100） |
| T14 | 三调度器状态 | 读三个配置文件 + 两个日志 | **FAIL**（两个开着互相不知情） |
| T15 | surfcore 桌宠进程 | `Get-Process -Id 31524` | **FAIL**（进程不存在） |
| T16 | dsh-companion-loop v0.1.0 存在性 | 全盘搜索 + GitHub search | **不存在**（total_count: 0） |

### 16.2 DSH-Go 侧需要新增的测试（全部 NOT_TESTED）

| # | 用例 | 判据 | 现状 |
|---|---|---|---|
| A01 | 真机装一个带 `dshGo` 清单的插件，胶囊是否出现 | 输入框上方看到按钮 | NOT_TESTED |
| A02 | 桥接是否真被加载并 `provide` 出 dshGo | `GET /dsh-go/v1/contributions` 返回 200 | NOT_TESTED |
| A03 | 工具卡是否真画出原生卡片 | 对话里看到卡片 | NOT_TESTED |
| A04 | 四条路由端到端 + ETag + 补丁 + 重连 | `docs/plugins/checklist.md` 全过 | NOT_TESTED |
| A05 | `audio.play` 是否真出声 | 听到声音 | NOT_TESTED |
| A06 | `voice.speechStart` 打断是否准 | 说话即停播 | NOT_TESTED |
| A07 | `micro` / `overlay` 字幕观感 | 视觉确认 | NOT_TESTED |
| A08 | 后台保活（熄屏 30 分钟后闹钟还响不响） | 收到通知 | NOT_TESTED |
| A09 | SenseVoice 在 proot arm64 能否加载 | `speech/catalog` 返回模型 | NOT_TESTED |
| A10 | 国产 ROM（MIUI / ColorOS / HarmonyOS）悬浮窗与自启 | 权限页可开 | NOT_TESTED |
| A11 | 宠物外悬浮窗 + 三档形态 | 视觉确认 | NOT_TESTED |
| A12 | v1.3 系统文件选择器 | 能选到文件 | NOT_TESTED |

### 16.3 建议的验收门（用 dsh-clinic）

```bash
node <clinic>/doctor.mjs --repo <插件绝对路径> --no-smoke --only R,K,X
# 输出 --format check：{ok, contract:"dsh-doctor/v1", checks:[{name,status,detail,id,group}], doctor:{...}}
# 退出码 0/1/2（另有 3/4/5/6）
```

⚠️ **本机 npm / pnpm 不在 PATH ⇒ R2 必假红、D 组跑不了**。这是 clinic 自己的 bug（D1 对 spawnError 有 `category:infrastructure`，R2 没有）。
⚠️ **clinic 不是安全认证**：grep `vulnerab|CVE-|security|exfiltrat` 在 6 个检查文件里**零命中真检查**。

---

## 17. 版本兼容矩阵

> ⚠️ **重要机制发现**：`dsh.compatibility.dshReleases` 在 DSH 0.2.1-rc.2 里**根本没被读取**——
> `app-boot` 只按 `engines.dsh` 判；全树 grep `dshReleases` = **0 命中**。
> 所以「声明覆盖/不覆盖」**不决定能不能加载**，只决定商城/文档层面的可读性。
> 下表「判定」一列同时给出「声明是否覆盖」与「实际能否加载」，两者要分开看。

| 插件 | 版本 | 声明 | 与 DSH 0.2.1-rc.2 | 判定 |
|---|---|---|---|---|
| dsh-pet | 0.3.8 | `dshReleases: {0.2.1-rc.2: compatible}` | 命中 | ✅ |
| dsh-qq-bridge | 1.4.0 | `dshReleases: {0.2.1-rc.3: compatible}` | **未命中** | ⚠️ WARN（实跑正常，声明没覆盖） |
| dsh-bili-subagent | 0.8.7 | `engines.dsh: >=0.1.7-rc.1` | 命中 | ✅ |
| @zhz8888/dsh-android-emulator-helper | 1.2.0 | `>=0.2.1-rc.2` | 命中 | ✅ |
| dsh-context | 0.73.3 | `>=0.1.7-rc.2 || >=0.2.0-rc.2` | 命中 | ✅ |
| dsh-cost-meter | 1.9.22 | `>=0.2.0-rc.1 <0.3.0-0` | 命中 | ✅ |
| dsh-fish-memory / feiyu-mood / speak-up / feiyu-app / dshmarket / surfcore | — | **无 compatibility 声明** | — | ⚠️ 只能实跑 |
| **待装** dsh-voice-mode | 0.7.34 | `engines.dsh: >=0.1.1-rc.2 <0.3.0-0` | 命中，且 `verify-dual.sh` 21 版矩阵显式含 rc.2 | ✅ |
| **待装** dsh-talk | 0.3.26 | `engines.dsh: >=0.2.1-0 <0.3.0` | 范围内，但只测到 0.2.1-alpha.1 | ⚠️ |
| **待装** dsh-auto-memory | — | — | — | 未核 |
| **待装** dsh-aura-scheduler | 0.2.0 | `peer cordis: >=4.0.0`（open-top） | 命中 | ⚠️ open-top 区间 |
| **待装** dsh-clinic | 0.8.0 | `engines.dsh: >=0.2.1-0 <0.3.0` | 命中 | ✅ |
| **待装** zhu 版 dsh-pet | 0.4.14 | `engines.dsh: >=0.2.1-rc.3` | **不命中** | ❌ 装不进 |
| dsh-Go 桥接 | 0.4.0 | — | App 1.0.9 运行时锁 DSH **0.2.1-rc.4** | ⚠️ 比桌面新一个 rc |

### 17.1 两个必须知道的版本错位

1. **dsh-Go 手机本机引擎跑的是 DSH 0.2.1-rc.4**（仓库 `runtime/locks/dsh-0.2.1-rc.4/`），桌面是 **rc.2**。
   ⇒ 同一份 `cordis.patch.yml` 两边表现可能不同；插件装到手机上要先验一遍。
2. **Cordis 精确 4.0.5**（不是 4.x）。任何声明 `>=4.0.0` 的插件（如 aura）理论上会放行 5.0。

---

## 18. 需要用户在手机上完成的验证步骤

### 18.1 前置（本机电脑侧，一次性）

1. 确认电脑上的 DSH 是 **0.2.1-rc.2**，Cordis **4.0.5**（本文 §2.1 已核）。
2. 备份目录已生成：`C:\Users\敖震宇\.dsh\backups\dsh-ecosystem-audit-2026-11-02T2230\`（12 个配置文件 + 2 个记忆数据文件，带 SHA256 清单）。
3. 若要做语音测试，先在电脑上装 `dsh-voice-mode` 到 **abtest-zseven** profile，**不要装到 desktop**。

### 18.2 手机侧（按顺序做，每步记录结果）

| # | 步骤 | 期望 | 记录什么 |
|---|---|---|---|
| M1 | 打开 DSH Go，确认本机引擎能启动 | 引擎页显示「运行中」 | 版本号 + 是否安全模式 |
| M2 | 在 App「设置 › 本机引擎 › 日志」里看有没有插件加载失败 | 无红字 | 截图 / 抄报错 |
| M3 | 装一个带 `dshGo` 清单的最小插件（建议用 `dsh-go-example-quick`） | 输入框上方出现胶囊 | 截图 |
| M4 | 访问 `GET /dsh-go/v1/contributions`（主机 ip:port） | 返回 JSON，`plugins` 里能看到那个插件 | 抄 `rev` 与 `state` |
| M5 | 长按一条助手回复，看有没有「朗读」菜单（装了 readaloud 示例时） | 菜单出现 | 截图 |
| M6 | 点「朗读」，再**开口说话** | 声音立刻停 | 是否停、延迟大概多少 |
| M7 | 观察字幕 | 一行字幕随朗读滚动 | 是否跟得上 |
| M8 | 按 Home 退到后台，再回来 | 声音和字幕**立刻停**，回来不自动恢复 | 是否符合 |
| M9 | 熄屏 30 分钟后再看 | 主动触发的通知是否到 | 到了 / 没到 / 迟到多久 |
| M10 | 在系统设置里关掉电池优化、锁定后台（参考 App 的「后台运行」页） | 引擎存活更久 | 记录品牌与系统版本 |
| M11 | 开「App 外显示」宠物悬浮窗 | 桌宠浮在其他 App 上 | 截图（含权限弹窗） |
| M12 | 在国产 ROM 上试「后台弹出界面」/「显示在其他应用上层」 | 能开或给出明确失败原因 | 品牌 + 结果 |

### 18.3 必须在手机上验的三条底线

1. **不要在没有真机证据前把任何 Android 项写成 PASS。** 本次所有 Android 项都是 NOT_TESTED。
2. **不要用模拟器结果代替真机**（dsh-Go 自己的冷启动数字就是 x86_64 模拟器自测）。
3. **验证时用 debug 包**，`adb logcat -s DshProactive DshLoop DshA11y` 才有日志。

---

## 19. 安装与部署步骤 / 回滚步骤

### 19.1 阶段 0：不动任何东西（本次已完成）

- 只读盘点、源码审计、备份。**本次到此为止。**

### 19.2 阶段 1：修三处断链（**这是最高优先级，不装任何插件**）

| 步 | 动作 | 谁做 | 回滚 |
|---|---|---|---|
| 1.1 | 让 `session-429facb2-…` 活起来：在 DSH 里打开一次那个会话（或让 speak-up 调 resolveAgent） | 用户 / 后续改造 | 无副作用 |
| 1.2 | 修记忆召回：调 `POST /dsh-fish-memory/policy` 降低 `rarityK` / `threshold`，或先手工加 ≥ 10 条记忆让稀有度统计有意义 | 用户 | 记下原 policy 值，改回 |
| 1.3 | 关掉 dsh-pet 碎碎念：`main-config.jsonc` → `whisperEnabled: false` | 用户 | 改回 `true` |
| 1.4 | 确认 surfcore 是唯一提示类说话人（`carrier=pet`） | 只读确认 | — |

### 19.3 阶段 2：隔离 profile 验语音（**不进生产**）

```bash
# 用自带的 pnpm（PATH 上没有 npm/pnpm）
# PNPM 位于 %USERPROFILE%\.dsh\dsh-runtimes\dsh-primary-runtime\dependencies\pnpm\bin\pnpm.cjs
node "$PNPM" --version

# 装到 abtest-zseven（空 patch 的试验田）
node "$PNPM" --dir "$env:USERPROFILE\.dsh\profiles\abtest-zseven" add dsh-voice-mode@0.7.34
```

**必须同时做**：在 abtest-zseven 的 patch 里禁用 `dsh-experimental-voice-input-bundle`（两者都做语音输入，会抢 `conversation.input.activity` 槽）。

### 19.4 阶段 3：影子模式验记忆（**不进生产**）

- 把 `dsh-auto-memory` 装到 `abtest-zseven`
- 写 `~/.dsh/dsh-auto-memory.json`（影子配置，见 `research/20-memory-scheduler-clinic.md` §C.3）
- **三个 archive/delete 开关必须全 false**（默认会 `rmSync` 删 `~/.dsh/sessions` 下的会话目录）
- 观察 3–7 天，比对 `~/.dsh/sessions/` 目录数与基线**完全一致**

### 19.5 阶段 4：DSH-Go 侧接入（App 开发者做）

1. 用 `dsh-go-bridge 0.4.0`（手机本机引擎自带）
2. 写一个 `dsh-go-companion` 插件：只暴露四个插槽 + `audio` / `voice` / `micro` / `overlay` 四能力
3. **不改 dsh-Go 主仓库**（本次纪律）。所有原生能力用现有 NA 协议表达。
4. 情绪 → 表情：App 直接 `GET /dsh-feiyu-mood/state`，映射在 App 侧做

### 19.6 完整回滚

```powershell
$bk = "$env:USERPROFILE\.dsh\backups\dsh-ecosystem-audit-2026-11-02T2230"
Copy-Item "$bk\speak-up-config.json"             "$env:USERPROFILE\.dsh\speak-up-config.json" -Force
Copy-Item "$bk\qq-bridge-config.json"            "$env:USERPROFILE\.dsh\qq-bridge-config.json" -Force
Copy-Item "$bk\fish-memory-config.json"          "$env:USERPROFILE\.dsh\fish-memory-config.json" -Force
Copy-Item "$bk\feiyu-mood-config.json"           "$env:USERPROFILE\.dsh\feiyu-mood\config.json" -Force
Copy-Item "$bk\feiyu-mood-state.json"            "$env:USERPROFILE\.dsh\feiyu-mood\state.json" -Force
Copy-Item "$bk\dsh-pet-main-config.jsonc"        "$env:USERPROFILE\.dsh\dsh-pet\main-config.jsonc" -Force
Copy-Item "$bk\surfcore-config.json"             "$env:USERPROFILE\.dsh\surfcore\config.json" -Force
Copy-Item "$bk\profile-desktop-package.json"     "$env:USERPROFILE\.dsh\profiles\desktop\package.json" -Force
Copy-Item "$bk\profile-desktop-cordis.patch.yml" "$env:USERPROFILE\.dsh\profiles\desktop\cordis.patch.yml" -Force
# 记忆数据在同目录的 fish-memory\ 下，改名回 session-*.json 即可

# 校验（MANIFEST.json 里有每个文件的 SHA256）
Get-Content "$bk\MANIFEST.json" | ConvertFrom-Json | Select-Object -ExpandProperty files |
  ForEach-Object { (Get-FileHash $_.src -Algorithm SHA256).Hash -eq $_.sha256 }
```

### 19.7 本次已经生成的备份

```
~/.dsh/backups/dsh-ecosystem-audit-2026-11-02T2230/
├── MANIFEST.json                        （12 + 2 个文件的 SHA256）
├── speak-up-config.json                 497 B
├── qq-bridge-config.json                4696 B
├── qq-bridge-state.json                 73 B
├── fish-memory-config.json              70 B
├── feiyu-mood-config.json               673 B
├── feiyu-mood-state.json                589 B
├── dsh-pet-main-config.jsonc            354 B
├── surfcore-config.json                 173 B
├── profile-desktop-package.json         2870 B
├── profile-desktop-cordis.patch.yml     3413 B
├── profile-desktop-cordis.yml           223 B
├── profile-desktop-pnpm-lock.yaml       86596 B
└── fish-memory/
    ├── session-2c93cd4f-….json          47428 B
    └── session-2c93cd4f-….json.bak      2416 B
```

---

## 20. 未解决的问题

| # | 问题 | 为什么重要 | 需要谁 |
|---|---|---|---|
| Q1 | **主会话 `session-429facb2-…` 为什么会死、怎么保证以后不再死** | 它是三条链路的共同根因。现在只能手工打开；没有「保活」机制 | 用户 + DSH 侧 |
| Q2 | **记忆召回 `rarityK` 的设计缺陷**：库小的时候 C 恒为 0 | 直接导致「记忆写了但用不上」 | 需要改 fish-memory 或换策略（**需用户授权**） |
| Q3 | **speak-up 的唤醒改造**（约 20 行）要不要做 | 不做就永远不能主动开口 | 需用户授权 + 隔离验证 |
| Q4 | **人格有两份**（persona.md + qq-bridge 内嵌） | 两份不一致时行为会分裂 | 需要收敛方案（小改 qq-bridge） |
| Q5 | **surfcore 的定位**：它同时是桌宠 + 第二调度器 + 记忆卡 + B站自动化 | 它自己就是一个「小生态」，与本任务的六子系统高度重叠 | 需要用户决定：保留 / 拆解 / 只留人格卡 |
| Q6 | **语音插件在手机上的真实可行性**：sherpa-onnx 无 android-arm64 预编译包 | 决定语音是「主机侧出音频」还是「手机侧本地推理」 | 需 DSH-Go 侧确认 proot 里能否加载 .node |
| Q7 | **第三方原生桌宠目前不存在**（`pet_adapter.dart` 空壳） | 想要「原生大肥鱼桌宠」必须等 dsh-Go 的 P3，或自己做 | DSH-Go 侧 |
| Q8 | **dsh-Go 运行时是 DSH 0.2.1-rc.4，桌面是 rc.2** | 同一插件两边表现可能不同 | 需要版本对齐策略 |
| Q9 | **没有任何 Android 真机证据** | 本次全部 A01–A12 都是 NOT_TESTED | 用户手机 |
| Q10 | **dsh-companion-loop v0.1.0 在本机不存在任何痕迹**（GitHub search total_count: 0） | 不能当成「已有模块」 | 用户确认它到底在哪 |

---

## 附录 A：证据索引

所有详细报告在 `dsh-go-handoff/research/`：

| 文件 | 内容 |
|---|---|
| `00-SHARED-FACTS.md` | 共享事实底稿（环境、profile、插件版本、仓库 commit、网络约束） |
| `05-LEAD-EVIDENCE.md` | Lead 亲自实测的证据 E-01 ~ E-08 |
| `06-LEAD-DRAFT-ARCH.md` | 架构草稿（六子系统、数据结构、唤醒方案） |
| `10-pet-and-voice.md` | 桌宠三选一 + 语音两选一（67KB） |
| `20-dsh-pet.md` | zhu 版 dsh-pet 0.4.14 源码级审计（68KB） |
| `20-memory-scheduler-clinic.md` | 记忆 / 调度 / 质检三模块（71KB / 937 行） |
| `21-voice-comparison.md` | voice-mode vs dsh-talk 逐维对比（78KB） |
| `30-dshgo-native.md` | DSH-Go 原生接入事实材料（65KB / 926 行） |
| `31-dsh-go-local-source-authority.md` | **本地真实源码** `D:\code\dshgo` 权威依据（132KB） |
| `40-runtime-truth.md` | 大肥鱼生态当前真实状态审计（17KB） |
| `40b-capability-matrix.md` | 能力边界三分法 38 行矩阵（16KB） |

**关键外部坐标**：
- DSH-Go GitHub：`ccch1mneyyy/dsh-Go` @ `8fe1c1cd804ad763dd357bc1dec4b2cbfdc6f7fa`
- DSH-Go **本地真实 checkout**：`D:\code\dshgo`，分支 `feat/r3-candidate`，HEAD `bf9757631e43a2aea6052857d97ca0dbf9fbf0b8`（2026-10-31）
- ⚠️ 两者不是同一棵树：GitHub main 是 `8fe1c1cd`，本地是 `bf97576`。**本地这份带 R3 主动内核、有更完整的 loop/proactive/，是更新的工作分支。** 交接时优先看本地。

---

## 附录 B：本次审计没有做的事

| 没做 | 原因 |
|---|---|
| 没有安装任何插件 | 用户明确要求「先盘点、验证，再实施」，且要「不破坏已部署模块」 |
| 没有改 desktop profile | 同上 |
| 没有改 DSH-Go 主仓库 | 任务书明确禁止 |
| 没有向 QQ / B站 发任何消息 | 任务书明确禁止 |
| 没有跑 Android 真机 | 本机无 adb、无设备 |
| 没有真机语音验证 | 同上 |
| 没有把源码静态分析当成 PASS | 全部标注了「源码已证实存在」或 NOT_TESTED |
| 没有把 dsh-companion-loop v0.1.0 当成已完成模块 | 本机查无此物 |

## 附录 C：诚实声明

- 本文所有 PASS 都有实测输出或源码行号。
- 本文所有 Android 项都是 **NOT_TESTED**。
- 本文没有为了「看起来完整」而填任何数字。
- 7 个待研究仓库的具体 NOT_TESTED 条目分别在 `10-pet-and-voice.md`（19 项）、`20-memory-scheduler-clinic.md`（I 节）、`30-dshgo-native.md`（18 项 + 5 条文档/源码不一致）。


## 附录 D：审计残留物（必须知道）

本次审计是只读的，但为了验证链路产生了三处有意为之的残留，全部有记录、可清理：

1. `~/.dsh/feiyu-mood/state.json` 的 `lastDecayAt` 变化 —— 插件自身衰减定时器所致，**不是审计写入**
2. 新增 `~/.dsh/fish-memory/session-6489a146-….json` —— 本审计会话自动创建；内含一条**已删除**的测试记忆（`id=mmvzdk3h4894`，回收站状态，不参与召回）
3. 工作目录 `fishy-audit/` 与 `_research/` 可删（交付物在 `dsh-go-handoff/`）

详见 `dsh-go-handoff/evidence/AUDIT-RESIDUE.md`。

**未产生的残留**：没有安装任何插件、没有改 desktop profile、没有改任何插件配置、没有向 QQ/B站 发消息、没有创建定时任务或 goal、没有改 DSH-Go 主仓库。

## 附录 E：审计过程中的自我纠正记录

审计中团队推翻了自己先前的 5 条判断，全部记录在此，避免交接方沿用错误前提。

| # | 先前说法 | 纠正后 | 影响 |
|---|---|---|---|
| C1 | 「当前会话没有 remember 工具」 | **误报**。PTC 模式下模型侧 schema 只保留 `run_code`，但我实测 `Object.keys(tools)` 有 96 个工具含 `remember`。工具在，只是模型看不到名字 | 记忆链路的问题是「模型不会主动调」+「召回恒 gate」，不是「工具丢了」 |
| C2 | 「`/feiyu/debug` 报取不到 agent ⇒ resolveAgent 失败」 | **feiyu-app 诊断代码的 bug**（只查 `r.followup`，没查 `r.agent.followup`）。真实返回值是 `{agent:…}` 且 `agent.followup` 是函数 | §9 结论降级为 **WARN**，不是 PASS |
| C3 | 「dsh-pet 碎碎念每 5 分钟调模型」 | 对**本地 0.3.8** 成立（`whisperEnabled:true`，走 `ctx.llm`）；但 **zhu 版 0.4.14 是本地文案池轮播，不烧 token** | 不能拿「288 次/天」去评价 zhu 版 |
| C4 | 「情绪已停更 3 天」 | 实测 `lastDecayAt` 是刚刚（衰减在跑）；`updatedAt` 是 26 小时前；**最后一次「心情更新」日志是 2026-10-30T06:12**（约 3 天） | 「判定停更 3 天」成立，「插件停更 3 天」不成立 |
| C5 | 「dsh-Go 插件是 JS bridge / WebView」 | 都不是。是 **Node 里的 cordis 服务 → HTTP/WS 到 127.0.0.1 → Flutter MethodChannel → Kotlin**。WebView 只用于网页岛 | 决定了「插件能做什么/不能做什么」的全部边界 |

| C6 | 「情绪 diag 距今约 3 天」 | 要分两件事：`diag` **最后一条是 2026-11-02T14:33**（「已挂心情」）；**最后一次真实情绪变化**是 2026-10-30T06:12（约 3.5 天前） | 表述必须分开，否则会被误读成"插件 3 天没运行" |
| C7 | 「```dsh.compatibility.dshReleases``` 决定插件能不能加载」 | **不决定**。DSH 0.2.1-rc.2 的 `app-boot` 只读 `engines.dsh`，全树 grep `dshReleases` = 0 命中 | dsh-qq-bridge 声明只写 rc.3 但照样加载成功（它有 `engines.dsh: >=0.2.1-rc.2`） |
| C8 | 「`state.json` 的 `timeline` 空证明情绪没变过」 | `diag.log` 里有 2 条「心情更新」（mood 68→69），但 `timeline` 是空 —— **两者矛盾，原因未查明** | 标 NOT_TESTED，不能拿任一侧当结论 |

另外一处**用户任务书里的错误**：

| 任务书写的 | 实际 |
|---|---|
| `dsh-qq-link` | 不存在。真实包名 `dsh-qq-bridge@1.4.0` |
| `dsh-companion-loop v0.1.0` | **本机查无此物**（全盘搜索 + GitHub search `total_count: 0`）。不能当成已有模块 |
| `src/client/RangeBubble.tsx` 是气泡组件 | 不是。它是设置面板的滑块气泡；真正的公告气泡在 `PetSprite.tsx` 的 `UsageAnnouncementBubble` |

## 附录 F：三条最容易被误传的"事实"（请勿沿用）

1. ❌「DSH-Go 能跑 Android 原生桌宠插件」 → 现在**不能**，`pet_adapter.dart` 是空壳。
2. ❌「NA 的 overlay 是系统悬浮窗」 → 是 App Activity 里的 View，**永不覆盖其他 App**。
3. ❌「手机后台能一直跑」 → `START_NOT_STICKY` + 开机不自启 + FAQ 明说「仍可能被系统结束」。
