# 任务通知与 agent hook 规范（WP1 + WP2a/2b/2c）

目标：rfauto 里的一个仿真任务（单次 run 或一批 campaign）结束后，正在使用本仓库的
AI 能**自己知道**，不需要人去提醒它。批量任务进行中，人和 AI 都能看到进度。
这套机制**不依赖某一家 AI 客户端**。

## 1. 设计原则

1. **单一数据源**：所有通知都来自 `core/events.py` 的 `EventBus` 和它写的
   `runs/<run_id>/events.jsonl`。不另造事件格式。
2. **核心层与 AI 无关**：rfauto 只负责产出「任务状态文件 + 收件箱记录 + 一组 CLI
   命令」。各家 AI 的 hook 都只是一层薄适配，调用同一条命令
   `rfauto hooks emit --agent <name>`。
3. **三层兜底**：L1 阻塞等待（任何能跑 shell 命令的 AI 都能用）→ L2 hook 注入
   （有 hook 机制的 AI 用它，防止漏掉通知）→ L3 MCP（走 MCP 的客户端用）。
   任何一层失效，其它层仍然可用。
4. **不重复投递**：同一个任务结束记录，对同一个 AI 会话只注入一次（游标 + ack 机制）。
5. **Windows 优先**：hook 命令一律写成「venv 里 python 的绝对路径 + `-m rfauto...`」，
   不依赖 bash 或 PowerShell 的语法细节。

## 2. 数据契约（WP1 产出）

### 2.1 任务状态 `runs/<job_or_run_id>/status.json`

任务运行中随时原子覆盖写（先写临时文件再 `os.replace`）。示例：

```json
{
  "schema": "rfauto-job-status/1",
  "job_id": "job_20261110_101500_ab12",
  "kind": "run | campaign | tune | cosim",
  "engine": "hfss | cst | ads | fake | mixed",
  "state": "queued | running | license_wait | done | failed | cancelled",
  "progress": {"done": 23, "total": 60, "failed": 1, "current": "trial 24: w1=1.20 h=3.05"},
  "started_at": "2026-11-10T10:15:00+08:00",
  "updated_at": "2026-11-10T10:41:12+08:00",
  "eta_s": 1850,
  "summary_path": null
}
```

- `progress.total` 对单次 run 取 1。HFSS 的自适应网格 pass、CST 的求解阶段可以
  写进 `progress.current`，属于锦上添花，不强制。
- `eta_s` 只用已完成条目的耗时中位数外推，算不出来就写 `null`，**不准编造**。

### 2.2 收件箱 `<workspace>/.rfauto/inbox/<ts>_<job_id>.json`

任务进入终态（`done/failed/cancelled`）时写一条，原子写入。示例：

```json
{
  "schema": "rfauto-inbox/1",
  "job_id": "...",
  "kind": "campaign",
  "state": "done",
  "finished_at": "...",
  "headline": "腔体滤波器联调 60/60 完成，1 个点失败；最大 |ΔM| = <值>（门限 <值>，来源 cm_extract_floor.yaml）PASS",
  "result_paths": {"report": "runs/.../report.md", "metrics": "runs/.../results/metrics.json"},
  "gates": {"pass": 5, "fail": 1, "failed_names": ["mesh_convergence"]},
  "next_hint": "rfauto report show <job_id>"
}
```

- `headline` 由 service 层根据 metrics 和质检门结果**用模板拼出来**，里面的数字都
  来自 metrics，不由 AI 生成。
- 消费游标：`.rfauto/inbox/.cursor/<agent>__<session>.json` 记录已投递的条目。
  同一会话不重复投递；新会话会收到「未 ack 的全部」。
- `rfauto inbox ack <job_id|--all>` 手动确认已处理；AI 处理完一条就应该 ack。

## 3. CLI 契约（WP1 + WP2a 产出）

