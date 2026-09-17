# 开发指南

本文档面向开发者，说明地球Online 的代码结构、核心机制和扩展方式。

## 技术选型说明

### 为什么是单文件 HTML？

MVP 阶段选择单文件原生 HTML/CSS/JS，不引入构建工具和框架，原因：

1. **零部署成本**：一个文件丢到任何静态服务器或直接本地打开就能跑
2. **无构建依赖**：不需要 Node.js、npm、webpack，降低贡献门槛
3. **隐私友好**：纯前端运行，用户数据不出浏览器
4. **快速迭代**：改完刷新即生效，适合 MVP 阶段快速验证

后续如果功能复杂度上升，可以考虑拆分为多文件 + 轻量构建（如 Vite），但核心逻辑仍保持框架无关。

### 为什么不用 React/Vue？

MVP 阶段的交互复杂度（4 个主视图 + 任务详情）用原生 JS 的状态重渲染完全可以 handle。引入框架会增加包体积和构建复杂度，而收益有限。代码中使用 IIFE + 命名空间（`window.App`）的方式组织，避免全局污染。

## 代码结构

`index.html` 内部按以下顺序组织：

```
<head>
  ├── CSS 变量（设计 token）
  ├── 通用组件样式
  ├── 各视图样式
  └── 响应式断点
<body>
  ├── 顶部 HUD 状态栏
  ├── 主内容区
  │   ├── view-setup        初始化/配置页
  │   ├── view-tasks        任务列表页
  │   ├── view-task-detail  任务详情页
  │   ├── view-character    角色页
  │   ├── view-timeline     时间线页
  │   └── view-settings     设置页
  ├── 底部导航栏
  └── 加载遮罩
<script>
  ├── CONFIG 常量配置
  ├── 状态管理（loadState/saveState）
  ├── 工具函数
  ├── 路由（navigate）
  ├── HUD 更新
  ├── 天气模块
  ├── LLM 调用封装（callLLm + parseJsonSafe）
  ├── 任务生成（buildTaskPrompt + generateDailyTasks）
  ├── 任务渲染（renderTasks + openTaskDetail）
  ├── 任务完成（completeTask + addExp + checkAchievements）
  ├── 各视图渲染函数
  ├── 初始化流程
  └── window.App 公共 API 暴露
```

## 核心机制

### 状态管理

使用一个全局 `state` 对象 + `saveState()` 持久化到 localStorage。没有使用响应式框架，状态变化后手动调用对应视图的 `renderXxx()` 函数重渲染。

```javascript
// 状态变更的标准模式
state.character.exp += 100;
saveState();           // 持久化
updateHUD();           // 更新受影响的 UI
```

localStorage key 为 `earth_online_state_v1`，版本号在 key 中，后续数据结构变更时可通过版本号做迁移。

### 路由

单文件应用使用 hash-less 的视图切换：每个视图是一个 `<section class="view">`，通过 `navigate(viewName)` 切换 `.active` 类。不使用 URL hash 是因为应用状态全在本地，不需要可分享的 URL。

### 经验与等级

```
升级所需经验 = floor(100 * 1.15^(level - 1))
```

指数增长曲线，前期升级快（给正反馈），后期逐渐变慢。升级时自动处理溢出经验（不会丢失）。

### 任务生成流程

```
用户点击「生成今日任务」
  → buildTaskPrompt() 组装上下文
  → callLLm() 调用 LLM（带 response_format 降级）
  → parseJsonSafe() 解析返回
  → 校验任务数量和字段
  → 存入 state.dailyTasks[today]
  → renderTasks() 渲染
```

每日仅生成一次，存在 `state.dailyTasks[日期]` 中。重新生成功能每日限 1 次，通过 `state.regenUsed[日期]` 标记。

### 感想分析

完成任务时如果填写了感想，会额外调用一次 LLM 分析关键词和情绪。这个调用是异步的，不阻塞任务完成的主流程——先标记完成、加经验，然后后台分析，完成后更新感想数据。

## 扩展指南

### 添加新的任务类型

1. 在 `CONFIG.TASK_TYPES` 中添加类型标识
2. 在 `buildTaskPrompt()` 的生成规则中说明新类型的定义
3. 在 `renderTasks()` 和 `openTaskDetail()` 中添加对应的标签颜色和文案
4. 在任务生成的 typeMap 中分配位置

### 添加新属性

1. 在 `getDefaultState().character.attributes` 中添加初始值
2. 在 `renderCharacter()` 的 `attrs` 数组中添加图标和名称
3. 在 CSS 中添加 `.attr-xxx .attribute-value` 颜色
4. 在任务生成 Prompt 的 attributeRewards 说明中添加

### 添加新成就

1. 在 `CONFIG.ACHIEVEMENTS` 中添加定义
2. 在 `checkAchievements()` 中添加判断条件

### 接入新的第三方 API

1. 在初始化页添加 Key 输入框
2. 在 `getDefaultState().profile.apiKeys` 中添加存储字段
3. 编写独立的调用函数（参考 `loadWeather()`）
4. 在任务生成 Prompt 中集成返回数据
5. 在 `docs/API.md` 中记录接口细节

## 调试技巧

### 查看当前状态

浏览器控制台执行：

```javascript
JSON.parse(localStorage.getItem('earth_online_state_v1'))
```

### 重置数据

设置页 → 清除所有数据，或控制台执行：

```javascript
localStorage.removeItem('earth_online_state_v1'); location.reload();
```

### 测试 LLM 连接

初始化页有「测试」按钮，会发送一个最小请求验证 Key 和模型名是否正确。

### 模拟不同天气

天气数据存在 `state.weather`，可在控制台手动修改后重新生成任务，观察 Prompt 变化：

```javascript
// 需通过 App 内部修改，或直接改 localStorage
const s = JSON.parse(localStorage.getItem('earth_online_state_v1'));
s.weather = { temp: '3', text: '小雪' };
localStorage.setItem('earth_online_state_v1', JSON.stringify(s));
location.reload();
```

## 性能考虑

- **首屏加载**：单文件约 63KB（未压缩），字体走 CDN，首屏渲染无阻塞
- **LLM 调用**：任务生成可能需要 5-15 秒（取决于模型和网络），有 loading 遮罩防止重复点击
- **localStorage 读写**：每次状态变更都完整序列化存储，数据量在 100KB 以内时性能无感知
- **DOM 重渲染**：视图切换时全量重渲染对应区域，数据量小（每日最多 3 个任务 + 历史列表），无需虚拟列表

## 浏览器兼容性

- 现代浏览器（Chrome / Edge / Safari / Firefox 最新两个大版本）
- 使用了 `backdrop-filter`、CSS Grid、`fetch`、`localStorage`、`geolocation` 等标准 API
- 不支持 IE
- iOS Safari 需注意 `env(safe-area-inset-bottom)` 已在底部导航中处理
