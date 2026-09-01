# oh-my-harness 项目当前进度

> 最后更新：2026-09-01（agent-team operator inbox 已改为 EventBus 发布成功后显式 ack，并新增重启回归测试；studio 100 项单元与 7 项 mock 集成测试通过，fmt/clippy 通过。此前完整 11 成员 coding-team 本地 LLM E2E 已通过：220.11s，judge 收齐 5/5 reviewer 报告并独立验证 PASS。）

---

## 项目背景

将 `@earendil-works/pi-coding-agent`（TypeScript）参考实现，用 Rust 重写为三层架构：

```
llm-api-adapter      ← provider 适配层（对应 pi 的 packages/ai）
llm-harness-core     ← agent 框架核心（对应 pi 的 packages/agent 核心部分）
llm-harness-runtime  ← 运行时基础设施（对应 pi 的 packages/coding-agent/src/core/）
coding-agent         ← coding agent 本体（对应 pi 的 packages/coding-agent 领域部分）
```

原始参考代码量约 10 万行（TypeScript），目前 Rust 实现约 1.4 万行。

---

## 各仓库状态

### llm-api-adapter ✅ 基本稳定
- Anthropic、OpenAI-compatible provider 实现完整
- 支持 streaming、tool use、thinking（extended thinking）
- DeepSeek 通过 OpenAI provider 复用实现
- `set_thinking_level` wire 格式修复已合入 main（commit `e44758b`，2026-07-17）：Anthropic 4.7+ 的 `effort` 从嵌套 `thinking.adaptive.effort`（被 API 拒绝）改为顶层 `output_config.effort`。runtime 已 bump rev `30ae9284 → e44758b8`
- **已知缺口**：Azure OpenAI 不支持；Responses API 不支持

### llm-harness-core ✅ 核心功能完整
三个 crate：
- `llm-harness-types`：Tool/AgentEvent/Session 等全部类型定义
- `llm-harness-loop`：agent loop 引擎，streaming 驱动，tool 调度
- `llm-harness`：AgentHarness、session 持久化、compaction、skills/PromptTemplate

已实现能力：ReAct loop、session 分支、compaction、skills、retry、steer/follow-up 队列

### llm-harness-runtime ✅ v0.2.1 已实现（2026-06-14）
设计文档位于 `llm-harness-runtime/docs/runtime-v0.2-design.md`。

8 个 crate，54 个测试全部通过：

| crate | 内容 |
|-------|------|
| `llm-harness-runtime` | 主 crate：全部 trait 定义 + TaskRunner 状态机（完整 AgentHarness 驱动）+ BudgetControlAdapter + HumanApprovalWrapper + TracingHookAdapter + CompositeHook 系列 |
| `llm-harness-runtime-audit-jsonl` | Hash 链式 JSONL 审计日志 |
| `llm-harness-runtime-auth` | EnvAuthHook + FileAuthHook |
| `llm-harness-runtime-mcp` | MCP 客户端适配器 |
| `llm-harness-runtime-sandbox-os` | OsEnv（ExecutionEnv impl）+ OS 沙箱 |
| `llm-harness-runtime-sandbox-bwrap` | Linux bwrap 沙箱 |
| `llm-harness-runtime-sandbox-seatbelt` | macOS seatbelt 沙箱 |
| `llm-harness-runtime-trace-otel` | OpenTelemetry TraceExporter |

**已完成（Phase 0-5 + v0.2.1）：**
- Sandbox / ToolRegistry / MCP / ResourceProvider / PromptTemplate / TraceExporter / AuditSink / HumanApprover / BudgetControlAdapter / SubAgentSpawner / TaskRunner 全部 trait + 核心实现
- 6/9 hook 组合矩阵集成测试覆盖
- `TaskRunnerImpl::start()` 完整实现：真实 AgentHarness 驱动（MockLlmClient E2E）
- 3 个 E2E 测试（smoke / multi-turn / tracing）
- `llm-harness-core` 的 `HarnessHooks` 新增 `Clone` derive

**待做：**
- coding-agent 临时实现迁移到 runtime（tools/settings/provider 选择逻辑）

### llm-tutor ✅ 可运行，含 mock 集成测试
三个 crate：`tutor-tools`（工具）、`tutor-agent`（agent 核心）、`tutor-web`（待实现）

已实现能力：
- Chat 模式：RAG 知识库检索 + 网络搜索 → 对话回答
- DeepSolve 模式：Pre-retrieve → Plan → Solve（含 ReplanHook）→ Synthesize 四阶段流水线
- 多 provider 支持（Anthropic / DeepSeek / OpenAI-compatible，通过 `LlmConfig`）
- 审计日志（`JsonlAuditSink`，hash 链式 JSONL）
- 5 个 mock 集成测试，无需真实 API key 即可运行

**注**：原 `oh-my-harness/tutor-agent` 独立仓库已迁入本仓库并 archive。

### eda-agent ✅ v0.5.6 — 技术债清理完成（TD-1~TD-7），38 stage pipeline 对齐 ArcGen AMC lite

**v0.5.9 (2026-07-20) ArcGen `5210576` 对齐审查**：
- ArcGen `5210576` 修复 gauge_check space gauge 误判 + fail gauge 文件 + 用户交互删除 + mask BO JAX 挂死。
- **eda-agent（Rust）**：space gauge 修复自动生效（wrapper import ArcGen `make_gauge_check()`，layout_parser.py 运行时直接用）。mask BO JAX 修复 N/A（vizier BO 未实现，走 pangen 路径）。**未对齐**：`user_prompt_fn` 未传入（fail gauge 默认保留，无交互删除）+ cal/val 双文件检查待确认 → 开 issue #49 跟踪。
- **eda-agent-py（Python）**：有独立 `_layout_parser.py`，`check_single_gauge` 缺 space 分支 → space gauge 误判 FAIL。缺 fail gauge 文件/用户交互/cleaned 文件/gds_layers 配置层优先选择 → 开 eda-agent-py #23 跟踪。

**v0.5.8 (2026-07-19) ArcGen `c15dc55` 对齐审查**：
- 修复 F/G 策略（`fg_strategy1_adjust_sigma`/`fg_strategy2_reduce_beta`）从写 `wizard.json` 改为写 `lite/term_pool.json`（commit `051bb99`）——旧实现写 wizard.json 在 lite 模式下被 PanGen 忽略，F/G 自动修复完全无效。
- cargo fmt ✅ / clippy 0 warning ✅ / test 83/83 ✅
- #43 已修复（commit `f6e68c1`）：`term_selection_lite` 现注入 mask/optical 搜索参数到 `fit_resist_model_ntd_w7.py`（`generate_fit_script_from_mask_params` helper，读 `mask_params.json`，兼容 string/number JSON 值）。`generate_fit_script` 增加 `in_corner` 参数。3 个单元测试。cargo test 86/86 ✅
- #44 layer 3 已实现（commit `3e3b7ba`）：Round 0 经验直接复用——相似度 ≥ 0.6 且非 FAIL 时，`find_experience_for_round0` 返回 (group, best_terms)，`map_experience_terms` 映射到 term_pool.json，跳过 LLM 直接 run baseline PanGen。layers 1-2（override + prev_selected_terms）待实现（与 #46 关联）。2 个单元测试。cargo test 88/88 ✅
- 剩余 3 个对齐差距 issue：#44（Round 0 缺 4 层优先级）、#45（缺 autoweight 集成）、#46（model check 回退体系未对齐）。详见 `eda-agent/HANDOFF.md`。

