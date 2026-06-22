# oh-my-harness 项目当前进度

> 最后更新：2026-06-22（eda-agent small_case1 完整验证，prompt 修复 BUG-006/007/008/009/DD-003/005）

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

### eda-agent ✅ v0.2 lite mode 完成 + small_case1 完整验证（2026-06-22）
针对 EDA 仿真软件内部 AMC 光刻模型校准流水线的专用 Agent。

6 个 EDA 专属工具：
- `run_eda_job`：PanGen 本地进程执行 + 轮询（早收敛检测、日志监控、CancellationToken；新增 `clear_existing_result` 参数）
- `write_eda_file`：写 job_dir 内文件，含路径穿越保护（lite/term_pool.json 迭代更新用）
- `read_eda_file`：读取 job_dir 下任意文件（路径穿越保护）
- `list_eda_files`：glob 模式列文件
- `search_knowledge`：递归 grep Obsidian Vault `.md` 文件
- `record_experience`：追加 `run_experience.jsonl`

已实现：
- EdaAgent + EdaAgentBuilder（注入 LlmClient + ExecutionEnv，RetryConfig 429/529 重试）
- System prompt：**7 节点 AMC Lite 校准流水线**（findoptics → optical_search → gridparam → mask_search → term_decision → **term_selection_lite** → model_check）
- `term_selection_lite` 迭代循环：Steps A-E，lite_check 字段（A/B/C/D/E），决策表，max 20 轮
- CLI：`--job-dir / --vault-dir / --pangen-bin / --gateway / --arcgen-dir / -p`
- 3 个 mock 集成测试，`cargo clippy` 零警告

**Phase B/C 验证（2026-06-18/19）**：详见上方历史记录。

**small_case1 完整 EDA Agent orchestrator 验证（2026-06-22，17:56~19:30，约 94 分钟）**：
- EDA Agent 独立 orchestrator 模式（不依赖 ArcGen Python），驱动完整 lite 模式流水线
- 执行路径：skip findoptics/optical/mask（结果已存在）→ term_decision → term_selection_lite（多轮迭代）→ model_check
- 发现并记录 BUG-006/007/BUG-008/BUG-009/DD-003/DD-005 共 6 个问题（见 eda-agent/ISSUES.md）
- Prompt 修复：BUG-006/007/BUG-008/DD-003（commit d2e3eba）+ BUG-009/DD-005（commit 1df81c1）均已修复
- 核心发现：small_case1 仅 10 cal gauges，NTD ultra 16 term 严重过拟合（cal=0.029, val=0.451）；11 项基础 NTD term 最佳（cal=0.109, val=0.311）；根本限制是 gauge 数量不足（需 20-30+ gauge 才能收敛）
- check_E 方向修复（DD-003）已生效：Agent 正确识别 cal>val=欠拟合，但仍违反了 add 规则（直接进 model_check），已强化 prompt（DD-005）

**待做（优先级排序）**：
- BUG-009：model_check result_pattern 已改为 gauge.txt（1df81c1），需新运行验证
- DD-005：check_E 进入 model_check 条件已强化（1df81c1），需新运行验证
- BUG-005：term_decision feedback 循环重新评估（独立 orchestrator 架构下）
- 更大 case 验证：需 60+ gauge case 才能充分验证 ntd_ultra 迭代收敛

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
