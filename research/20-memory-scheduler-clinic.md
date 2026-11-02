# 20 · 记忆 / 主动调度 / 插件质检 —— 三模块源码审计

> 审计对象与锁定 commit 见 `00-SHARED-FACTS.md` §5。本文只做**只读静态分析** + 本机已有产物的读取；
> 未安装任何插件、未改 `~/.dsh` 下任何生产配置、未向 QQ/B站 发任何消息、未改 DSH-Go 主仓库。
> 术语纪律：**「源码已证实存在」≠「实机跑通」**。凡未经运行验证的，一律标 NOT_TESTED。
> 写作时间 2026-11-02（Asia/Shanghai）。

---

## 0. 结论先行（三句话）

1. **记忆**：dsh-auto-memory 该「补」不该「换」——它与 fish-memory 的存储、工具名、注入面**三处都不重叠**（`~/.dsh/memory/`+`{ws}/.dsh-memory/` vs `~/.dsh/fish-memory/*.json`；`memory_*` vs `remember`），但它在**默认配置下会物理删除会话目录**、会每轮 spawn 子代理，直接装上就是一次高风险变更 ⇒ 必须先以「全线关闭写入 + 只开影子检索」的姿态落地。
2. **调度**：现在机上**同时有三个**主动说话源（speak-up / surfcore `[pet-purpose]` / dsh-pet 碎碎念），其中两个是开着的。aura-scheduler **不能替换 speak-up**——它从头到尾没有一条到 agent 的投递路径（无 `followup`、无 `webServer`、它的 webhook URL 根本没注册）。结论是「吸收 3 个机制、不引入第 4 个调度器」。
3. **质检**：dsh-clinic 可以当**本次插件验收的结构门**（R 包结构 / K cordis 契约 / X 宿主契约），**不是安全认证**（全仓零漏洞/权限/供应链检查）。本机因 npm 不在 PATH，D 组冒烟跑不了，且 R2 会把「没有 npm」误判成「插件缺陷」——这是它的一个真实缺陷，不是我们环境的问题被它正确识别。

---

## A. dsh-auto-memory @ `a244cf03f6f409b655d86e02d6c0a7b86d7496f4`

### A.1 包元数据（`package.json`，已读）

| 项 | 值 |
|---|---|
| name / version | `@a9i5k4/dsh-auto-memory` / **3.3.17** |
| type / main / license | `module` / `lib/index.js` / **BSD-3-Clause** |
| peerDependencies | `@deepseek-ai/cordis: ^4.0.1`（**唯一 peer**） |
| optionalDependencies | `@huggingface/transformers: ^3.7.6`（C2 语义档，可缺） |
| dependencies | **无**（零运行时依赖，只用 node 内置） |
| dsh.bundle.patch | `./cordis.patch.yml` |
| dsh.client | `platform: web`，inject 6 个 `@deepseek-ai/dsh-client-*` / `dsh-api-remotes` —— **有浏览器半边** |

**没有 `engines` 字段**（对照：dsh-clinic 自己声明了 `engines.node` 与 `engines.dsh`）。
**没有 `dsh.compatibility` 字段**——兼容性只靠 `peerDependencies.cordis` 一条走廊表达；对 DSH 宿主版本无任何约束（`peerDependencies` 里没有 `@deepseek-ai/dsh`）。
证据：`package.json` 全文。

`cordis.patch.yml` 只有一行 insert：

```yaml
- insert:
    - id: auto-memory
      name: '@a9i5k4/dsh-auto-memory'
```

### A.2 注册面（代码确认，不是看 README）

`lib/index.js:332-335`：

```js
export const name = 'auto-memory-pre'
export const inject = ['webServer', 'tools', 'systemPrompt', 'subagents', 'llm']
```

`lib/index.js:15639`：`export function apply(ctx, config)`（**同步 apply**，不是 class 插件）。

**事件钩子（`lib/index.js`，6 处）**：

| 事件 | 行 | 干什么 |
|---|---|---|
| `agent/session-start` | 17376 | 登记 runtime、补放 session 事件 |
| `agent/turn-stopping` | 17398 | 测水位 + **600ms 后 spawn 子代理做自动沉淀**（`consolidateTurn`） |
| `agent/pre-step` | 17440 | waterfall，登记 runtime + 每 15s 刷新一次记忆快照 |
| `agent/disposed` / `session/disposed` | 17532 / 17535 | 清理 |
| `session/event` | 17541 | **观察全部会话事件**（`observeSessionEvent`） |
| `tools/result` | 17544 | 观察工具执行结果 |

**注入面（3 条，全部对所有 agent 生效）**：

- `ctx.systemPrompt.context({ name: 'dsh:auto-memory-pre', ... })` —— `lib/index.js:17551`（动态快照，追加在历史尾部）
- `renderMemoryStatic()` 走的 `ctx.systemPrompt.section` —— 见 `lib/index.js:8708` / `:18294`
- M6 Reference Tail 动态 surface —— `lib/index.js:18301` 注释

⚠️ **它没有会话级开关**。全仓搜 `enabledSessions|sessionWhitelist|scopeSessions` → **零命中**；注入回调第一个判断是全局 `injectEnabled`（`lib/index.js:612` 定义，`:17551` 之后的 `text:` 首行消费），只管「开/关全部」。
对比 fish-memory 有 per-session `sessions: {sid: bool}`（`lib/index.js:556-575` `isEnabled`/`setEnabled`，本机 `fish-memory-config.json` 里就是这个结构）。**这是两者最关键的架构差异**。

**工具（19 个，`lib/index.js:18347` 起 `const tools = [`，`:21207` 循环注册）**：

`memory_log` `memory_note` `memory_user` `memory_read` `memory_recall` `memory_maintain` `memory_status` `memory_reflect` `memory_external`
`memory_rules` `memory_consolidate` `memory_procedure` `memory_procedure_list` `memory_expand` `memory_trace`
`calendar_add` `calendar_list` `calendar_done` `calendar_remove`

全部注册在**插件级全局 ctx**（`disposers.push(ctx.tools.register(tool))`，`:21207`），不是 `agent.ctx`。
→ **不依赖会话级挂载，所以不存在 fish-memory 那种「PTC 会话里工具消失」的机制性缺口**（见 B 节）。

**Web 路由**：`/api/dsh-auto-memory/{state,list,file,recall,config,reflect,calendar,summarize,workbench,subagent-probe,greet,notices,models}`，全部 loopback-only（`isLoopbackRequest`，白名单见 `panelAllowedHosts`，`lib/index.js:811` 定义）。

### A.3 存储：本地 Markdown + JSON，**没有数据库、没有向量库**

`lib/index.js:563-567`（`DEFAULT_CONFIG`，`:555` 起）：

```js
userMemoryDir: '~/.dsh/memory',              // 用户级 MEMORY.md
projectMemoryDir: '.dsh-memory',             // 项目级（旧位置）
memoryRoot: '~/.dsh/memory/workspaces',      // 集中式：每工作区一个子目录
```

三层（`lib/index.js:15-17` 文件头自述）：

1. 用户级 `~/.dsh/memory/MEMORY.md`（跨项目）
2. 项目笔记 `{ws}/.dsh-memory/MEMORY.md` + 每日日志 `{ws}/.dsh-memory/YYYY-MM-DD.md`（append-only）
3. 白板/账本 `handoff/PLAN.md` + `handoff/*.md`（四段式交接）

派生索引是 **JSON 文件**，不是数据库：`~/.dsh/memory/semantic-pre/l0/l0-index-<wsKey>-<layer>.json`（`l0IndexEnabled` 默认 true）、`~/.dsh/memory/semantic-pre/derived-corpus.json`。
全仓 grep `sqlite|better-sqlite|faiss|chroma|pgvector|lance` → **零命中**（唯一命中是 5522/5543 行注释里提到「历史 SQLite 存在时要谨慎」，指的是 DSH 自己的会话存储，不是本插件）。

**外部服务需求**：

| 档位 | 依赖 | 默认 |
|---|---|---|
| C1 词法 | 无 | —— |
| C2 内置语义 | `@huggingface/transformers` + **e5-small q8 模型 5 文件 / 112.8MB**（`lib/semantic-js.js:577-590`，SHA256 冻结，双源 `hf-mirror.com` / `huggingface.co`） | `semanticEngineMode: 'auto'` |
| Python 高级档 | 本地 `python` + bge-m3 int8 | `pythonBackendEnabled: false` |

**不需要 embedding API、不需要联网**（除首次下模型）。`localWasmPaths: 'auto'`（`:960` 附近）会把 transformers 的 wasm 资产本地化，离线可用。

### A.4 必查功能点逐项判定

| 功能 | 判定 | 证据 |
|---|---|---|
| 自动提取长期记忆 | **源码已证实存在** | `agent/turn-stopping` → `:17398` → 600ms 后 `consolidateTurn`；配置 `autoConsolidate: true`（`:683`）、`autoConsolidateMinChars: 240`、`autoConsolidateCooldownMinutes: 30`、`autoConsolidateDailyMax: 8` |
| 语义检索 | **源码已证实存在（含降级链）** | C2 e5-small（`lib/semantic-js.js`）+ `decideActivationV2`（`lib/semantic-decide.js`，LR+Platt 标定）+ Python bge-m3 档；失败降级到词法（`semanticEngineMode: 'auto'`） |
| 历史对话关联 | **源码已证实存在** | `sessionQuery` 服务（`:4662-4700`）+ 插件内词法兜底（`:4697`）；`memory_recall(scope='sessions')`。**前提：宿主侧会话内容检索要开**（注释 `:6665` 明确写「出厂默认关闭」） |
| 记忆去重 | **源码已证实存在（多道）** | ① `tailHas` 复读防护（`memory_log` 内，`:18387` 附近）② `sanitizeForWrite` 乱码/复读/重复行闸门 ③ `dedupeCardsPre` 看板卡片去重（`:4344`）④ 骨架白板里逐行 `dedupe`（`:5364`）⑤ 同 recordDigest 去重保最高分（M7 摘要 §H-7） |
| 重要性评分 | **源码已证实存在，但只作加权因子** | `lib/memory-importance.js` 全文：`importance = clamp01(0.5 + 0.3*pos − 0.5*correctionRate)`，`pos = 0.5*min(1,distinctSessions/3) + 0.5*min(1,(success+reuse)/4)`；文件头明写「importance 禁止作为唯一排序依据」；接线点 `lib/index.js:9354-9379`（dense 臂因子） |
| 时间衰减 | **源码已证实存在（检索侧，不是存储侧）** | `lib/shadow-retrieval.js` 的 `lexical_pre_v1` 含 recency 项；存储的 Markdown 不删不衰减。**没有 fish-memory 那种半衰期**（fish 是 `baseHalfLifeDays: 30` 的真半衰） |
| 自动召回 | **源码已证实存在** | `activationInboxEnabled: true`（`:924`）+ `activationSource: 'js'`（`:927`）+ M6 Reference Tail 渲染（`:18301`）；inbox claim 四重门见 `docs/M7-INTERFACE-DIGEST.md` §D |
| 重要事项提醒 | **源码已证实存在** | `calendar_*` 4 个工具 + `CALENDAR.md` + `withCalendarLock` 跨进程文件锁（`lib/calendar-lock.js`） |
| 跨会话上下文延续 | **源码已证实存在（招牌能力）** | 水位感知 `waterLevelThreshold`（默认 = `DEFAULT_WATER_LEVEL_THRESHOLD`）+ 白板骨架 + `autoContinue`（默认 **false**）+ `memory_note(kind=handoff)` 四段式账本 |
| 自动技能提炼 | **源码已证实存在，但默认不自动晋升** | `memory_procedure` 工具 + `procedure-store.js` 状态机 observed→candidate→validated→active→deprecated；`procedureInjectEnabled: true`（`:997`）。**注释明写**：「全仓无任何自动晋升代码路径，`applyAutomaticTransitions` 只导出零调用」 |

