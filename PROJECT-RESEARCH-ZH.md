# Shannon 项目研究笔记

本文档整理了对 Shannon Lite 仓库的结构化研究结果，目标不是重复 README，而是从工程实现角度解释：这个项目如何启动、如何执行、如何约束 agent、以及为什么它可以恢复和审计。

## 1. 项目定位

Shannon 是一个面向 Web 应用与 API 的白盒 AI 渗透测试框架。

它不是单纯“调用大模型扫描代码”，而是一个分阶段、多 agent、带审计与恢复能力的自动化 pentest 系统。它会结合源码分析、浏览器自动化、命令行工具和真实 exploitation，只把已经验证过的漏洞写进最终报告。

从仓库结构上看，这是一个 pnpm monorepo，主要包含两个应用：

- `apps/cli`：宿主机侧 CLI 编排层
- `apps/worker`：Temporal worker 与 pentest pipeline 的核心执行层

根目录主要负责统一工具链、文档和工作约束，而不承载主要业务逻辑。

## 2. 总体架构模型

把整个项目压缩成一个统一模型，最适合用“4 个纵向运行层 + 3 个横切能力”来理解。

### 2.1 四个纵向运行层

1. **根层工具链与 monorepo**
   - 负责 workspace、build/check、TypeScript/Biome 约束。
2. **CLI 宿主编排层**
   - 负责命令入口、凭证加载、路径解析、Docker 与 workspace 装配。
3. **worker / Temporal 编排层**
   - 负责 workflow、activity、phase 顺序、并发、resume 与 summary。
4. **service / AI 执行层**
   - 负责 prompt 执行、git checkpoint、deliverable validation、报告汇总等业务逻辑。

### 2.2 三个横切系统能力

1. **prompt / config 协议层**
   - 定义每个 agent 的输入、输出、工具边界和行为约束。
2. **structured queue / validator 阶段接口层**
   - 让 vuln analysis 与 exploit 之间通过 JSON queue 和文件契约衔接。
3. **audit / checkpoint / resume 持久化层**
   - 提供 `session.json`、`workflow.log`、prompt snapshot、git checkpoint 和恢复能力。

这个模型的价值在于：不会把 CLI、workflow、service、prompt 和 audit 混成“很多文件堆在一起”，而是能清楚看到每一层的边界。

## 3. 根层工具链与 monorepo

根目录的关键文件包括：

- `package.json`
- `pnpm-workspace.yaml`
- `turbo.json`
- `tsconfig.base.json`
- `biome.json`
- `.npmrc`

它们共同定义了仓库的工程约束：

- 用 `pnpm` 管理 workspace，只纳入 `apps/*`
- 用 `turbo run build` / `turbo run check` 管理跨包任务
- 用严格 TypeScript 选项控制类型安全
- 用 Biome 统一格式与 lint
- 用更保守的 npm 安装策略降低供应链风险

这说明仓库根层的职责是“统一编排和约束开发方式”，而不是实现 pentest 逻辑。

## 4. CLI 层：如何把一次命令变成一次扫描

CLI 层的核心文件包括：

- `apps/cli/src/index.ts`
- `apps/cli/src/commands/start.ts`
- `apps/cli/src/docker.ts`
- `apps/cli/src/env.ts`
- `apps/cli/src/mode.ts`
- `apps/cli/src/home.ts`
- `apps/cli/src/paths.ts`

### 4.1 CLI 的职责不是业务分析

CLI 层不负责真正的漏洞分析，也不负责 workflow 逻辑。

它的作用是把用户输入的命令、宿主机文件系统、Docker 环境、配置与凭证，整理成一个**可以稳定启动 worker 的执行上下文**。

### 4.2 local 与 npx 双模式

Shannon 支持两种运行模式：

- **local 模式**：以本地 clone 仓库方式运行，常见命令形态是 `./shannon ...`
- **npx 模式**：以 `npx @keygraph/shannon ...` 运行

