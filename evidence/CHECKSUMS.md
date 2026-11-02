# 交接包校验清单

生成时间：2026-11-02 23:50 (+08:00)
总计：**15 个文件 / 680 KB**

| 文件 | 大小 | SHA256（前 16 位） |
|---|---|---|
| DSH-GO-COMPANION-INTEGRATION.md | 91 KB | 3782be6662916d49… |
| DSH-GO-PLUGIN-INVENTORY.md | 14 KB | c8a742bcd76d9640… |
| research/00-SHARED-FACTS.md | 7 KB | a0500c7b70260192… |
| research/05-LEAD-EVIDENCE.md | 7 KB | e8ec2ab946cd0d75… |
| research/06-LEAD-DRAFT-ARCH.md | 12 KB | 7ecce716f1ff9058… |
| research/10-pet-and-voice.md | 66 KB | 647459c187af59e7… |
| research/20-dsh-pet.md | 66 KB | 7fa18246a7e0a6ae… |
| research/20-memory-scheduler-clinic.md | 70 KB | 85738c1ab59b86cb… |
| research/21-voice-comparison.md | 77 KB | 0705c1305470eab5… |
| research/30-dshgo-native.md | 64 KB | 6a35e4b2487ab9d4… |
| research/31-dsh-go-local-source-authority.md | 129 KB | 0792434995e12513… |
| research/40-runtime-truth.md | 55 KB | a6a6515c00e65f40… |
| research/40b-capability-matrix.md | 15 KB | e3693cd400d64f78… |
| evidence/AUDIT-RESIDUE.md | 3 KB | 095a3358cc8e7b28… |
| evidence/BASELINE-MANIFEST.json | 3 KB | 6dbb9637063718cd… |

完整 SHA256：`Get-FileHash <文件> -Algorithm SHA256`

## 基线备份

`C:\Users\敖震宇\.dsh\backups\dsh-ecosystem-audit-2026-11-02T2230\` — 12 个配置文件 + 2 个 fish-memory 数据文件，完整 SHA256 清单在 `evidence/BASELINE-MANIFEST.json`。

## 审计期间唯一变化的生产文件

`~/.dsh/feiyu-mood/state.json` — `lastDecayAt` 由插件自身衰减定时器（每 3 秒）更新，非审计写入。**不需要回滚。**

## 目录结构

```
dsh-go-handoff/
├── DSH-GO-COMPANION-INTEGRATION.md    主交接文档（20 章 + 6 附录）
├── DSH-GO-PLUGIN-INVENTORY.md         插件清单（9 章）
├── evidence/
│   ├── CHECKSUMS.md                   本文件
│   ├── BASELINE-MANIFEST.json         审计前基线（完整 SHA256）
│   └── AUDIT-RESIDUE.md               审计残留物说明
└── research/                          11 份详细报告
```

## 隐私

QQ 号已脱敏（`2079****46`）。全文扫描 `sk-` / Bearer / Cookie / SESSDATA / 令牌 / password **零命中**。
