# RFC-0021: Real-time Execution protocol v2 and transparent runner

- Status: Accepted
- Created: 2026-10-07
- Project: RunSeal
- Amends: RFC-0004, RFC-0006, RFC-0011, RFC-0013
- Preserves: RFC-0012 policy epoch and per-execution cleanup invariants

## 1. Summary and acceptance boundary

This amendment defines runseal.protocol/v2: one Execution lifecycle shared by CLI, JSON-RPC, service, and the narrow MCP adapter, with incremental byte output, interactive input, cancellation, terminal operation, and an explicit control channel. The runseal.policy/v1 contract remains unchanged.

This amendment is accepted as the local implementation contract. Acceptance is a design decision, not an implementation capability claim. This amendment supersedes conflicting protocol, audit, stdin, and service provisions in the amended RFCs. No parallel v1/v2 engine, automatic downgrade, compatibility parser, or silent unsandboxed fallback is specified.

Windows reference delivery requires real sandboxed pipe, stdin, cancellation, PTY, and control conformance. Unsupported or skipped Windows mandatory cases do not satisfy acceptance. Portable backends retain experimental status and fail closed for each unavailable combination; capability promotion requires matching conformance evidence.

This contract excludes remote listeners, automatic system-service installation, cloud tenancy, organizational approval UI, workflow orchestration, database history, arbitrary tool-protocol semantics, arbitrary inherited host handles, and mixed-policy scheduling. Host authorization remains the host responsibility. Execution guarantees cover only processes launched through RunSeal and their descendants.

The normative terms MUST, MUST NOT, SHOULD, and MAY carry their usual requirements meanings. In the detailed Chinese clauses below, 必须/不得 specify MUST/MUST NOT requirements and 可以 specifies MAY. The conformance matrix is mandatory evidence, not a claim of existing support.

## 2. 固定的产品决策

### 2.1 入口与协议版本

- 保留 `runseal exec`、`runseal rpc --stdio`、`runseal service --stdio` 和窄 MCP adapter；不新增独立守护进程安装要求。
- 新的活动 Execution、交互 I/O、订阅和终态字段统一定义为 **`runseal.protocol/v2`**。本 RFC 不在 v1 内静默改变 `execute` 返回和订阅含义。
- `runseal.policy/v1` 保持原义；纯传输参数不混入 SandboxPolicy；凡影响执行权限或资源边界的有效值，仍按既有有效策略哈希规则纳入 `policy_hash`，不得借传输配置绕过策略。
- 按仓库 greenfield 约定更新全部仓内消费者和文档，不维护 v1/v2 双执行引擎、自动降级或兼容 parser。若需公开兼容承诺，必须先另行接受兼容性 RFC。
- CLI plain 模式成为透明字节转发入口；`--json` 保留最终结构化结果；`--events` 输出实时 JSONL。三种模式互斥，共用同一执行引擎。

### 2.2 分工

`protocol/commands -> execution lifecycle -> SandboxBackend -> platform/vendor`。Service 负责连接、索引、订阅和资源归属；Backend 不依赖 Service 类型。低层 Windows enforcement 继续复用 vendored 实现。确有缺失时，先证明需求和限制，再在该低层边界补齐，避免在外层复制 ACL、token、网络 guard 或私有 runner IPC。

一个 Execution 只有一个生命周期 owner。开始、stdin、输出、取消、timeout、自然退出、清理失败和终态发布都汇入这个 owner；不保留第二套“CLI 取消状态”或“service 完成状态”。

## 3. FR-1：实时活动 Execution 与终态

### 请求/返回

`service --stdio` 与 `rpc --stdio` 的 v2 方法和消息语义一致。两者都能在执行期间处理控制请求；区别仅在终态记录保留：service 保留至其有界索引策略淘汰，direct 在最终事件完成发送后释放该 Execution 的索引。两者都保留已提交的审计文件。

`execute` 接受既有 `command`、`cwd`、`policy`、`network`、`env`、`stdin`、`timeout_ms`、`metadata`，并增加本文定义的 `io`。字段仍为 snake_case，方法仍为 lowerCamelCase；`command[0]` 继续要求 path-qualified argv，禁止隐式 shell/PATH 解释。

成功接纳后返回活动描述，不等待子命令退出：

```json
{
  "jsonrpc": "2.0",
  "id": 10,
  "result": {
    "execution_id": "exec_example",
    "session_id": "sess_example",
    "status": "preparing",
    "policy_id": "workspace-write",
    "policy_hash": "sha256:...",
    "policy_epoch": "epoch_example"
  }
}
```

响应中的 `status` 固定为 `preparing`，表示请求被接纳，不承诺子进程已启动。响应前完成 request validation、有效策略/epoch 的原子接纳和 ID 分配；这些操作不得等待旧 Execution 结束。冲突、队列满或不可支持的请求必须拒绝，不能隐藏排队。

接纳前的目录/输入校验、计划编译、策略占用和审计准备不得阻塞已有执行的控制请求。实现 MUST 以独立 owner 持有这些操作及其资源，并将待接纳请求计入同一执行容量约束，不能隐藏排队。controller 确认接纳后冻结时刻；目标仍须等接纳回执交付许可。断线或取消后返回的准备结果不得启动目标。owner 清理到期仍未确认时必须保留 owner、使新增接纳失败关闭，允许已接纳 peer 的控制与清理继续推进。对于已接纳、已知但清理事实无法确认的执行，查询 MUST 返回结构化 `EXECUTION_CLEANUP_FAILED`，不能把它作为未知 ID 返回 `EXECUTION_NOT_FOUND`，也不能伪造成功终态或持久化审计记录。

接纳决定 MUST 由 controller 唯一推进；准备 owner 等待该决定或连接关闭，不能因观察到取消而提前丢弃仍可接收决定的通道。controller 在发布 preparing 回执之前 MUST 确认决定已成功交付到仍有效的 owner 接纳通道；决定通道不可用时不得发布回执或安装启动许可。准备拒绝结果的清理期限从 owner 产生拒绝结果时开始，不能延迟到 controller 收到结果才启动。原期限之后才观察到原生退出不能恢复普通校验响应或撤销清理失败；请求是否已接纳及其持久化审计事实必须保持真实。

此响应必须先于同一 Execution 的事件写出；Backend 初始化或启动失败通过后续事件与 `getExecution` 反映。慢启动不能阻塞协议 reader 或其他 Execution 的控制请求。

### 状态与结果

`execution.started` 仅在 backend 确认命令进程已启动后发布，并先于该执行的输出事件。接纳回执和 setup 开始不是启动证明。没有启动命令进程的失败或取消不得生成 started 事件，其终态 `started_at` 必须为 null；真实启动后的终态时间与 started 事件时间一致。

状态转换固定为：

```text
preparing -> running -> finished
preparing/running -> canceling -> failed
preparing/running -> failed
```

Admission 前的参数错误、策略拒绝、能力缺失和 epoch 冲突直接返回结构化错误；不创建“看似正在执行”的记录。已有审计要求仍适用，不要求失败前必须存在 execution_id。

不新增 `cancelled` 终态。成功取消使用 `status: failed`、`termination_reason: cancelled` 和 `error.code: EXECUTION_CANCELLED`；timeout 使用 `termination_reason: timeout` / `EXECUTION_TIMEOUT`。自然退出使用 `termination_reason: exited`，退出码非零也属于命令已经执行完毕，不能误记为 sandbox setup 失败。

