# HFSS ↔ CST 联调：介质滤波器耦合矩阵对比（WP3 / WP4 / WP5）

## 1. 工程流程（仓库主人定义）

1. 在 **CST 2024** 中设定介质滤波器结构和指标要求，得到**理想耦合矩阵** `M_ideal`。
2. **HFSS 2024 R2** 把 CST 里的这个结构画出来，并做仿真。
3. 用 HFSS 的仿真曲线（S11/S21）**反提耦合矩阵** `M_hfss`。
4. 对比 `M_hfss` 和 `M_ideal`，输出哪些耦合偏了、偏了多少、可能对应哪个物理尺寸。
5. （WP7）由 AI 编排调谐：改尺寸 → 重新仿真 → 再反提，直到对比通过。

## 2. 流水线拆分

`linkage/hfss_cst_link.py` 按 `linkage/hfss_ads_link.py` 的分阶段结构来写。
每个阶段都可以注入替身函数，方便单元测试；每个阶段的产物都写进 run 目录，
并登记到 manifest。

| 阶段 | 输入 | 输出 | 实现 |
|---|---|---|---|
| S0 规格 | `recipes/dr_filter_*.yaml`：f0、带宽、阶数 N、回波损耗、传输零点、拓扑 | `spec.json` | recipe schema 扩展 |
| S1 理想矩阵 | CST 工程 | `M_ideal.json`（含来源：`cst_fd3d` / `rfauto_cm_core`） | WP3 的 `cst.get_ideal_coupling_matrix` |
| S1' 独立复核 | spec | `M_ref.json` | `core/calc_families/cm_core.coupling_matrix_synthesize_explicit` |
| G1 门：矩阵一致 | `M_ideal` 和 `M_ref` 化成同一种标准型后比较 | PASS/FAIL | 新门 `ideal_matrix_consistency` |
| S2 几何迁移 | CST 工程 + 参数表 | HFSS 工程 | WP4（两条路线，见 §3） |
| G2 门：几何等价 | 两边每个实体的体积、包围盒、材料、端口位置 | PASS/FAIL | 新门 `geometry_equivalence` |
| S3 HFSS 求解 | HFSS 工程 | `hfss.s2p`，端口去嵌到约定参考面 | `hfss_adapter` + `export_touchstone` |
| S4 反提 | `hfss.s2p` | `M_hfss.json` + 拟合残差 | `core/calc_families/cm_extract.coupling_matrix_extract` |
| G3 门：反提可信 | 拟合残差、重建响应与原曲线的偏差 | PASS/FAIL | 复用 cm_extract 的残差判据 |
| S5 对比 | `M_ideal`、`M_hfss` | `cm_compare.json` + 报告中的表格和热力图 | 新 service `cm_compare_service` |
| G4 门：矩阵对比 | 逐元素 |ΔM| 和阈值比较 | PASS/FAIL（并写出偏差最大的前 k 项） | 新门 `coupling_matrix_match` |
| S3'/S4'（可选） | CST 自己也仿真同一结构并反提 | `M_cst.json` | 用来区分偏差来源，见 §6 |

## 3. 几何迁移的两条路线（WP4）

**路线 G（通用，先做）：CST 导出 STEP/SAT，HFSS 导入。**

- CST worker 按当前参数导出 STEP，同时导出一个 sidecar JSON：每个实体的名字、材料
  （εr、tanδ、电导率）、哪些面是银层、端口的位置和类型。
- HFSS 侧 `modeler.import_3d_cad` 导入后，按 sidecar 重新指定材料、银层（有限电导率
  边界）、端口、辐射或空气盒、扫频设置。
- 优点：任意结构都能用，CST 始终是几何的唯一来源。每轮调参都是 CST 改参数 → 导出
  → HFSS 导入求解。
- 风险：导入后的面怎么和 sidecar 对应起来（按实体名或包围盒匹配）；导入的几何在 HFSS
  里不能参数化扫描。

**路线 P（参数化，后做，针对固定拓扑的介质滤波器族）：**

- 在 HFSS 里写一个同名参数的滤波器模板（`models/` 插件），从 CST 读出参数表后用
  `set_variables` 驱动。
