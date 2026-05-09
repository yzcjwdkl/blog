---
title: 地图旅行动画 - 波浪转场与详情页风格改造
date: 2026-05-09
tags:
  - 动画
  - GSAP
  - Vue3
  - 高德地图
  - SVG
  - SMIL
categories: 技术
---

今天对地图旅行动画项目做了两件大事：把进出详情页的转场改成了波浪动效，以及将详情页整体风格改成了温暖的暖色调 paper 美学。

> 项目地址：[trip-map-anim](https://github.com/yzcjwdkl/trip-map-anim)，基于 Vue 3 + 高德地图 AMap + GSAP。

## 一、背景：想要什么样的转场？

项目有两个主要页面：卡片列表页（TripCardList）和详情页（TripDetail）。用户点击某张卡片，从列表页进入详情页（地图 + 途经点控制面板），之前只是一个简单的 `v-if` 切换，没有任何过渡动画。

想要的效果是：从 switch-in-demo.html 里看到的波浪覆盖效果——波浪从下往上起来，盖住列表，然后展示详情页。退出时反过来，波浪继续向上走，露出列表页。

这个 demo 我之前就保存在项目里，一直没舍得用，今天终于上了。

## 二、第一版尝试：SMIL Path Morph（失败）

### 初始思路

看到 demo 的第一反应是：这是不是就是两个 SVG path 之间做 morph？找两个路径，一个扁平、一个拱形，让 SMIL `<animate>` 在它们之间插值不就行了：

```html
<!-- 初始：扁平在底部 -->
<path d="M 0 100 V 100 Q 50 100 100 100 V 100 z" />

<!-- 目标：拱形升起 -->
<path d="M 0 100 V 50 Q 50 0 100 50 V 100 z" />
```

然后让 SMIL `<animate>` 在这两个路径之间做 `attributeName="d"` 的 morph。

### 坑 1：Vue 模板里 `:from` / `:to` 对 SMIL 无效

最开始写了这样的 Vue 组件：

```html
<path ref="pathRef" :d="currentPath">
  <animate
    :from="waveHidden"
    :to="waveRisen"
    attributeName="d"
    dur="0.65s"
    fill="freeze"
  />
</path>
```

结果是——动画完全没有触发。Vue 编译模板时，把 `:from` 和 `:to` 解析成了字符串 `"undefined"`（因为在模板里 `from` 和 `to` 不是合法的 HTML 属性，会被 Vue 特殊处理），根本没有传递给 SMIL 元素。

### 坑 2：动态创建 SMIL 元素绕过 Vue 限制

改成完全用 JS 动态创建：

```javascript
function morphPath(from, to, dur) {
  const path = pathRef.value
  const anim = document.createElementNS('http://www.w3.org/2000/svg', 'animate')
  anim.setAttribute('attributeName', 'd')
  anim.setAttribute('from', from)
  anim.setAttribute('to', to)
  anim.setAttribute('dur', `${dur}s`)
  anim.setAttribute('fill', 'freeze')
  path.appendChild(anim)
  anim.beginElement()
}
```

动画能触发了，但还是平头矩形——SMIL 的 `d` 属性 morph 要求两个路径有完全相同的点结构和命令序列，而且它是线性插值，不是贝塞尔曲线插值。当拱形顶部 Q 控制点 y 从 100 变到 0 时，SVG 渲染器只会把端点线性移动，曲线本身不会产生拱形——最后出来的是一个斜的梯形，不是拱形。

### 坑 3：demo 的两段式时序

回头再仔细看 demo，发现它的动画分两段：

```javascript
const start = "M 0 100 V 50 Q 50 0 100 50 V 100 z";  // 拱形（升起状态）
const end   = "M 0 100 V 0 Q 50 0 100 0 V 100 z";    // 矩形（填满状态）

tl.to(path, { morphSVG: start, ease: "power2.in" })
  .to(path, { morphSVG: end, ease: "power2.out" })
```

第一段：扁平 → 拱形（波浪升起）
第二段：拱形 → 矩形（拱顶继续扩展，填满屏幕）

GSAP MorphSVG（Club 会员插件，不是免费的）对两个路径做智能插值，能产生拱形效果。SMIL 完全做不到。

我尝试用两个 `<animate>` 分两段做：

```javascript
// 第一段：扁平 → 拱形
const anim1 = makeAnim(flatPath, archPath, '0.5s', 'power2.in')
// 第二段：拱形 → 矩形
const anim2 = makeAnim(archPath, fullPath, '0.45s', 'power2.out')
anim2.setAttribute('begin', '0.45s') // 等第一段完成后

path.appendChild(anim1)
path.appendChild(anim2)
anim1.beginElement()
// 用 setTimeout 精确触发第二段
setTimeout(() => anim2.beginElement(), 450)
```

出来的效果根本不是拱形——SMIL 插值方式决定了这点。这条路彻底走不通。

## 三、第二版：clipPath + 圆形扩缩（还是不对）

放弃 path morph，尝试用圆形 clipPath 来做覆盖效果。思路是：圆形从屏幕底部中央向上扩，拱形波浪 path 在 clipPath 内，视觉上就是波浪从下往上升起。

```html
<clipPath id="wv-clip">
  <circle ref="clipCircleRef" cx="50" cy="100" r="0" />
</clipPath>
<g clip-path="url(#wv-clip)">
  <path d="M 0 100 V 60 Q 50 0 100 60 V 100 Z" />
</g>
```

然后 GSAP 驱动圆的 `r` 属性从 0 到屏幕对角线长度：

```javascript
const coverRadius = Math.sqrt(200 * 200 + 50 * 50) + 2
gsap.fromTo(clipCircleRef.value,
  { attr: { r: 0 } },
  { attr: { r: coverRadius }, duration: 0.65, ease: 'power2.inOut' }
)
```

问题是：这样出来的是一个圆形的遮罩，不是波浪。波浪 path 在 clipPath 内只显示被圆覆盖的部分，视觉上像是从圆形里冒出来，不是真正的波浪升起效果。

这个方案也不对。

## 四、第三版（最终）：Blake Bowen Shape Overlay

重新分析 demo 的实现原理，终于理解了：

- 不是用一条 path 做 morph，而是用**数组控制点**生成 path
- 每个 path 的顶部不是一条贝塞尔曲线，而是被**多个控制点**分割成多个微小的贝塞尔曲线段
- 用 GSAP 驱动这些点位（y 值从 100→0 或 0→100），配合 `onUpdate` 实时拼出完整 path

参考 [Blake Bowen 的 CodePen](https://codepen.io/osublake/pen/BYwgBg)。

### 核心数据结构

```javascript
const NUM_PATHS = 2        // 两层 path 叠在一起
const NUM_POINTS = 10      // 每层 10 个控制点
const DURATION = 0.9       // 每个点动画时长
const DELAY_PER_PATH = 0.25 // 路径之间的延迟间隔
const DELAY_POINTS_MAX = 0.3 // 每个点的随机延迟上限

let allPoints = [] // allPoints[pathIndex][pointIndex] = y值（0=顶部，100=屏幕外底部）
```

初始时所有点位都是 100（屏幕外底部），两层 path 都是一条扁平的线。

### 渲染函数

```javascript
function renderPath(pathEl, points, isOpened) {
  let d = ''

  // isOpened=true: 从顶部开始向下延伸
  // isOpened=false: 从当前第一个点开始向下延伸
  d += isOpened ? `M 0 0 V ${points[0]} C` : `M 0 ${points[0]} C`

  // 相邻两个点之间用三次贝塞尔曲线连接
  // 控制点取在两个正方形像素的中点，这样曲线在视觉上是平滑的
  for (let j = 0; j < NUM_POINTS - 1; j++) {
    const p  = ((j + 1) / (NUM_POINTS - 1)) * 100
    const cp = p - ((1 / (NUM_POINTS - 1)) * 100) / 2
    d += ` ${cp} ${points[j]} ${cp} ${points[j+1]} ${p} ${points[j+1]}`
  }

  // 闭合 path
  d += isOpened ? ` V 100 H 0` : ` V 0 H 0`
  pathEl.setAttribute('d', d)
}
```

当 `isOpened=true` 且所有 `points[j]=0` 时，path 的顶部 y 全为 0，底部是 y=100 的水平线——等于盖满整个屏幕。当点位在 0~100 之间时，顶部曲线起伏不平，形成波浪。

### 入场动画

```javascript
function runEnter() {
  const tl = gsap.timeline({
    onUpdate: () => renderAll(true), // isOpened=true，path 从顶部向下展开
    defaults: { ease: 'power2.inOut', duration: DURATION }
  })

  // 每个点的随机延迟，让波浪此起彼伏有机地运动
  const pointsDelay = []
  for (let i = 0; i < NUM_POINTS; i++) {
    pointsDelay[i] = Math.random() * DELAY_POINTS_MAX
  }

  // pathDelay 的方向：isOpened=true → delayPerPath * i
  // 外层 path 先动，内层后动，产生波浪从外向内的层次感
  for (let i = 0; i < NUM_PATHS; i++) {
    const points = allPoints[i]
    const pathDelay = DELAY_PER_PATH * i

    for (let j = 0; j < NUM_POINTS; j++) {
      const delay = pointsDelay[j] + pathDelay
      tl.to(points, { [j]: 0 }, delay) // 点位从 100→0，波浪从底部升起
    }
  }
}
```

### 退场动画

退场的关键是：波浪继续向上走，露出外部背景（即列表页）。

```javascript
function runExit(onComplete) {
  const tl = gsap.timeline({
    onUpdate: () => renderAll(true), // 注意这里用 true，不是 false
    defaults: { ease: 'power2.inOut', duration: DURATION },
    onComplete
  })

  const pointsDelay = []
  for (let i = 0; i < NUM_POINTS; i++) {
    pointsDelay[i] = Math.random() * DELAY_POINTS_MAX
  }

  // pathDelay = delayPerPath * i（与入场一致，都是外层先动）
  for (let i = 0; i < NUM_PATHS; i++) {
    const points = allPoints[i]
    const pathDelay = DELAY_PER_PATH * i

    for (let j = 0; j < NUM_POINTS; j++) {
      const delay = pointsDelay[j] + pathDelay
      // 点位从 0→100，配合 isOpened=true，path 从顶部向下收
      // 视觉上波浪继续向上走，波浪颜色从有到无
      tl.to(points, { [j]: 100 }, delay)
    }
  }
}
```

**为什么要用 `renderAll(true)` 而不是 `renderAll(false)`**：
- `renderAll(false)` 时 path 是 `M 0 ${points[0]} C`，从当前曲线位置开始向下封闭，在当前 demo 里波浪是向下收的（不对）
- `renderAll(true)` 时 path 是 `M 0 0 V ${points[0]} C`，先画到顶部再向下填充。点位从 0 增到 100 的过程中，顶部从 y=0 逐渐升高到 y=100，视觉上波浪向上消失（正确）

### 两层渐变

两层 path 各有不同的渐变，产生颜色流动的叠加效果：

```html
<!-- 下层：橙 → 粉紫 -->
<linearGradient id="wv-grad-1" x1="0%" y1="0%" x2="0%" y2="100%">
  <stop offset="0%"   stop-color="oklch(72% 0.12 65)" />
  <stop offset="100%" stop-color="oklch(65% 0.11 160)" />
</linearGradient>

<!-- 上层：淡橙 → 深橙 -->
<linearGradient id="wv-grad-2" x1="0%" y1="0%" x2="0%" y2="100%">
  <stop offset="0%"   stop-color="oklch(78% 0.09 65)" />
  <stop offset="100%" stop-color="oklch(65% 0.13 45)" />
</linearGradient>
```

两层 path 叠在一起，上层盖住下层的上半部分，露出下半部分的颜色过渡，视觉上像是颜色在流动。

### z-index 阶段管理

波浪在整个转场过程中扮演不同角色，需要精确控制 z-index：

| phase | z-index | 用途 |
|-------|---------|------|
| `idle` | 1 | 隐藏，不阻挡任何交互 |
| `entering` | 9999 | 波浪升起，覆盖整个屏幕，阻挡列表页点击 |
| `cover` | 1 | 波浪沉底，作为详情页背景，不阻挡控制面板交互 |
| `exiting` | 9999 | 波浪退出，再次覆盖全屏阻挡交互 |

`'cover'` 阶段 z-index 降到 1，波浪变成详情页的静态背景，控制面板浮在上面可以正常交互。

## 四、入退场时序控制

时序的核心原则：**内容在波浪动画完成后再出现/消失**。

```javascript
// TravelMap.vue - 入场
watch(viewMode, (next) => {
  if (next === 'detail') {
    wavePhase.value = 'entering'
    // 波浪动画最晚结束时间：
    // 最外层 path 的最后一个点的触发时间 = 0.3s(随机) + 0.25s(pathDelay=0.25*0)
    // 动画时长 0.9s
    // 总计约 1.25s，加 buffer 取 1.55s
    setTimeout(() => {
      if (wavePhase.value !== 'entering') return
      wavePhase.value = 'cover'
      effectiveMode.value = 'detail'
      gsap.fromTo(viewLayerRef.value,
        { opacity: 0, y: 10 },
        { opacity: 1, y: 0, duration: 0.5, ease: 'power3.out' }
      )
    }, 1550)
  } else {
    // 退场：先淡出详情内容，再做波浪退出
    gsap.to(viewLayerRef.value, {
      opacity: 0, y: -6, duration: 0.3, ease: 'power2.in',
      onComplete: () => {
        wavePhase.value = 'exiting'
        // 波浪退出（约 1.3s），切换回列表
        setTimeout(() => {
          effectiveMode.value = 'list'
          // 关键：重置 viewLayerRef 透明度，否则列表页被隐藏
          gsap.set(viewLayerRef.value, { opacity: 1, y: 0 })
        }, 1300)
      }
    })
  }
})
```

## 五、一个严重 bug：退出后卡片页面空白

**问题现象**：退出详情页时，波浪退出动画完成后，页面完全空白，但刷新后卡片列表正常显示。

**根本原因**：退场时 `viewLayerRef` 被 GSAP 动画到 `opacity: 0`（详情内容淡出）。650ms 后 `effectiveMode = 'list'` 切换回列表页，但 `viewLayerRef` 此时 `opacity: 0`，刚刚挂载的卡片列表被完全隐藏了。

**修复**：切换到列表后立即重置透明度：

```javascript
setTimeout(() => {
  effectiveMode.value = 'list'
  gsap.set(viewLayerRef.value, { opacity: 1, y: 0 }) // 关键修复
}, 1300)
```

## 六、详情页暖色调改造

### 为什么要改

详情页之前用的是紫蓝色系（#6366f1），和卡片页的暖橙色风格完全割裂。用户从暖色调的卡片页点进来，进入紫蓝色的详情页，视觉跳感很强。

这次改造的原则是：**让详情页融入卡片页的 warm paper 美学**。

### 颜色替换对照

| 元素 | 旧色值 | 新色值 | 说明 |
|------|--------|--------|------|
| 主强调色 | `#6366f1` | `oklch(55% 0.13 45)` | 橙色，暖色调 |
| 图标背景 | `linear-gradient(#6366f1,#818cf8)` | `oklch(72% 0.12 65)` | 单色橙，不再用渐变 |
| 状态栏背景 | `rgba(99,102,241,0.06)` | `oklch(96% 0.008 65/0.5)` | 极淡的暖琥珀 |
| 进度条填充 | `linear-gradient(#6366f1,#818cf8)` | `linear-gradient(oklch(65% 0.11 45), oklch(55% 0.13 45))` | 橙色渐变 |
| 面板背景 | `#ffffff` | `oklch(99.5% 0.003 80)` | 微暖白 |
| 边框 | `rgba(0,0,0,0.04)` | `oklch(88% 0.01 80)` | 纸感灰线 |
| 阴影 | `rgba(0,0,0,0.04)` | `oklch(25% 0.01 80/0.04)` | 暖调阴影 |
| 时间线当前节点 | `#6366f1` | `oklch(55% 0.13 45)` | 橙色 |
| 时间线已访问 | `#cbd5e1` | `oklch(78% 0.02 260)` | 暖灰色 |

### 字体

标题用 Noto Serif SC，正文用 Noto Sans SC，与卡片页完全一致。数字用 Noto Sans SC，权重加粗突出。

### AMap Polyline 颜色格式 bug

发现 `AMap.Polyline` 的 `strokeColor` 用 `oklch(...)` 字符串——AMap 内部用 SVG 渲染，不支持 oklch 格式，线段直接变成透明不可见。换成十六进制解决：

```javascript
// 错误：AMap 内部 SVG 渲染器不支持 oklch
mainPolyline = new AMap.Polyline({ strokeColor: 'oklch(55% 0.13 45)', strokeWeight: 4 })

// 正确：十六进制
mainPolyline = new AMap.Polyline({ strokeColor: '#D96B1A', strokeWeight: 4 })
trailLine = new AMap.Polyline({ strokeColor: '#B05A30', strokeWeight: 4 })
```

教训：高德地图 JS API 的样式属性内部传给 SVG 渲染器，只能用标准 CSS 颜色格式（hex、rgb、rgba），不支持 `oklch`、`hsl` 等现代颜色格式。

## 七、完整提交记录

```
commit 924562d
feat: 波浪入场退场 + 详情页暖色调改造

- 新增 TransitionWave 组件，基于 Blake Bowen shape overlay
  实现双层波浪动效（10点控制 + 随机延迟 + GSAP onUpdate）
- 入场：波浪从底部升起（points 100→0），1550ms 后详情内容淡入
- 退场：详情内容先 0.3s 淡出，再做波浪退出（向上消失）
- 详情页整体改造成 warm paper 风格
- 控制面板、状态栏、进度条、时间线、空状态全部重新设计
- 修复 AMap Polyline strokeColor oklch 不支持的问题
- 修复退出后卡片列表被隐藏的问题
```

## 八、未尽事宜

- [ ] AMap 十六进制颜色建议统一写到常量文件里，避免散落各处
- [ ] 波浪动画时长和随机延迟参数的微调，目前 0.9s 的节奏感可能偏慢
- [ ] `delayPerPath = 0.25` 让两层 path 有层次感，但如果加更多层（3+）需要调整
- [ ] 详情页地图容器在全屏和非全屏切换时 resize 后的表现待验证
