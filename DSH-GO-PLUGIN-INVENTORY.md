# DSH-GO 插件清单（Plugin Inventory）

> 生成时间：2026-11-02 23:00 (+08:00)
> 基线环境：DSH Desktop **0.2.1-rc.2** / Node **24.21.0** / pnpm **11.28.5** / Cordis **4.0.5** / win32 x64
> 证据来源：`~/.dsh/profiles/desktop/package.json`、各包 `node_modules/<pkg>/package.json`、`~/.dsh/backups/dsh-ecosystem-audit-2026-11-02T2230/MANIFEST.json`
> 状态口径：**PASS**=实测通过 / **WARN**=有限制或部分验证 / **FAIL**=失败 / **NOT_TESTED**=没测过（读代码不算 PASS）

---

## 0. 一句话结论

现有 31 个 bundle 里，**真正跑通核心闭环的只有 4 个**（fish-memory / feiyu-mood / qq-bridge / dsh-pet），
**主动发言 speak-up 是关着的**，**记忆里只有 1 条内容**，**情绪 30 小时没动过**，
**三个"主动通道"同时在跑但互相不知情**（speak-up 关着、surfcore pet-purpose 开着、dsh-pet 碎碎念开着）。
新插件（pet/voice/auto-memory/aura-scheduler）**一个都没装**，全部只做了源码审计。

---

## 1. 已装插件全表（desktop profile，31 个 bundle）

### 1.1 大肥鱼核心（自研，Yuuyuko-uu 系）

| # | 包名 | 版本 | 仓库 | 挂载 id | dsh 兼容声明 | 状态 | 说明 |
|---|---|---|---|---|---|---|---|
| 1 | dsh-fish-memory | **1.4.3** | Yuuyuko-uu/dsh-fish-memory | `memory` | 无声明 | **WARN** | 写入链路通，但 **memories 只有 1 条**；PTC 模式下 remember 工具不可见（见 §4） |
| 2 | dsh-feiyu-mood | **0.8.1** | Yuuyuko-uu/dsh-feiyu-mood | `mood` | 无声明 | **WARN** | 加载正常、HTTP 路由全部 200；但 mood=70 基线、dims 全 0、timeline 空，**30 小时无变化** |
| 3 | dsh-speak-up | **0.2.7** | Yuuyuko-uu/dsh-speak-up | `speak-up` | 无声明 | **NOT_TESTED** | **总开关=false**；且源码只用 `agents.get()`，**无法唤醒冷会话**（见 §5） |
| 4 | dsh-bili-subagent | **0.8.7** | Yuuyuko-uu/dsh-bili-subagent | `bili-subagent` | `dsh>=0.1.7-rc.1, node>=22` ✅ | **NOT_TESTED** | 工具已注册（bili_* 系列可见）；未实测抓取 |
| 5 | dsh-qq-bridge | **1.4.0** | Yuuyuko-uu/dsh-qq-bridge | `qq-bridge` | `dshReleases: {0.2.1-rc.3: compatible}` ⚠️**与当前 0.2.1-rc.2 对不上** | **FAIL（当前离线）** | NapCat `127.0.0.1:3000` 连接被拒；配置总开关=true 但基础设施没起 |
| 6 | dsh-feiyu-app | **0.1.0** | **本地** `C:\Users\<user>\feiyu\plugin`（link） | `feiyu-app` | 无声明 | **PASS** | `/feiyu/ping` 实测 200；但**无鉴权、CORS=***，历史只在内存 |

### 1.2 桌宠 / 视觉

| # | 包名 | 版本 | 仓库 | 状态 | 说明 |
|---|---|---|---|---|---|
| 7 | dsh-pet | **0.3.8** | **PC2005-cloud/dsh-pet**（注意：不是 zhu1090093659） | **WARN** | web 半侧加载；兼容声明覆盖 0.2.1-rc.2 ✅。**与 feiyu-mood 零桥接**（grep mood/心情/feiyu 命中 0 次） |
| 8 | surfcore | **0.7.10** | **本地** `~/.dsh/plugins/surfcore`（link） | **WARN** | 内含第三个桌宠（Electron+PixiJS `pet/app`）+ 第二个调度器（pet-purpose）+ 记忆卡系统。**pid 已死，start.log 报 "surfcore offline"** |
| 9 | dsh-whale-widget | 0.4.2 | MeteorNOX/DeepSeek-Balance-Whale-Widget | WARN | 鲸鱼余额挂件，与宠物视觉并存会重复 |
| 10 | dsh-plugin-wallpaper-engine | 1.4.0 | — | NOT_TESTED | 壁纸引擎，6 个子包 |

### 1.3 平台 / 基础设施

