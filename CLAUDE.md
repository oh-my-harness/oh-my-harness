# oh-my-harness 开发规范

## 仓库概览

```
oh-my-harness/
├── llm-api-adapter       # LLM provider 适配层
├── llm-harness-runtime   # 运行时层（agent loop + workflow engine + tools/auth/config/FFI）
│   ├── crates/llm-harness-types      零外部依赖（类型定义）
│   ├── crates/llm-harness-loop       依赖 types + llm-api-adapter（agent loop 引擎）
│   ├── crates/llm-harness-agent      依赖 loop（AgentHarness、session、compaction、skills）
│   ├── crates/llm-harness-rules      规则引擎
│   ├── crates/llm-harness-runtime    依赖 agent + types（workflow engine、sandbox、task store）
│   ├── crates/llm-harness-ffi        依赖 runtime（C ABI + Python SDK binding）
│   ├── crates/llm-harness-runtime-sandbox-*   沙箱实现（os/bwrap/seatbelt）
│   ├── crates/llm-harness-runtime-auth        认证
│   ├── crates/llm-harness-runtime-mcp         MCP 客户端
│   ├── crates/llm-harness-runtime-audit-jsonl 审计日志
│   └── crates/llm-harness-runtime-trace-otel  OpenTelemetry trace
├── eda-agent             # EDA 模型校准 agent（Rust，依赖 runtime）
├── eda-agent-py          # EDA 模型校准 agent（Python，通过 FFI 复用 runtime）
└── llm-tutor             # 教学辅导 agent
```

## 层级架构与依赖方向

```
eda-agent / eda-agent-py / other-agent      ← agent 层（领域专属）
──────────────────────────────────────────
llm-harness-runtime                         ← 运行时层（通用基础设施）
  crates/llm-harness-runtime    依赖 llm-harness-agent + types + rules
  crates/llm-harness-agent      依赖 llm-harness-loop + types
  crates/llm-harness-loop       依赖 llm-harness-types + llm-api-adapter
  crates/llm-harness-types      零外部依赖
  crates/llm-harness-ffi        C ABI + Python SDK（暴露 runtime 能力给 Python agent）
──────────────────────────────────────────
llm-api-adapter                             ← provider 适配层
```

依赖只能向下，不能向上，不能跨层。

**llm-harness-core 已合并到 llm-harness-runtime**（原 `llm-harness-types` / `llm-harness-loop` /
`llm-harness` 三个 crate 现在直接是 runtime workspace 的成员，不再有独立的 core 层）。
**coding-agent 已独立为单独仓库**（https://github.com/oh-my-harness/coding-agent.git）。

## 各层职责边界

### llm-api-adapter
- 负责：把各家 provider 的 wire 格式归一化为 canonical 类型
- 不负责：读 env var、选择用哪个 provider、retry 策略、prompt 模板

### llm-harness-runtime（含原 core 功能）
- **llm-harness-types**：Tool/AgentEvent/Session 等全部类型定义（零外部依赖）
- **llm-harness-loop**：agent loop 引擎，streaming 驱动，tool 调度
- **llm-harness-agent**：AgentHarness、session 持久化、compaction、skills/PromptTemplate 加载
- **llm-harness-runtime**：WorkflowEngine 状态机（step → execute → judge → transition）、
  TaskStore 持久化/崩溃恢复、Sandbox、ToolRegistry、AuditSink、BudgetControl
- **llm-harness-ffi**：C ABI 入口 + Python SDK binding（暴露 runtime 能力给 Python agent）
- 不负责：领域专属工具、领域 system prompt

### eda-agent / eda-agent-py / other-agent
- 负责：领域专属工具、领域 system prompt、CLI 入口、pipeline.yaml
- Rust agent（eda-agent）：直接依赖 `llm-harness-runtime` crate，实现 `workflow_adapter/`
- Python agent（eda-agent-py）：通过 `llm-harness-ffi` 的 Python SDK 复用 runtime，
  实现 `workflow_adapter/`（converter + EdaExecutor + EdaJudge）通过 FFI 回调接入
- 不负责：workflow 引擎实现、状态机、持久化（这些是 runtime 层的职责）

## 当前状态

`coding-agent` 已独立为单独仓库，其临时实现迁移工作在该仓库内进行。

`eda-agent`（Rust）：已完整接入 runtime WorkflowEngine（`workflow_adapter/` + `pipeline.yaml`），
E2E 验证通过（38 stage + 崩溃恢复 + loop counter 持久化）。