`getExecution` 返回活动状态或最终结果，始终携带相同 execution/session/policy 绑定。最终结果补齐：

- `termination_reason`: `exited | signaled | cancelled | timeout | output_limit | client_disconnected | input_failed | execution_failed | failed_to_start | cleanup_failed | backpressure`。
- `exit_code` 与 `signal` 保留真实退出事实；没有实际启动或无法获得时为 `null`，不能伪造成功退出码。
- `cleanup_complete: boolean`。最终结果表示本次生命周期处理已经结束；只有 `true` 才能对外声称进程范围和资源已清理。
- 已有时间、字节统计、backend、policy binding、sandbox/network summary 和审计定位字段继续存在；保留 setup failure 的公开诊断。
- 失败时包含稳定的 `error.code`，不能要求消费者解析 stderr 判断运行阶段。

RPC 的活动/最终结果不内嵌 stdout/stderr/control 全量 payload，只返回计数、截断信息和可用 event sequence 范围。原始 bytes 通过事件或下文 CLI JSON 的显式有界输出读取，避免终态 JSON 超过单帧上限。

现有 `execution.finished` / `execution.failed` 作为唯一终态事件；v2 增加 `result` 承载完整最终结果。每个已接纳 Execution 恰好一个终态事件，终态后不再发送该执行的 stdout/stderr/control 数据。

终态 `result.latest_seq` 等于终态事件的序号。`earliest_available_seq` 是 transport 在 owner 提交终态前采样的保留范围快照，计算时包含这份终态自身的预算和淘汰效果；该快照随终态一起保持不变，审计与 live/replay 终态不能各自改写它。后续查询返回查询时的当前可用范围，其他 Execution 的输出或保留淘汰可以使当前范围比终态快照更晚。不提供事件重放的入口报告空保留范围，即 earliest 为 latest + 1。

正常结束也要检查本 Execution 的存活子进程并完成清理，不能把顶层程序 exit 当成整个执行范围已清空。清理失败返回 `EXECUTION_CLEANUP_FAILED` / `cleanup_complete:false`，标记共享状态不可复用；禁止继续使用可能受污染的状态处理新执行，直至检查/修复通过。

## 4. FR-2：实时输出、订阅和有界资源

- 后端以增量 bytes 输出 stdout/stderr；不得先 `wait_with_output()` 再批量构造事件。stdout/stderr 每条消息保留标准 `base64:` 编码与各自单调 `stream_offset`。
- v2 为每个 Execution 的事件增加单调 `event_seq`。它是该执行内部顺序，不承诺不同 OS pipe 的原始全局写入顺序，也不跨 service 重启延续。
- `execute` 自动为所属连接订阅该执行的全部实时事件。`subscribeEvents` 对同一连接/Execution 设置或替换唯一订阅，参数是 `execution_id`、可选 `types` 和可选 `after_seq`；不会叠加重复订阅。新增 `unsubscribeEvents({execution_id})` 幂等取消该连接的事件投递，不取消执行。
- 省略 `after_seq` 只投递订阅生效后的新事件；显式 `0` 要求从该 Execution 的第一条事件开始完整重放。先原子捕获切点并连接 live tail，再重放旧事件；每个 `(execution_id,event_seq)` 在一次订阅中只交付一次。
- 请求的历史已淘汰时返回 `EVENT_HISTORY_UNAVAILABLE`，附 `earliest_available_seq` / `latest_seq`；请求未来序号返回 `INVALID_REQUEST`。不能静默跳过历史缺口。
- JSONL 审计默认只保存输出元数据；原始 stdout/stderr/control 只存在于授权连接、有限内存窗口或显式允许的输出文件中。`getAuditEvents`/`tailAudit` 不得把事件缓存中的原始 payload 当成审计内容返回。
- `listExecutions`、`getAuditEvents`、`tailAudit` 的单次响应含 envelope 最多 256 KiB，从当前保留数据的尾部取能完整容纳的最新记录，再按原顺序返回；有记录省略时 `truncated:true`。单条无法完整容纳时用公开-safe 的省略标记和记录 ID，不把半条 JSON 发给调用方。
- `tailAudit` 在本 RFC 中仍为有界快照，不伪装成持续订阅。`listExecutions`、`getAuditEvents`、`tailAudit` 的 v2 返回都带 `truncated`，明确结果是否因保留窗口截断；不接受任意调用方文件路径。
- RPC reader、writer 和执行器独立推进。一个子进程输出很多，不能阻止读取 cancel/close 等控制请求。只有 transport writer 写协议 stdout，禁止多个线程拼接 JSON 半行。
- 原始输出、输入等待队列、control、事件重放缓存、完成记录都必须有独立上限。`max_output_bytes` 是 stdout、stderr、terminal、control 的累计输出总量限额；溢出触发 `OUTPUT_LIMIT_EXCEEDED` 并清理，不能只在无限缓存后截断。
- stdout/stderr 的计数按实际 bytes；二进制、NUL、跨 chunk UTF-8 均无损。pipe 模式不允许隐式编码转换或合并 stdout/stderr。
- CLI 重定向到 pipe 或文件时逐字节转发；原生 Windows Console 是字符展示边界，按连续 UTF-8 解码并使用 Unicode Console 写入，不改变调用方 code page。跨 chunk 的不完整字符等待后续 bytes；非法序列以及最终不完整序列显示为替换字符。事件、审计计数和 JSON 捕获仍按原始 bytes，不按显示字符计数。
- 无损和有界要求覆盖整个 backend/helper 传输链，不能只限制外层 RPC 队列。任何内部输出队列都不得静默丢弃已产生的 bytes；内部消费者落后必须进入受控 backpressure 或结构化失败并清理，不能跳过数据后报告完整输出。输入/control 的内部转发队列也受字节预算约束，不能在外层接受后转入无界队列。
- Backend/helper 的传输断开、协议解码失败和内部诊断必须进入 lifecycle 的结构化失败通道。不得把 RunSeal 诊断伪装为 child stdout/stderr bytes，不得用合成 child exit code 替代无法取得的实际退出事实。
- 慢读端造成消息发送队列达到上限后进入有界 backpressure；无写出进展达到配置宽限（默认 5 秒）则取消该连接拥有的活动执行并记录 `CLIENT_BACKPRESSURE`。不能通过无界排队保住“实时”表象。断开的连接不要求收到结果，但必须完成清理和审计。

`RUNSEAL_BACKPRESSURE_MS` 配置无输出进展宽限，默认 5,000 毫秒，允许 100..60,000 的十进制整数，启动时验证并冻结，由 `getCapabilities.limits.backpressure_ms` 回报。RPC writer、CLI stdout/stderr/terminal/control 和有界 backend 输出交付使用同一配置；RPC 仅在有待发送帧时计时，实际成功写出才重置 writer 进展，不把 enqueue、重试或控制请求当作写出进展。CLI native 输出使用实际写入计数。内部输出队列等待预算不能延长已开始的统一清理期限；该传输设置不改变有效策略哈希，也不是 execution timeout。

stdin/control 是有序 byte stream，公开 write 请求边界不要求保留为 backend 写入边界。实现可在配置 chunk 上限内合并尚未交付的连续输入，以限制单字节请求形成的缓冲区/节点分配；不得合并或修改已交付且尚未 ACK 的数据。pending 字节预算仍包含 in-flight bytes，仅在实际写入 ACK 后释放。拒绝请求必须在缓存变更前检查全部字节，不能部分接纳；EOF 必须等待所有已接纳字节交付。输入缓存不得保留超出数据所需的任意调用方 Vec capacity；此缓存分配规则不替代整个连接的常驻内存验证。

