---
title: 地图旅行动画 - 波浪转场与详情页风格改造
date: 2026-05-09
tags:
  - 动画
  - GSAP
  - Vue3
  - 高德地图
  - SVG
categories: 技术
---

今天对地图旅行动画项目做了两件大事：把进出详情页的转场改成了波浪动效，以及将详情页整体风格改成了温暖的暖色调 paper 美学。

## 一、转场动画：从 SMIL Path Morph 到 Blake Bowen Shape Overlay

### 踩坑记录

最开始想用 SMIL `<animate>` 做 path morph，思路是让一条 SVG path 的 `d` 属性在两个路径之间插值：

```
M 0 100 V 100 Q 50 100 100 100 V 100 z  （扁平，初始状态）
     ↓
M 0 100 V 50  Q 50 0   100 50  V 100 z  （拱形升起）
     ↓
M 0 100 V 0   Q 50 0   100 0   V 100 z  （矩形填满）
```

实际操作中发现两个坑：

1. **SMIL 要求两个路径结构完全一致**：点数/命令类型必须相同才能正确插值。尝试了四阶贝塞尔、矩形顶线+曲线混合等多种方案，SMIL 渲染出来的要么是平头矩形，要么是变形的畸形曲线。
2. **Vue 模板里的 `:from` / `:to` 绑定对 SVG SMIL 属性不生效**：`<animate :from="waveHidden" :to="waveRisen">` 在 Vue 模板编译后属性没有正确传递给 SMIL 元素，导致动画根本没有触发。

换用纯 JS 动态创建 SMIL 元素（`document.createElementNS`）绕过 Vue 模板限制后，波浪形态依然不对——SMIL 的线性插值无法产生拱形视觉效果。

### 最终方案：GSAP 驱动 SVG 点位

参考了经典的 [Blake Bowen shape overlay 方案](https://codepen.io/osublake/pen/BYwgBg)，核心思路是用 GSAP 驱动数组里的点位，通过 `onUpdate` 实时拼成 SVG path 的 `d` 属性：

```javascript
// 两个 path 层叠，各自由 10 个控制点管理顶部曲线
const NUM_PATHS = 2
const NUM_POINTS = 10
let allPoints = [] // allPoints[pathIndex][pointIndex] = y值

function render() {
  for (let i = 0; i < numPaths; i++) {
    let d = isOpened
      ? `M 0 0 V ${points[0]} C`
      : `M 0 ${points[0]} C`

    for (let j = 0; j < numPoints - 1; j++) {
      let p = ((j + 1) / (numPoints - 1)) * 100
      let cp = p - ((1 / (numPoints - 1)) * 100) / 2
      d += ` ${cp} ${points[j]} ${cp} ${points[j+1]} ${p} ${points[j+1]}`
    }

    d += isOpened ? ` V 100 H 0` : ` V 0 H 0`
    path.setAttribute('d', d)
  }
}
```

入场时 `points: 100→0`（`isOpened=true`），路径从顶部向下填充，产生波浪从底部升起的效果。出场时同样 `points: 0→100`（但 `isOpened=true`），波浪从顶部向下收，视觉上是向上消失。

**随机延迟**是关键：每个点有 `Math.random() * 0.3s` 的随机延迟，让波浪的各部分此起彼伏地运动，看起来有机自然，而不是整齐划一的机械运动。

### 时序控制

```
入场：
  1. wavePhase = 'entering' → 波浪开始升起
  2. 等 1550ms（两段 SMIL 动画完整跑完）→ wavePhase = 'cover'
  3. effectiveMode = 'detail' → 详情内容淡入

退场：
  1. 详情内容 gsap.to 淡出（opacity:0, y:-6, 0.3s）
  2. 波开始退出动画（points 0→100，1.25s）
  3. 波浪退出完成 → effectiveMode = 'list' + viewLayerRef opacity 重置为 1
```

**一个重要的 bug**：退出时详情内容被 `gsap.to({ opacity:0 })` 隐藏，但切换回 `'list'` 后 `viewLayerRef` 依然是 opacity=0 的状态，导致刚挂载的卡片列表也被隐藏了。解决方法是在 `effectiveMode = 'list'` 后立即 `gsap.set({ opacity: 1, y: 0 })` 重置透明度。

## 二、详情页暖色调改造

### 设计语言统一

详情页原本使用紫蓝色系（#6366f1），与卡片页的暖橙色风格割裂。这次全部改成了与卡片页一致的 oklch 暖色调：

| 元素 | 旧色值 | 新色值 |
|------|--------|--------|
| 主强调色 | `#6366f1` | `oklch(55% 0.13 45)` |
| 图标背景 | `linear-gradient(#6366f1, #818cf8)` | `oklch(72% 0.12 65)` |
| 状态栏背景 | `rgba(99,102,241,0.06)` | `oklch(96% 0.008 65 / 0.5)` |
| 进度条填充 | `linear-gradient(#6366f1, #818cf8)` | `linear-gradient(oklch(65% 0.11 45), oklch(55% 0.13 45))` |
| 面板背景 | `#ffffff` | `oklch(99.5% 0.003 80)` |

字体统一用 Noto Serif SC（标题）+ Noto Sans SC（正文），卡片页同款字体。边框、阴影全部用 oklch 暖灰色系，整体呈现温暖的手工纸质感。

### AMap Polyline 颜色格式 bug

发现 `AMap.Polyline` 的 `strokeColor` 使用了 `oklch(...)` 字符串，高德地图内部用 SVG 渲染，不支持 oklch 格式——线段直接变成透明不可见。换成十六进制颜色解决：

```javascript
// 错误：AMap 不认 oklch 格式
mainPolyline = new AMap.Polyline({ strokeColor: 'oklch(55% 0.13 45)', strokeWeight: 4 })

// 正确：十六进制
mainPolyline = new AMap.Polyline({ strokeColor: '#D96B1A', strokeWeight: 4 })
```

## 三、提交记录

GitHub: [trip-map-anim](https://github.com/yzcjwdkl/trip-map-anim)

```
commit 924562d
feat: 波浪入场退场 + 详情页暖色调改造

- 新增 TransitionWave 组件，基于 Blake Bowen shape overlay 实现双层波浪动效
- 入场：波浪从底部升起 + 向上消失，内容在波浪完成后展示
- 退场：详情内容先淡出，再做波浪退出动效
- 详情页整体改造成 warm paper 风格（oklch 暖色调、Noto Serif SC）
- 控制面板、状态栏、进度条、时间线、空状态全部重新设计
- 修复 AMap Polyline strokeColor oklch 不支持的问题（改用十六进制）
- App.vue 头部和 trips-header 使用 impeccable 卡片编辑风格
- 修复退出后卡片列表被隐藏的问题（viewLayerRef opacity 重置）
```

## 四、待优化项

- [ ] 波浪动画时长与实际视觉节奏的微调
- [ ] 详情页地图在不同窗口尺寸下的表现
- [ ] 考虑把 AMap 换成十六进制颜色写成常量，统一管理
