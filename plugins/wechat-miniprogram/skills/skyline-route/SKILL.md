---
name: skyline-route
description: Skyline 自定义路由与页面转场：routeBuilder 动画、预设路由、返回手势、容器转场。
---

# Skyline 自定义路由与页面转场

## 文档索引

根据需求快速定位（路径相对于 `references/`）：

| 我想要... | 查阅文档 |
|-----------|----------|
| 了解自定义路由原理和接口 | `custom-route/custom-route-guide.md` |
| 查看半屏/手势返回代码模式 | `custom-route/route-patterns.md` |
| 快速使用预设路由 | `preset-route/preset-route.md` |
| 配置页面返回手势 | `pop-gesture/pop-gesture.md` |
| 实现卡片展开转场 | `open-container/open-container.md` |
| 查看 Router API | `api/router-api.md` |
| 了解 navigateTo 路由参数 | `api/navigate-to.md` |
| 监听路由事件 | `api/route-events.md` |

## 核心概念

### 路由能力层级

| 层级 | 能力 | 适用场景 |
|------|------|----------|
| 预设路由 | 一行代码使用 7 种内置效果 | 快速实现常见转场 |
| 自定义路由 | 通过 routeBuilder 完全控制动画 | 高度定制化转场 |
| 容器转场 | `<open-container>` 元素级过渡 | 卡片展开到详情页 |

### 动画控制器

| 属性 | 说明 |
|------|------|
| primaryAnimation | 页面进入/退出动画进度（0->1 进入，1->0 退出） |
| secondaryAnimation | 下一页进入时当前页动画进度（与下一页 primaryAnimation 同步） |
| userGestureInProgress | 当前路由进度是否由手势控制 |
| startUserGesture / stopUserGesture | 手势接管/释放路由控制 |
| didPop | 确认返回上一页 |

## 强制规则

### 自定义路由仅在连续 Skyline 页面间生效

A 页(Skyline) -> B 页(Skyline) 自定义路由生效；A 页(WebView) -> B 页(Skyline) 降级为默认路由。

### 动画处理函数必须声明 'worklet' 指令

```js
const handlePrimaryAnimation = () => {
  'worklet'
  return { transform: `translateX(${...}px)` }
}
```

缺少 `'worklet'` 指令的函数无法在 UI 线程执行。

### 手势接管必须成对调用 startUserGesture / stopUserGesture

```js
handleDragStart() {
  'worklet'
  this.customRouteContext.startUserGesture()
}
handleDragEnd() {
  'worklet'
  // ... 动画完成回调中：
  stopUserGesture()
}
```

### 确认返回时必须调用 didPop

引擎无法自动判断开发者是否要退出页面：

```js
primaryAnimation.value = timing(0.0, { duration }, () => {
  'worklet'
  didPop()
  stopUserGesture()
})
```

### navigator 组件在 Skyline 下只能嵌套文本节点

```html
<navigator url="/page">点击跳转</navigator>
```

不能嵌套 view 等普通节点。

### 不要在非 worklet 函数中访问 primaryAnimation.value，不要在未调用 startUserGesture 时直接修改 primaryAnimation.value

## Quick Reference

### 预设路由速查

| routeType | 效果 | 最低基础库 |
|-----------|------|-----------|
| `wx://bottom-sheet` | 底部弹出半屏 | 3.1.0 |
| `wx://upwards` | 自底向上全屏 | 3.1.0 |
| `wx://zoom` | 缩放进入 | 3.1.0 |
| `wx://cupertino-modal` | iOS 风格模态 | 3.1.0 |
| `wx://cupertino-modal-inside` | iOS 模态内嵌 | 3.1.0 |
| `wx://modal-navigation` | 模态导航 | 3.1.0 |
| `wx://modal` | 模态弹窗 | 3.1.0 |

```js
// 使用预设路由
wx.navigateTo({
  url: 'xxx',
  routeType: 'wx://bottom-sheet',
  routeOptions: { height: 60, round: true }
})
```

### API 速查

| API | 说明 | 最低基础库 |
|-----|------|-----------|
| `wx.router.addRouteBuilder(type, builder)` | 注册自定义路由 | 2.29.2 |
| `wx.router.removeRouteBuilder(type)` | 移除自定义路由 | 2.29.2 |
| `wx.router.getRouteContext(this)` | 获取路由上下文 | 2.29.2 |
| `wx.navigateTo({ routeType })` | 指定路由类型跳转 | 2.29.2 |
| `wx.navigateTo({ routeConfig })` | 覆盖路由配置 | 3.4.0 |
| `wx.navigateTo({ routeOptions })` | 传入路由参数 | 3.4.0 |
| `wx.navigateTo({ withOpenContainer })` | 容器转场跳转 | 3.12.2 |
| `wx.onBeforeAppRoute(fn)` | 路由执行前监听 | 3.5.5 |
| `wx.onAppRoute(fn)` | 路由执行后监听 | 3.5.5 |

### 自定义路由最小示例

```js
// 注册：从右滑入
wx.router.addRouteBuilder('slide', ({ primaryAnimation }) => {
  const { windowWidth } = wx.getWindowInfo()
  const handlePrimaryAnimation = () => {
    'worklet'
    const transX = windowWidth * (1 - primaryAnimation.value)
    return { transform: `translateX(${transX}px)` }
  }
  return { handlePrimaryAnimation }
})

// 跳转
wx.navigateTo({ url: 'pageB', routeType: 'slide' })
```

### 场景决策表

| 场景 | 推荐方案 |
|------|----------|
| 底部弹出半屏 | 预设路由 `wx://bottom-sheet` |
| iOS 风格模态 | 预设路由 `wx://cupertino-modal` |
| 自定义半屏 + 手势 | 自定义路由 + handlePrimaryAnimation |
| 卡片展开到详情页 | `<open-container>` 容器转场 |
| 页面渐显效果 | 自定义路由 + opacity 动画 |
| 需要纵向返回手势 | `popGestureDirection: 'vertical'` |
