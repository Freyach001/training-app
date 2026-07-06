# Training App · 设计文档

> Change Pro 龙之梦店 · 复健期训练记录工具  
> 版本 2.0 | 单文件 PWA | 2026-06

---

## 1. 项目定位

个人训练辅助工具，解决纸质记录易丢、App 功能臃肿的问题。面向有训练基础、处在复健期的健身者，聚焦于「打开就能记，记完就能看」。

### 核心原则

- **离线优先**：所有数据存本地，不依赖网络
- **操作极简**：单手可完成记录，不打断训练节奏
- **开箱即用**：单个 HTML 文件，Safari 打开即用，无需安装

---

## 2. 功能需求

### 2.1 训练计划查看

- 按三天循环显示训练内容：腿臀+核心 / 背+肩胛+手臂 / 轻训+有氧
- 每个动作列出器械名称、组数、次数、要点
- 嵌入热身、拉伸、注意事项
- 已完成动作在计划列表中标示

### 2.2 训练记录

- 每个动作可记录：重量、完成组数、每组次数
- 单击圆圈标记动作全部完成，单击组勾选框标记单组完成
- 点按组行弹出 stepper 调整重量和次数（±5lb 步进、快捷标签锚定到最近 10lb）
- 发力注意事项始终可见（无 SVG 图示）
- 完成单组后自动触发 60s 组间休息计时器（含振动）
- Apple Watch 数据分两组录入：**无氧训练**（力量）和 **有氧训练**（有氧），各含时长/心率/卡路里
- 保存进度（自动保存）/ 训练结束（标记完成并归档）

### 2.3 历史查看

- 按日期倒序展示所有训练记录
- 每笔记录包含：完成状态、Apple Watch 数据、各动作明细

### 2.4 数据导出

- JSON 导出（完整数据备份）
- PNG 图片导出（当日训练摘要，含 AW 数据）
- TXT 导出（全部历史记录，可分享或存档）

### 2.5 PWA

- 添加至主屏幕后全屏运行，无浏览器 Chrome
- 离线可用（数据完全在 localStorage）
- 部署于 GitHub Pages：https://freyach001.github.io/training-app/

---

## 3. 信息架构

```
底部 Dock 导航
├── 训练（默认）
│   ├── 天数 Tab（第一天/第二天/第三天）
│   ├── 统计概览（当前天数/总组数/已完成）
│   ├── 热身面板（通用+本日专用）
│   ├── 训练动作卡片列表
│   │   ├── 动作名称 + 靶肌肉 + 组数×次数
│   │   ├── 状态圆圈（全部完成/未完成）
│   │   ├── 发力注意事项（始终可见）
│   │   ├── 组表格（重量/次数/勾选框）
│   │   └── Stepper 浮层（点按组行弹出，调重量/次数）
│   ├── 有氧面板（独立，含多项选择）
│   ├── 拉伸面板
│   ├── 复健注意事项
│   └── Apple Watch 指标
│       ├── 🏋️ 无氧训练（时长/心率/卡路里）
│       ├── 🏃 有氧训练（时长/心率/卡路里）
│       └── 完成今日训练并归档
├── 历史
│   └── 记录列表（日期、无氧/有氧 AW 数据、动作明细）
└── 设置
    ├── 导出备份 (JSON)
    ├── 导入备份 (JSON)
    └── 危险区：清空所有数据
```

---

## 4. 数据模型

### 4.1 全局状态

```typescript
interface AppState {
  currentDay: number
  history: HistoryRecord[]
  logs: Record<string, ExerciseLog>
  _today: {             // 今日 Apple Watch 暂存
    dur: string, hr: string, cal: string,
    durC: string, hrC: string, calC: string
  }
}
// localStorage key: "tr_app_v2"
```

### 4.2 每笔历史记录

```typescript
interface HistoryRecord {
  date: string           // "YYYY-MM-DD"
  dayName: string        // "第一天 · 腿臀+核心"
  dur: number            // 无氧时长 (min)
  hr: number             // 无氧心率 (bpm)
  cal: number            // 无氧卡路里 (kcal)
  durC: number           // 有氧时长
  hrC: number            // 有氧心率
  calC: number           // 有氧卡路里
  totalSets: number      // 完成的组数
  details: { name: string, summary: string }[]
  timestamp: number      // Date.now()
}
```

