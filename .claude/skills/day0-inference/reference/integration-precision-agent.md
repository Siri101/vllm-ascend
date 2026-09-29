# 精度探针接入（npu-precision-agent @ $PAGENT）

> 本文是 Day0 流程对**精度探针**（`$PAGENT/model-precision-oob-probe/`，npu-precision-agent 仓内 skill）的接入契约声明：**何时调、怎么调、要什么证据、失败怎么路由**。探针的字段明细与判据定义**以 agent 仓为准**（`$PAGENT/model-precision-oob-probe/references/integration-bot.md`、`references/strength-ladder.md`），本文只引用不复制——两边抄写必然漂移。
>
> 归位一句话：**探针只供证不裁决**。它对着已拉起的服务产出机器可读的证据包（`precision_evidence_packet.json`），"这个门禁算过了没有"由主控按 `.claude/agents/accuracy.md` 裁决；探针不写 tracker、不翻阶段状态、不合并代码。探针（体检）与 npu-precision-agent 深路径（会诊）是两个东西——本文只接入前者；探针 FAIL 后的立案与深路径转诊见 §6。

## 1. 前置：解析与版本锚点

- **`$PAGENT` 与「探针版本锚点」读 tracker「环境信息」块**（占位符唯一来源）；子代理 prompt 一律传字面绝对路径（hook 注入次会话才生效，既有纪律不变）。
- **版本锚点复核（调探针前必做）**：`git -C $PAGENT rev-parse HEAD` 与 tracker 记录的探针版本锚点比对——不一致 → **停止并上报主控**，禁止自动 checkout（与 `$VLLM`「环境锚点以运行树为准」同款纪律：可能是 tracker 记错，也可能是探针仓被换过，两个方向处置相反，不得自动选边）。
- **`$PAGENT` 未配置**：探针不可用——G3 基线达标项走 accuracy.md 人工判据，tracker 备注显式声明「探针未接入」；**不是免检**，人工判据的证据要求（②③ 级）不变。
- **契约版本**：`api_version="1.0"`（以所用锚点版本的 agent 仓声明为准）。探针的证据包会回显 `api_version`，不一致 → 拒绝消费并上报，不做猜测式兼容。注意探针侧只有 `emit_packet.py` 兜底校验版本——首查是主控/执行者在调用前按本条完成。

## 2. 调用链（真实权重段五步）

执行者：Stage 1 由 tester 在真实权重段（tester.md Phase 2）代为执行，Stage 2/3 起由 accuracy agent 执行（同一链）。产物目录：`mkdir -p <输出根目录>/accuracy/probe`，四件产物（cases / probe / collected / packet）全部落此。

| 步 | 动作 | 要点 |
|---|---|---|
| 1 | **实例化委托请求** `accuracy/probe/request.json` | 字段样例：`$PAGENT/tests/fixtures/oob_request.example.json`。`service.base_url` = 本段 serve 地址（默认 `http://127.0.0.1:8000`）；`service.served_model_name` = tracker 的 served-model-name；`service.sampling` = `{"temperature":0,...}`（**greedy 硬约束**，非 0 探针拒绝出包）；`stage` / `code_state` / `env_evidence_ref` 取 tracker 现状 |
| 2 | **探服务** `probe_service.py --request <request.json> --out <输出根>/accuracy/probe/probe.json` | readiness = `/v1/models` 返回 200 **且** `data` 列表含目标模型——`Application startup complete` 不算 ready。退出码 0（内含 `readiness.ok=false` 也退 0，以 JSON 为准）/ 2 输入错误 |
| 3 | **采输出** `collect_outputs.py --cases <cases.json> --base-url <serve 地址> --sampling '{"temperature":0,"max_tokens":64}' --repeats 2 --out <输出根>/accuracy/probe/collected.json` | 脚本自动带 `return_token_ids:true`，token 取 choice 级 `token_ids`（兼容 message 级）。单 case 失败记 `error` 字段不炸整批；退出码 0 / 2。`repeats=2` 才有 L0 自洽证据；锚点可判时可 `--repeats 1` |
| 4 | **回填反假阴性** | `probe.json` 四项 hint 默认 `ok=false`（加载代码路径 / 配置生效 / pycache / 残留服务），执行者依本段 Phase 0 的实际证据**逐项置 ok 并填证据路径**（import 校验输出、serve-real.log、清理命令记录）——禁止不核对就出包，也禁止无证据置 true |
| 5 | **出证据包** `emit_packet.py --request <request.json> --collected <collected.json> --probe <probe.json> [--anchor <precision_baseline.json>] --out <输出根>/accuracy/probe/packet.json` | 退出码 0 / 2 输入错误 / **4 纪律或 schema 拒绝（产物不落盘）**。stdout 摘要 `{"strength","verdict","fingerprint"}` 供日志；完整证据以 packet.json 为准，随 G3 证据交主控裁决 |

