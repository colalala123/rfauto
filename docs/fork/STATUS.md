# 施工状态

> 规则：见 `AGENTS.md` §0。状态取值：`TODO` / `IN_PROGRESS (<谁>, <日期>)` /
> `BLOCKED (<原因>)` / `DONE (<日期>)`。验收项必须逐条写 PASS / FAIL / SKIP(原因)。

| WP | 名称 | 前置 | 状态 |
|---|---|---|---|
| WP0 | 基线与环境探测 | — | TODO |
| WP1 | 统一任务层 | WP0 | TODO |
| WP2 | agent hook + 安装自检（优先） | WP1 | TODO |
| WP3 | CST 2024 adapter | WP0；`fil_parser` 部分要等样例 | TODO |
| WP4 | HFSS 2024 R2 + STEP 导入 | WP0 | TODO |
| WP5a | 反提内核补强（纯 Linux） | WP0 | TODO（可立即开工，先用合成拓扑） |
| WP5b | 联调流水线（接引擎） | WP3, WP4, WP5a, 样例文件 | TODO |
| WP6 | ADS 2027 通道核实 | WP0 | TODO |
| WP7 | AI 调谐闭环 | WP5b | TODO |

## 仓库主人的答复（2026-11-09）

- [x] 滤波器形式：腔体（金属腔体内装介质谐振器），4 阶和 8 阶，有传输零点，有交叉耦合
- [x] 理想矩阵来源：CST Filter Designer 3D，可导出 `.fil`，整个理想矩阵也能读取并单独导出
- [x] dsh = DeepSeek Harness（hook 适配方案见 HOOKS_SPEC §4.3）
- [x] 引擎版本：ADS 2027、HFSS 2024 R2、CST 2024，同一台 Windows 机器

## 仍待仓库主人提供（见 README §7）

- [ ] 样例文件：用**示例设计**（不是真实产品）导出的 4 阶、8 阶 `.fil` 和单独矩阵文件，
      放进 `tests/fixtures/cst/`；最好附 CST 的理想响应曲线导出
- [ ] 谐振模式（TE01δ / TM 模 / 其它）
- [ ] 本机三家引擎的安装路径写入环境变量（不进仓库）
- [ ] 本机 Codex、dsh 的版本，以及 dsh 使用的 `cordis.yml` 位置

## 日志

<!-- 按时间倒序追加。每条：日期 / WP / 执行者 / 做了什么 / 验收结果 / 遗留问题 -->

- 2026-11-09 / 方案修订 / Claude（云端）/ 根据仓库主人答复更新方案：联调场景改为腔体介质
  滤波器（4、8 阶，有传输零点和交叉耦合）；反提改为 complex 域 + 相位加载去除 + folded
  标准型判定 + 掩码 LM 物理诊断；WP5 拆成 WP5a（纯 Linux 内核）和 WP5b（接引擎）；
  dsh 的 hook 适配改为走它自带的 Claude Code 兼容桥（依据其官方仓库 2026-11-04 提交中的
  `packages/experimental/hooks-claude-code` 文档）；Codex 的 hook 协议信息同样来自 dsh 的
  兼容桥文档，实施时须对照 Codex 官方文档核实。AGENTS.md 增加"公开仓库不提交真实设计"规则。
  未改动任何代码，也未运行测试（云端环境不允许安装并执行本仓库代码）。
- 2026-11-09 / 方案 / Claude（云端）/ 写入 AGENTS.md、CLAUDE.md、docs/fork/*。
  未改动任何代码。方案依据的上游提交是 `upstream/main` 当时的 HEAD（见本条提交的父提交）。