| 命令 | 行为 | 退出码 |
|---|---|---|
| `rfauto jobs wait <id> [--timeout S] [--json]` | 阻塞到任务进入终态，输出 inbox 那条记录。**超时不算失败**：输出当前进度并以退出码 3 结束，AI 可以再次调用接着等 | 0 done / 1 failed 或 cancelled / 3 仍在运行（超时）/ 2 找不到该任务 |
| `rfauto jobs watch <id>` | 在现有命令上加 rich 进度条（批量显示 done/total、失败数、当前点、ETA）；`--json` 时保持现有 JSONL 输出不变 | 保持现有语义 |
| `rfauto jobs list --active` | 列出未结束的任务 | 0 |
| `rfauto inbox [--unread] [--json]` | 列出收件箱 | 0 |
| `rfauto inbox ack <id>\|--all` | 标记已处理 | 0 |
| `rfauto hooks emit --agent <a> --event <e>` | **给 hook 调用的入口**：从 stdin 读该 AI 传入的 hook 负载（可能为空），按该 AI 要求的格式向 stdout 输出「未读的任务结束记录 + 正在运行任务的进度」；没有内容时输出空并以 0 退出。**必须很快返回**（目标 1 秒内，硬上限 5 秒）：它在提交提示词的关键路径上，只读本地小文件，不做求解，也不发网络请求 | 0；出错时也必须以 0 退出并输出空，**hook 出错不能卡住 AI**，错误写进 `.rfauto/hooks/hook.log` |
| `rfauto hooks install --agent <a> [--scope project\|user]` | 写入该 AI 的 hook 配置（见 §4），然后**自动运行 selftest** | selftest 不通过则返回非 0，并回滚刚写入的配置 |
| `rfauto hooks selftest [--agent <a>] [--live]` | 见 §5 | 0 全部通过 / 1 有失败 |
| `rfauto hooks uninstall --agent <a>` | 删除 install 写入的配置（只删自己写的那几项） | 0 |

## 4. 各 AI 客户端适配器

适配器放在 `src/rfauto/service/agent_hooks/<agent>.py`，统一接口：

```python
class AgentHookAdapter(Protocol):
    name: str
    def config_targets(self, scope) -> list[Path]          # 会改哪些配置文件
    def install(self, scope, python_exe) -> InstallRecord   # 幂等合并，不覆盖用户已有配置
    def uninstall(self, record) -> None
    def render(self, event, payload) -> str                 # 把 inbox 内容渲染成该 AI 要的 stdout 格式
    def sample_stdin(self, event) -> str                    # 自检用：模拟该 AI 调用 hook 时的 stdin
```

**实施顺序**（仓库主人 2026-11-09 决定）：先 Codex（WP2a，由 Codex 自己实现和核实），
再 Claude Code（WP2b），dsh 以后再做（WP2c）。

### 4.1 Claude Code

- 配置位置：`.claude/settings.json`（project 作用域）或 `~/.claude/settings.json`（user 作用域），
  写入 `hooks` 字段，并且**合并而不是覆盖**。
- 使用的事件：
  - `SessionStart`：注入所有未 ack 的结束记录和正在运行的任务。
  - `UserPromptSubmit`：注入自上次以来新结束的任务。
  - `Stop`（可选，默认关闭，`--block-stop-while-running` 开启）：还有本会话提交的任务
    在运行时，返回 `{"decision":"block","reason":"任务 X 仍在运行（23/60），请后台运行 rfauto jobs wait X"}`，
    防止 AI 提前收工。**必须自我限流**：同一会话对同一个任务最多拦一次（记在游标里）；
    stdin 里 `stop_hook_active` 为 true 时直接放行。原因：dsh 的兼容桥永远传
    `stop_hook_active=false`，也没有连续拦截上限，不自我限流就会死循环（见 4.3）。
- **输出格式统一用 JSON**：`{"hookSpecificOutput":{"hookEventName":"<事件名>","additionalContext":"..."}}`。
  Claude Code 本身也接受纯文本 stdout，但 dsh 的 Claude Code 兼容桥只认 JSON 的
  `additionalContext`（见 4.3）。统一用 JSON，两边都能用。
- **实施时以本机 Claude Code 版本的官方 hooks 文档为准**，写进适配器的注释并在 selftest 里校验。
- L1 用法写进 AGENTS.md：提交任务后，用后台方式运行 `rfauto jobs wait <id>`，
  命令结束时 Claude Code 会自动唤醒 AI。

### 4.2 Codex

**这一节由 Codex 自己实现和核实（WP2a）。** Codex 最了解自己的 hook 机制，下面的信息只是起点。
它们来自 dsh 的 Codex 兼容桥文档（`deepseek-ai/deepseek-harness` 仓库
`packages/experimental/hooks-codex/README.md`，2026-11 版），那份文档逐条对照了 Codex 当时的
hook 协议。实施时一律以本机 Codex 版本的官方 hook 文档（dsh 文档引用的地址是
`https://learn.chatgpt.com/docs/hooks`）和实测结果为准；发现不一致就以核实结果为准，
**回写本节**，并写进适配器注释。

