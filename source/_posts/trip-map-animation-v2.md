---
title: 地图旅行动画 2.0 — 体验升级记录
date: 2026-04-03
tags:
  - 地图
  - 动画
  - Vue3
  - 高德地图
categories: 技术
---

今天对地图旅行动画项目做了一次比较大幅度的体验升级，解决了几个长期痛点，加入了一些新特性，记录一下。

## 解决的老问题

### 添加点位按钮点了没反应

一开始按钮点了完全没反应，打开控制台发现地图加载失败，报了 `Cannot read properties of undefined (reading 'position')`。原因是 `sampleTrip` 数据根本没有导入，`tripData.points` 一直是空数组，`onMounted` 里直接访问 `points[0].position` 就崩了。修法很简单，把数据 import 进来作为初始值就行。

### 飞机模式下轨迹"瞬间转移"

之前飞机模式两个城市之间是直线，但动画看起来像是瞬间到达的——marker 一直在起点，突然跳到终点。原因是对只有两个点的路径做插值时，`eased * (length-1)` 全程几乎等于 0，最后一帧才跳到 1。改成了线性插值（lerp）计算起终点之间的中间位置，轨迹就能实时生长出来了。

### 地图大幅跳转导致白屏

旧版每段动画开始都会调用 `setZoomAndCenter`，跨城市大跨度时 zoom 变化剧烈，地图一直在加载新 tile，页面白屏闪烁。改成了：

- 大跨度（>2°）**只设一次 zoom**，动画过程中地图完全不动
- 用 `requestAnimationFrame` 驱动 `maybePan`，marker 出窗口 45% 安全区才触发一次 `panTo`

这样地图基本不动，只有 marker 和线在走，流畅多了。

## 新增功能

### 段间出行方式独立切换

之前全局一个"飞机/驾车"切换按钮，所有段共用。现在每个点位有一个 `travelTypeToHere` 字段，记录"到达这个点用的什么交通方式"。列表里两个点位之间多了一个 badge，点一下切换 ✈️ / 🚗。新建点位自动继承当前选中的方式。

### 飞机弧线 + 往返不重叠

飞机模式改成贝塞尔弧线，跨城市有漂亮的弧度。更重要的是——往返时弧线方向交替，第二次走同一条航线弧度会往反方向拱，不会和第一次重叠。

### 三步预热动画

每段出发前的准备流程，分三步执行，每步有可见的停顿过渡：

1. **旋转** — 地图旋转，让出发点在下、到达点在上
2. **设层级** — 根据两点距离算合适 zoom，动画过渡到位
3. **平移到出发点** — 等待过渡完成后，开始画轨迹

这个"先摆好视角再出发"的感觉，让整个播放有了节奏感。

### 旋转视角开关

旋转地图这个效果不是所有人都喜欢，所以加了一个 **🧭 旋转** 按钮，默认关闭，需要的人再打开。

### 地图跟随策略优化

之前用 `setInterval` 做跟随，改成了 `requestAnimationFrame` + `maybePan` 安全区判断，每帧检查 marker 是否出了窗口边界，没出就不动，出了才 `panTo`。流畅度和效率都更好。

## 部分代码实现细节

### 贝塞尔弧线

```js
function makeArcPath(from, to, routeIndex, segments = 30) {
  const mid = [(from[0] + to[0]) / 2, (from[1] + to[1]) / 2]
  const maxDist = Math.max(Math.abs(to[0] - from[0]), Math.abs(to[1] - from[1]))
  const bulge = Math.min(maxDist * 0.15, 1.5) // 拱起高度随距离缩放
  const dir = routeDirMap[routeIndex] ?? 1
  const sign = dir % 2 === 0 ? 1 : -1 // 往返交替方向
  routeDirMap[routeIndex] = dir + 1

  const dx = to[0] - from[0], dy = to[1] - from[1]
  const len = Math.sqrt(dx * dx + dy * dy) || 1
  const ctrl = [mid[0] - dy / len * bulge * sign, mid[1] + dx / len * bulge * sign]

  // 贝塞尔曲线生成路径点
  const result = []
  for (let i = 0; i <= segments; i++) {
    const t = i / segments
    const mt = 1 - t
    result.push([
      mt * mt * from[0] + 2 * mt * t * ctrl[0] + t * t * to[0],
      mt * mt * from[1] + 2 * mt * t * ctrl[1] + t * t * to[1]
    ])
  }
  return result
}
```

### 等待旋转/缩放完成的轮询

高德的 `setRotation` 和 `setZoom` 带动画但没有完成回调，用轮询判断：

```js
function waitRotationDone(callback) {
  const interval = setInterval(() => {
    if (Math.abs(map.getRotation() - targetRotation) < 0.5) {
      clearInterval(interval)
      setTimeout(callback, 400) // 停顿后再回调
    }
  }, 50)
}
```

### maybePan 安全区跟随

```js
function maybePan(pos) {
  const bounds = map.getBounds()
  const padding = 0.45 // 45% 安全区
  const lngRange = ne.lng - sw.lng
  const latRange = ne.lat - sw.lat
  const minLng = sw.lng + lngRange * padding
  // ...省略
  if (pos[0] < minLng || pos[0] > maxLng || pos[1] < minLat || pos[1] > maxLat) {
    map.panTo(pos)
  }
}
```

## 示例路线

目前默认数据换成了一个更大尺度的路线，测试各种跨度：

- 厦门市 → 昆明市（✈️ 飞机，约 15°）
- 昆明市 → 腾冲市（🚗 驾车，约 4°）
- 腾冲市 → 芒市（🚗 驾车，约 1°）
- 芒市 → 昆明市（🚗 驾车）
- 昆明市 → 厦门市（✈️ 飞机，约 15°）

长途打飞机会有漂亮的弧线，中距离开车走真实路线，近距离平缓过渡。

---

项目地址：`~/trip-map-anim/`
