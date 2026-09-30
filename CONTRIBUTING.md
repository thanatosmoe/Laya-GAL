# 贡献指南

## 首要规则

**不要提交任何商业 GAL 的原始游戏资源。** 这包括 `*.exe`、`*.pak`、`*.bnr`、`*.arc`、`*.xp3`、`*.nsa`、`VOICE.PAK`、`BGM.PAK`、`CHAR.PAK`、原始商业游戏脚本与对白，以及任何受版权保护的游戏资源。相关 PR 会被直接关闭。

同样不要提交模型权重（`*.safetensors`、`*.bin`）——这些通过 GitHub Releases 分发，由维护者上传。`.gitignore` 已覆盖以上两类。

## 可以提交什么

- 解析与逆向工程工具（LSCR / VM 反汇编、scene graph 构建）
- dataset schema、生成的 metadata、索引信息（不含原文）
- 审计报告与可复现性检查脚本
- 训练与评测代码、模型配置
- 不含受版权保护内容的小型 synthetic examples

## 报告问题

请开 issue 并说明：

- 涉及的作品与 decision point（如可公开）
- 期望行为与实际行为
- 若与数据相关，说明样本是如何从 `sample → decision → choice → source script → branch → oracle` 追溯得到的

## 提交 Pull Request

1. Fork 并基于 `main` 建立分支。
2. 保持改动聚焦，不在一个 PR 里混入无关重构。
3. 若改动影响评测结果，请在 PR 描述中给出前后对比，并说明是否改变了数据切分。
4. 提交前自测：训练相关改动请附上配置与运行命令，评测相关改动请附上指标输出。

### 数据与评测的硬性要求

任何新增数据或评测协议都必须满足：

1. **Source traceability** —— 每个样本可回溯到源脚本、分支与 oracle。
2. **No future leakage** —— 模型输入不得包含后续剧情、最终路线、oracle 输出、future dialogue 或明示答案的 metadata。
3. **Decision-level split** —— 同一 decision point 的多条候选路线不得跨 train / validation / test。
4. **Oracle independence** —— 模型预测不得参与 ground truth 生成。

若无法证明标签来源，就不要生成标签。

## 许可

贡献即表示你同意以本仓库的 [MIT](LICENSE) 许可发布你的贡献。