- 事件（10 个）：`SessionStart`、`UserPromptSubmit`、`PreToolUse`、`PostToolUse`、`Stop`、
  `PermissionRequest`、`PreCompact`、`PostCompact`、`SubagentStart`、`SubagentStop`。
  我们只用 `SessionStart`、`UserPromptSubmit`，以及可选的 `Stop`（同样要自我限流）。
- 配置：`hooks.json`（例如项目里的 `.codex/hooks.json`），也有 `config.toml` 内联写法。
  Codex 有用户、项目、会话、系统等多层配置，还有**信任控制**：项目级 hook 可能要用户
  先信任这个项目才生效。安装器要检测这一点；不生效时打印明确的操作提示，selftest 如实报告。
- 输出：`SessionStart` 和 `UserPromptSubmit` 的纯文本 stdout 和 JSON `additionalContext`
  都能用。我们统一输出 JSON。
- 只用同步的 `command` 处理器。Codex 有 `commandWindows` 字段，可以单独写 Windows 下的命令。
  我们的命令本身就是「python 绝对路径 + `-m`」，两个平台通用；如果本机 Codex 在 Windows
  上要求这个字段，就两个都写。
- Codex `~/.codex/config.toml` 里的 `notify` 是一轮结束时**向外**调用程序，方向和我们要的
  相反，不要误用。
- 兜底照样保留：`AGENTS.md` 写死「开工先跑 `rfauto inbox --unread`」，以及 L1 阻塞等待。
  Codex 跑长命令可能有超时，所以用 `jobs wait --timeout 540` 循环调用，靠退出码 3 判断是否继续等。

### 4.3 dsh（DeepSeek Harness）——以后再做（WP2c）

仓库主人 2026-11-09 决定先做 Codex，dsh 以后再做。下面是当时核实的资料，动手前要重新核实。

已核实的情况（`deepseek-ai/deepseek-harness`，2026-11-04 的提交，CLI 版本 `0.2.2-rc.1`；
官方声明处于开发者预览阶段，**会有不兼容变更**）：

- dsh 自己的扩展机制是 cordis 插件，订阅 `agent/session-start`、`agent/pre-step`、
  `agent/turn-stopping` 等拦截点。
- 它另外提供两个**实验性**兼容桥插件，可以原样执行 Claude Code 或 Codex 格式的 hook 配置：
  - `@deepseek-ai/dsh-hooks-claude-code`：支持 `SessionStart`、`UserPromptSubmit`、`PreToolUse`、
    `PostToolUse`、`Stop`、`SubagentStart`、`SubagentStop`。
  - `@deepseek-ai/dsh-hooks-codex`：支持 `SessionStart`、`UserPromptSubmit`、`PreToolUse`、
    `PostToolUse`、`Stop`。
- 挂载方式是在 dsh 的 `cordis.yml` 里加一项：

  ```yaml
  - name: '@deepseek-ai/dsh-hooks-claude-code'
    config:
      configPath: <hooks.json 或带 hooks 键的 settings 文件，写绝对路径>
  ```

**第一版走 Claude Code 兼容桥**，复用 4.1 生成的同一份 Claude Code 格式配置，不另写一套。
兼容桥文档里写明的限制，安装器和 emit 都要照顾到：

1. `configPath` 只在进程启动时读一次，相对路径按启动 dsh 的目录解析 → 一律写**绝对路径**。
2. 不支持热加载 → 安装后**必须重启 dsh** 才生效，安装器要明确提示。
3. `SessionStart` 和 `UserPromptSubmit` **只认 JSON 的 `additionalContext`，不认纯文本
   stdout** → 4.1 统一输出 JSON 就是为了这个。
4. `SessionStart` 异步执行，注入的内容**可能赶不上第一条请求** → 不能只靠 SessionStart，
   `UserPromptSubmit` 必须同时装上。
5. `Stop` 永远传 `stop_hook_active=false`，也没有连续拦截上限 → 见 4.1 的自我限流要求。
6. 只执行 `type: command` 的处理器；默认超时 600 秒（emit 本来就要求 5 秒内返回）。
7. 找不到 dsh 的 `cordis.yml` 时，安装器**不要猜位置**：打印要加的那段 YAML 让人手动加，
   selftest 报告里标明「dsh：配置未自动写入」。

以后可以考虑、但现在不做的：

- 写一个原生 cordis 插件替代兼容桥。能力更强，但要跟着 dsh 的不兼容变更持续维护。
- dsh 有 webhook 子系统，外部请求可以让它**新建**一个会话。理论上能做到「任务结束自动
  开一个 dsh 会话处理结果」，但那是开新会话而不是通知当前会话，暂不采用。

