# Lead 自撰草稿（架构 / 协议 / 数据 / 唤醒）— 供最终文档拼装

> 本文件只记录 Lead 亲自核实过的设计结论。凡标 [E-xx] 的见 05-LEAD-EVIDENCE.md。

## 1. 六个子系统与唯一真源（Single Source of Truth）

| 子系统 | 唯一真源（现状） | 存储/接口 | 状态 |
|---|---|---|---|
| 人格 | surfcore 角色卡 `~/.dsh/surfcore/characters/<id>/persona.md`（11803 字节，当前角色 char-mvuaxf54「大肥鱼」）+ dsh-qq-bridge 配置里注入的 `人设` 字段 | 两份！**重复** | 需统一 |
| 记忆 | dsh-fish-memory 1.4.3 `~/.dsh/fish-memory/<sid>.json`（version 2，memories[]/traces[]/policy） | remember 工具 + systemPrompt.context | 可用但有坑 |
| 情绪 | dsh-feiyu-mood 0.8.1 `~/.dsh/feiyu-mood/state.json`（version 2，moods.<slot>.{mood,dims,timeline}） | HTTP `/dsh-feiyu-mood/state` + systemPrompt.context | 可用 |
| 主动发言 | **两个都在抢**：dsh-speak-up 0.2.7（总开关=false）与 surfcore 的 `[pet-purpose]` 调度器（会自己 tick） | 见 §3 | **必须收敛到一个** |
| 信息源 | dsh-bili-subagent 0.8.7 + surfcore B站自动化（当前关闭）+ `~/.dsh/bili-duty/mcp_server.py` | 工具 | 需确认唯一出口 |
| 交互 | dsh-pet 0.3.8（web 网页半侧）+ surfcore pet（Electron 独立窗，pid 已死） | HTTP /feiyu/*, /dsh-surfcore/pet-api | 两个桌宠并存 |

## 2. 记忆数据结构（实测 schema，session-2c93cd4f 样本）

```jsonc
{
  "version": 2,
  "policy": { "baseHalfLifeDays":30, "emotionHalfLifeDays":7, "baselineWeight":0.55,
              "activationWeight":0.25, "importanceWeight":0.15, "emotionWeight":0.05,
              "threshold":0.3, "maxItems":6, "maxChars":800, "dailyQuota":8 },
  "memories": [{
    "id":"mmvw95m9j274", "text":"…", "at":"2026-10-31T10:22:38.839Z", "createdAt":"…",
    "category":"event", "importance":0.7, "emotion":null,
    "tags":["DSH Go R4","主动干预","拦截","设计红线"], "source":"她自己记的",
    "reinforcedAt":null, "lastReinforcedDay":null, "事实键":"dshgor4.拦截实现方式"
  }],
  "traces": [{ "at":1793459432128, "session":"session-…", "query":"…", "picked":[], "rows":[…] }],  // 最多 100
  "deleted": [], "purgeEpoch": 0
}
```
**实测规模：memories 只有 1 条（2026-10-31 写入），traces 100 条已满。**
→ 记忆系统"能写、能记召回轨迹"，但**内容几乎为空**。这不是插件坏了，是没人往里记（或会话开关/工具可见性有问题，见 §4）。

## 3. 情绪数据结构（实测 schema）

```jsonc
{
  "version": 2,
  "moods": {
    "__default__": {
      "mood": 70,                    // 0~100，基线 70
      "dims": {"happy":0,"calm":0,"sweet":0,"heartache":0,"worry":0,"sad":0,"jealous":0},
      "lastReason":"", "lastConfidence":0,
      "injectedFor":0, "lastInjectedText":"", "lastInjectedAt":0,
      "saidAt":[], "lastNotedMood":70, "minor":false, "kind":"",
      "updatedAt":1793535279201, "lastDecayAt":1793629686246, "timeline":[]
    }
  },
  "lastActive":"__default__"
}
```
**实测：mood=70（基线）、dims 全 0、timeline 空、updatedAt = 2026-11-01T12:14:39Z，距今约 30 小时无变化。**
→ 判定链路 `autoJudge=true` 但**没有产生任何情绪变化**。要么没在绑定的会话里说话，要么判定输出解析失败（diag.log 有"判断输出解析失败""判断返回空"记录）。

## 4. 记忆注入链路（源码级）

`dsh-fish-memory@1.4.3 lib/index.js`
- L36 `inject = ['systemPrompt','agents','timer','tools','webServer','sessions','sessionTitle','llm']`
- L4094-4127 `同步工具()`：**1.3.4 起 remember 改成插件级全局注册** `ctx.tools.register(makeRememberTool(ctx))`；总开关关掉才摘掉
- L4145-4154 `tick()` 每 3 秒：`同步工具()` + `ctx.agents.list()` 逐个挂记忆段
- L4269-4285 诊断钩子 `ctx.on('system-prompt/assemble')`：把当轮工具名 dump 进 diag.log
- L3920 `agent.ctx.systemPrompt.context(...)` 注入召回结果
- L3894 常驻提示："这一轮如果有值得长期记住的事…直接调用 remember"

**关键实测：** fiag.log 里 120 次工具快照，103 次「有 remember ✓」，17 次「没有 ✗」。
最后 4 次（2026-11-01T00:49 起）全是「没有 ✗」，且当轮 `assemble 的工具（1 个）：run_code` ——
**说明这些会话跑在 PTC 模式（只有 run_code），记忆插件注册的 remember 在 PTC 模式下不会出现在工具表里**。
这是一条真实能力限制：**PTC 模式下 remember 工具对模型不可见**（不是插件 bug，是呈现模式差异）。

## 5. 会话唤醒（关键技术结论）[E-03][E-04]

```
ctx.sessionController.resolveAgent(sessionId)
  → ApiSessionAgentController.resolve()
    → liveAgent() 命中就直接返回
    → 未命中 → resume(sessionId)
        → ctx.sessionQuery.observeSession(sessionId)
        → resumeObserved() → ctx.agents.resume({ resumeSessionId, agentOptions, setup })
```
- 源码：`app.asar → dsh/node_modules/@deepseek-ai/dsh-api-session-controller/lib/index.js` L210-246 / L370-404
- `ctx.sessionController.prompt({sessionId, content, mode})` 内部先 `resolveAgent`，再 `agent.followup(message)`（L925-950）
- **实测**：对冷的 `session-429facb2-…`（大肥鱼生产会话）调用 resolveAgent 返回 `agent … followup=function`。

→ **结论：DSH 侧已经有完整的冷会话唤醒能力，DSH-Go 不需要提供"唤醒"原语。**
→ **但 dsh-speak-up 0.2.7 只调 `ctx.agents.get()`（L460-475），拿不到就放弃（L693-696 明写"按设计不自动唤醒"）。**
→ 所以主动发言要真正生效，**必须把 speak-up 的发送路径改成先 resolveAgent 再 followup**。这是一个约 20 行代码的小改动，但属于改生产插件，需要用户授权 + 隔离 profile 验证。

## 6. 主动发言现状（两个调度器）[E-07]

| 调度器 | 位置 | 当前状态 | 触发方式 | 能否唤醒冷会话 |
|---|---|---|---|---|
| dsh-speak-up 0.2.7 | `~/.dsh/speak-up-config.json` | **总开关=false** | `ctx.interval` 每 10 分钟掷骰，命中率 25%(峰)/85%(谷)，安静 1~8 点 | ❌ 只用 agents.get() |
| surfcore pet-purpose | `~/.dsh/surfcore/characters/<id>/purpose-outreach.json` | **在跑**（plugin.log 每次启动都打印"已启动：载体=桌宠"） | 自研 drives/goals/pending/history 机制，nextFireAt | 走自己的 Electron 桌宠气泡，不走 agent 回合 |
| dsh-pet 碎碎念 | `~/.dsh/dsh-pet/main-config.jsonc` whisperEnabled=true | **开着** | 每 300 秒（eventsRefreshSec.whisper）调一次当前模型说一句话 | 不需要唤醒（自己调 llm） |

→ **三条主动通道同时在跑**，正是用户担心的"主动发言权分裂"。
→ 收敛方案：**唯一调度中心 = dsh-speak-up**；surfcore 的 pet-purpose 与 dsh-pet 的 whisper 全部退化为"内容源"（只提供候选话题，不自己发）。
→ 但 surfcore 是用户自己的本地代码（`~/.dsh/plugins/surfcore`），改造需用户同意；最小动作是先把它在设置里关掉，避免冲突。

## 7. 移动端接入的现成通路（实测可用）[E-05]

`dsh-feiyu-app@0.1.0`（`C:\Users\<user>\feiyu\plugin\index.mjs`，link 安装）
已实测在线：`GET http://127.0.0.1:57009/feiyu/ping` → `{"ok":true,"版本":"0.1.0"}`

| 方法 | 路径 | 请求 | 响应 | 备注 |
|---|---|---|---|---|
| GET | /feiyu/ping | — | {ok,版本} | 健康检查 |
| GET | /feiyu/sessions | — | {ok,会话:[{sid,消息数}]} | **只有内存里见过的会话** |
| GET | /feiyu/history | ?sid&after&limit | {ok,sid,总数,消息:[{id,role,text,turn,at,kind}]} | 内存环形缓冲，上限 300 条 |
| POST | /feiyu/send | {sid,text} | {ok,说} | 走 agent.followup，source.kind='user' |
| GET | /feiyu/debug | ?sid | {ok,sid,有活agent,唤醒,形状,服务} | 诊断用 |

**三个必须修的缺陷（DSH-Go 接手时要知道）：**
1. 无鉴权、`Access-Control-Allow-Origin: *` → 手机上任何 App/网页都能往大肥鱼会话里塞话
2. 历史只在内存，重启即丢；且实测 11 个会话里**没有** 429facb2（大肥鱼生产会话）→ 手机端读不到主会话历史
3. 它自己判断"有没有 agent"用的是 `ctx.agents.get`（保守），所以 /feiyu/send 对冷会话会**误报失败**，虽然 resolveAgent 其实能唤醒

## 8. dsh-Go 侧能力边界（原文核对）

### 8.1 不写 App 代码就能原生显示的六样（docs/plugins/native-for-free.md）
1. 斜杠命令 `ctx.commands` → 输入框 `/` 菜单
2. 技能 `SKILL.md` → `/` 菜单技能组、技能页
3. 插件配置（schemastery `Config`）→ 原生表单
4. 工具 `defineTool` → 通用工具行（摘要取值顺序 command→file_path→path→pattern→query→url→description→prompt→name）
5. 模型输出 ```dsh-ui 代码块 → 原生图表/指标/对比卡（≤4 KiB、≤40 节点）
6. 工具 `presentationMeta["dsh-go/view"]` → 原生卡片

### 8.2 需要 dshGo 清单的四个插槽（docs/plugins/slots.md）
| 插槽 | 位置 | 每插件上限 | 文字上限 |
|---|---|---|---|
| composer.actions | 输入框上方胶囊行 | 2 | label 10 字 |
| message.actions | 助手消息长按菜单 | 2 | label 12 字 |
| composer.strip | 输入框上方状态条（生成回复时让位） | 1 | text 24 字 |
| panels | 对话详情页「插件」页签 / 全部功能 / 聊天页面板 | 4 | title 16 字 |

宿主 API：`ctx.inject(['dshGo'], ctx => …)`（**必须写 ['dshGo']，不能写 ['dshGo?']，也不能进顶层 inject**）
方法：`on(id,handler)` / `strip(id,dataFn)` / `panel(id,{view,data,actions})` / `refresh(id,scope?)`
回调参数：`scope{sessionId?,turnId?,messageId?,callId?,draft?,clipboard?,message?}`、`caps{protocol,app,locale,surfaces,components}`、`event{surface,id,action?,value?,locale}`
超时 3 秒（媒体 30 秒）；连续 5 次违规 → 本次运行降级成网页版。

### 8.3 声音与字幕（docs/plugins/voice.md，协议 v1.2）
| 能力 | 插件要做 | App 做 |
|---|---|---|
| audio | 主机上生成音频字节 | 从桥接取回手机播放 |
| voice | 什么都不用做 | 手机上听；一开口先自己停声音再通知 |
| micro | 推 {text,progress}，≤240 字，最快每秒 10 次 | 画成一行字幕 |
| overlay | 指定跟哪条 micro | 悬浮字幕，浮在对话顶栏下，带关闭键 |

`media` 只回完整一段（audio/mpeg|wav|ogg|mp4|aac），**不支持流式**；≤16 MiB。
效果：`audio.play/pause/resume/stop`、`voice.start/stop`、`overlay.show/hide`，同 instance 归一个会话。
事件：`audio.ended/stopped/error`、`voice.speechStart/speechEnd/stopped/error`、`overlay.closed`。
App 替用户守着：麦克风每插件问一次、**只在前台**（退后台/锁屏/换对话立刻停）、全 App 同时只有一个声音会话、跟随声音自动停听、语音输入优先。

### 8.4 主机覆盖（docs/plugins/hosts.md）
| | 工具卡 | 胶囊·菜单·状态条·面板 | 网页版插件 |
|---|---|---|---|
| 本机引擎（手机） | ✅ | ✅（桥接随 App 运行时安装） | ✅ |
| SSH 主机 | ✅ | 第二期（桥接还装不上） | ✅ |
| 中转主机 | ✅ | ❌（中转不放行 /dsh-go/*） | ❌ |

### 8.5 本机引擎的真实形态（docs/architecture.md）
Flutter 3.47 + Kotlin；本地运行时 = **Ubuntu 24.04 base + Node + @deepseek-ai/dsh，经 proot 运行**，免 Root。
启动：`dsh web --host 127.0.0.1 --port 3080 --no-open`，从 stdout 解析 token，Dart 用 token 换 cookie。
→ **DSH-Go 的"本机引擎"就是在一台 Android 手机上用 proot 跑真正的 DSH**。
→ 这意味着：**大肥鱼的全部插件（fish-memory / feiyu-mood / speak-up / surfcore）可以原样装到手机上跑**，不需要移植。

## 9. 三方职责切分（最终版）

| 层 | 谁做 | 具体内容 |
|---|---|---|
| 人格 / 记忆 / 情绪 / 调度 / 采集 | **DSH 主机**（电脑或手机本机引擎） | 全部是 Node 插件，两边同一套代码 |
| 原生界面 / 语音 / 系统权限 / 保活 | **DSH-Go App** | Flutter + Kotlin，通过 dshGo 桥接与插件通信 |
| 网页界面 | DSH Web | dsh-pet、feiyu-mood 挂饰等 web 半侧，手机 App 里用网页岛兜底 |
