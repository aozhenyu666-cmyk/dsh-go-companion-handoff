# 40 · 运行时真实状态（只读验证报告）

> 验证人：runtime-verifier（子代理，只读）
> 时间：2026-11-02 22:31 → 23:0x +08:00
> 基线：DSH Desktop **0.2.1-rc.2** / Electron 44.7.0 / Node 24.21.0 / pnpm 11.28.5 / win32 x64（与 00-SHARED-FACTS 一致，已复核 `resources/runtime/primary-runtime/runtime.json`、`runtime/versions.json`）
> 只读声明：本报告全程未装/卸插件、未改 `~/.dsh` 下任何文件、未改 desktop profile、未向 QQ/B站 发任何消息、未改 DSH-Go 主仓库。所有"实测"均为：读文件 / 读日志 / 读状态 / **只读 HTTP GET**。

---

## 0. 口径（先看这条，否则会误读全表）

| 标记 | 含义 |
|---|---|
| **PASS（运行）** | 我在**这台机器当前进程**上拿到运行证据：HTTP 路由返回 200 + 有真实时间戳的日志/状态在动，或会话文件里有真实事件 |
| **WARN（运行）** | 进程确实活着，但功能被关着 / 数据为空 / 状态与日志矛盾 / 声明版本对不上 |
| **FAIL（运行）** | 有运行证据表明链路断了（连不上、文件里没有、调用报错） |
| **仅静态（源码）** | 只读了代码，**没有**运行证据。**这永远不算 PASS** |
| **NOT_TESTED** | 没测，附原因 |

⚠️ 本报告反复出现的两种"假活"，先说清楚：

1. **状态文件 mtime 新 ≠ 有活动。** `~/.dsh/feiyu-mood/state.json` 的 mtime 一直是"几十秒前"，因为插件每 3 秒 tick 一次会重写它（`index.js:1604 ctx.interval(tick,3000)` → `getState(...,{persist:true})` → `saveState`）。**看 mtime 会误判成"情绪在动"。**
2. **插件"加载成功" ≠ 功能可用。** 下面凡标 PASS 的，都是"该插件确实在跑"，**不代表它的效果验收过**。尤其"情绪是否改变了说话方式""主动发言是否自然"这类，本报告**没有一项**做过效果验收（见 §4、§5）。

---

## 1. 环境与版本（复核 + 补充：日志到底在哪）

### 1.1 复核结果（与 00-SHARED-FACTS 一致）

| 项 | 值 | 证据 |
|---|---|---|
| DSH Desktop | 0.2.1-rc.2 | `resources/runtime/primary-runtime/runtime.json` → `"desktopVersion": "0.2.1-rc.2"` |
| Node（自带） | 24.21.0 | 同上 `"node": "24.21.0"`；`runtime/versions.json` 同 |
| pnpm（自带） | 11.28.5 | 同上 |
| Electron | 44.7.0 | 崩溃日志头 `electron: 44.7.0` |
| Cordis | ~4.0.5 | 各 bundle peerDependencies 声明 `@deepseek-ai/cordis: ~4.0.5` |
| Web GUI | http://127.0.0.1:57009 | 会话上下文 + `netstat` 显示 57009 LISTENING（pid 20828） |

### 1.2 补充：DSH 的日志到底在哪（00-SHARED-FACTS 里没有）

