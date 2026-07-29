---
name: skyline-worklet
description: Skyline Worklet 动画系统：SharedValue、timing/spring/decay 动画、Easing 缓动、线程通信。
---

# Worklet 动画系统

## 文档索引

根据需求快速定位（路径相对于 `references/`）：

| 我想要... | 查阅文档 |
|-----------|----------|
| 了解 worklet 架构和完整概念 | `core/worklet-overview.md` |
| 使用 SharedValue 和 DerivedValue | `base/shared-derived.md` |
| 在 worklet 中操作 scroll-view | `base/scroll-view-context.md` |
| 使用 timing/spring/decay 动画 | `animation/timing-spring-decay.md` |
| 查看 Easing 缓动函数 | `animation/easing.md` |
| 使用序列/重复/延迟组合动画 | `animation/combine-animation.md` |
| 了解 runOnUI/runOnJS 线程通信 | `tool/thread-communication.md` |

## 核心概念

### 双线程架构与 Worklet 的意义

小程序双线程架构中，UI 事件需跨线程传递到 JS 线程再回传，交互动画会有明显延迟。Worklet 动画让动画逻辑直接运行在 UI 线程，实现类原生动画体验。

### 三大核心概念

| 概念 | 说明 | 关键 API |
|------|------|----------|
| **worklet 函数** | 可运行在 JS 或 UI 线程的函数，顶部声明 `'worklet'` 指令 | `runOnUI()`, `runOnJS()` |
| **共享变量** | 跨线程同步的变量，通过 `.value` 读写 | `shared()`, `derived()` |
| **动画驱动** | 将 SharedValue 绑定到节点样式 | `applyAnimatedStyle()` |

### 基本流程

```js
const { shared, timing } = wx.worklet

// 1. 创建共享变量
const offset = shared(0)

// 2. 绑定到节点样式（updater 为 worklet 函数）
this.applyAnimatedStyle('#box', () => {
  'worklet'
  return { transform: `translateX(${offset.value}px)` }
})

// 3. 修改值驱动动画
offset.value = timing(300, { duration: 200 })
```

## 强制规则

### worklet 函数必须声明 `'worklet'` 指令

```js
function handleGesture(evt) {
  'worklet'
  offset.value += evt.deltaX
}
```

缺少指令的函数无法在 UI 线程执行。

### SharedValue 必须通过 `.value` 读写

```js
const offset = shared(0)
offset.value = 100
```

直接赋值 `offset = 100` 会替换整个 SharedValue 对象。

### 访问非 worklet 函数必须使用 `runOnJS`

```js
function showModal(msg) {
  wx.showModal({ title: msg })
}
function handleTap() {
  'worklet'
  const fn = this.showModal.bind(this)
  runOnJS(fn)('hello')
}
```

worklet 中不能直接调用普通函数或 `wx` API，必须通过 `runOnJS` 回到 JS 线程。

### 页面方法必须通过 `this.methodName.bind(this)` 访问

```js
handleTap() {
  'worklet'
  const showModal = this.showModal.bind(this)
  runOnJS(showModal)(msg)
}
```

未 `bind(this)` 会导致 this 指向丢失。

### Worklet 动画仅在 Skyline 渲染模式下可用

确保 app.json 配置 `"renderer": "skyline"` 并且开发者工具勾选「将 JS 编译成 ES5」。

### 不要通过解构 `this.data` 访问属性

解构会导致 `Object.freeze` 冻结 `this.data`，`setData` 将失效：

```js
handleTap() {
  'worklet'
  const msg = this.data.msg  // 直接点访问，不要解构
}
```

## Quick Reference

### API 速查表

| 分类 | API | 说明 |
|------|-----|------|
| 基础 | `shared(initialValue)` | 创建 SharedValue |
| 基础 | `derived(updaterWorklet)` | 创建衍生值（类比 computed） |
| 基础 | `cancelAnimation(sharedValue)` | 取消动画 |
| 动画 | `timing(toValue, options?, callback?)` | 时间曲线动画（默认 300ms） |
| 动画 | `spring(toValue, options?, callback?)` | 弹簧物理动画 |
| 动画 | `decay(options?, callback?)` | 滚动衰减动画 |
| 组合 | `sequence(anim1, anim2, ...)` | 依次执行 |
| 组合 | `repeat(anim, reps, reverse?, callback?)` | 重复（负值=无限） |
| 组合 | `delay(ms, anim)` | 延迟执行 |
| 工具 | `runOnUI(workletFn)` | 在 UI 线程执行 |
| 工具 | `runOnJS(normalFn)` | 回调 JS 线程 |

### 场景 -> 方案映射

| 场景 | 推荐方案 |
|------|----------|
| 点击后平滑移动 | `timing` + `Easing` |
| 手势松开回弹 | `spring` |
| 手势松开惯性滑动 | `decay` + `velocity` |
| 先移动再弹回 | `sequence(timing, spring)` |
| 循环脉动效果 | `repeat(timing, -1, true)` |
| 延迟后开始动画 | `delay(ms, timing/spring)` |