**v0.5.7 (2026-07-13) 技术债清理 TD-1~TD-7 全部完成**：
- TD-1: executor.rs 1385 行 → 4 文件模块（mod/tool_handler/checker_handler/agent_handler）
- TD-2: 删 legacy ReAct + legacy orchestrator 路径 ~2000 行死代码
- TD-3: StageContext 加 typed accessors + 55 个变量名常量（编译期拼写检查）
- TD-4: 20+ 处危险 .ok() 改为 ? 传播或 warning 日志（含 Tension_1 crash 根因修复）
- TD-5: 测试漂移随 TD-2 解决
- TD-6: bin/eda-agent.rs 679 → 240 行，拆出 src/cli/ 模块（cli/status/provider/events）
- TD-7: ISSUES.md 精简为 GitHub Issues 速查表，历史 bug 全部迁移到 [GitHub Issues](https://github.com/oh-my-harness/eda-agent/issues)
- cargo build ✅ (2 pre-existing warnings), cargo test ✅ 30/30

**v0.5.5 (2026-06-29)**：A026 修复 — beam 分支从 action 检查内部移到外部。
E2E 全流程验证通过：11 轮 K=3 并发 beam search，每轮 3 候选并发 PanGen → pick_winner →
rollback → finalize_round，best_cal=0.040，pipeline success。
**v0.5.4 (2026-06-28)**：R-08 beam search 集成。term_selection_lite 支持 K=3 并发候选（run_beam_round），LLM 返回 {candidates:[...]} 时触发，否则 fallback 单候选。提取 finalize_round 共享状态逻辑。E2E 回归：22 节点 + 12 轮 term_selection 通过，无回归。发现 A024（ArcGen 无 beam_runner/lite 代码，beam 为 eda-agent 自有增强）+ A025（PanGen grid_check worker stall，非 pipeline bug）。

**E2E 验证**：2026-06-28 全流程跑通 `pipeline success`，36 stages 全部通过，model_check retry 循环正确终止。
针对 EDA 仿真软件内部 AMC 光刻模型校准流水线的专用 Agent。

**v0.4 E2E v18 验证结果（2026-06-26，fresh job_dir 从零开始）**：
- 完整 8 节点流程全部跑通：findoptics → optical → gridparam(跳过) → mask → term_decision → calibration_iter(20轮) → resist_tune → model_check
- 回流机制验证：model_check FAIL → execute_restart → term_decision → calibration_iter（对齐 ArcGen）
- BUG-A008~A012 全部修复（calibration_iter mixed agent+tool、lite/ 复制、val.txt 自动复制、gridparam 条件跳过、.src 文件复制）
- val_uwrms 不再为 0（BUG-A010 修复生效），resist_tune cal=0.025 val=0.399

**v0.4 架构**：
- Rust 声明式 YAML orchestrator 替换 LLM ReAct 流程控制，LLM 仅在 term_decision + calibration_iter 介入
- 节点逻辑抽离为独立 `async fn`（src/tools/nodes/）
- orchestrator 核心（config/context/executor/runner）
- model_check A~G 七项检查 + feedback 多目标回流（对齐 ArcGen）
- pipeline.yaml 11 stages 声明式流程
- `cargo build` 零错误，`cargo test` 11/11 通过

**与 ArcGen AMC lite 流程对齐状态**：
- ✅ term_selection_lite 统一节点（Phase 1，合并旧 term_decision + calibration_iter + resist_tune）
- ✅ 22 stage pipeline（route_findoptics + validate_pre×5 + resist_quality + resist_quality_llm 新增，对齐 ArcGen 5 类差异）
- ✅ Phase A: resist_quality + resist_quality_llm（退化检查 >0.5nm FAIL / >0.1nm WARNING）
- ✅ Phase C: route_findoptics 条件跳过（optics_search_engine == vizier）
- ✅ Phase D: validate_pre 节点×5（L1 文件 + L2 结构/物理量范围检查）
- ⏸️ Phase B: gauge_error_attribution LLM 归因（当前 skeleton）
- ⏸️ Phase E: 数据处理前置 7 节点（暂缓，设计差异）
- ✅ execute_restart 多目标回流（对齐 ArcGen route_after_restart）
- ✅ model_check A~G 检查 + feedback + gauge_error_attribution 骨架
- ✅ calibration_report 终节点（生成 calibration_report.md）
- ✅ LLM 调用重试 [10,30,60]s（对齐 ArcGen _LLM_RETRY_DELAYS）
- ✅ LLM quality gate 2 层（optical/mask_quality_llm，score≤60 FAIL→execute_restart）+ result_analysis 节点（optical/mask_result_analysis，informational，永不阻塞）

**待做（P1，后续）**：数据处理阶段节点（data_clean 等）、LLM 调用重试（指数退避）、删除 SKILL.md/agent.rs

### coding-agent ✅ 可运行，含临时技术债
- 完整 CLI（one-shot / interactive REPL / session 管理）
- 7 个工具：read / bash / edit / write / grep / find / ls
- 支持 Anthropic（`ANTHROPIC_API_KEY`）和 OpenAI-compatible provider（`LATTICE_API_KEY` + `LATTICE_API_BASE` + `LATTICE_MODEL`）
- **已用 MiniMax-M2.5（SiliconFlow）验证工具调用闭环**

**技术债（runtime 实现后需迁移）：**
- `src/tools/` → runtime-tools crate
- `src/settings.rs` → runtime SettingsManager
- bin 中的 provider 选择逻辑 → runtime ModelRegistry

---

## 已验证的 Demo

```bash
cd oh-my-harness/coding-agent
LATTICE_API_KEY="..." \
LATTICE_API_BASE="https://api.siliconflow.cn/v1" \
LATTICE_MODEL="Pro/MiniMaxAI/MiniMax-M2.5" \
cargo run --bin coding-agent -- -p "列出当前目录的文件，告诉我这是什么项目"
```

结果：agent 正确调用 ls/read 工具，返回完整项目分析。agent 层架构可行性验证通过。

---

## 下一步优先级

1. **将 coding-agent 的临时实现迁移到 runtime**（tools/settings/provider 选择逻辑）
2. **eda-agent 真实 EDA case 端到端验证**（需要完整 wizard.json + 校准数据）

---

## 参考实现

pi-coding-agent（TypeScript 原始参考）位于：`/data/leiqiaojie2/pi/`

关键目录：
- `packages/agent/src/` — 对应 llm-harness-core
- `packages/ai/src/` — 对应 llm-api-adapter
- `packages/coding-agent/src/core/` — 对应 llm-harness-runtime（设计参考）
- `packages/coding-agent/src/modes/` — TUI / interactive mode（暂未实现）

EDA agent 设计参考文档：`/data/leiqiaojie2/eda-agent-design.md`

---

## 本地开发说明

所有仓库在 `/data/leiqiaojie2/oh-my-harness/` 下，使用本地路径 patch 开发：

`coding-agent/.cargo/config.toml` 已启用本地路径覆盖：
```toml
[patch.'https://github.com/oh-my-harness/llm-harness-core']
llm-harness-types = { path = "../llm-harness-core/crates/llm-harness-types" }
llm-harness-loop  = { path = "../llm-harness-core/crates/llm-harness-loop" }
llm-harness       = { path = "../llm-harness-core/crates/llm-harness" }

[patch.'https://github.com/oh-my-harness/llm-api-adapter']
llm_adapter = { path = "../llm-api-adapter" }
```

`llm-harness-core` 的 `llm_adapter` 依赖使用本地路径（`path = "../llm-api-adapter"`）。

---

## 2026-07-01 更新：runtime WorkflowContext + EDA 适配层

### llm-harness-runtime
- **WorkflowContext**（commit `9d5d442`，已合并 main）：共享可变 KV 黑板，等价 LangGraph State。executor 读写、judge 读、LLM step prompt 注入。随 WorkflowState 持久化，崩溃恢复保留。204 测试全绿。
- **编排分工设计文档**（commit `7a4e812`）：`docs/design/2026-07-02-agent-runtime-orchestration-design.md`。明确 Agent（定义+翻译+领域逻辑）↔ Runtime（驱动+持久化+恢复）的分工边界、adapter 层职责、生命周期、与 EDA 旧 orchestrator 对比。
- **workflow issue 收尾**：12 个 issue 提到 GitHub（#22~#34），同事修复 8 个（#22~#24/#26~#27/#31~#33），我实现 #34（WorkflowContext）。4 个低优先级 OPEN。
- **main 分支**：已 fast-forward 合并 workflow 分支（含 WorkflowContext + PyO3 SDK + final answer contract）。

### eda-agent
- **workflow_adapter 模块**（commit `4fb005f` / `42cae1f` / `8b7d51e`）：
  - `converter`：pipeline.yaml → Workflow（stage → Step::Executor，routes → Edge）
  - `executor`（EdaExecutor）：桥接 runtime StepExecutor → execute_stage，通过 WorkflowContext 传递 StageContext.variables
  - `judge`（EdaJudge）：读 route_key → StageConfig::resolve_next → Transition
  - CLI `--runtime` flag：用 WorkflowEngine 跑同一 pipeline.yaml，复用 TaskStore 持久化/事件流
  - 2 个集成测试通过（mock executor 验证 context 共享 + route_key 路由）
- **已知缺口**：EdaExecutor 已接入真实 execute_stage，但未在真实 job_dir 上 E2E 验证（需 PanGen 环境）。loop_counter 跨 step 持久化未实现（当前 loop 由 execute_stage 内部管理）。
- **迁移规划**（commit `8c18e28`）：`docs/2026-07-02-orchestrator-to-runtime-migration-plan.md`。分 4 阶段将 orchestrator 编排基础设施下沉到 runtime：Phase 1 验证 adapter 路径 → Phase 2 默认切换 → Phase 3 删重复编排代码 → Phase 4 marker 退役。
- **P1.1 loop_counter 持久化**（commit `8d25344`）：EdaExecutor 通过 WorkflowContext（`__loop_counter_` 前缀）跨 step 持久化 loop counter，对齐旧 runner.rs 的 bump_loop/reset_loop 语义。EdaJudge 从 structured 读 counter 传给 resolve_next（不再 hardcode 0）。新增 1 个集成测试（3 次迭代递增 + loop_exhausted 路由）。
- **P1.2 + P1.3 E2E 验证完成**（commit `d624668`）：`--runtime` 路径在真实 job_dir（`/data/pangen_result/eda_test_fresh_v26`）上跑通 38 stage 全流程：init_job → ... → term_selection_lite（11 轮 loop + beam search）→ resist_tune → model_check → model_check_feedback（retry 10/10 终止）→ calibration_report → success。loop_counter 跨 step 持久化、done marker + TaskStore 并行、EdaJudge 路由全部验证通过。修复 pipeline.yaml 的 `next_on_gridparam_search` target 错误（`gridparam_search` → `run_gridparam_search`，旧 orchestrator 不校验 target 所以未暴露）。
- **P2.2 默认切换完成**（commit `51d4bbf`）：runtime WorkflowEngine 成为默认执行路径。移除 `--runtime` flag（默认即 runtime），新增 `--legacy-orchestrator` 回退到旧 v0.4 orchestrator。USAGE.md 同步更新。
- **崩溃恢复 E2E 验证**（commit `223211e`）：CLI 启动时扫描 TaskStore，发现未完成 task 则 `restore()` 精确续跑。E2E 验证：跑到 `mask_quality_llm` 被 kill → 重跑从 `mask_quality_llm` 续跑（跳过前 22 个已完成 step）→ `calibration_report` → pipeline finished。这是 runtime 相比旧 orchestrator（marker 文件幂等重放）的核心优势：结构化状态精确续跑。
- **runtime issue 收尾**（commit `a26f89e`/`a1f3dcb`）：关闭 #28（retry 乘法放大文档说明）+ #30（混用 Expr+Label 边 warning）。剩余 2 个 OPEN：#25（HTTP SSRF，已知限制）、#29（ConditionExpr AND/OR/NOT，功能补充，EDA 不用 Expr 暂不优先）。runtime 235 测试全绿。
- **事件流改进**（commit `c26e15b`）：`--runtime` 路径 StepFinished 事件现在打印 route_key + loop_counter，用户能看到每步路由决策（如 `→ pass`、`→ continue (loop=1)`）。
- **Simple case E2E + BUG-RT01 修复**（commit `4bdabd9`）：用 small_case1 fresh job_dir 测试发现 BUG-RT01（`--runtime` 新建路径未注入 case_dir/case_name 到 WorkflowContext），修复后数据准备全通过。eda_test_fresh_v26 完整 38 stage 全流程跑通 → calibration_report.md 生成。PanGen SIGSEGV（small_case1 的 .src 文件）是环境问题非代码 bug。详见 `eda-agent/BUGS_RUNTIME_E2E.md`。
- **BUG-RT02 修复**（commit `63a30e0`）：`prepare_job` 把 `.src`/`.oas`/`pool` 复制到 `job_dir/file/`（而非根目录）+ 重写 context 路径。修复后 PanGen 不再 SIGSEGV，能启动 TCC 计算。旧 orchestrator 也受此 bug 影响（从未在 fresh job_dir 跑通过 small_case1）。PanGen exit 1 是仿真器/数据层面问题。
- **✅ 真实全流程 E2E 完整跑通**（commit `3e2a617`）：`eda_test_fresh_v26` 删除 done marker 后真实重跑，38 stage + 10 轮 model_check 回流 = 155 turns，最终 `calibration_report.md` 生成（cal_uwrms=0.0170）。编排机制全部验证通过：loop_counter 跨 step 持久化、回跳循环 ×10 轮、loop_counter 跨轮回零、max_steps 安全阀、done marker + TaskStore 并行。修复 4 个 BUG（RT01 case_dir 注入、RT02 file/ 目录复制、RT03 .oas 空文件、max_steps 100→500）。
- **✅ GLM5.2 E2E 完整跑通**（2026-07-03，commit `19b1923`）：用 GLM5.2 替换 Claude（解决 LLM 网关余额耗尽问题），eda_test_fresh_v26 完整 38 stage + 10 轮 model_check 回流 = 76 turns，`calibration_report.md` 生成。编排机制全部验证通过：GLM5.2 作为 LLM 后端正常工作、done marker 跳过、WorkflowContext 变量传递、回跳循环 ×10 轮、PanGen 无崩溃。详见 `eda-agent/BUGS_RUNTIME_E2E.md`。
- **✅ resume-failed 实现并 E2E 验证**（2026-07-04，runtime commit `9c188c4` + eda-agent commit `0ec9aa8`）：runtime 的 `resume()` 现在接受 `Failed` 状态，重置为 `Paused` 后 `run()` 从 `current_step` 精确续跑。EDA Agent 的 `scan_for_resumable_task` 扩展为扫描 failed task，restore 后自动调 `resume()`。E2E 验证：12 步进度完整保留，从失败点 `run_findoptics` 续跑，不从头重放。对齐 LangGraph checkpoint 恢复语义。closes #36。


### eda-agent-py ✅ ArcGen 对齐 — E2E 全流程跑通

**2026-08-11 ArcGen `087eba2` pframe.py 统一架构对齐**：

ArcGen 最新版移除了所有独立 pframe/fit_model 脚本模板（`pframe_lite.py`、`pframe_resist_tune.py`、
`pframe_bo_resist_search.py`、`fit_model_bo_resist.py`、`fit_resist_model_ntd_w7.py` 等），
改为统一的 `pframe.py` + `fit_model.py` + `scene_config.json` 架构。eda-agent-py 全面对齐：

- **config/cli**：默认 `arcgen_dir` 从 `/data/leiqiaojie2/ArcGen`（旧）更新为 `/data/leiqiaojie/ArcGen`（最新）
- **amc_dir() 修复**：从 `__file__` 定位（指向不存在的 `eda_agent_py/amc_template/`）改为 `arcgen_dir` 定位
- **resist_tune_ga**：`pframe_resist_tune.py` → 统一 `pframe.py` + 写 `scene_config.json`（scene=resist_tune, iters=100）
- **resist_tune_bo**：`_generate_fit_model_bo_resist` → `_generate_fit_model_bo`，使用统一 `fit_model.py` 模板 + ArcGen 正则注入；
  BO validation 写 `scene_config.json`（scene=bo_resist_val, iters=1）
- **term_selection_lite**：`pframe_lite.py` → 统一 `pframe.py` + 写 `scene_config.json`（scene=lite, iters=40）
- **beam_runner**：`_REQUIRED_PATTERNS` 更新（`fit_model.py` + `_autoweight.py`/`_bo_eval.py`/`_lite_flow.py`）；
  每个 candidate 写 `lite_env.json`（SSH 环境传递）+ `scene_config.json`（scene=lite）
- **inject_mask_params_to_script**：从读旧模板改为读 `target_path`（由 prepare_job 复制），
  新增 `.src`/`.oas` 文件名修正（从 `calibration_context.json` 读取）
- **prepare_job**：模板文件列表更新为统一 `pframe.py` + `fit_model.py` + 依赖模块
- 测试：81 passed, 17 skipped, 4 pre-existing failures（test_mask_search_engine，与本次变更无关）

**2026-08-11 ArcGen `087eba2` 深度对齐审查（lite pipeline 全量比对）**：

系统对比 ArcGen `087eba2` 与 eda-agent-py 的 lite/ 目录全部文件 + 关键节点文件，发现并修复 10 处不对齐：

- **term_pool.json 全量同步**（commit `0b3c1f8`）：eda-agent-py 缺少 3 个 group（`ntd_1x_1`/`ntd_2x_1`/`ptd_1`），且全部 9 个已有 group 的参数范围与 ArcGen 不一致（如 ptd sigma 30-100 vs ArcGen 40-300）。直接从 ArcGen 同步完整 term_pool.json
- **`_GROUP_DESC` 补全**：`term_advisor_prompts.py` 缺少 `ntd_1x_1` 描述，影响 `select_initial_group` LLM prompt
- **F-51 PanGen 异常恢复**（async `_run_pangen`）：eda-agent-py 的 `nodes/_common.py:_run_pangen` 缺少 gateway 断连指数退避（1→2→4→8→16→30s，max 6 次）+ worker crash 单次重提交（10s delay）+ `crash_recovery.jsonl` 事件记录。ArcGen 的 `submit_with_recovery` 已实现这些。已移植完整 F-51 逻辑
- **`server_ip` SSH 远程提交**：`_run_pangen` 缺少 `server_ip` 参数，beam_runner 未传递 `server_ip=server_ip`。导致多服务器 `WORKER_NODES` 配置下所有 candidate 仍本地提交。已添加 SSH 远程提交支持 + 本机 IP 检测
- **`experience_cache.py` schema v5→v6**（commit `7a61ded`）：缺少 `_extract_winner_path`（从 `iteration_log.jsonl` 提取每轮 winner 优化路径）+ `fit_script_hash` 参数/字段 + `_build_case_info` 辅助函数。`term_selection.py` 未传递 `fit_script_hash=ctx.get(...)` 到 `save_experience`
- **`_effective_group` 逻辑缺失**：beam_runner 中 `group_eval` 候选必须用 `candidate.target_term` 作为 group_name 传给 `_prepare_candidate_dir`，否则所有 group_eval 候选用 R0 的 group_name 导致 term_pool 覆写错位
- **`MAX_ROUNDS` 默认值**：`lite_config.py` 默认 11（R0+R1~R10），ArcGen 默认 5（R0+R1~R4，多服务器并行后减少轮次）
- **retry 模式 `upgrade_group` 过滤缺失**（commit `65a84b4`）：retry 模式下 ArcGen 过滤掉 LLM 生成的 `upgrade_group` 候选（首次已充分尝试），eda-agent-py 未过滤导致 retry 浪费 beam slot

已验证对齐（无需修改）：`fit_model_codegen.py`（完全一致）、`effectiveness_check.py`/`term_advisor_ops.py`/`decision_cache.py`（仅注释/风格差异）、`pick_winner`/`_shallow_pool_for_pangen`/`_kill_stale_pangen`/`_append_beam_summary`/`_build_resist_model`/`_finalize`（逻辑一致）、`force_upgrade_group`/`_r0_group_eval_done`/`precheck_rejected`（逻辑一致）、`_REQUIRED_PATTERNS`/`_OPTIONAL_PATTERNS`（完全一致）、config 值（PANGEN_SUBMIT_STAGGER/PANGEN_TRIAL_TIMEOUT/SSH_CONNECT_TIMEOUT 等全部匹配）。56 测试通过，17 跳过。

**2026-08-11 small_case1 双跑验证（确定性节点产物比对）**：

使用 `small_case1` 在 `default` 模式（无 LLM）下对 ArcGen（`087eba2`）与 eda-agent-py 进行双跑验证，
比对确定性节点（init_job → build_calibration_context → data_clean → gauge_check → gauge_group → prepare_job）
的全部产物。发现并修复 3 处环境/依赖缺口：

- **`klayout` 未安装**（gauge_check 降级为 SKIPPED）：eda-agent-py venv 缺少 `klayout` 包，
  导致 `gauge_check` 的 3 项几何检查全部跳过（`overall_status=SKIPPED`，`per_gauge_results=[]`），
  而 ArcGen 正常执行（`overall_status=PASS`，12 gauge 逐条检查）。安装 `klayout==0.30.9` 后对齐
- **`pydantic` 未安装**（calibration_context 验证静默跳过）：eda-agent-py venv 缺少 `pydantic`，
  `_validate_ctx` 中 `_HAS_PYDANTIC=False` 导致 schema 验证被完全跳过，`film_stack.thickness` 保留
  int 类型（`30`）而非 ArcGen 的 float（`30.0`），进而 `fit_model.py` 生成的 `add_film_layer()`
  参数格式不一致。安装 `pydantic==2.13.4` 后对齐
- **BO 子进程依赖缺失**：`langgraph`、`langchain-core`、`aiohttp`、`httpx`、`click` 未安装，
  导致 mask_search BO 子进程无法 import `langgraph_pipeline` 模块。已全部安装对齐 ArcGen venv 版本
- **`pyproject.toml` 依赖更新**：将 `pydantic`、`pandas`、`klayout` 加入核心 dependencies；
  新增 `[project.optional-dependencies] bo` 组（`langgraph`、`langchain-core`、`aiohttp`、`httpx`、`click`）
- **gauge_check LLM 报告移除**（commit `7cff045`）：ArcGen 的 gauge_check 节点接受 `reporter` 参数
  但函数体内从未调用（F-48 注释提到但未实现）。eda-agent-py 却在 gauge_check 里调了
  `single_llm_call_json()` 生成 `llm_report` 字段，并额外加了 `check_4_misc` 字段。
  default 模式下这导致 23s 延迟（LLM 调用）vs ArcGen 的 3s，且产物多了两个字段。
  已移除 LLM 调用 + `llm_report` + `check_4_misc`，并修复 early SKIPPED 路径的 `counts` 格式

双跑产物比对结果（全部 ✓ IDENTICAL，含 gauge_check_report.json 深度比对）：
  `pframe.py`、`_lite_flow.py`、`_autoweight.py`、`_bo_eval.py`、`fit_model.py`、
  `cal.txt`、`calibration_gauges.txt`、`val.txt`、`validation_gauges.txt`、
  `lite/group_name.txt`、`lite/term_pool.json`、
  `data_process/cal_cleaned.txt`、`data_process/cal_cleaned_grouped.txt`、
  `data_process/calibration_gauges_annotated.txt`、
  `gauge_check_report.json`（keys 一致 + 内容 byte-identical，12 gauges, 0 mismatch）

**⚠️ 已知阻塞：PanGen license server（lmgrd）未运行**：
mask_search BO 的 `compute_tcc` session 需要 `model_ntd` license feature，但 FlexLM license server
未启动（`error code -15: Cannot connect to license server system`）。所有 BO trial 以 rc=-11
（SIGSEGV）失败。无法验证 mask_search 及后续节点的产物对齐。需运维启动 license server 后重跑。

**`fit_model_codegen.py` 已移植**（commit `10fdb09`）：从 ArcGen 移植 722 行 codegen 模块，
  `prepare_job` 调用 `generate_fit_model_script()` 生成 case 专用 `fit_model.py`（NA、film_stack、
  substrate、TNP/tone、3DM sectors、GDS layers 等全部从 `calibration_context.json` 内联）。
  `beam_runner` 每个 candidate 重新 codegen（不同 group/terms）。默认 NTD group 修正为 `ntd_1x_1`。

**2026-08-11 pipeline_result.json 聚合产物（Issue #77）**：

ArcGen 与 eda-agent-py 同步实现：Pipeline 结束时从内存 state 聚合各阶段结构化结果到
只读的 `pipeline_result.json`，写入 job_dir。8 个阶段的关键指标在一个 JSON 中可读：
`calibration_context` / `gauge_check` / `mask_params` / `selected_terms` /
`resist_model` / `model_check` / `feedback` / `gauge_error_attribution`。
设计要点：读端聚合（不读磁盘文件）、只写一次（永不更新）、不替代散落文件
（PanGen 接口契约和节点间通信通道保持不变）。已有先例：`best_snapshot/best_summary.json`。
ArcGen MR !337，eda-agent-py commit `9cf1b00`。

**2026-07-25 simple_case1 V2 验证通过（issue #52 修复确认）**：

V1 在 term_selection_lite Round 0 因 `term_advisor_lite.py` 缺少 `select_initial_group` re-export → PanGen `ImportError` → SIGSEGV（rc=-11）。V2 修复后（commit `1ef3501`）重跑：

- **33 步全部成功**：init_job → data_clean → gauge_check → gauge_group → prepare_wizard → findoptics（R²=0.997）→ optical_search（focus=39.8, metro_p=41.0）→ mask_search（bias=1.0, out_corner=5.0）→ **term_selection_lite 10 轮无崩溃** → resist_tune → model_check → calibration_report
- **关键修复确认**：term_selection_lite Round 0–10 全部 rc=0，`select_initial_group` import 正常
- **模型质量**：cal_uwrms=0.025, val_uwrms=0.275（val 高因 small_case1 仅 4 val gauges，属正常）
- **产物**：optical_result.json + mask_params.json + selected_terms.json（13 terms）+ resist_model.json + calibration_report.md
- **Job dir**：`/data/pangen_result/simple_verify_edapy_v2_20260725141624`
- **耗时**：~65 分钟（14:16–15:21），findoptics 22min + optical_search 22min + mask_search 8min + term_selection 10min + resist_tune 3min
- **结论**：链路完全通顺，可进入 0509 全量 case 双跑


**2026-07-24 对齐 ArcGen 284c53c bug 修复（issue #52）**：

ArcGen `a522e38..284c53c` 包含关键 bug 修复（TNP 推导值对调导致校准质量差）+ 质量增强。eda-agent-py 同步 9 处变更：

- **TNP 对调修复**（`fit_model_bo_mask.py` / `fit_resist_model_ntd_w7.py`）：Dark/Clear Field trans 值互换。eda-agent-py 直接引用 ArcGen `amc_template/` 目录，拉取后自动生效
- **Check E**：`cal_uwrms≥1.5` 从 PASS 改为 WARNING（模型尚未充分拟合，过拟合检查暂缓）
- **term 名归一化**：LLM 输出 term 名（`Ax_3`）时自动去掉 `_N` 后缀匹配 operation 名（`Ax`），影响 `effectiveness_check.py` + `term_advisor_ops.py`
- **beam_runner**：`CandidateResult` 新增 `cal_rms` 字段（从 `finalrms` 读取），p1_ratio 日志增加 cal_uwrms/cal_rms/val_uwrms
- **validation_source=none**：schema 新增 `"none"` 值（不做 validation），`_detect_validation_source` 支持用户显式指定，`gauge_group` 处理三种模式
- **gauge_check auto 模式**：auto 模式直接删除 FAIL gauge，不再交互询问
- **校准报告增强**：fallback 报告新增 `_format_check_detail()`，4 列检查表 + validation_uwrms/threshold/model_path/term 列表
- **取消 PASS 提前终止**：Round 0 和后续轮次 PASS 后不再 break，跑满全部轮次
- **retry 独立目录隔离**：retry 模式使用 `retry_N/` 子目录，symlink 大文件 + 复制小文件 + 保留缩窄的 `term_pool.json`

**2026-07-24 0509 全量 case 双跑完成（amc_template_0509，1414 cal + 306 val gauges）**：

- **Senza v1.0.0 / PanGen 2026.04.00 / Python 3.14**，完全无镜像/mock/容器，直接基于 Senza .so 原生扩展
- **前 7 阶段逐字一致**：data_clean（1449→1414）→ gauge_check → calibration_context → prepare_wizard → findoptics（R²=0.999286）→ optical_search（focus=40.8, metro_p=46.0）→ mask_search（focus=43.8, metro_p=49.0, bias=-1.0）
- **term_selection 分歧（非代码 bug）**：ArcGen 经验缓存命中 ntd_1x_2（相似度 80%，3 条 lessons 注入 prompt）→ 3 轮早停 PASS（cal_uwrms=7.04）；edapy 无经验命中 → 回退 ntd_ultra → 10 轮跑满 WARNING（cal_uwrms=4.72，反而更优）
- **根因**：两个仓库 experience.jsonl 独立积累，ArcGen 有历史 ntd_1x_2 记录（7/8、7/10），edapy 从未跑过该工况。round0 baseline 一致（12.163 vs 12.168），排除仿真层差异
- **耗时**：ArcGen ~13h22m（含 2h21m restart 间隙），edapy ~13h54m。term_selection 差异最大（2h49m vs 7h44m）
- **产物路径**：ArcGen `/data/pangen_result/optical_search_20260723133912`，edapy `/data/pangen_result/dualrun_0509_edapy_20260723133658`
- **结论**：代码逻辑完全对齐，分歧纯粹是经验缓存数据驱动。如需完全可复现，同步 experience.jsonl 即可

**2026-07-24 确认无镜像 Python**：
- `eda_agent_py/workflow_engine/`（原 mock WorkflowEngine）已废弃删除，目录为空
- 所有运行和测试均直接 `import senza`（PyO3 .so 原生扩展），无 Docker/容器/代理层
- `test_mock_pipeline.py` 虽名为 mock，实际使用真实 Senza WorkflowEngine，仅测试数据为最小化构造
- CLAUDE.md 已同步更新，移除所有"镜像/mock"相关描述

**2026-07-21 变更（#24-#26 + #33 全部修复）**：
- **#24 ContextAdvisor (LLM ReAct)**：新增 （list_files/read_file/ask_user/submit_context 4 个工具 + schema）+ （build_context_with_react，通过 senza SDK HarnessBuilder tool calling + should_stop_hook 自动 ReAct 循环）。 在 llm-auto/llm-interactive 模式下无 calibration_context.json 时启动 ContextAdvisor 收集参数。
- **#25 llm-interactive 模式**：CLI 新增  选项，EdaConfig 新增  +  回调字段，CLI 自动接线 stdin 交互回调。
- **#26 gauge_check 用户交互**：gauge_check 检测到 FAIL gauge 时调用 ，提供删除/保留/查看三个选项。删除 →  生成 cleaned gauge 文件。无回调时默认保留。
- **#33 关闭**：#24 和 #26 已实现，schema 对齐已完成。
- 16 个新测试（context_tools + gauge 交互 + config 回调）。总测试 83/83 通过。commit 791cf88。

**2026-07-21 变更（#27-#32 对齐 ArcGen c15dc55）**：
- **#27 resist_quality_llm 移除**：pipeline.yaml 删除 resist_quality_llm 节点，resist_quality 直接路由到 validate_pre_model_check。executor.py 质量门元组移除该节点。
- **#28 route_findoptics vizier skip 修正**：next_on_skip 从 validate_pre_run_optical_search 改为 validate_pre_mask_search（vizier 模式跳过整个光学路径）。
- **#29 route_prepare_wizard 条件路由**：新增 route_prepare_wizard checker 节点，双 vizier（mask+optics）时跳过 prepare_wizard。
- **#30 动态 fields_to_clear_for_restart**：新增 `orchestrator/ownership.py`（镜像 ArcGen orchestration/ownership.py），common.py 和 model_check.py 的硬编码 `_CLEAR_FROM` 字典替换为动态函数调用。
- **#31 lite_check Check A/C/D/E 对齐**：Check A/E 添加 final_uwrms 优先读取；Check C 无边界参数改为 continue（不产生 WARNING）；Check D real_contributions + terms_summarize 添加 FAIL 分级（<0.01 FAIL, <0.02 WARNING）。全部 diff 与 ArcGen c15dc55 一致。
- **#32 F/G 策略改写 term_pool.json**：策略1（调 sigma）和策略2（调 beta）从修改 wizard.json 改为读写 term_pool.json（Lite 模式下 PanGen 实际读取的文件）。移除 _load_wizard_model/_save_wizard/_resist_param_page 死代码。
- 总测试 67/67 通过。commit 05796cf。

**2026-07-21 变更（#35-#40 对齐 ArcGen 88c1862 — 6 项显式不对齐修复）**：
- **#35 ownership.py API 对齐**：`fields_to_clear_for_restart()` 返回值从 `set[str]` 改为 `dict[str, object]`（field→reset value），新增 `CONDITIONAL_NODES`/`QUALITY_GATE_NODES`/`NODE_PREDECESSOR`/`_FIELD_RESET`/`QUALITY_GATE_BY_RESTART_TARGET`，移除 `preserve_feedback` 参数（ArcGen 在 caller 层处理保留逻辑）。common.py 和 model_check.py 调用方更新。
- **#36 tried_decisions pipeline**：`DecisionCache` 添加 `tried_decisions_for_terms()` 方法；`_build_lite_prompt` + `decide_k_candidates_with_react_lite` 添加 `tried_decisions`/`force_upgrade_group` 参数；prompt 注入"已试过决策"和"强制 upgrade_group"段落；`decide_with_react_lite` 修复 `feedback_instructions` 传递；term_selection.py 调用方接线。
- **#37 beam_runner async/sync**：记录为 intentional deviation D-37（Senza PyO3 无 asyncio 事件循环，ThreadPoolExecutor 是正确并行方案）。
- **#38 term_advisor 文件拆分**：`term_advisor_lite.py`（1264 行）拆分为 3 文件匹配 ArcGen 结构：`term_advisor_lite.py`（入口，179 行）+ `term_advisor_ops.py`（启发式 ops，332 行）+ `term_advisor_prompts.py`（LLM prompts，790 行）。
- **#39 ContextAdvisor KB 集成**：`build_context_with_react` 添加 `kb=None` 参数 + KB 读取逻辑；新增 `try_build_kb()`（返回 None）+ `write_context_to_kb()`（no-op）stub，接口对齐 ArcGen F-42。
- **#40 gauge_check LLM reporter**：`result_analyzers.py` 添加 `GAUGE_CHECK_REPORTER_SYSTEM_PROMPT` + `build_gauge_check_reporter_prompt()`；`data_prep.py` 用 `single_llm_call_json` 替换 placeholder，对齐 ArcGen GaugeCheckReporter (F-48)。
- 总测试 83/83 通过。commits d3aef0a → 413c2c0。

**2026-07-21 变更（#43 runtime 恢复 API CLI 接线）**：
- **#43 CLI 恢复 API 接线**：`run_workflow()` 接受 `task_store_dir`/`resume_task_id`/`restart_from_step` 参数；新 run 调 `with_task_store()` 持久化 + 写 `.task_id` 文件；CLI 新增 `--restart-from NODE`/`--list-runs`/`--task-id` flag；`--resume` 从空壳接线到 `engine.restore()`；`--list-runs` 调 `WorkflowEngine.list_tasks()` 打印表格。TaskStore 目录：`<job_dir>/.task_store/`。runtime issue #74 (`list_tasks()`) 已实现。
- 总测试 89/89 通过。commit 44a6a34。

**2026-07-21 变更**：
- **Senza v0.4.6 升级**：从 v0.3.0 升级到 v0.4.6（跨 3 个版本）。`single_llm_call_json` 启用 `response_format(json_object)` 原生 JSON 模式（OpenAI 兼容 provider 受益，Anthropic 静默忽略走 regex fallback）。`single_llm_call` 新增 `json_mode` 参数。5 个新 FFI API 测试（response_format/fs_tools_plugin/after_turn_hook/structured_status）。67/67 测试通过。核心 API（HarnessBuilder/WorkflowEngine/create_executor/judge）完全向后兼容，无需改动。

**2026-07-20 变更**：
- **#13 mask_search BO 引擎路由**：添加 `_choose_engine()`（llm-auto → vizier，default → pangen）+ `_mask_search_bo()` 子进程调用 ArcGen `mask_bo_subprocess`（两阶段等待 + JAX 挂死兜底）+ `_bounds_from_ctx` + `_detect_gpus`。`run_mask_search` 重构为路由器，pangen 路径提取为 `_mask_search_pangen`。EdaConfig 添加 `bo_batches`/`bo_random_seed`/`bo_patience`/`bo_min_batches`/`bo_gpu_ids`。13 个新测试通过。
- **#15 retry 上下文传递**：`model_check_feedback` 添加 `_build_terms_override()`（Check D 低贡献 term 自动禁用），LLM + rule-based 双路径写入 `prev_selected_terms` + `max_rounds_override=4` + `initial_terms_override` + 上下文注入。`term_selection` 读取 `max_rounds_override` + `_is_retry` + 降低 `consecutive_fail_limit`（10→3）。6 个新测试通过。
- 总测试 62/62 通过。

**Python 版 EDA Agent**：通过 PyO3 SDK 复用 runtime，对齐 ArcGen AMC lite pipeline。
Rust 版（eda-agent）作为交叉验证。ArcGen 是对齐基准（源头）。

**当前状态**（2026-07-17）：38 stage pipeline 全部节点已实现，E2E 全流程跑通（default + LLM 模式）。
PyO3 SDK（`llm_harness_py`）已替换 cffi FFI（`llm_harness_sdk`），CLI 已迁移到 PyO3 路径。
PyO3 E2E 全流程跑通：33 步全部执行，pipeline succeeded。
resist_tune → resist_quality → model_check → gauge_error_attribution →
model_check_feedback → calibration_report 全部通过。

**2026-07-13 变更**：
- **cffi FFI 全删，迁移到 PyO3 SDK**：`llm-harness-ffi` crate 删除，`llm-harness-py` crate（PyO3 0.29）替代。eda-agent-py 完成 migration（commit `efba9a1`）：agent_call.py 用 HarnessBuilder，ffi_bridge.py 用 PyO3 WorkflowEngine + create_executor/create_judge，cli.py 简化为 run_workflow() 入口。43/43 测试通过，33-stage E2E --no-llm pipeline 成功。
- data_clean/gauge_group/prepare_wizard 添加 skip-if-wizard-exists（对齐 Rust tool_handler.rs）
- gauge_group 修复 self-copy crash（resume 时 validation_gauges 路径与目标相同）
- validate_pre_gauge_group/validate_pre_prepare_wizard 添加 pre-prepared 模式
- 12 个 PyO3 SDK 能力测试通过（test_ffi_gaps.py 重写为 PyO3 测试）
- 33 stage E2E 序列与 ArcGen graph.py 拓扑完全对齐

**本轮对齐修复**（ArcGen 源码逐节点对比）：

- **term_selection.py**：添加 vault_path 解析（OBSIDIAN_VAULT_RELPATH + fallback）、save_experience case_info 从 wizard.json 提取、status 使用 check_report（最后循环值）、_finalize 使用 best_terms or current_terms
- **calibration.py (resist_tune)**：添加 optical_result/mask_params 空值检查（对齐 ArcGen 优雅降级）
- **model_check.py Check A~E**：全面对齐 ArcGen
  - Check A：添加 val_uwrms 读取（validation/.calibratefiles）+ anchor_max_abs（gauge.txt anchor 行）+ 3 条标准全部检查
  - Check B：使用 config.py 常量（3.0/5.0）替代错误的 lite_config（500.0），添加 ax_/bx_ 检查，包含所有 term
  - Check C：从 lite/term_pool.json 读取边界（替代 wizard.json 错误源），fallback 到 autoterm yaml
  - Check D：修复阈值 1.0/2.0 → 0.01/0.02（分数非百分比），FAIL → WARNING，添加 active_term_names 过滤
  - Check E：实现完整曲率方向检查（numpy，AI vs RI 二阶导数不一致 > 20% → FAIL），替代 stub
  - 添加 valcheck_enabled 逻辑 + _try_read_val_uwrms() + _read_anchor_max_abs() + _find_grid_txt model_path 检查
- **optical.py**：_read_optical_ranges 添加 pages[1] fallback，_expand_optical_range 添加近边界扩展
- **data_prep.py (gauge_check)**：从 ArcGen 移植完整结构（_read_gauge_rows + _compute_gauge_overall + _layout_parser.py），klayout 可用时执行几何检查
- **验证对齐（无需修改）**：lite/beam_runner、lite/term_advisor_lite、lite/decision_cache、lite/effectiveness_check、lite/lite_check、mask_quality、data_clean

**已知限制**：
- ✅ FFI WorkflowEngine 已暴露（G1/G2/G3 已修复），CLI 已迁移到 FFI 路径（commit c416ee0）
- FFI system_prompt 已修复（G4/F-01），通过 HarnessConfig 传递
- klayout 未安装，gauge_check 几何检查 SKIPPED（非阻塞）
- ✅ vizier BO 引擎路由已实现（#13）：`_choose_engine` + `_mask_search_bo` 子进程调用 ArcGen `mask_bo_subprocess`，llm-auto 模式自动路由到 BO，default 模式走 pangen
- pframe_model_check.py grid_check session（distributed 模式）segfault — PanGen 内部 bug，非阻塞
- PanGen cal session 死锁 — stall detection workaround 已覆盖 calibrate/compute_tcc session

**E2E 验证结果**（2026-07-11）：
- Job dir: `/data/pangen_result/eda_py_e2e_1783742578`
- FFI E2E Job dir: 同上（复用 cached done markers），日志 `/tmp/eda_py_ffi_cached.log`
- 运行模式：`--no-llm`（quality gates auto-pass，term_selection heuristic fallback）
- LLM 模式 E2E（2026-07-13）：Claude Sonnet 4 驱动，37 步 pipeline succeeded。Job dir `/data/pangen_result/eda_py_ffi_llm_1783911053`，日志 `/tmp/eda_py_ffi_llm.log`。optical score=92 PASS, mask score=62 WARNING, resist LLM 触发 execute_restart 回流, term_selection 11 轮 best_uwrms=0.004, resist_tune cal_uwrms=0.0039, gauge_error_attribution dominant=resist, calibration_report LLM 生成
- resist_tune: cal_uwrms=0.0149, 13 terms, threshold=1.087
- model_check: overall=FAIL（Check B Gx 系数超限 — 测试数据质量问题，非代码 bug）
- calibration_report: 已生成
- 已修复 Bug：PanGen subprocess 死锁、stall detection、resist_tune TCC 参数不匹配（A027）、node_name NameError（#1）、from_env base_url 缺失（#3）

---

## Senza (森座) Python SDK — 2026-07-14 更新

**FFI 接口打磨**（`llm-harness-py` crate，PyO3 0.29）：

### WorkflowEngine 新增方法（pyworkflow.rs）
- **P0**: `restore()` classmethod — 从 TaskStore 恢复崩溃中断的 workflow
- **P1**: `pause(reason)` / `resume()` / `cancel(reason)` — 流程控制
- **P1**: `state()` / `current_step()` / `step_history()` — 运行状态查询
- **P1**: `checkpoint(desc, payload)` — 手动检查点
- **P1**: `total_cost()` — 累计 token/成本查询
- **P2**: `with_task_store(dir)` / `with_max_steps(n)` / `with_max_retries(n)`
- 核心架构变更：`engine` 字段从 `Option<WorkflowEngine>` 改为 `Option<Arc<WorkflowEngine>>`，使 `pause()`/`cancel()` 等可在 `run()` 期间从另一线程调用

### AgentHarness 新增方法（pyharness.rs）
- 动态配置：`set_model` / `set_system_prompt` / `set_temperature` / `set_thinking_level` / `set_max_tokens` / `set_tools`
- Steering：`steer` / `follow_up` / `next_turn` / `continue_run`
- 成本：`usage()` / `reset_usage()`
- 等待：`wait_for_idle()` / `wait_for_settled()`

### 测试
- 20 个新测试（`test_new_methods.py`）全部通过
- 93 个已有测试全部通过（14 个 pre-existing 失败：`Agent` 实例化 + `tool.drive()` 缺失，与本次改动无关）

### Senza 仓库（github.com/oh-my-harness/Senza，原 llm-harness-py-wheels）
- 仓库已改名 → Senza，PyPI 包名 `senza-sdk`（import 名 `senza`，PyPI `senza` 被 Zalando 占用）
- 14 个 examples（5 agent + 6 runtime + 3 advanced，含 `09_composite_judge.py`）已推送
- 类型 stub（`.pyi` + `py.typed`）：276 行手写 `senza-pkg/senza/__init__.pyi`，maturin `--generate-stubs` 构建
- CI pipeline（`.github/workflows/build-wheel.yml`）：tag push → checkout Senza + runtime → `maturin build --release --generate-stubs` → GitHub Release → PyPI publish
- GitHub Release v0.1.0 已发布（wheel asset 附带）
- eda-agent-py 已从 `import llm_harness_py` 迁移到 `import senza`（43 测试通过）
- **待做（阻塞，需用户操作）**：PyPI 发布需用户注册 `senza-sdk`、设置 `PYPI_API_TOKEN` secret 后重新 tag 触发 CI
- **待做（非阻塞）**：CI 平台矩阵扩展（当前仅 manylinux_2_34_x86_64，待加 macOS + Windows）

### 2026-07-14 追加更新

- **docstring gap 已关闭**：PyO3 0.29 自动导出 Rust doc comments 为 `__doc__`，全部函数/类/方法已覆盖（补了 `version()` 和 `create_event_channel()` 的 doc comment）
- **AgentHarness context manager**：`__enter__`/`__exit__` 已添加（`__exit__` 调 `abort()` 清理，不抑制异常）
- **Builtin executor factories**：`create_shell_executor(commands, ...)` 和 `create_http_executor(allowed_hosts, ...)` 已暴露。不自动注册（安全设计），用户需 `engine.with_executor(name, exec)` 显式注册。`PyExecutorWrapper` 改为持有 `Arc<dyn StepExecutor>` 以支持任意 executor 类型
- **测试**：23 个新测试全部通过，119 个已有测试无回归
- **剩余 gap**：仅 `WorkflowEngine.run()` async 版本（P2，设计层面 — 当前同步阻塞够用）

### 2026-07-17 更新

- **llm_adapter rev bump**：runtime `Cargo.toml` 的 `llm_adapter` rev `30ae9284 → e44758b8`
  （llm-api-adapter 同事 Ryan Yu 推入的 `set_thinking_level` 修复）。
  `e44758b` 把 Anthropic 4.7+ 的 `effort` 从嵌套 `thinking.adaptive.effort`（API 拒绝）
  移到顶层 `output_config.effort`。改动为 `pub(crate)` 内部，公开 API 不变。
- 验证：`cargo build --workspace` ✅，`cargo test -p llm-harness-loop --features test-utils` 89 passed ✅，
  Senza wheel 构建可见 `rev=e44758b8` ✅，`import senza` ✅，eda-agent-py 43 测试 ✅
- commit `5ed4915`（runtime main），rebase 后线性推送

### 2026-07-19 Senza FFI 生产级交付修复（issue #63/#64/#65）

三个阻塞生产级交付的 FFI issue 全部修复并关闭（commits 在 Senza 仓库 main）：

- **#65** `create_anthropic_provider` 暴露 `messages_path` 参数（commit `0effe42`）：Rust 侧 `AnthropicProviderBuilder.messages_path()` 本已存在，未暴露到 Python。现与 `create_openai_provider` 的 `chat_path` 对称，Anthropic 兼容代理（Azure/Bedrock/自建网关）客户可自定义 API 路径。`.pyi` 同步。
- **#63** `PyAgent` 类注册门控到 `test-utils` feature（commit `8fec8ac`）：此前 `add_class::<PyAgent>` 无条件注册但 `#[new]` 用 `MockLlmClient`（test-only），生产 wheel 下 `senza.Agent(...)` 报 `TypeError`，19 测试失败。现生产 wheel 不暴露 `Agent`（入口仍是 `HarnessBuilder`→`AgentHarness`）。新增 `tests/conftest.py`：生产 wheel 下自动跳过 4 个 test-utils 测试模块。
- **#64** 8 个 `with_*` 方法静默失败修复（commit `c7ced22`）：`with_tool`/`with_external_tool`/`with_max_tokens`/`with_step_plugin`/`with_executor`/`with_task_store`/`with_max_steps`/`with_max_retries` 在 `Arc::try_unwrap` 失败（engine 共享/运行中）时静默丢弃操作。现统一返回 `PyResult`，失败抛 `RuntimeError`，与 `with_hooks` 一致。

验证：release 构建通过（test-utils + 生产 abi3 两 wheel）；`cargo fmt --check` 干净；`cargo clippy --lib -D warnings` 无警告；`check_stubs.py` 140 签名无漂移；test-utils wheel 225 passed，生产 wheel 204 passed + 21 skipped。

此前已核实关闭的已修未关 issue：#62（`restore_from_step`）、#66（README/examples/docs）、#67（CI wheel workflow）、Senza #3（`with_max_retries` docstring）、Senza #4（`create_os_env` 接入）。至此 Senza 仓库 0 open issue，runtime 仓库 FFI/交付阻塞 issue 清零。

### 2026-07-19 Senza FFI 后续修复（Senza #5/#6）

- **Senza #6** `.pyi` stub 漂移修复（commit `dff2d98`）：#63 只门控了 `m.add_class`，但 PyO3 experimental-inspect 的 stub 生成器编译时看到 `#[pyclass]` 就生成 `class Agent`，导致生产 `.pyi` 声明了 Agent 的 5 个方法但运行时无 `senza.Agent`。现将整个 `PyAgent`（`#[pyclass]` + inherent impl + `#[pymethods]`）门控到 `test-utils`，从生产 `.pyi` 删除 `class Agent`，`check_stubs.py` 白名单加入 5 个 Agent 方法。验证：生产 + test-utils 两种 wheel `check_stubs` 均零漂移（135 签名）。
- **Senza #5** PyPI Linux wheel manylinux 标签修复（commit `a702282`）：CI 在 ubuntu-latest（glibc 2.39）直接构建，产出 `manylinux_2_39`，RHEL 8/Ubuntu 20.04/Debian 11 装不了。改用 `PyO3/maturin-action@v1` + `manylinux: 2_28` 容器构建，产出 `manylinux_2_28`（覆盖 glibc 2.28+）。加 `before-script-linux` 在容器内重建 `RUNTIME_PAT` 凭证使 cargo 能拉私有 runtime crate。macOS/Windows 不受影响。建议打 v0.4.3 tag 触发 CI 验证 PyPI 实际标签。

### 2026-07-19 Senza examples/README 修复 + v0.4.3 发布验证（Senza #7/#8/#9）

- **Senza #8** README `create_anthropic_provider` 签名补 `messages_path`（commit `74601d6`）。已关闭。
- **Senza #7** 4 个 example bug 修复（commit `f2f1516`）：① `03_executor_steps.py` judge 总 abort + ctx key `output`→`prev_output`；② `09_composite_judge.py` 加 `parse_score` executor step 产 structured 让 condition 可匹配；③ `01_basic_prompt.py` env var `OPENAI_BASE_URL`→`OPENAI_API_BASE`；④ 14 个 example 硬编码 `gpt-4o`→`os.environ.get("SENZA_MODEL", "gpt-4o")`。已关闭。
- **v0.4.3 发布验证**：tag `v0.4.3` 已触发 CI，Build Wheel 成功（首次 `98dc474` 因 `ls|head` SIGPIPE pipefail 失败，`948f92a` 修复后重跑成功）。GitHub Release 发布 3 个 wheel：`senza_sdk-0.4.3-cp39-abi3-macosx_11_0_arm64.whl`、`senza_sdk-0.4.3-cp39-abi3-manylinux_2_28_x86_64.whl`、`senza_sdk-0.4.3-cp39-abi3-win_amd64.whl`。**Linux wheel 标签确认为 `manylinux_2_28`，Senza #5 修复验证通过。**
- **Senza #9**（新开，enhancement）：examples 覆盖缺口。经 `rg` 全量比对 `.pyi` stub 与 examples 实际调用，确认以下 API 零示例覆盖——P0 安全/合规/成本（Rules 6 API + Hooks 11 API + Budget/Pricing 3 API）、P1 核心功能（Skills/Plugin/`create_sync_tool`）、P2 AgentHarness 高级方法（steering/session-branch/context-manager/waiting/queue/builder 配置）、P3 WorkflowEngine 补充（`with_hooks`/`with_max_retries`/`restore_from_step`）、P4 其他（Anthropic 独立示例/utilities）。
- **Senza #9 P0 已补**（commit `a84be97`）：新增 3 个 example 覆盖 P0 全部 API——`examples/agent/06_hooks.py`（11 个生命周期 hook，观察性 + before_tool_call/should_stop 决策）、`07_rules.py`（Rules 4 种 predicate + 审批 hook，规则链 first-match-wins 经逻辑验证）、`08_budget_pricing.py`（pricing provider + budget exceeded hook + usage 成本追踪）。验证：3 文件 py_compile 通过、senza 模块 API 存在性检查通过（missing: NONE）、RuleChainBuilder/HarnessBuilder 方法签名验证通过。剩余 P1–P4 缺口已全部补充完成（commit `b324129`），新增 6 个 example：
- P1：`09_skills.py`（load_skills + skill_read 工具发现）、`10_plugins.py`（create_plugin 打包 tools+hooks，agent+workflow 双层，sync+async 工具）
- P2：`11_steering.py`（多轮 steer/follow_up/next_turn/continue_run + 队列 + 上下文管理器）、`12_session_branch.py`（fork_branch/navigate_tree/list_branches/generate_branch_summary/delete_branch）
- P3：`runtime/10_hooks_retries.py`（with_hooks + with_max_retries + restore_from_step）
- P4：`13_anthropic_standalone.py`（Anthropic provider + version/to_json/from_json）
验证：6 文件 py_compile 通过、API/方法存在性检查全通过。现有 20 个 example 覆盖 .pyi stub 全部主要 API。issue #9 已关闭。

### 2026-07-17 eda-agent-py 更新
### 2026-07-17 eda-agent-py 更新

- **E2E default + LLM 模式跑通**：33 步 pipeline 完整执行（Qwen3.5-397B LLM），与 ArcGen 参考运行（optical_search_20260713115140）双跑对比：
  - Stage 序列 ✅ 一致（33 步）
  - model_check ✅ 一致（FAIL）
  - 数值差异 ⚠️ 预期内（LLM 选了不同 group，级联差异）
- **LLM 集成修复**（commit 3459fb4）：
  - `agent_call.py`：`prompt_and_collect()` 替代 `prompt()`+`collect_until_settled()`（修复 broadcast channel 晚订阅导致 0 事件）
  - `config.py`：strip `base_url` 尾部 `/v1`（senza chat_path 已含 `/v1`）
  - `beam_runner.py`：`_REQUIRED_PATTERNS` 添加 `calibration_context.json`（修复 PanGen SIGSEGV）
  - `term_selection.py`：添加 done marker 缓存（断点续跑）
- **Case 数据修复**：`calibration_context.json` 补回 `gds_layers`（submodule commit 8705bcb 添加后被移除）
- **G6/G7 已暴露**：PyO3 SDK 已暴露 `restore()` / `pause/resume/cancel`（default 模式暂未使用）
- Issues：#5（stall detection 已移除）、#6（beam_runner calibration_context.json 已修）
- 43/43 测试通过，CLAUDE.md 已更新

### 2026-07-29 runtime issue #97 修复：BwrapSandbox 真实隔离 + SeatbeltSandbox fail-closed

- **问题**（issue #97）：`BwrapSandbox` 和 `SeatbeltSandbox` 的 `env()` 返回无限制的 `OsEnv`，声称提供 OS 级隔离但实际未执行任何隔离——`start()` 仅验证二进制存在，`build_bwrap_args()`/`build_sandbox_profile()` 被 `#[cfg(test)]` 限定，生产代码从不调用。
- **BwrapSandbox（Linux）修复——option 1，真实 bwrap 隔离**：
  - 新增 `BwrapEnv`（`ExecutionEnv` 实现）：文件操作委托 `OsEnv`，`execute_shell` 通过 `bwrap --unshare-all` 在独立 mount/net/pid namespace 中执行每条命令。
  - 隔离机制：`--ro-bind-try` 绑定系统目录（/usr /etc /bin /lib /lib64 /sbin）；`--bind` 绑定工作目录 + `fs_allowlist`；`--tmpfs /tmp`；`--proc /proc --dev /dev`；`--die-with-parent --new-session`。
  - 环境隔离：`env -i` 清空所有环境变量，仅注入 `PATH`、`HOME` 和调用方指定的变量——主机凭证（API key、token）不继承。
  - 网络隔离：`net_allowlist` 为空时 `--unshare-all` 阻断网络；非空时 `--share-net`。
  - `reset()`：清空工作目录（disposable root），不影响外部路径。
  - `start()`：验证 bwrap 可用性，不可用时 fail-closed。
  - 复用 `OsEnv::execute_shell` 的 drain/timeout/abort 逻辑（`utf8_safe_boundary` 已从 sandbox-os 导出为 pub）。
- **SeatbeltSandbox（macOS）修复——option 2，fail-closed**：
  - `start()` 返回错误（sandbox-exec 策略编译未实现）。
  - `env()` 返回 `UnsupportedEnv`（所有操作返回 `EnvError::Other`），不再返回无限制 `OsEnv`。
  - 移除了未使用的 `llm-harness-runtime-sandbox-os` 依赖。
- **测试**（11 个，全部通过，通过 `sandbox.env().execute_shell()` 执行真实命令）：
  - 文件系统隔离：shell 命令无法读取 allowlist 之外的宿主文件 ✓
  - 网络隔离：空 net_allowlist 阻断网络（`/dev/tcp/8.8.8.8/53` → "Network is unreachable"）✓
  - 凭证隔离：主机环境变量（USER/LANG/HOSTNAME/LOGNAME/TERM）不泄露 ✓
  - 超时杀进程树：`sleep 30` + 2s timeout → 2s 内终止 ✓
  - abort 取消：CancellationToken 取消运行中命令 ✓
  - workspace reset：仅清空工作目录，外部文件存活 ✓
  - fs_allowlist 授权读取 ✓；注入环境变量可见 ✓；工作目录文件可访问 ✓
- **全量验证**：`cargo test --workspace` 762 测试全通过；`cargo clippy` 零警告。

### 2026-07-29 runtime issue #97 round-2：审查缺口全部封闭（commit 86fa2ef）

- **背景**：round-1（9e9edd3）提交后收到安全审查，列出 7 项缺口（3×P0、3×P1、1×P2）。round-2 逐项修复并验证。
- **P0 文件 API 绕过 bwrap**：`BwrapEnv` 的 read/write/list/remove/file_info 全部经 `check_path()` 校验，路径必须在 work_dir 或 `fs_allowlist` 内；绝对路径、`..` 遍历、跨目录逃逸均被拒。
- **P0 配置静默降级**：`BwrapSandbox::new()` 改为 `Result<Self>`，`validate_config()` 对 `fs_denylist`、`net_allowlist` 规则、`max_cpus/max_memory_mb/max_disk_mb` 直接 bail（fail-closed），不再静默放宽。
- **P0 /etc 暴露**：`system_ro_binds()` 移除 `/etc`（不再泄露 `/etc/passwd`、`/etc/shadow` 等）。
- **P1 reset() 误删宿主目录**：新增 `is_safe_disposable_root()`，拒绝根路径、当前工作目录及其祖先；`reset()` 调用前先校验。
- **P1 进程树未回收**：抽取共享 `drain_child_output()`（sandbox-os/src/shell_runner.rs），超时/abort 时 `start_kill()` + `wait()`；bwrap `--unshare-all` 使其成为 PID namespace init，杀死后内核回收整个进程树。sentinel 测试验证后台子进程被终止。
- **P1 测试假绿**：bwrap 依赖测试改为 `#[ignore = "requires bwrap; run with --include-ignored"]`，默认 `cargo test` 报告 "ignored"（不再是假 "passed"）；`require_bwrap()` 在 `--include-ignored` 且 bwrap 缺失时 panic（大声失败）。不依赖 bwrap 的测试（config 校验、文件 API 路径检查、reset 安全）始终运行。
- **P2 耦合/重复**：`drain_child_output` 抽到 sandbox-os 共享，`utf8_safe_boundary` 恢复为私有，不再跨 crate 暴露。
- **验证**：bwrap 19 测试全过（`--include-ignored`，真实 bwrap 0.4.0）；sandbox-os 15 测试全过；三 crate `clippy -D warnings` 零警告；`cargo build --workspace --all-targets` 通过。默认 `cargo test -p sandbox-bwrap` = 9 passed / 10 ignored（诚实）。
- **CI 提示**：Linux CI 需安装 bwrap 并用 `cargo test -- --include-ignored` 才会执行真实隔离测试。

### 2026-07-30 runtime issue #97 round-3：cap-std 能力文件系统 + CI 真实隔离门禁

- **背景**：round-2（86fa2ef）后第二轮审查发现 round-2 的词法路径检查仍可被符号链接绕过（P0），`reset()` 可删除 `/tmp`（P0），`create_temp_dir()` 写入宿主 `/tmp`（P1），配置级 timeout 被忽略（P1），CI 未执行 bwrap 测试（P1），`cargo fmt --check` 失败（P1），能力检查仅测 `--version`（P2）。round-3 用 cap-std 能力文件系统彻底修复。
- **P0 符号链接逃逸**：所有文件操作改为通过 `cap_std::fs::Dir` 能力对象执行。每个允许根（work_dir + fs_allowlist）在构造时以 ambient authority 打开一次，后续 read/write/list/remove/exists/create_dir/file_info/append 全部委托给 `Dir` handle。cap-std 在 syscall 层阻止 `..` 遍历、绝对路径逃逸和符号链接逃逸——不再存在检查与使用分离（TOCTOU）。
- **P0 reset() 误删共享目录**：`is_safe_disposable_root()` 现在 canonicalize 路径，拒绝根 `/`、少于 2 个路径分量的浅路径（如 `/tmp`、`/home`）、当前工作目录及其祖先。`reset()` 调用前先校验。
- **P1 create_temp_dir() 写入宿主 /tmp**：改为在 workspace 内通过 cap-std `Dir::create_dir` 创建，返回路径在 work_dir 下。
- **P1 配置级 timeout 被忽略**：`execute_shell` 现在取 `min(opts.timeout, config_timeout)` 作为强制上限；若调用方未设 timeout，则用 config timeout。
- **P1 CI 未执行 bwrap 测试**：`.github/workflows/ci.yml` 新增 Linux 步骤安装 bubblewrap + `cargo test -p llm-harness-runtime-sandbox-bwrap -- --include-ignored`。
- **P1 cargo fmt 失败**：已 `cargo fmt --all`，`--check` 通过。
- **P2 能力检查不充分**：`run_bwrap_probe()` 运行真实 `bwrap --unshare-all ... true`（不是仅 `--version`），使用 `system_ro_binds()`（含 `/lib64`/`/sbin`）+ `env_clear()`。`BwrapEnv::new` 改为 `pub(crate)` 防止绕过 `validate_config`。
- **修复 bug**：初版 `check_bwrap_available()` 硬编码 `/usr`/`/bin`/`/lib` ro-bind 但遗漏 `/lib64`，导致 RHEL 上 `/usr/bin/env` 因找不到动态链接器而 execvp 失败，全部 11 个 bwrap 测试失败。改用 `BwrapEnv::system_ro_binds()` 后修复。
- **测试修复**：原 `bwrap_sandbox_start_fails_when_unavailable` 测试要求 bwrap 不存在，在 bwrap 存在的机器上 `--include-ignored` 会 panic。替换为 `bwrap_probe_fails_when_binary_missing`——用不存在的二进制路径测试 probe 的 not-found 错误路径，任何机器都能跑。
- **验证**：bwrap 25 测试全过（`--include-ignored`，真实 bwrap 0.4.0，kernel 4.18）；workspace 全量测试 0 失败；`cargo fmt --check` 通过；`cargo clippy --workspace --all-targets --all-features -- -D warnings` 零警告；`cargo build --workspace --all-targets` 通过。默认 `cargo test -p sandbox-bwrap` = 15 passed / 10 ignored。

### 2026-07-30 runtime issue #97 round-4：reset 所有权模型 + 4 项测试覆盖缺口

- **背景**：round-3 提交后第三轮审查确认 25 测试全过，但指出 4 项未被测试覆盖的场景，补充探针后全部失败。round-4 逐项修复。
- **P0 reset() 使 Dir handle 失效**：`reset()` 原先执行 `remove_dir_all` + `create_dir_all`，导致 cap-std `Dir` 能力对象指向已删除 inode，后续 `write_file()` 返回 NotFound。改为 `clear_work_dir_contents()`——通过 `Dir` handle 遍历并删除所有条目（使用 `symlink_metadata` 防止跟随符号链接），不销毁目录 inode，`Dir` handle 保持有效。reset 后 `write_file()` / `read_text_file()` 正常工作。
- **P0 /var/lib 通过 is_safe_disposable_root**：`/var/lib` 有 2 个路径分量且非 cwd，原检查放行。新增两道防线：(1) `SYSTEM_DIRECTORIES` 常量列出 22 个已知系统目录（/var/lib、/var/log、/etc、/usr 等），`is_system_directory()` 检查 canonical 路径是否命中；(2) sentinel 文件 `.bwrap-disposable`——`BwrapEnv::new()` 仅在 `is_potentially_disposable()` 通过时写入 sentinel，`is_safe_disposable_root()` 要求 sentinel 存在才放行。/var/lib 不满足 `is_potentially_disposable()` → 不写 sentinel → reset 拒绝。
- **P0 work_dir=None 共享 /tmp/sandbox-bwrap**：多个 sandbox 使用相同默认路径，存在跨 Agent 文件泄露和相互 reset。改为 `work_dir=None` 时生成 `/tmp/sandbox-bwrap-<uuid>` 唯一路径。
- **P0 symlink work_dir 被静默 canonicalize**：`BwrapEnv::new()` 原先将 canonical 路径存入 `work_dir`，`working_dir()` 返回 canonical 路径而非配置路径。拆分为 `work_dir`（配置路径，`working_dir()` 返回此值）和 `work_dir_canon`（canonical 路径，cap-std Dir、bwrap bind/chdir、路径路由使用此值）。
- **验证**：bwrap 29 测试全过（`--include-ignored`，真实 bwrap 0.4.0）；4 项新测试（reset 保留文件 API、系统目录拒绝、work_dir=None 唯一性、symlink workdir 契约）全过；workspace build/clippy/fmt 全部通过。

### 2026-07-30 runtime issue #97 round-5：内存所有权模型 + 0700 权限 + 互斥锁 + shutdown 清理

- **背景**：round-4（ab8b099）引入 sentinel 文件所有权模型后，第四轮审查发现 sentinel 方案存在根本缺陷：sentinel 位于 Agent 可写目录内、构造器可主动认领任意既有目录、目录权限为 0755、shutdown 不清理、reset 与执行操作无互斥。round-5 彻底废弃 sentinel，改为内存所有权标记。
- **P0 sentinel 不能证明目录所有权**：构造器原先给任何通过路径启发式检查的既有目录写 sentinel，`/var/lib/dpkg` 等系统子树可被"认领"后 reset 清空。改为 `owned: bool` 内存字段——仅 `work_dir=None`（自动生成）时 `owned=true`，调用方提供的目录 `owned=false`。`reset()` 仅对 `owned=true` 执行，`owned=false` 直接拒绝。删除 `SYSTEM_DIRECTORIES`、`is_system_directory`、`is_potentially_disposable`、`SENTINEL_NAME`、`place_sentinel`、`is_safe_disposable_root`。
- **P0 默认 workspace 权限 0755**：`create_dir_all()` 受 umask 控制，同机其他 Unix 用户可读 Agent 文件。改为 `std::fs::DirBuilder::new().recursive(true).mode(0o700)` 创建自动生成目录，绕过 umask。
- **P1 Agent 可删除 sentinel 使 reset 失效**：sentinel 位于 Agent 可写 workspace 内，`remove(".bwrap-disposable")` 后 reset 永久拒绝。所有权改为内存 `owned: bool`，Agent 无法篡改进程内状态。
- **P1 shutdown 不清理自动生成目录**：`shutdown()` 原为空操作，`/tmp/sandbox-bwrap-<uuid>` 残留。改为 `owned=true` 时 `shutdown()` 获取锁并 `remove_dir_all`；`owned=false` 时不删除调用方目录。
- **reset 与执行操作无互斥**：运行中 Agent 可在清理期间重新创建内容，reset 空目录后置条件无保证。新增 `reset_lock: tokio::sync::Mutex<()>` 字段，`reset()`、`shutdown()`、`execute_shell()` 和全部文件 API 方法（read/write/list/remove/exists/create_dir/create_temp_dir/file_info/append）均获取锁，确保互斥。
- **验证**：bwrap 34 测试全过（`--include-ignored`，真实 bwrap 0.4.0，kernel 4.18）；新增 8 项测试覆盖所有权（reset 清空 owned 目录、拒绝 caller-provided 目录、拒绝 /var/lib、0700 权限、Agent 无法破坏 reset、shutdown 删除 owned 目录、shutdown 保留 caller 目录、reset 保留文件 API）；workspace 全量测试 0 失败；`cargo fmt --check` 通过；`cargo clippy --workspace --all-targets --all-features -- -D warnings` 零警告；`cargo build --workspace --all-targets` 通过。

### 2026-07-30 runtime issue #97 round-6：生命周期状态机 + 有界 shutdown + 排队取消感知

- **背景**：round-5（904c9b4）后第五轮审查发现两个生产阻断生命周期缺陷：(P0) 无 timeout 的 shell 命令持有 `reset_lock`，`shutdown()`/`reset()` 等待同一锁可被无限阻塞；(P1) 排队中的操作在被取消后仍会执行（取消仅在获取锁前检查，排队等待期间不感知取消）。
- **P0 无界命令阻塞 shutdown**：新增生命周期状态机 `LifecycleState`（Running/ShuttingDown/Stopped），存储为 `AtomicU8`。`shutdown()` 先 `begin_shutdown()` 原子转换到 ShuttingDown（拒绝新操作），再 `shutdown_token.cancel()` 取消活跃命令，再 `kill_active_children()` 在 `SHUTDOWN_KILL_GRACE`（5s）内有界杀死所有 bwrap 子进程，最后用 `tokio::time::timeout` 有界获取锁并删除 owned 目录。`execute_shell` 通过 `tokio::select!` 将命令与 `shutdown_token` 竞争，shutdown 触发时立即返回。
- **P1 排队取消不感知**：新增 `acquire_op_lock()` 方法，用 `tokio::select!` 将锁获取与调用方 `abort` token 和 `shutdown_token` 竞争。排队中的操作被取消时立即返回 `EnvError::Aborted`，不等待锁释放。获取锁后再次检查取消/生命周期状态（post-acquisition recheck），关闭 TOCTOU 窗口。
- **活跃子进程注册表**：`active_children: Mutex<Vec<u32>>` 存储 bwrap 子进程 PID。`register_child()` 在 `execute_shell` 中注册 PID，`ChildGuard` 在 drop 时移除。`kill_active_children()` 通过 `kill(2)` FFI 发送 SIGKILL，bwrap `--new-session` 使内核回收整个进程树。
- **契约缺口**：`reset()` 现在对所有 caller-provided `work_dir` fail-closed（仅 `work_dir=None` 的 owned 目录可 reset）。所有权/capability 由 `owned: bool` 内存字段显式标记，不再从 `Option<PathBuf>` 推断。
- **验证**：bwrap 36 测试全过（`--include-ignored`，真实 bwrap 0.4.0，kernel 4.18）；新增 2 项回归测试（无 timeout 命令的 有界 shutdown、排队中取消感知）；workspace 全量测试 0 失败；`cargo fmt --check` 通过；`cargo clippy --workspace --all-targets --all-features -- -D warnings` 零警告；`cargo build --workspace --all-targets` 通过。

### 2026-07-30 runtime issue #97 round-7：移除裸 PID 模型 + drain 内置 shutdown + kill_on_drop + 可交换 token

- **背景**：round-6（2c471b1）后第六轮审查发现三个生产阻断缺陷：(P0) `shutdown()` 不检查锁超时 `Err(Elapsed)`，直接 `remove_dir_all` 删除 workspace；(P0) spawn/register 竞态——child 在 spawn 后、register 前遇到 shutdown，cancel 分支丢弃 `drain_child_output` future 但不 kill/wait child；(P0/P1) 裸 PID 所有权模型不安全——`ChildGuard::drop()` 用 `try_lock()` 失败时残留 PID，后续 `kill(pid, SIGKILL)` 不 wait/reap，PID 复用可能误杀宿主无关进程。
- **P0 shutdown 锁超时不删 workspace**：`shutdown()` 用 `match` 检查 `tokio::time::timeout(SHUTDOWN_KILL_GRACE, reset_lock.lock())` 的 `Err(Elapsed)` 分支，超时时返回 `Err("shutdown timed out waiting for lock; workspace preserved")` 并不删除 workspace。
- **P0 drain 内置 shutdown kill+wait**：`drain_child_output()` 新增 `shutdown: Option<CancellationToken>` 参数，在两个 `select!` 中增加 biased `shutdown_token.cancelled()` 分支，触发时 `start_kill()` + `wait().await` + 返回 `EnvError::Other`。`execute_shell` 不再有外层 `select!`。
- **P0 kill_on_drop**：`execute_shell` 在 `Command` 上设置 `kill_on_drop(true)`。
- **P0 移除裸 PID 模型**：删除 `kill_process()` 函数、`libc_kill` FFI、`active_children` 注册表、`ChildGuard`、`kill_active_children()`。所有子进程生命周期由 `tokio::process::Child` 句柄管理。
- **可交换 shutdown token**：`shutdown_token` 从 `CancellationToken` 改为 `std::sync::Mutex<CancellationToken>`。`reset()` 通过 `swap_shutdown_token()` 原子替换为新 token 并取消旧 token。`shutdown()` 通过 `cancel_shutdown_token()` 取消当前 token。
- **验证**：bwrap 41 测试全过（`--include-ignored`，真实 bwrap 0.4.0，kernel 4.18）；新增 5 项确定性回归测试。

### 2026-07-30 runtime issue #97 round-8：epoch 线性化 + ReapGuard 保证 drop reap + 恢复原始签名

- **背景**：round-7（849ce51）后第七轮审查发现两个 P1：(P1) reset token epoch 不线性化——命令通过 acquire_op_lock 后再次 clone token，若 reset 恰好在两次 clone 之间 swap，已获准命令拿到新 token 逃过 reset 取消，确定性探针复现 reset 超时；(P1) 直接 drop/abort future 时不保证 reap——只依赖 kill_on_drop(true)，Tokio 明确说明 Drop 无法 await 子进程，只提供后台 best-effort reap。另外 no_orphan 测试基线测量有误；共享 drain_child_output 的公开签名被不必要地修改。
- **P1 epoch 线性化**：`acquire_op_lock()` 返回 `(MutexGuard, CancellationToken)`——在锁获取时 snapshot 当前 token 并返回给调用方。`execute_shell` 使用返回的 token（不再从字段重新 clone），保证 token epoch 线性化。所有文件 API 调用方更新为解构 `(guard, token)` 元组。
- **P1 ReapGuard 保证 drop reap**：新增 `ReapGuard` 结构体拥有 `tokio::process::Child`，Drop 时 `start_kill()` + `tokio::spawn(async { child.wait().await })` 后台 reap。新增 `drain_child_output_guarded(child, opts, shutdown)` 函数使用 ReapGuard——所有取消路径通过 `guard.kill_and_wait().await` kill+wait，future 被 drop 时 ReapGuard 的 Drop 保证后台 reap。移除了 `kill_on_drop(true)`。
- **恢复 drain_child_output 原始签名**：`drain_child_output(child, opts)` 恢复为 2 参数（无源码级破坏）。新增 `drain_child_output_guarded` 作为 bwrap 专用函数。OsEnv 继续使用原始 `drain_child_output`。
- **no_orphan 测试修复**：改为 sentinel 文件方式——命令 `sleep 3 && touch sentinel.txt`，shutdown 后等待 4s 验证 sentinel 不存在。不再依赖 `pgrep -c bwrap` 全局进程计数。
- **验证**：bwrap 43 测试全过（`--include-ignored`，真实 bwrap 0.4.0，kernel 4.18，并行执行）；新增 2 项确定性测试（reset epoch 线性化、dropped future 保证 reap）；sandbox-os 15 测试全过；workspace 全量测试 0 失败；`cargo fmt --check` 通过；`cargo clippy --workspace --all-targets -- -D warnings` 零警告。

### 2026-07-30 runtime issue #97 round-9：runtime-independent ReapGuard + Resetting lifecycle 关闭 reset admission race

- **背景**：round-8（76aafde）后第八轮审查发现两个 P1：(P1) ReapGuard 的 Drop 用 `tokio::spawn` 后台 reap，但 Tokio runtime 关闭时 task 可能在 poll 前被取消，且无 runtime 时 Drop 不触发任何 reap；`kill_and_wait()` 无论 `wait()` 成功与否都设 `reaped=true`。(P1) `reset()` 在 token swap 和 lock 获取之间不关闭 admission，新命令可能拿到未取消的新 token 滑入，导致 reset 超时。
- **P1 ReapGuard runtime-independent reap**：Drop 中的 `tokio::spawn` 替换为 `std::thread::spawn` + `try_wait()` 轮询循环（10ms 间隔）。OS 线程独立于 Tokio runtime 生命周期——runtime 关闭后或无 runtime 时仍能完成 reap。`kill_and_wait()` 仅在 `wait().await.is_ok()` 时设 `reaped=true`，失败时 Drop 仍尝试 reap。
- **P1 Resetting lifecycle 关闭 admission**：新增 `Resetting` 生命周期状态（Running→Resetting→Running）。`begin_reset()` 在 token swap 前原子转 Running→Resetting，`acquire_op_lock()` 在 pre-lock 和 post-lock 两处检查 `lifecycle != Running` 即拒绝，使 reset 期间新操作被立即拒绝而非排队等待。`finish_reset()` 重开 admission；超时时回滚到 Running。`begin_shutdown()` 同时接受 Running 和 Resetting 作为合法起始状态。
- **新增测试**：`reset_closes_admission_atomically`（手动持锁使 reset 阻塞在 lock 获取阶段，验证 Resetting 状态下新命令被拒绝，释放锁后 reset 快速完成并恢复可用）；sandbox-os 新增 4 项 ReapGuard 单元测试（无 runtime、runtime 关闭中、runtime 存活、kill_and_wait reaped 标记）。
- **验证**：bwrap 44 测试全过（`--include-ignored`，真实 bwrap 0.4.0，kernel 4.18，并行执行）；sandbox-os 19 测试全过（15 原有 + 4 ReapGuard）；workspace 全量测试 0 失败；`cargo fmt --all -- --check` 通过；`cargo clippy --workspace --all-targets -- -D warnings` 零警告。

### 2026-07-30 runtime issue #97 round-10：RAII lifecycle guard + 集中式 child reaper

- **背景**：round-9（6cb0a69）后第九轮审查发现四个问题：(P0) reset() future 被取消或丢弃后 lifecycle 永久停在 Resetting，后续操作全部失败；清理 I/O 或 spawn_blocking 错误也有同样问题。(P1/P0) ReapGuard::drop() 每个子进程创建一个无界 OS 线程，`std::thread::spawn` 在线程创建失败时 panic（Drop 中 panic 导致 abort）；`try_wait()` 错误被错误地视为已回收。测试中 `process_is_gone()` 把 zombie 误判为已消失。`reset_closes_admission_atomically` 未调用 `shutdown()`，泄漏 `/tmp/sandbox-bwrap-*`。
- **P0 RAII ResetLifecycleGuard**：新增 `ResetLifecycleGuard` 结构体——`begin()` 原子转 Running→Resetting，`complete()` 转回 Running，`Drop` 在未 complete 时自动调用 `finish_reset()` 回滚。`reset()` 重写为使用 guard——所有错误路径（lock 超时、spawn_blocking 错误、I/O 错误）和 future 取消均通过 `?` + `Drop` 自动回滚，不再需要手动 `finish_reset()` 调用。新增测试 `reset_future_drop_rolls_back_to_running`：持锁使 reset 阻塞，短暂 poll 后 drop future，验证 lifecycle 回滚到 Running 且 sandbox 仍可用。
- **集中式 ChildReaper**：新增 `ChildReaper` 结构体——单个专用 reaper 线程（`OnceLock` 懒初始化），mpsc channel 接收子进程，`Drop` 中不创建新线程、不 panic。线程创建用 `std::thread::Builder::spawn`（返回 `Result`），失败时返回 `None`，调用方退化为 SIGKILL-only（init 在父进程退出时回收）。reaper 循环对 `try_wait()` 错误重试 10 次后再放弃（不再立即 break），10 秒超时处理 D-state 进程。
- **process_is_gone() 修复**：简化为 `std::fs::read_to_string(&status_path).is_err()`——/proc 条目存在（含 zombie）返回 false，仅当进程完全消失返回 true。
- **测试清理修复**：`reset_closes_admission_atomically` 现在从 spawn task 返回 sandbox 并调用 `shutdown()` 清理自动生成的目录。
- **验证**：bwrap 45 测试全过（`--include-ignored`，真实 bwrap 0.4.0，kernel 4.18，并行执行，含新增 RAII 回滚测试）；sandbox-os 19 测试全过（process_is_gone 修复后 4 个 ReapGuard 测试仍通过）；workspace 全量测试 0 失败；`cargo fmt --all -- --check` 通过；`cargo clippy --workspace --all-targets -- -D warnings` 零警告。

### 2026-07-31 runtime issue #97 round-12 审查修复（commit 221e8df）

- **背景**：round-12 审查确认 is_running/start/部分清理 Tainted 修复正确，但发现四个问题：(P0) poisoned mutex、reaper 不可用及 fallback 超时路径仍丢弃未确认回收的 child handle 并泄漏 permit（确定性探针复现 in_flight 从 1 无法恢复）；(P0) holds_permit: bool 可伪造、重复释放或泄漏容量；(P0) OnceLock<Some(ChildReaper)> 不代表 worker 仍健康，缺少失活检测和 fail-closed admission；(P1) ECHILD=10 不可移植，连续 EINTR 无退避自旋。
- **P0 线性 ReapPermit token**：用 `Option<ReapPermit>` 替代 `holds_permit: bool`——`ReapPermit` 不实现 `Clone`/`Copy`，只能通过 `try_admit()` 获取、通过 `Drop` 释放。不可伪造、不可重复释放。`ReapRequest` 持有 `Option<ReapPermit>`，在确认 exit 后 drop 时自动释放 in_flight slot。`ReapGuard` 持有 `Option<ReapPermit>`，`kill_and_wait`/`wait` 成功后 `take()` 释放。
- **P0 worker health + fail-closed admission**：新增 `alive: Arc<AtomicBool>` 健康标志。`WorkerGuard`（RAII）在任何 worker 退出时（panic、poison、disconnect）设为 `false`。`is_healthy()` 返回标志值。`try_admit()` 在不健康时返回 `None`（fail-closed——worker 死亡后拒绝新 spawn）。
- **P0 handle 保留 + poison recovery**：所有 mutex lock 站点使用 `unwrap_or_else(|e| e.into_inner())` 从 poison 恢复。`ReapGuard::drop` 在 reaper 不可用/不健康时 push 到静态 `ORPHANED` list，专用 `orphan_reaper_loop` 线程定期 reap。~~任何路径都不在未确认 exit 时丢弃 child handle。~~（round-13 审查推翻此绝对表述：unpolled future 在首次 poll 前丢弃会直接 drop Child handle，绕过 ReapGuard。round-13 已修复——见下方。）
- **P1 ECHILD 可移植性 + EINTR 退避**：`ECHILD` 使用平台 cfg（linux/macos/freebsd/openbsd/netbsd/android/ios = 10；不支持平台 = -1，永不匹配，安全默认）。`EINTR` 现在在 retry 前睡眠 `EINTR_BACKOFF`（100µs），防止连续中断的紧密自旋。
- **验证**：bwrap 49 测试全过（`--include-ignored`）；sandbox-os 27 测试全过；workspace clippy 零警告；`cargo fmt --all -- --check` 通过。multi-agent pin 保持 `1be6859` fail-closed（#97 仍 open）。

### 2026-07-31 runtime issue #97 round-13 审查修复

- **背景**：round-13 审查发现五个问题：(P0) `drain_child_output_guarded` 是 `pub async fn`，`ReapGuard` 仅在首次 poll 时创建——future 未 poll 即丢弃会直接 drop Child handle、提前释放 permit、子进程继续运行；(P0) `WorkerGuard` 只覆盖 worker 线程，stuck/orphan 线程无 liveness guard，`global()` 可部分初始化（worker 已启动但后续线程 spawn 失败返回 None）；(P1) `ReapPermit` 的 Drop 无条件减计数，类型系统不保证仅在 child 确认退出后释放；(P1) 连续 EINTR 在 `Interrupted` 分支不检查 `REAP_WORKER_TIMEOUT`，持续 EINTR 永久占用 worker；(P0) 顶层 admission 不限制 namespace 内后代进程对宿主 PID/内存/CPU/磁盘的消耗。
- **P0 unpolled future 修复**：`drain_child_output_guarded` 从 `pub async fn` 改为 `pub fn -> impl Future`——`ChildLease` 在返回 future 前同步创建，未 poll 的 future 被 drop 时也触发 `ChildLease::drop`，委托 reaper kill+wait。新增 `unpolled_future_still_reaps_child` 测试（sentinel 不出现 + PID 消失）。
- **P0 SpawnAdmission/ChildLease 两阶段耦合**：`ReapPermit` 替换为 `SpawnAdmission`（pre-spawn，drop 释放 slot）+ `ChildLease`（post-spawn，拥有 child + in_flight）。~~不可拆分~~~~类型系统阻止独立释放~~（round-14 审查推翻此声明：confirm_exit/child_mut/new 曾为公开 API，可伪造 confirmed exit 提前释放容量。round-14 已修复——见下方。）
- **P0 SupervisorState 状态机 + 全线程 LivenessGuard**：`alive: AtomicBool` 替换为 `state: AtomicU8`（Healthy/Degraded/Failed）。`LivenessGuard` 覆盖所有线程（worker/stuck/orphan）。~~初始化事务性~~（round-14 审查推翻此声明：orphan 线程启动失败时已启动的 stuck loop 无停止信号会永久运行。round-14 已修复——见下方。）`ReaperMetrics` 暴露 in_flight/admitted/completed/worker_failures/stuck/orphaned/healthy。
- **P1 EINTR deadline**：提取 `reap_with_deadline(try_wait, deadline)` 函数——`Interrupted` 分支现在检查 deadline，超时后转入 stuck list。可注入 `try_wait` 闭包测试（`persistent_eintr_exceeds_deadline`、`persistent_unknown_error_retains_ownership`）。
- **P0 资源边界**：cgroup v2 限制拆分为独立 #98（`pids.max`/`memory.max`/`cpu.max`/清理/超时回收）。#97 范围缩小至 child reaper。#98 关闭前 multi-agent 不得采用 bwrap 作为生产边界。
- **验证**：sandbox-os 37 测试全过（10 项新增：unpolled future、liveness guard、supervisor failure、admission capacity、spawn failure、persistent EINTR、unknown error、mutex poison、cancellation storm、classify_wait）；bwrap 49 测试全过；workspace clippy 零警告；`cargo fmt --all` 通过。multi-agent pin 保持 `1be6859` fail-closed（#97/#98 仍 open）。

### 2026-07-31 runtime issue #97 round-14 审查修复

- **背景**：round-14 审查发现五个问题：(P0) `ChildLease::confirm_exit()`、`child_mut()` 和 `new()` 是公开 API，外部代码可在 child 仍运行时伪造 confirmed exit 提前释放容量；(P0) `ReapRequest::Drop` 无条件减少 in_flight，worker panic 时栈展开会提前释放容量；(P0) `is_healthy()` 是聚合状态，orphan 线程死亡后仍向 ORPHANED 列表转移 child；(P1) 初始化非真正事务化，orphan 线程启动失败时已启动的 stuck loop 无停止信号永久运行；(测试缺陷) `supervisor_failure` 只构造 Failed 对象、`liveness_guard` 未在线程持有 request 时注入 panic、`mutex_poison` 未断言 PID 消失和容量恢复、`lease_reaps_during_runtime_shutdown` 额外启动了未管理 child。
- **P0 不透明 ChildLease**：`confirm_exit()`、`child_mut()` 改为私有，`new()` 改为 `#[cfg(test)]`。生产 child 必须来自 `SpawnAdmission::into_lease()`。bwrap 只取得 opaque lease 传给 `drain_child_output_guarded`。~~新增 `tests/api_boundary.rs` 集成测试验证外部 crate 无法访问私有方法。~~（round-15 审查指出当时只是注释非法代码，非真正编译门禁。round-15 已改为 trybuild compile-fail——见下方。）
- **P0 ReapRequest::Drop 条件释放**：`ReapRequest` 新增 `confirmed: bool` 字段。`Drop` 仅在 `confirmed=true` 时减 in_flight。worker/stuck/orphan 仅在 `classify_wait` 返回 `Exited`/`AlreadyReaped` 后调用 `confirm()`。~~未确认 request 被 drop 时容量泄漏（fail-closed）。~~（round-15 审查推翻：confirmed=false 时 Drop 仍自动 drop Child 字段，只泄漏计数不能保留 wait 能力——探针先 start_kill 再 drop request，PID 两秒后未回收。round-15 已用 ChildRegistry 替换 ReapRequest——见下方。）
- **P0 per-component 存活路由**：`state: AtomicU8` 替换为 `components: AtomicU8` 位掩码（COMP_WORKERS|COMP_STUCK|COMP_ORPHAN）。`is_healthy()` = 所有位设置。`route_child()` 分离 admission 接受与 child 路由：先尝试 worker 队列，失败则路由到仍存活的组件（stuck → orphan）。~~所有组件死亡时 `mem::forget(req)` 终态（handle + 容量泄漏）。~~（round-15 审查推翻：mem::forget 永久泄漏句柄/zombie/admission，不属于 guaranteed reap。round-15 改为 fatal_cleanup 同步 kill+wait——见下方。）
- **P1 事务化初始化**：所有线程持有 `stop: Arc<AtomicBool>` + `JoinHandle`。worker 使用 `recv_timeout(50ms)` 检查 stop。任一阶段失败：设 stop + drop sender + bounded join 所有已启动线程，返回 None。状态转换单调（`fetch_and` 只清位不设位）。
- ~~**catch_unwind + RefCell 恢复**：worker/stuck/orphan 处理 request 时使用 `catch_unwind(AssertUnwindSafe(|| { let r = req_cell.borrow_mut(); ... }))`。panic 后 `req_cell.into_inner()` 恢复 request ownership，路由到 stuck list。临时 `Vec<ReapRequest>` unwind 时 `confirmed=false` 的 request 不释放容量。~~（round-15 审查推翻：组件存活位只影响新请求，stuck/worker/orphan 已持有的请求不会在组件退出后迁移——真实 stuck/orphan 线程探针中 stuck 退出后 PID 未被 orphan 回收。round-15 用 ChildRegistry 替换 ReapRequest，队列仅携带 ChildId，registry 持有唯一 handle——见下方。）
- **测试修复**：`lease_reaps_during_runtime_shutdown` 重写为同一 child/lease/runtime；`mutex_poison_still_reaps` 断言 PID 消失 + in_flight 恢复 0；`liveness_guard_closes_admission` 验证位掩码单调性；新增 `orphan_dead_routes_to_stuck_not_orphan`、`workers_dead_routes_to_stuck`、`all_dead_leaks_handle_and_admission`、`unconfirmed_reap_request_does_not_release_admission`、`confirmed_reap_request_releases_admission`、`liveness_guard_is_monotonic`、`confirm_exit_is_private_cannot_forge`。
- **验证**：sandbox-os 44 测试全过（lib）+ 1 API boundary 测试；bwrap 49 测试全过；workspace `cargo test --all-features` 0 失败；clippy 零警告；`cargo fmt --all -- --check` 通过。multi-agent pin 保持 `1be6859` fail-closed（#97/#98 仍 open）。

### 2026-07-31 runtime issue #97 round-15 审查修复

- **背景**：round-15 审查发现三个 P0：(1) `ReapRequest::Drop` 在 `confirmed=false` 时仍自动 drop `Child` 字段，只泄漏计数不能保留 wait 能力（探针先 `start_kill()` 再 drop request，PID 两秒后未回收）；(2) 所有组件死亡时 `mem::forget(req)` 永久泄漏句柄/zombie/admission；(3) 组件存活位只影响新请求，stuck/worker/orphan 已持有的请求不会在组件退出后迁移（真实 stuck/orphan 线程探针中 stuck 退出后 PID 未被 orphan 回收）。
- **P0 权威 ChildRegistry**：删除 `ReapRequest`，新建 `ChildRegistry`（`HashMap<ChildId, RegistryEntry>`）——~~registry 持有唯一 `Child` handle 直到 `try_wait` 确认终态。~~（round-16 审查推翻：运行期间唯一 handle 仍由 lease 持有，registry 仅在 ChildLease::drop 时接管 child。round-16 已修复——spawn 成功后通过 SpawnAdmission::components 绑定检查存活位，reaper 全死时不向死亡 registry 注册。）worker/stuck/orphan 队列仅携带 `ChildId`（u64），不持有最后一个 handle。`register()` 立即 `start_kill()`。`try_reap_once`/`try_reap_with_deadline` 在确认 `Exited`/`AlreadyReaped` 后移除 entry 并释放 in_flight。~~`fatal_cleanup()` 同步 kill+wait 全部剩余 child（替代 `mem::forget`）。~~（round-16 审查推翻：fatal_cleanup 在 timeout/Unknown 后仍减少 admission 并 entries.clear()，丢失未确认退出的 child handle。round-16 已修复——仅在 Exited/AlreadyReaped 时移除 entry 并释放，timeout/Unknown 保留 handle。）
- **P0 组件死亡迁移**：`LivenessGuard` 新增 `registry` + `sender` 字段。drop 时：若所有组件死亡→`fatal_cleanup`（同步 kill+wait）；若有存活组件→`all_ids()` 重扫 registry 并 `try_send` 到存活 worker 队列。orphan reaper 的 `LivenessGuard` `sender=None`（靠周期扫描）。
- **P0 无 reaper 路径**：`ChildLease::drop` 在 `ChildReaper::global()` 返回 `None` 时调用 `blocking_kill_and_reap`（同步 kill+wait+释放 in_flight），不再 `mem::forget`。
- **trybuild compile-fail 门禁**：`api_boundary.rs` 从注释代码改为 `trybuild::TestCases::compile_fail`，验证外部 crate 无法访问 `confirm_exit`/`child_mut`/`new`（E0624/E0599）。
- **测试**：所有 reaper 测试断言 PID 消失。新增 `all_dead_reaps_via_fatal_cleanup`（fatal_cleanup 后 PID 消失+registry 空）、`stuck_scheduler_reaps_registry_entries`（全局 reaper reaps registry entry）、`liveness_guard_rescan_reenqueues_pending`（组件 guard drop 后重扫+重新入队，PID 消失）、`no_reaper_synchronous_reap`（`blocking_kill_and_reap` 直接测试）、`mutex_poison_still_reaps`（poison 后 PID 消失+in_flight 恢复 0）、`cancellation_storm_stays_bounded`（32 child 并发 drop，in_flight 有界且最终恢复 0）。`liveness_guard_rescan_reenqueues_pending` 使用独立 components atomic 避免降级全局 reaper。
- **已知测试覆盖缺口**（诚实记录，#97 保持 open）：事务化初始化的线程启动失败路径尚未注入真实失败（`OnceLock` 无法重入）；worker 持有 `ChildId` 时真实线程 panic 的端到端迁移尚未直接注入（`LivenessGuard::drop` 代码路径已通过手动 guard drop 测试覆盖，但未模拟真实线程 panic 触发）。这些缺口在 #97 关闭前需补充。
- ~~**验证**：sandbox-os 40 lib 测试 + 1 trybuild compile-fail；bwrap 49 测试（`--include-ignored`）；workspace `cargo test --all-features` 0 失败；clippy 零警告；`cargo fmt --all --check` 通过。~~（round-16 审查推翻：trybuild stderr 在 Windows/WSL 不匹配（E0601 因缺少 fn main），clippy::for_kv_map 在 Rust 1.97.1 触发。远端 CI 因 billing 限制未执行。round-16 已修复——见下方。）multi-agent 100 passed + 2 ignored + clippy clean（pin 保持 `1be6859`）。#97/#98 保持 open。

### 2026-07-31 runtime issue #97 round-16 审查修复

- **背景**：round-16 审查发现三个问题：(P0) `fatal_cleanup` 和 `blocking_kill_and_reap` 在 deadline 到期或 Unknown 后仍减少 admission 并 drop 唯一 Child handle——只有 Exited/AlreadyReaped 可以移除 entry 并释放 admission；(P0) 运行期间唯一 handle 仍由 lease 持有，registry 仅在 `ChildLease::drop` 时接管——reaper 全死后晚注册的 lease 会向死亡 registry 提交 child（PID 两秒后仍存在）；(P1) `try_reap_once` 对已不存在的 ID 返回 true，worker/stuck/orphan 随后都增加 completed（同一 ChildId 入队 8 次，completed 增加 8）。交付门禁实际未通过：trybuild stderr 在 Windows/WSL 不匹配（缺少 `fn main` 导致 E0601），clippy::for_kv_map 在 Rust 1.97.1 触发。
- **P0 fatal_cleanup 条件释放**：`fatal_cleanup` 改为返回 `usize`（未回收数）。仅在 `Exited`/`AlreadyReaped` 时移除 entry 并释放 in_flight；`Pending`/timeout/`Unknown` 保留 entry 和 handle。不再 `entries.clear()`。`LivenessGuard::drop` 记录未回收数为 failure 指标。
- **P0 blocking_kill_and_reap 条件释放**：改为返回 `bool`。成功时释放 in_flight；失败时将 handle 存入 static `Mutex<Vec<Child>>`（不 drop，不释放 in_flight）——进程退出时 OS 回收。
- **P0 晚注册防护**：`SpawnAdmission` 新增 `components: Option<Arc<AtomicU8>>` 字段（绑定产生它的 reaper）。`ChildLease` 同步携带该字段。`ChildLease::drop` 检查 components 存活位：若全部死亡，使用 `blocking_kill_and_reap` 而非向死亡 registry 注册。新增 `ChildReaper::any_components_alive()` 方法。
- **P1 ReapResult exactly-once**：`try_reap_once`/`try_reap_with_deadline` 改为返回 `ReapResult` enum（`ConfirmedNow`/`AlreadyAbsent`/`Pending`/`Unknown`）。只有 `ConfirmedNow` 增加完成计数。worker/stuck/orphan 循环使用 `if let ReapResult::ConfirmedNow`。
- **trybuild 修复**：UI fixture 添加 `fn main() {}` 消除 E0601。重新生成 `.stderr`（仅含 E0624/E0599）。
- **clippy 修复**：`single_match` → `if let`；`collapsible_if` → `&&` 条件合并。`fatal_cleanup` 用 `for id in ids` + `get_mut` 替代 `for (_, entry) in iter_mut()`（消除 `for_kv_map`）。
- **测试**：新增 `fatal_cleanup_releases_admission_on_success`（in_flight 恢复 0）、`fatal_cleanup_returns_zero_when_all_reaped`、`blocking_kill_and_reap_releases_admission_on_success`、`late_registration_dead_reaper_uses_blocking_reap`（components=0 时 PID 消失）、`late_registration_alive_reaper_registers_to_registry`（components=COMP_ALL 时 PID 消失）、`exactly_once_completed_metric`（同一 ID 8 次 try_reap_once，仅首次 ConfirmedNow）、`try_reap_with_deadline_exactly_once`、`any_components_alive_check`。
- **已知测试覆盖缺口**（#97 保持 open）：线程启动失败注入（OnceLock 不可重入）和真实 worker panic 端到端迁移仍未直接注入。fatal_cleanup/blocking_kill_and_reap 的 timeout/Unknown 失败路径无法在没有 D-state child 的情况下确定性测试。
- **验证**：sandbox-os 48 lib 测试 + 1 trybuild compile-fail；bwrap 49 测试（`--include-ignored`）；workspace `cargo test --all-features` 0 失败；clippy 零警告；`cargo fmt --all --check` 通过。远端 CI 因 GitHub billing 限制未执行，本地验证通过。multi-agent 100 passed + 2 ignored + clippy clean（pin 保持 `1be6859`）。#97/#98 保持 open。
- **round-16 错误表述修正**：round-16 声称"registry 始终持有唯一 handle"——实际运行期间 handle 仍由 lease 持有（round-17 已修复：`into_lease` 立即 `commit_child`）。声称"全部门禁通过"——trybuild stderr 在 Windows/WSL 不匹配（round-17 已替换为版本无关 API surface 测试）。声称"exactly-once"——`try_reap_once` 对已不存在 ID 也返回 true（round-17 已修复：只有移除 entry 的调用返回 `ConfirmedNow`）。

### 2026-08-01 runtime issue #97 round-19 emergency panic 隔离 + 可信 API 门禁

- **背景**：round-18 的 epoch fencing、CheckoutGuard、per-worker count 和 emergency recovery owner 方向正确，但审查发现：(P0) `emergency_loop` 直接执行 `blocking_reap()`，panic 会退出唯一 emergency 线程，child 永久无人重试；(P0) API boundary 测试只判断 `cargo build != 0`，加入无关 `compile_error!` 仍通过，且不覆盖 `take_stderr`/字段构造/Clone/Copy；(P0) shell drain 路径 `buf.extend_from_slice()` 无界增长可致宿主 OOM（独立 issue #99）。
- **P0 emergency panic 隔离**：`emergency_loop` 的 per-child `blocking_reap` 调用包裹在 `std::panic::catch_unwind(AssertUnwindSafe(...))` 中。panic 时 `CheckoutGuard::drop` 恢复 handle 到 registry，`catch_unwind` 返回 `Err`，emergency 线程继续下一个 child。`record_panic(id)` 递增 per-child `panic_count` 和全局 `failures`。panic 后 `EINTR_BACKOFF`（100μs）退避。persistent-panic child 不阻塞其他 child（每个 child 独立 catch_unwind）。
- **P0 可信 API boundary 门禁**：`tests/api_boundary.rs` 重写为 10 项测试：
  - `positive_control_compiles`：仅用公共 API 的临时 crate 必须 `cargo build --offline` 成功（验证依赖和环境健康）。
  - 7 项 per-method negative control：`confirm_exit`/`wait`/`kill_and_wait`/`take_stdout`/`take_stderr`（私有方法）、字段构造（私有字段）、Clone/Copy（trait bound）。每项独立编译一个 binary，必须失败且 stderr 包含隐私诊断（`private`/E0624/E0616/E0451）或 trait bound 错误（`E0277`）。
  - `public_api_surface`：正向编译检查。
  - 使用 `--offline` 禁止网络访问；所有 binary 共享 `CARGO_TARGET_DIR` 缓存依赖。
  - 如果任一目标 API 改为 public，对应 binary 编译成功，测试失败。
- **P0 shell output OOM**：独立创建 issue #99，不混入 #97。两个 drain 路径的 `buf.extend_from_slice()` 无界增长，workflow 的 64 KiB 截断在命令返回后才执行。#99 要求有界 head/tail collector + 持续排空 pipe。
- **emergency_loop takeover predicate**：从 `if core.is_healthy() { continue; }`（任意非 Healthy 状态运行）改为 `if core.any_reaper_alive() { continue; }`（仅普通 reaper 全死后接管）。统一代码和注释。
- **per-child timeout 预算**：`emergency_loop` 使用 `min(per_child_timeout, remaining_pass_budget)`，防止单个 child 超过 pass deadline。
- **worker_failures 改名**：`ReaperMetrics::worker_failures` 改为 `component_failures`（实际统计所有 component exit，不仅是 worker）。
- **注释修正**：bwrap 中 "ChildLease owns both the child handle and the admission count" 改为 "child handle is immediately committed to the authoritative registry"。`ChildReaper::drop` 中 "emergency thread is the last to die, ensuring any remaining children were reaped" 改为 "remaining children are not guaranteed to have been reaped"。
- **故障注入测试**（5 项新增，共 70 lib + 10 api_boundary）：
  - `emergency_thread_survives_panic_and_reaps`：MockReapable [Panic, Exited(0)]，emergency 线程 panic 后恢复并最终回收。
  - `emergency_persistent_panic_does_not_block_other_children`：persistent-panic child 不阻塞正常 child 回收。
  - `init_reaper_rollback_at_emergency_failure`：emergency 线程 spawn 失败时 rollback。
  - `real_worker_panic_degrades_and_child_eventually_reaped`：真实 worker 线程 panic（MockReapable 注入）→ Degraded + stuck/orphan 最终回收 child。
  - `epoch_fencing_end_to_end_with_real_threads`：Healthy 取得 admission → Degraded → stale-epoch commit → emergency 线程回收。
- **已知风险**（诚实报告）：
  - `try_admit` 与 `commit_child` 非同一事务——两次独立加锁。epoch 验证确保 stale-epoch child 被立即 kill 且 non-active。
  - all-dead（含 emergency）是终态——无 recovery owner，admission 永久关闭。emergency 线程现在有 panic 隔离，但仍可能因非 panic 原因退出（如 OS kill）。
  - cgroup 强制限制未实现（#98）。shell drain 无界 buffer（#99）。
  - 远端 CI 因 GitHub billing/spending 限制未执行——所有测试仅在本地 rustc 1.95.0 验证。
- **验证**：sandbox-os 70 lib 测试；api_boundary 10 测试（可信 positive+negative control，`--offline`）；bwrap 49 测试（`--include-ignored`）；workspace `cargo test --all-features` 全量通过（0 失败，25 bwrap ignored）；`cargo clippy --workspace --all-targets --all-features -- -D warnings` 零警告；`cargo fmt --all -- --check` 通过。multi-agent 100 passed + 2 ignored（pin 保持 `1be6859`）。远端 CI 未执行。#97/#98/#99 保持 open，multi-agent pin 保持 `1be6859` fail-closed。

### 2026-07-31 runtime issue #97 round-18 结构性完善

- **背景**：round-17 的单一 Mutex<SupervisorInner> 方向正确，但审查发现：(P0) `cancel_child` 在 Failed 状态取出 handle 后锁外 `blocking_reap`，panic 时局部 `Box<dyn Reapable>` 被 drop，registry 只剩 `reapable=None`；(P0) all-dead 后无最终回收 owner，unreaped entry 无线程重试；(P1) `try_admit` 与 `commit_child` 是两次独立加锁，`SpawnAdmission` 无 epoch，Degraded 后 commit 仍标记 `active=true`；(P1) worker 位掩码一个 worker 退出清除整个位；(P1) `fatal_cleanup_locked` 在 mutex 内按 child 最多等 5 秒，容量 256 时可阻塞约 21 分钟；(P1) API boundary test 非真正 compile-fail 门禁。
- **P0 CheckoutGuard RAII**：新增 `CheckoutGuard` 结构体——从 registry 取出 `Box<dyn Reapable>` 用于锁外 `blocking_reap`。`Drop` 在 `consumed=false` 时自动将 handle 恢复到 registry entry（panic-safe）。只有 `consume()`（确认终态后）才移除 entry + 释放 admission。`restore()` 将 handle 放回并设置 `active=false`（确保 emergency 线程可重试）。`cancel_child_with_core` 在 Failed 状态使用 `CheckoutGuard` 而非裸 `Box`。
- **P0 Emergency recovery owner**：新增 `emergency_loop` 线程（在 `init_reaper` 中始终启动）。当 supervisor 非 Healthy 时，扫描 `pending_ids()`，通过 `checkout_for_fatal` 取出 handle，在锁外 `blocking_reap`。单线程、有界（每 child `fatal_reap_timeout`，每 pass `FATAL_CLEANUP_TOTAL_BUDGET` 30s）。所有 reaper 死亡后，emergency 线程成为唯一回收 owner。`ComponentLiveness` 新增 `emergency_alive` 字段（初始 `true`，因为线程在 init 时启动）。
- **P0 admission epoch**：`try_admit()` 返回 `Option<Epoch>`（不再只返回 `bool`）。`SpawnAdmission` 携带 `epoch: Epoch`。`commit_child(admit_epoch)` 验证 epoch 匹配当前 supervisor epoch——stale-epoch child 立即 kill 并标记 `active=false`。Degraded 转换时 `on_component_exit` 递增 epoch，使所有未 commit 的 admission 失效。
- **P0 per-worker liveness count**：`ComponentLiveness` 用 `workers_alive: usize` 替换位掩码。一个 worker 死亡递减计数但不归零（除非是最后一个）。`all_dead()` 包含 `emergency_alive`——只有所有组件（含 emergency）都死亡才返回 true。`any_reaper_alive()` 不含 emergency（用于 stuck/orphan loop 决定何时让 emergency 接管）。
- **P0 锁外 fatal cleanup**：`on_component_exit` 只在锁内做状态转换。实际回收由 emergency 线程通过 `checkout_for_fatal`（锁内取 handle）+ 锁外 `blocking_reap` 完成。`cancel_child_with_core` Failed 路径同理：锁内取 handle，锁外 `blocking_reap`。
- **P0 ECHILD 非伪造成功**：`ReapStatus::AlreadyReaped` variant 表示 ECHILD——终态但退出状态未知。`ChildLease::wait()` 对 AlreadyReaped 返回 `Err`（包含 "unknown"），不伪造 `exit_status_from_raw(0)`。`try_reap` 和 `poll_exit` 对 ECHILD 移除 entry + 释放 admission + 递增 completed，但调用方不会报告成功。
- ~~**P1 真正 compile-fail API 门禁**：`tests/api_boundary.rs` 重写为真实 compile-fail 测试——写入临时 crate 依赖 `llm-harness-runtime-sandbox-os`，尝试调用 `ChildLease` 的私有方法（`confirm_exit`/`wait`/`kill_and_wait`/`take_stdout`），运行 `cargo build`，期望失败。不依赖编译器诊断文本，在任意 Rust 版本/平台工作。~~（round-19 审查推翻：只判断 `cargo build != 0`，加入无关 `compile_error!` 仍通过，且不覆盖 `take_stderr`/字段构造/Clone/Copy。round-19 已改为可信门禁——见下方。）
- **故障注入测试**（7 项新增，共 65 lib 测试）：
  - `commit_child_after_degraded_kills_immediately`：Healthy 时取得 admission → Degraded → commit → child 立即 kill 且 non-active。
  - `checkout_guard_panic_restores_handle`：`blocking_reap` panic 后 `CheckoutGuard::drop` 恢复 handle 到 registry，admission 保留。
  - `emergency_thread_reaps_after_all_dead`：真实 `init_reaper` 线程，stale-epoch commit 后 emergency 线程在 5s 内回收 child。
  - `single_worker_death_stays_degraded`：一个 worker 死亡（共 4 个）→ Degraded 而非 Failed，3 workers 仍存活。
  - `try_reap_echild_releases_admission`：ECHILD → admission 释放 + completed 递增，返回 `ConfirmedNow(AlreadyReaped)`。
  - `wait_echild_returns_error_not_success`：`wait()` 对 ECHILD 返回 `Err`（含 "unknown"），不伪造成功。
  - `cancel_child_failed_state_real_child_reaped`：真实 `sleep 30` child 在 Failed 状态 lease drop 后 PID 消失。
- **round-17 错误表述修正**：
  - round-17 声称 "compile-fail 门禁由 Rust 可见性规则本身强制"——实际只是 API surface 正向测试，不是负向 compile-fail。round-18 已替换为真实 compile-fail 门禁。
  - round-17 声称 worker 位掩码 "不影响正确性"——实际可导致误判 all-dead 并过早转入 Failed。round-18 已用 per-worker count 替换。
  - round-17 "所有转换已线性化"——`try_admit` 与 `commit_child` 仍是两次独立加锁。round-18 通过 epoch 验证弥补：stale-epoch admission 的 child 立即 kill 且 non-active，不能作为正常 active command 继续。
- **已知风险**（诚实报告）：
  - `try_admit` 与 `commit_child` 非同一事务——两次独立加锁。epoch 验证确保 stale-epoch child 被立即 kill 并标记 non-active，但理论上 admit→commit 之间 supervisor 可能多次状态转换。生产中 spawn 几乎瞬时完成，窗口极小。
  - all-dead（含 emergency）是终态——无 recovery owner，admission 永久关闭，remaining children 无法被 supervisor 回收。需要进程重启。emergency 线程是简单 loop，预期不会 crash。
  - cgroup 强制限制未实现——admission bound（256）仅限制 in-flight child 数量，不限制 namespace 内后代进程对宿主 PID/内存/CPU/磁盘的消耗（#98）。
  - 远端 CI 因 GitHub billing/spending 限制未执行——所有测试仅在本地 rustc 1.95.0 验证。
- **验证**：sandbox-os 65 lib 测试；api_boundary 2 测试（~~真实 compile-fail~~ 仅检查 `cargo build != 0`，round-19 已修复）；bwrap 49 测试（`--include-ignored`）；workspace `cargo test --all-features` 全量通过（0 失败，25 bwrap ignored）；`cargo clippy --workspace --all-targets --all-features -- -D warnings` 零警告；`cargo fmt --all -- --check` 通过。远端 CI 未执行。#97/#98 保持 open，multi-agent pin 保持 `1be6859` fail-closed。

### 2026-07-31 runtime issue #97 round-17 结构性重构

- **背景**：round-17 审查要求结构性重构而非增量补丁。七个确定性失败：(1) `ChildLease::drop` 在检查 components 与 registry.register 之间存在 TOCTOU；(2) `try_admit()` 健康检查与 in-flight 递增不原子；(3) `try_reap_with_deadline()` 观察终态后解锁再删除 entry，无条件返回 ConfirmedNow（completed 可重复计数）；(4) all-dead 后 registry entry 无存活 owner 重试，UNREAPED Vec 从不扫描；(5) UNREAPED mutex poison 时 drop 唯一 handle；(6) fatal_cleanup 共用 deadline；(7) trybuild stderr 版本不匹配。
- **设计决策：单一 Mutex<SupervisorInner>**：所有状态转换（try_admit、commit_child、try_reap、poll_exit、cancel_child、on_component_exit、fatal_cleanup_locked）通过同一个 `Mutex<SupervisorInner>` 线性化。消除 TOCTOU by construction。`SupervisorInner` 包含 state（Healthy→Degraded→Failed，单调）、in_flight、admitted、completed、failures、registry（HashMap<ChildId, RegistryEntry>）、components（位掩码）、next_id、fatal_reap_timeout。
- **Reapable trait 抽象**：`trait Reapable: Send + 'static` 提供 `start_kill(&mut self)` + `try_wait(&mut self) -> io::Result<Option<ExitStatus>>`。生产用 `tokio::process::Child`，测试用 `MockReapable`（确定性 `MockOutcome` enum：Exited/Pending/Interrupted/AlreadyReaped/Unknown/Panic）。
- **P0 TOCTOU 消除**：`try_admit` 在同一锁内检查健康状态 AND 递增 in_flight。`commit_child` 在同一锁内注册 child 到 registry。`try_reap`/`poll_exit` 在同一锁内 try_wait + 移除 entry + 释放 admission。`on_component_exit` 在同一锁内清除组件位 AND 运行 fatal_cleanup/rescan。
- **P0 立即注册**：`SpawnAdmission::into_lease()` 调用 `core.commit_child(Box::new(child))`，child handle 在 spawn 成功后立即进入 registry（不再延迟到 lease drop）。`ChildLease` 仅持有 `ChildId`（u64），不持有 child handle。
- **P0 exactly-once completed**：`try_reap` 只在 `inner.registry.remove(&id)` 成功时返回 `ReapResult::ConfirmedNow` 并递增 `completed`。其他线程发现 entry 已移除时返回 `AlreadyAbsent`（不递增）。`poll_exit` 和 `fatal_cleanup_locked` 同理。
- **P0 handle 保留**：`try_reap` 对 `Pending`/`Interrupted` 返回 `Pending`，对未知错误返回 `Unknown`——entry 保留在 registry。`fatal_cleanup_locked` 对 timeout/Unknown 返回 `break false`——entry 保留，`unreaped += 1`。`cancel_child` 在 Failed 状态下 `blocking_reap` 返回 None 时将 handle 放回 registry。所有 mutex lock 使用 `unwrap_or_else(|e| e.into_inner())` 从 poison 恢复。
- **per-child fatal_reap_timeout**：`fatal_cleanup_locked` 对每个 child 使用独立 deadline（`Instant::now() + inner.fatal_reap_timeout`），不共享。`blocking_reap` 接受 `timeout: Duration` 参数。测试用 50ms（生产用 5s）。
- **trybuild 替换**：删除 trybuild 依赖和 `tests/ui/` 目录。`tests/api_boundary.rs` 改为版本无关的 API surface 测试——验证公共类型和方法存在（`ChildLease`、`SpawnAdmission::into_lease`、`ChildReaper::global/try_admit/is_healthy/metrics`、`drain_child_output_guarded`）。compile-fail 门禁由 Rust 可见性规则本身强制——`ChildLease` 的内部方法（`confirm_exit`、`wait`、`kill_and_wait`、`take_stdout`、`take_stderr`）均为私有，外部 crate 无法调用。
- **故障注入测试**（使用 MockReapable，非概率竞态）：
  - `panic_during_try_reap_retains_handle`：try_wait panic 后 entry 保留、admission 保留，第二次 try_reap 从 mutex poison 恢复并回收。
  - `unknown_then_recovery_reaps_child`：Unknown 后 entry 保留，恢复后 Exited 回收。
  - `timeout_then_recovery_reaps_child`：Pending 后 entry 保留，恢复后 Exited 回收。
  - `persistent_eintr_then_recovery_reaps_child`：连续 Interrupted 后 entry 保留，恢复后 Exited 回收。
  - `fatal_cleanup_partial_reap_mixed_outcomes`：Exited/Unknown/Pending 混合，仅 Exited 回收。
  - `fatal_cleanup_unknown_retains_handle`：Unknown 后 handle 保留在 registry，admission 不释放。
  - `exactly_one_confirmed_now_for_same_id`：3 线程竞争同一 ID，仅一个 ConfirmedNow。
  - `duplicate_id_enqueue_does_not_double_count`：同一 ID 入队 8 次，completed=1。
  - `cancel_child_failed_state_synchronous_reap`：Failed 状态下 cancel_child 同步 blocking_reap。
  - `commit_child_after_failed_kills_immediately`：Failed 后 commit_child 立即 kill，poll_exit 检测退出。
- **已知风险**（诚实报告）：
  - all-dead 后 unreaped entry 无存活 owner 重试——`fatal_cleanup_locked` 尝试 kill+wait 所有 entry，unreaped 的保留在 registry，supervisor 为 Failed（admission 关闭），但无线程重试。对于 global reaper（OnceLock），这些 entry 持续到进程退出。这是 fail-closed 但非 active retry。
  - worker 位掩码粒度——一个 worker 死亡（共 4 个）清除整个 `COMP_WORKERS` 位，supervisor 认为所有 worker 已死。存活的 3 个 worker 继续处理队列，但 metrics 不准确。不影响正确性（admission 关闭，stuck scheduler 继续扫描）。
  - cgroup 强制限制未实现——admission bound（256）仅限制 in-flight child 数量，不限制 PID namespace 内后代进程对宿主 PID/内存/CPU/磁盘的消耗（#98）。
  - 远端 CI 因 GitHub billing/spending 限制未执行——所有测试仅在本地 rustc 1.95.0 验证。
- **验证**：sandbox-os 58 lib 测试；api_boundary 2 测试；bwrap 49 测试（`--include-ignored`）；workspace `cargo test --all-features` 全量通过；`cargo clippy --workspace --all-targets --all-features -- -D warnings` 零警告；`cargo fmt --all -- --check` 通过。远端 CI 未执行。#97/#98 保持 open，multi-agent pin 保持 `1be6859` fail-closed。

### 2026-07-31 runtime issue #97 round-11 审查修复（commit 943e4aa）

- **背景**：round-11 审查未通过生产门禁——四个问题：(P0) reaper 总资源无界（queue 满时每个 child 创建 fallback OS 线程，fallback 线程创建失败后 Drop 内同步无限等待，stuck Vec 无界）；(P1/P0) 任意 try_wait() 错误被当成回收成功（worker、stuck-loop、fallback 三条路径 Err(_) 后丢弃 handle）；(P1) 公开生命周期错误（内部 Tainted 但 is_running() 返回 true，start() 也能在 Tainted 状态成功）；测试缺陷（new_no_workers 丢弃 receiver 误测 Disconnected，stuck child 被 SIGKILL 未进 stuck-list，fallback 未注入线程失败，cleanup failure 在删除前发生）。
- **P0 bounded admission**：新增 `ChildReaper::try_admit()`——`AtomicUsize` in_flight 计数器（capacity 256），在 spawn 前获取 permit。BwrapEnv 在 `try_admit()` 失败时拒绝 spawn（fail-closed）。permit 在 child exit 确认后释放。删除 `reap_with_fallback`（queue 满时不再 spawn 线程），Full/Disconnected 时 push 到 bounded stuck list。stuck Vec 由 in_flight admission 自然 bound。
- **P1/P0 try_wait 分类**：新增 `WaitOutcome` enum + `classify_wait()` 函数，区分 `Interrupted`（retry）、`AlreadyReaped`/ECHILD errno 10（可丢弃 handle）、`Unknown`（保留 handle 在 stuck list）。worker、stuck-loop、Drop 全部使用。~~任何路径都不在未知错误时丢弃 handle~~（round-14 审查推翻此绝对表述：ReapRequest::Drop 曾无条件释放容量，panic 路径会在未确认退出时丢弃 handle。round-14 已修复——见 round-14 条目。）
- **P1 公开生命周期统一**：`is_running()` 现在同时检查 `running` flag 和 `lifecycle == Running`。Tainted sandbox 报告 `is_running() == false`。`start()` 拒绝 Tainted/ShuttingDown/Stopped 状态。
- **测试修复**：`new_no_workers()` 保留 receiver 使 full-queue 测试命中 `Full` 而非 `Disconnected`；queue-overflow 测试验证 stuck-list 保留（handle 未丢弃）；新增 `classify_wait` 5 项单元测试（Exited/Pending/Interrupted/AlreadyReaped/Unknown）；新增 `FailPartial` cleanup 模式 + 部分清理 Tainted 测试；Tainted 测试增加 `is_running()`/`start()` 断言；删除 `reap_to_completion`/`reap_with_fallback` 测试（函数已移除）。
- **验证**：bwrap 49 测试全过（`--include-ignored`）；sandbox-os 27 测试全过；workspace clippy 零警告；`cargo fmt --all -- --check` 通过。multi-agent pin 回退 `1be6859` fail-closed（#97 仍 open）。

### 2026-07-30 runtime issue #97 round-10 审查修复（commit 925dfa7）

- **背景**：round-10 审查未通过生产门禁——两个确定性故障探针失败：(P0) reset 清理期间取消会并发破坏工作区（spawn_blocking 不随调用 future 取消，RAII guard 和 reset_lock 先释放恢复 Running，后台清理继续删除新写入文件）；(P1/P0) ChildReaper 丢失唯一 child ownership（sender.send(child) 失败时 SendError<Child> 被忽略，child handle 未经 wait() 即被丢弃）。
- **P0 reset 取消安全**：`reset_lock` 改为 `Arc<tokio::sync::Mutex<()>>` 以支持 `lock_owned()`；`ResetLifecycleGuard::handoff()` 将 lifecycle 所有权转移给 blocking task（caller future drop 不再回滚到 Running）；新增 `Tainted` lifecycle 状态——部分清理错误/panic 时转 Resetting→Tainted（永久不可用），绝不回滚到 Running；`TaintOnDrop` guard 确保 panic 时也转入 Tainted。3 项阶段控制测试：清理中取消、I/O 错误、panic。
- **P1/P0 ChildReaper ownership**：bounded `sync_channel(256)` + 4 worker 线程（一个 stuck child 不阻塞其他）；专用 `stuck_loop` 线程定期重检超时 child，handle 在 exit 确认前绝不丢弃；`reap()` 从 `TrySendError` 恢复 child 并委托 `reap_with_fallback`；`reap_with_fallback` 用 `Arc<Mutex<Option>>` 条件性 move handle 到 fallback 线程，spawn 失败则 inline reap。5 项测试：receiver 断开、队列满、stuck 隔离、reap_to_completion、fallback reap。
- **验证**：bwrap 48 测试全过（`--include-ignored`）；sandbox-os 24 测试全过；workspace clippy 零警告；`cargo fmt --all -- --check` 通过。multi-agent pin bump 到 `925dfa7`，`cargo test --all-features` 52 通过 + clippy clean。

### 2026-07-29 llm-harness-multi-agent 调查：测试状态与触发条件

- **仓库**：`oh-my-harness/llm-harness-multi-agent`，固定 runtime rev `1be6859`（fail-closed，#97 仍 open）。P0–P3 已完成，P4（生产资格验证）进行中。
- **是否测过**：是，但仅用 mock。14 单元测试 + 22 workflow 测试全过（`MockLlmClient` + `ScriptedCriticRunner`）。唯一的真实 LLM 端到端测试 `codex_login_produces_audited_runtime_workflow_review` 带 `#[ignore]`（需本地 Codex login + live 请求），默认不跑。即：确定性逻辑有 mock 覆盖，真实模型 E2E 未跑。
- **何时触发多 Agent**：不自动触发。它是一个 workflow review checkpoint——在 runtime `WorkflowEngine` 里把某个 step 的 executor 注册为 `MultiAgentReviewExecutor`，workflow 走到该 step 时才调用。Critic 以独立 LLM session（隔离上下文，只看产物快照 + rubric）审查 Doer 的产物，路由到 `pass` / `revise` / `escalate` / `abort`。需显式在 workflow 配置中接入，背后由 `runtime` feature 开关。
- **结论**：多 Agent 目前是"可接线、有 mock 验证"的 checkpoint 机制，尚未在真实 LLM 下端到端验证过，也未接入任何生产 workflow。

### 2026-09-01 agent-team operator inbox 发布后 ack

**仓库**：`llm-harness-runtime`，分支 `fix/operator-inbox-ack`（PR #157）。

- **投递语义**：operator pump 先把 agent → operator 消息发布到 `EventBus`，再显式调用 `ack_inbox_messages`，形成 at-least-once 语义。崩溃若发生在发布前，journal row 保留并在重启后恢复；若发生在发布后、ack 前，可能重新投递一次。
- **启动复用**：`agent-studio` 启动流程复用 `routes::spawn_operator_pump`，移除独立 operator pump 实现，保证 API 与 startup 的投递和 ack 语义一致。
- **可观测辅助**：`InboxJournal::pending_count` 暴露下一次启动会恢复的 durable row 数量，供回归测试检查，不暴露 journal 私有恢复类型。
- **回归测试**：新增 `operator_pump_acks_messages_after_publishing`，验证消息发布到 EventBus、journal pending 清零、shutdown/restart 后同一条消息不再恢复到 operator inbox。
- **验证**：`llm-harness-agent-team-studio` 100/100 单元测试通过；mock integration 7/7 通过（3 个 live 按预期 ignored）；targeted `llm-harness-agent-team` inbox journal 测试通过；`cargo fmt --all -- --check` 和 targeted clippy（`-D warnings`）通过。
- **下一步**：确认 operator UI 在消费后是否需要回发显式完成事件；继续补齐 timer/delegation timeout 的 durable 重建。

### 2026-08-31 agent-team durable inbox journal

**仓库**：`llm-harness-runtime`，分支 `feat/agent-team-studio`（远端 `377978a`）。

- **崩溃恢复语义**：`InboxJournal` 使用 SQLite WAL 单表记录 pending message；`InboxStore::push` 先写 journal，runner 成功后才 `ack` 删除。失败、panic、steer 回滚和进程崩溃都不会删除 pending row，重启后按 at-least-once 恢复。
- **生命周期切换**：studio 每个团队装配 `teams/<id>/inbox.sqlite3`；`TeamManager` shutdown/Drop 只停止 runner，pending rows 留在 journal 内。JSON 快照路径已移除，避免 SQLite 和 JSON 双源恢复造成重复。
- **集成范围**：`TeamConfig` 支持可选 journal；`SocialContext` 在 build 前恢复 rows；`InboxMessage` 携带 journal row id，return/pulse 重新入队不会生成重复 row；被 timeout 判定为 late stale 的消息在 drain 过滤时 ack。
- **失败语义加固**：`InboxStore::push` 在 journal 写失败时不再把消息放入内存队列，而是 fail-fast 向上返回错误；`send_message`、broadcast、delegate 和 operator submit 会把失败显式暴露。timeout/self-reminder 未成功入队时不继续清理 timeout registry。studio pulse 重新入队失败时会把消息返回内存并标注错误。
- **验证**：`agent-team` 160/160 单元 + 9/9 集成；`agent-team-studio` 99/99（默认并行）与 7/7 mock 集成；fmt 和 clippy 通过。
- **待办**：为旧分支的 `inbox.json` 快照增加一次性迁移；为 operator drain 增加 UI 确认后的 ack；timer/delegation timeout 仍需重建。

### 2026-08-31 agent-team-studio inbox 快照恢复加固

**仓库**：`llm-harness-runtime`，分支 `feat/agent-team-studio`（远端 `ab915d2`）。

- **恢复幂等性**：`TeamManager::create` 成功把旧 `inbox.json` 重新入队后立即删除快照，避免进程在恢复后崩溃时再次启动导致同一批消息重复投递。
- **快照持久性**：shutdown 快照现在写入临时文件、`fsync` 文件、rename 覆盖目标，并 `fsync` 团队目录；避免半写快照或 rename 后目录项未落盘。
- **测试隔离**：`env_cfg` 新增测试专用 scoped provider factory guard，tests 持有互斥锁期间覆盖全局工厂、结束后清除；team-manager 和 route 测试改为持有该 guard，消除并行测试互相覆盖 factory 导致的偶发 runner 断言失败。
- **验证**：`cargo test -p llm-harness-agent-team-studio --lib` 通过 99/99（默认并行）；`--test integration` 通过 7/7（3 个 live 按预期 ignored）；fmt 和 clippy 通过。

### 2026-08-31 agent-team-studio clean-shutdown inbox 恢复

**仓库**：`llm-harness-runtime`，分支 `feat/agent-team-studio`（远端 `e7f8ef0`）。

- **恢复范围补齐**：startup 已能恢复 persisted specs 并重建 issues、contacts 和 runners；本次补齐 clean shutdown 时 pending inbox 的持久化，并在重新 create 团队后按 priority 恢复队列。
- **核心能力**：`InboxStore` 新增 `drain_everything` / `restore`，`SocialContext` 和 `AgentTeam` 暴露 pending message drain/restore；`TeamManager` 在显式 shutdown、shutdown-all 和 Drop 时写入 `teams/<id>/inbox.json`，create 恢复后自动注入。
- **回归覆盖**：新增 team 生命周期恢复测试和 clean-shutdown pending inbox 恢复测试；`TeamEntry` 模型引用从歧义别名改为 `main`，避免持久化后 explicit/tier 判定不确定。
- **验证**：`cargo test -p llm-harness-agent-team --lib inbox` 通过 18/18；`cargo test -p llm-harness-agent-team-studio --lib -- --test-threads=1` 通过 99/99；fmt 和 clippy 通过。并行测试首次出现全局 mock provider factory 竞态，隔离和串行复跑均稳定。

### 2026-08-31 agent-team-studio 完整 coding-team 模板 E2E

**仓库**：`llm-harness-runtime`，分支 `feat/agent-team-studio`（远端 `37978eb`）。

- **完整模板 E2E**：新增 ignored 测试 `live_llm_full_coding_team_template_completes_task`，加载内置 11 成员 `coding-team`，在临时 Git 仓库执行 bounded 任务并断言 `answer.txt` 与 `team/e2e` 分支。
- **本地 LLM 结果**：使用 `http://10.0.20.110:3001` + `moonshotai/kimi-k2.6`，完整任务在 220.11s 通过；judge 收到 5/5 reviewer 报告，独立执行字节级/命令验证，operator 收到最终裁决 PASS。
- **模板修复**：5 个 reviewer 从无工具改为挂载 `fs`，使其 persona 声明的 read/grep/git 能力可执行；planner 审查召集从 volunteer `broadcast` 改为 5 次 tracked `delegate_task`；reviewer 明确必须直发 judge 而非 operator；judge 增加 5/5 findings 收集门。
- **版本与升级**：内置模板版本从 1 提升到 2，触发存量团队的模板升级检查。
- **监测发现**：一次本地 provider 返回 `Database error`；`agent_runner` 记录失败并 re-queue，流程自动恢复，未导致 E2E 失败。
- **验证**：`cargo test -p llm-harness-agent-team-studio --lib` 通过 97 个单元测试；`--test integration` 通过 7 个 mock 集成测试（3 个 live 测试按预期 ignored）；fmt 和 clippy 通过。

### 2026-08-31 agent-team-studio 本地 LLM review 闭环 E2E

**仓库**：`llm-harness-runtime`，分支 `feat/agent-team-studio`（远端 `1528756`）。

- **覆盖扩展**：新增 ignored 测试 `live_llm_review_loop_reaches_operator`，使用本地 OpenAI 兼容端点和 `moonshotai/kimi-k2.6` 驱动 planner、reviewer、coder、judge 四个真实成员。
- **实跑结果**：完整链路 planner→reviewer→judge→planner（NEEDS_CHANGES）→coder→planner→judge→operator（PASS）通过，耗时 162.87s，operator 收到 `## 总体结论: PASS`。
- **验证**：默认 `--test integration` 通过 7 个 mock 集成测试（2 个 live 测试按预期 ignored）；`cargo fmt --all -- --check` 通过；`cargo clippy -p llm-harness-agent-team-studio --all-targets -- -D warnings` 通过。

### 2026-08-31 agent-team-studio review 闭环联系人修复

**仓库**：`llm-harness-runtime`，分支 `feat/agent-team-studio`（远端 `73d560c`）。

- **问题**：`coding-team` 模板要求 reviewer 通过 `send_message("judge")` 直发审查结果，judge 再向 planner 回流 `NEEDS_CHANGES`；但核心 `AgentTeam::build` 只为成员预置 operator 联系人，delegate 建立的联系人只覆盖 planner↔reviewer。首个完整 review-loop 回归用例确认 reviewer→judge 会因联系人不足卡住。
- **修复**：在 `agent-team-studio` 装配层为所有声明成员建立双向联系人；`max_contacts_per_agent` 提升为 `max(20, members.len())`，保留核心社交层的稀疏联系人语义不变。
- **回归**：新增确定性 `coding_team_review_loop_returns_changes_and_passes`，覆盖 reviewer→judge、judge→planner（NEEDS_CHANGES）、planner→coder、coder→planner、planner→judge、judge→operator（PASS）。
- **验证**：`cargo test -p llm-harness-agent-team-studio --test integration` 通过 7 个 mock 集成测试（1 个 live 测试按预期 ignored）；`cargo fmt --all -- --check` 通过；`cargo clippy -p llm-harness-agent-team-studio --all-targets -- -D warnings` 通过。

### 2026-08-30 agent-team-studio 初步收尾

**仓库**：`llm-harness-runtime`，分支 `feat/agent-team-studio`（远端 `722774e`）。

- **装配校验收口**：`TeamSpec` 现在在 assemble 前拒绝空/相对路径逃逸的 team id、重复 member id、空/路径逃逸/保留 member id，以及指向不存在成员的 `Handoff` 目标。团队 issues 目录改为装配成功后创建，避免启动恢复持久化 spec 时目录写到 `teams/` 之外；启动后 handoff 也不再静默失败。
- **模板目录安全**：`TemplateStore` 拒绝包含路径分隔符、`.`、`..` 或 NUL 的模板 id；用户目录列表只接受目录名与模板 JSON 内 id 一致的模板，load 时也校验这一致性。
- **成员更新校验**：`TeamManager::update_member` 在改动生效和持久化前重新校验 toolkit，避免 live update 绕过 assemble 的 fail-fast 约束。
- **成员密钥路径安全**：`PUT /api/team/agent/config` 现在先校验 project/agent 存在，再写 0600 secrets 文件；新增回归测试覆盖 `project`/`agent` 携带 `..` 的路径逃逸场景。
- **成员密钥持久化安全**：member provider override 的明文 key 只在 rebuild 装配前注入临时副本，`team.json` / entry spec 仍保留空占位；回归测试确认 `team.json` 不包含明文。
- **key-only 变更生效**：`update_member` 接收显式 `key_changed` 信号，即使 model/base_url/toolkit 没变，替换 secrets 后也会 rebuild 成员 harness；新增连续两次 key-only 更新的回归测试。
- **测试稳定性**：pulse/scout 单测为每个 team 使用独立 memory/session 临时目录，消除并发测试共享默认 `agent-team-memory` 导致 SQLite 初始化竞态。
- **模板升级执行**：`TeamSpec` 新增创建时的 `template_baseline`；`POST /api/team/upgrade-apply` 会用 baseline 识别未被用户覆写的成员，仅升级这些成员，保留覆写结果，结构性差异触发 member rebuild，并同步 handoff 路由、patrol 行为和持久化 template version。成员集合变化当前显式拒绝，等待后续成员管理流程。
- **模型引用往返修复**：`ModelRef` 反序列化只在命中 `strong/main/cheap` 时归为 tier，其余字符串保留 explicit 语义，避免持久化后误判为未知 tier。
- **升级前置校验**：模板升级前整体校验成员 ID / handoff 目标 / toolkit，避免畸形模板造成部分成员已更新的状态。
- **验证**：`cargo test --lib -p llm-harness-agent-team-studio` 通过 92 个单元测试，`--test integration` 通过 5 个集成测试；`cargo clippy -p llm-harness-agent-team-studio --all-targets -- -D warnings` 通过；`cargo fmt -p llm-harness-agent-team-studio` 通过。
- **下一步**：保持 `coding-team` 作为模板数据而非代码内角色，继续补 MCP toolkit、真实 LLM 稳定性、team 生命周期恢复与 UI 操作流验证。

### 2026-08-30 agent-team-studio MCP toolkit 装配

**仓库**：`llm-harness-runtime`，分支 `feat/agent-team-studio`（远端 `d56340e`）。

- **MCP toolkit 声明**：成员 `toolkits` 现在支持 `mcp:<server>` 格式；`validate_toolkits` 校验 server 名非空且不含路径分隔符/NUL，其余字符串仍按原有 `fs/web/memory/knowledge` 校验。
- **配置加载**：装配时从 `data_dir/.mcp.json` 加载 `McpConfigFile`（JSON，含 `mcpServers` 映射）；文件不存在或请求的 server 未配置时 fail fast 并返回 `McpServerNotConfigured`。
- **MCP 装配**：`prepare_mcp_tools` 收集所有成员的 unique MCP server 名，创建 `McpManager` 调用 `discover_and_connect`，然后通过 `get_tools()` 按过滤出每个 server 的工具列表。`assemble` 的 `IndividualSpec::configure` 闭包将匹配的 MCP 工具（`McpStudioTool` 适配器）注入对应成员的 harness builder。
- **工具适配器**：studio 内部实现 `McpStudioTool` 实现 `Tool` trait，通过共享 `McpManager.call_tool()` 路由执行，支持 text content 和 structured_content 输出；失败时返回 `ToolFailure`。
- **测试**：6 项新测试覆盖 server 名校验（有效/空/路径逃逸）、配置缺失报错、server 未在配置文件中报错、server 名去重。
- **MCP 生命周期修复**：`McpManagerLifecycle` RAII guard 存入 `TeamEntry`，Drop 时调用 `cancel_background_tasks()` 终止连接/重连任务，防止 team rebuild 或 shutdown 时 MCP 连接泄漏。lock 文件从 CWD 相对路径改为 `data_dir/.mcp-lock.json`。`McpServerNotConfigured` 错误现在携带请求该 server 的成员 ID。`discover_and_connect` 后逐一检查 `ConnectionStatus::Connected`，未连接的 server fail fast 返回 `McpConnection` 错误。
- **验证**：`cargo test --lib -p llm-harness-agent-team-studio` 通过 97 个单元测试，`--test integration` 通过 5 个集成测试；`cargo clippy -p llm-harness-agent-team-studio --all-targets -- -D warnings` 通过；`cargo fmt -p llm-harness-agent-team-studio` 通过。
- **下一步**：真实 LLM 稳定性验证、team 生命周期恢复、UI 操作流验证。