**cases 文件来源**：复用 accuracy.md 基线梯子 ②③ 已落盘的 prompt 集/题集（`$PAGENT/tests/fixtures/oob_cases.example.json` 为格式模板）——探针**不另造题集**。

**锚点（可选，`--anchor`）**：② 路径本就要跑 transformers 参考实现——把其 greedy 参考输出转成 `$PAGENT/examples/anchor.example.json` 格式传入，L2 逐 token 对比即可判；无锚点 → 探针自动落在 L0/L1 并显式披露档位，不得声称做过 token 级对齐。

**离线演练**：无 NPU 环境时可以 `fake://` 前缀作 base_url 走通五步（探针读本地 fixture 当响应，如 `$PAGENT/tests/fixtures/fake_completions_ok.json`），用于接入自检与判据回归——演练产物**不是**真机证据，不得随 G3 归档。

## 3. 失败三分类与裁决语义

服务是 bot 侧拉起的，**探针失败 ≠ 精度失败**，按下表归因路由：

| 信号 | 归因 | 路由 |
|---|---|---|
| `readiness.ok=false` 或服务不可达 | **环境/服务问题** | 不产生精度 FAIL；G3 挂起，回服务拉起/环境自检（tester Phase 1 失败动作） |
| `collected` 单 case 带 `error`（malformed 响应等） | **服务问题** | G3 挂起，路由 vLLM 服务侧排查（版本 / 启动参数），不进精度裁决 |
| 服务健康 + 判据不过（packet `verdict=FAIL`） | **真精度 FAIL** | G3 不过：主控裁决——弱档签字降级（§4）或立案转深路径（§6）；packet 自动带 `escalation=full_pipeline` |
| case 全过但反假阴性 hint 未全 ok（`verdict=ANOMALY`） | **证据有缺口** | 不得当 PASS：补齐缺口重跑，或主控显式签字接受缺口（记录进 tracker 备注） |
| Phase 2 被主控裁决暂缓 | — | 探针不执行（探针属真实权重段组成）；G3 未过=未验证；恢复后从 Phase 0 重走 |

## 4. 档位披露与签字降级

- 证据包 `strength`（L0–L4）**显式声明本次证据强度**，判据与参照物的对应关系见 `$PAGENT/.../references/strength-ladder.md`；与 accuracy.md 基线梯子的映射：②「greedy token 序列完全一致」= 探针 **L2**（锚点 = ② 参考输出），③ 固定题集 = **L4 方向**（判分仍归主控/流程），无锚点重复跑自洽 = **L0**，仅可读性 = **L1**。
- **弱档不得冒充强档**：`token_checked_cases` < `case_count`（L1 落到文本级 oracle）、`symptom` 含 `token_ids_unavailable`、或 L0/L2 未实际跑足时，主控**不得**把该包当作干净的 token 级通过。
- **签字降级**：Day0 新模型常无强参照物——允许在弱档证据上放行 G3，但必须由主控**显式签字**（谁、依据哪档、为何可接受），记录留 tracker S1.4 备注；沉默降级 = 流程违规，与「仅凭 dummy 证据签收」同级。
- **sanity 与探针不可互替**：accuracy.md 准出 3 的 sanity（语义底线：预期关键词 / 无复读 / 无乱码）与探针判据（形态稳定 + 逐 token）**都做、都归档**——跑了探针不免除 sanity，反之亦然。

## 5. 与深路径的接续（FAIL 之后）

1. 探针 FAIL 的证据包**不含** `problem_card_ref`（立案是裁决之后的事，时序上探针不可能先拿到引用）；`escalation=full_pipeline` 是唯一转深信号。
2. **立案转换（`oob_handoff`）落地前为人工步骤**：主控依据 packet（symptom / fingerprint / first_divergence / 反假阴性核对结果）构造 `problem_card.json`（格式见 `$PAGENT/examples/problem_card.example.json`），前置确认可稳定复现；`oob_handoff` 自动转换落地后此步才自动化——在那之前不得宣称自动立案。
3. 立案后进入 npu-precision-agent 深路径（triage → 定位 → 修复 → 验证），其编排与门禁见该仓 `AGENTS.md` / `workflows/pipeline.md`——bot 主控此时只做委托与验收，不代行深路径内部编排。

## 6. 红线（两侧共同遵守）

- 探针**五不**：不写 tracker.md、不翻阶段状态、不合并代码、不拉起/停止服务、不自动 checkout 版本。
- 证据包**结构上无** `stage_passed` 类字段（emit 时硬拒绝）——裁决权不可能经证据包泄漏给探针。
- `api_version` 不匹配 → 拒绝消费；不做猜测式兼容。
- 本接入的验收口径见设计文档 §12（`docs/design/2026-09-29-oob-mode-a-integration-design.md`）：健康服务必须能拿到 `verdict=PASS` 的干净证据包——日常见到 ANOMALY/FAIL 即是值得停下核查的信号，而非常态。