| # | 包名 | 版本 | 状态 | 说明 |
|---|---|---|---|---|
| 11 | dshmarket | 1.66.47 | PASS | 插件市场（`dsh-market` 行） |
| 12 | dsh-context | 0.73.3 | PASS | 上下文管理 |
| 13 | dsh-cost-meter | 1.9.22 | PASS | 费用统计（日志有 OpenRouter 价格刷新超时，降级保留本地价） |
| 14 | cc-safety-net | 2.6.16 | PASS | 安全网 |
| 15 | dsh-better-sidebar | 0.28.0 | PASS | 侧边栏 |
| 16 | @wenaixi/dsh-superpower | 8.0.0 | PASS | 15 个工程技能 |
| 17 | @maci0/dsh-caveman | 0.18.0 | PASS | 压缩输出 |
| 18 | dsh-flash-presets | 0.2.2 | PASS | agent 预设 |
| 19 | dsh-startup-screen | 1.1.2 | PASS | 启动动画 |
| 20 | dsh-emil-skills | 1.0.0 | PASS | 设计技能 |
| 21 | dsh-i-have-adhd | 1.0.2 | WARN | — |
| 22 | @nanmicoder/dsh-agent-teams | 0.1.25 | PASS | 多 agent 团队（node ^22.19 \|\| >=24） |
| 23 | @michengai/dsh-skills-manager | 1.2.9 | PASS | 技能管理 |
| 24 | @liustack/modsearch | 5.10.10 | PASS | 模型搜索 |
| 25 | @deepseek-ai/dsh-subagent-codex | 0.2.1-rc.2 | PASS | 官方 |

### 1.4 Android / 开发工具

| # | 包名 | 版本 | 状态 | 说明 |
|---|---|---|---|---|
| 26 | @zhz8888/dsh-android-emulator-helper | 1.2.0 | PASS | 提供本会话全部 `android_*` 工具 |
| 27 | @zseven-w/dsh-android | 0.1.0-rc.12 | NOT_TESTED | — |

### 1.5 官方 bundle（`@deepseek-ai/*`，非第三方）

`dsh-base`、`dsh-web-app`、`dsh-experimental-voice-input-bundle`、`dsh-experimental-auto-review`、`dsh-experimental-inspector-profile`、`dsh-experimental-tool-worktree`、`dsh-experimental-plugin-market-bundle`、`dsh-experimental-agent-team-profile`

> 注意 `dsh-experimental-voice-input-bundle` **已经装在 profile 里** —— 语音输入可能已有一部分官方能力，需在语音选型时先确认它的边界。

---

## 2. 待评估的新插件（一个都没装，全部只做了源码审计）

| 仓库 | 锁定 commit | star | 最后推送 | 结论 | 详见 |
|---|---|---|---|---|---|
| zhu1090093659/dsh-pet | `9ed843bc` | 60 | 2026-11-02 | 与本地 0.3.8 不同血统（v0.4.14）；**不升级**，先做 mood 桥接 | 10-pet-and-voice.md |
| qishuilalala/dsh-voice-mode | `3524db40` | 25 | 2026-10-30 | 语音主方案候选 | 10-pet-and-voice.md |
| PerryLink/dsh-talk | `6a5f768b` | 18 | 2026-11-01 | 语音备选候选 | 10-pet-and-voice.md |
| Aik358/dsh-auto-memory | `a244cf03` | 179 | 2026-11-02 | **只做对照，不替换**；影子模式 | 20-memory-scheduler-clinic.md |
| ljsysfurryACE/dsh-aura-scheduler | `f9048cd5` | 4 | 2026-09-24 | **不装**（会变成第 4 个调度器）；只吸收机制 | 20-memory-scheduler-clinic.md |
| PerryLink/dsh-clinic | `bd1b3e84` | 0 | 2026-11-02 | 可当**结构验收工具**，不是安全认证 | 20-memory-scheduler-clinic.md |
| ccch1mneyyy/dsh-Go | `8fe1c1cd` | 5559 | 2026-11-02 | App 侧，本次**不改代码** | 30-dshgo-native.md |

---

## 3. 依赖与挂载关系（实测）

```
desktop profile
├── dsh.profile.bundles[]  ← 31 项，顺序 = 加载顺序
└── cordis.patch.yml       ← 只覆盖 agent-default-model / ui-* / llm-pi-ai / android-emulator-helper
每个 bundle 自带 cordis.patch.yml，各自 insert 一行 entry：
  memory / mood / speak-up / bili-subagent / qq-bridge / dsh-pet / dsh-market / feiyu-app / surfcore(3 个 patch)
```

