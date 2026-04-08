# Shannon 阅读地图（按文件路径索引）

本文档面向第一次接手 Shannon 仓库的工程师。

目标不是解释所有细节，而是回答三个问题：

1. **先看什么**
2. **每个关键文件负责什么**
3. **读这个文件时要回答什么问题**

如果你已经看过 `PROJECT-RESEARCH-ZH.md`，可以把本文档当成配套导航；如果还没看，也可以直接按这里的顺序进仓库。

---

## 一、最快上手顺序

如果你时间有限，建议按下面顺序读：

1. `README.md`
2. `apps/cli/src/commands/start.ts`
3. `apps/cli/src/docker.ts`
4. `apps/worker/src/temporal/worker.ts`
5. `apps/worker/src/temporal/workflows.ts`
6. `apps/worker/src/temporal/activities.ts`
7. `apps/worker/src/services/agent-execution.ts`
8. `apps/worker/src/services/prompt-manager.ts`
9. `apps/worker/src/services/config-loader.ts`
10. `apps/worker/src/ai/claude-executor.ts`
11. `apps/worker/src/session-manager.ts`
12. `apps/worker/src/config-parser.ts`
13. `apps/worker/src/audit/audit-session.ts`
14. `apps/worker/prompts/`

这个顺序遵循的是一条真实控制链：

> 用户命令 → CLI 编排 → worker 启动 → workflow 编排 → activity 适配 → service 执行 → prompt/config 约束 → audit/resume 持久化

---

## 二、根目录：先建立全局认知

### 1. `README.md`

**为什么先读它：**

- 这里定义了产品定位、运行方式、工作区语义、provider 支持范围、输出结果和整体架构介绍。

**读它时要回答的问题：**

- Shannon 是什么，不是什么？
- local 模式和 npx 模式有什么区别？
- 为什么必须用 Docker？
- 用户视角下的 workspace / resume 是什么意思？

**读完后的收获：**

- 你会知道这是白盒 AI pentest 框架，而不是通用代码审查工具。

### 2. `package.json`

**为什么读：**

- 看根层命令、包管理器版本、build/check/biome 的统一入口。

**关键问题：**

- 仓库的标准验证命令是什么？
- 根目录是不是主要业务入口？

### 3. `pnpm-workspace.yaml`

**为什么读：**

- 确认 monorepo 的边界，只包含哪些包。

**关键问题：**

- 仓库里是不是只有 `apps/cli` 和 `apps/worker` 两个主要 app？

### 4. `turbo.json`

**为什么读：**

- 看跨包 build/check 是怎样编排的。

**关键问题：**

- `build` 和 `check` 的依赖关系是什么？

### 5. `tsconfig.base.json` / `biome.json`

**为什么读：**

- 理解全仓库统一的类型和风格约束。

**关键问题：**

- 类型系统有多严格？
- 风格规则是 ESLint/Prettier 还是 Biome 驱动？

---

## 三、CLI 层：一次扫描是怎么被启动的

### 6. `apps/cli/src/index.ts`

**作用：**

- CLI 手写命令分发入口。

**关键问题：**

- CLI 支持哪些命令？
- 哪些命令只允许 local，哪些只允许 npx？

### 7. `apps/cli/src/commands/start.ts`

**作用：**

- 整个系统的宿主机启动编排入口。

**这是最值得在 CLI 层优先精读的文件。**

**关键问题：**

- 一次扫描从用户命令开始，具体经历了哪些步骤？
- workspace 是什么时候创建的？
- repo/config 的路径是怎样转换成容器可见路径的？
- worker container 是在什么条件下启动的？

**你应该重点抓住：**

- env 加载
- credential 校验
- repo/config path resolve
- workspace overlay 目录准备
- ensure image / ensure infra
- spawn worker

### 8. `apps/cli/src/docker.ts`

**作用：**

- Docker 与 infra 管理层。

**关键问题：**

- 为什么 repo 以只读方式 mount？
- `.shannon/deliverables`、`.scratchpad` 为什么要单独覆盖挂载？
- `ensureInfra()`、`ensureImage()`、`spawnWorker()` 分别负责什么？

**读完后的关键认知：**

- Shannon 不把整个 repo 当成可随意写入的沙箱。
- 它把副作用尽量限制在 workspace-backed 覆盖目录里。

### 9. `apps/cli/src/env.ts`

**作用：**

- 处理 provider 凭证与环境变量逻辑。

**关键问题：**

- Anthropic / OAuth / Bedrock / Vertex / router 是怎么分流的？
- 环境变量与持久化配置哪个优先？

### 10. `apps/cli/src/mode.ts`

**作用：**

- 决定当前是 local 还是 npx 模式。

