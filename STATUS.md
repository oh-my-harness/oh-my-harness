# oh-my-harness 项目当前进度

> 最后更新：2026-07-21（eda-agent-py 全部 issue 清零：#24-#32 + #33 全部修复/关闭。ContextAdvisor LLM ReAct agent + llm-interactive 模式 + gauge_check 用户交互 + ArcGen c15dc55/88c1862 对齐；83/83 测试通过）

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