### A.5 是否会往别的记忆系统写 / 有没有全局副作用

**结论：不会写 `~/.dsh/fish-memory/`，但有三个真正危险的全机副作用。**

写作面**只有**上面 A.3 的三层路径（grep `fish-memory` → 零命中；grep `dsh-auto-memory.json` → 只命中自己的配置 `path.join(dshHome(),'dsh-auto-memory.json')`，`:2379`）。

三个必须知道的副作用：

**① 会 hook 所有会话的 assemble/ingest —— 是的，无条件。**
`session/event` + `tools/result` 观察**全部**会话（`:17541/17544`）；`systemPrompt.context` 对每个 agent 都渲染一次。仅靠 `injectEnabled` 全局关。**没有 per-session 豁免**。

**② 会按天自动归档、并物理删除 `~/.dsh/sessions/` 下的会话目录（默认开）。**

`lib/index.js:820-828`：

```js
sessionArchiveEnabled: true,     // 总开关
autoArchiveEnabled: true,        // 2 天后归档
autoArchiveDays: 2,
autoDeleteEnabled: true,         // 归档后再 7 天物理删除
autoDeleteDays: 7,
autoArchiveCheckMin: 60,
```

实现在 `lib/session-archive.js`（**移植自 `@linxin666/dsh-session-archive` v0.4.1，Apache-2.0**，文件头有完整署名）+ `sessionArchiveSweep`（`lib/index.js:12997`）。
删除动作是真的 `rmSync`：`removeSessionDir(index.byId.get(id), root)`（`:13119`），还会连带删 `storages/session_projcache/sessions/<id>`（`:13121`）和 rdb 里的记录。

**好消息（源码确实做了约束）**：
- 只回收「记忆中枢工作区」，解析不出就整体跳过（`:13052-13057` `hubDirs.size===0 → return {reason:'hub-unresolved'}`，fail-closed）
- 删除前再复验目录属于中枢（`:13112-13115`）
- 归档时间未知的历史归档**永不自动删除**（`lib/session-archive.js` `autoDeleteSeedCandidates` 注释：「这是安全默认值，不得放宽」）

**但这仍然是一条「装上一个第三方插件 → 两天后开始删我会话」的默认路径。** 任何人评估这个插件，第一个要关的就是这三项。

**③ 每轮 turn 结束会 spawn 子代理，并自己 GC 这些子代理会话。**
`:17398` 的 `consolidateTurn` + `runSubagent(...,'auto-memory-consolidate',...)`（`:13414`）。子代理会话由本插件自己的 `subagentGcSweep`（`:13190`）搬进备份目录——`subagentGcEnabled: true` 默认开。
源码注释自承实测：「全机 686 个子代理会话中 638 个来自本插件」（`:816` 附近）。这是真实成本：额度 + 会话列表污染 + 落盘 IO。

**④ 外部记忆「接入」是链接模式，不复制内容。**
`importInto`（`:14792` 起）：只往记忆文档追加一段带 `<!-- external-source: sha256 -->` 的**路径指针**，注释明写「不整段写入内容 / 防止语义朊病毒」。
→ 所以它**不会把 fish-memory 的记忆吸走再写一份**（fish-memory 的 `~/.dsh/fish-memory` 根本不在它的 13 个外部源清单里）。

### A.6 关键文档读到的事实

- `docs/ARCHITECTURE.md`：图基于 **v3.2.10 / `8280500`**，明写「图上存在某项能力，不表示你的安装环境已经启用它」。当前代码是 3.3.17，**架构图比代码旧**。
- `docs/M7-INTERFACE-DIGEST.md`：字段级契约（`ContextPushEnvelopePre` / `validateActivationRequestPre` 16 条硬校验 / claim 四重门 / `SidecarClient` code 全集 / Python 侧 14 条雷区）。这份文档质量很高，**可以直接当接口设计参考**，尤其是「JS 管身份/时序/预算，Python 管语义分」的分工。
- `docs/INTEGRATION-ANALYSIS.md`：作者自己复盘了前期「只读 3 个文件就下结论」的错误，并给出四个参考仓（OpenViking/Hindsight/MetaMem/MemOS）的可吸收性评分。§1.4 的三层 store 接口描述与 `lib/fact-store.js`/`episodic-store.js`/`procedure-store.js` 对得上。
- **文档总量 300+ 份**（`docs/` + `docs/internal/`），但其中大量是内部施工记录（`AUDIT-*`/`HANDOFF-*`/`RESUME-*`）。**信号密度不高**，别被文件数迷惑。

---

## B. fish-memory 1.4.3 对照（本地文件直读）

源码：`C:\Users\<user>\.dsh\profiles\desktop\node_modules\dsh-fish-memory\lib\index.js`（**6501 行**）。

### B.1 数据模型（读 `~/.dsh/fish-memory/session-2c93cd4f-….json`，45748 字节）

顶层键（实测）：

```
version: 2
policy: { baseHalfLifeDays:30, emotionHalfLifeDays:7, baselineWeight:0.55,
          activationWeight:0.25, importanceWeight:0.15, emotionWeight:0.05,
          threshold:0.3, maxItems:6, maxChars:800, dailyQuota:8,
          rarityK:0.8, queryGate:0.8, factsMax:2, unknownActivation:0.5 }
memories: []      // 实测长度 = 1
traces:   []      // 实测长度 = 100（上限）
deleted:  []
purgeEpoch: ""
```

单条 memory 的形状（唯一那条实例）：

```json
{ "id":"mmvw95m9j274", "text":"在 DSH Go R4 的主动干预系统里…",
  "at":"2026-10-31T10:22:38.839Z", "createdAt":"2026-10-31T10:22:38.839Z",
  "category":"event", "importance":0.7, "emotion":null,
  "tags":["DSH Go R4","主动干预","拦截","设计红线"],
  "source":"她自己记的", "reinforcedAt":null, "lastReinforcedDay":null,
  "事实键":"dshgor4.拦截实现方式" }
```

可选字段（代码里见过、这份实例没出现）：`refinedSkip`（提炼判跳过）、`已被取代`（被新条目取代）、`related`（扩词）。

单条 trace 的形状（每次注入一条，**保留最近 100 条**）：

```json
{ "at":1793459432128, "session":"session-2c93cd4f-…",
  "query":"You are the Team Lead named \"lead\".…",
  "gate":null, "gateLimit":0.8, "picked":[],
  "rows":[{"id":"mmvw95m9j274","C":0,"P":0,"omission":"hard_gate",
           "timeHit":false,"status":null}] }
```

`omission` 枚举（`lib/index.js:3766`）：`hard_gate`（没命中线索）/ `below_threshold`（分数不够）/ `limit`（超条数）/ `budget`（字数超）/ `query_gate`（问句没内容）/ `fact_cap`（事实名额满）。
`status`：`assembled`（进了提示词）→ `consumed`（回复里像用上了）。

**实测数据（这份 47KB 文件）**：memories **只有 1 条**，traces 100 条已满。
最后一条 trace 的 query 是「不是让你都转到d盘去吗」，`C=0 / omission=hard_gate` ⇒ **那条唯一的记忆从来没被召回成功过**。
这是「记忆系统在跑，但库里几乎没东西，且唯一的东西召不回」的实证。

### B.2 注册面（代码确认）

`lib/index.js:36`：
```js
export const inject = ['systemPrompt','agents','timer','tools','webServer','sessions','sessionTitle','llm']
```

- **工具：只有 1 个 —— `remember`**（`makeRememberTool`，`:3185`）；参数 `{text, importance, tags, 取代, 事实键}`。
- **没有定时任务业务逻辑**，只有一个 `ctx.interval(tick, 500)` 的**挂载守护**（`:4251`）：0.5 秒扫一遍活着的 agent，把记忆段挂上（`:4145` `tick`）。
- **事件**：`agent/inbox/inserted`（`:3813`）、`agent/turn-stopping`（`:3829`，判"像不像用上了"）、`system-prompt/assemble`（`:4269`，纯诊断 dump）。
- **Web 路由**：`/dsh-fish-memory/{page,list,add,policy,update,…}`。

### B.3 记忆怎么写、怎么召回

**写**：只有一条路——模型主动调 `remember`。没有自动沉淀、没有定时提炼、没有语义抽取。
（`lib/生成档案.js` 是「从会话生成档案」的旁路功能，不写 memories。）

**召回**（`recall(data, query, now, sid)`，`:2877`）：纯**词法**，四步。

1. 过滤：`!m.refinedSkip && !m.已被取代`（`:2888`）
2. **问句门槛**（`:2906-2920`）：`问句信息量(sid, query, df) < policy.queryGate(0.8)` → 直接返回空，**连打分都不做**
3. **只拿稀有词算证据**：`df` 库里出现率 > 1% 的词不算线索；再经通用中文词频表过一道；两条安全垫（剩 <2 个 / 全都价值为 0 就退回全量）
4. 打分：`P = C × (基础 + 鲜活 + 重要 + 情绪)`，门槛卡的是 **C**（证据），不是 P（`:2976-2990` 有很长的注释解释为什么必须分开：鲜活度参与门槛会让整库分数一起下移）