`RUNSEAL_SENDER_BYTES` 配置每连接待发送协议数据的部署预算，默认 8 MiB，允许 5 MiB..64 MiB 的十进制字节数，启动时验证并冻结，`getCapabilities.limits.sender_bytes` 回报有效值。预算包含已编码帧、队列节点计费及 writer 中尚未完成的帧：其中 2 MiB 保留给 controller staging，剩余 writer 预算再保留 1 MiB 给控制帧；事件 enqueue 与继续 poll 必须使用同一派生限制。已出队但尚未写完的帧不能提前释放计费。此传输参数不改变有效策略哈希，不放宽执行输出上限、replay 或其他资源限制；这些计费规则本身不替代全连接常驻内存上界的实际验证。
- 写出进展以实际传输完成为准；一次内部缓冲块分多次写出时，每次已完成的写出均刷新无进展期限，不能等整块确认后才计入，也不能以排队或启动 worker 代替写出事实。

### 有界默认值

这些是本 RFC 的部署默认值，不是实测性能数字。参数必须可配置、启动时验证，并由 `getCapabilities.limits` 回报。所有限制在分配相应资源前执行；内存上界须包含 base64/JSON 膨胀和重复队列，不能只计算 raw bytes。

| 参数 | 默认值 | 超限行为 |
|---|---:|---|
| JSON-RPC 一行最大长度 | 1 MiB | 拒绝超长帧，有限内存排空至换行后继续；无法恢复时关闭连接并清理。 |
| 单个 stream/input/control chunk 解码后大小 | 64 KiB | `INVALID_REQUEST`，不向目标写入任何字节。 |
| 每个执行 stdin/control 各自待写队列 | 256 KiB | `INPUT_BACKPRESSURE`；这次请求不入队，调用方可在消费推进后重试。 |
| 每连接活动 Execution 数 | 8 | `EXECUTION_LIMIT_EXCEEDED`；不排队、不启动。 |
| 单个 Execution 的 output replay 缓存 | 1 MiB | 淘汰最早事件，记录 earliest seq；不能宣称完整历史。 |
| 每连接全体 replay 缓存总量 | 8 MiB | 同上；同时受单执行限额约束。 |
| 每连接待发送协议数据 | 8 MiB | 有界 backpressure；阻塞宽限到期按上文清理。 |
| 单个 Execution 输出总量 | 16 MiB | `OUTPUT_LIMIT_EXCEEDED`；有效策略可进一步收紧。 |
| 完成摘要保留 | 最多 1,024 条且最多 8 MiB | 只淘汰最早终态摘要，不淘汰活动记录；已落盘审计不受影响。 |
| 每连接 audit 查询缓存 | 8 MiB | 只保留脱敏事件；快照明确 `truncated`。 |
| 终止宽限 / 全范围清理等待 | 2 秒 / 10 秒 | 升级终止；仍未清理则 `cleanup_failed`，不假报成功。 |

执行总超时继续由请求与有效策略决定，不对长寿命 stdio helper 擅自增加短超时。策略限额不能被 CLI 或 RPC 的部署参数放宽。

请求通过校验、策略/epoch 接纳后冻结 Execution 的接纳时刻，执行总超时覆盖 preparing 和 running。worker 调度、接纳回执交付闸门、准备阶段的事件交付和计划编译都不得重置这一起点。等待接纳回执交付时仍须推进超时与取消；已接受终止原因后，即使闸门或准备调用随后返回，也不能启动目标。截止检查与独立计时 worker 使用同一接纳时刻，终止原因继续遵守首个接受原因规则。阻塞的原生准备调用及其资源 owner 仍须遵守统一清理期限和失败关闭要求。

每连接活动执行上限通过启动环境 `RUNSEAL_MAX_ACTIVE_EXECUTIONS` 配置，默认 8，允许十进制整数 1..64。进程启动时读取并冻结；无效值在建立协议连接或启动目标前拒绝，不回显配置值。`getCapabilities.limits.max_active_executions` 返回该进程实际采用的值。此项是连接 admission 配额，不改变单个 Execution 的 SandboxPolicy 或策略权限，也不允许客户端通过请求字段改写。

host 全范围清理等待通过启动环境 `RUNSEAL_CLEANUP_TIMEOUT_MS` 配置，默认 10000 毫秒，允许严格十进制整数 100..60000。进程启动时冻结；无效值在资源分配前拒绝，不回显值；`getCapabilities.limits.cleanup_timeout_ms` 回报实际值。该值是 host 等待上界，不延长 SandboxPolicy 的执行总超时。各阶段、回执及审计范围查询与 Drop MUST 复用最早绝对清理截止时间，不得重新启动此预算；backend MAY 施加更早期限，但不得延长已选期限。终止升级宽限的配置与其 backend 行为仍需独立验收，不能由此清理等待配置代替。

Windows runner 的私有请求 MUST 携带 host 冻结的清理预算，且在创建命令进程前校验；不得从命令环境或公开执行参数另选预算。自然退出、连接断开与显式终止沿用该预算及已选最早绝对期限，不能回退到硬编码默认值。缺失、无效预算或私有 IPC 版本不匹配 MUST 失败关闭；诊断不得回显帧内容。此配置贯通仍需真实 sandbox 执行链验收，单独的 runner 解码与原生等待测试不构成完整证明。

replay 保留预算通过启动环境 `RUNSEAL_REPLAY_EXECUTION_BYTES`（默认 1 MiB，允许 64 KiB..64 MiB）和 `RUNSEAL_REPLAY_CONNECTION_BYTES`（默认 8 MiB，允许 64 KiB..256 MiB）配置，值为十进制整数的字节数。连接预算不得小于单执行预算。两者启动时验证并冻结，由 `getCapabilities.limits.replay_execution_bytes` 和 `replay_connection_bytes` 回报；无效值在建立协议连接或启动目标前拒绝且不回显。预算计入内存驻留开销并保持全局最早事件淘汰规则；终态范围快照采用同一预算。它们只控制可重放历史的保留，不改变目标输出上限、SandboxPolicy 或策略哈希。

终态摘要和脱敏审计查询缓存的部署保留上限分别使用以下启动环境，十进制整数、启动验证并冻结，不允许请求覆盖，`getCapabilities.limits` 以对应字段报告实际值：

| 启动环境 | limits 字段 | 默认值 | 范围 |
|---|---|---:|---:|
| `RUNSEAL_COMPLETED_EXECUTIONS` | `completed_executions` | 1,024 | 1..65,536 |
| `RUNSEAL_COMPLETED_EXECUTION_BYTES` | `completed_execution_bytes` | 8 MiB | 64 KiB..256 MiB |
| `RUNSEAL_AUDIT_CACHE_BYTES` | `audit_cache_bytes` | 8 MiB | 64 KiB..256 MiB |

摘要计数和字节预算同时生效，只淘汰最早终态摘要；活动记录不参与该保留预算。审计缓存仅保留脱敏事件，超限淘汰最早缓存事件并报告查询范围不完整，不能删除已落盘审计或改写已提交的终态。以上仅控制宿主查询保留，不改变 SandboxPolicy 或策略哈希。无效值在建立协议连接或启动目标前拒绝且不回显。

