# 工作包（WP）清单

每个 WP 都分成：目标 / 改动位置 / 验收（Linux 上能跑的）/ 真机验证（Windows 机器，
仓库主人或本地 AI 执行）。完成标准：**Linux 验收项全部 PASS，并且真机验证步骤写进了
STATUS.md**（真机步骤可以等主人来跑）。

---

## WP0 · 基线与环境探测

**目标**：fork 能在 Linux 和目标 Windows 机器上跑通，并且能识别三家引擎的版本。

- 改动：
  - `knowledge/compat_matrix.yaml`：新增 `aedt."2024.2"`（先标 `best_effort`）、
    `cst."2024"`（新段）。
  - `rfauto doctor`：增加 CST 安装探测（读 `RFAUTO_CST_DIR`），并检查 AEDT 2024 R2
    （读 `ANSYSEM_ROOT242`）和 ADS 2027（读 `RFAUTO_HPEESOF_DIR`）。
  - `docs/env_vars.md`：登记新增的环境变量。
- Linux 验收：
  - 全量单元测试 + ruff + lint-imports 都通过（记录基线耗时）。
  - 新增的 doctor 探测在没有引擎时输出「未安装」，不报错。
- 真机验证：
  - `rfauto doctor` 三家引擎都显示已检测到，版本正确。
  - `rfauto run recipes/wilkinson_pd_v1.yaml --adapter fake` 通过。
  - pyaedt（`>=1.4,<2`）连接 AEDT 2024 R2，冒烟测试：新建工程 → 画一个盒子 → 关闭。
    结果写进 compat_matrix，PASS 后升级为 `verified`。

## WP1 · 统一任务层（HOOKS_SPEC §2、§3、§6）

**目标**：单次 run、批量 campaign、tune、cosim 都以 job 的形式存在，有状态文件、
收件箱和锁。

- 改动：
  - 新增 `service/jobs_notify.py`：订阅 EventBus，维护 `status.json`，在终态时写
    inbox 记录（headline 用模板拼接）。
  - 扩展 `cli/domains/jobs.py`：新增 `wait`、`list --active`；给 `watch` 加 rich 进度条
    （`--json` 的行为保持不变）。
  - 新增 `cli/domains/inbox.py`。
  - 新增 `infra/locks.py`：引擎文件锁和重型求解信号量。
  - campaign executor 在每个条目开始、结束时发出 `PROGRESS` 事件
    （先确认现有代码是否已经发出，已有就复用）。
- Linux 验收：
  - `status.json`、inbox 的 schema 有单元测试；原子写入有并发测试（两个进程同时写）。
  - `jobs wait` 的退出码 0/1/2/3 都有测试。
  - 进度条在 `CliRunner` 下不崩溃；`--json` 输出和改动前完全一致（golden 测试）。
  - 锁：持有者进程已死时锁可以被回收；两个任务抢同一个引擎时，后来的进入 `license_wait`。
- 真机验证：用 fake 跑一个 20 点的 campaign，在另一个终端运行 `rfauto jobs watch`，
  确认进度条正常（附截图或终端输出）。

## WP2 · agent hook + 安装自检（HOOKS_SPEC §4、§5）—— 优先项

**目标**：Claude Code、Codex、dsh（先用通用适配器）都能在任务结束时得知结果；
安装时自动证明 hook 通了。

- 改动：
  - 新增 `service/agent_hooks/{base,claude,codex,generic}.py`。
  - 新增 `cli/domains/hooks.py`（`emit`、`install`、`uninstall`、`selftest`）。
  - 扩展 `mcp_tools/jobs.py`：`wait_job` 上报进度；新增 `inbox_list`、`inbox_ack` 和
    resource（按 AGENTS.md §2 同步 README 计数）。
  - AGENTS.md 加一节「提交仿真后怎么等」：L1 的具体用法。
- Linux 验收：
  - selftest 的 T1–T6 在 CI 上通过（使用伪造的 `HOME` 和配置目录）。
  - install 是幂等的：重复安装不会产生重复的 hook 条目；用户原有配置不被破坏
    （测试：先写入一个带其它 hook 的 settings.json，安装后再卸载，文件和原来逐字节一致）。
  - `emit` 在各种异常情况下都以 0 退出（T6）。
- 真机验证（**每种 AI 都要做**）：
  - `rfauto hooks install --agent claude` → 输出 selftest 表，T1–T6 PASS。
  - `rfauto hooks selftest --agent claude --live` → T7 PASS（或者写明 SKIP 的原因）。
  - Codex 同样做一遍；如果 Codex 没有入站 hook，报告里要如实写明退化方式。
  - 实际体验一次：让 AI 提交一个 fake campaign，然后什么也不说，确认 AI 在任务结束后
    主动汇报了结果。

## WP3 · CST 2024 adapter

**目标**：rfauto 能驱动 CST：打开已有工程 → 设参数 → 求解 → 取 S 参数 → 导出几何 →
取理想耦合矩阵。

- 改动：
  - 新增 `adapters/cst/`：
    - `worker.py`：独立子进程，使用 CST 自带或与之兼容的 Python 运行，import CST 的
      `cst.interface` / `cst.results`。通过 stdin/stdout 的 JSON-RPC 和主进程通信
      （遵守 CONTRIBUTING 第 3 条：外部 EDA 进程隔离）。
    - `adapter.py`：实现 `EMSolverAdapter`，并在 `EMSolverType` 和注册表中登记 `cst`。
    - `protocol.py`：RPC 的消息 schema。
    - `fake_worker.py`：用于测试的假 worker（返回固定的 S 参数）。
  - 用到的 RPC 方法：`open_project`、`list_parameters`、`set_parameters`（之后重建模型）、
    `run_solver(kind=time|frequency)`、`get_sparams`、`export_touchstone`、
    `export_step(+sidecar)`、`get_ideal_coupling_matrix`、`close`。
  - `pyproject.toml` 增加 `cst` extra（如果需要第三方依赖的话）。
