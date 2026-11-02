# 共享事实底稿（Lead 已核实，2026-11-02 22:4x +08:00）

> 本文件是事实基线。凡与此冲突的，以本文件为准；凡本文件没有的，必须自己验证并标注证据来源。

## 1. 环境实测
- DSH Desktop 版本：0.2.1-rc.2（来自 resources/runtime/primary-runtime/runtime.json 的 desktopVersion）
- Node（DSH 运行时自带）：24.21.0（runtime/versions.json）
- Node（PATH 上的）：v24.19.0
- pnpm（自带）：11.28.5
- Cordis：@deepseek-ai/cordis ~4.0.5（来自 profile 的 pnpm-lock.yaml peer 解析）
- @deepseek-ai/dsh-tools / dsh-llm / dsh-skill：0.2.1-rc.2
- schemastery：3.18.5
- 平台：win32 x64，用户名目录 C:\Users\<user>
- DSH 用户目录：C:\Users\<user>\.dsh
- DSH 安装目录：C:\Users\<user>\AppData\Local\Programs\DeepSeek Harness\resources

## 2. Profile
- 存在三个 profile：desktop（生产，正在用）、headless（空 patch）、abtest-zseven（空 patch）
- 生产 profile 目录：C:\Users\<user>\.dsh\profiles\desktop
- 挂载方式：package.json 的 dsh.profile.bundles 数组 + cordis.patch.yml 配置层
- desktop 的 bundles（31 项，顺序即加载顺序）见 package.json

## 3. desktop profile 已装插件与版本（node_modules/package.json 实测）
| 包名 | 版本 | 仓库 |
|---|---|---|
| dsh-fish-memory | 1.4.3 | github.com/Yuuyuko-uu/dsh-fish-memory |
| dsh-feiyu-mood | 0.8.1 | github.com/Yuuyuko-uu/dsh-feiyu-mood |
| dsh-speak-up | 0.2.7 | github.com/Yuuyuko-uu/dsh-speak-up |
| dsh-bili-subagent | 0.8.7 | github.com/Yuuyuko-uu/dsh-bili-subagent |
| dsh-qq-bridge | 1.4.0 | github.com/Yuuyuko-uu/dsh-qq-bridge |
| dsh-pet | 0.3.8 | github.com/PC2005-cloud/dsh-pet |
| dsh-feiyu-app | 0.1.0 | 本地 link: C:/Users/<user>/feiyu/plugin |
| surfcore | 0.7.10 | 本地 link: C:/Users/<user>/.dsh/plugins/surfcore |
| dshmarket | 1.66.47 | github.com/dsh-market/dsh-market |
| dsh-context | 0.73.3 | github.com/bowenliang123/dsh-context |
| dsh-cost-meter | 1.9.22 | github.com/Han-1413141/dsh-cost-meter |
| @zhz8888/dsh-android-emulator-helper | 1.2.0 | github.com/zhz8888/dsh-android-emulator-helper |
| @zseven-w/dsh-android | 0.1.0-rc.12 | - |
| @nanmicoder/dsh-agent-teams | 0.1.25 | github.com/NanmiCoder/dsh-agent-teams |
| cc-safety-net | 2.6.16 | - |
| dsh-better-sidebar | 0.28.0 | - |
| @wenaixi/dsh-superpower | 8.0.0 | github.com/Wenaixi/dsh-superpower |
| @maci0/dsh-caveman | 0.18.0 | - |
| dsh-flash-presets | 0.2.2 | github.com/LarryE135/dsh-flash-presets |
| dsh-startup-screen | 1.1.2 | github.com/6shenhonghong9/dsh-startup-screen |
| dsh-whale-widget | 0.4.2 | github.com/MeteorNOX/DeepSeek-Balance-Whale-Widget |
| dsh-emil-skills | 1.0.0 | - |
| dsh-plugin-wallpaper-engine | 1.4.0 | - |
| dsh-i-have-adhd | 1.0.2 | - |
| @deepseek-ai/dsh-subagent-codex | 0.2.1-rc.2 | - |
| @michengai/dsh-skills-manager | 1.2.9 | - |
| @liustack/modsearch | 5.10.10 | - |

注意：`dsh-qq-link` 实际不存在，真实包名是 `dsh-qq-bridge@1.4.0`。
注意：存在 `dsh-feiyu-app`（本地手机端后端，link），会与 dsh-qq-bridge 竞争同一会话。

