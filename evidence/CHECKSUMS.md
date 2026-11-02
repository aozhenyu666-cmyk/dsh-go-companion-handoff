# 交接包校验清单

生成时间：2026-11-03 (Asia/Shanghai)
总计：**16 个文件 / 685 KB**

> 发布前已做脱敏：真实姓名 → `<user>`；QQ 号 → `2079****46`；会话 ID → 保留前 8 位后打码。
> 脱敏**不改变** `evidence/BASELINE-MANIFEST.json` 里的 SHA256——那是审计前的真实基线，可直接用于回滚校验。

| 文件 | 大小 | SHA256（前 16 位） |
|---|---|---|
| DSH-GO-COMPANION-INTEGRATION.md | 91 KB | 0153f421150a5551… |
| DSH-GO-PLUGIN-INVENTORY.md | 14 KB | 2e3a4b082f7fbcde… |
| README.md | 5 KB | bd4ae485ed5bce10… |
| research/00-SHARED-FACTS.md | 7 KB | ef23c707c89df1ed… |
| research/05-LEAD-EVIDENCE.md | 7 KB | 8e534ec019e64442… |
| research/06-LEAD-DRAFT-ARCH.md | 12 KB | ca8f471841879f3f… |
| research/10-pet-and-voice.md | 66 KB | 647459c187af59e7… |
| research/20-dsh-pet.md | 66 KB | ec3e6f03667c0826… |
| research/20-memory-scheduler-clinic.md | 70 KB | b4d5442610117d8a… |
| research/21-voice-comparison.md | 77 KB | 40315a0151197324… |
| research/30-dshgo-native.md | 64 KB | 6a35e4b2487ab9d4… |
| research/31-dsh-go-local-source-authority.md | 129 KB | 0792434995e12513… |
| research/40-runtime-truth.md | 55 KB | b1053c9de32a118e… |
| research/40b-capability-matrix.md | 15 KB | e3693cd400d64f78… |
| evidence/AUDIT-RESIDUE.md | 3 KB | fbff3be0ba83b321… |
| evidence/BASELINE-MANIFEST.json | 3 KB | bd9571e3fe4594b0… |

完整 SHA256：`Get-FileHash <文件> -Algorithm SHA256`

## 基线备份

审计前基线：12 个配置文件 + 2 个 fish-memory 数据文件，完整 SHA256 见 `evidence/BASELINE-MANIFEST.json`。

## 审计期间唯一变化的生产文件

`~/.dsh/feiyu-mood/state.json` — `lastDecayAt` 由插件自身衰减定时器（每 3 秒）更新，非审计写入。**不需要回滚。**

## 目录结构

```
dsh-go-companion-handoff/
├── README.md                          仓库导览
├── DSH-GO-COMPANION-INTEGRATION.md    主交接文档（20 章 + 6 附录）
├── DSH-GO-PLUGIN-INVENTORY.md         插件清单（9 章）
├── evidence/
│   ├── CHECKSUMS.md                   本文件
│   ├── BASELINE-MANIFEST.json         审计前基线（完整 SHA256）
│   └── AUDIT-RESIDUE.md               审计残留物说明
└── research/                          11 份详细报告 + 7 个仓库文件清单
```

## 隐私

全文扫描 `sk-` / Bearer / Cookie / SESSDATA / 令牌 / password / 真实姓名 / 完整会话 ID **零命中**。