stream/input/control 的解码后 chunk 上限通过 `RUNSEAL_STREAM_CHUNK_BYTES` 配置，默认 64 KiB，允许 8 KiB..64 KiB 的十进制字节数；stdin 和 control 各自的待写预算通过 `RUNSEAL_INPUT_PENDING_BYTES` 配置，默认 256 KiB，允许 8 KiB..16 MiB，且不得小于 chunk 上限。启动时验证并冻结，`getCapabilities.limits.stream_chunk_bytes` 和 `input_pending_bytes` 报告实际值。RPC 在解码前检查编码长度，解码后再次检查精确字节数；超限 chunk 不入队、不写入目标。待写预算包括队列和已取出但尚未确认写完的数据；stdin/control 分别计账。输出各流按同一 chunk 上限分片，保持字节和 offset 不变；CLI stdin/control 的原始输入按该上限分片，不把调用方的大段写入当成非法 RPC chunk。这些只控制宿主传输资源，不改变 SandboxPolicy 或策略哈希。无效配置在建立协议连接或启动目标前拒绝且不回显。

JSON-RPC 输入和输出的一行长度上限使用 `RUNSEAL_RPC_FRAME_BYTES`，默认 1 MiB，允许 128 KiB..1 MiB 的十进制字节数，包含行末换行。启动验证并冻结，`getCapabilities.limits.rpc_frame_bytes` 报告实际值。超长输入在有限缓冲中排空至换行、返回解析错误，后续有效请求可继续；无法恢复连接时关闭连接并清理已有 Execution。输出在分配编码帧前计数验证同一上限。列表和审计快照的完整响应采用 `min(256 KiB, rpc_frame_bytes)` 预算（`limits.query_response_bytes` 报告这一派生值），保留最新完整记录，必要时明确报告 `truncated`，不能因默认快照大于收紧后的帧上限而关闭可恢复的连接。此项只控制宿主协议传输，不改变 SandboxPolicy 或策略哈希。无效配置在建立协议连接或启动目标前拒绝且不回显。

单个 Execution 的全部输出（stdout、stderr、terminal、control 合计）通过 `RUNSEAL_MAX_OUTPUT_BYTES` 配置部署上限，默认 16 MiB，允许 1..16,777,216 的十进制字节数，启动验证并冻结；`getCapabilities.limits.max_output_bytes` 回报部署值。归一化后的有效 `SandboxPolicy.resources.max_output_bytes` 始终明确填写：请求未指定时采用部署值，请求指定时取两者较小值，包含请求明确设置为零的情况。canonical JSON、policy hash、explain、admission 回执、首事件到终态及审计均使用这份有效策略，不允许执行阶段再叠加未进入 hash 的隐藏上限。此限制影响实际资源行为，必须进入策略哈希，不能被其他 CLI/RPC 参数放宽。恰好达到上限允许正常完成，多一个字节返回 `OUTPUT_LIMIT_EXCEEDED`，不向调用方交付超出的字节并执行范围清理。

清理期限覆盖执行边界、宿主 I/O 和前端资源；阶段切换、响应帧分片、对端保持管道打开及 owner 的 Drop 均不得续期或阻止检查截止时间。取消、请求超时或 I/O worker 失败后，缺少可信清理确认必须按期返回 `EXECUTION_CLEANUP_FAILED`，保留未完成资源的 owner 和 fail-closed 约束，不能把本地线程结束或对端 EOF 当作整个执行范围已清理的证明。

线程函数返回不等于原生线程及其退出回调完成。Windows worker 的 join/reap 必须先确认原生线程已结束，不能仅凭 Rust `is_finished()` 进入可能阻塞的 join。期限到达仍无法确认时保留 join owner；transport reader/writer 也不能通过丢弃 JoinHandle 脱离该所有权约束。此线程事实仅证明相应 worker 的结束，不代替执行范围、I/O 资源和其他清理项目的确认。

执行超时计时 worker 也属于生命周期 owner，必须在提交终态之前显式停止并确认结束；不能依赖终态提交后的 Drop 清理。计时 worker 未能在同一清理期限内确认结束时，报告 `EXECUTION_CLEANUP_FAILED` 和 `cleanup_complete:false`，保留已确认的原生退出事实及原先接受的终止原因。重复 finish 和 Drop 不得续期，未完成 worker 的 owner 必须保留。

backend worker 必须持有独立的 backend、执行参数和资源 owner，不能用借用宿主栈数据的 scoped worker 迫使生命周期在截止后继续等待。完成结果可以独立交给 owner，但结果交付与 Rust 线程函数返回都不能代替原生 worker 退出确认。截止仍未确认时保留 worker，终态清理失败；已交付的原生退出码或 signal 保留，尚未交付的退出状态维持未知，不从目标已消失或取消请求推断退出码。持有未确认执行的策略占用也必须保持失败关闭约束。

已接纳 Execution 的准备阶段必须以独立 owner 持有目录检查和计划编译所需数据；原生调用未返回时也必须能检查取消与清理截止时间。计划结果交付不能代替准备 worker 原生退出确认。准备正常完成不得提前启动整个 Execution 的清理时钟；准备结束确认有局部期限时，取消后的清理只能沿用更早的截止时间，不能因此续期。到期仍未确认必须保留 worker owner，返回清理失败，且随后返回的计划不得启动目标。接纳前的校验及策略占用仍需独立证明其有界性，不能由该准备阶段证据代替。

策略占用释放中的状态文件 I/O 和持有的同步资源必须由独立 owner 执行，不能在生命周期线程或失败后的 Drop 中产生无界等待。只有执行范围已确认清理，才允许移除本次占用；释放结果已返回但 owner 原生退出尚未确认时，也不得声明清理成功。截止仍未确认时必须先使再接纳失败关闭，再保留未退出 worker 及其资源；随后返回的读取或写入结果不能撤销已提交的清理失败。此释放证据不代替接纳前占用获取的有界性或宿主死亡后的真实资源核验。

## 5. FR-3：stdin、控制通道与持久终端

### 5.1 pipe 与流式 stdin

默认 `io` 为 `{"mode":"pipe"}`。`stdin.empty`、`stdin.bytes`、`stdin.file` 保持 RFC-0011 的数据语义，所有宣称支持的真实 sandboxed backend 都必须实际交付 bytes 后关闭 stdin。

新增 `stdin:{"mode":"stream"}`，搭配：

```json
{"jsonrpc":"2.0","id":11,"method":"writeExecutionInput","params":{"execution_id":"exec_example","stream":"stdin","encoding":"base64","data":"base64:aGVsbG8K"}}
{"jsonrpc":"2.0","id":12,"method":"closeExecutionInput","params":{"execution_id":"exec_example","stream":"stdin"}}
```