**没有向量、没有 embedding、没有外部服务**。100% 本地字符串相似度 + 词频统计（`词典.txt` 919KB / `词频.txt` 199KB 就躺在那）。

注入：`agent.ctx.systemPrompt.context({ name:'memory', order:900, interpolate:false, text })`（`:3920`）。**按 agent 挂**，per-session 开关控制（`isEnabled(sid)`，`:556`）。

### B.4 「assemble 里没有 remember」——设计如此，还是 bug？

**答案：既不是纯设计，也不是纯 bug，是「PTC 呈现模式 × 会话级挂载」的必然结果。责任在 fish-memory 的架构选择，不在 DSH。**

证据链：

1. `~/.dsh/fish-memory-diag.log` 实测 120 次工具快照：**103 次有 `remember` ✓，17 次没有 ✗**；工具数分布 `{1个:17次, 61个:5次, 91个:39次, 97个:59次}`。**17 次「没有」全部是「只有 1 个工具 = `run_code`」那种快照**。最后 4 次快照全是这种（2026-10-31T16:46、11-01T00:27/00:49/05:26/12:18、11-02T14:08）。
2. 宿主源码确认 `assembly.tools` 就是**模型可见的 schema 列表**：`@deepseek-ai/dsh-system-prompt/lib/index.js:395` `tools: orderTools(collected, this.toolOrder, knownNames)`，而 `collected` 来自 `wireSchemas`。
3. `wireSchemas`（`@deepseek-ai/dsh-tools/lib/index.js:2932-2944`）在 **ptc 模式**下：
   ```js
   if (mode === "native") return { schemas: [...view.visible.values()].map(...), knownNames: [...] };
   this.requirePtcRuntime(mode);
   return { schemas: [...view.visible.values()].map(...).filter(s => s.name === RUN_CODE_NAME), knownNames: [RUN_CODE_NAME] };
   ```
   → **PTC 模式下模型侧 schema 只剩 `run_code` 一个**，其余全部走生成的 SDK（`sdkSchemas`，`:3123`）变成 `tools.xxx()` 绑定。
4. 所以 `:4269` 那个诊断钩子打印的「assemble 的工具（1 个）：run_code」**是 PTC 模式的正常形态**，不是 remember 丢了。诊断钩子看的是**模型侧 wire schema**，而 PTC 模式下那里本来就只有一个 `run_code`。
5. 为什么另外 103 次是 61/91/97 个？因为那些 agent 跑在 **native 模式**（子代理/不同 preset）。

**实证（本会话就是 PTC）**：我在 `run_code` 沙箱里读 `Object.keys(tools)` → **96 个工具，含 `remember`**。
→ remember **确实在场**，只是在 PTC 下以 `tools.remember(...)` 的形式存在，不出现在模型侧 wire schema 里。
→ 换句话说：**诊断行里的「没有 ✗」是误报**，它把「PTC 模式」误读成「工具没挂上」。

**但确实有一个真的挂载缺口**（不是 PTC 造成的）：fish-memory 的 remember 是**双层注册** ——
- 插件级全局：`ctx.tools.register(makeRememberTool(ctx))`（`:4115`，1.3.4 起）
- 按 agent：`at.register(makeRememberTool(ctx))`（`:4229`，1.3.9 为修 issue#15 补的）

按 agent 那份挂在 `agent.ctx` 上，而 `ctx.interval(tick, 500)`（`:4251`）每 0.5 秒才扫一次。源码注释自己写了这个竞态（`:4238` 附近）：
> 窗口内如果先跑了一轮（比如 QQ 桥转进来一条），那一轮的工具列表里就没有 remember

→ 这是**真 bug 类风险**（新会话首轮可能拿不到 per-agent 那份），但**已由插件级全局注册兜住**（`:4110 同步工具` 在总开关开着时保证全局那份在）。本机 `fish-memory-config.json` 是 `{"全局开关": true, "switchMigrated": true, "sessions": {}}` ⇒ 全局那份**在**。

**另一个真问题（值得报）**：`ctx.interval` 不在 `inject` 列表里（`:36` 只有 `timer`）。我用 clinic 跑出来的 K1 就抓到了：
> `[WARN] K1 … ctx.interval（1 处，如 index.js）`

即依赖的是 `timer` seam 提供的 `ctx.interval`，但 inject 只声明了 `timer`——这在 cordis 里是「用了未声明的服务」，行为取决于实现细节。

---

## C. 记忆结论：**补，不换**

### C.1 能不能补上 fish-memory 的短板？能，而且几乎每一块都正好错位互补

| 短板 | fish-memory 现状（实测/源码） | auto-memory 的对应能力 |
|---|---|---|
| **库里几乎没东西** | 唯一那个会话文件 memories=**1 条**；写入只能靠模型主动调 `remember` | 自动沉淀（`turn-stopping` → 子代理提炼，`autoConsolidate` 默认 on，日上限 8）+ `memory_log` append-only 日志 |
| **召回失败率高** | 最后一条 trace `C=0 / hard_gate`，唯一一条记忆从没召回成功；纯词法 | 语义档（e5-small 本地）+ 六级渐进激活（index→hint→excerpt→checklist→resource→full） |
| **没有重要性以外的排序信号** | 有 importance 权重，但只参与 P 的乘数 | evidence 六类计数 → importance（跨会话多样性 + success/reuse，correction 重罚） |
| **没有跨会话延续** | 无（只有 per-session 文件） | 白板 PLAN.md + 四段式交接账本 + 水位感知（招牌能力） |
| **没有历史会话检索** | 无 | `memory_recall(scope='sessions')`（需宿主开 sessionQuery）+ 插件内词法兜底 |
| **不会自动提醒** | 无 | `calendar_*` + `CALENDAR.md` + 跨进程文件锁 |
| **工具在 PTC 下不可见（误判）** | 诊断行「没有 ✗」是 PTC 误报 | auto-memory 的工具也是全局注册，同样面对 PTC；但它的**注入走 `systemPrompt.context`，不依赖工具表** |

**反过来，fish-memory 有三样 auto-memory 没有、也不该丢的**：

1. **per-session 开关 + per-session 文件隔离**（`~/.dsh/fish-memory/<session>.json`）。auto-memory 是**全工作区集中式**（`~/.dsh/memory/workspaces/<wsKey>/`），一旦开就是所有会话共享一份。
2. **真时间衰减**（`baseHalfLifeDays: 30` / `emotionHalfLifeDays: 7`）。auto-memory 只在检索侧算 recency，不衰减存储。
3. **模型零成本的写入路径**（纯字符串，不调模型）。auto-memory 的自动沉淀每次都 spawn 子代理 = 真花钱。

### C.2 要不要替换？**不要。** 三条硬理由

1. **用户明确说不要**（任务书原文）。
2. **auto-memory 默认配置带删除动作**（`autoArchiveDays:2` + `autoDeleteDays:7` + `sessionArchiveEnabled:true`，会 `rmSync` 会话目录）。这不是「多一个记忆源」级别的变更，是「会删我数据」级别的变更。
3. **两者不冲突，替换不会带来净收益**：存储路径、工具名、注入段名全部不同（`~/.dsh/memory/` vs `~/.dsh/fish-memory/`；`memory_*` vs `remember`；`dsh:auto-memory-pre` vs `memory`/`memory-hint`），可以并存。

### C.3 「只读 / 影子模式」怎么落地（具体到开关名与文件）

**原则：先让它跑起来但一个字都不写、一条都不注入；确认无副作用后再逐项打开。**

第 0 步（安装前）：记下当前会话目录清单，作为回滚比对基线。
```powershell
# 只读，不改动
Get-ChildItem "$env:USERPROFILE\.dsh\sessions" -Directory | Select-Object Name,LastWriteTime | Sort-Object LastWriteTime | Out-File "$env:TEMP\sessions-before.txt"
```

第 1 步：装进**独立 profile**（`abtest-zseven` 是空 patch，天然适合当试验田），**不要进 desktop**。
理由：auto-memory 无 per-session 开关，进 desktop = 全部 31 个 bundle 的会话都被观测。

第 2 步：写 `~/.dsh/dsh-auto-memory.json`（配置路径已由代码确认：`lib/index.js:2379` `path.join(dshHome(),'dsh-auto-memory.json')`）。

**影子模式配置（全部值为「关」，一行不改代码）：**

```json
{
  "injectEnabled": false,                 // 关掉动态快照注入（:612 / :17551 的 text 首行）
  "autoConsolidate": false,               // 关掉每轮 spawn 子代理提炼（:683 / :17398）
  "consolidateScheduleEnabled": false,    // 关掉每日 09:30 定时固化
  "maintainScheduleEnabled": false,       // 关掉每日 10:00 定时 30 天蒸馏
  "handoffEnabled": false,                // 关掉 PLAN.md / 账本读写
  "waterLevelAdvisory": false,            // 关掉水位提醒注入
  "waterLevelAutoHandoff": false,         // 关掉水位骨架账本
  "autoContinueEnabled": false,           // 已经是默认 false，显式写死

  "sessionArchiveEnabled": false,         // ★★★ 必须关：会话归档总开关（:818）
  "autoArchiveEnabled": false,            // ★★★ 必须关：2 天自动归档
  "autoDeleteEnabled": false,             // ★★★ 必须关：7 天物理删除
  "subagentGcEnabled": false,             // 不搬移任何子代理会话
  "autoSummaryTimes": [],                 // 不做时段总结
  "welcomeTourEnabled": false,            // 不弹首启向导

  "rulesLayeringMode": "off",             // 不动规则段注入
  "promptSectionToggles": {},             // 保持默认
  "tier0CatalogEnabled": false,           // 不注入 Tier-0 目录

  "associativeMemoryEnabled": true,       // ★ 影子检索的前提门（:907，默认 false）
  "shadowRetrievalEnabled": true,         // ★ 影子检索：只记录候选、不投递（:911，默认 true）
  "contextBridgeEnabled": true,           // 只做观测/证据（:914，默认 true）
  "activationInboxEnabled": false,        // ★ 关掉「投递」（:924）——这是唯一会让记忆进对话的门
  "activationSource": "js",

  "l0IndexEnabled": false,                // 不建 L0 索引（省 IO）
  "pythonBackendEnabled": false,          // 不启 Python
  "semanticEngineMode": "lexical",        // 不下载 112MB 模型，纯词法
  "externalSources": {},                  // 一个外部源都不扫（13 个源全部置空）

  "boardMode": "legacy",                  // 白板走旧档（默认是 graph，新版会多建 sidecar）
  "memoryRoot": "~/.dsh/memory/workspaces",
  "userMemoryDir": "~/.dsh/memory",
  "projectMemoryDir": ".dsh-memory"
}
```

