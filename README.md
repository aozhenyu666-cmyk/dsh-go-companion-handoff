# 大肥鱼 AI 伴侣 · DSH → DSH-Go 技术交接包

> 电脑端 DSH 对现有 AI 伴侣生态的系统性审计 + 对 DSH-Go（Android 原生）的技术交接材料。
> 生成时间：2026-11-02 / 2026-11-03 (Asia/Shanghai)

## 这是什么

一份**有真实代码依据**的交接包，不是规划书。用于让 DSH-Go 开发者依据真实接口和测试证据开展 Android 原生集成，而不需要重新研究整个 DSH 生态。

## 先说三条最反直觉的结论

1. **问题不是缺插件，是三处真实断链。** 大肥鱼主会话 `session-429facb2-…` 没有活着的 agent，而情绪、主动发言、QQ 桥三条链路都挂在它上面。
2. **7 个待研究的新插件，建议装 0 个到生产。** 其中 2 个在当前宿主/平台装不上或用不了，1 个默认会删用户数据，1 个根本没有投递能力。
3. **DSH-Go 的「本机引擎」就是在一台 Android 手机上用 proot 跑真 DSH。** 插件可原样装到手机上，不需要移植；移植压力全在「界面 + 语音 + 系统权限」。

## 目录结构

```
├── DSH-GO-COMPANION-INTEGRATION.md   主交接文档（20 章 + 6 附录）
├── DSH-GO-PLUGIN-INVENTORY.md        插件清单（9 章）
├── evidence/
│   ├── CHECKSUMS.md                  全部交付物的 SHA256
│   ├── BASELINE-MANIFEST.json        审计前基线（配置 + 记忆数据的 SHA256）
│   └── AUDIT-RESIDUE.md              审计残留物说明
└── research/                         11 份详细研究报告
```

## 主文档章节

| 章 | 内容 |
|---|---|
| 0–4 | 一句话总结 / 项目目标 / 生态清单 / 插件实际测试状态 / 新插件采用与淘汰结论 |
| 5–9 | 整体架构 / 模块调用关系 / 人格·记忆·情绪数据结构 / 主动交流事件流 / Agent 会话唤醒方案 |
| 10–13 | DSH-Go 原生插件接入 / Android 语音 / 桌宠视觉 / 权限与后台限制 |
| 14–17 | API·事件·数据协议 / 失败降级 / 测试用例 / 版本兼容矩阵 |
| 18–20 | 手机端验证步骤 / 安装部署与回滚 / 未解决问题 |
| 附录 A–F | 证据索引 / 没做的事 / 诚实声明 / 审计残留物 / 自我纠正记录 / 三条易误传的"事实" |

## 详细研究报告

| 文件 | 内容 |
|---|---|
| `research/00-SHARED-FACTS.md` | 共享事实底稿（环境、profile、插件版本、仓库 commit、网络约束） |
| `research/05-LEAD-EVIDENCE.md` | Lead 亲自实测的证据 E-01 ~ E-08 |
| `research/06-LEAD-DRAFT-ARCH.md` | 架构草稿（六子系统、数据结构、唤醒方案） |
| `research/10-pet-and-voice.md` | 桌宠三选一 + 语音两选一 |
| `research/20-dsh-pet.md` | zhu 版 dsh-pet 0.4.14 源码级审计 |
| `research/20-memory-scheduler-clinic.md` | 记忆 / 主动调度 / 插件质检 三模块 |
| `research/21-voice-comparison.md` | dsh-voice-mode vs dsh-talk 逐维对比 |
| `research/30-dshgo-native.md` | DSH-Go 原生接入事实材料 |
| `research/31-dsh-go-local-source-authority.md` | 本地真 checkout（分支 `feat/r3-candidate`）的源码权威依据 |
| `research/40-runtime-truth.md` | 大肥鱼生态当前真实状态审计 |
| `research/40b-capability-matrix.md` | 能力边界三分法 38 行矩阵 |
| `research/trees_*.txt` | 7 个待研究仓库的完整文件清单（锁定 commit） |

## 状态口径

| 标记 | 含义 |
|---|---|
| **PASS** | 实际跑过并通过，有输出/日志证据 |
| **WARN** | 有限制、部分通过、或声明与实际不符 |
| **FAIL** | 跑过且失败，或静态可证永不成立 |
| **NOT_TESTED** | 没跑过（**读源码不算 PASS**） |

**本包所有 Android 相关结论均为 NOT_TESTED**（本次审计环境无 adb、无真机）。

## 研究的 7 个仓库（全部锁定 commit）

| 仓库 | 锁定 commit |
|---|---|
| `zhu1090093659/dsh-pet` | `9ed843bccee3d2604ce0ac31f55cbde9bda97c7f` |
| `qishuilalala/dsh-voice-mode` | `3524db40938f16e7c8f0726e30647afe5bfbef25` |
| `PerryLink/dsh-talk` | `6a5f768b2607ec1204fc256ede492c2abd39a3e1` |
| `Aik358/dsh-auto-memory` | `a244cf03f6f409b655d86e02d6c0a7b86d7496f4` |
| `ljsysfurryACE/dsh-aura-scheduler` | `f9048cd59399dbc2b38b0d3be4a750342b8b72ca` |
| `PerryLink/dsh-clinic` | `bd1b3e84736c7fc23b89342cae94487e15f08504` |
| `ccch1mneyyy/dsh-Go` | `8fe1c1cd804ad763dd357bc1dec4b2cbfdc6f7fa` |

基线环境：DSH Desktop **0.2.1-rc.2** / Cordis **4.0.5** / Node **24.21.0** / pnpm **11.28.5** / win32 x64。

## 审计纪律

- 全程**只读**：未安装任何插件、未修改生产 profile、未改动 DSH-Go 主仓库、未向 QQ / B站 等外部渠道发送任何消息
- 没有把源码静态分析当成 PASS
- 没有为了"看起来完整"填任何数字
- 敏感信息（QQ 号、令牌、API Key）已脱敏或排除

## 从哪里开始读

1. 先读 `DSH-GO-COMPANION-INTEGRATION.md` 的 §0（一句话总结）和 §4（选型结论）
2. 再看 `evidence/CHECKSUMS.md` 确认交付完整性
3. DSH-Go 开发者重点看 §9（会话唤醒）、§10（原生接入）、§11（语音）、§13（权限与后台）