- `writeExecutionInput` 成功返回 `accepted_bytes`，表示该块已原子加入有限写队列，不表示子程序已经处理。单连接同一 stream 的接受顺序就是写入顺序，不得部分入队后用错误诱导调用方重复发送。
- `closeExecutionInput` 在之前已接受的写入后关闭该 stream，返回 `closed:true` 表示 EOF 已排队；重复 close 幂等。EOF 排队后拒绝任何新写入，使用 `EXECUTION_INPUT_CLOSED`。
- Execution 仍在 preparing 时允许流式数据入队至上述上限；启动失败时释放队列，返回执行失败，不能把数据交给另一 Execution。
- 已启动执行的输入来源或输入传输失败使用 `EXECUTION_INPUT_FAILED`，其首先被 owner 接受时原因是 `input_failed`；不能误记为启动失败。先前已接受的取消、超时或其他原因保持不变。若已可信确认真实退出和全部资源清理，仍保留退出码并返回 `cleanup_complete:true`；清理无法确认时优先报告 `EXECUTION_CLEANUP_FAILED`，保留最初原因和可获得的退出事实。
- 其他运行管理故障（例如必需审计写入或生命周期事件处理失败）首先被 owner 接受时，原因是 `execution_failed`，保留适用的结构化错误码。已有启动确认或可观察输出后不能将这些故障分类为 `failed_to_start`；缺失或重复的 backend 启动确认属于生命周期契约故障，不得伪造启动事件。已确认启动的 backend 返回错误但未提供可信清理证明时，必须优先报告 `EXECUTION_CLEANUP_FAILED` / `cleanup_complete:false`；缺少退出事实时退出状态保持未知。先前接受的终止原因仍保持不变。
- 对不存在的 Execution 返回 `EXECUTION_NOT_FOUND`；终态执行返回 `EXECUTION_NOT_RUNNING`。bytes/file/empty 模式拒绝额外写入。
- 同连接的 JSON-RPC stdin 永远只用于协议解析，不能直接继承给子进程。RPC `stdin.inherit` 返回 `INVALID_REQUEST`；只有 CLI plain 的显式 `--stdin inherit` 转发调用方 stdin。
- Windows 修复点必须贯通到实际受限进程。不能用 `danger-full-access` 的测试替代 Windows sandboxed stdin 测试。

### 5.2 一个显式双向 control 通道

pipe 模式允许 `io.control:{"mode":"pipe","child_fd":3}`。本次只支持一个名为 `control` 的字节通道，child endpoint 固定为逻辑 fd 3；拒绝其他 fd 值、任意句柄继承和与 PTY 同时启用。

RPC 使用同一个 `writeExecutionInput` / `closeExecutionInput`，`stream` 为 `control`；子进程发出的 control bytes 用 `execution.control` 事件承载，独立 offset、上限与半关闭语义。stdout/stderr 保持分离，不用日志、文件轮询或 TCP 端口代替双向控制通道。

CLI plain 通过 `--control-fd 3` 显式转发调用方提供的同名双向端点，映射到 child fd 3。不存在或不支持的端点必须在子命令启动前返回能力/请求错误。Windows 内部端点映射由 backend/vendored boundary 负责，公开协议不暴露私有 handle、token 或 IPC 细节。

### 5.3 PTY

RPC `io:{"mode":"pty","rows":24,"cols":80}`，仅搭配 `stdin.stream`；CLI plain 使用 `--pty --stdin inherit`。尺寸为 1..1000 的整数。禁止用 pipes 冒充 PTY，禁止在 backend 不支持时静默降级。

新增 `resizeExecution({execution_id,rows,cols})` 与 `signalExecution({execution_id,signal:"interrupt"})`。v2仅支持公开语义 `interrupt`；其他值 `INVALID_REQUEST`，终止整个 Execution 统一走 cancel。

PTY 输入仍使用 `writeExecutionInput(stream:"stdin")`。输出统一为 `execution.terminal` bytes，单独 offset；终态中报告 `stderr_merged:true`，不能声称仍有独立 stderr。`interrupt` 针对当前前台任务，持续 Shell 应可继续工作；cancel 则终止整个 Execution 范围。

PTY 的 EOF 不等价于管道 close：协议 `closeExecutionInput(stream:"stdin")` 对 PTY 返回 `BACKEND_CAPABILITY_MISSING`，不隐式发送 Ctrl-D；需要应用级 EOF 的调用方显式写入相应终端字节。CLI 自身 stdin 结束则视为所属调用结束，执行取消并清理，避免遗留交互 Shell。

禁用 PTY 时，不加载或分配终端资源。平台无法实现的输入、resize、interrupt 或 control 能力必须按请求组合报告和拒绝；Windows 上这些必交项任一未通过则本 RFC 未完成。

## 6. FR-4：取消、timeout、断连与清理

`cancelExecution` 参数为 `execution_id` 和可选 `reason`；v2 reason 只接受 `user_requested`（默认）或 `host_shutdown`，不把任意自由文本写入审计。

`cancelExecution` 接纳后立即返回 `{"execution_id":"exec_example","status":"canceling"}`，不能等进程终止后才处理 cancel。它是取消受理回执；只有终态 `cleanup_complete:true` 证明范围已清理。

- preparing/running 都可以取消。重复取消活动 Execution 幂等，不增加第二次清理；终态执行返回 `EXECUTION_NOT_CANCELLABLE`，未知 ID 返回 `EXECUTION_NOT_FOUND`。
- 每个 Execution 有独立的终止范围，覆盖 Shell 启动的子孙进程。取消 A 不能结束并发 B，不能按系统账户杀进程。
- 取消必须下传到底层等待/捕获 API。对等待 stdin、持续输出、拒绝正常终止、顶层已退出但子进程仍运行等场景，均完成升级终止与等待。
- timeout 与 user cancellation 走同一清理路径；最终原因按 owner 首个接受的终止原因决定。自然退出只有在结果尚未被取消/超时接管时成为 `exited`。任何 cleanup failure 最终优先报告 `cleanup_failed`，另保留 `requested_termination_reason` 说明最初原因。
- 真正的信号退出使用 `termination_reason:signaled`，不把所有非零退出都标记为取消。平台映射如下：

| 事实/操作 | POSIX | Windows reference |
|---|---|---|
| 自然退出 | `exit_code` 为真实退出码，`signal:null`，原因 `exited`。 | `exit_code` 保留可获得的原生退出状态，`signal:null`，原因 `exited`；异常退出状态不能伪造为 POSIX 信号。 |
| OS 信号退出 | `exit_code:null`，`signal` 为实际正整数信号编号，原因 `signaled`。 | 无对应 POSIX 信号事实时 `signal:null`；原因按 lifecycle owner 已接受的终止请求或原生退出事实分类。 |
| PTY `interrupt` | 对当前前台任务交付终端 interrupt 语义，保留 Shell 和 Execution；不得对整个执行范围调用 cancel。 | 通过真实终端/backend 能力交付前台 interrupt 语义，保留 Shell；不能以 kill Shell 模拟。 |
| plain 信号退出 | 清理完成后使 wrapper 以同一信号结束，保留调用方可观察的 signal 事实。 | 保留可获得的原生退出状态，不合成 POSIX 信号。 |

Windows CLI 收到控制台 `Ctrl-C` 或 `Ctrl-Break` 时，将其交给当前 Execution owner 作为主动取消请求；plain/JSON/events 在确认清理后返回外层 130。OS 控制回调只通知 owner，不在回调中执行 backend、写输出或获取生命周期锁。该请求与 timeout/自然退出保持首个接受原因规则；不将原生退出状态伪造成 POSIX signal。最终审计/结果交付期间保留无借用引用的回调保护，避免重复中断直接打断已完成的执行清理；交付结束后恢复控制处理。控制台 close/logoff/shutdown、不可捕获的宿主结束及 portable CLI 信号必须另有清理证据，不能由这两种事件的测试推导。

