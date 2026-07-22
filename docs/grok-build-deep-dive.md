# 拆解 Grok Build：Coding Agent 远比想象中复杂，而它的解法是通用的

> 对 [xai-org/grok-build](https://github.com/xai-org/grok-build) 源码的深度阅读笔记。
> 这篇文章不介绍"它是什么"，而是回答两个问题：
> **(1) 为什么一个 coding agent 需要 85 个 crate？(2) 这些设计里，哪些是可以原样搬进企业级 Agent 应用的思想？**

---

## 零、先泼一盆冷水：ReAct 循环只值 100 行

任何写过 Agent demo 的人都知道核心循环长什么样：

```python
while not done:
    response = llm.chat(messages, tools=tools)
    if response.tool_calls:
        for call in response.tool_calls:
            result = execute(call)
            messages.append(result)
    else:
        done = True
```

这确实是 Grok Build 的心跳。但如果这就是全部，它不需要 85 个 crate、几十万行 Rust。**真正的工作量不在循环里，而在循环周围**——在于当这个循环被真实用户、真实网络、真实文件系统、真实模型缺陷、真实安全威胁反复捶打之后，你还能让它不犯错、可恢复、可观测、可治理。

读完源码我得到一个核心结论，也是本文的主线：

> **Coding Agent 的本质不是"调用 LLM 的程序"，而是一个以 LLM 为不确定执行引擎的分布式状态机。** Grok Build 的所有架构决策，都是在为"执行引擎会犯错、会中断、会幻觉、会被注入"这个前提兜底。

下面按"企业级 Agent 应用同样会遇到的问题"来组织，每个问题给出 Grok Build 的解法和可迁移的思想。

---

## 一、执行引擎是不可信的：把 LLM 当作会犯错的分布式依赖

### 1.1 模型会陷入死循环，而循环检测是个产品功能

LLM 偶尔会陷入"doom loop"——反复输出相同的 tool call，每次都觉得"这次一定能成"。Naive 实现里这会烧光用户的额度直到超时。Grok Build 给它建了一整个子系统：

- **服务端在 SSE 流里内嵌检查事件**：`response.doom_loop_check` 事件随流下发，报告 `tail_repetition:4@thinking` 这类触发信号（`xai-grok-sampler/src/doom_loop.rs`）
- **客户端有专门的中断与恢复机制**：`DoomLoopSignalCollector` 在流式解码层收集信号，达到一定置信度就**中流 abort**，然后以几乎为零的退避（<251ms jitter）立刻重新采样——因为"循环是采样温度的随机现象，等待没有价值，重新抽一次就是解药"（`retry.rs` 里 `doom_loop_backoff` 的注释原文）
- **恢复预算用尽后解除 abort**：最后一次尝试必须跑完，否则用户什么都拿不到

注意这个设计的层次感：检测在服务端、信号在协议层、决策在 retry 策略层、执行在 actor 层。**它不是"加个 try-catch 重试"，而是贯穿协议、流处理、重试策略三层的完整闭环。**

> **迁移思想**：企业 Agent 应用（客服 Agent、数据分析 Agent、RPA Agent）同样会遇到模型循环。要点：①循环检测应该是**带置信度的信号**，不是布尔值；②恢复策略和常规重试策略**分开建模**（doom loop 的退避逻辑和 5xx 的退避逻辑完全不同）；③必须保留"最后一次尝试跑完"的逃生门，否则恢复机制本身会成为故障源。

### 1.2 重试不是 `retry(3)`，是一张分类决策表

`xai-grok-sampler/src/retry.rs` 的文件头注释就是一份 SRE 文档，我原样摘录它的分类（这是生产系统用真金白银换来的知识）：

| 错误类别 | 策略 | 理由 |
|---|---|---|
| 500/502/503/504/520、连接错误、流中断、空响应 | 重试，上限 15 次（约 6 分钟预算） | 瞬时故障 |
| 429 限流 | 重试上限压到 2 次，遵从 `Retry-After` | "长退避后还是会被限流，不值得烧" |
| 413 / 图像处理错误 | **剥离图片后重试一次**，不占重试预算 | 请求内容问题，不是服务问题 |
| 400/401/403/404/408/422、配置错误 | 立即 Fatal | 客户端错误，重试无意义 |
| 上下文超长 | 立即 Fatal | "确定性失败——重发相同或更大的 payload 永远失败" |
| 服务端 `x-should-retry: false` | 无条件 Fatal，覆盖状态码 | 服务端知道错误是不是请求内容引起的 |

还有两个极细腻的决策：`RetryWithClientRebuild`（首次传输错误时用 HTTP/1.1 重建客户端再试——HTTP/2 的某些中间件问题用降级解决）和 `EmitToSession`（401 不在采样层处理，抛给 session 层做凭证刷新，因为**鉴权状态归 session 所有**）。

整个 `classify_error()` 是**纯函数**——不 sleep、不 log、不做 I/O，只返回 `RetryDecision` 枚举。Actor 拿到决策后自己执行副作用。这就是它能被 100% 单测覆盖的原因。

> **迁移思想**：①把"重试与否、如何重试"建模为**返回决策的纯函数**，副作用留给调用方——可测性天差地别；②区分"传输故障"和"请求内容故障"，后者的修复动作是**修改请求**（剥图片）而不是重发；③给服务端留一个**否决重试的协议通道**（header），让服务端可以在不发版的情况下演进策略；④错误所有权要分层——401 归 session 层，采样层不碰。

---

## 二、状态是真问题：会话不是聊天记录，是可重放的事件流

### 2.1 Leader 崩溃与"unknown session id"之战

Grok Build 支持 leader 模式：一个常驻 leader 进程持有所有 session，多个客户端（TUI、IDE、stdio bridge）通过 Unix socket 连上去。这立刻带来分布式系统的经典问题：**leader 挂了怎么办？**

`main.rs` 里 400 多行的 `StdioReplayState` + `replay_acp_state_after_reconnect` 就是答案。stdio bridge 在转发消息时**旁路缓存**每个 session 的 `initialize` / `session/new` / `session/load` 请求；leader 崩溃重连后，按序重放这些请求重建状态，然后给客户端发 `x.ai/leader_reconnected` 通知。

关键细节全部写在注释里，每条背后都是一个真实 bug：

> "Returning before the `session/load` response is the root cause of the 'unknown session id' failures after a leader crash: the bridge declared the reconnect complete while the new leader was still loading the session, and the client's next `session/prompt` raced (and lost against) the load."

所以重放必须**逐条等待响应**（`replay_request_until_response`），且 `session/load` 的响应之前会先到一大批 replay 通知——这些通知要原样转发给客户端，只有响应本身被吞掉（客户端已经拿过原始响应，不能重复）。多 session 场景下，缓存按 session id 索引、用 Vec 保持首见顺序保证重放确定性——"IDE 客户端一条 bridge 上开多个 session，leader 崩溃必须全部恢复，否则其余 session 下次 prompt 死于 unknown session id"。

### 2.2 Session 是事件溯源系统

`xai-grok-shell/src/session/` 有 100+ 个文件：`persistence.rs`、`replay_events.rs`、`merge.rs`、`fork.rs`、`export.rs`……session 可以持久化、重放、**从中间点 fork 出新会话**、合并、导出。这是事件溯源（event sourcing）模式在 Agent 领域的完整落地：对话历史不是可变状态，而是追加的事件序列，当前状态是 fold 出来的。

> **迁移思想**：①任何长生命周期的 Agent（企业审批 Agent、运维 Agent）都需要把 session 建模为**可重放的事件流**，而不是数据库里被反复 UPDATE 的行；②"崩溃恢复"的正确姿势是**重放意图（请求）而不是重放状态**——缓存的是 `session/load` 请求本身；③**恢复的完成条件是响应到达**，不是消息发出；④多租户/多 session 场景下，恢复要全量且保序。

---

## 三、Prompt 是编译产物，不是字符串拼接

企业级 Agent 应用的 prompt 工程通常会腐化成"到处 format! 拼字符串"。Grok Build 的做法是把 prompt 当作**编译管线**：

```
PromptContext (结构化数据)
  ├── AGENTS.md 文件列表
  ├── skills、personas、memory
  ├── 模板选择 + TemplateOverride
  ├── audience: Primary | Subagent
  └── build_timestamp
        │
        ▼ render(&tool_bridge)
  system_prompt: String  (缓存的编译产物)
```

`Agent` 结构体持有 `prompt_context`（原料）和 `system_prompt`（产物）两份东西。这带来三个直接收益：

1. **可重渲染**：会话中途切换模式（进入 plan mode、切换 agent definition），克隆 context、改几个字段、重新渲染，`ToolBridge` 原样复用（`render_prompt_for_definition`）
2. **可观测**：prompt 的原料是数据，可以 dump、可以 diff、可以进 trace
3. **分受众渲染**：同一个 AGENTS.md，主 agent 拿全量，subagent 拿压缩版；某些 reminder 对 subagent 抑制（`prompt_audience()`）

### 3.1 行为约束靠"提醒策略"，不靠祈祷

模型"应该"用 todo 工具跟踪任务，但它会忘。Grok Build 的解法是 `ReminderPolicy`（`xai-grok-agent/src/system_reminder.rs`）——两种机制，注意它们的参数设计：

- **TodoNudge**：模型连续 3 轮没调 `todo_write`，就在上下文里注入 `<system-reminder>` 提醒；两次提醒之间至少隔 5 轮（防止提醒本身刷屏）
- **TodoGate**（更激进，默认关闭）：如果模型输出了"纯内容消息"（没有 tool call，打算结束 turn）但还有 pending todo，**强制再来一轮**——通过 reminder 注入告诉模型"你还没干完"。关键设计：`max_fires_per_prompt = 2`，**硬上限约束最坏情况下的额外推理成本**。

而且 TodoGate 的默认状态是 `enabled: false`，需要远端设置或 CLI flag 显式开启——**有成本的行为默认关闭，成本上限写死在类型里**。

> **迁移思想**：①把 system prompt 建模为"结构化 context → render → 缓存产物"的管线，支持中途重渲染；②主/子 agent 的 prompt 差异通过 audience 参数化，不要复制粘贴两份模板；③对模型的行为约束（该用工具时要用）做成**带频率上限和成本上限的策略对象**，而不是往 system prompt 里加感叹号；④有推理成本的行为一律默认关闭 + 显式 opt-in。

---

## 四、权限与安全：四层防御，每层都假设上一层会失守

一个能执行任意 shell 命令、读写任意文件的 Agent，其安全模型必须是纵深防御。Grok Build 有四层，每一层独立成立：

### 第 1 层：权限规则引擎（策略层）

`xai-grok-workspace/src/permission/` 是一个完整的规则引擎：`PermissionRule`（tool filter × glob pattern × allow/ask/deny）编译成 `CompiledPolicy`（预编译 glob），评估顺序 **deny > ask > allow**（与规则书写顺序无关——这是防呆设计）。

最精彩的是 **bash 命令的语义分析**（`evaluate_bash_command_policy`）：

- 命令先被拆成管道/链式 segment（`cat x | rm y && curl z` 三段都要过规则）
- `timeout 10 rm -rf /` 这类 wrapper 会被**剥壳**（`unwrap_wrappers`），wrapper 和被包装的程序两种形式都检查
- `bash -c '...'` 会**递归**进入脚本内容评估，深度上限 8 层——"超过合法嵌套深度就 fail closed 到 Ask，不让未评估的脚本运行"
- 无法解析的脚本**fail closed 到 Ask**，而不是放行

注释原文："A script that can't be decomposed fails closed to `Ask` rather than falling through." **解析失败时默认升级，不是默认放行**——这是安全工程的核心直觉。

### 第 2 层：权限模式（模式即策略包）

`defaultMode` 支持 `default / acceptEdits / plan / auto / dontAsk / bypassPermissions`。注意 `FromStr` 的注释："Unknown strings fail FromStr and are treated as Default at the call site (fail-safe) **while still claiming the settings scope so a typo in a more-specific file blocks a looser parent mode**"——拼错的配置不是被忽略，而是**降级到最保守模式**，同时阻断更宽松的上级配置生效。

### 第 3 层：内核级沙箱（假设策略层被绕过）

`xai-grok-sandbox` 用 Linux Landlock / macOS Seatbelt 在**进程启动时**把文件系统访问收敛到 workspace，进程内所有 `tokio::fs` 和子进程全部受约束。网络策略则反过来：agent 进程网络开放（要调 LLM API），**子进程网络按域名策略经 seccomp 阻断**。威胁模型一句话说清：模型生成的命令不可信，agent 进程本身可信。

### 第 4 层：目录信任与签名配置（治理层）

`folder_trust.rs` 实现"这个目录是否被用户信任"的模型；`xai-grok-config/signed_policy.rs` 实现**企业下发的带签名配置**——管理员可以强制最低版本、推送托管配置，客户端验签后才应用。

> **迁移思想**：企业 Agent（尤其是能操作数据库、调用内部 API 的 Agent）可以直接搬这套分层：①规则引擎要**编译期预编译 + deny > ask > allow 的顺序无关语义**；②对 shell/SQL 这类"字符串即程序"的输入，必须做**语义拆解**而不是正则匹配前缀，且**解析失败 fail closed**；③配置解析失败要降级到最保守模式并阻断宽松上级；④应用层权限之外，用 OS 级能力（Landlock/seccomp/容器）做**不依赖应用正确性**的兜底；⑤企业治理靠**签名配置**下发。

---

## 五、架构组织：85 个 crate 不是过度设计，是风险隔离

### 5.1 组合根纪律

`xai-grok-pager-bin`（3000 行 `main.rs`，零业务逻辑）是唯一装配点。启动序列本身就值得企业应用抄作业：panic handler → jemalloc hooks → 子进程分流 → 内存 trace → fd 上限 → 签名策略校验 → 崩溃报告检查 → 构建 tokio runtime → `run_and_shutdown(fut, 2s)`。

**优雅停机不是免费的**：普通 `runtime.drop()` 会在不可取消的阻塞任务上永久 hang，所以他们用 `shutdown_timeout(2s)` 兜底。信号处理区分退出码（Ctrl-C=130、SIGTERM=143、SIGHUP=129），退出前 flush 全部 telemetry。

### 5.2 依赖反转用函数指针，不是 trait object 全家桶

jemalloc 的 unsafe 调用只存在于 bin crate；下游 `xai-grok-shell` 只定义 `HeapProfileHooks { stats, set_prof_active, dump_to_path, ... }` 函数指针结构体，由 bin crate 注入。Leaf crate 不需要依赖 jemalloc，平台特定代码全部上浮到组合根。

### 5.3 闸门函数：规则只有一个家

```rust
/// Central gate for auto-update checks; add new suppression rules here,
/// not at call sites.
fn should_check_for_updates(no_auto_update_flag: bool) -> bool {
    if cfg!(debug_assertions) { return false; }  // debug build 永不更新
    ...
}
```

同样的模式出现在 workspace 命令开关（`GROK_WORKSPACE_COMMAND` env > 远端设置 > Unknown——且 **Unknown 与 Disabled 显式分开**："both fail closed, but Unknown earns an honest message"）、yolo mode 判定等处。

### 5.4 移植代码作为学习材料

`xai-grok-tools/src/implementations/` 同时存在 `codex/`（移植自 OpenAI codex）、`opencode/`（移植自 sst/opencode）和原生 `grok_build/` 三套工具实现（Apache §4(b) 有变更声明）。对学习者而言，这是**同问题域三种工业解法的横向对比库**，别无分号。

> **迁移思想**：①二进制装配和业务逻辑物理分离，下游 crate 只收构造好的 Config；②平台相关/unsafe 代码通过 hooks 结构体从组合根注入，leaf crate 保持纯净；③任何"要不要做某事"的判断收敛为**单一闸门函数**，注释声明"新规则加在这"；④三态开关（Enabled/Disabled/Unknown）比布尔值诚实。

---

## 六、可观测性：三套日志、内存快照、采样归因

- **Telemetry 分层**：sentry（崩溃）、OTel（分布式 trace）、以及三套独立 log layer——`sampling_log`（每次模型调用的完整记录）、`hooks_log`（hook 执行）、`debug_log`（firehose）。Agent 应用的排障高度依赖"那次模型调用的完整输入输出是什么"，所以采样日志是一等公民。
- **401 归因**（`attribution.rs`）：多消费者共享 HTTP 连接时，记录"这个 Bearer token 前缀属于哪个消费者"，鉴权失败时能定位到具体调用方。
- **jemalloc 作为观测基础设施**：长会话恢复后主动 `arena.4096.purge` 释放 retained pages（macOS 会把 MADV_FREE 的页面留在 RSS 里直到系统内存压力）；heap profile 可随时 dump。
- **Inference 延迟百分位统计**（`metrics.rs` 的 `compute_percentiles`）。

> **迁移思想**：Agent 应用的可观测性 ≠ 业务日志 + APM。"每次模型调用的完整请求/响应/耗时/token 数/重试轨迹"是必须单独建设的管道，且采样层错误要能归因到具体调用方。

---

## 七、提炼：可迁移的十条设计原则

把全文收拢成可以直接拿去评审企业级 Agent 应用设计的 checklist：

1. **把 LLM 当作不可信的执行引擎**：循环检测（带置信度信号）、确定性失败识别（上下文超长永不重试）、恢复预算与逃生门。
2. **重试是分类决策表，且是纯函数**：传输故障重发、内容故障改请求、鉴权故障上交所有权层、服务端可一票否决。
3. **Session 即事件流**：可重放、可 fork、可导出；崩溃恢复重放的是请求意图，恢复完成的判据是响应到达。
4. **Prompt 是编译管线**：结构化 context → render → 缓存产物；支持中途重渲染；主/子 agent 用 audience 参数化差异。
5. **行为约束是策略对象**：带频率上限、成本上限，默认关闭，显式 opt-in。
6. **权限纵深防御**：规则引擎（顺序无关 deny>ask>allow）→ 语义拆解（shell/SQL 解析失败 fail closed）→ OS 沙箱（不依赖应用正确性）→ 签名配置（治理）。
7. **配置失败降级到最保守**，且阻断更宽松的上级配置；三态比布尔诚实。
8. **组合根纪律**：装配集中一处，平台代码函数指针注入，闸门函数唯一。
9. **观测先行**：模型调用全量日志、内存/延迟指标、错误归因，都是一等基础设施。
10. **注释记录"为什么不能更简单"**：代码库里每个非直觉决策旁都有成因（很多直接引用修过的 bug）——这是能把系统演化十年的组织能力。

---

## 八、推荐阅读顺序（按投入产出比）

| 顺序 | 文件 | 为什么 |
|---|---|---|
| 1 | `xai-grok-sampler/src/retry.rs` | 856 行，一份完整的生产级重试决策表，注释即文档 |
| 2 | `xai-grok-sampler/src/doom_loop.rs` | 229 行，看"模型缺陷"如何被工程化闭环 |
| 3 | `xai-grok-agent/src/agent.rs` | 296 行，Agent 核心抽象：不可变定义 + 策略对象 |
| 4 | `xai-grok-agent/src/system_reminder.rs` | 127 行，行为约束策略的最小完整样本 |
| 5 | `xai-grok-workspace/src/permission/policy.rs` | 权限规则引擎，重点看 bash 语义拆解 |
| 6 | `xai-grok-pager-bin/src/main.rs`（1150-1300 行附近） | stdio bridge 的崩溃重连重放，分布式状态恢复实战 |
| 7 | `xai-grok-tools/src/implementations/` | codex / opencode / grok_build 三套实现横向对比 |

**结语**：Grok Build 证明了 coding agent 的门槛不在"让模型写代码"，而在让一个有幻觉、会中断、会被注入的概率系统，在真实世界的文件系统和网络上**安全地、可恢复地、可观测地、可治理地**运行。这套问题清单和解题范式——不信任执行引擎、状态可重放、prompt 编译化、权限纵深化——与具体模型无关、与 coding 场景无关，是任何企业级 Agent 应用都绕不开的通用地基。
