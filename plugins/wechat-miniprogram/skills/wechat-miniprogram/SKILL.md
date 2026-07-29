---
name: wechat-miniprogram
description: "微信小程序开发。构建、调试或优化小程序时使用：wx.* API、WXML/WXSS/WXS、云开发、setData 性能、分包加载、组件开发、双端兼容（iOS/Android）、app.json 配置。也适用于涉及 AppID、真机预览、微信开发者工具等小程序特有工作流的任务。"
version: 1.0.0
---

# 微信小程序开发

## setData — 性能核心

小程序逻辑层（JS）与渲染层（WebView）通过 Native 桥通信。setData 是唯一数据通道，每次调用都经历：序列化 → 跨线程传输 → 反序列化 → diff → 渲染。优化 setData 是小程序性能的首要任务。

### 规则

1. **数据路径局部更新**，杜绝全量替换：
   ```javascript
   this.setData({
     'userInfo.nickName': 'New Name',
     'list[0].status': 'done'
   })
   ```

2. **合并多次调用为一次**：
   ```javascript
   const updates = {}
   updates[`list[${index}].checked`] = true
   updates['count'] = this.data.count + 1
   this.setData(updates)
   ```

3. **只传渲染用数据**。内部状态挂在实例属性上（`this._cache = ...`），不进 setData。

4. **单次传输 < 1024KB**，调用频率 < 10 次/秒。

5. **在 onPageScroll 中使用节流**，绝不直接 setData — 会严重影响滚动流畅度。

6. **hidden vs wx:if**：频繁切换用 `hidden`（节点始终存在，切换 display）；不频繁切换用 `wx:if`（条件为 false 时销毁节点，节省初始渲染开销）。

详细优化指南：`references/performance-optimization.md`

## 双端陷阱

### iOS 日期格式

```javascript
// iOS 返回 Invalid Date
new Date('2024-03-31 12:00:00')

// 用斜杠或 replace
new Date(dateStr.replace(/-/g, '/'))
```

### iOS 键盘遮挡

监听 `wx.onKeyboardHeightChange` 动态调整输入区域 padding。

### API 可用性

使用 `wx.canIUse` 检查后提供降级：
```javascript
if (wx.canIUse('getUserProfile')) {
  // 基础库 2.10.4+
} else {
  // 降级处理
}
```

版本兼容性速查：`references/version-compatibility.md`

## 页面栈（最大 10 层）

navigateTo 在栈满时静默失败。

```javascript
function navigateTo(url) {
  if (getCurrentPages().length >= 10) {
    wx.redirectTo({ url })
  } else {
    wx.navigateTo({ url })
  }
}
```

- Tab 页面只能用 `wx.switchTab`
- 清空页面栈用 `wx.reLaunch`
- 获取上一页实例：`getCurrentPages()[getCurrentPages().length - 2]`

## 原生组件层级

`video`、`map`、`canvas`、`camera`、`live-player` 等原生组件层级最高，普通 view 无法覆盖。在原生组件上方叠加内容只能使用 `cover-view` 和 `cover-image`。

## 分包策略

| 限制 | 值 |
|------|-----|
| 主包 | < 2MB |
| 单个分包 | < 2MB |
| 总包（含分包） | < 20MB |

```json
{
  "subPackages": [
    { "root": "pkgA", "pages": ["pages/detail/detail"] },
    { "root": "pkgB", "pages": ["pages/profile/profile"], "independent": true }
  ],
  "preloadRule": {
    "pages/index/index": { "network": "all", "packages": ["pkgA"] }
  }
}
```

独立分包（`independent: true`）可独立运行，不依赖主包。用 `preloadRule` 提前加载可能访问的分包。

## 云开发安全

1. **敏感查询放云函数**，前端不直接查询敏感集合。
2. **用 `_openid` 隔离用户数据**：
   ```json
   { "read": "doc._openid == auth.openid", "write": "doc._openid == auth.openid" }
   ```
3. **云开发初始化需基础库 2.2.3+**。

云开发完整指南：`references/cloud-development.md`