### 4.4 generic（通用）

- 不写任何配置，只生成一段可以粘贴的说明（命令行 + 期望输出）。`render` 输出纯文本。
  所有支持「执行命令并读取 stdout」的 AI 都能这样接入。

### 4.5 MCP（L3）

- 现有 `wait_job` 在等待期间通过 fastmcp 的 `ctx.report_progress(done, total)` 上报进度。
- 新增 resource `rfauto://jobs/{id}`（返回 status.json）和 `rfauto://inbox`。
- 新增工具 `inbox_list` 和 `inbox_ack`。
- 加了 MCP 工具会改变 README 里的计数，需要按 AGENTS.md §2 同步。

## 5. 安装自检（selftest）——「证明 hook 能用」

`rfauto hooks install` 结束时自动运行，也可以单独运行。全部用 fake 引擎，不需要任何
商业许可，总耗时目标小于 60 秒。报告写到 `.rfauto/hooks/selftest_report.json`，
同时在终端打印一张表。

| 编号 | 检查内容 | 通过条件 |
|---|---|---|
| T1 事件链 | 用 fake 跑一次 `recipes/wilkinson_pd_v1.yaml` | `events.jsonl` 里有 `RUN_COMPLETED`；`status.json` 为 `done`；收件箱里出现且只出现一条对应记录 |
| T2 等待语义 | 后台提交一个 fake 任务（人为加 3 秒延迟），立即 `jobs wait` | 任务结束后 2 秒内返回，退出码 0；对一个故意注入失败的任务返回 1；`--timeout 1` 时返回 3 |
| T3 批量进度 | fake 跑一个 5 点的 campaign | `status.json` 的 `progress.done` 单调递增到 5；整批结束后收件箱只有 1 条批次记录（不是 5 条） |
| T4 hook 契约 | **按 install 实际写进配置文件的那条命令原样执行**（从配置文件读出，而不是自己再拼一遍），stdin 喂 `sample_stdin`。stdin 样例按各自协议构造：Claude Code 格式（dsh 走 Claude Code 兼容桥，用同一格式）末尾带换行；Codex 格式为 snake_case、带 `turn_id` 等字段、末尾不带换行（以 4.2 核实结果为准） | 退出码 0；stdout 是合法 JSON，`additionalContext` 字段齐全；内容包含 T1 的 job_id；耗时小于 5 秒 |
| T5 不重复投递 | 同一会话 ID 连续调用两次 T4 | 第二次不再包含 T1 的记录；换一个新会话 ID 再调用，又能拿到 |
| T6 故障隔离 | 把收件箱目录设为只读或写坏一个 json 后调用 hook | 依然以 0 退出、stdout 为空或合法；错误写进 `hook.log` |
| T7 端到端（`--live`，可选） | 在临时目录装好 hook，用该 AI 的无头模式问「有哪些仿真任务完成了？」（例如 `claude -p`、`codex exec`；dsh 先重启，再用它的无头模式，第三方资料写的是 `dsh --profile headless "<提示词>"`。都以本机版本为准） | 回答里包含 T1 的 job_id。需要该 AI 已登录；缺条件就 **SKIP 并写原因，不算 PASS** |
| T8 Stop 自我限流 | 装了 Stop hook 时：同一会话、同一个运行中的任务，连续 10 次用 `stop_hook_active=false` 调用 | 只有第 1 次返回 block，其余全部放行；`stop_hook_active=true` 时始终放行 |

- install 时 T1–T6、T8 任何一项失败：回滚配置，退出码非 0，并打印失败项和日志路径。
- T7 默认不跑，因为会消耗 AI 额度。仓库主人在每台新机器上至少手动跑一次 `--live`，
  把结果记到 STATUS.md。
- selftest 本身也要有单元测试：在 Linux CI 上用伪造的配置目录跑 T1–T6。

## 6. 单机资源协调（WP1 内做）

三家引擎在同一台机器上，需要防止互相抢资源：

- 每个引擎一把文件锁：`.rfauto/locks/<engine>.lock`，记录持有者 PID 和 job_id。
  持有者进程已死时，doctor 可以清理。
- 全局「重型求解」信号量，默认 1（同一时间只跑一个全波求解）。可以通过
  `RFAUTO_HEAVY_SOLVER_SLOTS` 调大。
- 等锁时任务状态为 `license_wait`，并在 `progress.current` 里写明在等谁。