### 4.3 动作日志

```typescript
interface ExerciseLog {
  sets: SetRecord[]
  completed: boolean
}

interface SetRecord {
  weight: number  // lb，0 表示自重
  reps: number    // 次数
  done: boolean
}
```

### 4.4 计划数据

三天计划固化在代码中（`DAYS` 数组），分层结构：

```
DAY
├── id (1/2/3)
├── label / sub
├── warmup[]     — 热身动作列表（含 extra 标记本日专用）
├── exercises[]  — 训练动作（id/名称/靶肌肉/组数/次数/默认重量/要点）
├── stretch[]    — 拉伸列表
└── cardio[]     — 有氧选项数组 [{name, detail}]
```

---

## 5. 技术方案

| 层 | 选型 | 说明 |
|---|---|---|
| 容器 | 单 HTML 文件 | 无构建工具，无依赖安装 |
| 样式 | 原生 CSS + CSS 变量 | 暗色主题，移动端适配 |
| 交互 | 原生 JavaScript | 无框架，无虚拟 DOM |
| 重量调节 | Stepper + 快捷标签 | ±5lb 步进，锚定到最近 10lb 快速切换 |
| 计时器 | setInterval + Vibration API | 组间休息 60s，振动提醒 |
| 存储 | localStorage | key `tr_app_v2`，~5MB 容量 |
| 部署 | GitHub Pages | GitHub Contents API 推送 |
| PWA | iOS Meta Tags | Safari 添加到主屏幕 |

### 5.1 SVG 图示引擎（已废弃）

- 原基于关节坐标的 stick figure 渲染（15 种动作类型），v2.0 已移除
- 当前保留 `svgCapsule()` 函数作为 dead code，未被渲染调用
- 发力注意事项改用纯文字列表，始终可见，无需点击展开

### 5.2 为什么不用框架

- 唯一使用者是本人，不需要多人协作
- 单文件部署，无需构建步骤
- 数据量和交互复杂度很低，不产生维护问题

---

## 6. 视觉设计

### 6.1 设计风格

暗色主题，橙色点缀，偏工具感。

- 背景 `#09090b`，卡片层 `#18181b` / `#121214`
- 边框 `rgba(255,255,255,.06)`
- 主色 `#f97316`（橙色，强调和状态）
- 成功 `#22c55e`（绿色，完成状态）
- 字体系统默认 Apple system font

### 6.2 交互细节

- 底部 Dock 栏，当前视图橙色高亮
- 动作卡片默认展示所有信息（名称/靶肌肉/组数/组勾选框/注意事项）
- 单击组行弹出 Stepper 浮层调重量和次数
- 单击状态圆圈标记动作全部完成
- 组完成自动触发 60s 组间休息计时器（含振动）
- 输入框实时保存（input 事件触发）
- Toast 轻提示反馈

### 6.3 布局约束

- 最大宽度 600px，居中，适配手机竖屏
- 顶部 / 底部安全区适配（`env(safe-area-inset-top)` 和 `env(safe-area-inset-bottom)`）
- Header 背景毛玻璃（`backdrop-filter: blur`）
- Dock 固定底部，内容区滚动

---

## 7. 边界情况与限制

| 场景 | 处理方式 |
|---|---|
| 首次打开 | 自动创建今日空记录，显示第一天计划 |
| 同一天切换天数 | 不同天数的 exercise ID 前缀不同（`d1-*`/`d2-*`/`d3-*`），同一日的记录会保留所有前缀的数据 |
| localStorage 满 | 约 5MB，按每日 ~2KB 估算可存 2500+ 天，个人使用不构成问题 |
| 离线 | 完全离线运行，无外部依赖 |
| 页面关闭后恢复 | 每次变更即时保存到 localStorage |
| GitHub Pages 部署 | Contents API 手动推送，无 CI/CD |

---

## 8. 未来可能扩展

（明确不纳入当前版本，记录备选）

- iCloud 同步（多设备）
- 训练模板自定义（修改组/添加动作）
- 心率图表趋势
- 1RM 估算
- 周/月训练量统计
- 浅色主题

---

## 9. 文件结构

```
training-app/
├── training_app.html    # 完整应用（~835 行，所有代码内联）
└── DESIGN.md            # 本设计文档
```
