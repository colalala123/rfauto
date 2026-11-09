# 施工状态

> 规则：见 `AGENTS.md` §0。状态取值：`TODO` / `IN_PROGRESS (<谁>, <日期>)` /
> `BLOCKED (<原因>)` / `DONE (<日期>)`。验收项必须逐条写 PASS / FAIL / SKIP(原因)。

| WP | 名称 | 前置 | 状态 |
|---|---|---|---|
| WP0 | 基线与环境探测 | — | TODO |
| WP1 | 统一任务层 | WP0 | TODO |
| WP2 | agent hook + 安装自检（优先） | WP1 | TODO |
| WP3 | CST 2024 adapter | WP0 | TODO |
| WP4 | HFSS 2024 R2 + STEP 导入 | WP0 | TODO |
| WP5 | 介质滤波器联调流水线 | WP3, WP4, 主人确认滤波器形式 | BLOCKED（等 README §6 第 1、2 条答复） |
| WP6 | ADS 2027 通道核实 | WP0 | TODO |
| WP7 | AI 调谐闭环 | WP5 | TODO |

## 待仓库主人回答（见 README §6）

- [ ] 介质滤波器的具体形式、阶数、有无交叉耦合 / 传输零点
- [ ] CST 理想耦合矩阵的来源和导出方式
- [ ] dsh 的 hook 机制
- [ ] 本机三家引擎的安装路径已经写入环境变量（不进仓库）

## 日志

<!-- 按时间倒序追加。每条：日期 / WP / 执行者 / 做了什么 / 验收结果 / 遗留问题 -->

- 2026-11-09 / 方案 / Claude（云端）/ 写入 AGENTS.md、CLAUDE.md、docs/fork/*。
  未改动任何代码。方案依据的上游提交是 `upstream/main` 当时的 HEAD（见本条提交的父提交）。
