---
title: 2026-04-20 — OpenClaw Skills 批量安装 + trip-map-anim 问题修复记录
date: 2026-04-20
tags:
  - OpenClaw
  - Vue3
  - Skills
  - 高德地图
  - Three.js
categories: 技术
---

今天是密集的一天，既有现有项目的 bug 修复，也完成了 AI Skills 的批量整理，为接下来的个人网站项目打好了技术基础。

## 🐛 trip-map-anim 项目修复

### 光环动画叠加多个的问题

之前光环效果会出现"水波纹"一样连续多个光环向外扩散的现象，原因是每次到达终点触发 `triggerArrivalRing` 时，如果上一轮光环动画还没完全结束（`ringState.active` 仍为 true），新的 RAF 循环又会被启动，导致多个光环叠加。

修复方案：在 `triggerArrivalRing` 开头加保护逻辑，若上一轮光环未结束则先 `cancelAnimationFrame` + 隐藏 SVG 再启动新的。同时在 `cancelAnim` 里也彻底关闭 SVG overlay，避免暂停时残留。

### 历史轨迹消失的问题

这是 `cancelAnim` 里的一个逻辑 bug：

```js
if (!isPlaying.value) { traveledPath = []; return }
traveledPath = []  // ← 这里没有 return，播放中也会执行！
```

播放中调用 `cancelAnim` 时，`isPlaying=true` 不进第一个 if，但没有 `return`，直接执行了 `traveledPath = []`，导致到达终点的轨迹瞬间消失。

修复：加上 `return`，播放中调用 `cancelAnim` 时直接返回，保留 `traveledPath`。

### 移动图标第二段消失的问题

从 emoji 图标迁移到 `AMap.Icon` + SVG 时，发现第二段轨迹开始时图标消失。原因是 `AMap.Icon` 依赖图片异步加载，`setIcon` 在图片未加载完成时被高德地图忽略。

修复：回退为 HTML `content` 方式，用 CSS `background: url('/assets/xxx.svg')` 引用 SVG，放在 `public/assets/` 下作为静态资源，避免异步加载问题。

### `Pixel(NaN, NaN)` 崩溃

`triggerArrivalRing` 接收到的坐标有时是 `undefined` 或 `[NaN, NaN]`，传给 `new AMap.LngLat` 直接崩溃。加了 `isNaN` 校验，坐标无效时直接 return 防止崩溃。

### 点位事件链详细注释

为 `planAndAnimate`、`animLoop`、`triggerArrivalRing` 加了完整的 ①~㉘ 步骤注释，标注了每个阶段触发了什么事件、做了什么操作，方便后续维护和交接。

---

## 📦 OpenClaw Skills 批量安装

为接下来的**个人网站项目**（3D avatar + 滚动驱动）做技能储备，审核并安装了以下 Skills：

### `vue-best-practices` — Vue3 最佳实践

来源：[moeru-ai/airi](https://github.com/moeru-ai/airi)，MIT 许可证

覆盖 Composition API + `<script setup lang="ts">` 规范，包含 22 个 references：
- `reactivity.md` / `sfc.md` / `component-data-flow.md` / `composables.md`（必读）
- 动画体系：class-based / state-driven 两种模式
- 性能优化、组件拆分原则、Composables 模式

### `impeccable` — 前端设计规范

来源：[pbakaus/impeccable](https://github.com/pbakaus/impeccable)，Apache 2.0（基于 Anthropic frontend-design skill）

作者 Piotr Bała 是前 Google Web Devrel，HTML5 Rocks 早期贡献者，业界权威。

覆盖字体选择决策树、配色体系、动效节奏、布局规范，专门避免"AI slop"审美。有 9 个 references：
- `typography.md`（字体排版）
- `color-and-contrast.md`
- `motion-design.md`
- `spatial-design.md`
- `interaction-design.md`
- `ux-writing.md`
- 等

### `scroll-experience` — 滚动叙事

来源：[sickn33/antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills)，Apache 2.0

Repo ⭐ 34,063，今日活跃更新。

覆盖 GSAP ScrollTrigger、滚动驱动动画、视差叙事、Sticky sections、Scroll snapping 等。610 行纯知识型 Skill，无执行风险。

### `mrgoonie-threejs` — Three.js 系统学习路径

来源：[mrgoonie/claudekit-skills](https://github.com/mrgoonie/claudekit-skills)，MIT 许可证

⭐ 2,009，19 个 references，渐进式学习路径 L1~L5：
- L1 基础：`00-fundamentals.md` / `01-getting-started.md`
- L2 常用：`02-loaders.md`（GLTF加载）/ `06-animations.md`（骨骼动画）/ `07-math.md`
- L3 交互：`08-interaction.md`（射线检测）/ `09-postprocessing.md`（后处理）
- L4 进阶：`12-performance.md`（LOD/实例化）/ `13-node-materials.md`（TSL）
- L5 专业：`14-physics-vr.md` / `16-webgpu.md`

---

## 🎯 个人网站项目规划

**技术栈：**
- 框架：Vue 3 + Composition API
- 3D：Three.js + TresJS（Vue 封装）
- 滚动：GSAP ScrollTrigger
- 设计：`impeccable` 规范

**网站核心交互：**
```
用户滚动 → GSAP ScrollTrigger 记录进度(0~1)
    ↓
3D Avatar ← 旋转/位移/骨骼动画切换（由滚动值驱动）
    ↓
页面 ← 不同模块淡入淡出（About / Books / Projects / Contact）
```

**下一步待办：**
- 获取 3D avatar 模型（ReadyPlayerMe 方案）
- 搭建 Vue + TresJS 项目框架
- 设计系统搭建（字体/配色/动效节奏）

---

*Skills are tools. The craftsman sharpens first.*