- **spike（先做，写成报告放进 STATUS.md）**：
  1. CST 2024 的 Python 库位于 `<CST安装目录>/AMD64/python_cst_libraries`。确认它支持哪些
     Python 版本，能否直接使用 rfauto 的 venv（3.10–3.12）。不行的话，worker 改用 CST
     兼容的解释器启动（用 `RFAUTO_CST_PYTHON` 指定）。
  2. Filter Designer 3D 的耦合矩阵能否用程序读出（参见 COSIM §4）。
  3. 导出 STEP 时实体名和材料信息是否保留。
- Linux 验收：
  - 基于 fake_worker 的 adapter 单元测试：完整生命周期、worker 崩溃时的报错、超时、
    重复 close。
  - RPC schema 有测试。
  - 理想矩阵解析器使用 `tests/fixtures/cst/` 里的样例做测试（样例由真机 spike 产出）。
- 真机验证：拿一个简单的 CST 工程（比如两个耦合谐振器）：改参数 → 时域求解 → 导出的 S
  参数和在 GUI 中手动导出的逐点一致（容差 1e-6）。

## WP4 · HFSS 2024 R2 兼容 + STEP 几何导入

**目标**：HFSS 侧能吃进 CST 导出的几何（COSIM §3 路线 G）并求解。

- 改动：
  - `hfss_adapter` 中处理 2024 R2 的版本差异（以 WP0 冒烟测试的发现为准）。
  - 新增 `adapters/hfss_import.py` 的 STEP + sidecar 流程（文件已经存在，**先读现有实现，
    能复用就扩展**）：导入 → 按 sidecar 指定材料、银层边界、端口、空气盒、扫频。
  - 新增质检门 `geometry_equivalence`（放在 pipeline 的 gates 中）：比较每个实体的体积和
    包围盒（相对误差小于 1e-4）、材料是否一致、端口位置。
- Linux 验收：用 fake 的 HFSS session 测试 sidecar 映射逻辑；几何等价门用合成数据测试
  PASS 和 FAIL 两种情况。
- 真机验证：用 WP3 那个两谐振器工程导出 → 导入 HFSS → 求解 → 与 CST 的 S 参数在通带内
  逐点比较（先只记录差值，不设门限）。

## WP5 · 介质滤波器联调流水线（COSIM 全文）

**前置**：WP3、WP4 完成；仓库主人已确认滤波器形式（README §6 第 1 条）。

- 改动：
  - 新增 `linkage/hfss_cst_link.py`（阶段 S0–S5）。
  - 新增 `service/cm_compare_service.py`、相位加载去除内核、G1–G4 门。
  - round-trip 标定脚本，结果写入 `knowledge/cm_extract_floor.yaml`。
  - 新增 CLI `rfauto cosim dr-filter <recipe>`，以 job 形式运行（自动获得进度和通知）。
  - 新增配方 `recipes/dr_filter_*.yaml`。
  - 报告：矩阵对照表、|ΔM| 热力图、偏差来源诊断（COSIM §6）。
- Linux 验收：
  - 整条流水线用 fake 的 CST 和 HFSS 跑通（fake 返回由 `M_ideal` 计算出的理想响应），
    此时 G4 必须 PASS。
  - 人为给某个耦合加 5% 误差，G4 必须 FAIL，并在偏差最大的条目中准确指出这个元素。
  - round-trip 标定可以复现。
- 真机验证：在仓库主人的第一个介质滤波器工程上端到端跑一次，把报告和四种对比
  （COSIM §6）的数值写进 STATUS.md。

## WP6 · ADS 2027 通道核实（独立）

**目标**：确认 ADS 2027 在本机上 B 档和 M 档的实际可用程度，定下默认通道。

- 改动：
  - 在 2027 上运行现有 `ads_mcp_channel` 的冒烟测试和 `ads_batch` 回归。
  - 判断 M 档（官方 MCP）能否替代 B 档完成电路仿真：上游结论是「官方文档未保证电路仿真
    语义」，需要在本机重新验证。
  - 结果写进 `compat_matrix.yaml`。如果 M 档已足够可靠，再讨论是否改成默认（需要仓库主人同意）。
- 真机验证：同一个 ADS 网表，M 档和 B 档的结果逐点一致。

## WP7 · AI 调谐闭环（介质滤波器）

**前置**：WP5。

- 改动：
  - 新增 `optimization/cm_space_mapping.py`：用有限差分估计雅可比 ∂M/∂dims，然后做
    Newton 或阻尼最小二乘迭代。物理映射来自模板元数据。
  - 以 campaign 的形式运行，每一轮是一个 job，自动带进度和通知。
  - MCP / CLI：`rfauto cosim dr-filter tune <recipe> --max-iter N`。
  - AI 的职责：读每轮的通知 → 决定继续、调整步长阻尼或停止 → 生成说明。
    尺寸数值由内核给出，AI 不直接修改数字。
- Linux 验收：在 fake 上（用解析模型模拟 dims → M 的关系），从 ±5% 偏差出发，
  N 轮以内收敛到 G4 PASS。
- 真机验证：在仓库主人的工程上从人为加的失调状态开始，跑通 3 轮以上并记录收敛曲线。