模式本身会影响：

- 凭证来源
- home 目录位置
- workspace 根目录位置
- worker image 的来源

例如：

- local 模式优先读取 `./.env`
- npx 模式优先读取 `~/.shannon/config.toml`

### 4.3 start 命令的真实含义

`start` 不是“直接开始扫”。

它真正做的是：

1. 初始化 home 和 workspace 目录
2. 加载环境变量和 provider 配置
3. 校验凭证
4. 解析 repo/config 的 host 路径与 container 路径
5. 确保 Docker image 存在
6. 确保 Temporal / router 等基础设施存在
7. 创建一次扫描专属的 workspace 和 overlay 目录
8. 启动一个临时 worker 容器

所以 CLI 层本质是一个**运行时装配器**。

## 5. Docker 与 workspace 隔离设计

这一层主要由 `apps/cli/src/docker.ts` 与 `start.ts` 共同实现。

最重要的设计点是：

- 目标 repo 以 **只读** 方式挂进容器
- `.shannon/deliverables`
- `.shannon/scratchpad`
- `.shannon/.playwright-cli`

这些目录会被 workspace-backed 的可写目录覆盖。

这意味着 Shannon 追求的是一种平衡：

- agent 需要对白盒源码有完整视野
- 但不应该在整个 repo 内随意写入

因此它把运行副作用尽量限制在 `.shannon` 子树和 workspace 目录中。

这也是 resume 和审计能力成立的基础，因为所有关键工件都集中在受控位置。

## 6. Worker 启动模型

worker 的入口是：

- `apps/worker/src/temporal/worker.ts`

这个文件非常关键，因为它定义了 Shannon 的运行单位：

> 一次扫描 = 一个临时 worker 进程 / 容器

这个进程会同时承担两个角色：

- Temporal worker
- Temporal client

它的主要流程是：

1. 解析 CLI 参数
2. 决定是 fresh run 还是 resume
3. 必要时终止之前仍在运行的旧 workflow
4. 读取 pipeline config
5. 启动当前扫描专属的 Temporal worker
6. 提交一个 workflow 到该 task queue
7. 通过 query 轮询进度并等待结果
8. 完成后退出

这说明 Shannon 并不是一个常驻 worker 池，而更像**面向单次任务的一次性执行器**。

## 7. Temporal workflow：真正的流程骨架

相关文件：

- `apps/worker/src/temporal/workflows.ts`
- `apps/worker/src/temporal/shared.ts`
- `apps/worker/src/temporal/activities.ts`

### 7.1 workflow 负责什么

workflow 的职责不是执行业务细节，而是管理：

- phase 顺序
- 并发模型
- retry 策略
- exploitation gating
- resume
- progress query
- 最终 summary

### 7.2 整体阶段顺序

Shannon 的主流程可以概括为：

1. preflight validation
2. deliverables git init
3. pre-recon
4. recon
5. 五条 vuln / exploit pipeline
6. reporting

其中中间的 5 条 pipeline 分别对应：

- injection
- xss
- auth
- ssrf
- authz

### 7.3 pipeline 不是两段式 barrier

这里的设计很重要：它不是“先全量 vuln 分析，再统一 exploit”。

而是每条 lane 自己走：

> vuln agent → queue check → conditional exploit agent

不同 lane 可以并发推进，因此某一类漏洞在分析完成后可以立即进入 exploit，而不需要等待其它 lane。

## 8. activities：Temporal 边界适配层

`apps/worker/src/temporal/activities.ts` 的作用很容易被误解。

它不是核心业务层，而是**Temporal world 和 service world 之间的适配层**。

它保留的职责包括：

- heartbeat
- ApplicationFailure 分类
- activity attempt 处理
- workflow 级 container 获取与释放
- resume 记录
- checkpoint restore
- phase / workflow 日志入口