## 异步竞态

快速切换页面时，异步回调可能更新已销毁的页面。用版本号守卫：

```javascript
async loadData() {
  this._dataVersion = (this._dataVersion || 0) + 1
  const currentVersion = this._dataVersion
  const data = await fetchData()
  if (currentVersion === this._dataVersion && this.data) {
    this.setData({ data })
  }
}
```

或用 `requestTask.abort()` 取消前一次请求。

## 存储限制

单个 key < 1MB，总容量 < 10MB。空间满时 `setStorageSync` 抛异常，需 catch 处理。

## Component 构造器

优先使用 Component 而非 Page 构造页面，获得 properties、lifetimes、样式隔离等能力：

```javascript
Component({
  options: { styleIsolation: 'isolated', multipleSlots: true },
  properties: {
    title: { type: String, value: '' }
  },
  lifetimes: {
    attached() { this.init() },
    detached() { this.cleanup() }
  },
  methods: {
    handleTap() {
      this.setData({ count: this.data.count + 1 })
      this.triggerEvent('tap', { count: this.data.count })
    }
  }
})
```

通信方式：父→子 `properties`，子→父 `triggerEvent`，跨组件用事件总线或 MobX。

## wx:for 必须指定 wx:key

缺少 `wx:key` 导致列表重渲染时状态丢失和性能下降：
```xml
<view wx:for="{{list}}" wx:key="id">{{item.name}}</view>
```

## 合规要求

- **备案**：2023年9月起强制，未备案无法发布
- **隐私保护指引**：2024年起强制，未配置则隐私 API 不可用
- **域名白名单**：必须 HTTPS + ICP 备案，每月最多修改 5 次

发布与合规详情：`references/deployment-guide.md`

## 项目结构

```
miniprogram/
├── app.js / app.json / app.wxss
├── pages/
│   └── index/
│       ├── index.js / index.json / index.wxml / index.wxss
├── components/
├── utils/          # request.js, storage.js, util.js
├── api/            # 接口定义
├── assets/         # images, icons, fonts
├── cloud/functions/          # 云开发
└── miniprogram_npm/          # npm 构建产物
```

## 性能指标

| 指标 | 目标 |
|------|------|
| 首屏渲染 | < 2s |
| setData 单次数据 | < 1024KB |
| setData 频率 | < 10 次/秒 |
| 主包 | < 2MB |
| 页面栈 | ≤ 10 层 |
| 长列表 | < 100 项（否则用虚拟列表） |
| 图片体积 | 单张 < 100KB（用 webp） |

## 参考文档

按需查阅 `references/` 目录：

**实践指南**（策略、模式、代码模板）：

| 场景 | 文件 |
|------|------|
| API 速查（wx.* 接口签名与用法） | `references/api-reference.md` |
| 性能优化（setData 深入、渲染、启动、网络、内存） | `references/performance-optimization.md` |
| 云开发（云函数、云数据库、云存储完整模式） | `references/cloud-development.md` |
| 发布部署（备案、隐私协议、域名、CI/CD、审核） | `references/deployment-guide.md` |
| 错误处理（统一处理模板、错误码对照表） | `references/error-handling.md` |
| 测试调试（单元测试、自动化、真机调试） | `references/testing-guide.md` |
| UI 组件库（WeUI / TDesign / CloudBase Agent UI 选型） | `references/ui-components-guide.md` |
| 基础库版本兼容性（API 版本要求、版本分布） | `references/version-compatibility.md` |

**官方文档索引**（原始文档，深度查阅）：

| 场景 | 文件 |
|------|------|
| 框架机制（逻辑层、视图层、生命周期、Skyline） | `references/framework.md` |
| 内置组件 | `references/components.md` |
| 微信 API（基础接口、服务端接口） | `references/api.md` |
| 云开发（官方文档） | `references/cloud.md` |
| 全局/页面配置项 | `references/reference.md` |
| 快速上手 | `references/getting_started.md` |
| 商业能力、城市服务等 | `references/other.md` |