**这样做的语义**（严格按代码）：`activationInboxEnabled:false` ⇒ `activation-host` 的 `effectiveEnabled()` 恒 false ⇒ pre-step claim、reference-tail 渲染、状态三处全早退（`lib/index.js` 里 M6 注释明确写这一点，「结果永不投递给对话」）。`injectEnabled:false` ⇒ 唯一那条动态注入返回 `''`。`autoConsolidate:false` ⇒ 不 spawn。三个 archive/delete 全 false ⇒ 不碰 `~/.dsh/sessions`。
**净结果：它只会读会话事件、算候选、写自己的诊断/影子日志，不向任何记忆文件写、不往提示词里塞一个字、不删任何东西。**

第 3 步：观测 3–7 天，只读地看这几个文件（全部落在自己的目录，不碰 fish-memory）：
- `~/.dsh/memory/polling-heartbeat.json` —— 心跳在多长时间内写过（`:21176` 附近的 `writeHeartbeat`，15s 一次）
- `~/.dsh/memory/` 下有没有 `MEMORY.md` / `workspaces/` 冒出来（影子模式下不该有内容增长）
- `~/.dsh/sessions/` 目录数与第 0 步基线比对（**必须完全一致**）
- 影子检索候选落在 `lib/shadow-retrieval.js` 相关路径（该模块是纯函数，落盘由 host 侧决定）

第 4 步：逐项打开，每次只开一个，观察 1 天：
`shadowRetrievalEnabled` → `l0IndexEnabled` → `semanticEngineMode:"auto"`（首次会下 112MB 模型）→ `activationInboxEnabled`（**此时才会真正注入，这一步是"功能上线"**）→ `handoffEnabled` → `autoConsolidate`（**最后开，因为要花钱**）。
`sessionArchiveEnabled`/`autoArchive*`/`autoDelete*` **建议永远保持 false**——DSH 自带会话归档就用官方的，不需要第三方插件代劳。

**回滚**：`rm ~/.dsh/profiles/abtest-zseven/node_modules/@a9i5k4` + 从 profile `package.json` 的 `dsh.profile.bundles` 删掉一行（**不需要碰 desktop**）。影子模式下没有产生任何需要回滚的记忆数据。

### C.4 一个必须澄清的既有事实

本机 `~/.dsh/memory/` **不存在**（`existsSync` = false），profile 里也**没有** `@a9i5k4`。⇒ auto-memory **当前完全没装过**，A 节全部结论都是纯静态分析（见 NOT_TESTED）。

---

## D. dsh-aura-scheduler @ `f9048cd59399dbc2b38b0d3be4a750342b8b72ca`

仓库 16 文件，全部读完（`src/index.ts` 260 行 / `src/sensors.ts` ~120 行 / `src/webhook.ts` ~110 行 + `lib/index.js` 编译产物 + `tests/smoke.ts`）。

### D.1 元数据

| 项 | 值 |
|---|---|
| name / version | `@agentframe/dsh-aura-scheduler` / **0.2.0** |
| **license** | **GPL-3.0** ⚠️（对照：fish-memory MIT、speak-up MIT、auto-memory BSD-3-Clause、clinic Apache-2.0） |
| peerDependencies | `@deepseek-ai/cordis: >=4.0.0`（**open-top 区间**，会放行 5.0/1.0 任何未来版本） |
| dependencies | 无（运行期只 import `@deepseek-ai/schemastery`） |
| **没有 LICENSE 文件** | clinic R0 实测 WARN：「缺 LICENSE 文件」 |
| 导出形状 | `export default AuraSchedulerService`（**class**，无 `name`/`apply` 导出） |

**GPL-3.0 + 我们要「吸收它的机制到 surfcore」= 许可证传染风险。** 这本身就是「不装、只借鉴思路、不复制代码」的独立理由。

### D.2 它到底能不能让 agent 说话？**不能。**

这是这一节最重要的结论。三条独立证据：

