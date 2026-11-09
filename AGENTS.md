# AGENTS.md — 本 fork 的 AI 施工须知

> 读者：在本仓库干活的任何 AI 编码助手（Codex、Claude Code 本地版、dsh 等）。
> 本 fork 基于上游 `geer1895/rfauto`（GPL-3.0-only），目标与施工计划见
> [`docs/fork/README.md`](docs/fork/README.md)。**开工前先读它，再读
> [`docs/fork/STATUS.md`](docs/fork/STATUS.md) 看哪个工作包轮到你。**

## 0. 每次开工的固定动作

1. `git fetch upstream && git log --oneline -5 upstream/main`，了解上游最近动了什么。
2. 读 `docs/fork/STATUS.md`，只认领状态为 `TODO` 且前置已 `DONE` 的工作包（WP）。
3. 认领时把该 WP 状态改成 `IN_PROGRESS (<你的名字>, <日期>)` 并单独提交。
4. 收工时更新 `STATUS.md`：做了什么、验收项逐条 PASS/FAIL/SKIP（SKIP 必须写原因）、
   遗留问题。**没跑过的验收项不许写 PASS。**
5. 如果本机装了 rfauto 的 agent hook（见 `docs/fork/HOOKS_SPEC.md`），开工时先跑
   `rfauto inbox --unread`，看看有没有已完成但还没人处理的仿真任务。

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