**关键问题：**

- 模式切换会影响哪些行为？

### 11. `apps/cli/src/home.ts` / `apps/cli/src/paths.ts`

**作用：**

- 决定 `~/.shannon`、`workspaces`、repo/config 的路径规则。

**关键问题：**

- 为什么 local 和 npx 模式下 workspace 根路径不同？
- host path 和 container path 是怎样映射的？

---

## 四、worker 入口：容器内发生了什么

### 12. `apps/worker/src/temporal/worker.ts`

**作用：**

- worker 容器内的主入口。

**这也是全仓库最值得优先精读的文件之一。**

**关键问题：**

- 为什么一个进程同时充当 Temporal worker 和 Temporal client？
- 为什么每次扫描都是一个独立 worker 容器？
- resume 是在这里接入的，还是在 workflow 里接入的？

**重点抓：**

- CLI arg parsing
- workspace resolve
- resume old workflow termination
- pipeline config loading
- workflow submit + progress polling

---

## 五、workflow 层：真正的流程骨架

### 13. `apps/worker/src/temporal/workflows.ts`

**作用：**

- Shannon 的主状态机。

**关键问题：**

- phase 顺序是什么？
- 哪些步骤顺序执行，哪些并行执行？
- vuln analysis 和 exploit 的耦合点在哪里？
- resume / skip / summary 是怎么做的？

**最关键的认知：**

- workflow 负责“什么时候做什么”，不是“具体怎么做”。

### 14. `apps/worker/src/temporal/shared.ts`

**作用：**

- workflow 输入、状态、query 输出的共享类型。

**关键问题：**

- `PipelineInput`、`PipelineState`、`PipelineProgress` 分别表示什么？

### 15. `apps/worker/src/temporal/activities.ts`

**作用：**

- Temporal 边界适配层。

**关键问题：**

- 哪些逻辑留在 activity，哪些下沉到 service？
- error classification 是在哪一层做的？
- resume、checkpoint restore、workflow cleanup 为什么还在这里？

**重要提醒：**

- 不要把 activities 当成核心业务层，它是 adapter。

---

## 六、service 层：真正的业务执行核心

### 16. `apps/worker/src/services/agent-execution.ts`

**作用：**

- 单个 agent 的完整生命周期执行器。

**如果你只能精读一个 worker 业务文件，就读它。**

**关键问题：**

- 一个 agent 从开始到结束都经历了什么？
- prompt 是什么时候拼好的？
- git checkpoint 为什么在执行前就创建？
- structured queue 是什么时候写盘的？
- deliverable validation 为什么必须在成功前执行？

### 17. `apps/worker/src/services/preflight.ts`

**作用：**

- 在昂贵 agent 执行前做低成本检查。

**关键问题：**

- repo、config、credentials、URL reachability 的预检规则是什么？
- 为什么要在 workflow 开头单独做 preflight？

### 18. `apps/worker/src/services/reporting.ts`

**作用：**

- 汇总 exploitation evidence，生成最终综合报告。

**关键问题：**

- 报告是如何拼装的？
- executive summary 的元数据从哪里来？

### 19. `apps/worker/src/services/git-manager.ts`

**作用：**

- 管理 deliverables 私有 git repo、checkpoint、rollback、success commit。

**关键问题：**

- 为什么要用 git checkpoint 而不是只靠内存或 JSON 状态？
- 并行 agent 如何避免 git index.lock 冲突？

### 20. `apps/worker/src/services/container.ts`

**作用：**

- per-workflow 级别的轻量 DI 容器。

**关键问题：**

- 哪些 service 是共享的，哪些状态不能共享？

### 21. `apps/worker/src/services/config-loader.ts`

**作用：**

- 包装 config parse / distribute 逻辑，给 service 层提供统一入口。

### 22. `apps/worker/src/services/prompt-manager.ts`

**作用：**

- 读取 prompt 模板，处理 includes，注入变量。

**关键问题：**

- `@include(...)` 是怎样解析的？
- `{{AUTH_CONTEXT}}`、`{{LOGIN_INSTRUCTIONS}}`、`{{DESCRIPTION}}` 等变量从哪里来？

### 23. `apps/worker/src/services/exploitation-checker.ts`

**作用：**

- 决定某个 exploit agent 应不应该运行。

### 24. `apps/worker/src/services/queue-validation.ts`

**作用：**

- 验证 vuln deliverable 与 queue 文件是否配对、是否合法。

**关键问题：**

- exploitation gating 的根判断依据是什么？

---

## 七、AI runtime：模型如何被工程化约束

### 25. `apps/worker/src/ai/claude-executor.ts`

**作用：**

- Claude SDK 的受控执行壳。

**关键问题：**

