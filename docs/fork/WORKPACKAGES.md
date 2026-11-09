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
  - `.gitignore` 加上 `.rfauto/`。收件箱、游标、锁、hook 状态都是本机运行状态，不进仓库；
    现在 `.gitignore` 只忽略了 `.rfauto_cache/`，没有忽略 `.rfauto/`。
  - campaign executor 在每个条目开始、结束时发出 `PROGRESS` 事件
    （先确认现有代码是否已经发出，已有就复用）。
- Linux 验收：
  - `status.json`、inbox 的 schema 有单元测试；原子写入有并发测试（两个进程同时写）。
  - `jobs wait` 的退出码 0/1/2/3 都有测试。
  - 进度条在 `CliRunner` 下不崩溃；`--json` 输出和改动前完全一致（golden 测试）。
  - 锁：持有者进程已死时锁可以被回收；两个任务抢同一个引擎时，后来的进入 `license_wait`。
- 真机验证：用 fake 跑一个 20 点的 campaign，在另一个终端运行 `rfauto jobs watch`，
  确认进度条正常（附截图或终端输出）。

## WP2a · 通知层公共部分 + Codex 适配（第一阶段的终点，由 Codex 自己完成）

**目标**：Codex 提交的仿真跑完后，Codex 自己能知道结果；安装时自动证明 hook 通了。

**由 Codex 自己实现、自己核实。** Codex 最清楚自己的 hook 机制。HOOKS_SPEC §4.2 里的信息
只是起点，一律以本机 Codex 版本的官方文档和实测为准；发现不一致就以核实结果为准，
并回写 HOOKS_SPEC §4.2。

- 改动：
  - 新增 `service/agent_hooks/base.py`：适配器接口、游标与 ack、安装记录（用于卸载和回滚）、
    selftest 框架（T1–T8）。后面的 Claude Code、dsh 适配器都复用它。
  - 新增 `service/agent_hooks/codex.py`：写 Codex 的 hook 配置，`render` 输出 JSON `additionalContext`。
  - 新增 `service/agent_hooks/generic.py`：通用适配器（HOOKS_SPEC §4.4）。
  - 新增 `cli/domains/hooks.py`（`emit`、`install`、`uninstall`、`selftest`）。
  - 扩展 `mcp_tools/jobs.py`：`wait_job` 上报进度；新增 `inbox_list`、`inbox_ack` 和
    resource（按 AGENTS.md §2 同步 README 计数）。
  - AGENTS.md 加一节「提交仿真后怎么等」：L1 的具体用法（Codex 用 `jobs wait --timeout 540` 循环等）。
- Linux 验收：
  - selftest 的 T1–T6、T8 在 CI 上通过（使用伪造的 `HOME` 和配置目录）。
  - install 是幂等的：重复安装不会产生重复的 hook 条目；用户原有配置不被破坏
    （测试：先写入一个带其它 hook 的 Codex 配置，安装后再卸载，文件和原来逐字节一致）。
  - `emit` 在各种异常情况下都以 0 退出（T6），且 5 秒内返回。
- 真机验证（Windows，本机 Codex）：
  - `rfauto hooks install --agent codex` → 输出 selftest 表，T1–T6、T8 PASS。注意 Codex 的项目信任设置。
  - `rfauto hooks selftest --agent codex --live` → T7 PASS（或者写明 SKIP 的原因）。
  - 记录 Codex 版本号，以及核实后和 HOOKS_SPEC §4.2 不一致的地方。
  - 实际体验一次：让 Codex 提交一个 fake campaign，然后什么也不说，确认它在任务结束后汇报了
    结果。如实记录通知是从哪条路到达的：L1 等待返回、Stop hook 拦下后去等待，还是下一次提问时
    由 UserPromptSubmit 注入。

## WP2b · Claude Code 适配

**前置**：WP2a。

- 改动：新增 `service/agent_hooks/claude.py`（HOOKS_SPEC §4.1），复用 WP2a 的公共部分。
- Linux 验收：同 WP2a，针对 `.claude/settings.json`。
- 真机验证：`rfauto hooks install --agent claude` → T1–T6、T8 PASS；`--live` 跑 T7；记录 Claude Code 版本。