`signalExecution.signal` 的请求枚举仅有字符串 `interrupt`；结果的 `signal` 则记录 OS 退出事实，二者不是同一字段类型。
- stdin EOF、stdout EPIPE、transport reader/writer 不可恢复错误或宿主进程消失时，取消这个连接拥有的活动 Execution。仅停止订阅不等于宿主断连。
- `disposeSession` 先停止该 session 接纳新的输入，取消并等待其执行清理，再释放 runtime root、synthetic home、proxy lease、审计句柄、订阅；正常返回必须附 `cleanup_complete:true`。有未清理资源时返回 `EXECUTION_CLEANUP_FAILED` 与公开的 execution ID 列表，不返回成功。
- `disposeSession` 的 `released_executions` 计数为此次请求等待完成清理的活动 Execution 数，已经清理的终态摘要不计为活动资源；终态摘要和提交审计仍按各自保留规则查询。释放 session 的订阅及 raw replay 数据后，查询的当前可用事件范围可以为空。无资源的有效 session ID 可幂等返回 `cleanup_complete:true` 和零计数；同一 session 的首次 disposal 尚未完成时，重复请求返回 `INVALID_REQUEST`，不能提前返回清理成功。disposal 达到 10 秒清理截止时间仍无证明时返回 `EXECUTION_CLEANUP_FAILED`，继续保留未完成执行的 owner 和 fail-closed 约束，不以响应超时为资源已清理的证据。
- 已提交审计不删除；终态索引按保留策略处理。服务正常关闭必须等待清理；异常退出必须靠实际 backend 的 parent-death/process-range 机制约束后代，并用 conformance 验证。

## 7. FR-5：CLI 透明 runner

公开入口维持 `runseal exec`，不增加与其并行的第二个执行器。现有 policy/network/cwd/timeout 参数继续有效，追加本文规定的 `--stdin empty|inherit`、`--pty`、`--control-fd 3`。

| 模式 | 标准流 | 外层退出语义 |
|---|---|---|
| plain | 子 stdout/stderr 分流、二进制实时转发；无 banner、进度行、最终摘要或 JSON 污染。默认 stdin empty，显式 inherit 才连接调用方输入。 | 自然退出保留子命令 exit code；POSIX 信号退出在清理后保留可观察的 signal 语义；Windows 保留可获得的退出状态。 |
| `--json` | stdout 恰好一份最终结构化 ExecutionResult 或 error。child output 只放入 `output.stdout` / `output.stderr`，每项为 `{encoding:"base64",data:"base64:...",bytes:N,truncated:false}`；受有效 output 总限额约束。达到总限额则按 OUTPUT_LIMIT_EXCEEDED 返回结构化失败及计数，不构造超限全量 JSON。CLI 最终 JSON 不受 RPC 单行帧上限限制。 | RunSeal 正常得到命令结果时外层 0，子命令成败读 `exit_code`；RunSeal 拒绝/超时/启动或清理失败时外层非零。 |
| `--events` | stdout 是实时 JSONL，child bytes 只能作为事件 payload；一个终态事件携带最终结果。 | 正常完成协议输出时外层 0；RunSeal 失败时非零，仍尽力交付唯一终态/结构化错误。 |

CLI 三种输出模式对 RunSeal 自身的失败统一使用外层 exit 125，timeout 使用 124，主动取消使用 130；JSON/events 的子命令正常结果仍采用外层 0，子命令的退出状态保留在结构化结果中。启动前参数、cwd、policy 或前端初始化失败：plain 写一条公开-safe 的 `[runseal:<CODE>]` stderr 诊断，JSON/events 在可用 stdout 上写一份标准结构化 error，不启动目标。运行失败已有唯一终态事件时，events 不再附加第二份 error/终态。错误输出不可用时有限尽力交付，不重试生成重复消息或回显原始 native 错误。

最终 CLI JSON 的输出尝试一旦开始，写入或输出清理失败只能以非零外层状态结束，不得在部分 JSON 后追加另一份结构化 error、重新发送全量结果或续期清理 deadline。已提交的唯一 Execution 终态保持不变；它不是调用方已收到完整结果的确认。宿主必须同时检查外层退出状态和结构化输出完整性。最终交付使用同一有界输出/清理机制，缺少宿主输出资源清理确认时返回 125，保留未完成 owner，不宣称交付成功。

plain 模式中，RunSeal 自身的启动前拒绝或运行管理失败使用外层 exit 125，并向 stderr 写公开-safe 的 `[runseal:<CODE>]` 诊断；timeout 使用 124，调用方主动取消使用 130。**这些数值和 stderr 前缀不是可信来源标记**：子命令也能输出相同内容或退出相同数值。需要准确分类的宿主必须使用结构化 RPC/JSON，不能靠匹配 prefix 决定放宽权限。

`--stdin inherit`、`--pty`、`--control-fd` 仅用于 plain 模式；与 `--json` / `--events` 同时出现时在执行前拒绝。结构化 PTY/control 使用 RPC。`--json` 不承诺实时显示，不能作为实时验收入口。

RunSeal 的失败诊断不得泄露私有 backend 参数或秘密；不得吞掉子程序 stderr 来隐藏失败。plain 转发 child bytes 不自动提供内容脱敏保证，宿主展示/存档仍需自己的数据策略。

## 8. FR-6：策略、网络、能力与 MCP 边界

### 策略与并发

保留单一活动有效策略 cohort：同策略可并发，要求改变共享执行约束的请求在已有活动执行时返回 `POLICY_TRANSITION_BUSY`。本 RFC 不实现不同 workspace/策略的并发调度，不排队、不自动重试、不杀旧任务腾位置，也不通过多启动 RunSeal 进程绕过 gate。

共享状态与协调锁必须定位同一 backend 安全绑定；调用方的环境变量、runtime home 拼写或路径别名不能将该绑定拆成不同的活动状态。共享状态属于 backend-private protected subpath，命令不得因 workspace 或额外写入根覆盖该路径而修改它。协调锁等待必须有界；无法取得可信状态或协调锁时在 admission 阶段结构化拒绝，不无限阻塞控制面，不创建目标执行或放宽策略。setup/repair 对共享执行约束的修改也必须先经过该绑定的 gate。

策略/epoch 接纳必须先于启动，引用计数覆盖 preparing 到 cleanup 完成；失败路径释放本次持有份额，不能释放其他执行的 guard/proxy。跨进程拒绝和失败后的再接纳都要测试。

接纳时持有的策略占用属于 Execution 清理项目，必须在提交清理成功的终态之前确认释放。占用释放失败或无法在原清理期限内确认时，终态报告 `EXECUTION_CLEANUP_FAILED` 和 `cleanup_complete:false`，保留未确认占用及失败关闭约束。不得在终态提交后依赖忽略错误的 Drop 释放占用，也不得因重复 finish、Drop 或清理阶段切换续期。释放必须只移除当前 owner 的确切占用记录，不得清除其他执行或未确认记录。

执行边界或其他已完成清理阶段尚未提供可信清理确认时，不得释放其接纳占用。缺失确认与明确清理失败均按未确认处理；确认事实必须先传递给占用清理 owner，再提交终态。

缺少有效终态或终态与接纳身份、策略绑定、序号及清理事实不一致时，不能因为移出活动索引、缓存淘汰或会话释放而解除未确认接纳约束。保留原接纳 Execution 的范围，后续新执行按 `EXECUTION_CLEANUP_FAILED` 拒绝；已有执行及控制查询仍可推进。不能将其他 Execution 或不同策略的终态当作当前范围的清理确认。

