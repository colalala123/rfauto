# AGENTS.md — 本 fork 的 AI 施工须知

> 读者：在本仓库干活的任何 AI 编码助手（Codex、Claude Code 本地版、dsh 等）。
>
> - **本仓库**：<https://github.com/colalala123/rfauto>（默认分支 `main`）。你的改动提交并推送到这里，
>   git 远端名是 `origin`。
> - **上游**：<https://github.com/geer1895/rfauto>（GPL-3.0-only）。**只读**：只用来 fetch，
>   不要往上游推送，也不要向上游开 PR。
>
> 目标与施工计划见 [`docs/fork/README.md`](docs/fork/README.md)。**开工前先读它，再读
> [`docs/fork/STATUS.md`](docs/fork/STATUS.md) 看哪个工作包轮到你。**

## 0. 开工流程

按顺序逐条做。每一条都确认成功了再做下一条；某一条失败了，先解决它，解决不了就按 0.3 处理。

### 0.1 第一次接手（每台机器只做一次）

1. 本机还没有这个仓库，就克隆并进入目录：

   ```bash
   git clone https://github.com/colalala123/rfauto
   cd rfauto
   ```

2. 检查有没有 `upstream` 远端：运行 `git remote -v`。如果输出里**没有** `upstream` 开头的行，就添加：

   ```bash
   git remote add upstream https://github.com/geer1895/rfauto
   ```

   再运行一次 `git remote -v`，应该能看到 `origin`（本仓库）和 `upstream`（上游）两个远端。

3. 建虚拟环境并安装开发依赖。Python 必须是 3.10、3.11 或 3.12，先用 `python --version` 确认
   （Windows 上没有 `python` 命令时，可以试 `py -3.12`）：

   ```bash
   python -m venv .venv
   ```

   激活虚拟环境：Windows 用 `.venv\Scripts\activate`，Linux/macOS 用 `source .venv/bin/activate`。然后：

   ```bash
   pip install -e ".[dev]"
   ```

   之后所有命令都在这个虚拟环境里运行。机器上装了多个 Python 时，确认用的是 `.venv` 里的那个。

4. 跑一遍基线测试，记下结果（通过数、失败数、耗时），做 WP0 时写进 STATUS.md：

   ```bash
   python -m pytest tests/unit -q
   ```

   测试有 2 万多条，要跑一段时间。上游自己的记录里也有少量已知失败（见
   `docs/release/unit_gate_public.log`），所以**基线有失败不代表是你弄坏的**。如实记录下来，
   不要为了让基线变绿去改上游的测试或代码。

### 0.2 每次开工

1. `git status` 确认没有未提交的改动，然后 `git pull origin main` 拿到最新代码。
2. `git fetch upstream && git log --oneline -5 upstream/main`，了解上游最近动了什么。
3. 读 `docs/fork/STATUS.md`，只认领状态为 `TODO` 且前置已 `DONE` 的工作包（WP）。
   **一次只做一个 WP**，不要顺手改别的 WP 的内容。
4. 认领时把该 WP 状态改成 `IN_PROGRESS (<你的名字>, <日期>)`，单独提交并推送
   （推送之后，别人才知道这个 WP 有人在做）。
5. 收工时更新 `STATUS.md`：做了什么、验收项逐条 PASS/FAIL/SKIP（SKIP 必须写原因）、
   遗留问题。**没跑过的验收项不许写 PASS。**
6. 如果本机装了 rfauto 的 agent hook（见 `docs/fork/HOOKS_SPEC.md`），开工时先跑
   `rfauto inbox --unread`，看看有没有已完成但还没人处理的仿真任务。

### 0.3 遇到不确定的情况：不要猜

碰到下面任何一种情况，在 STATUS.md 里把这个 WP 标成 `BLOCKED (<卡在哪、需要谁提供什么>)`，
提交并推送，然后**停下来问仓库主人**：

- 需要仓库主人提供文件、路径、账号，或者需要仓库主人做决定；
- 验收项过不了，而且查不出原因；
- 要做的事会违反第 1 节的任何一条规则；
- 文档写的和本机软件的实际行为对不上，而且文档没说以哪个为准。

例外：文档明确写了"以本机官方文档和实测为准"的地方（例如 HOOKS_SPEC §4.2），就按核实结果做，
并把差异写进 STATUS.md。