1. **没有任何投递调用**：全仓 grep `followup` → **零命中**；grep `prompt` → 只命中注释。它从不接触 `agent` 对象。
2. **它的「投递」只是 emit 一个 cordis 事件或向一个 URL POST**：
   `src/index.ts` 的 `_maybeProactive`：
   ```ts
   const cb = this.config.onProactive
   if (cb) {
     if (cb.startsWith('http')) await fetch(cb, { method:'POST', body: JSON.stringify({event:'proactive', ...event}) })
     else this.ctx.emit(cb as any, event)
   } else { this.ctx.logger.info(`[aura] proactive #${this.proactiveCount}`) }
   ```
   → `onProactive` 为空（`DEFAULT_CONFIG.onProactive: ''`）时，**唯一结果是一行日志**。
3. **它的 webhook URL 是死的**：`addWebhook` 返回字符串 `/aura-webhook/<name>?token=<tok>`，但这个路由**从未注册到 `ctx.webServer`**（全仓 grep `webServer` → **零命中**）。`WebhookHub.handle()` 是一个纯函数式入口，必须有人替它接 HTTP 请求。**没有那一层，外部系统 POST 到那个 URL 会 404。**
   → README 里「外部系统 POST 唤醒 Agent」这句话，在当前源码下**不成立**。

**所以 aura 只是一个「决定什么时候该说话」的计算器 + 一个没人接的事件源。它不能替代 speak-up；它连「说话」这一步都没有。**

### D.3 它有什么，逐项判定

| 项 | 判定 | 代码位置与事实 |
|---|---|---|
| 主动发言调度 | **源码已证实存在（决策层）** | `start()` → `setInterval(_tick, 60_000)`（`src/index.ts:107`） |
| 心跳 | **源码已证实存在** | 同上，60 秒；`heartbeatCount++`，`status()` 暴露 |
| 定时触发 | **源码已证实存在** | 同上 + `tickNow()` 强制触发 |
| webhook | **源码已证实存在（库），但无 HTTP 接入** | `src/webhook.ts` 全文：HMAC-SHA256 `createHmac('sha256',secret).update(`${name}:${bodyText}`)` + `timingSafeEqual` + 60 次/分钟限流 + 404/401/429/500 五种返回 |
| 打扰成本 | **源码已证实存在，但是硬编码常量** | `_interruptionCost() { return 0.3; }` —— 注释写「Placeholder: could read session activity / user presence」 |
| 话题价值评分 | **源码已证实是装饰性的** | `relevance` 是**配置里的常数字段**（默认 0.5），不是从内容算的。`V = α·(idleRatio) + β·relevance − δ·0.3`；`idleRatio = min(elapsed/maxInterval, 1)`。展开后 `V = α·elapsed/maxInterval + const` ⇒ **本质就是一个带阈值的定时器** |
| 触发冷却 | **源码已证实存在** | `now - lastProactiveAt < cooldown*1000 → false`（`cooldown` 默认 600s） |
| 用户活跃状态 | **不存在** | `_interruptionCost` 的 Placeholder；没有任何 `agent.status` / 会话活跃度读取 |
| 用户活跃/静默时段 | **源码已证实存在（时钟窗口）** | `_inQuietHours()` 用 `new Date().getHours()`，`[23,8]` 支持跨午夜 |
| off-peak 省钱窗口 | **源码已证实存在** | `_inOffPeak()` + `offPeakOnly`，默认 `[2,7]` |
| 传感器（file/process/http/port） | **源码已证实存在** | `src/sensors.ts`：`_probeFile`（stat + ≤4096B 读 + regex）/ `_probeProcess`（`pgrep -f`，**Windows 没有 pgrep**）/ `_probeHttp`（fetch + 8s abort）/ `_probePort`（net.connect + 3s） |
| 传感器触发语义 | **源码已证实存在** | `probe()`：**只在状态翻转时 fire**（`prev.state !== current.state`），首次探针只做基线不 fire |

### D.4 三个实测缺陷（clinic 跑出来的，不是猜的）

在 `_cache/staged_aura`（原样复制的 16 文件）上跑 clinic：

```
[WARN] R0 基础字段 —— 缺 LICENSE 文件
[WARN] R4 入口契约 —— 入口未检出 apply 导出 / 未检出 name 导出
[WARN] R8 peer 范围 —— >=4.0.0 是 open-top 区间
[WARN] K3 定时器生命周期 —— index.ts:83/84/107/168 定时器疑似未包进 ctx.effect
[SKIP] X1/X4 —— 未声明 @deepseek-ai/dsh* peer / 源码未见数组形态 inject
```

逐条核实（**K3 是真缺陷，不是误报**）：

- `start()` 里 `this.timer = setInterval(...)`（`src/index.ts:107`）**没有包进 `ctx.effect`**。Cordis 的规矩是「Cordis API 之外的资源必须包在 ctx.effect 中」，否则插件卸载/HMR 后定时器不回收。
- `_startSensors()` 更糟：`const id = setInterval(probe, interval)` 存进 `(this as any)._sensorTimers`（`src/index.ts:168-172`），而 `stop()` 只清 `this.timer` 和 `this.sensorTimer`——**`sensorTimer` 这个字段从未被赋值**，`_sensorTimers` 从不被清。⇒ **传感器定时器永久泄漏，stop() 无效。**
- `_probeProcess` 用 `pgrep -f`：**Windows 上不存在这个命令**，`exec` 报错 → `alive=false` → 永远 `absent`。process 类传感器在本机（win32）100% 失效。
- R4 的 warn `clinic` 自己标了「若无 apply 导出，可能是 default 对象/class 形式，请人工确认」——我核实过：class 形态是合法的（loader 的 `isConstructor` 分支会 `new callback(ctx, config)`，见 `@deepseek-ai/cordis/lib/index.js:1066-1071`）。**这是 clinic 的假阳性，它自己也知道。**

### D.5 一个可能真正炸的点（静态推断，未实跑）

`src/index.ts` 构造函数：

```ts
;(ctx as any).provide?.('aura')      // ← 只传了 name，value = undefined
ctx.set('aura', this)
```

`@deepseek-ai/cordis/lib/index.js:800` `provide(name, value, check)` 注册时 `value` 是 `undefined`；紧接着 `:782` `set(name, value)` 检查 `impl.fiber !== this.ctx.fiber → throw`——同 fiber，能过。
**静态看这条路是通的**，但它依赖「先 provide 一个 undefined 再 set」这种非常规写法。
→ 标 **NOT_TESTED**：没有实机装过，不能断言它能加载。

---

## E. speak-up 0.2.7 对照（本地文件直读，1298 行）

源码：`~/.dsh/profiles/desktop/node_modules/dsh-speak-up/lib/host.js`。

### E.1 调度算法（读代码，不是看 README）

```js
// host.js:802
停表 = ctx.interval(() => { void 掷一次('定时','自动') … }, 间隔分钟 * 60 * 1000)
```

**固定间隔轮询 + 掷一次本地骰子（零模型成本）**。一轮的判定顺序（`掷一次()`，`:648` 起）：

1. `模式==='自动' && !配置.总开关` → 记账「总开关是关着的」并返回（`:653`）
2. `在安静时段(小时, 安静起, 安静止)` → 跳过（`:658-660`）；安静时段支持跨午夜
3. `目标会话.length===0` → 跳过
4. 逐个判活：`找agent(ctx,id)`（`:460`，只 `ctx.agents.get()`，两种 id 写法都试）+ `a.status !== undefined && a.status !== 'idle'` → 忙；`workspaceRegistry.archivedSessionIds` → 已归档
   **`可用.length===0` 时**（`:693-696`）：
   > `勾的目标会话现在都不能说话（…）—— 按设计**不自动唤醒**，这轮什么都不做`
5. 随机挑一个可用会话（`可用.length>1` 时只挑一个——设计上「她只在一个地方说话」）
6. 自我冷却：`Date.now() - 自己上次开口 < 搭话冷却分钟` → 跳过（`:705-710`）
7. **命中线**（`:719-724`）：
   ```
   命中线 = min(100, 基础命中线 × 时机因子 + min(加成上限, 连没中 × 没中加成))
   ```
   - `基础命中线`：峰时（周一到周五 9:00–12:00 / 14:00–18:00）取 `峰时命中线`，其余取 `谷时命中线`（`算基础命中线`，`:385`）
   - `时机因子`：距上次说话 < `近阈值分钟`(20) → ×`近因子`(0.15)；> `远阈值分钟`(360) → ×`远因子`(1.5)（`:624-630`）
   - `连没中`：没掷中 +1，掷中清零；加成 `没中加成`(10)/次，上限 `加成上限`(40)（`:723-750`）
8. `掷值 = Math.random()*100`；`掷值 >= 线` → 没掷中，攒加成；否则掷中
9. `配置.角色==='developer'` → **主动拒绝**（`:753-757`，理由见 E.3）
10. 试运行 → 只记账；否则 `await agent.followup(消息)`（`:772`）

### E.2 关键问题：它能不能唤醒已经结束/休眠的 agent？

**代码事实：不能，而且这是它主动做的选择，代码里写得很明确。**

- `找agent`（`:460-475`）只用 `ctx.agents.get(id)`。`@deepseek-ai/dsh-agent/lib/index.js:596-598`：
  ```js
  get(id) { return this.store.get(id)?.agent; }   // 只查「活着的」注册表
  ```
  ⇒ 冷会话（进程重启过、agent 已 disposed）查不到 → `undefined` → 归入「没活着」。
- `可用.length===0 → 按设计不自动唤醒`（`:693-696`）——**这句注释是作者自己写的**。
- 唯一的「建会话」能力在设置页路由 `/new-session`（`:1234` 起 `/new-session` 路由），用 `ctx.get('sessionController').create()`，需要人工点按钮。

**但 DSH 本身是有这个能力的**（Lead 已核实，我复核了行号）：

`@deepseek-ai/dsh-api-session-controller/lib/index.js`：
- `:210` `async resolveAgent(sessionId)` → `:221` `async resolve(sessionId, observation)`
- `:221-246`：先 `liveAgent(sessionId)`，拿不到再 `this.resume(sessionId, observation)`
- `:370` `async resume(sessionId, supplied)` → `:399` / `:455` `return (await this.ctx.agents.resume({...}))`
- `:925` `async prompt(request, source)` —— 内部就是 resolveAgent + followup

⇒ **结论：把 `ctx.agents.get()` 换成 `ctx.sessionController.resolveAgent()`，speak-up 就能对冷会话开口。** 「不能唤醒」是 speak-up 的实现选择，不是 DSH 的平台限制。

**代价（必须如实说明）**：
- `resolveAgent` 对冷会话会**真的 resume 一个 agent**（读会话日志、重建 runtime、占内存）。一次掷中 = 一次冷启动。
- 它可能返回 `{error}`：`:239-246` 处理了 `session/not-found`、`session/writer-held`（另一进程持有写权）、`gateway/internal`。**多开 DSH 时 `session/writer-held` 是常见路径**。
- 冷启动后 `followup` 会立刻跑一个完整 turn = **完整 token 成本**（不是 "续上" 而是 "重开一轮，读全历史"）。

### E.3 配置项含义（对照 `~/.dsh/speak-up-config.json` 实测值）

当前文件内容（**总开关 = false**，即主动发言是关的）：

| 键 | 当前值 | 含义（源码 `:99-137` 默认值 / 注释） |
|---|---|---|
| `总开关` | **false** | 关掉就完全不动 |
| `试运行` | false | **默认是 true**（掷中只记账不真发）。当前被改成 false = 真发 |
| `目标会话` | `["session-429facb2-…"]` | 数组，可多个；掷中一次只在一个里开口 |
| `间隔分钟` | 10 | 多久掷一次骰子（默认 30） |
| `峰时命中线` | 25 | 周中 9–12/14–18 的命中线（默认 20） |
| `谷时命中线` | 85 | 其余时间（默认 60）**当前值很激进** |
| `峰谷时区` | `""` | 空 = 跟本机时区；可填 `Asia/Shanghai` |
| `没中加成` | 15 | 每没掷中一次，下次命中线 +15（默认 10） |
| `加成上限` | 50 | 加成最多 +50（默认 40） |
| `安静起/安静止` | 1 / 8 | 1:00–8:00 不掷。**起===止 表示不启用** |
| `近阈值分钟` / `近因子` | 20 / 0.15 | 距上次说话 <20 分钟 → 命中线 ×0.15 |
| `远阈值分钟` / `远因子` | 360 / 1.5 | >360 分钟 → ×1.5 |
| `搭话冷却分钟` | 10 | **她自己开口之后**的冷却（与上面两个阈值是两回事） |
| `触发文案` | 「有点无聊，去看看鱼片在干嘛」 | 会被 `框住内心话()` 包一层「这是你自己冒出来的念头」 |
| `角色` | `user` | `developer` 会被插件主动拒绝（DSH 只放行 user 角色，理由见 `host.js:16-33` 三条源码证据） |

**算一下当前配置的实际开口率**：谷时命中线 85 × 远因子 1.5 = 127.5 → 夹到 100 ⇒ **超过 6 小时没说话时，掷中概率 100%**；每 10 分钟掷一次 ⇒ 一旦过了 6 小时空窗，**10 分钟内必开口**。峰时 25 × 1.5 = 37.5 ⇒ 约 1/3。
（这是纯算术推演，未实跑，见 NOT_TESTED。）

### E.4 speak-up 的两个真实缺陷

1. **`ctx.interval` 生命周期**：`换表` 里 `停表 = ctx.interval(...)`（`:802`）是 Cordis API，会随插件卸载回收 —— 比 aura 的裸 `setInterval` 好。
2. **`找agent` 只查活着的** —— 即 E.2 那条。这是它最大的能力缺口。

---

## F. 调度结论：**三个调度器要收成一个，aura 只吸收机制**

### F.1 现状：机上同时有三个主动说话源（两个开着）

| # | 来源 | 状态 | 机制 | 是否产生 agent turn |
|---|---|---|---|---|
| 1 | **dsh-speak-up 0.2.7** | **总开关 = false（关）** | `ctx.interval` 每 10 分钟掷骰 → `agent.followup()` | **是**（真开一轮） |
| 2 | **surfcore `[pet-purpose]` 0.7.10** | **开**（实测 `lastOutcome=sent`） | `setInterval(tick, 60_000)`（`lib/pet-purpose.js:610`，`TICK_MS = 60*1000`） | **否**（推一句给桌宠气泡/语音） |
| 3 | **dsh-pet 0.3.8 碎碎念** | **开**（`whisperEnabled: true`） | `automaticWhisperSchedule(…, whisperPeriodSec(conf))`（`lib/index.js:5841`），默认 300 秒；每周期**真的调一次模型**（`generateWhisper`，`:4080`） | **否**（只弹气泡） |

实证证据：

- **surfcore**：`~/.dsh/surfcore/characters/char-mvuaxf54/purpose-outreach.json` 实测
  `{"lastOutcome":"sent","lastSentAt":1793325347022,"lastPurpos e":"最近想学点新东西…","lastText":"主人，肥鱼最近有点想学新东西了…","nextFireAt":1793340792359,"drives":{"connect":0.95,"install":0.38,…},"goals":[…],"lastEmotion":"curious"}`
  → `lastSentAt` 换算 = **2026-10-30T01:55Z**，`countToday: 2`。**它真的发过。**
  → 且有 `drives`（9 维驱力）、`goals`、`history`、`materials` —— **这是三者中唯一有"动机状态"的实现。**
  → 源码 `lib/pet-purpose.js:1-20` 明确写了载体互斥：`carrier=pet` 时桥接的 `bridge.js` 自动 tick 跳过；`carrier=qq` 时本模块 tick 直接返回。**所以 QQ 与桌宠之间已经做过一次收敛了。**
- **dsh-pet**：`~/.dsh/dsh-pet/main-config.jsonc` 实测 `pets[0].whisperEnabled = true`（name="蓝毛小女仆"）。周期默认 300 秒（源码 `lib/index.js:4150-4156` 注释「非法兜底 300」）。
- **speak-up**：`~/.dsh/speak-up-config.json` 实测 `"总开关": false`。

**「主动发言权分裂」的具体形态**：桌宠上的「蓝毛小女仆」每 5 分钟用模型说一句碎碎念，同时 surfcore 的「肥鱼」按驱力在桌宠气泡里说一句，而"大肥鱼"这个主会话（`session-429facb2-…`）被 speak-up 指着但关着。**三个角色、三套节奏、一个屏幕上两个气泡源。**

### F.2 能不能用 aura 替换 speak-up？**不能。** 理由按重要性排序

1. **aura 没有投递能力**（D.2）：无 `followup`、无 `webServer`、webhook URL 未注册。它连"说一句话"都做不到。
2. **用户明确要求 speak-up 优先保留**。
3. **aura 的价值网络是装饰**：`_interruptionCost()` 硬编码 0.3，`relevance` 是配置常量 ⇒ `V` 退化成一个定时器。而 surfcore 有 9 维真实驱力 + 目标假设 + 素材挖掘。
4. **GPL-3.0** ⇒ 吸收要"只学思路不抄代码"。
5. **它有泄漏缺陷**（D.4）：传感器定时器永不回收。

### F.3 收敛顺序（四步，每步给回滚）

**原则：先关最便宜的噪音，再确定唯一"说话人"，最后才考虑加东西。任何一步都不是"装新插件"。**

**第 1 步 · 关掉纯烧钱的碎碎念（立刻可做，零风险）**
- 改 `~/.dsh/dsh-pet/main-config.jsonc`：`pets[0].whisperEnabled: true → false`
- 为什么先动它：它每 300 秒调一次模型，**产出只是一句气泡，不进入任何会话历史**，信息价值最低、成本确定。
- **回滚**：改回 `true`。

**第 2 步 · 认下唯一「主动说话人」= surfcore `[pet-purpose]`**
- 理由：它是三者中**唯一有状态**的（drives/goals/history/materials），且已经做过 carrier 互斥（`lib/pet-purpose.js:1-20`）。
- 动作：确认 `~/.dsh/surfcore/config.json`（或 `readUnified` 落点）里 `purpose.carrier = 'pet'`，并确认 `qq-bridge` 那条自动 tick 确实是跳过状态（`lib/pet-purpose.js:8-11` 说 `carrier=pet` 时 `bridge.js` 会跳过）。**这一步只读不写，确认即可。**
- **回滚**：`carrier` 改回 `'qq'`（或删掉该字段回默认）。

**第 3 步 · speak-up 保持关，但改造成「唯一有权开真回合的人」**
- 现状 `总开关=false`，什么都不做，**这一步不改配置**。
- 未来若要让"大肥鱼"主动开口，只走 speak-up，且必须先修它的 `找agent`：
  ```js
  // 现在（host.js:460-475）
  const a = ctx.agents.get(一个)                    // 冷会话 → undefined → 判"没活着"
  // 应改为（DSH 已提供的能力，见 E.2）
  const r = await ctx.get('sessionController').resolveAgent(id)   // 冷会话会 resume
  if ('error' in r) → 当"没活着"处理（writer-held / not-found）
  ```
  同时 `可用` 判定里 `a.status !== 'idle'` 要保留（避免往正在跑的会话里插话）。
- 打开顺序建议：`总开关=true` 但**先 `试运行=true`** 观察 2–3 天，看日志里"掷中"与实际想要的比例，再关试运行。
- **回滚**：`总开关=false`（就是当前值，无需任何回滚动作）。

**第 4 步 · 从 aura 只吸收 3 个机制（不装 aura）**

| 吸收什么 | aura 的位置 | 落到哪 | 为什么值得 |
|---|---|---|---|
| **传感器触发**（file/process/http/port 状态翻转） | `src/sensors.ts` `SensorEngine`，`probe()` 只在 `prev.state !== current.state` 时 fire | surfcore `pet-purpose` 的 `collectMaterials()` 里加一路素材源 | 这是 aura 唯一**别人没有**的东西：让"主动说话"有真实外部事件驱动，而不是纯计时。**注意 Windows 上 `process`（pgrep）不可用，只吸收 file / http / port 三种。** |
| **off-peak 窗口**（`offPeakOnly` + `[2,7]`） | `src/index.ts` `_inOffPeak()` | surfcore 的 `evaluatePurposeTick` 决策前加一道 | 省钱的正确形态。surfcore 现在只有 `quiet/daily/gap` 三档节奏，没有"便宜时段才开口" |
| **HMAC webhook 限流**（`createHmac('sha256', secret).update(name+':'+body)` + `timingSafeEqual` + 60/min + 401/429/404） | `src/webhook.ts` 全文 | 若要给 DSH 开"外部事件唤醒"面，这是现成的正确做法 | 但它必须**先有 `ctx.webServer.register` 那一层**——aura 缺的就是这层。吸收时要把那层补上（参照 speak-up 的 `路线()` 封装，`host.js` 里已有 `ctx.effect(() => ctx.webServer.register({kind:'exact', path, handler}))` 的写法） |

**不吸收的**：价值网络（`V = α·idleRatio + β·relevance − δ·0.3`，标准 `_interruptionCost` 硬编码）、`onProactive` emit 机制、class 插件 + `ctx.provide('aura')` 的写法。

**第 4 步的回滚**：因为是改 surfcore 自己的代码/配置，回滚 = `git checkout` surfcore 的对应文件 + 重启插件。**不涉及任何新包安装，所以没有"卸不干净"的风险。**

### F.4 一条硬规矩（建议写进 DSH-Go 的调度设计）

> **在同一个 DSH_HOME 下，「产生 agent turn 的调度器」最多 1 个；「不产生 agent turn 的提示类调度器」最多 1 个。**
>
> 依据：三者之所以会分裂，是因为它们分别落在「开真回合」（speak-up）和「弹气泡」（surfcore pet-purpose / dsh-pet whisper）两类里，而**没有人在管「同类的第二个」**。surfcore 自己做过一次载体互斥（pet vs qq），说明这个约束是必要的，只是没有推广。

---

## G. dsh-clinic @ `bd1b3e84736c7fc23b89342cae94487e15f08504`：能不能当本次验收工具？

### G.1 一句话结论

**能当「结构验收门」，不能当「安全认证」。** 它检查的是**插件包能不能被 DSH 正确加载、会不会干扰别的插件、注入的服务名是否真实存在**；它**完全不检查**安全、权限、隐私、网络出口、供应链、依赖漏洞。

### G.2 它到底检查什么（分组逐项，源码确认）

版本 **0.8.0**，license **Apache-2.0**，`engines: { node: "^22.19.0 || >=24.0.0", dsh: ">=0.2.1-0 <0.3.0" }`，**零依赖**。
CHECKSET 常量：`'R0-R8+K1-K14+D0-D3,D9+CC1-CC5/3+X1-X5'`（`doctor.mjs:21`）。

**R 组 · 静态包结构（9 项）** `lib/checks-package.mjs`

| ID | 查什么 |
|---|---|
| R0 | name/version/license/**README 文件**/type |
| R1 | **`dsh.bundle.patch` 存在**（`critical: true` —— 唯一的关键项） |
| R2 | `npm pack --dry-run --ignore-scripts` 后**入口与 patch 是否真在 tarball 里**、是否空文件、tarball 名是否合规 |
| R3 | `cordis.patch.yml` 结构（启发式，明标「以 D2 --dump-config 为准」） |
| R4 | 入口是否导出 `name`/`apply`（**启发式，class/default 形态会误报**） |
| R5 | 依赖口径：`@deepseek-ai/cordis` 是否 rescope、有没有裸上游名 |
| R6 | `engines.node` 声明 |
| R7 | 预构建产物与 `files` 白名单覆盖 |
| R8 | peer 范围旧 rc 残留 / open-top 区间 |

**K 组 · 静态 cordis 契约扫描（14 项）** `lib/checks-cordis.mjs`

| ID | 查什么 |
|---|---|
| K1 | `ctx.<service>` 访问是否都在 `inject` 里声明 |
| K2 | 有没有对 `ctx` 做 `JSON.stringify`/`structuredClone`/展开（Context 是 Proxy） |
| K3 | `setInterval/setTimeout/setImmediate/process.on/addEventListener` 是否包进 `ctx.effect` |
| K4 | Schema 旁是否有函数值（进不了 schema） |
| K5 | `apply` 返回形状（v4 Effect 契约：函数/null/undefined/Promise/iterable） |
| K6 | `inject` 的服务名是否属于已知 seam（比对**生成物**，`:262` 注释：91 服务 + 30 框架成员） |
| K7 | v3 遗留 API（3.x→4.x 已删） |
| K8 | `Config` 是否是 Standard Schema 校验器 |
| K9 | `name="apply"` 特例 |
| **K10** | **waterfall 监听器必须委托 `next()`**（否则静默吞掉全部下游） |
| **K11** | **工具名遮蔽内置/保留名**（含 `tools.data.delete(def.name)` 两步遮蔽法） |
| **K12** | 服务键与宿主 seam 同名（同 isolate scope 只能一个提供方） |
| **K13** | patch 是否覆写宿主自带行 config（比对生成物的 94 个 `dsh-base` 行 id） |
| K14 | 跨仓注入点重名（需 `--workspace`） |

**X 组 · 动态面·宿主契约（5 项）** `lib/checks-runtime.mjs`
X1 peer 走廊与实际宿主版本求交（调宿主自己的 `evaluatePluginCompatibility`，读不到就退回复刻实现）/ X2 版本载体自洽 / X3 `dsh.bundle.patch` 形态 / **X4 `inject` 名单 vs 宿主真实服务面**（`:359` 注释：「不存在的服务 ⇒ apply 永不执行且零报错」）/ X5 patch 行可定位性。

**D 组 · 动态沙箱冒烟（5 项）** `lib/checks-smoke.mjs`
D0 `npm pack` / D1 `npm install @deepseek-ai/dsh@<实测版本>` + `dsh plugin add --ignore-scripts` + 断言 `dsh.profile.bundles` 含本包 / D2 `--dump-config` 找层标记 / D3 keyless headless 跑一句，判据是 `exit 1 + dsh: MISSING_CREDENTIAL`（严格匹配 code，防正则假阳性）/ D9 沙箱隔离。
**全程 `%TEMP%\doctor-sbx-*`，结束 rename 到 `doctor-quarantine-*` 不删除**（`lib/util.mjs` `makeSandbox`/`quarantineSandbox`，注释明写「本工具不再有任何 rmSync/rm -rf 路径」）。

**CC 组 · 生态清单（5 项）** `lib/checks-collections.mjs`
CC1 认证注册表 / CC2 awesome-dsh-plugin 收录 / CC3 dsh-catalog / CC4 dshWorkshop 清单 / CC5 插件三门（license/五语 README/seam 三角色）。**这一组对"能不能用"没有意义，是对"能不能被生态收录"的检查。**

### G.3 它**不**检查什么（这就是「不是安全认证」的意思）

我把 6 个检查文件全 grep 了 `vulnerab|CVE-|audit|security|malicious|exfiltrat|permission|network access`：

```
clinic__doctor.mjs                  (none)
clinic__plugin.mjs                  (none)
clinic__lib~checks-package.mjs      (none)
clinic__lib~checks-cordis.mjs       permission:2   ← 是 KNOWN_SEAMS 名单里的服务名 'permission'，不是检查
clinic__lib~checks-runtime.mjs      audit:1        ← 是错误消息里提到宿主的 auditStartupEntries
clinic__lib~checks-smoke.mjs        audit:1        ← 是 npm install 的 --no-audit 参数
clinic__lib~checks-collections.mjs  security:1     ← 是 omdsh 的分类标签数组里的一个字符串
```

**零命中真检查。** 它不检查：

- ❌ 有没有 **CVE / 依赖漏洞**（D0/D1 甚至显式传 `--no-audit`）
- ❌ 插件**读了什么文件、往外发了什么网络请求**（D1 是 `--ignore-scripts` 装的，**装完只跑 `--dump-config` 和一句 headless prompt，不审计运行期行为**）
- ❌ **持久化副作用**（会不会写 `~/.dsh`、会不会 spawn 子进程、会不会定时器常驻、会不会删会话目录）—— 对照 A.5：auto-memory 的「2 天归档 7 天删除」**clinic 一个字都不会报**
- ❌ 权限/审批面（`permission`/`approval` 只是 inject 名单里的字符串）
- ❌ 许可传染性（**aura 是 GPL-3.0，clinic 的 CC5 只查"有没有 license 文件"，不查是什么 license**）
- ❌ 数据外泄、凭据处理、隐私
- ❌ 性能/成本（token 消耗、子代理数量）

**它反而是「反面证据采集器」**：它只证明「按 Cordis 契约组装不出错」，这是必要的、但远不充分。

### G.4 实际调用方式（本机实测命令）

**我实际跑过的三条（原样可复现）：**

先把仓库落到本地（我只读了需要跑的 18 个文件）：
```
_cache/clinic_run/         ← doctor.mjs + package.json + lib/*.mjs（18 文件）
```

**① 静态 R+K（对已装的 fish-memory）**
```powershell
node C:\...\_cache\clinic_run\doctor.mjs `
  --repo "C:\Users\<user>\.dsh\profiles\desktop\node_modules\dsh-fish-memory" `
  --no-smoke --only R,K --format check
```
实测：**exit 1**，R0 PASS / R1 PASS / R2 **FAIL（`npm` 不在 PATH）** / R3 PASS / R4 PASS / R5 WARN(skip) / R6 WARN（未声明 `engines.node`）/ R7 PASS / R8 PASS / **K1 WARN（`ctx.interval` 1 处未 inject）** / K3 WARN（client.js 5 处裸 setTimeout）/ K14 WARN(skip) / 其余 PASS。

**② X 组（宿主契约）**
```powershell
node ...\doctor.mjs --repo "<插件路径>" --no-smoke --only X
```
- fish-memory：**exit 0**，X1–X5 全 PASS（inject 的 9 个服务名都真实存在，服务面来源：快照 @ dsh 0.2.1-alpha.1，158 个键）
- speak-up：**exit 0**，但判定 **SKIP**（X1 SKIP 因未声明 `@deepseek-ai/dsh*` peer；X2 WARN）

**③ 全组（对 aura）**
```powershell
node ...\doctor.mjs --repo "<aura 仓>" --no-smoke --only R,K,X
```
实测：**exit 1**，warn=6 pass=18 fail=1 skip=3（fail 那条又是 R2 的 npm）。

**环境前提（本机实测）**：`node v24.19.0`（`D:\node.exe`）在 PATH；**`npm` 与 `pnpm` 都不在 PATH**（`pnpm` 实际在 `...\resources\runtime\primary-runtime\dependencies\pnpm`）。⇒ **D 组完全跑不了，R2 必然 FAIL。**

### G.5 我发现的两个 clinic 自身缺陷（诚实报告）

**缺陷 1（会误判、影响验收）：R2 把「环境没有 npm」报成「插件缺陷」。**
`lib/checks-package.mjs` R2：
```js
const r = runStep('r2-pack','npm',['pack','--dry-run','--json','--ignore-scripts'],...)
if (!r.ok) return fail(`npm pack 失败（exit ${r.code}）:\n${tail(r.err)}`)   // ← 没有 category
```
对照 D1（`checks-smoke.mjs`）对同一情况的处理是**正确的**：
```js
if (i.spawnError || i.code === 127)
  return { status:'fail', message:'环境不可用：找不到 npm/node…', category:'infrastructure' }
```
⇒ R2 缺 `category:'infrastructure'`。**后果：任何没装 npm 的机器上，所有插件都会被判 exit 1。** 这是它的 bug，不是我们环境的问题被它正确识别。

**缺陷 2（已知且有兜底）：R4 对 class 插件形式误报。**
它自己在 message 里承认：「启发式扫描；若为 default 对象/class 形式请人工确认」。我核实过 class 形态合法（`@deepseek-ai/cordis/lib/index.js:1066-1071` `isConstructor → new callback(ctx, config)`）。

### G.6 结论：怎么用它当验收门

**PASS 的含义**：能被 DSH 正确组装、不会在 cordis 层面打架、inject 的服务名真实存在。
**PASS 不代表的**：安全、无副作用、成本可接受、功能可用。

**建议的验收口径（三层，逐层加严）**：

1. **门 1（每次提交必须过）**：`--only R,K,X --no-smoke`
   判据：**无 fail**；R1 必须 PASS；K3/K10/K11/K12/K13 无 WARN（这五项是"会干扰别人"的类别）；X4 必须 PASS。
   ⚠️ 因本机无 npm，R2 的 FAIL 要**人工忽略并记录**（或修 clinic 的 R2）。

2. **门 2（装机前必须人工补）**：clinic **不覆盖**的那部分，必须自己读代码确认：
   - 有没有 `rmSync`/`unlink`/`archiveSession`/`deleteSession` 之类**破坏性动作**（auto-memory 就命中，见 A.5）
   - 有没有 `setInterval` 虽然包了 `ctx.effect` 但逻辑上会永久跑（speak-up 的 `ctx.interval` 是可回收的，aura 的 `setInterval` 不是）
   - 有没有 `spawn`/子代理（auto-memory 每轮 spawn）
   - license 是什么（**GPL-3.0 的 aura**）
   - 会不会往共享目录写（`~/.dsh/sessions`、`~/.dsh/storages`）

3. **门 3（实机，非本报告范围）**：`--only D` 需要 npm+pnpm 在 PATH 且能联网装 `@deepseek-ai/dsh`。本机 **NOT_TESTED**。

**更实用的替代**：把 `dsh-clinic` 当**只读工具**，通过它的 `plugin_doctor` tool 或 `/doctor` 命令用（`plugin.mjs` 注册了这两个入口，`inject = ['tools','commands']`）——但**这需要把 dsh-clinic 本身装进 profile**。本报告是外挂跑 `doctor.mjs`，**没有安装它**。

---

## H. 证据清单（全部可复核）

### H.1 远程源码（`raw.githubusercontent.com`，按锁定 SHA 取）

| 仓库 | SHA | 文件 | 行数 | 用途 |
|---|---|---|---|---|
| Aik358/dsh-auto-memory | `a244cf03…` | `lib/index.js` | 21233 | 全部 A 节结论 |
| 同上 | 同上 | `package.json` | — | A.1 |
| 同上 | 同上 | `cordis.patch.yml` | 5 | A.1 |
| 同上 | 同上 | `lib/memory-importance.js` | 3243B | A.4 重要性评分 |
| 同上 | 同上 | `lib/semantic-js.js` | 31135B | A.3 e5-small 112.8MB |
| 同上 | 同上 | `lib/shadow-retrieval.js` | 35659B | A.4 时间衰减 |
| 同上 | 同上 | `lib/injection-policy.js` | 3019B | 敏感段剥离（附加发现） |
| 同上 | 同上 | `lib/session-archive.js` | 15371B | A.5 归档/删除 |
| 同上 | 同上 | `docs/ARCHITECTURE.md` | 3120B | A.6 |
| 同上 | 同上 | `docs/M7-INTERFACE-DIGEST.md` | 7968B | A.6 |
| 同上 | 同上 | `docs/INTEGRATION-ANALYSIS.md` | 16748B | A.6 |
| ljsysfurryACE/dsh-aura-scheduler | `f9048cd5…` | `src/index.ts` `src/sensors.ts` `src/webhook.ts` `lib/index.js` `package.json` `cordis.patch.yml` `README.md` `tests/smoke.ts` | 16 文件里读了 8 个 | D 节全部 |
| PerryLink/dsh-clinic | `bd1b3e84…` | `doctor.mjs` `plugin.mjs` `package.json` `lib/{framework,checks-package,checks-cordis,checks-runtime,checks-smoke,checks-collections,compat,standards,host-surface,removed-seams,util}.mjs` | 18 文件 | G 节全部 |

缓存落点：`dsh-go-handoff/research/_cache/`（含 `clinic_run/` 与 `staged_aura/` 两个可跑副本）。

### H.2 本地文件（直接 `node:fs` 读，未经任何修改）

| 路径 | 用途 |
|---|---|
| `~\.dsh\profiles\desktop\node_modules\dsh-fish-memory\lib\index.js`（6501 行） | B 节全部；行号锚点 `36/556/2877/3185/3796/3920/4110/4115/4145/4229/4251/4269` |
| `~\.dsh\profiles\desktop\node_modules\dsh-speak-up\lib\host.js`（1298 行） | E 节全部；行号锚点 `99/385/460/624/693/719/753/772/802` |
| `~\.dsh\fish-memory\session-2c93cd4f-….json`（45748B） | B.1 schema + memories=1 / traces=100 |
| `~\.dsh\fish-memory-diag.log`（244608B，1339 行） | B.4 实证：120 次快照 / 103 有 / 17 无 / 17 全是 1 工具 |
| `~\.dsh\fish-memory-config.json` | B.4 `{"全局开关": true, "switchMigrated": true, "sessions": {}}` |
| `~\.dsh\speak-up-config.json` | E.3 全部当前值（总开关=false） |
| `~\.dsh\dsh-pet\main-config.jsonc` | F.1 `pets[0].whisperEnabled=true` |
| `~\.dsh\surfcore\characters\char-mvuaxf54\purpose-outreach.json` | F.1 `lastOutcome=sent, lastSentAt=1793325347022` |
| `~\.dsh\plugins\surfcore\lib\pet-purpose.js`（633 行） | F.1 `TICK_MS=60s`（:47）、`setInterval(tick,TICK_MS)`（:610）、载体互斥（:1-20） |
| `~\.dsh\profiles\desktop\node_modules\dsh-pet\lib\index.js` | F.1 `automaticWhisperSchedule`（:5841）、`whisperPeriodSec` 兜底 300（:4150） |
| `~\.dsh\profiles\desktop\package.json` + `cordis.patch.yml` | 31 个 bundle 清单；确认 auto-memory **未装** |
| `~\.dsh\memory\` | **不存在**（`existsSync` = false）⇒ auto-memory 从未运行过 |

### H.3 宿主源码（`app.asar/dsh/node_modules/@deepseek-ai/`，只读）

| 包 / 文件 | 行 | 用途 |
|---|---|---|
| `dsh-system-prompt/lib/index.js` | 395 | 证明 `assembly.tools` = 模型可见 schema 列表 |
| `dsh-tools/lib/index.js` | 2932-2944 `wireSchemas` / 3123 `sdkSchemas` / 2991 `restrict` / 2981 保留名 | 证明 **PTC 模式模型侧只剩 `run_code`** |
| `dsh-tools/lib/index.js` | 955 `RUN_CODE_NAME` | 保留名 |
| `dsh-agent/lib/index.js` | 596 `get(id)` 只查活着的 / 614 `list()` / 466 `resume()` | 证明 `agents.get` 拿不到冷会话 |
| `dsh-agent-loop/lib/index.js` | 784-791 `phase/status` / 806 `followup` / 830 `wakeDriver` | 证明 `followup` 会真开一轮 |
| `dsh-api-session-controller/lib/index.js` | 210 `resolveAgent` / 221 `resolve` / 370+399 `resume` / 925 `prompt` | **证明 DSH 有冷会话唤醒能力** |
| `dsh-schedule/lib/index.js` | 1493-1620 `ScheduleRuntime` —— `resolveAgent` → `agent.followup` → `sessions.flush` | 官方调度器**也靠 followup 开真回合**，官方先例 |
| `cordis/lib/index.js` | 782 `set` / 800 `provide` / 1066-1071 `isConstructor → new` / 1446 `isApplicable` | 判定 aura class 插件形态合法 / `provide(仅 name)` 行为 |
| `dsh-subagent/lib/index.js` | 354-356 `origin:'subagent'`+`delegationDepth` / 395 `tools.restrict` | 解释 fish-memory `是子代理()` 判据 |
| `dsh-agent-tool-presentation/lib/index.js` | 全 48 行 | `presentAs('ptc'/'native')` 的来源 |

### H.4 我实跑过的命令（可复现）

1. `node clinic_run/doctor.mjs --repo <fish-memory> --no-smoke --only R,K --format check` → **exit 1**（R2 npm）
2. `node clinic_run/doctor.mjs --repo <fish-memory> --no-smoke --only X` → **exit 0**（X1–X5 全 PASS）
3. `node clinic_run/doctor.mjs --repo <speak-up> --no-smoke --only X` → **exit 0 / verdict SKIP**（X1 SKIP, X2 WARN）
4. `node clinic_run/doctor.mjs --repo <staged_aura> --no-smoke --only R,K,X` → **exit 1**（warn=6 pass=18 fail=1 skip=3）
5. `node -v` → v24.19.0；`npm -v` → `'npm' 不是内部或外部命令`；`pnpm -v` → 同上
6. 在 `run_code` 沙箱内 `Object.keys(tools)` → **96 个工具，含 `remember`**（本会话 PTC 模式反证 B.4 的「没有 ✗」是误报）

---

## I. NOT_TESTED（没跑过就是没跑过）

### I.1 dsh-auto-memory（整节都是静态分析）

| 项 | 状态 | 原因 |
|---|---|---|
| 插件能否加载 | **NOT_TESTED** | 从未安装（`~\.dsh\memory\` 不存在，profile 无 `@a9i5k4`） |
| 自动沉淀实际产出质量 | **NOT_TESTED** | 需运行 + 花 token |
| 语义检索（e5-small）实际效果 | **NOT_TESTED** | 模型 112.8MB 未下载 |
| Python sidecar 档 | **NOT_TESTED** | `pythonBackendEnabled:false`，未启 |
| `memory_recall(scope='sessions')` | **NOT_TESTED** | 需宿主侧开 sessionQuery（注释说出厂默认关） |
| 归档/删除**真的会删** | **NOT_TESTED**（源码证实有 `rmSync` 路径） | 未安装，不敢实跑 |
| 影子模式配置块是否能真跑起来 | **NOT_TESTED** | 是我基于 `DEFAULT_CONFIG` 推导的，未实机验证过任意一版 |
| 与 fish-memory 并存的 token 增量 | **NOT_TESTED** | 需实跑对比 |

### I.2 dsh-aura-scheduler

| 项 | 状态 | 原因 |
|---|---|---|
| 能否被 cordis 加载（class 形态 + `provide('aura')`） | **NOT_TESTED** | 未安装；静态推断可行（D.5），但依赖非常规写法 |
| webhook 端到端（对外 POST → 唤醒） | **NOT_TESTED 且源码说明不可能** | `handle()` 无 HTTP 接入层（D.2 证据 3） |
| sensors 实际触发 | **NOT_TESTED** | 未安装；且 `process` 类在 win32 必然失效（无 pgrep） |
| `tests/smoke.ts` 12/12 | **NOT_TESTED** | 需 `tsx`，本机无；README 自称通过不作证据 |
| 定时器泄漏是否真发生 | **NOT_TESTED**（源码证据强：`sensorTimer` 从未赋值、`_sensorTimers` 从不清理） | 未实跑 |

### I.3 dsh-speak-up

| 项 | 状态 | 原因 |
|---|---|---|
| 实际开口率是否等于算术推演（谷时 6h 后 100%） | **NOT_TESTED** | 纯算术；未实跑 |
| 它现在是否在发消息 | **NOT_TESTED**（配置 `总开关=false` ⇒ 静态判断**不会**发） | 未读插件运行日志 |
| 改成 `resolveAgent` 后能否真唤醒冷会话 | **NOT_TESTED** | 只是设计建议，未改一行代码 |
| `followup` 对冷会话的实际成本 | **NOT_TESTED** | 未测量 |

### I.4 dsh-clinic

| 项 | 状态 | 原因 |
|---|---|---|
| **D 组（D0–D3, D9）沙箱冒烟** | **NOT_TESTED** | `npm`/`pnpm` 不在 PATH；且需联网装 `@deepseek-ai/dsh` |
| R2 tarball 完整性 | **NOT_TESTED**（实跑 FAIL，但是 `npm` 缺失所致，非插件问题） | 同上 |
| `plugin_doctor` tool / `/doctor` 命令 | **NOT_TESTED** | 需把 dsh-clinic 装进 profile（本报告是外挂跑 `doctor.mjs`，**未安装**） |
| `--host <version>` 多宿主求交 | **NOT_TESTED** | 未传该参数 |
| K14 / CC1–CC5 | **NOT_TESTED**（跑成 SKIP） | 未提供 `--workspace`；CC 组需生态清单数据 |

### I.5 三个调度器

| 项 | 状态 | 原因 |
|---|---|---|
| surfcore `[pet-purpose]` 当前是否仍在发 | **NOT_TESTED** | 只有状态文件 `lastSentAt=2026-10-30T01:55Z` 这一个快照；没有它的实时日志 |
| dsh-pet 碎碎念实际每分钟成本 | **NOT_TESTED** | 未测量；只知道每 300s 调一次模型 |
| 「关掉 whisper」后是否真的停 | **NOT_TESTED** | 未改配置（只读纪律） |
| 三个源是否真的会造成「同一时刻两句话」 | **NOT_TESTED** | 需同时观察三个前端 |

---

## J. 给 Lead 的三句话

1. **记忆**：auto-memory **补不换** —— 存储路径/工具名/注入段名三处都不重叠，但它默认会**2 天归档 7 天物理删除 `~/.dsh/sessions`**、每轮 spawn 子代理；必须先以 `sessionArchiveEnabled:false + autoArchiveEnabled:false + autoDeleteEnabled:false + injectEnabled:false + autoConsolidate:false + activationInboxEnabled:false + shadowRetrievalEnabled:true` 的「影子模式」跑在**独立 profile**（abtest-zseven）里观察，再逐项打开。
2. **调度**：机上**三个**主动说话源（speak-up 关 / surfcore `[pet-purpose]` 开 / dsh-pet 碎碎念开）。aura **不能替换 speak-up** —— 它没有 `followup`、没有 `webServer`，连 webhook URL 都没注册，**它连"说一句话"的都做不到**；「价值网络」的 `_interruptionCost()` 硬编码 0.3、`relevance` 是配置常量 ⇒ 本质是定时器。收敛顺序：关 whisper → 认下 surfcore 为唯一说话人 → speak-up 保持关（要开先修 `找agent`→`resolveAgent`）→ 只从 aura 吸收 sensors/off-peak/HMAC-webhook 三个机制到 surfcore，**不装 aura**（GPL-3.0 + 定时器泄漏）。
3. **质检**：dsh-clinic 能当**结构门**（`node doctor.mjs --repo <path> --no-smoke --only R,K,X`），**不是安全认证** —— 6 个检查文件里 `vulnerab|CVE|audit|security|exfiltrat` 零命中真检查，且它**不会**报出 auto-memory 的删除副作用。本机无 npm ⇒ R2 必 FAIL（**是 clinic 的 bug：缺 `category:'infrastructure'`**，D1 有而 R2 没有）、D 组全跑不了。