## WP2c · dsh 适配（以后再做）

**仓库主人 2026-11-09 决定以后再做。** dsh 处于开发者预览阶段，变化快，动手前先重新核实它的
版本和兼容桥文档（HOOKS_SPEC §4.3 记录的是 2026-11-04 的状态）。

- 改动：新增 `service/agent_hooks/dsh.py`，复用 WP2b 的 Claude Code 格式配置，只负责往 dsh 的
  `cordis.yml` 挂兼容桥。
- Linux 验收：对伪造的 `cordis.yml` 做幂等安装、卸载测试；`configPath` 必须是绝对路径。
- 真机验证：安装 → 重启 dsh → T1–T8；记录 dsh 版本。

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
    - `fil_parser.py`：解析 `.fil` 和单独导出的理想矩阵文件，转换成 rfauto 的 N+2 约定，
      并记录归一化方式、符号约定和拓扑掩码（COSIM §4）。**样例到位之前不写。**
      这是纯 Python，放在主进程，不需要 CST。
  - 用到的 RPC 方法：`open_project`、`list_parameters`、`set_parameters`（之后重建模型）、
    `run_solver(kind=time|frequency)`、`get_sparams`、`export_touchstone`、
    `export_step(+sidecar)`、`get_ideal_coupling_matrix`（可选，见 spike 第 2 条）、`close`。
  - `pyproject.toml` 增加 `cst` extra（如果需要第三方依赖的话）。
- **spike（先做，写成报告放进 STATUS.md）**：
  1. CST 2024 的 Python 库位于 `<CST安装目录>/AMD64/python_cst_libraries`。确认它支持哪些
     Python 版本，能否直接使用 rfauto 的 venv（3.10–3.12）。不行的话，worker 改用 CST
     兼容的解释器启动（用 `RFAUTO_CST_PYTHON` 指定）。
  2. 文件导出已确认可行（`.fil` 和单独的矩阵文件）。spike 只需确认：理想矩阵能不能直接用
     API 或 VBA 读出，用来和文件解析结果互相核对。读不出来也不影响主流程。
  3. 导出 STEP 时实体名和材料信息是否保留。
- Linux 验收：
  - 基于 fake_worker 的 adapter 单元测试：完整生命周期、worker 崩溃时的报错、超时、
    重复 close。
  - RPC schema 有测试。
  - `fil_parser` 用 `tests/fixtures/cst/` 里的 4 阶、8 阶样例做测试：矩阵、归一化、符号、
    拓扑掩码都正确；`.fil` 和单独导出的矩阵文件解析结果一致；坏文件给出明确报错。
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

## WP5a · 反提内核补强（纯 Linux，可立即开工）

**目标**：反提链在"4 阶 / 8 阶 + 非对称传输零点 + 交叉耦合"的条件下可靠，并给出 G4 阈值的
标定来源（COSIM §5）。不需要任何引擎，也不需要等样例文件（先用合成拓扑，样例到了再补）。

- 改动（优先放新模块，例如 `core/calc_families/cm_phase_loading.py`、`cm_topology.py`，
  以及 `service/cm_compare_service.py`）：
  1. 相位加载去除内核，登记出处。
  2. 任意拓扑矩阵 → transversal → folded 标准型的统一函数（复用 `cm_core` 的约简），
     以及统一的符号约定。
  3. `cm_refine_lm` 支持 `topology="mask"`（支撑集取初值矩阵的非零模式）。尽量在新模块里
     包装；必须改上游函数时，提交信息写 `upstream-touch`。
  4. 解多重性检测（多起点精修，比较残差和解之间的距离）。
  5. round-trip 标定脚本，结果写入 `knowledge/cm_extract_floor.yaml`。
