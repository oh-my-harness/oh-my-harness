# oh-my-harness 开发规范

## 仓库概览

```
oh-my-harness/
├── llm-api-adapter       # LLM provider 适配层
├── llm-harness-core      # agent 框架核心
├── llm-harness-runtime   # 运行时基础设施（tools/auth/config/registry）
├── coding-agent          # coding 领域 agent
├── eda-agent             # EDA 模型校准 agent（东方晶源/光科芯图）
└── llm-tutor             # 教学辅导 agent
```

## 层级架构与依赖方向

```
coding-agent / eda-agent / other-agent      ← agent 层（领域专属）
──────────────────────────────────────────
llm-harness-runtime                         ← 运行时层（通用基础设施）
  crates/runtime-types    零外部依赖
  crates/runtime-tools    只依赖 runtime-types
  crates/runtime          依赖 runtime-tools + runtime-types + harness-core + llm-api-adapter
──────────────────────────────────────────
llm-harness-core                            ← 框架核心层
  crates/llm-harness-types   零外部依赖（不含 llm-api-adapter）
  crates/llm-harness-loop    依赖 llm-harness-types + llm-api-adapter
  crates/llm-harness         依赖 llm-harness-loop + llm-harness-types
──────────────────────────────────────────
llm-api-adapter                             ← provider 适配层
```

依赖只能向下，不能向上，不能跨层。

## 各层职责边界

### llm-api-adapter
- 负责：把各家 provider 的 wire 格式归一化为 canonical 类型
- 不负责：读 env var、选择用哪个 provider、retry 策略、prompt 模板

### llm-harness-core
- 负责：agent loop、session 持久化、compaction、skills/PromptTemplate 加载、AgentHarness
- 不负责：工具实现、模型注册、凭证存储、配置文件读取

### llm-harness-runtime（v0.2 已实现）
- 负责：通用工具集（read/bash/edit/write/grep/find/ls）、ModelRegistry、AuthStorage、SettingsManager、扩展系统、AgentRuntime 组装
- 不负责：领域专属工具、领域 system prompt

### coding-agent / eda-agent 等
- 负责：领域专属工具、领域 system prompt、CLI 入口
- 不负责：通用工具实现（应依赖 runtime-tools）、模型选择逻辑（应依赖 ModelRegistry）

## 当前临时状态（已知技术债）

`llm-harness-runtime` v0.2 已实现，但 `coding-agent` 的临时实现尚未迁移：

| 文件 | 临时位置 | 目标位置 |
|------|---------|---------|
| `src/tools/` | coding-agent | runtime-tools crate |
| `src/settings.rs` | coding-agent | runtime SettingsManager |
| bin 中的 provider 选择逻辑 | coding-agent | runtime ModelRegistry |

**迁移时不要在 coding-agent 中继续扩展这些模块。**

## 耦合红线

以下行为会引入耦合，禁止：

1. **llm-harness-types 引入外部 crate 依赖**（包括 llm-api-adapter）
2. **llm-harness-core 直接依赖 llm-api-adapter**（只能通过 llm-harness-loop 的 pub use 获取其类型）
3. **agent 层（coding-agent 等）直接依赖 llm-harness-core**（runtime 实现后应通过 AgentRuntime 接口）
4. **runtime-tools 依赖 runtime**（工具不感知上层组装逻辑）
5. **跨层跳依赖**（如 coding-agent 直接依赖 llm-harness-loop）

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
| 这个工具 coding-agent 和 eda-agent 都会用吗？ | 是 → runtime-tools；否 → 对应 agent |
| 这个能力和"EDA 工艺"或"代码编辑"有关吗？ | 是 → 对应 agent；否 → 往下层放 |
| 这个逻辑需要知道用的是哪个 LLM provider 吗？ | 是 → runtime ModelRegistry；否 → 可能属于 harness-core |
| 这个类型其他所有 crate 都要用吗？ | 是 → llm-harness-types 或 runtime-types（零依赖） |

## 各仓库的 CLAUDE.md

每个仓库有自己的 CLAUDE.md，包含该仓库的详细开发规范：

- `llm-harness-core/CLAUDE.md` — harness-core 的实现原则和测试规范
- `llm-api-adapter/CLAUDE.md` — adapter 的类型设计和 provider 实现约定

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