**共享服务的插件（按 inject 声明统计）：**
| 服务 | 谁 inject |
|---|---|
| `systemPrompt` | fish-memory、feiyu-mood |
| `agents` | fish-memory、feiyu-mood、speak-up |
| `webServer` | fish-memory、feiyu-mood、speak-up、qq-bridge、feiyu-app、surfcore、dsh-pet |
| `llm` | fish-memory、feiyu-mood、surfcore、dsh-pet |
| `tools` | fish-memory |
| `timer` | fish-memory、feiyu-mood |
| `sessionController`（`ctx.get`，非 inject） | feiyu-app、speak-up |

→ **`webServer` 上挂了 7 个插件的路由**，路由冲突是首要风险。已知真实冲突：dsh-pet 的 `/chat` 路由无鉴权。

---

## 4. 【重点】PTC 模式下 remember 工具不可见

证据：`~/.dsh/fish-memory-diag.log`
```
120 次工具快照：103 次「有 remember ✓」/ 17 次「没有 remember ✗」
最后 4 次（2026-11-01T00:49 起）全部「没有 ✗」，且当轮：
  assemble 的工具（1 个）：run_code
```
原因：DSH `dsh-tools` 有 `mode: native | ptc` 两种呈现模式（源码 `@deepseek-ai/dsh-tools` README）：
- `native` = 每个工具 schema 都进提示词 → remember 可见
- `ptc` = **只给模型 `run_code` 加一份生成 SDK** → 单个工具名不出现在工具表里

**这不是 fish-memory 的 bug**，但后果是真的：**在 PTC 模式的会话里，模型不会自己调 remember，记忆只能靠自动通道（而 fish-memory 没有自动通道）。**
→ 这就是"记忆只有 1 条"的最可能解释。

**修法（三选一，都需用户授权）：**
1. 把相关会话的 tools 模式切回 `native`
2. 给 fish-memory 增加 PTC 下的 remember 通道（SDK 侧暴露）
3. 引入 dsh-auto-memory 的自动提取（它不依赖模型主动调工具）

---

## 5. 【重点】speak-up 无法唤醒冷会话

源码 `~/.dsh/profiles/desktop/node_modules/dsh-speak-up/lib/host.js`
- L460-475 `找agent(ctx,id)` = **只用 `ctx.agents.get()`**
- L676-697 目标会话逐个判活；`可用.length === 0` 时 L693-696 直接
  `记账('勾的目标会话现在都不能说话（…）—— 按设计**不自动唤醒**，这轮什么都不做')`
- L802-816 定时器 = `ctx.interval(cb, 间隔分钟*60*1000)` —— 固定间隔轮询 + 掷骰
- L763-783 真发 = `await agent.followup(消息)`

而 DSH 本身**有**冷会话唤醒（见 06-LEAD-DRAFT-ARCH.md §5，实测 resolveAgent 对冷会话返回 `followup=function`）。
→ **这是 speak-up 的实现选择，不是 DSH 能力缺失。**
→ 修复成本很低（约 20 行：先 `ctx.get('sessionController').resolveAgent(id)`，拿不到再放弃）。
→ 但属于**改生产插件**，必须先经用户授权 + 隔离 profile 验证。

---

## 6. 版本兼容矩阵

| 插件 | 声明 | 与 0.2.1-rc.2 实际 | 判定 |
|---|---|---|---|
| dsh-pet 0.3.8 | `0.2.1-rc.2: compatible` | 命中 | ✅ 声明相符 |
| dsh-qq-bridge 1.4.0 | `0.2.1-rc.3: compatible` | **未命中当前版本** | ⚠️ WARN（实跑正常，但声明没覆盖） |
| dsh-bili-subagent 0.8.7 | `dsh >= 0.1.7-rc.1` | 命中 | ✅ |
| @zhz8888/dsh-android-emulator-helper 1.2.0 | `>=0.2.1-rc.2` | 命中 | ✅ |
| dsh-context 0.73.3 | `>=0.1.7-rc.2 \|\| >=0.2.0-rc.2` | 命中 | ✅ |
| dsh-cost-meter 1.9.22 | `>=0.2.0-rc.1 <0.3.0-0` | 命中 | ✅ |
| dsh-fish-memory / feiyu-mood / speak-up / feiyu-app / dshmarket / surfcore | **无 compatibility 声明** | — | ⚠️ 无法静态判定，只能实跑 |
| dsh-Go 桥接 `dsh-go-bridge` 0.4.0 | — | — | 见 30-dshgo-native.md |

---

## 7. 是否建议接入（给 DSH-Go 的结论）

