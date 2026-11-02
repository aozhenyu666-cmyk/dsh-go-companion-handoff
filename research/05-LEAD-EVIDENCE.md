# Lead 亲自实测的证据（E-xx）

时间：2026-11-02 22:30–23:05 (+08:00)
环境：DSH Desktop 0.2.1-rc.2 / Node 24.21.0 / pnpm 11.28.5 / Cordis 4.0.5 / win32 x64

---

## E-01 DSH 版本（命令行实测，非猜测）
来源：`resources/runtime/primary-runtime/runtime.json`
```json
{ "desktopVersion": "0.2.1-rc.2", "platform": "win32", "arch": "x64", "node": "24.21.0", "pnpm": "11.28.5",
  "payloadDigest": "e2cef445c262ce6e8d28480b5ebecc8b0c54c06bea7183374debfcf02f30eb7d" }
```
app.asar 内 root package.json：`"name":"@deepseek-ai/dsh-desktop","version":"0.2.1-rc.2"`，`dshBuildCommit: 332895e0f7c9d4dbfb29729f5eb367fa263ff986`
app.asar 内 dsh/package.json：`"name":"@deepseek-ai/dsh-desktop-runtime","version":"0.2.1-rc.2"`，显式依赖 `"@deepseek-ai/cordis": "4.0.5"`
→ **Cordis 精确版本 = 4.0.5**（不是 "~4.0.5" 的模糊声明）

## E-02 app.asar 可解包（后续查真实 API 的方法）
asar 头格式：前 16 字节 = [u32 pickleSize][u32 headerPickleSize][u32 headerStringSize][u32 jsonSize]，JSON 从 offset 16 开始，data 区从 16+jsonSize 开始。
本机实测：jsonSize=4779780，baseOffset=4779796。已把 306 个 @deepseek-ai 包列入可读范围。
关键包：`@deepseek-ai/dsh-api-session-controller`（lib/index.js 132142 字节）、`@deepseek-ai/dsh-api-gateway`、`@deepseek-ai/dsh-agent`、`@deepseek-ai/dsh-tools`、`@deepseek-ai/dsh-host-webserver`。

## E-03 【关键】DSH 自己就有"冷会话唤醒"能力（推翻"必须靠 DSH-Go 补"的假设）
文件：`app.asar → dsh/node_modules/@deepseek-ai/dsh-api-session-controller/lib/index.js`
- L210-212：`async resolveAgent(sessionId) { return this.resolve(sessionId) }`
- L221-246 `resolve()`：先看 `this.liveAgent(sessionId)`；不在就 `this.resume(sessionId)`；
- L370-391 `resume()` → `ctx.sessionQuery.observeSession(sessionId)` → L392-404 `resumeObserved()` → `ctx.agents.resume({ resumeSessionId: sessionId, ... })`
- L925-950 `prompt()`：`const agent = await this.resolveAgent(request.sessionId)` …… `agent.followup(message)`
→ **结论：`ctx.sessionController.resolveAgent(id)` / `.prompt()` 可以把已经冷掉（没有活 agent）的会话重新拉起来并照常开一轮。**
→ 对比：**dsh-speak-up 0.2.7 只做 `ctx.agents.get()`（已装源码 L460-475 找agent），拿不到就放弃，所以它确实不能唤醒冷会话。**

## E-04 【关键】上面这条不是纸上推理，是实测
利用本机已装的 `dsh-feiyu-app@0.1.0` 的只读诊断路由：
```
GET http://127.0.0.1:57009/feiyu/debug?sid=dd762176-bbe0-42e2-90ab-c45528aa2c30
→ {"ok":true,"sid":"dd762176-...","有活agent":false,
   "唤醒":"取不到可用 agent",
   "形状":"object :: agent | agent 键: loopCtx,id,options,session,inbox,phase,activityDone,scope,ctx,dispatch,... | followup=function | agents 有: 无"}
```
`有活agent:false` 说明 `ctx.agents.get()` 当时返回 undefined（会话是冷的）；
但 `resolveAgent()` 返回了 `object :: agent` 且 **`followup=function`** —— 冷会话被成功解析成了可用 agent。
对 `429facb2-ccc8-49dc-b99d-cf44d77d8dca`（大肥鱼生产会话）同样返回 `followup=function`。
（注：feiyu-app 自己的 `找agent` 用 `ctx.agents.get` 所以报"取不到"，是它自己的判定逻辑保守，不代表 resolveAgent 失败。）