| 类型 | 路径 | 说明 |
|---|---|---|
| Electron userData | `C:\Users\敖震宇\AppData\Roaming\@deepseek-ai\dsh-desktop\` | Chromium profile + 崩溃日志 |
| **主进程/宿主崩溃日志** | 同上 `logs\crash-<时间>-main.log` / `-host.log` / `-web-boot.log` | **只有崩溃/致命错误时才写，不是常规运行日志** |
| 现有全部崩溃日志 | 共 7 个，最新 `crash-2026-10-31T21-20-50-079Z-host.log`（6669 B） | 见 1.3 |
| 插件自写诊断 | `~/.dsh/fish-memory-diag.log`(244 KB)、`~/.dsh/feiyu-mood/diag.log`(4.6 KB)、`~/.dsh/surfcore/plugin.log`、`~/.dsh/bili-duty/duty.log` | **插件加载成功/失败的唯一可查证据就在这里** |

**关键结论（WARN）：DSH 桌面端没有一个"插件逐个加载结果"的日志文件。** 我把 userData 目录翻完了，只有崩溃日志。所以"某插件到底加载了没有"只能靠两条路推：(a) 插件自己写的 diag/log 有本次启动的时间戳；(b) 它注册的 HTTP 路由能不能 GET 到。**§2 的表就是按这两条路做的，两条都没有的一律 NOT_TESTED。**

### 1.3 三次历史崩溃（都是真的，都不是"最近这次启动"）

| 时间(UTC) | 来源 | 错误 | 判读 |
|---|---|---|---|
| 2026-10-25T14:33:45 / 14:34:01 | main + web-boot | `DesktopHostFatalError: cannot create effect on inactive context`（`CordisError: INACTIVE_EFFECT`，cordis `lib/index.js:1133 Fiber.assertActive`） | 启动期崩溃，宿主没起来。当时的问题，与当前进程无关 |
| 2026-10-30T16:01:20 | main + web-boot | `net::ERR_CONNECTION_REFUSED`（boot 阶段） | 宿主端口没监听就 launch 了 |
| 2026-10-31T21:20:50 | host | `dsh desktop host exited with 1073807364`；退出前刷 `[qq-tool-restrict] restrict 跳过（示例）：pwsh — tools.restrict() names unknown global tool "pwsh"`、`跳过 66/67 个当前版本不存在的工具名`；另有大量 `[deepseek-account] /api/v0/users/get_user_summary no-response`、`OpenRouter price refresh timed out`、`api.deepseek.com/models timed out after 5000 ms` | 宿主**整体退出**。同时暴露：surfcore 的 qq-tool-restrict 名单里 66/67 个工具名在当前 DSH 版本**根本不存在**（那套安全限制基本空转，只剩执行期白名单兜底） |

**最近一次启动是成功的**：speak-up 日志第一行 `11-02 21:39:20 已装好 v0.2.7 …`，且 57009 端口正常服务。→ **"最近一次启动有没有插件加载失败"：没有证据显示有失败，但也没有成功清单可查（见 1.2）。**

### 1.4 §1 状态判定

| 项 | 状态 | 证据 |
|---|---|---|
| 版本/运行时与底稿一致 | **PASS（运行）** | runtime.json / versions.json 直读 |
| 找到主进程日志位置 | **PASS（运行）** | `%APPDATA%\@deepseek-ai\dsh-desktop\logs\` 7 个 crash 文件 |
| 最近一次启动无致命错误 | **WARN（运行）** | 只有间接证据（speak-up 21:39 装载日志 + 端口在听）；无集中启动日志可证 |
| 逐插件加载成功/失败清单 | **FAIL（运行）** | 该日志不存在（这本身是 DSH 的可观测性缺口） |
| 历史崩溃已定位 | **PASS（运行）** | 3 类错误原文见上表；1073807364 那次根因未查明 |

---

## 2. 插件加载状态（逐个，31 个 bundle）

### 2.1 先澄清一处底稿偏差

00-SHARED-FACTS §3 把 8 个 `@deepseek-ai/dsh-*` 官方 bundle 列成"profile 里装的"。**实际它们不在 `profiles/desktop/node_modules` 里**，而是从安装包 `app.asar/dsh/node_modules/@deepseek-ai/` 解析。实测：

- `profiles/desktop/node_modules/@deepseek-ai/` 只有：`cosmokit dsh-brand dsh-sdk-protocol dsh-skill dsh-subagent-codex dsh-tool-subagent dsh-tools dsh-util-values schemastery`
- `dsh-base / dsh-web-app / dsh-experimental-* / dsh-experimental-agent-team-profile` 8 个的 package.json 全部在 `app.asar\dsh\node_modules\@deepseek-ai\`，版本均为 **0.2.1-rc.2**，均声明 `dsh.bundle`。

影响：**这 8 个不在 pnpm 依赖图里，不受 profile lockfile 约束，跟着 DSH 本体走。**

### 2.2 31 个 bundle 逐项表

判定依据只用三类运行证据：**H** = 它的 HTTP 路由 GET 通（非 404）；**L** = 它写了带本次启动时间戳的日志；**T** = 它的工具出现在真实会话的工具表/技能清单里。

| # | bundle | 版本 | 证据（运行） | 状态 |
|---|---|---|---|---|
| 1 | `@deepseek-ai/dsh-base` | 0.2.1-rc.2 | 整棵树的基础，官方包，asar 内 | **PASS（运行，间接）** |
| 2 | `@deepseek-ai/dsh-web-app` | 0.2.1-rc.2 | 57009 在服务、Web UI 可用 | **PASS（运行）** |
| 3 | `@deepseek-ai/dsh-experimental-voice-input-bundle` | 0.2.1-rc.2 | asar 内，bundle 声明完整 | **NOT_TESTED** |
| 4 | `@deepseek-ai/dsh-experimental-auto-review` | 0.2.1-rc.2 | 同上 | **NOT_TESTED** |
| 5 | `@deepseek-ai/dsh-experimental-inspector-profile` | 0.2.1-rc.2 | 同上 | **NOT_TESTED** |
| 6 | `@deepseek-ai/dsh-experimental-tool-worktree` | 0.2.1-rc.2 | `create_worktree` 工具在真实工具表里 | **PASS（运行）** |
| 7 | `@deepseek-ai/dsh-experimental-plugin-market-bundle` | 0.2.1-rc.2 | `/dsh-market/*` 全 200 | **PASS（运行）** |
| 8 | `@deepseek-ai/dsh-subagent-codex` | 0.2.1-rc.2 | `subagent_codex` 在工具表里 | **PASS（运行）** |
| 9 | `@michengai/dsh-skills-manager` | 1.2.9 | `create_skill` / `skill` 工具在表里；技能清单 60+ 条可见 | **PASS（运行）** |
| 10 | `@liustack/modsearch` | 5.10.10 | 无路由、无日志、无工具名 | **NOT_TESTED** |
| 11 | `dsh-flash-presets` | 0.2.2 | 只插 agent preset 行，无探针 | **NOT_TESTED** |
| 12 | `dshmarket` | 1.66.47 | `/dsh-market/status` 200（`boot:"20828-1793626754"`）、`/installed` 200 8468 B、`/registry` 200 8.5 MB | **PASS（运行）** |
| 13 | `dsh-startup-screen` | 1.1.2 | 无路由探针 | **NOT_TESTED** |
| 14 | `@wenaixi/dsh-superpower` | 8.0.0 | 技能清单有 `brainstorming / writing-plans / executing-plans / verification-before-completion` 等 20+ 条 | **PASS（运行）** |
| 15 | `@maci0/dsh-caveman` | 0.18.0 | `caveman`、`caveman-compress` 工具 + 8 个 caveman-* 技能 | **PASS（运行）** |
| 16 | `dsh-emil-skills` | 1.0.0 | 技能清单有 `emil-design-eng / animate / apple-design / review-animations` 等 | **PASS（运行）** |
| 17 | `@deepseek-ai/dsh-experimental-agent-team-profile` | 0.2.1-rc.2 | `agent_teams_*` 14 个工具在表里 | **PASS（运行）** |
| 18 | `dsh-pet` | 0.3.8 | `/dsh-pet-7340/state` 200，`counter` 每秒在动（`1793630439625`），`workStatus.state:"working"` | **PASS（运行）** |
| 19 | `@nanmicoder/dsh-agent-teams` | 0.1.25 | 同 #17 工具面 | **PASS（运行）** |
| 20 | `dsh-better-sidebar` | 0.28.0 | `/sidebar/api` **403**（存在但拒绝）= 路由已注册 | **PASS（运行）** |
| 21 | `cc-safety-net` | 2.6.16 | 无路由、无日志、无工具名 | **NOT_TESTED** |
| 22 | `dsh-context` | 0.73.3 | `/api/dsh-context/fleet`、`/latest`、`/detail` 均 **401**（存在，需鉴权） | **PASS（运行）** |
| 23 | `dsh-cost-meter` | 1.9.22 | `/api/cost-meter/state` **401**；宿主日志有 `[dsh-cost-meter] OpenRouter 价格刷新失败` | **PASS（运行）** |
| 24 | `@zhz8888/dsh-android-emulator-helper` | 1.2.0 | `android_*` 24 个工具在表里；`~/.dsh/android-emulator-helper/logs/` 有 11 份历史 emulator 日志 | **PASS（运行）** |
| 25 | `surfcore` | 0.7.10 | `/dsh-surfcore/pet-api/health` 200；`plugin.log` 末行 `2026-11-02T13:39:20 已激活 v0.7.10` | **PASS（运行）＋ WARN（功能空转，见 §8）** |
| 26 | `dsh-fish-memory` | 1.4.3 | `/dsh-fish-memory/list` 200；diag 末行 `2026-11-02T14:46:59 这一轮用哪句检索…` | **PASS（运行）** |
| 27 | `dsh-feiyu-mood` | 0.8.1 | `/dsh-feiyu-mood/state` 200；diag 末行 `2026-11-02T14:33:56 已挂心情到 session-429facb2…` | **PASS（运行）** |
| 28 | `dsh-speak-up` | 0.2.7 | `/dsh-speak-up/state` 200；日志每 10 分钟一条 `总开关是关着的，这轮不动`（21:49/21:59/22:09/22:19/22:29） | **PASS（运行）＋ WARN（功能关闭）** |
| 29 | `dsh-bili-subagent` | 0.8.7 | `/api/bili-duty/status` 200；15 个 `bili_*` 工具在表里；`duty.log` 末行 `2026-11-02T13:39:18 已挂载` | **PASS（运行）** |
| 30 | `dsh-qq-bridge` | 1.4.0 | `/dsh-qq-bridge/state` 200；`NapCat状态: {"错":"fetch failed"}` | **PASS（加载）／ FAIL（通道）＋ WARN（版本声明）** |
| 31 | `dsh-feiyu-app` | 0.1.0 | `/feiyu/ping` → `{"ok":true,"版本":"0.1.0"}`；`/feiyu/sessions` 200 | **PASS（运行）** |

装了但**不在** bundles 里（等于没启用，别当活的）：`dsh-i-have-adhd@1.0.2`、`dsh-whale-widget@0.4.2`、`dsh-plugin-wallpaper-engine@1.4.0`（+6 子包）、`@zseven-w/dsh-android@0.1.0-rc.12`。

### 2.3 兼容声明核对（含用户点名的那两条）

机制（源码已读，`app.asar/dsh/node_modules/@deepseek-ai/dsh-app-boot/lib/index.js`）：

- `unmetRequirements()` L279-284：**只看 `engines.dsh`；没有才看 `@deepseek-ai/dsh*` peers。**
- `evaluatePluginCompatibility()` L311-326 → `loadProfileDirectory()` L1025-1026：**不兼容就 throw → 该 bundle 进 `skippedBundles`，整个 bundle 被跳过。**
- **`dsh.compatibility.dshReleases` 在这个版本里根本没被读取**（app.asar 全树 grep `dshReleases`，0 命中）。它只给商城/插件详情页做展示判定。

按真实规则重算 31 项（semver + `includePrerelease`）：

| bundle | engines.dsh / peers | 结论 |
|---|---|---|
| `@maci0/dsh-caveman` | `>=0.2.1-rc.2 <0.3.0` | 满足 |
| `@zhz8888/dsh-android-emulator-helper` | `>=0.2.1-rc.2` | 满足 |
| `dsh-qq-bridge` | `engines.dsh: ">=0.2.1-rc.2"` | **满足**（所以它加载了） |
| `dsh-pet` | 无 engines.dsh；peers `^0.2.0-rc.1` | 满足 |
| `dsh-context / dsh-cost-meter / dsh-bili-subagent / dsh-fish-memory` 等 | 见各 package.json | 全部满足 |
| **无任何声明** | `dsh-feiyu-mood / dsh-speak-up / surfcore / dsh-feiyu-app / dsh-flash-presets / dsh-startup-screen / dsh-emil-skills / cc-safety-net / @liustack/modsearch` | 无约束 = 不校验 |

**用户点名的两条，结论与我预期不同，请注意：**

- **`dsh-pet 0.3.8`**：`dsh.compatibility.dshReleases = {"0.1.7-rc.2":"incompatible","0.2.0-rc.1":"compatible","0.2.0-rc.2":"compatible","0.2.1-rc.2":"compatible"}` → **覆盖当前版本，声明相符 → 不算 WARN。**
- **`dsh-qq-bridge 1.4.0`**：`dsh.compatibility.dshReleases = {"0.2.1-rc.3":"compatible"}` → **和当前 0.2.1-rc.2 对不上 → 按用户要求标 WARN。**
  但必须补一句：**它同时声明了 `engines.dsh: ">=0.2.1-rc.2"`，而 DSH 只按 engines 判，所以它照样加载成功**（`/dsh-qq-bridge/state` 200 就是证据）。→ **这是"声明没覆盖但实际能跑"，不是"加载失败"。**

### 2.4 §2 状态判定

| 项 | 状态 | 证据 |
|---|---|---|
| 23 个插件有运行证据在跑 | **PASS（运行）** | 上表 |
| 8 个插件无任何运行探针 | **NOT_TESTED** | 见上表 |
| 4 个包装了没启用 | **WARN（运行）** | `bundles` 数组里没有它们 |
| qq-bridge 版本声明未覆盖当前版本 | **WARN（静态）** | `dshReleases: {0.2.1-rc.3}`，实跑 0.2.1-rc.2 |
| dsh-pet 版本声明相符 | **PASS（静态）** | `dshReleases["0.2.1-rc.2"]="compatible"` |
| 兼容性只按 engines 判、`dshReleases` 不被执行 | **PASS（源码）** | app-boot L279-284 / L311-326；全树无 `dshReleases` 引用 |

---

## 3. 记忆链路真伪（重点）

源码：`~/.dsh/profiles/desktop/node_modules/dsh-fish-memory/lib/index.js`（6501 行 / 340 KB）
版本 1.4.3 ← pnpm-lock 锁定 tarball commit **`2bb9be7b83bac5e69737a457873027c3933e12cc`**（仓库 HEAD 现为 `003716b0f682177c9237214dac19a7a052a39812`，本地已落后于上游）

### 3.1 什么时候写记忆

**只有一条路：模型自己调 `remember` 工具。插件没有任何自动写入钩子。**

- `makeRememberTool(ctx)` — L3185-3427：`defineTool({name:'remember', …})`，`execute` 里做配额（`policy.dailyQuota` 默认 8/天，L3289）、人称归一、事实键收敛（L3403-3407）、`saveStore`（L3408）、写 diag（L3409）。
- 注册点两处（都不是自动写）：
  - 全局：`同步工具()` L4110-4127，条件 `globalOn()`，每 500 ms tick 调（`ctx.interval(tick,500)` L4251）。
  - 按 agent 再注册一份：L4226-4233（注释写明原因：PTC/native 两种模式下工具表都可能漏，L4222-4224）。
- 没有任何"监听对话自动提取"的代码。**这点决定了：模型不主动记 = 永远没有新记忆。**

### 3.2 什么时候召回

`agent.ctx.systemPrompt.context({name:'memory', order:900, interpolate:false, text:()=>{…}})` — L3920-4015。

1. 取查询词：`选查询词(lastUserTextOf(agent.session), lastText)` L3934；短于 6 字再 `补线索()` L3937。
2. `loadStore(sid)` → 空库直接 return ''（L3958-3959）。
3. `recall(data, query, now, sid)` L3961 → 打分 `P = C×(基础+鲜活+重要+情绪)`（文件头 L10）。
4. 每次召回都往 `data.traces` 记一条（L3982-3999）并 `saveStore`。
5. `render(selected, now, selfNameOf(sid))` L4003 → **"【你记得的事】…"**（L3160-3168）。
6. 用 `context` 不用 `section` 的理由写在 L3907-3918：**保 DeepSeek 前缀缓存**（section 会废掉 12k 缓存，实测 0.001→0.02）。

### 3.3 数据文件 schema（实测）

路径：`~/.dsh/fish-memory/session-<sid>.json`
现有 2 个非空库：`session-2c93cd4f…json`(47 KB, 1 条记忆, 100 条 trace)、`session-6489a146…json`(4.7 KB, 0→1 条记忆, 6 条 trace)。另有 `.bak` 两份。

```
{ version: 2, purgeEpoch,
  policy: { baseHalfLifeDays:30, emotionHalfLifeDays:7, threshold:0.3, maxItems:6, maxChars:800,
            dailyQuota:8, queryGate:0.8, … },
  memories: [{ id:"mmvw95m9j274", text, at, createdAt, category:"event", importance:0.7,
               emotion:null, tags:[…], source:"她自己记的", "事实键":"dshgor4.拦截实现方式" }],
  traces:  [{ at, session, query, gate, gateLimit:0.8,
              picked:[id…], rows:[{id, C, P, omission:"hard_gate", timeHit, status}] }],
  deleted: [] }
```

### 3.4 diag.log 实测（244 KB / 1336 行）

| 统计 | 值 |
|---|---|
| `remember：<sid> 记了「…」` 成功写入 | **1 次**（2026-10-31T10:22:38，session-2c93cd4f） |
| `注入 N 条` 总次数 | **434 次**，其中**非空仅 8 次**（全在 10-31 的 session-2c93cd4f），其余 426 次都是 `注入 0 条：（空）` |
| `这一轮像是用上了记忆` | **0 次** |
| `assemble 的工具（1 个）：run_code` + `↑ 里面有 remember 吗：没有 ✗` | **18 次**（首次 2026-10-30T03:56:04，末次 2026-11-02T14:08:22） |
| 同一日志里 `有 remember ✓` | 103 次（97/91 工具的 native 装配） |

### 3.5 【纠正 lead 的一个判断】"当前会话没有 remember 工具"是**误报**

用户点名要把这条写进报告。我实测的结论是：**这句 diag 是插件检测方式过时造成的假警报，不是"AI 没有 remember 可用"。**

三条运行证据：

1. **本会话（PTC / run_code 模式）的 `tools` 绑定里真的有 `remember`**：
```
typeof tools.remember  ===  'function'          // 实测：true
```
它是 PTC 宿主绑定（`port.postMessage({type:"call", …})`），可以直接调。
2. **PTC 模式下模型看到的是"生成的 SDK"，不是逐个工具名列表**：我在会话文件里 grep 到生成的 TypeScript 声明，里面**明写**着
```
remember: { text: string; importance?: number; tags?: string[]; "取代"?: string; … }
```
（`~/.dsh/sessions/…/79d6816d-…/session.v4.jsonl.zstd`、`3bbebcb4-…`、`6489a146-…` 三个 PTC 会话都有这段）
3. 机制侧对应：`@deepseek-ai/dsh-tools` 有 `mode: "native"|"ptc"`（`lib/index.js:2763 mode: z.union(["native","ptc"]).default("native")`），PTC 下"the model's only declaration of the tools is this generated block"（L2030 注释）；`PTC_ONLY_INSTRUCTION` 明确 "run_code is the only tool you can call directly"（L2614）。**所以 `assembly.tools` 只剩 `run_code` 是设计如此，不是 remember 丢了。**

**但假警报底下压着一个真问题（这条要保留）：** 大肥鱼那间会话（`session-429facb2`）里，`remember` **被模型调用过 1 次，然后失败**：

```
tool/call  turn 23  name:"remember"  arguments:{"text":"鱼片今天被领导当众骂了一顿，心里堵得慌、不想干活。",…}
tool/result turn 23 isError:true text:"The tool call was interrupted after it was recorded, but no result was durably recorded…"
turn/end   turn 23 reason:{"kind":"error","error":{"message":"Cannot read properties of undefined (reading 'prepare')"}}
```
（`~/.dsh/sessions/…/session-429facb2-…`，解压后 288 事件；**30 轮里 6 轮是这个错**：turn 4/8/9/11/13/23）

这个错误串 `Cannot read properties of undefined (reading 'prepare')` 在 `profiles/desktop/pnpm-workspace.yaml` 的注释里被点名：`@maci0/dsh-caveman` 把 `@deepseek-ai/dsh-tools` 复制出第二份 → 两份各自 Symbol 不相等 → `dsh-agent-loop` 取 `ctx.tools[该 Symbol]` 得 undefined → 工具调用抛 `…reading 'prepare'`。修法是 `overrides: '@deepseek-ai/dsh-tools': 'file:C:/Users/敖震宇/.dsh/shims/dsh-tools'`，该 shim 的 mtime 是 **2026-10-30T15:54:37**，晚于 429 最后一次活动（06:12）。→ 今天新会话（`session-dd762176`，22 轮、1320 次 tool/call）**0 次该错误**。**即：这个故障大概已被 shim 修掉，但我只拿到"之后两天没复现"，不等于验证通过。**

### 3.6 大肥鱼会话（session-429facb2）的记忆实况 = 死的

| 检查 | 结果 | 证据 |
|---|---|---|
| 库文件是否存在 | **不存在**（`~/.dsh/fish-memory/` 只有 2c93cd4f、6489a146 两库） | 目录列表 |
| `/dsh-fish-memory/list` 里 429 那条 | `{enabled:true, live:true, count:0, lastTrace:null}` | 实测 HTTP |
| 有没有召回过 | **0 次**（`lastTrace: null`） | 同上 |
| 会话里有没有 【你记得的事】 | **0 次**（该会话全文 grep 命中 0） | 解压后 665 KB / 288 事件 |
| 会话里有没有 【记忆】常驻提示 | **5 次**（5 条 system/message 结尾都有） | 同上 |

→ **大肥鱼那间会话：写入 0、持久化 0、召回 0；只有"提示她记得去记"那条常驻提示在。**

今天唯一一次成功写入+召回的完整链路（在 `session-6489a146`）是 **lead 自己做审计测试写进去的**：

```
~/.dsh/fish-memory/session-6489a146….json  count=1
注入内容：- 今天 22:49：【审计测试·可删】2026-11-02 记忆写入链路自测：这条由 lead 在 fishy-audit 审计中写入…
（出现在 session-6489a146 的 runtime-context sections[].name==="memory"）
```
**这算"写入→落盘→召回"三段在真机上跑通过一次**（PASS 级证据），但样本是测试数据、不是真实对话。

### 3.7 §3 状态判定

| 功能点 | 状态 | 证据 |
|---|---|---|
| 插件加载、`remember` 可被模型调用 | **PASS（运行）** | 429 turn 23 真实发起 remember 调用；PTC 会话 SDK 声明了 remember |
| **写入（自动）** | **FAIL（源码+运行）** | 源码无自动钩子；diag 全历史只有 1 次真实写入 |
| **写入（手动/按需，PTC）** | **WARN（运行）** | 能调用；429 那次因 `'prepare'` 崩溃失败（6/30 轮）；此后 2 天未复现（shim 修复，未经复验） |
| **持久化** | **PASS（运行）** | 2 个库文件在盘上，schema 完整 |
| **召回（机制）** | **PASS（运行）** | 434 次 recall trace；2c93cd4f 里 8 次真实注入（内容带时间） |
| **召回（大肥鱼会话）** | **FAIL（运行）** | 429 库为空、lastTrace=null、会话里 0 次注入 |
| 召回命中率 | **WARN（运行）** | 434 次召回里 426 次"注入 0 条"（98%）；`queryGate:0.8` 门槛偏硬 |
| "像是用上了记忆"（consumed） | **NOT_TESTED / 0 次** | diag 该行 0 命中 |
| 记忆对她说话方式的影响 | **NOT_TESTED** | 没做对照验收 |

---

## 4. 情绪链路（dsh-feiyu-mood 0.8.1）

源码：`node_modules/dsh-feiyu-mood/lib/index.js`（1900 行）+ `client.js`（38 KB）
版本 0.8.1 ← pnpm-lock 锁定 tarball commit **`8c1759c9bbd1d7e3580852da49e63d801d1d2381`**（仓库 HEAD `1881cef1…`）

### 4.1 它到底做什么（三种注入格式，全是"文字"）

| injectMode | 注入内容（`moodLine()` L931-958） |
|---|---|
| `off` | 不注入（L1511） |
| `label` | `（此刻你）心情开心，有点甜、挺踏实。这是背景，不是这一轮必须有的情绪；鱼片说别的事，你就按自己的判断来，别把这句话念出来。` |
| `event`（当前配置） | 三行：`（这会儿）心里是松的，还挺高兴` / `（刚才）鱼片说被领导当众骂了一顿…我当时胸口一紧…` / `这是这会儿的底子，不是非这样不可——鱼片说别的，你按自己的判断接，也别把这段念出来。` |

注入通道：`agent.ctx.systemPrompt.context({name:'feiyu-mood', order:900})` — L1505-1545。

**行为侧只有这一条通道。** 全文件 grep：

- `tools.register` / `tools.restrict` / `tools.guard`：**0 处**
- `ctx.agents` 除遍历挂载点外没有任何"改行为"的调用
- 与 speak-up / qq-bridge / dsh-pet 的耦合：**0**（speak-up、dsh-pet 源码里 `心情|mood` 0 命中；qq-bridge 只有一句注释提到"改了鱼的心情"）

### 4.2 情绪怎么产生（不是关键词猜，是真调模型）

`agent/turn-stopping` 触发（L1450）→ `judge()`（L1260）→ `ctx.llm.stream` 用 `deepseek-account / deepseek-flash` 判 JSON `{moodDelta, dims, reason, confidence}` → `applyDelta` → `saveState`。判定提示词 L1142-1200（要求过去式、禁代词、禁万能词）。

三道闸（都在 `text()` 里，L1508-1543）：
1. `s.injectedFor === s.updatedAt` → 同一次变化只说一次（L1513）
2. `minor`（小波动）不说（L1514）
3. `lastConfidence < injectConfidence(55)` 不说（L1517）；`injectPerDay(6)` 日额度（L1526-1533）

### 4.3 现有状态（实测 + 与日志矛盾，这条要注意）

| 项 | 值 | 来源 |
|---|---|---|
| mood | **70**（基线） | `/dsh-feiyu-mood/state` + `state.json` |
| dims | 7 维全 **0** | 同上 |
| timeline | **[]** | 同上 |
| `updatedAt` | `1793535279201` = **2026-11-01T12:14:39Z** | state.json |
| `lastDecayAt` | `1793630406887` = **2026-11-02T14:40:06Z**（几十秒前） | state.json |
| state.json mtime | 一直在刷新（每 3 秒 tick 写一次） | 文件系统 |
| diag.log 最后一条 | `2026-11-02T14:33:56 已挂心情到 session-429facb2…` | diag.log |
| **diag.log 里真实的心情变化** | `2026-10-30T06:11:55 心情更新：mood=68，鱼片说被领导当众骂了一顿…`、`06:12:12 心情更新：mood=69，鱼片说有人夸做饭好吃…` | diag.log |
| **diag.log 里"告诉她心情"成功记录** | **1 次**：`2026-10-30T06:12:06 告诉她心情（event）：（这会儿）心里是松的…` | diag.log |

**矛盾点（WARN，原因未查明）**：diag 明明记了 10-30 有 mood 68→69 的两次变化，但 `state.json` 的 `timeline` 是空的、`updatedAt` 停在 11-01T12:14:39、`lastReason`/`lastConfidence` 全空。排查过：`/dsh-feiyu-mood/reset` 会打一条 `心情已重置到基线`（L1780），**diag.log 里没有这条**。→ 不是通过 reset 路由清的；**状态文件被外部覆盖或换过一份，只读条件下无法确认来源。**

### 4.4 关键判断：是不是"只在 prompt 里注入一个 mood 标签就算完成"？

**比"只注入一个标签"多，但也没多到"驱动行为"。逐条对标：**

| 用户批评点 | 现有实现做到什么 | 判定 |
|---|---|---|
| 只在 Prompt 注入一个 mood 标签 | **不只是标签**：`event` 模式给的是"此刻心情 + 这件事的来历 + 允许你变"三行，且是**模型现写的口语**（不是枚举），并带可信度门槛 / 日上限 / 变化去重，防"变成背景音" | **WARN（设计合格，效果未验）** |
| 情绪影响**说话方式/语气** | 只有"一段文字进上下文"这一条路。**没有任何机制约束输出**（无 tool、无 guard、无后处理、无采样参数）。是否真改变语气，**取决于模型是否服从那段中文** | **NOT_TESTED** |
| 情绪影响**主动发言频率** | **完全没有**。speak-up 命中线只看峰谷/距上次说话/连没中（§5.2），源码 `心情|mood` 0 命中；mood 插件也不碰 speak-up | **FAIL（运行+源码）** |
| 情绪影响**话题选择** | 无机制。`event` 那行文字带了"鱼片说了什么"，**间接**可能影响话题，但和 mood 值本身无关 | **NOT_TESTED（设计上无）** |
| 情绪**长期一致性** | 有：`decay()` L841-858 指数衰减（总心情半衰 12 h、单维 90 min），跨会话共享一份 state | **PASS（运行）** |
| 情绪**影响别的插件** | 0 耦合 | **FAIL** |
| 情绪判定真的在跑 | **WARN**：最后一次真实判定 2026-10-30T06:12；此后 diag 只有"已挂心情"两条，无判定记录。原因：429 那间会话之后没有新对话（会话末事件 `session/end-seed` 2026-10-30T13:50），判定挂在 `turn-stopping` 上，没对话就不判 | **WARN** |

### 4.5 "距今约 3 天"这个说法要修正

- **diag.log 最后一条是 2026-11-02T14:33:56**（"已挂心情"），**不是 10-30**。3 天前那条是"最后一次**心情变化**"。
- "已挂心情"只是挂载日志，**不代表发生过一次情绪判定**。
- 准确表述：**最后一次真实的情绪变化在 2026-10-30 06:12（约 3.5 天前）；最后一次挂载在 2026-11-02 14:33；这期间没有任何判定记录。**

### 4.6 §4 状态判定

| 功能点 | 状态 | 证据 |
|---|---|---|
| 插件加载 + 状态路由可用 | **PASS（运行）** | `/dsh-feiyu-mood/state` 200；diag 有 11-02 挂载 |
| 心情判定（真调模型）能跑 | **PASS（运行，历史）** | diag 10-30 两条 `心情更新` 带完整 reason |
| 心情判定**当前**是否在跑 | **WARN** | 3.5 天无判定；根因是目标会话没有新 turn |
| 注入通道存在 | **PASS（源码）** | `systemPrompt.context` L1505 |
| 注入**真的进了模型上下文** | **NOT_TESTED** | 429 会话 288 事件里 **0 条 `feiyu-mood` section**，只在 diag 见到那一行。**无法证实那行字到过模型** |
| 状态持久化 | **PASS（运行）** | state.json 每 3 秒被写 |
| 状态与日志一致 | **WARN（运行）** | timeline 为空 vs diag 有 2 条变化；原因未查明 |
| 情绪影响语气 | **NOT_TESTED** | 无对照实验 |
| 情绪影响主动发言 | **FAIL** | speak-up 源码 0 处读 mood |
| 情绪影响话题 | **NOT_TESTED（设计上无）** | — |

---

## 5. 主动发言链路（dsh-speak-up 0.2.7）—— 最关键

源码：`node_modules/dsh-speak-up/lib/host.js`（1298 行 / 63 KB）
版本 0.2.7 ← pnpm-lock 锁定 tarball commit **`fee91af30d0242609ed1c8a89e8852942a296f20`**（仓库 HEAD `438bb636…`）

### 5.1 配置（实测，只读读文件 + 读它的 /state）

`~/.dsh/speak-up-config.json`：
```json
{ "总开关": false, "试运行": false, "目标会话": ["session-429facb2-…"],
  "间隔分钟": 10, "峰时命中线": 25, "谷时命中线": 85, "没中加成": 15, "加成上限": 50,
  "安静起": 1, "安静止": 8, "近阈值分钟": 20, "近因子": 0.15, "远阈值分钟": 360, "远因子": 1.5,
  "搭话冷却分钟": 10, "触发文案": "有点无聊，去看看鱼片在干嘛", "角色": "user" }
```
→ **总开关 = false，主动发言当前是关着的。**

### 5.2 调度算法（逐行）

| 环节 | 位置 | 实现 |
|---|---|---|
| 定时器 | `换表()` L786-817 | `停表 = ctx.interval(cb, 间隔分钟*60*1000)` → **固定 10 分钟一跳**，不是 cron |
| 安静时段 | `在安静时段()` L292-296 | 小时 1~8 点不掷 |
| 峰谷判档 | `判峰时()` L371-375 | 写死：周一~五 09–12、14–18 = 峰；其余（含周末全天）谷。**无设置项** |
| 基础命中线 | `算基础命中线()` L385-391 | 峰 25% / 谷 85% |
| 时机因子 | `算这次命中线()` L624-638 | 距她上次说话 <20 min → ×0.15；>360 min → ×1.5；否则 ×1 |
| 攒加成 | `算命中线()` L408-412 | `线 = min(100, 基础×因子 + min(上限50, 连没中×15))`；**先乘因子再加加成** |
| 掷骰 | L724-738 | `掷值 = Math.random()*100`；`掷值 >= 线` → 没中，`连没中+=1`；否则命中、清零 |
| 选会话 | L668-700 | 目标里挑"活着且 `status==='idle'`"的，随机选 1 个；**全不可用 → 直接放弃** |
| 冷却 | L705-710 | 自己刚开过口 10 分钟内不重复 |

### 5.3 【核心问题】触发时是"发一个事件"还是"真的开一轮 agent 会话"？

**答：真的开一轮 agent 会话，走 DSH 自己的 agent loop。不是发事件、不是塞死文本。** 调用链（带行号）：

```
ctx.interval(cb, 10min)                       L802  ← ① 定时器
  └─ 掷一次('定时','自动')                      L648
       ├─ 检查 总开关/安静时段/目标可用/冷却     L655-710
       ├─ 算这次命中线 → 掷骰                    L719-738
       ├─ 框住内心话(配置.触发文案)              L763   ← 给文案包一层"这是你自己冒出来的念头"
       ├─ 拿造消息工厂()                        L765   ← 优先 @deepseek-ai/dsh-llm 的 createUserMessage
       ├─ 手搓用户消息(文案)                    L488-495 (fallback)
       │     { id, role:'user', content:[{type:'text',text}], source:{kind:'speak-up'} }
       ├─ await agent.followup(消息)            L772   ★★★ 真正的入口
       └─ ctx.sessions.flush(agent.session)     L777   （落盘）
```

**`agent.followup()` 意味着**（源码文件头 L1-14 自己写明，我核对过 `@deepseek-ai/dsh-*` 的行为）：向**活着的同一个 agent 的收件箱**推一条消息，`dsh-agent-loop` 认领它 → 起 turn → 跑 step，**跟用户手打走同一条路**（人格、历史、工具、记忆全自带）。

**已知硬限制（源码 L16-33，三条证据写在注释里，我认同）**：角色只能是 `user`，做不出 `developer`。所以它送的是 `source.kind="speak-up"` 的 user 消息，并在正文里用 `框住内心话()` 明确告诉模型"这是你自己心里冒出来的念头，不是对方说的"（L508-513）。配置里填 `角色: developer` 会被主动拦下不发（L753-757）。

### 5.4 【对 DSH-Go 的结论】需不需要"会话唤醒能力"？

**需要。而且是唯一硬缺口。**

- speak-up 判活只靠 `找agent()` → `ctx.agents.get(id)`（L460-475）。**没有 `sessionController.resolveAgent` 冷唤醒。**
- 全不可用时它明确放弃并记账，L693-696：
```
勾的目标会话现在都不能说话（…）—— 按设计**不自动唤醒**，这轮什么都不做
```
- 实测这条确实触发过（`/dsh-speak-up/state` 日志）：
```
{ "时间": "11-02 22:28:50", "文本": "勾的目标会话现在都不能说话（d77d8dca(没活着)）—— 按设计**不自动唤醒**，这轮什么都不做" }
```
- 对照：**同一时刻 `/feiyu/debug?sid=429facb2` 说 `有活agent:false`、`唤醒:"取不到可用 agent"`**，而 `/dsh-speak-up/state` 的 `目标情况[0].活着` 是 `true`（它用不同判据）。→ **两个插件对"是否活着"给相反答案，说明这个判据本身不稳。**
- `dsh-feiyu-app` 里**已经写了**冷唤醒（`取agent()` L212-236 调 `sessionController.resolveAgent`），**但实测也取不到**（见 §7）。→ **所以"DSH 有冷唤醒"这件事，在这台机器上目前拿不到正面证据。**

### 5.5 有没有真实触发记录

有，7 条（`/dsh-speak-up/state` → `日志`，全部 11-02 21:39–22:29）：
```
21:39:20 已装好 v0.2.7：每 10 分钟掷一次骰子｜现在命中线 85.0 = 谷时命中线 85% × 时机因子 1…｜总开关 关
21:49:20 总开关是关着的，这轮不动
21:59:20 总开关是关着的，这轮不动
22:09:20 总开关是关着的，这轮不动
22:19:20 总开关是关着的，这轮不动
22:28:50 勾的目标会话现在都不能说话（d77d8dca(没活着)）—— 按设计**不自动唤醒**
22:29:20 总开关是关着的，这轮不动
```
历史真实命中（大肥鱼会话里 `source.kind='speak-up'` 的 user 消息）：**4 条**，全在 2026-10-30 04:15–04:38 → **那时它是开着的、真的掷中过、真的走了一轮。**

### 5.6 §5 状态判定

| 功能点 | 状态 | 证据 |
|---|---|---|
| 插件加载 / 定时器在跳 | **PASS（运行）** | 10 分钟一条日志，连续 7 跳 |
| 调度算法实现正确 | **PASS（源码）** | L292-412 / L624-738，纯函数可测 |
| 触发 = 真开一轮会话 | **PASS（源码+运行）** | `await agent.followup(消息)` L772；429 里有 4 条 `speak-up` 来源消息 |
| **当前是否可用** | **NOT_TESTED / WARN（关闭中）** | `总开关:false`，每跳都记"这轮不动" |
| **冷会话能否被唤醒** | **FAIL** | 无 `resolveAgent`；实测打出"没活着…不自动唤醒" |
| 触发文案是否被正确理解 | **WARN（历史运行）** | 4 次真实触发，模型侧是否自己起话头**未验收** |
| 主动发言的自然度/频率 | **NOT_TESTED** | 需开着跑几天才有样本，本次不能开 |

**明确回答用户的问题：能不能在关掉状态下说它"可用"？→ 不能。** 当前只能写："已装载、定时器在跑、算法在、但总开关关着，且不具备冷会话唤醒能力 → **功能未验收**。"

---

## 6. QQ 桥（dsh-qq-bridge 1.4.0）

源码：`node_modules/dsh-qq-bridge/lib/host.js`（2742 行 / 141 KB）
版本 1.4.0 ← pnpm-lock 锁定 tarball commit **`fd8f30f02d64b135c26e5f010b6e72fb1e465bcc`**（仓库 HEAD `fba7e3ee…`）

### 6.1 配置（实测）

`~/.dsh/qq-bridge-config.json`：`总开关:true / 试运行:false（真发）/ 私聊会话=目标会话=session-429facb2… / 私聊QQ:2079****46 / NapCat:"http://127.0.0.1:3000" / 令牌:"" / 绑定[1]{类型:私聊, 号码:2079****46, 会话:429facb2…, 称呼:"鱼片", 开:true} / 轮询间隔毫秒:2000 / 人设开:true（人设正文 4.8 KB）`

### 6.2 只读探测 NapCat（**没有发任何消息**）

| 探测 | 结果 |
|---|---|
| `GET http://127.0.0.1:3000/` | **fetch failed** |
| `GET /get_status`、`/get_login_info` | **fetch failed** |
| TCP connect 127.0.0.1:3000 | **ECONNREFUSED** |
| `netstat -ano` 找 :3000 | **无任何监听** |
| 进程列表 | 有 `QQPCExternal`×4、`QQPCRTP`、`QQPCTray`（**这是 QQ 客户端，不是 NapCat**）；**没有 NapCat / LLOneBot 进程** |
| 插件自己的判断 | `/dsh-qq-bridge/state` → `"NapCat状态": {"错":"fetch failed"}` |

### 6.3 结论：QQ 通道当前**离线**

- **出站（DSH→QQ）**：`落地一发()` L1011 起；`发一个(配置.NapCat,'send_private_msg',…)` L971 —— 现在必然失败。
- **入站（QQ→DSH）**：`轮询收消息()` L2353 起，每 2 秒 POST NapCat —— 现在每跳都失败。
- 代码本身是完整双向（`转入一条()` L1882 → `agent.followup({source:{kind:'qq'}})` L1988）；**且它也只用 `ctx.agents.get`，同样不能唤醒冷会话**（`找agent` L1156-1169；拿不到就返回"那间会话现在没活着（…），先把它打开" L1890）。

### 6.4 §6 状态判定

| 项 | 状态 | 证据 |
|---|---|---|
| 插件加载、配置正确 | **PASS（运行）** | `/dsh-qq-bridge/state` 200，配置全对 |
| 版本声明覆盖当前版本 | **WARN（静态）** | `dshReleases:{0.2.1-rc.3:compatible}`；但 `engines.dsh:>=0.2.1-rc.2` 满足 → 能加载 |
| NapCat 在线 | **FAIL（运行）** | 3000 ECONNREFUSED，无监听进程 |
| **QQ 通道当前是否在线** | **FAIL —— 离线** | 上述探测 + 插件自报 `fetch failed` |
| 收发链路代码完整性 | **PASS（源码）** | 出站 L964-1007；入站 L1882-2006 |
| 冷会话唤醒 | **FAIL（源码）** | 同 speak-up，只用 `agents.get` |

---

## 7. dsh-feiyu-app（本地 0.1.0，link: C:/Users/敖震宇/feiyu/plugin）

源码：`~/.dsh/profiles/desktop/node_modules/dsh-feiyu-app/index.mjs`（240 行 / 8.7 KB）
包声明：`dsh.bundle.patch: ./cordis.patch.yml` → `- insert: [{id: feiyu-app, name: dsh-feiyu-app}]`；`inject = ['webServer']`

### 7.1 接口全表（DSH-Go 移动端最直接的接入点，按原文记录）

**Base URL：`http://127.0.0.1:57009`（= DSH 自己的 webServer 端口，不是独立端口；不带前缀）**

| 方法 | 路由 | 请求 | 响应 | 行为 |
|---|---|---|---|---|
| GET | `/feiyu/ping` | — | `{"ok":true,"版本":"0.1.0"}` | 探活 |
| GET | `/feiyu/sessions` | — | `{"ok":true,"会话":[{"sid":"<裸uuid>","消息数":N}]}` | 内存里有哪些会话 |
| GET | `/feiyu/history?sid=&after=&limit=` | `sid`=**裸 uuid（不带 `session-` 前缀）**；`after`=毫秒时间戳（只取 `at > after`）；`limit` 默认 60、夹 1..200 | `{"ok":true,"sid":"…","总数":N,"消息":[{"id","role":"user|assistant","text","turn","at","kind"}]}` | 读历史 |
| POST | `/feiyu/send` | `{"sid":"<裸uuid>","text":"..."}` | 成功 `{"ok":true,"说":"送进去了"}`；失败仍 **HTTP 200** + `{"ok":false,"说":"缺 sid"/"缺 text"/"那间会话现在没开着…"/"送不进去：…"}` | 往会话 `agent.followup({role:'user', source:{kind:'user'}})` 送一条 |
| GET | `/feiyu/debug?sid=` | `sid` | `{"ok":true,"sid","有活agent":bool,"唤醒":"…","形状":"…","服务":[…]}}` | 排障用 |

实现细节（`index.mjs`）：
- 路由注册：`挂(路径, handler)` L95-106，走 `c.webServer.register({kind:'exact', path, handler})`。
- **HTTP 方法不做限制**：handler 只判 `OPTIONS` → 204，其余一律进 handler（L99-103）。所以 GET 打 `/feiyu/send` 也会执行（body 空 → "缺 sid"）。
- `/feiyu/send` 的 `source.kind` **故意写 'user'**（L131-137 注释：心情插件按 `source.kind` 分口径，写别的会让心情 timeline 一条不记）。

### 7.2 鉴权：**没有**

- 响应头写死 `'access-control-allow-origin': '*' / 'access-control-allow-headers':'content-type' / 'access-control-allow-methods':'GET,POST,OPTIONS'`（L44-50）。
- **没有 token / cookie / Origin 校验**（同机器的 surfcore pet-api 是要求 `x-pet-token` 的）。
- → 任何能访问 127.0.0.1:57009 的本机进程/页面，都能**读会话内容 + 冒充用户发话**。**移动端接入前必须补的第一件事。**
- 端口：**没有自己的端口**，寄生 DSH webServer（当前 57009）。DSH 换端口 = 移动端配置跟着变。

### 7.3 历史只存在内存（进程重启就空）

`const 历史 = new Map()` + `上限 = 300`（L18-19），靠 `ctx.on('session/event', …)`（L62）实时攒。**不落盘、不读盘。**

实测：
```
GET /feiyu/history?sid=429facb2-…&limit=3
→ {"ok":true,"sid":"429facb2-ccc8-49dc-b99d-cf44d77d8dca","总数":0,"消息":[]}
     （该会话盘上有 245 KB / 288 事件）
GET /feiyu/sessions
→ 只有 11 个 sid，最大"消息数":13
```
→ **"读历史"只能读到"插件加载之后、本进程内发生过的事"，历史会话一律 0 条。对移动端等于不可用。**

### 7.4 冷唤醒：写了，但这台机器上验不出来

`取agent()` L212-236：先 `找agent`（`ctx.agents.get`），拿不到就 `sessionController.resolveAgent(sid)`，还拿不到就看返回值本身有没有 `followup`。
实测 `GET /feiyu/debug?sid=429facb2…` →
```json
{"有活agent":false,
 "唤醒":"取不到可用 agent",
 "形状":"object :: agent | agent 键: loopCtx,id,options,session,inbox,phase,activityDone,scope,ctx,dispatch,… | followup=function | agents 有: 无"}
```
→ `resolveAgent` 返回了一个**长得像 agent 的对象**（有 `followup=function`），但 `找agent` 仍取不到 → 插件回落到 `找agent(原)` = `undefined` → **`/feiyu/send` 现在会返回"那间会话现在没开着，而且唤醒失败"**。
**这是"冷唤醒到底能不能用"的第二份负面证据（第一份是 speak-up 的"不自动唤醒"）。DSH-Go 要依赖这个能力，必须先单独打通它。**

### 7.5 与 dsh-qq-bridge 的冲突

| 项 | feiyu-app | qq-bridge |
|---|---|---|
| 目标会话 | `/feiyu/send` 由调用方传 `sid`，**任意会话** | 固定 `session-429facb2…` |
| 注入方式 | `agent.followup({role:'user', source:{kind:'user'}})` | `agent.followup({role:'user', source:{kind:'qq'}})` |
| 出站 | 无（只回 HTTP） | 把 assistant 末句发到 QQ |

→ **两个插件都往同一间会话 followup，但 source.kind 不同。** 后果（源码可推，未实测）：
- 心情插件按 `source.kind` 判：`'user'` = 本人说话（正常判）；
- qq-bridge 出站会把这轮 assistant 末句发到 QQ —— **如果这条 turn 是手机 App 发起的，它会被 QQ 桥当成"她在说话"转发到 QQ**（除非开 `只回QQ唤醒`，当前 = false）。→ **手机端发一句话，鱼片 QQ 里会收到她的回复。这是真实串台风险（源码可推，NOT_TESTED）。**

### 7.6 §7 状态判定

| 项 | 状态 | 证据 |
|---|---|---|
| 加载 / ping | **PASS（运行）** | `/feiyu/ping` → `{"ok":true,"版本":"0.1.0"}` |
| 读历史可用 | **FAIL（运行）** | 盘上 288 事件的会话，接口返回 `总数:0`；只存内存不落盘 |
| 发消息可用 | **FAIL（运行）** | `/feiyu/debug` → `有活agent:false / 唤醒:"取不到可用 agent"` |
| 鉴权 | **FAIL（源码）** | 无鉴权；CORS `*` |
| 与 qq-bridge 同会话冲突 | **WARN（源码）** | 都 followup 429facb2；`只回QQ唤醒=false` → 手机端发话会漏到 QQ |
| 独立端口 | **无** | 寄生 DSH webServer 57009 |

---

## 8. surfcore（本地 0.7.10）与 dsh-pet 0.3.8

### 8.1 谁控谁：**互不认识，两个独立桌宠，两条独立调度**

源码证据（全树 grep）：
- `dsh-pet` 源码出现 `surfcore`：**0 次**；出现 `feiyu-mood` / `心情`：**0 次**
- `surfcore` 源码出现 `dsh-pet`：**0 次**
- `dsh-pet/lib/index.js` 无 `followup`、无 `ctx.agents` —— **它根本不进 agent 会话**，只做 web 挂件 + `ctx.llm` 碎碎念（whisper）+ 余额/工作状态。

`~/.dsh/plugins/surfcore/docs/PET.md`（15 KB，已读）说清定位：**surfcore 的桌宠是"独立栏目"，装 SurfCore 不会自动装桌宠；卸载只删 `~/.dsh/surfcore/pet/app`**。

| | dsh-pet 0.3.8 | surfcore 的桌宠 |
|---|---|---|
| 形态 | DSH 插件（web 半侧，`/dsh-pet-7340/*`） | **独立 Electron 进程**（`~/.dsh/surfcore/pet/app` + 自备 `electron.exe`） |
| 进 agent 会话 | 否 | 否（`PET.md` §3 明确"对话不会走 DSH 的完整 agent 回合"，直接调 `ctx.llm`） |
| 谁写记忆 | 不写 | 写自己的"记忆卡"（`~/.dsh/surfcore/memory/` + `backups/*.json`，每 10 分钟一份，最新 2026-11-02T14:49） |
| 主动搭话 | whisper（碎碎念，独立节奏） | purpose 机制（`purpose-outreach.json`） |
| 两者关系 | **零耦合，谁都不控谁** | 同上 |

### 8.2 dsh-pet 实测是活的

`GET /dsh-pet-7340/state` →
```json
{"counter":1793630439625,   // 每秒在变
 "sections":{"balance":{"data":{"ok":false,"provider":"workbuddy","reason":"unsupported"}},
             "workStatus":{"data":{"state":"working","task":null}},
             "notify":{"data":{"type":"session/event","event":{"type":"turn/end"},"sessionId":"session-dd762176-…"}}}}
```
→ 挂件在跑、在收会话事件、余额接口返回 `unsupported`（provider 是 workbuddy，查不了余额，**已知降级**）。
配置 `~/.dsh/dsh-pet/main-config.jsonc`：`id:main / name:"蓝毛小女仆" / size:462 / display:web / corner:top-right / whisperEnabled:true / balanceEnabled:true / workStatusEnabled:true`。
⚠️ **`whisperEnabled:true` = 第三个"主动说话"通道，与 speak-up、surfcore purpose 并存，互相不知道**（源码无任何互斥逻辑）。

### 8.3 surfcore 当前是不是"空架子"：**基本是，但有一台还在空转的调度器**

`~/.dsh/surfcore/config.json`：
```json
{ "platforms": { "start": { "qq": false, "bili": false } },
  "bridge": { "autoStart": false },
  "updates": { "enabled": false, "checkOnStart": false } }
```
`plugin.log` 每次启动都是同一行：`生物钟未启动：B站 启动开关关闭（设置 → B站 → 启动 B站）`。

但 purpose 子系统**开着**：
```
GET /dsh-surfcore/pet-api/purpose/status
{"carrier":"pet","enabled":true,"online":false,"chatEnabled":true,"fallbackToQQ":false,
 "lastTickAt":1793630600528,
 "state":{"countToday":0,"dailyMax":4,"lastSentAt":1793325347022,"lastOutcome":"sent",
          "lastText":"主人，肥鱼最近有点想学新东西了，还没想好学什么……你有没有什么觉得适合我的？",
          "lastPurpose":"最近想学点新东西，想跟主人说说","lastEmotion":"curious",
          "nextFireAt":1793340792359,"nextFireText":"10-30 14:13"}}
```
关键点：
- `carrier = "pet"` → 目标载体是桌宠；而桌宠 **`running:false`、pid 31524 已 DEAD**（`Get-Process -Id 31524` → DEAD；`pet.pid` 里还留着 31524；`start.log` 末尾两次 `[chat] surfcore offline: fetch failed`）。
- `fallbackToQQ = false` → 桌宠不在时**不回落**到 QQ，这一轮 = 什么都不发。
- `nextFireAt = 1793340792359` = **2026-10-30T06:13**，已过去 3 天。
- 最后一次真的说出去的是 **2026-10-30T01:55:47**（`plugin.log`：`[pet-purpose] 桌宠主动搭话（text）：主人，肥鱼最近有点想学新东西了…`）。

→ **结论：surfcore = 一个关掉了 QQ/B站、桌宠进程已死、只剩一个 purpose 调度器在空转的架子。** 它和 speak-up 是**两个互不知情的调度器**（用户明确要求"不要分裂"，这条是现存违规实现）。

### 8.4 §8 状态判定

| 项 | 状态 | 证据 |
|---|---|---|
| dsh-pet 加载并运行 | **PASS（运行）** | `/dsh-pet-7340/state` counter 每秒在变 |
| dsh-pet 余额功能 | **WARN（运行）** | `{"ok":false,"provider":"workbuddy","reason":"unsupported"}` |
| dsh-pet whisper（主动碎碎念） | **NOT_TESTED** | 开关 true，但没等到它的输出，也不能改配置去等 |
| surfcore 主插件加载 | **PASS（运行）** | `plugin.log` 有 11-02 13:39 的 `已激活 v0.7.10` |
| surfcore「空的架子」 | **PASS（运行，即"确实是空架子"）** | qq/bili/bridge/updates 全 false；桌宠 pid 死；purpose 空转 |
| surfcore 桌宠进程 | **FAIL（运行）** | pid 31524 DEAD；`status.pet.running:false`；`start.log` → `surfcore offline` |
| surfcore purpose 调度器在跑 | **WARN（运行）** | `lastTickAt` 几十秒前；但 `nextFireAt` 已过期 3 天、`lastSentAt` 停在 10-30 |
| surfcore 与 dsh-pet 的关系 | **PASS（源码）** | 双向 grep 0 命中，互不相干 |

---

## 9. dsh-companion-loop v0.1.0 —— 找不到

### 9.1 搜索范围与结果（全部只读）

| 搜索 | 方法 | 结果 |
|---|---|---|
| 文件名含 `companion-loop`（`~/.dsh` 递归 4 层） | node fs 递归 | **0** |
| 文件名含 `companion-loop`（DSH 安装目录递归 4 层） | node fs 递归 | **0** |
| 文件名含 `companion-loop`（default-workspace 递归 3 层） | node fs 递归 | **0** |
| 文件名含 `companion`（`Documents\Codex`、`Downloads`、`D:\`） | PowerShell `-Filter *companion*` | 只有无关项（`D:\Coopanion\data\home\companion`、VS Code 扩展 `js-debug-companion`） |
| **文件内容**含 `companion-loop`（`dsh-go-handoff` 全树 .md/.txt/.json/.log） | node 全文 grep | **0** |
| 文件内容含 `companion`（`dsh-go-handoff` 全树） | 同上 | **0** |
| npm 全局 | `%APPDATA%\npm\node_modules` 不存在；`%LOCALAPPDATA%\pnpm` 只有 `store` | 无 |
| `~/.dsh/plugins` | 目录列表 | 只有 `surfcore` |
| `C:\Users\敖震宇\dsh-plugins-staging` | 目录列表 | 9 个 tgz，**无 companion** |
| npm registry | `https://registry.npmjs.org/dsh-companion-loop` | **404 Not found** |
| GitHub | `api.github.com/search/repositories?q=dsh-companion-loop` | **total_count: 0** |
| GitHub | `api.github.com/search/repositories?q=companion-loop+deepseek-harness` | **total_count: 0** |

### 9.2 结论

**"dsh-companion-loop v0.1.0" 在这台机器上、在任何我能只读访问的位置，都不存在。既没有安装、没有源码、没有任何文档提到它，公共 registry/GitHub 也搜不到。**

→ 明确写：**NOT_TESTED，无 Android 真机验收证据。**（更准确地说：**连"待验收的东西"本身都找不到**。）若它确实存在于某个我没权限/没想到的位置，请 lead 给出路径，我可再做一次定向核验。

### 9.3 §9 状态判定

| 项 | 状态 | 证据 |
|---|---|---|
| 装在哪 | **找不到** | 上表 11 条搜索全部 0 命中 |
| 是否安装 | **FAIL（不存在）** | npm 404；profile 里无此包；`~/.dsh` 无此目录 |
| 有无真机验收记录 | **NOT_TESTED（无证据）** | 无 |

---

## 10. 一句话总结

**现在这套东西：唯一"真的活、而且真的在产生效果"的是 dsh-pet 桌宠挂件（每秒在跳）和 dshmarket / dsh-context / dsh-cost-meter / 官方 bundle 这些基础设施；记忆、心情、主动发言、QQ 这四条"大肥鱼人格链路"全部处于"插件在跑、效果为零"或"通道断开"的状态——记忆库对大肥鱼是空的（写入 0 / 召回 0）、心情最后一次真实变化在 3.5 天前且只有一行文字注入、主动发言总开关是关的且不具备冷会话唤醒、QQ 因为 NapCat 根本没起而整条离线；dsh-feiyu-app 是唯一具备移动端形状的入口，但无鉴权、历史只存内存、冷唤醒实测失败。**

---

## 附 A. NOT_TESTED 总清单（14 项，全部有原因）

| # | 项目 | 为什么没测 |
|---|---|---|
| 1 | 情绪是否真的改变了说话方式/语气 | 需要"同一句 × 有/无心情注入"对照实验，本次只读，不能构造 |
| 2 | 情绪注入是否真的进了模型上下文 | 429 会话 288 事件里 0 条 `feiyu-mood` section；只有 diag 一行，无法闭合。需实机复验一轮并抓 prompt |
| 3 | 情绪是否影响话题选择 | 设计上没有这条通道（源码 0 命中），且无对照实验 |
| 4 | speak-up 主动发言的自然度/频率 | 总开关 false，只读条件下不能开 |
| 5 | speak-up 触发文案是否还会"语境错位" | 4 次真实触发在 10-30，模型侧表现未人工验收 |
| 6 | dsh-pet whisper 实际输出 | 开关 true，但没等到它自己说一句（只读，不能手动触发） |
| 7 | `@liustack/modsearch` 是否加载 | 无路由、无日志、无工具名，三条证据都缺 |
| 8 | `dsh-flash-presets` 是否生效 | 只插 agent preset 行，无探针 |
| 9 | `dsh-startup-screen` 是否渲染 | 纯 client 侧，无服务端可观测点 |
| 10 | `cc-safety-net` 是否生效 | 无探针 |
| 11 | 官方 `experimental-voice-input-bundle / auto-review / inspector-profile` 是否可用 | 无探针；只验证了 asar 内 bundle 声明完整 |
| 12 | feiyu-app 与 qq-bridge 的串台风险 | 源码可推（source.kind + `只回QQ唤醒=false`），但"手机发一句 → QQ 收到"没实测（不能发消息） |
| 13 | 记忆"被用上了"（consumed） | diag 该行 0 命中 |
| 14 | dsh-companion-loop 的任何东西 | **连包都找不到** |

## 附 B. 关键证据路径索引

| 内容 | 路径 |
|---|---|
| 版本/运行时 | `…\DeepSeek Harness\resources\runtime\primary-runtime\runtime.json` |
| 崩溃日志（7 个） | `%APPDATA%\@deepseek-ai\dsh-desktop\logs\crash-*.log` |
| 31 bundle 清单 | `~/.dsh/profiles/desktop/package.json` → `dsh.profile.bundles` |
| 精确安装 commit | `~/.dsh/profiles/desktop/pnpm-lock.yaml`（L42-79 依赖图 + L896-1049 resolution） |
| 兼容性判定实现 | `app.asar\dsh\node_modules\@deepseek-ai\dsh-app-boot\lib\index.js` L279-284 / L311-326 / L1025-1026 |
| PTC 模式实现 | `app.asar\…\@deepseek-ai\dsh-tools\lib\index.js` L954-955 / L2614 / L2763 |
| 记忆源码 / 诊断 | `node_modules\dsh-fish-memory\lib\index.js`；`~/.dsh\fish-memory-diag.log`；`~/.dsh\fish-memory\session-*.json` |
| 心情源码 / 状态 | `node_modules\dsh-feiyu-mood\lib\index.js`；`~/.dsh\feiyu-mood\{config.json,state.json,diag.log}` |
| 主动发言源码 / 状态 | `node_modules\dsh-speak-up\lib\host.js`；`~/.dsh\speak-up-config.json`；`GET /dsh-speak-up/state` |
| QQ 桥源码 / 状态 | `node_modules\dsh-qq-bridge\lib\host.js`；`~/.dsh\qq-bridge-config.json`；`GET /dsh-qq-bridge/state` |
| 手机端后端 | `node_modules\dsh-feiyu-app\index.mjs`（源在 `C:\Users\敖震宇\feiyu\plugin`） |
| surfcore 桌宠文档 | `~/.dsh\plugins\surfcore\docs\PET.md` |
| 会话原文 | `~/.dsh\sessions\--C-Users-…--\<sid>\session.v4.jsonl.zstd` |
| 本次解压样本 | `D:\code\temp\s429.jsonl`（429 会话，288 事件 / 665 KB） |

**会话文件解压方法（已验证可用，只读）**：`session.v4.jsonl.zstd` 是 **116 个连续 zstd frame** 拼起来的，`zstdDecompressSync(整个 buffer)` 只能解出第一帧（242 字节，就是 session header）。必须按 magic `28 B5 2F FD` 切帧后逐帧解：

```js
const buf = readFileSync(p); const idx=[];
for (let i=0;i<buf.length-3;i++)
  if (buf[i]===0x28&&buf[i+1]===0xb5&&buf[i+2]===0x2f&&buf[i+3]===0xfd) idx.push(i);
let out='';
for (let k=0;k<idx.length;k++){
  const s=idx[k], e = k+1<idx.length ? idx[k+1] : buf.length;
  try { out += zlib.zstdDecompressSync(buf.subarray(s,e)).toString('utf8') } catch {}
}
```
