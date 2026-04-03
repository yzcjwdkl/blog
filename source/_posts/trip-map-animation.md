---
title: 做了一个地图旅行动画生成器，支持驾车路线规划和自由添加点位
date: 2026-04-02 18:00:00
updated: 2026-04-02 18:00:00
tags:
  - Vue3
  - 高德地图
  - 前端
  - 地图可视化
categories: 项目记录
cover: https://images.unsplash.com/photo-1476514525535-07fb3b4ae5f1?w=800&q=80
description: 用 Vue 3 + 高德地图做了一个可交互的旅行轨迹动画生成器，支持飞机直线模式、驾车真实路线模式，还能自由添加和删除点位。
---

# 做了一个地图旅行动画生成器，支持驾车路线规划和自由添加点位

之前在携程上看到那种地图逐步"画出"旅行轨迹的动画，觉得挺有意思，就想自己做一个。刚好有个朋友要去云南德宏芒市参加泼水节，正好拿来练手。

今天把核心功能基本做完了，顺手记录一下实现思路。

---

## 先看看效果

功能包括：

- **两种出行模式** — ✈️ 飞机模式走直线，🚗 开车模式调高德路径规划 API，走真实道路
- **逐步延伸的轨迹线** — 轨迹不是一下子全画出来，而是跟着车一帧一帧往前画
- **到达弹跳** — 圆点点在到达每个点位时有弹跳动画
- **半透明渐变标签** — 目的地标签在行进中显示，半透明渐变背景不挡路线
- **自由添加/删除点位** — 搜索地点 → 地图预览 → 确认添加，不用写代码

---

## 技术栈

```
Vue 3 + Vite
@amap/amap-jsapi-loader   高德地图 JS API v2.0
Element Plus               控制面板 UI
高德 PlaceSearch 插件      地点搜索
高德 Driving 插件          驾车路径规划
```

选高德而不是 Google Maps，主要是因为国内用，高德开放平台免费，JS API 直接在网页里调，不需要服务端代理。

---

## 几个关键实现

### 1. 高德地图加载和安全密钥

高德 JS API v2.0 强制要求配置安全密钥，不然所有 API 请求都会失败。这步卡了好久才找到原因。

```javascript
// 安全密钥配置（必须在加载前设置）
window._AMapSecurityConfig = {
  securityJsCode: '你的安全密钥'
}

AMapLoader.load({
  key: '你的Web端Key',
  version: '2.0',
  plugin: ['AMap.Scale', 'AMap.Driving']
}).then(AMap => {
  map = new AMap.Map('container', { zoom: 14, center: [...] })
})
```

安全密钥不是在高德控制台创建应用时那个 AppCode，而是在"我的应用"里对应 Key 的那个"安全密钥"字段，很容易忽略。

### 2. 驾车路径规划的坑

一开始调 `AMap.Driving.search()` 怎么都不返回路线坐标，后来才发现高德默认 `extensions: 'base'`，只返回地址信息，**不返回路径点**！必须改成 `extensions: 'all'` 才能拿到每一步的 path 坐标。

```javascript
const driving = new AMap.Driving({
  policy: AMap.DrivingPolicy.LEAST_TIME,
  extensions: 'all',  // 关键！不写就拿不到路径坐标
  hideMarkers: true
})

driving.search(from, to, function(status, result) {
  if (status === 'complete' && result.routes && result.routes[0]) {
    const route = result.routes[0]
    const path = []
    route.steps.forEach(step => {
      step.path.forEach(p => path.push([p.lng, p.lat]))
    })
    // path 就是整条路的坐标数组
    animateAlongPath(path)
  }
})
```

路径点拿到了之后，动画其实就简单了——用定时器沿坐标数组逐帧插值，`trailLine`（跟随线）始终画到当前位置，车 marker 同步移动。

### 3. 两种出行模式的统一接口

飞机模式和开车模式最后都调用同一个 `animateAlongPath(pathCoords)` 函数，区别只是传进来的路径不同：

- **飞机模式**：`pathCoords = [起点, 终点]`
- **开车模式**：`pathCoords = [高德返回的完整路径点数组]`

这样切换模式只需要在 `planAndAnimate()` 里换一条分支，动画逻辑完全不用改。

### 4. 弹跳动画

到达点位时圆点点有个弹跳效果，用关键帧逐帧改变 `CircleMarker` 的半径：

```javascript
function bounceDot(dot) {
  dot.setRadius(6) // 恢复初始
  const frames = [11, 3, 9, 4, 7, 5, 6]  // 半径序列
  const times  = [0, 120, 240, 360, 480, 580, 660] // 毫秒
  frames.forEach((r, i) => {
    setTimeout(() => dot.setRadius(r), times[i])
  })
}
```

弹跳完成后才触发下一段动画，用链式 setTimeout 控制时序，不依赖 setInterval 避免竞态。

### 5. 点位搜索添加

添加新点位用高德 PlaceSearch 插件，流程是：

1. 搜索框输入关键词 → 调用 `PlaceSearch.search()`
2. 选一个结果 → 弹小地图预览坐标
3. 填类型、描述 → 确认添加
4. 地图上实时显示新点位标记

点位数据完全响应式，Vue 3 的 `ref()` + 数组操作，添加删除都直接操作 `tripData.value.points`，地图上的 marker 同步增删。

---

## 遇到的主要问题

**1. vue-amap 不兼容 Vue 3**

最初用了 `vue-amap` 这个封装库，但它是 Vue 2 的，直接报错。后来换成官方的 `@amap/amap-jsapi-loader` + 直接调用 API，绕过去了。

**2. 地图加载时机**

因为地图是异步加载的，如果用户点播放时地图还没好，所有地图操作都会崩溃。解决方案是加了一个 `mapReady` 状态，地图加载完成才解锁播放按钮和添加功能。

**3. 驾车 API 路线返回空**

两个原因叠加：一是没配安全密钥，二是 `extensions` 默认为 `'base'` 不返回路径点。两个都修好之后才正常。

---

## 项目结构

```
trip-map-anim/
├── public/
│   └── test-driving.html     测试页（单独验证地图功能）
├── src/
│   ├── components/
│   │   └── AddPointDialog.vue   搜索添加点位弹窗
│   ├── data/
│   │   └── sampleTrip.js        示例行程数据
│   ├── views/
│   │   └── TravelMap.vue        核心地图+动画页面
│   ├── App.vue
│   └── main.js
└── package.json
```

---

## 可以继续优化的地方

- 点位支持拖拽调整顺序
- 保存行程数据到 localStorage
- 支持导入 GPX 文件
- 点位信息更丰富（电话、开放时间等）
- 动画速度根据实际距离动态调整

---

总的来说不算复杂，主要的时间都花在调试高德 API 的各种细节上了。地图类项目还是得上手实测，光看文档不够。完整代码在本地，有需要可以拿去改。
