# Training App v3 — 实现计划

**Overall Progress:** 100%

## TLDR

将 Training App 从"计划展示器"（v2）重构为"训练引导器"（v3）：动作列表→点击进入大卡片→逐动作专注完成→自动记录顺序。核心变更在训练页交互模型，其他页面（历史/体重/设置）逻辑不变，仅数据格式微调。

## Critical Decisions

- **存储 key**: `tr_app_v2` → `tr_app_v3`，首次启动自动迁移，无弹窗
- **交互模型**: 训练页变成上下两层——上层是列表视图（展示动作名+状态），下层被大卡片覆盖（专注单个动作）
- **卡片布局**: 发力要点 → 备注框（始终可见）→ 组表格+Stepper → 完成/重置按钮
- **已完成动作**: 列表页标 ✓ 不可点击，想看数据去历史页
- **额外动作**: `extra-{timestamp}` ID 格式，默认 60s 计时，切天自动清除
- **归档**: 遍历 `state.logs` 中所有动作（含 extra- 前缀），note 和 orderIndex 写入 history
- **纯有氧日**: 只有 AW 有氧数据也可归档，dayId=0

## Tasks

- [x] 🟩 **Phase 1: 脚手架 & 数据迁移**
  - [x] 🟩 将 `tr_app_v2` → `tr_app_v3`，新增 `dataVersion: 3`
  - [x] 🟩 `loadState()` 增加 v2 自动迁移逻辑：检测到 `tr_app_v2` → 读取并写入 `tr_app_v3` → 清除旧 key
  - [x] 🟩 数据结构更新：`ExerciseLog` 新增 `note: string`、`orderIndex: number`（初始值 `-1`）
  - [x] 🟩 `saveState()` 保存时使用新 key
  - > **Verify**: 打开页面，localStorage 中出现 `tr_app_v3`，旧 `tr_app_v2` 消失

- [x] 🟩 **Phase 2: 列表视图（训练页上半层）**
  - [x] 🟩 改写 `renderTrain()`：白天 Tab + 概览状态条 + 动作名称列表（不含组数据）
  - [x] 🟩 每行三项：名称、靶肌肉·组数、状态圆圈（空心○/橙色⋯/绿色✓）
  - [x] 🟩 已完成的动作（`completed: true`）不可点击
  - [x] 🟩 热身/有氧/拉伸面板：保留 v2.0 的 session-panel 展开式，位于列表下方
  - [x] 🟩 AW 指标 + 归档按钮：保留 v2.0 位置，在列表最底部
  - > **Verify**: 切换 D1/D2/D3，列表只显示动作名+肌肉+状态圆圈，不显示组数据

- [x] 🟩 **Phase 3: 动作大卡片（训练页下半层）**
  - [x] 🟩 点击列表行 → 隐藏列表，展示大卡片（动画或即时切换）
  - [x] 🟩 大卡片布局：顶部 ← 返回 → 动作名+靶肌肉 → 发力要点 → 📝 备注框 → 组表格+Stepper → [× 重置] [✓ 完成此动作]
  - [x] 🟩 备注框：内容即时写入 `state.logs[exId].note`，input 事件触发
  - [x] 🟩 组表格：已完成组标 ✓ + 半透明；当前操作组高亮橙色；未做组显示 `—`
  - [x] 🟩 点组行激活 Stepper（重写 `renderStepper` 适配大卡片）
  - [x] 🟩 勾选组 → 自动触发计时器（复用 v2 的 `triggerRest`/`closeTimer`）
  - [x] 🟩 "✓ 完成此动作" → 设置 `completed: true`，记录 `orderIndex`，退回列表
  - [x] 🟩 "× 重置" → 清除该动作所有组的 done + 清除备注
  - > **Verify**: 点动作名进入大卡片 → 做组/调重/写备注 → 点完成退回 → 列表标记 ✓

- [x] 🟩 **Phase 4: 额外动作**
  - [x] 🟩 列表底部" + 添加额外动作"按钮
  - [x] 🟩 弹出表单：名称、靶肌肉（下拉选择）、组数、默认重量、单位（lb/kg 切换）
  - [x] 🟩 提交后写入 `state.logs`，ID 格式 `extra-{timestamp}`，`orderIndex: -1`
  - [x] 🟩 额外动作行右侧显示 ✕ 按钮（点即删，无确认）
  - [x] 🟩 额外动作默认计时 60s（`getRestSec` 中 fallback）
  - [x] 🟩 切换天数时自动清除所有额外动作
  - > **Verify**: 添加"Glute Press" → 列表出现 → 可点进去做大卡片 → 切天再切回看它消失

- [x] 🟩 **Phase 5: 归档逻辑重写**
  - [x] 🟩 `finishWorkout()` 遍历 `state.logs` **所有**动作（含 DAYS 中的 + extra- 前缀的）
  - [x] 🟩 每个 detail 写入 `note` 和 `orderIndex`
  - [x] 🟩 归档后清除 `state.logs`（全部，不只是 day.exercises）
  - [x] 🟩 归档判断条件：至少一组 done **或** 至少填了一项 AW 有氧数据（纯有氧日）
  - [x] 🟩 纯有氧日 `dayName` 设为"有氧日"，`dayId` 为 0
  - > **Verify**: 做几组+填备注 → 归档 → 历史页显示含备注的数据行

- [x] 🟩 **Phase 6: 历史页展示备注**
  - [x] 🟩 `renderHistory()` 中每个 detail 增加备注显示行（如 `note` 不为空）
  - [x] 🟩 纯有氧日的记录正确显示 🏃 标签
  - > **Verify**: 归档后的历史记录，有备注的动作下方多一行备注文字

- [x] 🟩 **Phase 7: 备份文件兼容**
  - [x] 🟩 `exportJSON()` 导出时使用 `tr_app_v3` 数据结构
  - [x] 🟩 `importJSON()` 导入时识别 `dataVersion` 字段，兼容 v2 和 v3 格式
  - > **Verify**: 导出 → 清空 → 导入 → 数据完整（含 note、orderIndex）

- [x] 🟩 **Phase 8: 核对 & 收尾**
  - [x] 🟩 将桌面 `training_backup_v3_2026-07-06.json` 通过导入测试
  - [x] 🟩 逐一检查：大卡片返回/切换天/归档/备注/纯有氧日
  - [x] 🟩 确认 v2 遗留的无关功能正常：体重追踪、设置、Dock 切换
