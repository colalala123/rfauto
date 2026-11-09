# 本 fork 的目标与总体方案

> 上游：`geer1895/rfauto`（GPL-3.0-only）。本 fork 及其改动同样是 GPL-3.0-only。
> 施工分工：方案由云端会话写好；落地编码和真机测试由本地 AI（Codex、
> Claude Code 本地版、dsh 等）按工作包执行。进度看 [`STATUS.md`](STATUS.md)。

## 1. 要解决的问题

让 AI 帮工程师盯住射频仿真里重复的活，并能接上 AI 做优化：

1. **三家引擎都能驱动**：ADS 2027、HFSS 2024 R2、CST Studio Suite 2024，都装在同一台 Windows 机器上。
2. **HFSS ↔ CST 联调**，第一个场景是**腔体介质滤波器**（4 阶和 8 阶，有传输零点和交叉耦合）。
   流程是：CST 里定结构和指标，得到理想耦合矩阵；HFSS 把同一结构画出来仿真；
   用仿真曲线反提耦合矩阵；再和 CST 的理想矩阵对比。详见 [`COSIM_DR_FILTER.md`](COSIM_DR_FILTER.md)。
3. **任务完成主动通知 AI**：单次仿真跑完会通知 AI；批量仿真能看到进度条，整批跑完
   也会通知 AI。通知机制**不绑定某一家 AI**，Claude Code、Codex、dsh（DeepSeek Harness）
   都要能用。**先做 Codex**，由 Codex 自己完成自己的适配；再做 Claude Code；dsh 以后再做。
   安装时自动跑自检，证明 hook 真的通了。详见 [`HOOKS_SPEC.md`](HOOKS_SPEC.md)。
4. 暂不做：推送到手机（飞书 / webhook 等外部推送）。上游已有的
   `campaign_notify_service` 保持原样，不扩展。

## 2. 上游已有、可以直接复用的部分

| 能力 | 位置 | 本 fork 怎么用 |
|---|---|---|
| 引擎基类 + 注册表 | `adapters/em_solver_base.py`（`EMSolverAdapter` / `EMSolverRegistry`，支持 entry-point 插件） | CST adapter 按它实现并注册 |
| HFSS adapter | `adapters/hfss_adapter.py`（`open_or_create_project`、`set_variables`、`export_touchstone`）、`hfss_batch.py`、`hfss_session.py` | 补齐 2024 R2 兼容，并新增 STEP 几何导入流程 |
| ADS 2027 | `linkage/ads_mcp_channel.py`（ADS 2027 官方 `bin/ads-mcp.exe`，M 档）、`ads_batch.py`（B 档 hpeesofsim） | 上游已在 ADS 2027 上冒烟通过；生产默认走 B 档，M 档显式启用 |
| 两阶段联动范式 | `linkage/hfss_ads_link.py` | `hfss_cst_link` 照这个结构写 |
| 耦合矩阵综合 | `core/calc_families/cm_core.py`（广义切比雪夫，Cameron N+2，folded/arrow 约简） | 用来独立复核 CST 给的理想矩阵 |
| 耦合矩阵反提 | `core/calc_families/cm_extract.py`（Cauchy 拟合 + Cameron Y 留数；complex 域支持非对称传输零点；magnitude 域只适用于对称响应）；`cm_diag.py`（`cm_refine_lm`：以给定矩阵为初值做 LM 精修，支持有耗；`cm_extract_vf`：矢量拟合路线） | HFSS 曲线反提的内核。**拓扑和阶数的覆盖范围不够，见 COSIM 文档 §5** |
| TE01δ 介质谐振器 | `core/calc_families/dr_te01d.py` | 用于初值和合理性检查 |
| 事件总线 | `core/events.py`（每个 run 写 `events.jsonl`，有 `RUN_COMPLETED`、`TRIAL_*`、`PROGRESS`、`LICENSE_WAIT` 等事件） | 通知机制的唯一数据源 |
| 异步任务 | `mcp_tools/jobs.py`（`create_run_async`/`poll_job`/`wait_job`）、`cli/domains/jobs.py`（`jobs status/cancel/watch`，`meta.json` 落盘） | 在这上面扩展，不重写 |
| 批量 / 排队 | `service/campaign_manager.py`、`campaign_executor.py`、`campaign_queue_daemon.py`、`pipeline/watchdog.py` | 批量任务就是 campaign，进度复用它的阶段进度 |

## 3. 需要新写的部分