- prompt 最终如何变成一次 Claude query？
- provider env 是怎样透传的？
- spending-cap、billing、rate limit 是怎样识别的？
- structured output 是怎样返回上层的？

### 26. `apps/worker/src/ai/message-handlers.ts`

**作用：**

- 消费 SDK stream，按消息类型分类处理。

**关键问题：**

- assistant/tool/result/system 消息各自如何处理？

### 27. `apps/worker/src/ai/models.ts`

**作用：**

- small / medium / large 模型 tier 解析。

### 28. `apps/worker/src/ai/queue-schemas.ts`

**作用：**

- 定义 vuln agent 的 structured output schema 和 queue 文件映射。

**关键问题：**

- 为什么 exploit 不是靠自由文本判断，而是靠 queue schema 判断？

---

## 八、协议层：agent registry、config 与 prompt 模板

### 29. `apps/worker/src/session-manager.ts`

**作用：**

- 定义 agent metadata、依赖关系、deliverable filename、validator、model tier。

**关键问题：**

- 系统是如何知道每个 agent 用哪个 prompt、产出哪个文件、用哪个 validator 的？

### 30. `apps/worker/src/config-parser.ts`

**作用：**

- 解析 YAML，做 AJV 校验和额外安全校验，输出 distributed config。

**关键问题：**

- config 为什么不只是“读个 YAML”？
- 哪些危险模式会被主动拒绝？
- config 如何影响 auth、rules、description、retry preset、concurrency？

### 31. `apps/worker/configs/config-schema.json`

**作用：**

- config 的结构契约。

### 32. `apps/worker/prompts/`

**作用：**

- 各阶段 agent 的行为协议模板。

**建议阅读顺序：**

1. `recon.txt`
2. `vuln-injection.txt`
3. `exploit-injection.txt`
4. `report-executive.txt`
5. `shared/*`

**关键问题：**

- prompt 如何定义输入文件、输出 deliverable、工具边界、证据标准和上下游交接？

---

## 九、审计与恢复：为什么它能 resume

### 33. `apps/worker/src/audit/audit-session.ts`

**作用：**

- 审计系统总入口，协调 metrics、workflow log、agent log 和 prompt snapshot。

### 34. `apps/worker/src/audit/metrics-tracker.ts`

**作用：**

- 维护 `session.json`，记录 session 状态、agent 结果、成本、resumeAttempts、checkpoint。

### 35. `apps/worker/src/audit/workflow-logger.ts`

**作用：**

- 维护人类可读的 `workflow.log`。

### 36. `apps/worker/src/audit/logger.ts`

**作用：**

- 写 per-agent append-only 日志，并保存 prompt snapshot。

### 37. `apps/worker/src/audit/utils.ts`

**作用：**

- 统一生成 audit 路径与目录结构。

**关键问题：**

- 为什么一个 workspace 下会固定出现 `session.json`、`workflow.log`、`agents/`、`prompts/`、`deliverables/`？

---

## 十、读仓库时最容易误解的点

### 1. 不要把 CLI 当成业务核心

CLI 负责的是宿主机编排，不是 pentest 实现。

### 2. 不要把 workflow 当成业务核心

workflow 负责状态机、并发、resume 和 summary；真正业务逻辑在 services。

### 3. 不要把 prompt 当成“纯文本资源”

prompt 是运行协议的一部分。

### 4. 不要把 resume 理解成“接着跑”

Shannon 的 resume 是基于 durable state + checkpoint + deliverable 清理的恢复过程。

### 5. 不要把 exploit 理解成“分析完成就一定运行”

exploit 必须经过 queue gate，只有分析产出了合格且非空的 queue 才会启动。

---

## 十一、建议的阅读目标

如果你是为了不同目的来读这个仓库，可以直接跳到这些文件组：

- **想看启动链路**：`start.ts` → `docker.ts` → `worker.ts`
- **想看流程骨架**：`workflows.ts` → `activities.ts`
- **想看业务核心**：`agent-execution.ts` → `git-manager.ts` → `reporting.ts`
- **想看模型执行**：`claude-executor.ts` → `message-handlers.ts`
- **想看配置与提示词协议**：`session-manager.ts` → `config-parser.ts` → `prompt-manager.ts` → `prompts/`
- **想看恢复与审计**：`audit-session.ts` → `metrics-tracker.ts` → `workflow-logger.ts`

---

## 十二、和研究文档的关系

- `PROJECT-RESEARCH-ZH.md`：回答“这个系统整体是什么，为什么这么设计”
- `READING-MAP-ZH.md`：回答“应该先读什么文件，每个文件要解决什么问题”

建议先看研究文档建立全局图，再用这份阅读地图深入源码。