- Linux 验收：
  - 合成用例至少包括：4 阶 CQ（一个交叉耦合、非对称传输零点）；8 阶、两个以上交叉耦合、
    非对称传输零点。样例到位后加上样例里的真实拓扑。
  - 无噪声 round-trip：反提结果和原矩阵的差接近数值精度（容差参照上游
    `tests/unit/test_coupling_matrix.py` 复往返用例的量级）。
  - 加入已知相位加载后再去除：结果和无相位加载时一致（容差同上）。
  - 两条路线（Cauchy → folded，掩码 LM → folded）结果一致。
  - 加噪声：给出 |ΔM| 随噪声水平、频点数变化的误差地板，写入 knowledge。
  - 多解检测：多起点得到互不相同但残差相当的解时，必须报"解不唯一"。可以用文献中已知
    多解的拓扑（例如 Cameron 书中讨论的 extended box，实施者核对）构造用例。
  - 8 阶用例记录 Cauchy 拟合的条件数。**如果 8 阶过不了，如实写 FAIL 和原因，不准放宽阈值。**

## WP5b · 联调流水线（接引擎）

**前置**：WP3、WP4、WP5a 完成；样例文件已提供（README §7 第 1 条）。

- 改动：
  - 新增 `linkage/hfss_cst_link.py`（COSIM §2 的阶段 S0–S5）。
  - 新增 G1–G4 门；G4 阈值引用 `knowledge/cm_extract_floor.yaml`。
  - 新增 CLI `rfauto cosim dr-filter <recipe>`，以 job 形式运行（自动获得进度和通知）。
  - 新增配方 `recipes/dr_filter_*.yaml`（只放示例设计）。
  - 报告：矩阵对照表、|ΔM| 热力图、解多重性提示、偏差来源诊断（COSIM §6）。
- Linux 验收：
  - 整条流水线用 fake 的 CST 和 HFSS 跑通（fake 返回由 `M_ideal` 计算出的理想响应，
    并加上已知相位加载），此时 G4 必须 PASS。
  - 人为给某个耦合加 5% 误差，G4 必须 FAIL，并在偏差最大的条目中准确指出这个元素。
- 真机验证：在示例设计（4 阶、8 阶各一）上端到端跑一次，把报告和四种对比（COSIM §6）
  的数值写进 STATUS.md。真实产品工程由仓库主人在本机自行运行，结果不进仓库。

## WP6 · ADS 2027 通道核实（独立）

**目标**：确认 ADS 2027 在本机上 B 档和 M 档的实际可用程度，定下默认通道。

- 改动：
  - 在 2027 上运行现有 `ads_mcp_channel` 的冒烟测试和 `ads_batch` 回归。
  - 判断 M 档（官方 MCP）能否替代 B 档完成电路仿真：上游结论是「官方文档未保证电路仿真
    语义」，需要在本机重新验证。
  - 结果写进 `compat_matrix.yaml`。如果 M 档已足够可靠，再讨论是否改成默认（需要仓库主人同意）。
- 真机验证：同一个 ADS 网表，M 档和 B 档的结果逐点一致。

## WP7 · AI 调谐闭环（腔体介质滤波器）

**前置**：WP5b、WP2a（调谐的每一轮都靠任务通知推动）。

- 改动：
  - 新增 `optimization/cm_space_mapping.py`：用有限差分估计雅可比 ∂M/∂参数，然后做
    Newton 或阻尼最小二乘迭代。物理映射来自映射文件（路线 G）或模板元数据（路线 P），
    见 COSIM §7。
  - 以 campaign 的形式运行，每一轮是一个 job，自动带进度和通知。
  - MCP / CLI：`rfauto cosim dr-filter tune <recipe> --max-iter N`。
  - AI 的职责：读每轮的通知 → 决定继续、调整步长阻尼或停止 → 生成说明。
    尺寸数值由内核给出，AI 不直接修改数字。
- Linux 验收：在 fake 上（用解析模型模拟 dims → M 的关系），从 ±5% 偏差出发，
  N 轮以内收敛到 G4 PASS。
- 真机验证：在仓库主人的工程上从人为加的失调状态开始，跑通 3 轮以上并记录收敛曲线。
