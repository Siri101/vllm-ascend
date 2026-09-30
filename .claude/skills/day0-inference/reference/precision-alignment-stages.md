# 阶段精度环节规范（Stage 2 / 3 / 4 的探针接入）

> 本文定义 Day0 四阶段中**每阶段精度环节**的标准动作与差异参数。环节标准动作（五步调用链、失败三分类、档位披露）与探针字段明细见 `.claude/skills/day0-inference/reference/integration-precision-agent.md`，本文不重复——本文只回答"每个阶段：何时探、跟谁比、判据是什么、产物放哪、出了接力锚点没有"。
>
> 范围：parallel / feature / performance 三条 flow 的**精度环节已实现**，其余段落仍为占位（占位处置见 SKILL.md）；本规范在 flow 完整实现后继续作为精度环节的权威定义。

## 1. 接力棒（precision_baseline.json）

上一阶段的出口证据就是下一阶段的对比锚点——四阶段各采一次"本配置的 greedy 逐 token 基线"，像接力棒一样传递。

- **生成**：探针五步采集（`collected.json`）+ 门禁**通过**后，由执行者调
  `python3 $PAGENT/model-precision-oob-probe/scripts/emit_baseline.py --collected <collected.json> --meta <meta.json> --out <baseline.json>`
  转成 `precision_baseline.json`（格式见 `$PAGENT/examples/precision_baseline.example.json`；`--meta` 必填 `stage` / `code_state{worktree,sha,verified}` / `config{model_cfg_hash, sampling{temperature}}`，可选 `feature_state` / `noise_floor` / `stale_on`）。
- **脚本纪律（违者拒绝落盘，退码 0/2/4 同 emit_packet）**：全 case 真实 token（`token_source=="token_ids"`）、repeats≥2 且逐 case 自洽通过、greedy、tokens 取 runs[0]。
- **归档位置**：`<stage 目录>/baseline/precision_baseline.json`（S1: `accuracy/baseline/`，S2: `parallel/baseline/`，S3: `feature/baseline/`，S4: `acceptance/baseline/`）。
- **失效规则**：锚点头部 `stale_on`（code_sha / config_hash / sampling / case_set 任一变化即失效）——失效锚点不得再作对比基准。
- **单一接力方向**：每阶段只认"上一阶段签收配置"的锚点，**禁止跳阶段取锚点**（S3 不直接对 golden；跨阶段总对账是 S4.2 的事）。
- **门禁状态归调用方**：脚本只管格式与纪律；"FAIL/ANOMALY 配置不得出锚点"由执行者与主控把关——未过门禁的采集结果只能作证据，不得 emit_baseline。

## 2. 阶段参数卡

### S1 出口（Golden，已接入）

| 项 | 值 |
|---|---|
| 触发 | G3 通过、S1.4 签收前（tester.md 第 5 步的探针调用之后） |
| 对比锚点 | ② transformers 参考输出（accuracy.md 基线梯子） |
| 判据 | L2 锚点逐 token 一致（无锚点则 L0/L1，档位如实披露） |
| 产物 | `accuracy/probe/`（证据包）+ **`accuracy/baseline/`（golden 接力锚点，S2 的入口证据）** |

### S2.4 量化精度对齐

| 项 | 值 |
|---|---|
| 触发 | 量化 + 并行配置部署运行后（本环节只要求服务可探，不等 flow 其余环节） |
| 对比锚点 | **S1 接力锚点**（`accuracy/baseline/`） |
| 服务配置 | 量化 + 目标并行策略（真实权重） |
| 判据 | 主判据：锚点逐 token 一致（L2）。**量化是近似**——Designer 可在设计产物中显式定义容差判据（题集判分口径，对齐 accuracy.md ③）并附理由归档；执行时启用任何容差须**主控签字**记录进 tracker 备注。容差以题集判分承载，不以"漂移 token 数量"承载（证据包只含 first_diff_pos 定位信息） |
| 产物 | `parallel/probe/` + `parallel/baseline/`（量化配置接力锚点） |

### S3.4 叠加精度回归（逐特性）

| 项 | 值 |
|---|---|
| 触发 | **每叠加一项特性、服务重启后**（逐项开启、逐项回归，禁止一次性全开） |
| 对比锚点 | **叠加前配置的接力锚点**（第一项特性用 S2 接力锚点） |
| 判据 | L2 锚点逐 token 一致（叠加的是执行路径而非数值路径，token 级无劣化是默认预期） |
| 循环 | 开特性 → 拉起 → 探针五步 vs 叠加前锚点 → PASS → `emit_baseline` 更新接力棒 → 下一特性；FAIL → 该特性打回（错误签名 + 证据，可入深路径），**接力棒不更新** |
| 粒度 | 逐特性一对比；量化 × 图 × 投机 × CP/PD 组合矩阵不在本环节（S4.2） |
| 产物 | `feature/probe/<feature>/`（每特性一组）+ `feature/baseline/`（最终配置接力锚点） |

### S4.2 全量精度终验

| 项 | 值 |
|---|---|
| 触发 | 最终配置（与 performance 的 benchmark 并行） |
| 对比锚点 | **golden 接力锚点**（跨阶段总对账——S1→S4 的漂移要在终点现形）+ 题集全量；组合矩阵按 Designer 清单抽检 |
| 判据 | L2 逐 token 一致为主；题集判分兜底（口径同 S2.4 容差纪律，签字要求不变） |
| 产物 | `acceptance/probe/` + `acceptance/baseline/`（出口配置锚点） |

## 3. 失败路由与门禁语义

- 与 P1 完全一致：失败三分类（环境/服务/精度）、ANOMALY=证据有缺口、探针只供证不裁决、弱档签字降级归主控——见 `integration-precision-agent.md` §3/§4。
- 阶段差异只在**归因粒度**：S3.4 的 FAIL 直接指向"刚叠加的那一项特性"（接力棒保证对比对象唯一）；S2.4 的 FAIL 指向量化/并行配置变更。
- FAIL 转深路径的立案转换已由 `oob_handoff` 自动化（`integration-precision-agent.md` §5）：主控裁决后运行脚本产出 `problem_card.json`（`source: bot_oob`）；`BLOCKED_FLAKY`（复现不稳）拒立案——按三向路由补跑复现后再试，不得带病进深路径。立案后**自动委托深路径**（P3，同节：staging 四件套 + 委托 prompt 模板）；深路径报修复后，主控重跑探针按**同一接力棒**复核，`verdict=PASS` 才记账——接力棒机制在此闭合。
