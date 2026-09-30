<div align="center">

# Laya-GAL

<img src="gal_laya.png" alt="Laya-GAL 看板娘" width="260">

</div>

GAL（视觉小说）路线可达性预测。给定当前剧情状态、玩家当前面对的选择，以及一条候选角色路线，判断该路线在选择之后是否仍然可达。

[下载模型](https://github.com/thanatosmoe/Laya-GAL/releases) · [实验结果](#实验结果) · [数据集](#数据集) · [模型与训练](#模型与训练) · [评测协议](#评测协议) · [Roadmap](#roadmap) · [贡献指南](CONTRIBUTING.md) · [变更记录](CHANGELOG.md)

输入保留真实的日文剧情文本与选项，`A / B` 只是模型内部的二分类标签，不会替换游戏中的真实选择。Ground truth 来自脚本解析、VM 语义与状态感知的可达性 oracle，不来自模型预测。

## 快速开始

权重与配置通过 [GitHub Releases](https://github.com/klarkxy/laya-multigame/releases) 分发，不进入 Git history。下载对应版本的 `laya_multigame_v1.tar.gz` 后解压到仓库根目录即可：

```text
model.safetensors        # MultiGame 模型权重（614 MB）
encoder/config.json      # ModernBERT 编码器配置
tokenizer/               # 分词器
checkpoint_epoch1/       # epoch 1 检查点
```

运行环境：Python 3.12，bf16 推理，单卡 23GB 显存（A10）可完整加载。`training_config.json` 与 `rl_agent_config.json` 分别给出训练超参与推理配置（`max_len` 1024、`head_max_len` 256、`max_prefixes` 6）。

训练与评测脚本、dataset schema 及 LSCR / VM 逆向工具将在后续版本整理发布，见 [Roadmap](#roadmap)。

## 这是什么

GAL 的剧情结构本质上是一张受状态约束的有向图：玩家的每次选择都会关闭一部分后续路线。Route Reachability 把这个问题形式化为可监督、可评测的二分类任务：

```text
        State + Current Choice + Candidate Route
                          │
                          ▼
                 A (still reachable) / B (no longer reachable)
```

- **多游戏联合训练** —— Sakura no Uta 与 Aokana 的 route-reachability 数据联合微调，显著提升跨作品判断能力。
- **决策点级切分** —— 同一 decision point 下的多条候选路线不跨 train / validation / test。
- **聚类统计** —— 同一决策点内的判断并非独立样本，主比较按 `sample_id` 做 cluster bootstrap 并报告置信区间。
- **可追溯构建** —— 每个样本可回溯至 `sample → decision → choice → source script → branch → oracle`。

## 实验结果

### 主对比

| Model | Sakura-JA Test | Aokana-JA Test |
| --- | ---: | ---: |
| Pretrained (mmBERT-base) | 49.14% | 52.08% |
| Sakura Head-only FT | 82.05% | 52.71% |
| Sakura Full FT | 99.95% | 49.17% |
| **MultiGame FT (Ours)** | **99.91%** | **87.92%** |

### MultiGame · Aokana-JA held-out

测试集为 80 个 decision points × 6 条候选路线 = **480 atomic decisions**。

| Metric | Value |
| --- | ---: |
| Accuracy | 0.8792 |
| Macro-F1 | 0.8788 |
| Balanced Accuracy | 0.8785 |
| MCC | 0.7601 |
| Decision-level exact match (6/6) | 46 / 80 = 57.5% |

### Cluster Bootstrap

按 decision point 聚类，n = 80。

| Comparison | Atomic Accuracy | Decision exact 6/6 |
| --- | ---: | ---: |
| MultiGame vs Pretrained | +35.82 pp `[+30.42, +41.25]` | +57.53 pp `[+46.25, +68.75]` |
| MultiGame vs Head-only | +35.21 pp `[+28.54, +41.88]` | +57.53 pp `[+46.25, +68.75]` |
| MultiGame vs Sakura Full FT | +38.75 pp `[+33.33, +44.17]` | +57.53 pp `[+46.25, +68.75]` |

Aokana 结果是 **within-game held-out evaluation**：Aokana 的其它 decision points 参与了 MultiGame training，因此 87.92% 不等价于「完全未见该作品的 zero-shot 结果」。

## 数据集

### Sakura no Uta

| Item | Value |
| --- | ---: |
| Reachable atomic samples | 131,064 |
| Train / Val / Test | 105,100 / 12,644 / 13,320 |
| 完整选择树 | 14 menus · 28 options · 4 terminal routes · 32,766 decision-prefix states |
| 原子任务总规模 | 450,208（reachable / locked / possible-final / forced） |

Test 集 13,320 条：accuracy 0.9991，MCC 0.9982（gold A = 5,986 / B = 7,334）。可追溯性审计：staged traceability `34/34`、option → branch `34/34`、route-set labels `34/34`、future-information leak `0`。

### Aokana

使用日文原文，按 `sample_id` 做 decision-point 级切分。

| Item | Value |
| --- | ---: |
| All source decision points | 217 |
| Held-out test decision points | 80 |
| Test atomic decisions | 480（A = 236 / B = 244） |

### WHITE ALBUM 2

WHITE ALBUM 2 未参与任何训练，当前处于逆向工程阶段。

```text
LSCR format / VM          187 / 187 .bnr exact parse
.fnc command table, command dispatch
CF op2 / op13 / op4 semantics, EXPR 8 / 16 / 27 / 30
VM storage model, real player-choice mechanism
34 decision points · 68 real Japanese option texts
scene graph: 187 nodes · 208 scene-level edges
```

已确认真实选择机制为 `Player Input → selection index → SetSelectMess → native selection machinery`，但 `selection → script bridge` 仍为 UNKNOWN。因此 `choice_edges = 0`、`A/B labels = 0`、`route oracle = not frozen`。证明不了的标签不会生成。

## 模型与训练

| Field | Value |
| --- | --- |
| Backbone | `jhu-clsp/mmBERT-base`（ModernBERT，22 layers，hidden 768，vocab 256k） |
| Head | 2-layer classification head |
| Max length | 1024（head 256）· max prefixes 6 |
| Training items | 210,200（Sakura 105,100 + Aokana 105,100，Aokana 由 660 条源样本采样） |
| Epochs / updates | 1 / 3,279 |
| Micro batch × Grad accum | 8 × 8 |
| Learning rate | encoder 2.5e-5 · head 1e-4 |
| AMP / Hardware | bf16 · A10 23GB · wall time 4.97 h |
| Seed | 20260927 |
| Laya version | 0.3.20（commit `23a1752`） |

## 仓库结构

```text
.
├── gal_laya.png
├── model.safetensors
├── encoder/config.json
├── tokenizer/
│   ├── tokenizer.json
│   └── tokenizer_config.json
├── checkpoint_epoch1/
│   ├── checkpoint_meta.json
│   ├── model.safetensors
│   ├── encoder/
│   └── tokenizer/
├── training_config.json
├── rl_agent_config.json
├── sakura_multigame_v1_results.json
├── aokana_multigame_v1_results.json
├── CONTRIBUTING.md
├── CITATION.cff
├── LICENSE
└── README.md
```

## 评测协议

1. **Source traceability** —— 每个样本可回溯到源脚本、分支与 oracle。
2. **No future leakage** —— 模型输入不包含后续剧情、最终路线、oracle 输出、future dialogue 或明示答案的 metadata。
3. **Decision-level split** —— 同一 decision point 的多条候选路线不跨 train / validation / test。
4. **Oracle independence** —— 模型预测不参与 ground truth 生成。

## Roadmap

- [x] Sakura dataset / oracle
- [x] Sakura Laya training
- [x] Aokana Japanese dataset / oracle
- [x] Aokana held-out evaluation
- [x] MultiGame training
- [x] Cluster bootstrap
- [ ] WHITE ALBUM 2 choice → branch bridge
- [ ] WHITE ALBUM 2 flag dependency graph
- [ ] WHITE ALBUM 2 route definition
- [ ] WHITE ALBUM 2 state-aware reachability oracle
- [ ] WHITE ALBUM 2 unseen-game inference
- [ ] Cross-game analysis & final ablations
- [ ] Reproducible release（scripts / tools / dataset schema）

## 局限

当前结果不支持「Laya 已具备通用 GAL 推理能力」「87.92% 代表完全 unseen-game 泛化」「模型理解所有 GAL 剧情」「WA2 oracle 已完成」或「跨作品能力已被第三部作品证明」。

当前可以明确支持的结论是：

> 在 Sakura no Uta + Aokana 的多游戏训练实验中，MultiGame fine-tuning 在 Aokana 的 held-out decision points 上显著优于现有基线；真正的 unseen-game 泛化仍需第三部 GAL 的独立测试验证。

## 版权与合规

本仓库的目标是逆向工程研究、可复现的机器学习实验、route-reachability 建模与相关工具。

**本项目不包含、也不应包含任何商业 GAL 的原始游戏资源。** 请勿上传 `*.exe`、`*.pak`、`*.bnr`、`VOICE.PAK`、`BGM.PAK`、`CHAR.PAK`、原始商业游戏脚本与对白，或其他受版权保护的游戏资源。Sakura no Uta、Aokana、WHITE ALBUM 2 的原始资源均不构成本项目的公开内容。使用者应自行合法拥有相应游戏本体。

## 引用

若本项目对你的研究有帮助，请引用 [CITATION.cff](CITATION.cff)。

## 许可

[MIT](LICENSE)