它不应该承载真正的 pentest 业务逻辑。真正的执行逻辑被刻意下沉到 `apps/worker/src/services/`。

这是理解整个仓库边界时非常关键的一点。

## 9. services：真正的业务核心

核心目录：

- `apps/worker/src/services/`

其中最关键的文件是：

- `agent-execution.ts`
- `preflight.ts`
- `reporting.ts`
- `git-manager.ts`
- `prompt-manager.ts`
- `config-loader.ts`
- `container.ts`
- `exploitation-checker.ts`
- `queue-validation.ts`

### 9.1 agent-execution.ts 是核心中的核心

如果只允许挑一个文件代表 Shannon 的业务心脏，那就是 `agent-execution.ts`。

它定义了单个 agent 的完整生命周期：

1. 加载 distributed config
2. 根据 registry 取 prompt template
3. 构造 prompt
4. 创建 git checkpoint
5. 启动 audit session
6. 调用 Claude executor
7. 执行 spending-cap 检测
8. 必要时写 structured output queue
9. 验证 deliverable
10. 成功则 commit，失败则 rollback

这个顺序表明系统并不信任“模型说自己完成了”，而是要求必须产出可验证的文件工件。

### 9.2 service 层的本质

service 层是**Temporal 无关的业务边界**。

这也是仓库里最重要的架构原则之一：

- workflow 管状态机
- activities 管 Temporal 边界
- services 管真实业务

## 10. AI runtime：模型如何被受控执行

相关目录：

- `apps/worker/src/ai/`

最关键文件：

- `claude-executor.ts`
- `message-handlers.ts`
- `models.ts`
- `queue-schemas.ts`

Shannon 并不是把 prompt 直接丢给模型“自由发挥”。

它会给模型套上一层运行时壳：

- 固定工作目录 `cwd`
- 统一 provider env passthrough
- 模型 tier 选择（small / medium / large）
- 流式消息消费
- 工具和结果日志
- structured output
- 成本统计
- typed error handling

这意味着 Shannon 的“AI”部分是被强工程约束包住的，而不是开放式脚本。

## 11. prompt / config：不是附属资源，而是协议层

相关文件：

- `apps/worker/src/session-manager.ts`
- `apps/worker/src/config-parser.ts`
- `apps/worker/src/services/prompt-manager.ts`
- `apps/worker/configs/config-schema.json`
- `apps/worker/prompts/*.txt`

### 11.1 agent registry 的作用

`session-manager.ts` 不只是“列个 agent 名单”。

它实际上把每个 agent 与以下元数据绑定：

- prompt template
- deliverable filename
- optional model tier
- prerequisites
- validator

因此它是连接“代码层”和“prompt 层”的主干。

### 11.2 config 不是简单 YAML

`config-parser.ts` 会：

- 校验 YAML 大小和格式
- 通过 AJV 校验 schema
- 做额外的安全语义校验
- 处理规则冲突和重复
- 输出规范化的 distributed config

这些配置最终会影响：

- auth context
- login instructions
- focus / avoid rules
- description 注入
- pipeline retry preset
- max concurrent pipelines

所以 config 是扫描行为的**控制平面**，而不是一个简单的可选输入文件。

### 11.3 prompt 是阶段协议

每个 prompt 文件都在规定：

- 当前 agent 的角色
- 输入文件是什么
- 输出 deliverable 是什么
- 工具限制是什么
- 和后续 agent 如何协作
- 报告结构如何组织

因此 prompt 层本质上是系统运行协议的一部分，而不是静态文案。

## 12. structured queue：analysis 与 exploit 的硬接口

Shannon 的一个关键设计是：

vuln analysis 和 exploit 之间不是靠自然语言衔接，而是靠结构化 queue 文件衔接。

关键文件：

- `apps/worker/src/ai/queue-schemas.ts`
- `apps/worker/src/services/queue-validation.ts`
- `apps/worker/src/services/exploitation-checker.ts`

其核心语义是：