宿主进程死亡、进程标识复用或无法验证活动记录的所有者均不构成 cleanup 完成的证据。不得自动删除这些记录后接纳同策略或不同策略，也不得由健康执行的释放路径清除其他执行的未验证记录；应保留绑定并结构化报告 `EXECUTION_CLEANUP_FAILED`。只有能证明原执行进程范围、runtime roots 和共享约束已释放的显式修复才能恢复该绑定；普通准入、setup 状态读取或单纯重启不得充当修复。

显式修复的公开入口为 `runseal repair execution-gates [--json] [--accept-unverified-release]`，默认只处理当前机器的安全绑定。修复必须在有界互斥量等待内完成，并同时满足：全部被记录的接纳 owner 均已消失（按进程标识与创建时刻核对）；沙箱身份下不存在仍在运行的进程；每条被记录的 runtime root 均已不存在或经标记校验后删除。任一条件不成立时保持绑定与 quarantine 不变并结构化报告 `EXECUTION_CLEANUP_FAILED`，不产生任何副作用。无法检查的进程 token 与未记录 runtime roots 属于未验证证据，默认同样拒绝；只有显式 `--accept-unverified-release` 才可继续，且结构化报告必须逐项标明未验证内容与无法检查的进程数量。修复不得删除存活 owner 的占用，不得释放其他绑定，也不得在无法取得可信状态或协调锁时报告成功。修复是运维动作而不是准入路径：不得放宽任何执行策略，修复结果不得替代清理成功终态。

已知限制：plain 模式下，若 sandboxed 执行的 stdout/stderr 是句柄保持打开但停止读取的 Console，原生 console 写 worker 无法被取消。目标进程范围、runtime roots 与策略占用仍必须释放，但终态可以报告 `EXECUTION_CLEANUP_FAILED`；此时必须保留原始 `requested_termination_reason`（例如 `backpressure`），且该场景不计入 `timeout`/`backpressure` oracle 的通过集。在该限制被显式修订（把 console 转发改为可取消或有界策略，或把宿主前端输出清理与执行范围清理分离）之前，该组合的 conformance 以 `ignored` 记录，不能按通过计入。

清理失败标记持久化失败不能释放 reservation 或恢复绑定接纳。仍存活的宿主必须保留未完成资源的 owner 和跨进程可观察的 fail-closed 约束；实现使用的协调资源应限制访问，不能由受约束的执行清除。宿主死亡后仍按上述未验证记录规则处理，不能把短期协调资源消失解释为清理完成。

标准 `read-only` profile 延续 RFC-0008 的广泛读取语义，禁止工作目录写入；执行私有 runtime root 仍可写。其归一化读取声明必须与实际 backend 行为一致，自定义策略显式提供的读取范围不得被默认 profile 放宽。

filesystem level 与 network mode 独立。文件权限变化不能隐式关闭既有 `disabled` / `proxy` 约束。请求 `danger-full-access` 与受控 network 的组合若不可落实，返回 `BACKEND_CAPABILITY_MISSING`，不得执行后再声称网络受控。返回值要分别说明文件约束与网络约束的实际状态，不能用单个 `enforced:true` 概括全部安全能力。

审批仍在宿主；RunSeal 接受明确请求、返回 `APPROVAL_REQUIRED`/拒绝，不主动弹 UI，不由模型修改固定部署策略，也不把网络失败自动改成 unmanaged。setup 状态读取无副作用；服务不隐式提权。

### 能力报告

`getCapabilities` 保留 backend/platform/sandbox_levels/network_modes，追加明确的执行能力状态及 `limits`：

```text
streaming_output, active_execution_query, execution_cancel,
stdin_bytes, stdin_file, stdin_stream, transparent_exec,
pty, pty_resize, pty_interrupt, control_channel,
same_policy_concurrency, mixed_policy_concurrency
```

新增 `execution_profiles` 数组，每行包含 `sandbox_level`、`network_mode`、`io_mode` 和 `feature_statuses`，枚举当前 backend 的可请求组合；调用方必须以完整组合做能力决策。能力按 backend、io mode 和实际支持的组合描述；单字段为 supported 不能覆盖同组合中缺失的能力。无法支持的有效请求使用 `BACKEND_CAPABILITY_MISSING`，安装/启动环境暂不可用使用 `BACKEND_UNAVAILABLE`；格式错误使用 `INVALID_REQUEST`。新增稳定错误码和状态必须在 RFC 中先行定义。

更新粗粒度 `features` 时必须与细粒度状态一致，尤其修复 Windows sandboxed stdin 不能消费却报告 supported 的情况。不能通过删掉 capability assertion 让测试转绿。

### MCP

现有窄 MCP adapter 继续只暴露固定策略下的 `exec`，不把 PTY/control/政策提升等新控制面直接交给模型。内部转为消费同一个 Execution lifecycle，正确处理取消与终态，并保留原有参数限制和错误语义。

宿主通过 RunSeal 启动其他 stdio 服务时，只保证这个进程及其后代受有效策略约束；RunSeal 不负责理解任意工具方法，也不覆盖未经过 RunSeal 的宿主文件、网络或其他本地进程。

## 9. FR-7：日志、错误和失败时的安全行为

所有 v2 新错误码、事件字段、method 参数和平台能力先进入 RFC。继续使用 JSON-RPC 标准 envelope 错误分类，domain error 位于 `error.data.code`。

新增或明确的 domain codes：`EXECUTION_CANCELLED`、`EXECUTION_CLEANUP_FAILED`、`EXECUTION_NOT_RUNNING`、`EXECUTION_INPUT_CLOSED`、`INPUT_BACKPRESSURE`、`EXECUTION_LIMIT_EXCEEDED`、`EVENT_HISTORY_UNAVAILABLE`、`CLIENT_BACKPRESSURE`、`CLIENT_DISCONNECTED`。现有 `EXECUTION_NOT_CANCELLABLE`、`POLICY_TRANSITION_BUSY` 等保留其明确含义。

secret canary 必须覆盖 stdin bytes/file/stream、control payload、env 值、metadata、proxy credentials、setup 错误、取消原因。审计中不能出现 raw input、base64 input、control body、认证材料或 backend-private identifiers。

终态信息记录 execution_id、原因、策略绑定、计数、cleanup_complete 和必要的公开诊断。仅把错误文本改成“已取消”不算取消实现。审计写失败时不得伪报审计成功：启动前无法建立必需审计则拒绝，运行中必需审计失败则停止执行并走清理；若磁盘不可写，向可用的结构化通道报告失败，并明确缺少持久记录。

## 10. 验收矩阵：必须验证可观察行为

下列用例都使用真实 RunSeal 二进制和真实进程。不得以源码字符串匹配、mock handler 或仅比较 JSON 字段代替行为证明。新增 regression 必须先在基线复现旧行为，再在候选实现上通过。测试辅助程序只使用临时 workspace、显式 argv 和无害本地操作。