## E-05 dsh-feiyu-app 是"手机端"最现成的接入点（已在跑）
文件：`C:\Users\敖震宇\feiyu\plugin\index.mjs`（8730 字节，本机 link 安装，不是 npm 包）
```
export const inject = ['webServer']
GET  /feiyu/ping      → { ok, 版本 }
GET  /feiyu/sessions  → { ok, 会话: [{ sid, 消息数 }] }        // sid 去掉了 session- 前缀
GET  /feiyu/history?sid=&after=&limit=  → { ok, sid, 总数, 消息: [{id, role, text, turn, at, kind}] }
POST /feiyu/send      → { sid, text } → { ok, 说 }              // 走 agent.followup，source.kind='user'
GET  /feiyu/debug?sid= → { ok, sid, 有活agent, 唤醒, 形状, 服务 }
```
- 无鉴权（`access-control-allow-origin: *`，任意 origin 可 POST）
- 历史只在**内存环形缓冲**里（上限 300 条），**重启即丢**
- 端口 = DSH Web 端口（当前 57009）
- 实测：`GET /feiyu/ping` → `{"ok":true,"版本":"0.1.0"}`；`/feiyu/sessions` 返回 11 个会话
- **实测：11 个会话里没有 `429facb2`（大肥鱼生产会话）** → 手机端目前拿不到大肥鱼那间会话的历史

## E-06 NapCat / QQ 通道当前不在线
`GET http://127.0.0.1:3000/` → `TypeError: fetch failed`（连接被拒）
而 `~/.dsh/qq-bridge-config.json` 里 `总开关=true`、NapCat 指向 `http://127.0.0.1:3000`
→ **QQ 通道是"配置开着、进程没起"的状态**。不能报 PASS。

## E-07 dsh-speak-up 0.2.7 源码实测（主动发言）
文件：`~/.dsh/profiles/desktop/node_modules/dsh-speak-up/lib/host.js`（1298 行）
- L460-475 `找agent(ctx,id)`：**只用 `ctx.agents.get()`**，两种 id 写法都试，拿不到返回 undefined
- L676-697：遍历配置的目标会话，`找agent` 为 undefined 就记 `不可用.push('... (没活着)')`；
  可用列表为空时 **L693-696 明确 `return 记账('... 按设计**不自动唤醒**，这轮什么都不做')`**
- L760-783：真正发送 = `await agent.followup(消息)`，消息是手搓的 `{id, role:'user', content:[{type:'text'}], source:{kind: 源标记}}`
- L802-816：定时器 = `ctx.interval(cb, 间隔分钟*60*1000)`，**固定间隔轮询 + 掷骰**（不是事件驱动）
- L719-747：命中线算法 = 基础命中线（峰25%/谷85%）+ 距上次说话时间因子 + 连没中加成（+15/次，上限50）
- L753-757：`角色='developer'` 会被主动拦下（当前 DSH 送不了 developer 角色）
→ **结论：dsh-speak-up 是"只在活着的会话里说话"，死不唤醒。这是它的根本限制，不是配置问题。**
→ 当前 `speak-up-config.json` 的 `总开关=false` → **主动发言整个是关的，任何"主动发言可用"的结论都是假的**。

## E-08 dsh-Go 插件体系文档（原文，非二手）
source: `ccch1mneyyy/dsh-Go@8fe1c1cd804ad763dd357bc1dec4b2cbfdc6f7fa`
`docs/plugins/native-for-free.md` 原文给出"不写 App 代码就原生显示的六样东西"：
1. 斜杠命令 `ctx.commands` → 输入框 `/` 菜单
2. 技能 `SKILL.md` → `/` 菜单技能组、「技能」页
3. 插件配置（导出 schemastery `Config`）→ 原生表单（`.volatile()` 即改即生效；`role('secret')` 只写不读）
4. 工具 `defineTool` → 通用工具行（摘要取值顺序：command → file_path → path → pattern → query → url → description → prompt → name → 第一个字符串参数）
5. 模型输出 ```dsh-ui 代码块 → 原生图表/指标/对比卡（≤4KiB、≤40 节点）
6. 工具 `presentationMeta` 里放 `"dsh-go/view"` → 工具行展开区的原生卡片
只有这六样覆盖不到，才需要 package.json 里的 `dshGo` 清单（胶囊 / 消息长按菜单 / 状态条 / 原生面板）。

`docs/guide/zh/03-engine-hosts.md` 原文关键事实：
- 引擎可以**跑在手机上**（本机引擎），也可以远程连电脑
- 「打开 App 时自动启动」默认开；「自动进入安全模式」连续两次启动失败后不加载插件启动
- 「后台运行」章节在 11-faq.md（需另行核对）
- 经 DSH Remote 连接时：`App 内打不开网页版；插件只能开关；DS Bot 和导出对话日志不可用`
