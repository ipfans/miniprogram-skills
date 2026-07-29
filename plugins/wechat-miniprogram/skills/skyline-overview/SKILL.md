---
name: skyline-overview
description: Skyline 渲染引擎概览与迁移：架构优势、环境要求、迁移检查清单、WebView 差异对比。
---

# Skyline 渲染引擎概览

## 文档索引

根据需求快速定位（路径相对于 `references/`）：

| 我想要... | 查阅文档 |
|-----------|----------|
| 了解 Skyline 架构和优势 | `introduction/overview.md` |
| 查看支持的特性 | `introduction/features.md` |
| 查看性能对比数据 | `performance/comparison.md` |
| 开始迁移项目 | `migration/getting-started.md` |
| 处理兼容性问题 | `migration/compatibility.md` |
| 发布上线指南 | `migration/release.md` |
| 查看更新日志 | `changelog/changelog.md` |
| 检测 Skyline 支持 | `api/getSkylineInfo.md` |
| 预加载 Skyline 环境 | `api/preloadSkylineView.md` |

## 核心概念

### Skyline 是什么？

Skyline 是微信小程序的新一代渲染引擎，相比 WebView 有以下优势：

| 对比项 | WebView | Skyline |
|--------|---------|---------|
| 渲染线程 | 与 JS 逻辑同线程 | 独立渲染线程 |
| 页面内存 | 每页一个 WebView | 共享渲染实例 |
| 首屏性能 | 较慢 | 快 66%+ |
| 光栅化 | 异步分块 | 同步（无白屏） |
| 页面栈限制 | 最多 10 层 | 无限制（连续 Skyline 页面） |

### 环境要求

| 平台 | 最低版本 | 基础库版本 |
|------|----------|------------|
| Android | 微信 8.0.40+ | 3.0.2+ |
| iOS | 微信 8.0.40+ | 3.0.2+ |
| HarmonyOS | 微信 1.0.10+ | 3.11.3+ |
| 开发者工具 | Stable 1.06.2307260+ | - |

## 快速开始配置

### 必需配置（app.json）

```json
{
  "renderer": "skyline",
  "lazyCodeLoading": "requiredComponents",
  "componentFramework": "glass-easel",
  "rendererOptions": {
    "skyline": {
      "defaultDisplayBlock": true,
      "defaultContentBox": true,
      "tagNameStyleIsolation": "legacy",
      "enableScrollViewAutoSize": true,
      "keyframeStyleIsolation": "legacy"
    }
  }
}
```

### 页面配置（page.json）

```json
{
  "navigationStyle": "custom",
  "disableScroll": true
}
```

Skyline 不支持原生导航栏，必须设置 `navigationStyle: custom` 并自行实现导航栏。

## 强制规则

1. app.json 必须包含 `renderer`、`lazyCodeLoading`、`componentFramework` 三项
2. 所有 Skyline 页面必须设置 `navigationStyle: custom`
3. 使用 `scroll-view` 实现滚动，不要依赖全局滚动
4. 纯文本必须用 `<text>` 组件包裹
5. 跳转 Skyline 页面前调用 `wx.preloadSkylineView()`
6. 不要使用原生导航栏配置
7. 不要依赖 `Page.onPageScroll` 事件（使用 scroll-view 的滚动事件替代）
8. 不要在 Skyline 页面使用 `web-view` 组件
9. 不要假设所有 CSS 属性都支持（参考 WXSS 支持文档）

## 迁移检查清单

### 配置检查

- [ ] app.json 添加 `renderer: "skyline"`
- [ ] app.json 添加 `lazyCodeLoading: "requiredComponents"`
- [ ] app.json 添加 `componentFramework: "glass-easel"`
- [ ] app.json 添加 `rendererOptions.skyline` 配置
- [ ] 页面 json 添加 `navigationStyle: "custom"`
- [ ] 页面 json 添加 `disableScroll: true`

### 代码检查

- [ ] 实现自定义导航栏组件
- [ ] 全局滚动改为 scroll-view 局部滚动
- [ ] 纯文本用 `<text>` 包裹
- [ ] 检查 WXSS 属性支持情况
- [ ] 测试原生组件（map/canvas/video）显示

### 发布检查

- [ ] 配置 We 分析 AB 实验（灰度发布）
- [ ] 测试 WebView 降级兼容性
- [ ] 低版本微信表现正常

## 判断当前渲染引擎

### JS 方式

```javascript
Page({
  onLoad() {
    // 页面/组件实例上有 renderer 属性
    console.log(this.renderer) // 'skyline' 或 'webview'
  }
})
```

### API 方式

```javascript
// 异步方式
wx.getSkylineInfo({
  success(res) {
    console.log(res.isSupported) // 是否支持 Skyline
    console.log(res.version)     // Skyline 版本号
  }
})

// 同步方式
const info = wx.getSkylineInfoSync()
console.log(info.isSupported, info.version)
```

## 性能优化提示

### 预加载 Skyline 环境

```javascript
// 在可能跳转到 Skyline 页面的页面中
Page({
  onShow() {
    // 延迟调用避免阻塞当前页面
    setTimeout(() => {
      wx.preloadSkylineView()
    }, 500)
  }
})
```

### 长列表优化

```html
<!-- 列表项必须作为 scroll-view 直接子节点 -->
<scroll-view type="list" scroll-y>
  <view wx:for="{{list}}" wx:key="id">{{item.name}}</view>
</scroll-view>

<!-- 启用样式共享 -->
<scroll-view type="list" scroll-y>
  <view wx:for="{{list}}" wx:key="id" list-item>{{item.name}}</view>
</scroll-view>
```