| 插件 | 建议 | 理由 |
|---|---|---|
| dsh-fish-memory | **保留，先修 PTC 可见性** | 已在写、schema 完整，没必要换 |
| dsh-feiyu-mood | **保留** | 数据模型够用；缺的是"情绪驱动表达"，不是插件本身 |
| dsh-speak-up | **保留为唯一调度中心，但先改唤醒** | 用户明确要求不分裂 |
| surfcore pet-purpose | **关掉**（其调度权交回 speak-up） | 否则就是第二个调度器 |
| dsh-pet whisper | **关掉** | 第三个主动通道；且 dsh-Go 官方明确把它当反例 |
| dsh-feiyu-app | **保留为移动端 API 底子，但必须加鉴权 + 落盘** | 现成可用的最短路径 |
| dsh-qq-bridge | 保留，等 NapCat 起来再验 | 当前离线 |
| dsh-bili-subagent | 保留为内容源（只出候选，不许自己发） | 用户要求 |
| zhu1090093659/dsh-pet | **不装新版** | 与本地同源不同血统，升级风险大于收益 |
| dsh-voice-mode / dsh-talk | **二选一**，先隔离 profile 验 | 见 10 号报告 |
| dsh-auto-memory | **不装进生产**，只做影子对照 | 避免记忆库双写 |
| dsh-aura-scheduler | **不装** | 会变成第四个调度器 |
| dsh-clinic | 装到**测试 profile**当验收工具 | 便宜、只读检查 |

---

## 8. 运行时真相补充（来自 40-runtime-truth.md 的实测）

这一节是插件清单的**实测校准**，与 §1 的静态表并存。凡冲突以本节为准。

### 8.1 31 个 bundle 的真实运行状态分布

| 分类 | 数量 | 举例 |
|---|---|---|
| 有运行证据（路由 200 / 自有日志带本次启动时间戳 / 工具在真实工具表） | **23** | fish-memory、qq-bridge、pretty much 全部基础设施类 |
| 插件活着但**效果为零** | 2 | **dsh-feiyu-mood**（判定停更）、**dsh-speak-up**（总开关关 + 不能唤醒） |
| 插件活着但**通道离线** | 1 | **dsh-qq-bridge**（NapCat 未启动） |
| 接口在但**能力虚** | 1 | **dsh-feiyu-app**（无鉴权 + 历史只在内存） |
| 缺探针 / 未验证 | 8 | modsearch、flash-presets、startup-screen、cc-safety-net、官方 voice-input / auto-review / inspector-profile 等 |

### 8.2 记忆的真实规模（**关键**）

| 库 | 记忆条数 | 召回 |
|---|---|---|
| `session-2c93cd4f-…`（DSH Go R4 设计会话） | **1** | traces 100 条，picked 全 0 |
| `session-6489a146-…`（本次审计会话） | 0（测试记忆已删） | traces 6 条 |
| **`session-429facb2-…`（大肥鱼生产会话）** | **库文件不存在** | `/list` → `count:0, lastTrace:null` |

全系统 `diag` 里 **434 次召回，426 次「注入 0 条」**（98.2% 空转）。

### 8.3 版本声明机制的真相

`dsh.compatibility.dshReleases` 在 DSH 0.2.1-rc.2 里**根本没被读取**（`app-boot` 只按 `engines.dsh` 判；全树 grep = 0 命中）。

⇒ 判一个插件能不能装，看 `engines.dsh`，不看 `dshReleases`。
⇒ dsh-qq-bridge 的 `dshReleases` 只写 `0.2.1-rc.3`，但它有 `engines.dsh: ">=0.2.1-rc.2"`，所以**照样加载成功**。

### 8.4 DSH 桌面端**没有**逐插件加载日志

只有 crash 日志（7 个），最新一条：
```
crash-2026-10-31T21-20-50-079Z-host.log
Error: dsh desktop host exited with 1073807364: [qq-tool-restrict] restrict 跳过（示例）：pwsh — 
tools.restrict() names unknown global tool "pwsh"; known global tools: …（42 个）
[qq-tool-restrict] 跳过 66/67 个当前版本不存在的工具名
```
⇒ **"插件是否加载成功"只能靠各插件自己的路由/日志反推，没有统一真相源。** 这是验收的一个结构性困难。

---

## 9. 最终建议（一句话版）

| 动作 | 建议 |
|---|---|
| 装新插件 | **0 个到生产**（语音装到 abtest-zseven，auto-memory 装到 abtest-zseven 影子模式） |
| 保留为唯一调度中心 | dsh-speak-up（**先修唤醒**） |
| 保留为唯一提示类说话人 | surfcore `[pet-purpose]`（并关掉 dsh-pet whisper） |
| 记忆 | 保留 fish-memory，**先修召回率**（rarityK/threshold），不替换 |
| 情绪 | 保留 feiyu-mood，**先让 429 会话活起来** |
| 人格 | 收敛成 persona.md 一份 |
| 移动端 | 保留 dsh-feiyu-app 当 API 底子，**必须加鉴权 + 落盘** |
| 桌宠 | Web 侧用现有 dsh-pet；Android 侧等 dsh-Go P3 或自建 |
| 语音 | DSH 侧主选 dsh-voice-mode；Android 侧用 NA 四能力 + 官方 ASR |