| AC | 测试场景 | 通过条件 |
|---|---|---|
| 01 | 先写 READY，再等输入，最后退出 | 调用方在发输入前已收到 READY；没有任何靠子命令自然结束触发输出的捷径。 |
| 02 | 长执行期间同连接 getVersion/getExecution | 在子命令被测试闸门释放前收到响应，并观察 preparing/running。 |
| 03 | 活动执行取消 | 先拿到 activity/heartbeat 后 cancel；先收到受理回执，再收到唯一取消终态，实际进程范围已清空。 |
| 04 | preparing 阶段取消 | 使用有控制点的测试 backend fixture 阻止启动；取消后释放闸门也不能启动目标命令。另以真实 backend 测启动/取消竞态。 |
| 05 | waiting-input、busy-output、忽略正常终止 | 三种目标都在清理 deadline 内结束；超时只能报告 cleanup_failed，不能假通过。 |
| 06 | 顶层 Shell 退出而后代存活 | 完成/取消后后代 heartbeat 停止，无持续文件写入；不能只检查 wrapper PID。 |
| 07 | 并发 A/B 隔离 | 取消 A 后 B 继续响应与输出；B 的 runtime root、proxy lease 和策略仍有效。 |
| 08 | bytes/file/empty stdin | Windows 实际受限 backend 上精确 echo 二进制并观察 EOF，覆盖空输入、NUL、大输入和跨 chunk 数据。 |
| 09 | 双向 stdio 往返 | 同一子进程至少完成三次 request/response，保持活动，最后显式 close；不依赖批量预灌全部输入。 |
| 10 | close/write 竞态 | 接受顺序确定，先入队 bytes 不丢失；close 后写入稳定拒绝，重复 close 无副作用。 |
| 11 | output 正确性 | stdout/stderr 各自字节与 offset 一致；多字节 UTF-8 切分、非 UTF-8、NUL 不损坏。 |
| 12 | CLI exit/stderr | child exit 0、7、125 及 stderr-only；plain 原样转发，JSON 外层与 child exit 明确分离，不能误判 child 125 是 RunSeal 拒绝。 |
| 13 | CLI pipe 协议透明 | 无 stdout banner/摘要污染；调用方交互输入可达 child，child 提前退出不使 wrapper 卡住。 |
| 14 | PTY | 子程序实际检测到终端；持续 Shell 中 cwd/env 状态保留；resize 可被子程序观察；interrupt 终止前台任务后 Shell 可继续工作；cancel 清空整个范围。 |
| 15 | control channel | fd 3 双向三轮、二进制与半关闭；同时 stdout/stderr 不混流。请求 control+PTY 或未提供端点在启动前拒绝。 |
| 16 | 同策略并发 | 至少两个执行在同一 service 同时活跃，共享绑定且运行资源隔离；单个 cleanup 不撤销共享约束。 |
| 17 | 跨策略/跨 workspace | 同进程和不同 RunSeal 进程均在旧执行活跃时拒绝需要转变共享策略的新执行，返回 POLICY_TRANSITION_BUSY，无副作用。 |
| 18 | drain 后策略接纳 | 旧执行和清理完全结束后，新策略可成功接纳；每个执行的 policy_hash/epoch 从首事件到终态不变。 |
| 19 | 网络约束 | disabled 阻断直接 egress；proxy 只能走受控路径；放宽文件权限不放宽网络，不可支持组合不启动。 |
| 20 | unsubscribe/resubscribe | cursor 重放接 live tail 无缺口无重复；历史缺口显式报错；停止订阅不停止执行。 |
| 21 | 输出/输入/记录资源上限 | 边界值、超一字节、持续小 chunk、大 chunk、多个并发执行；有界内存，无 deadlock，不把 truncated 隐藏成完整结果。 |
| 22 | 慢消费者/断连/宿主消失 | 测试主动停止读、关闭 pipe、终止宿主；受影响的执行清理，另一独立连接/进程不受误伤。 |
| 23 | timeout/cancel/natural-exit 竞态 | 重复运行受控交错；只生成一个终态、一份清理，最终原因与定义一致。 |
| 24 | setup/spawn/cleanup/audit 失败 | 确保原命令是否执行可被测试观察；清理失败不能宣称成功、释放不安全共享状态或继续静默复用。 |
| 25 | 审计与数据安全 | canary 不出现在 JSONL、audit query、错误和摘要中；授权 live output 与默认审计内容保持区分。 |
| 26 | capability 一致性 | 每个 supported 组合都实际成功；不支持组合在执行前失败；不得以 Windows 非沙箱结果代替受限 backend。 |
| 27 | 窄 MCP adapter 回归 | 固定 policy/network 不可由模型放宽；原有请求、失败、取消、资源关闭行为与共享引擎一致。 |
| 28 | plain、RPC、service 对照 | 相同 argv/策略在三入口中的 side effect、child exit 和实际限制一致，仅展示/保留方式不同。 |

时序测试以 READY/ACK、显式释放闸门、进程存活证据和子进程 heartbeat 为主要 oracle，不以固定 `sleep(1.5)` 后比较一组毫秒数作为唯一证据。正常宿主上控制请求响应 watchdog 为 2 秒、清理 watchdog 为上述 10 秒；超时视为失败或带明确基础设施原因的重跑，不能无限等待。

## 11. 验证命令与实现验收条件

实现时保留并更新原有测试入口，新增测试按职责落在现有 `protocol_contract`、`cli_contract`、`filesystem_conformance`、`mcp_contract` 或明确命名的 execution conformance 测试中。禁止用一个“全集通过”替代 AC 对应表。

```text
cargo fmt --check
cargo clippy --tests -- -D warnings
cargo test
git diff --check
```

Windows reference 验收先按仓库脚本构建全部 helpers，再运行：

```powershell
.\scripts\build-windows.ps1
powershell -NoProfile -ExecutionPolicy Bypass -File scripts\windows-smoke.ps1
```

setup/elevation 仅在明确授权的测试主机上执行。portable 验收继续运行 `python3 scripts/portable-probe-smoke.py`，并增加本文中已宣称支持的执行/I/O组合；测试矩阵必须给出 `supported`、`unsupported`、`experimental`、`denied`、`failed`、`skipped` 的真实结果，不能把 skip 算通过。

接受实现验收前必须全部满足：

- [ ] RFC/实现的 exact SHA、协议版本和发布候选版本可追溯；不以 Draft RFC 冒充 Accepted。
- [ ] AC01–28 均有测试位置、命令、平台/模式、预期、实际结果与 evidence；Windows 必交项无 unsupported/skip 豁免。
- [ ] 关键 regression 至少证明一次“基线失败/缺失、候选通过”；不能只证明 JSON 字段变了。
- [ ] Windows reference sandboxed 路径完成 pipe/stdin/cancel/PTY/control 组合验收；portable 的状态宣称与实际证据一致。
- [ ] 同策略并发、不同策略拒绝、无旁路、取消不误杀、宿主退出清理和秘密不入审计全部通过。
- [ ] 第三方消费示例完成命令、长寿命 stdio 服务、PTY、control 和主动取消，不调用私有执行 API。
- [ ] README/README.zh-CN、RFC、帮助文本、capabilities、协议示例、MCP adapter、tests 和 release notes 同步；删除过期的 live cancellation/streaming 宣称。
- [ ] 检查变更中没有私有产品/仓库名称、内部路径、账号、聊天记录、客户信息或 backend-private identifiers。
- [ ] 没有未解决的 P0/P1 安全或数据一致性缺陷；没有隐式权限放宽/自动无沙箱回退。

本 RFC 的完成含义是**在上述明确平台与接口范围内完成可验证的执行集成能力**。它不证明宿主的全部插件、文件访问或网络请求自动进入 RunSeal，也不构成对所有内核逃逸的保证。

## Review decision

Accepted by the repository owner on 2026-10-07 for local implementation. The accepted repository revision SHA must be recorded when the separately authorized change is committed. No commit, push, release, or setup elevation is authorized by this decision. Capability claims still require all applicable conformance evidence.
