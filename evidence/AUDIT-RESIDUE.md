# 审计残留物清单（必须知道的三处）

本次审计是**只读**的，但为了验证链路，产生了三处「有意为之的残留」。全部可清理，全部有记录。

## 1. 生产文件：~/.dsh/feiyu-mood/state.json 的 lastDecayAt 变化

- 备份时 SHA256：`82a92e47894c…`
- 审计结束时：`34c2db5a8730…`
- **原因**：feiyu-mood 自己有 `ctx.interval(tick, 3000)` 衰减定时器，每秒都在算衰减并写 `lastDecayAt`。这是插件自身的正常行为，不是审计写入的。
- **处理**：不需要回滚。但回滚配置时如果用 `Copy-Item` 覆盖这个文件，会把这 3 小时的衰减进度回退，无害。

## 2. 新增数据文件：~/.dsh/fish-memory/session-6489a146-….json

- 这是**本次审计所在会话**（session-6489a146-****）的记忆文件，由 fish-memory 插件在会话首次挂载时自动创建。
- 内容：`memories: 0`（测试记忆已删）、`deleted: 1`（回收站里那条测试记忆）、`traces: 6`
- **回收站里的那条**：`id=mmvzdk3h4894`，text = 「【审计测试·可删】2026-11-02 记忆写入链路自测：这条由 lead 在 fishy-audit 审计中写入，验证 remember → 落盘 → 召回 → 删除 全链路，测完即删。」
- **处理（三选一）**：
  - 保留：它是「已删除」状态，不参与召回，不影响任何东西
  - 清理回收站：调 `POST /dsh-fish-memory/delete {session, id, action:"purge"}`（如果插件支持 purge 动作）
  - 直接删文件：`Remove-Item ~/.dsh/fish-memory/session-6489a146-*.json*`（这个会话以后还能自动重建）
- **建议**：保留。它是这次审计唯一的写入证据，删了就没法复核 T02–T06 那五步实测了。

## 3. 审计工作目录（不影响 DSH）

| 目录 | 内容 | 处理 |
|---|---|---|
| `dsh-go-handoff/` | **交付物**（2 个主文档 + research/ 11 份报告 + evidence/） | **保留** |
| `fishy-audit/` | 各 teammate 的工作目录（备份、克隆的仓库、中间产物） | 可删（约 60MB，含 3 个 tarball 共 57MB） |
| `_research/` | Lead 自己的临时分析文件（asar 解析脚本、inventory.json、下载的 tarball） | 可删 |

## 4. 未产生的残留（明确说明）

- ❌ 没有安装任何插件
- ❌ 没有修改 desktop profile 的 package.json / cordis.patch.yml
- ❌ 没有修改 speak-up / qq-bridge / dsh-pet / surfcore 的任何配置
- ❌ 没有向 QQ / B站 发送任何消息
- ❌ 没有创建任何 scheduled task（`schedule_list` 返回 `[]`）
- ❌ 没有创建任何 goal（`get_goal` 返回 `null`）
- ❌ 没有改 DSH-Go 主仓库（`D:\code\dshgo` 的 `git status` 只有审计前就存在的 `?? _emuwork/ _evidence/ avdhome/`）
