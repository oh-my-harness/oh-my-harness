# oh-my-harness 项目当前进度

> 最后更新：2026-06-19（Phase C ArcGen term_selection_lite + resist_tune 集成验证通过）

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
- PR #6 待合并（`extended_thinking_budget` 字段支持）
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

### eda-agent ✅ v0.1 实现完成（2026-06-14）
针对 EDA 仿真软件内部 AMC 光刻模型校准流水线的专用 Agent。

5 个 EDA 专属工具：
- `run_eda_job`：PanGen 本地进程执行 + 轮询（早收敛检测、日志监控、CancellationToken）
- `read_eda_file`：读取 job_dir 下任意文件（路径穿越保护）
- `list_eda_files`：glob 模式列文件
- `search_knowledge`：递归 grep Obsidian Vault `.md` 文件
- `record_experience`：追加 `run_experience.jsonl`

已实现：
- EdaAgent + EdaAgentBuilder（注入 LlmClient + ExecutionEnv）
- System prompt：8 节点 AMC 校准流水线（findoptics → term_selection → model_check 等）
- CLI：`--job-dir / --vault-dir / --pangen-bin / --gateway / -p`
- 3 个 mock 集成测试，`cargo clippy` 零警告
- 默认配置：pangen `/data/pangen/pangen_2026.04.00.release/bin/pangen`，gateway `192.168.18.116:4730`

**已验证（2026-06-18 真实环境，small_case1 ArcGen pipeline 集成）：**
- EDA Agent 替换 ArcGen 所有 LLM 调用节点，完整跑通 AMC 校准全流程（08:49~10:29，约 100 分钟）
- 调用节点：`build_calibration_context` / `optical_result_analysis` / `mask_result_analysis` / `term_decision` / `term_selection_lite`（6 轮 LLM 驱动迭代）
- `term_decision`：EDA Agent 正确分析"10 gauges 严重欠约束"，给出保守决策（add_iter=0, del_iter=1, 4 terms）
- `term_selection_lite`：EDA Agent 驱动逐轮 del，诊断过拟合（cal=0.034, val=0.407）并做出有物理依据的 del 决策
- pipeline 最终完成，产出校准报告（overall=FAIL，预期，因 pframe_lite 无 grid 优化）

**Phase B 实现（2026-06-18）：**
- `_TermAdvisorProxy` 替换 `term_decision` 节点的 Python ReActEngine，EDA Agent 自主决策
- `read_vault_file` / `list_vault_dir` 两个新工具，让 Agent 直接读取 Obsidian Vault 文档
- D-008 修复：`_build_decide_prompt` 移除预消化数据，Agent 必须主动调用工具获取信息
- `agent.run()` 广播缓冲区竞态修复（D-fix）：改用 `build_context()` 从 session 直读最后一条助手消息，彻底消除 >256 事件时 stdout 为空的问题
- 端到端验证（term_decision 风格）：list_vault_dir + read_vault_file×3 + search_knowledge×3，产出完整合规 JSON（含 customized_add_map、terms、rationale）

**Phase C ArcGen 真实 Pipeline 集成验证（2026-06-19）：**
- EDA Agent decide_fn 替换 `term_selection_lite` 每轮 add/del/stop 决策，端到端验证通过
- C-001/C-002/C-003 三个 bug 全部修复（commit 01afeb9, 4d6b6b8 on `fix/resist-tune-ssh-check-arg`）：
  - C-001：add 失败时在下轮 prompt 中明示（`last_add_failed_op`）
  - C-002：add_pool 展示为 "Ax（已用 2/4）" 格式，仅列有余量的 operation
  - C-003：del 未改善 best_uwrms 时，下轮 prompt 明示回滚（`last_del_no_improve`）
- `resist_tune._poll_any` API 修复：旧 `(job_dir, list, timeout_sec=)` → 新 `([Path], timeout=)`（commit 4d6b6b8）
- Phase C v2 验证结果：`term_selection_lite` 4 轮完成（v1 需 8 轮，无 C-003 fix）
  - Round 0: del UV_1 → val 0.291→0.285
  - Round 1: del UV_2 → uwrms 0.030→0.027 (best)
  - Round 2: del UV_3 → no improve → C-003 反馈 → Round 3 stop
  - `resist_tune` 100 轮 calibrate 完成，cal_uwrms=0.033，验证通过
  - pipeline 推进至 model_check（threshold=0.782 > 0.6，属 small_case1 9-gauge 数据质量限制）
- ArcGen MR !253：http://10.0.20.120/opc-agent-group/ArcGen/-/merge_requests/253（含所有 Phase B/C fix）
- `--restart-from term_decision` 跑 `amc-pipeline-20260618-084928`，`_TermAdvisorProxy` 路径全程通过
- `term_decision` 节点 EDA Agent（Rust）完成，耗时 ~1min10s；产出物理合规决策：
  - add_iter=1, del_iter=1，TopoField 加入 customized_add_map（知识库 Dark Field 推荐），Ax/Bx sigma 范围从 [30,40] 放宽至 [30,120]
  - 完整 rationale：NTD+Binary+Dark Field+TE+NA=1.35，10 gauges 过拟合风险 → 保守配置
- pipeline 继续推进至 `term_selection_lite`（Python LLM agent）正常运行
- ArcGen 侧 bug fix：`run.py` 的 `restart_from`/续跑路径 `_resist_engine` 未初始化 → `UnboundLocalError`，已修复并提交（`feat/eda-agent-phase-c`）

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