- **CST adapter**：上游完全没有。
- **HFSS 2024 R2 兼容验证**：上游 `knowledge/compat_matrix.yaml` 只验证过 2023 R1 和 2026 R1。
- **CST → HFSS 几何迁移**，加上几何等价质检门。
- **`hfss_cst_link` 联调流水线**和矩阵对比报告。
- **与具体 AI 无关的任务通知层**：任务收件箱（inbox）、`jobs wait`、hook 适配器，以及安装自检。
- **批量任务的统一进度条**：`jobs watch` 目前是轮询打点，还没有进度条。

## 4. 总体架构（新增部分）

```
                ┌──────────── 任意 AI 客户端（Claude Code / Codex / dsh）────────────┐
                │  L1: 后台运行 `rfauto jobs wait <id>` → 命令返回即"被通知"            │
                │  L2: hook（会话开始 / 提交提示词 / 想结束时）→ 调 `rfauto hooks emit` │
                │  L3: MCP 工具 wait_job（带 progress 通知）                            │
                └───────────────▲───────────────────────────────▲──────────────────────┘
                                │ 读                             │ 读
                    .rfauto/inbox/*.json（任务结束记录）   runs/<id>/status.json（进度）
                                ▲                               ▲
                                └──────── service/jobs_notify ──┘  ← 订阅 core/events.EventBus
                                                 ▲
     campaign / run / tune ──→ adapters：fake │ hfss(2024R2) │ cst(2024, worker 子进程) │ ads(2027)
                                                 ▲
                         linkage/hfss_cst_link（腔体介质滤波器联调）
```

## 5. 工作包与顺序

详见 [`WORKPACKAGES.md`](WORKPACKAGES.md)。依赖关系：

| 阶段 | 工作包 | 依赖 |
|---|---|---|
| **第一阶段：Codex 自己完成自己的适配** | WP0 基线 → WP1 任务层 → WP2a 通知层 + Codex 适配 | 依次进行 |
| 可与第一阶段并行 | WP5a 反提内核补强（纯 Linux）、WP3 CST adapter、WP4 HFSS 2024 R2 + STEP 导入、WP6 ADS 2027 通道核实 | 都只依赖 WP0 |
| 之后 | WP2b Claude Code 适配 | WP2a |
| 之后 | WP5b 联调流水线 | WP3、WP4、WP5a、样例文件 |
| 之后 | WP7 AI 调谐闭环 | WP5b、WP2a |
| 以后再做 | WP2c dsh 适配 | WP2b |

第一阶段做完，Codex 提交的仿真跑完后它自己就能知道结果。hook 得有东西可报，所以任务层
（WP1）必须排在 WP2a 前面。WP5a 不需要任何引擎，也不需要等样例文件，可以和第一阶段并行。

## 6. 已确认的事项（2026-11-09，仓库主人答复）

1. **滤波器形式**：腔体（金属腔体内装介质谐振器）。有 **4 阶和 8 阶**两种，
   **有传输零点，也有交叉耦合**。
2. **理想矩阵来源**：CST Filter Designer 3D。可以导出 `.fil` 文件，整个理想矩阵
   也能读取并单独导出。
3. **dsh = DeepSeek Harness**（`deepseek-ai/deepseek-harness`，MIT，开发者预览版）。
   它自带实验性的 Claude Code / Codex hook 兼容桥，适配方案见 HOOKS_SPEC §4.3。**以后再做。**
4. 三家引擎在同一台 Windows 机器上：ADS 2027、HFSS 2024 R2、CST 2024。
5. **优先级**：第一步让 Codex 自己完成自己的 hook 适配（第一阶段，见 §5）。

## 7. 仍待仓库主人提供

1. **样例文件**（WP3 解析器、WP5a 标定都要用）：分别用一个**专门做的示例设计**
   （不是真实产品，因为仓库公开）导出 4 阶、8 阶各一套：`.fil` 文件和单独导出的
   理想矩阵文件，放进 `tests/fixtures/cst/`。最好能带上 CST 里对应的理想响应曲线导出。
2. 谐振模式：TE01δ 还是 TM 模或其它？只影响初值估算和报告说明，不影响主流程。
3. 本机安装路径写进环境变量（不进仓库）：`RFAUTO_CST_DIR`、`ANSYSEM_ROOT242`
   （AEDT 2024 R2 的惯例变量）、`RFAUTO_HPEESOF_DIR`。
4. 本机 Codex 的版本：WP2a 由 Codex 自己记录即可。dsh 的版本和 `cordis.yml` 位置等做 WP2c 时再提供。