`eda-agent-py`（Python）：通过 FFI SDK 复用 runtime。FFI 已暴露 AgentHarness（LLM prompt + tool calling），
WorkflowEngine 尚未暴露到 FFI（已知缺口 G1/G2/G3，见 `eda-agent-py/FINDINGS.md`）。
Python 版采用 `workflow_engine/` Python 镜像作为临时编排器（1:1 复刻 runtime 语义），
待 FFI 暴露 WorkflowEngine 后迁移到 FFI 路径。
**当前进度**（2026-07-11）：38 stage pipeline 全部节点已实现，38 测试通过，E2E 全流程跑通（success）。
resist_tune → model_check → calibration_report 全部通过。resist_tune TCC 参数不匹配 bug 已修复（A027）。
model_check_feedback 已达生产级（F/G 4 策略 + FeedbackAdvisor LLM + best_uwrms 回退 + history 追踪），
gauge_error_attribution 已接入 LLM 分析器，F-51 PanGen 断连恢复已实现。
FFI system_prompt 已修复（G4/F-01）。PanGen stall detection 已覆盖 calibrate/compute_tcc session。
运行环境：python3.14（/usr/local/bin/python3.14），pandas + pyyaml 可用。

## 耦合红线

以下行为会引入耦合，禁止：

1. **llm-harness-types 引入外部 crate 依赖**（包括 llm-api-adapter）
2. **agent 层（eda-agent 等）绕过 runtime 直接依赖 llm-api-adapter**（应通过 runtime 或 FFI SDK）
3. **Python agent 自建 runtime 层**（runtime 是 Rust crate 的职责，Python agent 通过 FFI 复用，不在 agent 层重写）
4. **跨层跳依赖**（如 eda-agent 直接依赖 llm-harness-loop 而非 llm-harness-runtime）

## 开发前必做：先定层级，再动手

**任何新功能、新工具、新类型，在写第一行代码之前，必须先回答：它属于哪一层？**

流程：
1. 用下方"归属判断"表确定目标层
2. 确认该层的依赖规则允许这个实现
3. 如果目标层尚未实现（如 runtime），明确是"临时放在上层、等待迁移"还是"等 runtime 实现后再做"
4. 以上三步达成共识后再写代码

跳过这一步直接写代码是引入耦合的主要原因。

## 新增功能的归属判断

| 问题 | 判断方法 |
|------|---------|
| 这个工具多个 agent 都会用吗？ | 是 → runtime 层；否 → 对应 agent |
| 这个能力和"EDA 工艺"或"代码编辑"有关吗？ | 是 → 对应 agent；否 → 往下层放 |
| 这个逻辑需要知道用的是哪个 LLM provider 吗？ | 是 → runtime 层；否 → 可能属于 llm-harness-types |
| 这个类型其他所有 crate 都要用吗？ | 是 → llm-harness-types 或 runtime-types（零依赖） |

## 各仓库的 CLAUDE.md

每个仓库有自己的 CLAUDE.md，包含该仓库的详细开发规范：

- `llm-harness-runtime/CLAUDE.md` — runtime 的实现原则和测试规范（含原 core 功能）
- `llm-api-adapter/CLAUDE.md` — adapter 的类型设计和 provider 实现约定
- `eda-agent/CLAUDE.md` — eda-agent Rust 版开发规范
- `eda-agent-py/CLAUDE.md` — eda-agent Python 版开发规范

## 开发完成后必做：更新 STATUS.md

**每次开发任务完成后，必须同步更新 `STATUS.md` 并提交到 `oh-my-harness/oh-my-harness` 仓库。**

更新内容包括：
- 对应仓库/模块的状态标记（⚠️ → ✅ 等）
- 已完成的能力简述
- 待做事项调整
- 更新"最后更新"日期

```bash
# 在 oh-my-harness/ 目录下执行
git add STATUS.md CLAUDE.md
git commit -m "docs: update STATUS.md after <任务名>"
git push
```

## 文档同步原则（所有子项目通用）

**有效结果及时同步**：跑通验证、修了 bug、完成一个阶段后，**当场**更新对应文档（HANDOFF / ORCHESTRATOR / STATUS 等），不要攒着留给下一次。判断标准：新开一个对话只看这些文档能否快速接手？不能则需要更新。