## 1. 硬性规则（继承上游，不得放松）

1. **确定性数字铁律**：频率、损耗、几何尺寸、耦合矩阵元素等物理数字只能来自
   `src/rfauto/core/` 的确定性内核或求解器输出。AI/agent 代码只做编排与解释。
2. **质检门不许放水**：门 FAIL 就如实报 FAIL。不准为了让测试变绿而放宽阈值；
   阈值只能由标定（round-trip、双引擎对照）推出，并在文档写明来源。
3. **零许可可测**：核心包和单元测试必须在没有 HFSS/CST/ADS 的 Linux 上跑通。
   商业引擎测试用 marker 隔离，在缺引擎时**显式 skip 并写原因**，不能伪装成 pass。
4. **分层依赖**：`cli / mcp_server / ui → service → linkage / optimization / models →
   adapters → pipeline → infra → core`，只许自上而下 import，由
   `lint-imports --config .importlinter` 强制。
5. **新功能先写 service 函数**（JSON 进 JSON 出），CLI / MCP / UI 只做薄壳。
6. **可替换组件 = 基类 + 注册表**：新引擎实现 `EMSolverAdapter` 并注册到
   `EMSolverRegistry`，不要另起炉灶。
7. **外部 EDA 进程隔离**：CST 走独立 worker 子进程（见 WP3）；不要把 CST 的
   Python 库 import 进 rfauto 主进程。
8. **零敏感**：不提交许可证文件、license server 地址、API key、本机绝对路径。
   本机路径一律走环境变量（`RFAUTO_*`），在 `docs/env_vars.md` 登记。
   **本 fork 是公开仓库**（公开仓库的 fork 默认公开），所以也**不得提交真实产品设计数据**：
   CST/HFSS 工程、`.fil` 文件、真实耦合矩阵、仿真或实测 S 参数都不行。
   测试样例一律用专门做的示例设计（dummy）。
9. **Windows 纪律**（HFSS/CST 真机）：工程目录、输出目录、日志一律 ASCII 绝对路径；
   同一 AEDT Desktop 只建一个 Hfss 实例；一次失败赋值即整轮重建会话，绝不带毒续跑。
   详见 `docs/unattended_hfss_automation.md`。

## 2. 提交前必须通过

```bash
python -m pytest tests/unit -q -x            # 全量较慢，可先跑你改动相关的子集，提交前跑全量
ruff check src/ tests/ scripts/              # 必须 0 findings
lint-imports --config .importlinter
python scripts/check_numbers.py              # 新增 CLI 命令 / MCP 工具会改变 README 计数
python scripts/sync_readme_numbers.py --write   # 计数漂移时重跑并同笔提交
```

## 3. fork 专属约定

- **改动尽量增量**：新功能放新模块（例如 `adapters/cst/`、`linkage/hfss_cst_link.py`、
  `service/agent_hooks/`），少改上游已有文件，方便以后 rebase 上游。
- 必须改上游文件时，在提交信息里写 `upstream-touch: <文件>`，便于同步时排查冲突。
- 新增文档放 `docs/fork/`；对外文档用中文即可。
- 目标机器版本（硬约束）：**ADS 2027、HFSS（AEDT）2024 R2、CST Studio Suite 2024**，
  三者在同一台 Windows 机器上。不要为其它版本写分支逻辑，除非 STATUS.md 里有人要求。
- 真机验证由仓库主人在 Windows 机上执行（通常交给本地 Codex）。云端/Linux 上写的
  真机相关代码，必须附带一份「真机验证步骤」写进对应 WP 的 STATUS 条目。

## 4. Git 规则

- 只推送到 `origin`（本仓库）。**不要往 `upstream` 推送，也不要向上游开 PR。**
- 不要强推（不要用 `git push --force`），不要改写已经推送过的历史。
- 提交信息第一行写清楚是哪个 WP、做了什么，例如 `WP1: add jobs wait command`。
- 只提交和当前 WP 有关的文件。不要提交虚拟环境（`.venv/`）、运行结果（`runs/`）、
  本机状态（`.rfauto/`），也不要提交真实设计数据（见第 1 节第 8 条）。
  提交前先用 `git status` 看清楚要提交哪些文件。
- 第 2 节的检查全部通过之后，再提交并推送。