- 优点：HFSS 能自己扫参，迁移快。缺点：每种拓扑都要写一个模板。
- 前提：仓库主人确认滤波器形式（见 README §6 第 1 条）。

不论走哪条路线，**都必须通过 G2 几何等价门**才能继续往下走。

## 4. CST 理想矩阵的获取（WP3 spike，需要真机确认）

按优先级尝试：

1. CST Filter Designer 3D 的综合结果，如果能通过 CST Python API 或 VBA 读出来，就直接读。
2. 如果只能在 GUI 中导出：约定由人在 GUI 中导出到 `<工程目录>/rfauto/M_ideal.<ext>`，
   adapter 负责解析。实施者要把 CST 2024 实际导出的格式样例放进
   `tests/fixtures/cst/`（去掉敏感信息），并为解析器写单元测试。
3. 两者都不行时，用 S1' 的 rfauto 内核综合结果作为 `M_ideal`，并在报告里**明确标注
   来源是 `rfauto_cm_core` 而不是 CST**。

无论用哪种方式，G1 都会比较 CST 结果和内核结果（如果两者都有）。这一步顺便也是在
互相检验。

## 5. 反提的已知限制（必须读）

上游 `cm_extract.py` 的注释记录了实测结论，施工时不能忽略：

- **complex 域**会受到端口参考面相位（馈线、探针带来的相位加载）污染。纯时延去嵌
  （`ref_delay_scan`）能让拟合残差过门，但耦合系数仍不可信。
- **magnitude 域**（只用 |S|）天生不受相位影响，上游实测耦合系数逐元素偏差约 2e-2。
  但只用幅值时，**交叉耦合的符号无法唯一确定**。对有传输零点的拓扑，只能比较 |M_ij|，
  或者需要额外信息来确定符号。
- 因此：
  1. HFSS 端口要尽量去嵌到谐振器附近（wave port 的 deembed 距离，或在报告中写明参考面）。
  2. 新增一个**相位加载去除**步骤：用带外的 S11 相位估计并扣除常数加线性项，这是滤波器
     反提的常用做法。实现放在 `core/calc_families/`（确定性内核），并配上出处。
  3. **阈值来源**：G4 的阈值不准拍脑袋。必须先做 round-trip 标定：用 `M_ideal` 算出理想
     响应 → 加上和 HFSS 同等的频点、带宽和噪声 → 反提 → 得到「反提方法自身的误差地板」。
     G4 阈值 = 地板 × 安全系数，标定结果写进 `knowledge/`，并在报告中给出引用。
  4. 对比之前，两个矩阵都要转换成同一种标准型（folded 或用户指定的拓扑），并统一符号约定
     （`cm_core.coupling_matrix_folded` 的 `sign_mode`）。

## 6. 偏差来源诊断（报告必须包含）

`M_hfss` 和 `M_ideal` 对不上时，偏差可能来自以下几种原因，报告要尽量分开：

| 对比 | 能说明什么 |
|---|---|
| `M_ideal` vs `M_ref` | CST 的综合结果和 rfauto 内核是否一致（规格是否理解一致） |
| round-trip 地板 | 反提方法本身的误差有多大 |
| `M_cst`（CST 自己仿真后反提）vs `M_ideal` | 结构本身是否已经实现了理想矩阵（这是设计问题，不是迁移问题） |
| `M_hfss` vs `M_cst` | 几何迁移和两个求解器之间的差异（这是联调真正要看的数字） |

`M_cst` 这一路是可选的，但强烈建议在第一个样例上做：如果没有它，
`M_hfss` vs `M_ideal` 的偏差没法分清是设计没调好还是 HFSS 和 CST 不一致。

## 7. 物理映射（WP7 调谐用）

`cm_compare.json` 中每个耦合元素关联一个物理尺寸（例如 M_ii ↔ 谐振孔深或谐振器高度，
M_i,i+1 ↔ 耦合窗或槽的尺寸）。关联关系写在滤波器模板的元数据里，不由 AI 推测。
WP7 用有限差分估计雅可比矩阵 ∂M/∂dims（每个尺寸做一次扰动仿真），然后做
Newton 或空间映射迭代。每一步建议的尺寸都由内核计算，AI 只负责决定是否继续、
是否加密采样、何时停止。
