# 变更记录

## [1.0.0] - 2026-09-30

首个公开发布版本。

### Added

- **MultiGame 模型**（`laya_multigame_v1`）：Sakura no Uta + Aokana 联合微调，背靠 `jhu-clsp/mmBERT-base`（ModernBERT，22 层，hidden 768），2 层分类头。
- **模型配置**：`training_config.json`（训练超参）、`rl_agent_config.json`（推理配置）、`encoder/config.json`。
- **分词器**：`tokenizer/`，以及 `checkpoint_epoch1/` 下的 epoch 1 独立检查点。
- **评测结果**：`sakura_multigame_v1_results.json`、`aokana_multigame_v1_results.json`，含逐样本概率与完整混淆矩阵。
- 项目 Logo `gal_laya.png`。
- `LICENSE`（MIT）、`CONTRIBUTING.md`、`CITATION.cff`、`.gitignore`。

### Results

| 测试集 | 规模 | Accuracy | MCC |
| --- | ---: | ---: | ---: |
| Sakura no Uta（JA） | 13,320 | 0.9991 | 0.9982 |
| Aokana（JA, held-out） | 480 atomic / 80 decision points | 0.8792 | 0.7601 |

Aokana 按 decision point 聚类做 cluster bootstrap：相对 Pretrained 基线 atomic accuracy +35.82 pp（95% CI `[+30.42, +41.25]`），decision-level exact match 6/6 提升 +57.53 pp（95% CI `[+46.25, +68.75]`）。

### Known Limitations

- Aokana 结果为 within-game held-out evaluation，不等于 unseen-game zero-shot。
- WHITE ALBUM 2 尚未进入训练，`selection → script bridge` 仍为 UNKNOWN，其 A/B 标签与 route oracle 均未生成。
- 训练与评测脚本、dataset schema、LSCR / VM 逆向工具尚未开源。

### Distribution

模型权重约 614 MB / 个，不进入 Git history，通过 GitHub Releases 分发。

[1.0.0]: https://github.com/thanatosmoe/Laya-GAL/releases/tag/v1.0.0
