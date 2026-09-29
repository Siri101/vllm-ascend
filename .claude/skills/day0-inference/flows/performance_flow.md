# Performance Flow（占位）— Day0 Stage 4：精度/性能验收版本（出口）

> ⚠️ **占位声明**：本 flow **除精度环节（见下方"精度环节（已实现）"节）外均未实现**。主控 `SKILL.md` 推进到 Stage 4 时，应**显式提示【精度/性能验收 flow 除精度环节外尚未接入】**，输出本文件的框架定义供人工或后续版本接管，不得自行编造执行步骤；精度环节按已实现口径执行。

## 阶段目标（对齐 AscendBot 四阶段定义）

基于特性叠加版本，通过**实际性能瓶颈分析实施定向调优**，并结合精度测试结果闭环修复偏差，确保交付版本**性能达标、精度合格**——构建出口达标版本。

## 入口条件（Stage 3 出口证据）

- 特性叠加版本签收：特性叠加矩阵 + 最终特性配置归档；
- Stage 3 最终配置下的精度基线与 benchmark 数据（本阶段的调优起点）。

## 规划出口判据（草案，实现时细化）

1. **性能达标**：吞吐 / latency / TTFT / TPOT 达到目标值（目标值由用户在验收启动时给定，禁止自定义口径）；benchmark 数据含完整命令、环境、硬件代次、图级别；
2. **瓶颈分析闭环**：全链路 profiling（各阶段耗时）→ 瓶颈定位 → 定向调优 → 复测，每轮调优有前后对比数据；
3. **精度合格**：全量精度测试（Golden 基线 + 组合矩阵）在最终配置下全部通过；精度偏差的修复有闭环记录；
4. **出口交付物齐备**：E2E 组合矩阵配置、教程与支持矩阵、patch 台账、signed-off commit、服务矩阵验证报告（render / tool_choice 全组合 / 多轮 reasoning / 长上下文泄漏——方法论见 `.claude/skills/day0-inference/reference/golden-service-knowledge.md`）；
5. **性能未达标项处置**：定位到算子级瓶颈且非本阶段可解的，显式转交算子团队并附 profiling 证据，不得静默降低目标值。

## 已知技术难点（实现时需覆盖）

- 性能瓶颈跨层耦合（模型量化、通信带宽、算子、框架、部署多维交织），根因定位链条长；
- 优化手段组合空间巨大，需以 profiling 数据驱动而非人工穷举试错。

## 精度环节（已实现）

S4.2 全量精度终验按 `.claude/skills/day0-inference/reference/precision-alignment-stages.md` 的 S4.2 参数卡执行：最终配置下探针五步对比 **golden 接力锚点**（跨阶段总对账，S1→S4 的漂移在终点现形）+ 题集全量；组合矩阵按 Designer 清单抽检。产物落 `acceptance/probe/`，出口锚点落 `acceptance/baseline/`。与 performance 的 benchmark 并行，精度侧证据供 G4/出口签收。

## 产物目录（规划）

`./.day0/<model>/acceptance/`：profiling 报告、调优前后对比、最终 benchmark 报告、精度验收报告、出口签收单。