## 4. 运行时状态文件（实测内容摘要）
- ~/.dsh/speak-up-config.json：**总开关=false**，间隔10分钟，峰时命中率25，谷时85，安静起1点/止8点，目标会话 session-429facb2-...  → **主动发言当前是关的**
- ~/.dsh/feiyu-mood/config.json：targetSession 同上，mode=session，slot __default__ 标签"大肥鱼"，ownerName"鱼片"，injectMode=event，injectPerDay=6，autoJudge=true，decayHalfHours=12，moodBase=70
- ~/.dsh/feiyu-mood/state.json：version=2，moods.__default__ mood=70，dims 全 0，timeline=[]，lastDecayAt=1793629686246
- ~/.dsh/feiyu-mood/diag.log：最后一条 2026-10-30T13:50:18 "已挂心情到 session-429facb2-..."，之后无活动
- ~/.dsh/qq-bridge-config.json：总开关=true，目标会话 session-429facb2-...，NapCat http://127.0.0.1:3000，私聊QQ 2079****46，绑定称呼"鱼片"
- ~/.dsh/fish-memory/：只有一个 session-2c93cd4f-....json（47428 字节）+ .bak，另有 ~/.dsh/fish-memory-backup/2026-10-31
- ~/.dsh/fish-memory-diag.log：有 "assemble 的工具（1 个）：run_code / ↑ 里面有 remember 吗：没有 ✗" —— **说明某些会话里 remember 工具没挂上，记忆写入链路可疑**
- ~/.dsh/dsh-pet/main-config.jsonc：pets[0] id=main name="蓝毛小女仆" size=462 display=web corner=top-right
- ~/.dsh/surfcore/config.json：platforms.start.qq=false / bili=false，bridge.autoStart=false，updates.enabled=false
- ~/.dsh/bili-duty/：asr.py、mcp_server.py(62KB)、duty.log、seen.md
- ~/.dsh/speech-to-text/sensevoice/：本地 SenseVoice STT 引擎目录
- ~/.dsh/storages/：cost-meter、session_projcache、workspace.json

## 5. 七个待研究仓库（已核实存在 + 已锁定 commit）
| 仓库 | default branch | 锁定 commit SHA | 文件数 | 最新推送 |
|---|---|---|---|---|
| zhu1090093659/dsh-pet | main | 9ed843bccee3d2604ce0ac31f55cbde9bda97c7f | 3831 | 2026-11-02T08:15Z |
| qishuilalala/dsh-voice-mode | main | 3524db40938f16e7c8f0726e30647afe5bfbef25 | 304 | 2026-10-30T19:33Z |
| PerryLink/dsh-talk | main | 6a5f768b2607ec1204fc256ede492c2abd39a3e1 | 88 | 2026-11-01T09:23Z |
| Aik358/dsh-auto-memory | main | a244cf03f6f409b655d86e02d6c0a7b86d7496f4 | 1772 | 2026-11-02T09:12Z |
| ljsysfurryACE/dsh-aura-scheduler | master | f9048cd59399dbc2b38b0d3be4a750342b8b72ca | 16 | 2026-09-24T11:13Z |
| PerryLink/dsh-clinic | main | bd1b3e84736c7fc23b89342cae94487e15f08504 | 130 | 2026-11-02T13:50Z |
| ccch1mneyyy/dsh-Go | main | 8fe1c1cd804ad763dd357bc1dec4b2cbfdc6f7fa | 4641 | 2026-11-02T14:01Z |

star 数：dsh-pet 60、voice-mode 25、dsh-talk 18、auto-memory 179、aura-scheduler 4、dsh-clinic 0、dsh-Go 5559。

## 6. 网络与工具约束（重要）
- 本机 **npm 不在 PATH** 上；pnpm 在 "C:\Users\<user>\AppData\Local\Programs\DeepSeek Harness\resources\runtime\primary-runtime\dependencies\pnpm"
- `git clone` 极慢/会超时；`codeload.github.com` 的 tarball 下载约 30KB/s，十几 MB 的会很久
- **推荐**：用 `gh api "repos/OWNER/REPO/contents/<path>" --jq .content` 或 `fetch https://raw.githubusercontent.com/OWNER/REPO/<SHA>/<path>`（后者 1-3 秒/文件，最稳）
- 已下载的完整文件清单在：_research/trees/<owner>__<repo>.txt（每行一个 blob 路径）
- GitHub API 速率对你可能有限，注意别刷爆；优先读关键源码
- 工具 `pwsh` 需要传 description 参数；PowerShell 有中文编码问题，**读文件请用 node fs 或 `Get-Content -Encoding utf8`**，不要用 Select-String 读含中文的 JSON
- `run_code` 里不能写 import 语句在顶层（需 await import()）

## 7. 纪律（违反即作废）
- 只读：**禁止修改** desktop profile、禁止改 ~/.dsh 下任何生产配置、禁止装插件、禁止改 DSH-Go 主仓库
- 禁止向 QQ/B站等外部渠道发送任何消息；所有网络发送类功能保持关闭
- 不得伪造测试结果；没跑过就是 NOT_TESTED
- 每个结论必须带证据：文件路径 + 行号 或 命令输出
- 不得把"插件加载成功"当"功能验收通过"
- 不得把模拟器测试当 Android 真机测试
