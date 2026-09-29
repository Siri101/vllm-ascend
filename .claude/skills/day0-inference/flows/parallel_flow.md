# Parallel Flow（占位）— Day0 Stage 2：并行量化版本（跑得稳）

> ⚠️ **占位声明**：本 flow **除精度环节（见下方"精度环节（已实现）"节）外均未实现**。主控 `SKILL.md` 推进到 Stage 2 时，应**显式提示【并行量化 flow 除精度环节外尚未接入】**，输出本文件的框架定义供人工或后续版本接管，不得自行编造执行步骤；精度环节按已实现口径执行。

## 阶段目标（对齐 AscendBot 四阶段定义）

基于 Golden 基线版本，按选定的**量化策略**与 **KV 缓存方案**，设计合理的**并行策略**（TP / EP / DCP / PCP），完成推理服务部署运行，确保推理精度正确——构建**资源使用合理的版本**。

## 入口条件（Stage 1 出口证据）

- Golden 阶段 G0-G4 全过：`./.day0/<model>/` 下 preflight/design/impl/smoke/accuracy/review 产物齐全；
- eager + bf16 精度基线（G3 证据）归档——本阶段的精度对比基准。

## 规划出口判据（草案，实现时细化）

1. **并行策略合法性**：TP/EP/DCP/PCP 整除与互斥约束算术校验通过（MLA 模型 `TP % DCP == 0`；GQA `num_q_per_kv % DCP == 0`；PCP/DCP 互斥且 PCP 仅 MRV2）；
2. **量化路径正确**：量化格式自动检测命中、反量化路径无权重缺失/尺寸不匹配（grep 口径见 `.claude/agents/accuracy.md`），EPLB 量化白名单校验（如需 EPLB）；
3. **服务部署运行**：目标并行配置下真实权重拉起成功，HTTP 200 且输出非空；
4. **精度对齐**：并行 + 量化配置下精度对齐 Golden eager 基线，量化损失在约定阈值内（阈值随模型量化格式在实现时定义）；
5. **组合回归**：量化 × 并行组合纳入 E2E 配置（`tests/e2e/models/configs/<Model>.yaml`）。

## 已知技术难点（实现时需覆盖）

- 并行切分策略需随量化权重分布动态重算；
- 量化策略 × 并行方案组合多，验证矩阵需显式裁剪；
- KV cache 方案（block size、C8、DCP 分片）与并行策略联动校验。

## 精度环节（已实现）

S2.4 量化精度对齐按 `.claude/skills/day0-inference/reference/precision-alignment-stages.md` 的 S2.4 参数卡执行：探针五步对比 **S1 接力锚点**（`accuracy/baseline/`），主判据锚点逐 token 一致（L2）；量化是近似——Designer 显式定义容差 + 主控签字才可放宽（容差以题集判分承载）。产物落 `parallel/probe/`；门禁通过后 `emit_baseline.py` 采 `parallel/baseline/` 接力锚点交 S3。失败按三分类路由，劣化归因指向量化/并行配置变更。

## 产物目录（规划）

`./.day0/<model>/parallel/`：并行策略报告、量化验证证据、精度对比报告。