1. vuln agent 必须输出符合 schema 的 exploitation queue
2. deliverable 和 queue 必须成对存在
3. queue JSON 必须合法
4. `vulnerabilities.length > 0` 才允许 exploit agent 启动

这让阶段之间的接口变成**机器可验证的契约**。

## 13. audit / checkpoint / resume：为什么这个系统能恢复

相关文件：

- `apps/worker/src/audit/audit-session.ts`
- `apps/worker/src/audit/metrics-tracker.ts`
- `apps/worker/src/audit/workflow-logger.ts`
- `apps/worker/src/audit/logger.ts`
- `apps/worker/src/audit/utils.ts`

### 13.1 审计目录结构

每个 workspace / session 下会生成：

- `session.json`
- `workflow.log`
- `agents/`
- `prompts/`
- `deliverables/`

### 13.2 三类关键信息

- `session.json`：机器可读的 durable state
- `workflow.log`：人类可读的执行时间线
- prompt snapshot / agent logs：可复盘、可审计的执行证据

### 13.3 resume 的本质

Shannon 的 resume 不是“从内存状态继续”。

它真正做的是：

1. 读取 session durable state
2. 校验 workspace URL 一致性
3. 终止旧 workflow
4. 找到最新成功 checkpoint
5. 恢复 deliverables 私有 git repo
6. 删除未完成 agent 的部分产物
7. 记录新的 resume attempt
8. 跳过已经完成的 agent，继续后续流程

这说明 Shannon 的恢复基点是：

> 审计工件 + deliverables checkpoint

而不是短暂的内存态。

## 14. 为什么这个架构成立

从工程设计角度看，Shannon 之所以没有退化成“很多 prompt + 一个大循环脚本”，是因为它同时满足了几件事：

- **隔离性**：每次扫描一个独立容器
- **可恢复性**：checkpoint + session.json + workflow.log
- **可并发性**：五条 vuln/exploit lane 并行
- **可验证性**：deliverable validator + queue validator
- **可审计性**：agent log、prompt snapshot、workflow log
- **可配置性**：config 控制 auth、rules、description、retry、concurrency

因此，它更像一个**agentic security workflow system**，而不只是一个“会调 Claude 的安全工具”。

## 15. 建议的阅读顺序

如果你是第一次接手这个仓库，建议按下面顺序读：

1. `apps/cli/src/commands/start.ts`
2. `apps/cli/src/docker.ts`
3. `apps/worker/src/temporal/worker.ts`
4. `apps/worker/src/temporal/workflows.ts`
5. `apps/worker/src/temporal/activities.ts`
6. `apps/worker/src/services/agent-execution.ts`
7. `apps/worker/src/services/prompt-manager.ts`
8. `apps/worker/src/services/config-loader.ts`
9. `apps/worker/src/ai/claude-executor.ts`
10. `apps/worker/src/session-manager.ts`
11. `apps/worker/src/config-parser.ts`
12. `apps/worker/src/audit/*.ts`
13. `apps/worker/prompts/*.txt`

这样读的好处是：

- 先理解系统怎么启动
- 再理解系统怎么编排
- 然后理解系统怎么执行 agent
- 最后理解系统如何被 prompt/config/audit 约束和支撑

## 16. 最终总结

Shannon 的核心不是“它能不能调用大模型”，而是它把 AI pentest 这件事工程化了。

它通过：

- CLI + Docker 组装运行环境
- Temporal 管理阶段状态机
- services 承担真正业务逻辑
- prompt/config 定义 agent 协议
- structured queue 连接 analysis 与 exploit
- audit/checkpoint/resume 保证过程可追溯可恢复

最终形成了一个完整的、可以落地运行的白盒 AI 渗透测试框架。

如果只把它看成“一个自动扫描工具”，会低估它。

更准确的理解应该是：

> Shannon 是一个以文件工件、阶段协议和可恢复执行为核心的 agentic security workflow system。
