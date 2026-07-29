# 微信小程序性能优化详解

主 SKILL.md 覆盖 setData 的基本规则，本文覆盖完整的性能优化体系：指标定义、setData 深度优化、渲染、代码包、启动、网络、内存。

## 性能指标

### 目标值

| 指标 | 及格 | 优秀 | 说明 |
|------|------|------|------|
| 首屏渲染 | < 2s | < 1s | 从页面 onLoad 到首次有效内容渲染完成 |
| 页面切换 | < 300ms | < 150ms | navigateTo 到新页面首次渲染 |
| 代码包总体积 | < 8MB | < 4MB | 主包硬限制 2MB，总包硬限制 20MB |
| setData 单次耗时 | < 30ms | < 15ms | 逻辑层调用到渲染层更新完成 |
| 渲染帧率 | > 50fps | > 58fps | 滚动、动画期间 |
| 内存占用 | < 200MB | < 100MB | iOS 超过约 200MB 有被系统回收的风险 |

### 硬性限制（超出会报错或静默失败）

| 限制项 | 值 |
|--------|-----|
| setData 单次数据量 | 1024KB（超出抛 `Data transmission length exceeds maximum` 错误，建议单次 < 256KB） |
| setData 调用频率 | 建议 < 10 次/秒 |
| 主包 / 单个分包 | 2MB |
| 总包（含全部分包） | 20MB |
| 页面栈 | 10 层 |
| Storage 单 key / 总量 | 1MB / 10MB |

### 采集真实性能数据：wx.getPerformance

`wx.getPerformance()` 返回性能管理器，通过 observer 订阅性能条目。**基础库 2.11.0+**。

```javascript
// app.js
App({
  onLaunch() {
    this.initPerformanceMonitor()
  },

  initPerformanceMonitor() {
    if (!wx.getPerformance) return

    const performance = wx.getPerformance()
    const observer = performance.createObserver((entryList) => {
      entryList.getEntries().forEach((entry) => {
        // entry.entryType: 'navigation' | 'render' | 'script' | 'loadPackage'
        // entry.name:      'appLaunch' | 'route' | 'firstRender' | 'evaluateScript'
        console.log(`[perf] ${entry.entryType}.${entry.name}`, {
          path: entry.path,
          duration: entry.duration,
          startTime: entry.startTime
        })

        // 超阈值上报
        if (entry.name === 'firstRender' && entry.duration > 1000) {
          this.reportSlowRender(entry)
        }
      })
    })

    observer.observe({ entryTypes: ['navigation', 'render', 'script', 'loadPackage'] })
  }
})
```

| entryType | 典型 name | 含义 |
|-----------|-----------|------|
| `navigation` | `appLaunch` / `route` | 小程序启动、页面跳转耗时 |
| `render` | `firstRender` | 页面首次渲染耗时（首屏关键指标） |
| `script` | `evaluateScript` | 注入脚本执行耗时（代码包体积直接影响） |
| `loadPackage` | `loadPackage` | 代码包下载/加载耗时 |

### 自定义测速上报

`wx.reportPerformance(id, value, dimensions?)` 把业务耗时上报到「小程序管理后台 → 性能分析 → 自定义测速」。测速 id 需先在后台创建。**基础库 2.9.2+**。

```javascript
const start = Date.now()
await loadOrderList()
wx.reportPerformance(1001, Date.now() - start)   // 1001 为后台配置的测速 ID
```

### 工具链

- **开发者工具 → 调试器 → Audits（体验评分）**：自动检测节点数过多、图片过大、setData 频繁等问题，给出评分与整改项。
- **调试器 → Performance 面板**：查看逻辑层/渲染层时间线、setData 调用记录。
- **真机调试 → 性能面板**：CPU、内存、帧率实时曲线，只有真机数据可信（模拟器性能不代表真机）。
- **管理后台 → 性能分析**：线上大盘（启动耗时、页面渲染、内存、网络），按机型/网络分位数查看。

> ⚠️ 模拟器上 setData 耗时通常比真机低一个数量级，性能结论必须以真机（尤其是 Android 中低端机）为准。

## setData 深入优化

### 通信原理

小程序双线程架构下，逻辑层与渲染层不共享内存，setData 是唯一数据通道：

```
┌────────────────────────┐                          ┌────────────────────────┐
│      逻辑层 (JS)        │                          │    渲染层 (WebView)     │
│   AppService Thread     │                          │    View Thread          │
│                        │                          │                        │
│  this.setData(data)    │                          │                        │
│         │              │                          │                        │
│         ▼              │                          │                        │
│  ① JSON 序列化          │                          │                        │
└─────────┼──────────────┘                          └───────────▲────────────┘
          │                                                     │
          ▼                                                     │
   ┌──────────────────────────────────────────────────────────┐ │
   │                Native 层（微信客户端）                     │ │
   │   ② 跨线程消息传输（数据量越大，耗时越长）                   │ │
   └──────────────────────┬───────────────────────────────────┘ │
                          │ ③ evaluateJavascript 注入            │
                          ▼                                     │
              ④ JSON 反序列化 → ⑤ 虚拟 DOM diff → ⑥ 真实节点更新 ──┘
                                                                 
              ⑦ 渲染完成 → 回调 setData callback（回传逻辑层）
```

六个阶段中，**② 跨线程传输**与 **⑤ diff** 的耗时都与数据量正相关；**每次 setData 调用**都会完整走一遍全流程，与数据量无关的固定开销约 5～10ms（真机）。

由此得到两条主线：**减少调用次数** + **减少每次的数据量**。

### 1. 降低调用频率

❌ 循环内多次 setData：

```javascript
list.forEach((item, index) => {
  this.setData({ [`list[${index}].checked`]: true })   // N 次跨线程通信
})
```

✅ 合并为一次：

```javascript
const updates = {}
list.forEach((item, index) => {
  updates[`list[${index}].checked`] = true
})
updates.selectedCount = list.length
this.setData(updates)                                   // 1 次跨线程通信
```

✅ 跨函数的高频更新用微任务批处理（基于 `wx.nextTick`，基础库 2.7.1+）：

```javascript
// utils/batch-set-data.js
export function batchSetData(ctx, data) {
  if (!ctx._batchQueue) {
    ctx._batchQueue = {}
    wx.nextTick(() => {
      const queue = ctx._batchQueue
      ctx._batchQueue = null
      if (queue && Object.keys(queue).length) ctx.setData(queue)
    })
  }
  Object.assign(ctx._batchQueue, data)
}

// 使用：同一帧内多处调用只产生一次真实 setData
batchSetData(this, { count: 1 })
batchSetData(this, { title: 'hello' })
batchSetData(this, { 'user.name': 'Tom' })
```

### 2. 减少数据量

**只传变化的字段，用数据路径局部更新：**

```javascript
// ❌ 1000 条数据全量回传，只为改一个状态
const list = this.data.list
list[5].liked = true
this.setData({ list })

// ✅ 只传一个布尔值
this.setData({ 'list[5].liked': true })
```

**过滤掉渲染不需要的字段：**

```javascript
// ❌ 接口返回的完整对象（含 20+ 冗余字段）直接进 setData
this.setData({ list: res.data.list })

// ✅ 裁剪为渲染所需的最小结构
this.setData({
  list: res.data.list.map(({ id, title, cover, price }) => ({ id, title, cover, price }))
})
```

**长文本/大数组分片渲染，避免一次性阻塞：**

```javascript
renderInChunks(list, chunkSize = 20) {
  let index = 0
  const next = () => {
    if (index >= list.length) return
    const updates = {}
    list.slice(index, index + chunkSize).forEach((item, i) => {
      updates[`list[${index + i}]`] = item
    })
    this.setData(updates, () => {
      index += chunkSize
      setTimeout(next, 0)      // 让出主线程，保证交互响应
    })
  }
  next()
}
```

### 3. 不要把非渲染数据放进 data

WXML 里用不到的数据一律挂在实例属性上，完全不走跨线程通信：

```javascript
Page({
  data: {
    list: []          // ✅ 渲染需要
  },

  onLoad() {
    this._rawResponse = null    // ✅ 原始响应，仅逻辑层使用
    this._pageNum = 1           // ✅ 分页游标
    this._timer = null          // ✅ 定时器句柄
    this._scrollTop = 0         // ✅ 滚动位置缓存
  },

  onDataLoaded(res) {
    this._rawResponse = res                      // 不进 data
    this.setData({ list: this.formatList(res) }) // 只传渲染数据
  }
})
```

❌ 常见反模式：把整个接口响应、SDK 实例、Canvas 上下文、大 Map 塞进 `data`。这些数据每次 setData 都可能被序列化，且常驻内存。

### 4. 用 setData 回调获取渲染完成时机

`this.setData(data, callback)` 的 callback 在**渲染层更新完成后**触发，是测量真实渲染耗时和做后续 DOM 查询的正确时机：

```javascript
const start = Date.now()
this.setData({ list }, () => {
  console.log('渲染耗时', Date.now() - start)

  // 渲染完成后才能查到新节点的位置
  wx.createSelectorQuery()
    .select('#last-item')
    .boundingClientRect((rect) => { /* ... */ })
    .exec()
})
```

❌ 在 setData 之后立刻查询节点会拿到旧布局或 `null`：

```javascript
this.setData({ list })
wx.createSelectorQuery().select('#last-item').boundingClientRect(cb).exec()  // 可能为 null
```

### 5. 节流与防抖

**滚动、输入、拖拽、resize 场景必须节流/防抖**，否则 setData 会以每秒数十次的频率打爆通信通道。

```javascript
// utils/throttle.js —— 节流：固定间隔执行一次，首尾都触发
export function throttle(fn, interval = 100) {
  let lastTime = 0
  let timer = null
  return function (...args) {
    const now = Date.now()
    const remaining = interval - (now - lastTime)
    if (remaining <= 0) {
      if (timer) { clearTimeout(timer); timer = null }
      lastTime = now
      fn.apply(this, args)
    } else if (!timer) {
      timer = setTimeout(() => {
        lastTime = Date.now()
        timer = null
        fn.apply(this, args)
      }, remaining)
    }
  }
}

// 防抖：停止触发 delay 后才执行一次
export function debounce(fn, delay = 300, immediate = false) {
  let timer = null
  return function (...args) {
    if (timer) clearTimeout(timer)
    if (immediate && !timer) fn.apply(this, args)
    timer = setTimeout(() => {
      timer = null
      if (!immediate) fn.apply(this, args)
    }, delay)
  }
}
```

应用：

```javascript
const { throttle, debounce } = require('../../utils/throttle')

Page({
  onLoad() {
    this._onScroll = throttle((scrollTop) => {
      const showBackTop = scrollTop > 500
      if (showBackTop !== this.data.showBackTop) {
        this.setData({ showBackTop })       // ✅ 状态没变就不 setData
      }
    }, 100)

    this._onSearch = debounce((keyword) => this.search(keyword), 300)
  },

  onPageScroll(e) {
    this._onScroll(e.scrollTop)             // ✅ 节流 + 差异判断
  },

  onInput(e) {
    this._onSearch(e.detail.value)          // ✅ 输入防抖，减少请求与渲染
  },

  onUnload() {
    this._onScroll = null
    this._onSearch = null
  }
})
```

> 纯 UI 联动（如滚动视差、跟手动画）优先用 **WXS 响应事件** 或 **Skyline worklet**，逻辑运行在渲染层，完全不触发 setData。

## 渲染性能

### 减少 WXML 节点数

节点数直接决定首次渲染和每次 diff 的成本。经验阈值：**单页节点 < 1000，节点层级 < 20 层，单个节点子节点 < 60**。

```xml
<!-- ❌ 无意义的嵌套包裹，4 层节点只为渲染一行文字 -->
<view class="wrapper">
  <view class="container">
    <view class="content">
      <view class="text">{{title}}</view>
    </view>
  </view>
</view>

<!-- ✅ 扁平化，样式合并到一层 -->
<view class="title">{{title}}</view>
```

其他要点：

- 用 `<block>` 做逻辑包裹（`wx:if`/`wx:for`），它不生成真实节点。
- 列表项内部结构越简单越好，`item` 里的节点数会乘以列表长度。
- 避免在 WXML 里写复杂表达式（`{{list.filter(...).length}}`），改在 JS 里算好再传。
- 组件的 `styleIsolation` 用 `apply-shared`/`shared` 时会增加样式计算成本，非必要保持 `isolated`。

### hidden 与 wx:if 的取舍

| 维度 | `wx:if` | `hidden` |
|------|---------|----------|
| 条件为假时 | 节点不创建 / 销毁 | 节点存在，仅 `display: none` |
| 初始渲染开销 | 低（不渲染） | 高（始终渲染） |
| 切换开销 | 高（重建节点树、组件重新走生命周期） | 低（只改样式） |
| 组件状态 | 销毁后丢失 | 保留 |

```xml
<!-- ✅ 频繁切换（Tab 面板、折叠展开）用 hidden -->
<view hidden="{{activeTab !== 0}}">面板 A</view>
<view hidden="{{activeTab !== 1}}">面板 B</view>

<!-- ✅ 极少出现、内容重的用 wx:if（弹窗、空态、权限区块） -->
<view wx:if="{{showHeavyModal}}">
  <heavy-chart data="{{chartData}}" />
</view>
```

组合技：弹窗用 `wx:if` 控制首次挂载，之后用 `hidden` 控制显隐：

```xml
<view wx:if="{{everOpened}}" hidden="{{!visible}}">
  <expensive-content />
</view>
```

### 长列表优化

#### 方案一：分页加载（首选，实现成本最低）

```javascript
Page({
  data: { list: [], hasMore: true, loading: false },

  onLoad() {
    this._page = 1
    this._pageSize = 20
    this.loadMore()
  },

  async loadMore() {
    if (this.data.loading || !this.data.hasMore) return
    this.setData({ loading: true })

    try {
      const res = await api.getList({ page: this._page, size: this._pageSize })

      // ✅ 用数据路径逐条追加，已渲染的数据不回传
      const updates = { hasMore: res.list.length === this._pageSize, loading: false }
      const offset = this.data.list.length
      res.list.forEach((item, i) => { updates[`list[${offset + i}]`] = item })

      this.setData(updates)
      this._page++
    } catch (err) {
      this.setData({ loading: false })
    }
  },

  onReachBottom() {
    this.loadMore()
  }
})
```

❌ 对比：`this.setData({ list: this.data.list.concat(res.list) })` 在第 10 页时会把 200 条数据全量序列化回传一次，而索引路径写法始终只传新增的 20 条。

> 分页列表建议设置上限（如累计 200 条后清理头部或改用虚拟列表），否则节点数会持续增长直到卡顿。

#### 方案二：虚拟列表（数据量 > 数百条时必需）

只渲染可视窗口内的项，上下用占位高度撑开滚动条。**要求列表项高度固定或可预估**。

```xml
<!-- pages/feed/feed.wxml -->
<scroll-view
  class="feed"
  scroll-y
  bindscroll="onScroll"
  bindscrolltolower="loadMore"
  lower-threshold="200"
>
  <view style="height: {{topPlaceholder}}px"></view>

  <view
    class="item"
    wx:for="{{visibleList}}"
    wx:key="id"
    style="height: {{itemHeight}}px"
  >
    <image class="item__cover" src="{{item.cover}}" mode="aspectFill" lazy-load />
    <view class="item__title">{{item.title}}</view>
  </view>

  <view style="height: {{bottomPlaceholder}}px"></view>
</scroll-view>
```

```javascript
// pages/feed/feed.js
const { throttle } = require('../../utils/throttle')

const ITEM_HEIGHT_RPX = 200      // 列表项固定高度（rpx）
const BUFFER = 5                 // 上下各多渲染 5 项，避免快速滚动露白

Page({
  data: {
    visibleList: [],
    itemHeight: 0,
    topPlaceholder: 0,
    bottomPlaceholder: 0
  },

  onLoad() {
    const { windowWidth, windowHeight } = wx.getWindowInfo()
    this._itemHeight = (ITEM_HEIGHT_RPX * windowWidth) / 750       // rpx → px
    this._visibleCount = Math.ceil(windowHeight / this._itemHeight) + BUFFER * 2
    this._allList = []                                              // 全量数据不进 data
    this._startIndex = -1

    this.setData({ itemHeight: this._itemHeight })
    this._onScroll = throttle((scrollTop) => this.updateWindow(scrollTop), 50)
    this.loadMore()
  },

  onScroll(e) {
    this._scrollTop = e.detail.scrollTop
    this._onScroll(this._scrollTop)
  },

  updateWindow(scrollTop) {
    const start = Math.max(0, Math.floor(scrollTop / this._itemHeight) - BUFFER)
    if (start === this._startIndex) return                          // ✅ 窗口没变就不 setData
    this._startIndex = start

    const end = Math.min(this._allList.length, start + this._visibleCount)
    this.setData({
      visibleList: this._allList.slice(start, end),
      topPlaceholder: start * this._itemHeight,
      bottomPlaceholder: (this._allList.length - end) * this._itemHeight
    })
  },

  async loadMore() {
    const res = await api.getFeed({ offset: this._allList.length, size: 50 })
    this._allList = this._allList.concat(res.list)
    this._startIndex = -1
    this.updateWindow(this._scrollTop || 0)
  },

  onUnload() {
    this._allList = null
    this._onScroll = null
  }
})
```

要点：

- **全量数据存实例属性**（`this._allList`），只有可视窗口进 `data`。
- **窗口未变化时直接 return**，避免滚动过程中无意义的 setData。
- **必须写 `wx:key`**，否则复用失效，滚动时会整块重建。
- 高度不固定的场景，先用 `createSelectorQuery` 测量首屏项的真实高度做预估，或改用官方扩展 `miniprogram-recycle-view`（`<recycle-view>` / `<recycle-item>`）。
- Skyline 渲染引擎下可用 `<scroll-view type="custom">`，框架内置节点回收，无需手写虚拟列表。

### 图片优化

图片通常是小程序内存和首屏耗时的最大来源。

```xml
<!-- ✅ 懒加载 + 指定裁剪模式 + 服务端缩略图 -->
<image
  src="{{item.cover}}?imageView2/1/w/300/h/300/format/webp/q/80"
  mode="aspectFill"
  lazy-load
  webp
  binderror="onImageError"
/>
```

| 手段 | 做法 | 收益 |
|------|------|------|
| 懒加载 | `<image lazy-load>`（基础库 2.6.3+，仅在 `scroll-view` / page 中生效） | 首屏只加载可视区图片 |
| WebP | 图片 URL 加 `format/webp`，`<image webp>` 开启解析 | 体积比 JPEG 小 25%～35% |
| CDN 缩略图 | 按展示尺寸请求（七牛 `imageView2`、阿里云 `x-oss-process`、腾讯云 `imageMogr2`） | 列表图从 2MB 降到 30KB |
| 指定尺寸 | `mode="aspectFill"` + 固定容器宽高 | 避免图片加载后布局抖动（CLS） |
| 雪碧图 / iconfont | 小图标合并或改用字体/SVG | 减少请求数 |
| 本地图片瘦身 | 主包内图片 < 20KB，大图一律走 CDN | 直接减小代码包 |
| 预加载 | 关键首屏图提前 `wx.getImageInfo` 触发缓存 | 切页时秒开 |

```javascript
// 列表页预热详情页大图
prefetchDetailImage(url) {
  wx.getImageInfo({ url, fail: () => {} })      // 失败静默，仅用于填充缓存
}
```

> ⚠️ 大量高清图会迅速吃掉内存。长列表滚动时，虚拟列表 + `lazy-load` 是防止 iOS 内存告警的关键组合。

### 骨架屏

首屏数据到达前先渲染结构占位，把「白屏等待」变成「内容加载中」，显著提升体感速度。开发者工具支持右键页面 → **生成骨架屏代码**。

```xml
<!-- pages/index/index.wxml -->
<view wx:if="{{loading}}" class="skeleton">
  <view class="skeleton__banner shimmer"></view>
  <view class="skeleton__row" wx:for="{{[1,2,3,4]}}" wx:key="*this">
    <view class="skeleton__avatar shimmer"></view>
    <view class="skeleton__lines">
      <view class="skeleton__line shimmer" style="width: 70%"></view>
      <view class="skeleton__line shimmer" style="width: 45%"></view>
    </view>
  </view>
</view>

<view wx:else class="content">
  <!-- 真实内容 -->
</view>
```

```css
/* pages/index/index.wxss */
.shimmer {
  background: linear-gradient(90deg, #f2f3f5 25%, #e6e8eb 37%, #f2f3f5 63%);
  background-size: 400% 100%;
  animation: shimmer 1.4s ease infinite;
  border-radius: 8rpx;
}

@keyframes shimmer {
  0%   { background-position: 100% 50%; }
  100% { background-position: 0 50%; }
}

.skeleton__banner { width: 100%; height: 300rpx; margin-bottom: 24rpx; }
.skeleton__row    { display: flex; align-items: center; padding: 24rpx 0; }
.skeleton__avatar { width: 96rpx; height: 96rpx; border-radius: 50%; margin-right: 24rpx; }
.skeleton__lines  { flex: 1; }
.skeleton__line   { height: 28rpx; margin-bottom: 16rpx; }
```

```javascript
Page({
  data: { loading: true },
  async onLoad() {
    const data = await api.getHome()
    this.setData({ ...data, loading: false })   // ✅ 一次 setData 完成切换
  }
})
```

> 骨架屏结构要与真实内容的布局尺寸一致，否则切换瞬间会有明显跳动。骨架屏本身要极轻（纯 view + CSS，不含图片和组件）。

## 代码包优化

代码包体积直接决定**下载耗时**和 `evaluateScript` 耗时，是冷启动性能的第一因素。

### 分包加载

```json
// app.json
{
  "pages": [
    "pages/index/index",
    "pages/category/category"
  ],
  "subPackages": [
    {
      "root": "packageGoods",
      "name": "goods",
      "pages": ["pages/detail/detail", "pages/comment/comment"]
    },
    {
      "root": "packageUser",
      "name": "user",
      "pages": ["pages/profile/profile", "pages/settings/settings"]
    },
    {
      "root": "packageActivity",
      "name": "activity",
      "pages": ["pages/lottery/lottery"],
      "independent": true
    }
  ],
  "preloadRule": {
    "pages/index/index": { "network": "wifi", "packages": ["goods"] },
    "pages/category/category": { "network": "all", "packages": ["goods"] }
  }
}
```

- **主包只放 Tab 页和公共依赖**，业务页面全部下沉到分包。
- `preloadRule` 在进入指定页面后台预下载分包，配合 `network: "wifi"` 避免消耗用户流量（`packages` 里填 `name` 或 `root`）。
- 分包内的资源（图片、JSON）跟随分包走，不占主包体积。

### 独立分包

`"independent": true` 的分包**不依赖主包即可启动**，适合广告落地页、活动页、扫码入口页——用户从外部直达时无需下载主包，启动速度提升明显。

约束：

- 独立分包内**不能 require 主包代码**，也不能用主包的自定义组件。
- `getApp()` 默认拿不到主包 App 实例，需 `getApp({ allowDefault: true })`，且主包 `App.onLaunch` 未必执行过。
- 公共逻辑要在独立分包内复制一份，或抽成不依赖 App 实例的纯函数模块。

```javascript
// 独立分包页面中安全地访问 App
const app = getApp({ allowDefault: true })
const token = (app.globalData && app.globalData.token) || wx.getStorageSync('token')
```

### 分包异步化

跨分包引用 JS 与组件（基础库 2.17.3+），把重依赖挪出主包：

```javascript
// 主包页面里异步加载分包中的重型模块
require('../../packageGoods/utils/chart.js', (chart) => {
  chart.render(this.data.points)
}, ({ errMsg }) => {
  console.error('分包加载失败', errMsg)
})
```

```json
// 页面 json：跨分包引用组件 + 加载占位组件
{
  "usingComponents": {
    "heavy-chart": "../../packageGoods/components/chart/index"
  },
  "componentPlaceholder": {
    "heavy-chart": "view"
  }
}
```

### 按需引入，避免整包依赖

```javascript
// ❌ 引入整个 lodash（约 70KB+），只用了一个函数
import _ from 'lodash'
_.debounce(fn, 300)

// ✅ 子路径引入，只打包用到的模块
import debounce from 'lodash/debounce'

// ✅✅ 更好：小工具自己实现，零依赖（见前文 throttle/debounce）
import { debounce } from '../../utils/throttle'
```

其他常见体积杀手：

| 依赖 | 问题 | 替代方案 |
|------|------|----------|
| `moment` | 200KB+，含全部语言包 | `dayjs`（2KB）或自写格式化函数 |
| `lodash` | 全量引入 70KB+ | 子路径引入 / 自实现 |
| 完整 UI 库 | 全量注册所有组件 | 只在页面 `usingComponents` 中按需引用 |
| `echarts` | 完整包 400KB+ | 自定义构建裁剪图表类型，并放进分包 |

npm 依赖通过「工具 → 构建 npm」生成 `miniprogram_npm`，只有被引用的模块会被打包，因此**子路径引入是关键**。

### 代码压缩与产物瘦身

```json
// project.config.json
{
  "setting": {
    "es6": true,
    "enhance": true,
    "minified": true,
    "minifyWXML": true,
    "minifyWXSS": true,
    "uglifyFileName": true,
    "postcss": true,
    "urlCheck": true
  }
}
```

上传代码时勾选「压缩代码」「过滤无用文件」。此外：

- 用「开发者工具 → 代码依赖分析」找出体积大户和未被引用的文件。
- 删除 demo 页面、未使用的组件、示例图片。
- 字体文件、大图、音视频一律放 CDN，通过网络请求加载。
- `.gitignore` 之外再配 `packOptions.ignore`，把文档、测试文件排除出代码包：

```json
// app.json
{
  "packOptions": {
    "ignore": [
      { "type": "folder", "value": "docs" },
      { "type": "suffix", "value": ".md" }
    ]
  }
}
```

## 启动性能

冷启动链路：**代码包下载 → 注入执行（App.onLaunch）→ 页面 onLoad → 首次渲染**。优化目标是让网络请求与渲染准备并行。

### 首屏数据预请求

在 `App.onLaunch` 就发起首屏请求，页面 `onLoad` 时直接消费 Promise，省下「页面初始化 + 组件注册」这段时间：

```javascript
// app.js
App({
  globalData: { homeDataPromise: null },

  onLaunch() {
    // ✅ 不 await，立即发起，与页面初始化并行
    this.globalData.homeDataPromise = this.prefetchHome()
  },

  prefetchHome() {
    return new Promise((resolve) => {
      wx.request({
        url: `${API_BASE}/home`,
        success: (res) => resolve(res.data),
        fail: () => resolve(null)        // 失败返回 null，由页面回落到正常请求
      })
    })
  }
})
```

```javascript
// pages/index/index.js
const app = getApp()

Page({
  data: { loading: true },

  async onLoad() {
    const prefetched = app.globalData.homeDataPromise
    app.globalData.homeDataPromise = null           // 只消费一次

    const data = (prefetched && await prefetched) || await api.getHome()
    this.setData({ ...data, loading: false })
  }
})
```

**登录也要并行，不要串行阻塞首屏：**

```javascript
// ❌ 先登录拿 token，再请求首屏 —— 两次 RTT 串行
await login()
const data = await api.getHome()

// ✅ 非敏感首屏数据与登录并行
const [_, data] = await Promise.all([login(), api.getHome()])
```

### 数据预拉取（BackgroundFetch）

在管理后台「开发 → 开发设置 → 数据预拉取」配置后，微信会在**小程序启动前**就向开发者服务器发起请求，冷启动时直接读缓存：

```javascript
// app.js
onLaunch() {
  wx.getBackgroundFetchData({
    fetchType: 'pre',
    success: (res) => {
      const data = JSON.parse(res.fetchedData)
      this.globalData.preData = data
    },
    fail: () => {}
  })
}
```

### 首屏用骨架屏而非 Loading 图标

见前文「骨架屏」。原则：**onLoad 中同步 setData 出骨架结构，数据到达后一次性替换**，中间不要有多次局部 setData 造成的闪烁。

### 避免同步 API 阻塞

同步 API 会阻塞逻辑层线程，期间无法处理任何渲染和交互。

```javascript
// ❌ 启动路径上的同步调用
onLaunch() {
  const info = wx.getSystemInfoSync()           // 部分机型 > 50ms
  const cache = wx.getStorageSync('bigCache')   // 大数据反序列化耗时
  const userInfo = wx.getStorageSync('user')
}

// ✅ 用新版分包信息 API（更快）并缓存结果；大数据异步读取
onLaunch() {
  this.globalData.windowInfo = wx.getWindowInfo()      // 基础库 2.20.1+
  this.globalData.appBaseInfo = wx.getAppBaseInfo()

  wx.getStorage({
    key: 'bigCache',
    success: ({ data }) => { this.globalData.cache = data },
    fail: () => {}
  })
}
```

| 同步 API | 替代 |
|----------|------|
| `wx.getSystemInfoSync()` | `wx.getWindowInfo()` / `wx.getDeviceInfo()` / `wx.getAppBaseInfo()`，且结果缓存复用 |
| `wx.getStorageSync(bigKey)` | `wx.getStorage`（异步），或拆分 key 只同步读小数据 |
| `wx.setStorageSync(bigData)` | `wx.setStorage`，并在写入前裁剪数据 |

其他启动期禁忌：

- `App.onLaunch` / `Page.onLoad` 里跑复杂计算（大数组排序、加解密、JSON 深拷贝）——挪到首屏渲染后或用 Worker。
- 在 `app.js` 顶层 import 大量模块——`evaluateScript` 阶段就要全部执行。
- 启动时并发发起十几个请求——微信同时最多 10 个 `wx.request` 并发，超出会排队，关键请求反而被拖慢。

## 网络优化

### 并行请求

```javascript
// ❌ 串行，总耗时 = t1 + t2 + t3
const user = await api.getUser()
const banner = await api.getBanner()
const goods = await api.getGoods()

// ✅ 并行，总耗时 = max(t1, t2, t3)
const [user, banner, goods] = await Promise.all([
  api.getUser(),
  api.getBanner(),
  api.getGoods()
])

// ✅ 非关键接口失败不阻塞首屏
const [user, banner] = await Promise.all([
  api.getUser(),
  api.getBanner().catch(() => ({ list: [] }))
])
```

### 请求缓存（TTL + 并发去重）

```javascript
// utils/request-cache.js
const cache = new Map()

function makeKey(options) {
  return `${options.method || 'GET'}:${options.url}:${JSON.stringify(options.data || {})}`
}

function rawRequest(options) {
  return new Promise((resolve, reject) => {
    wx.request({
      ...options,
      success: (res) => {
        if (res.statusCode >= 200 && res.statusCode < 300) resolve(res.data)
        else reject(new Error(`HTTP ${res.statusCode}`))
      },
      fail: reject
    })
  })
}

/**
 * ttl: 缓存有效期（毫秒），0 表示不缓存
 */
export function requestWithCache(options, ttl = 60000) {
  const key = makeKey(options)
  const hit = cache.get(key)

  if (hit) {
    if (hit.promise) return hit.promise                       // 并发去重：复用在途请求
    if (Date.now() - hit.time < ttl) return Promise.resolve(hit.data)
    cache.delete(key)
  }

  const promise = rawRequest(options)
    .then((data) => {
      if (ttl > 0) cache.set(key, { data, time: Date.now() })
      else cache.delete(key)
      return data
    })
    .catch((err) => {
      cache.delete(key)                                        // 失败不缓存
      throw err
    })

  cache.set(key, { promise })
  return promise
}

export function invalidateCache(urlPrefix) {
  for (const key of cache.keys()) {
    if (!urlPrefix || key.includes(urlPrefix)) cache.delete(key)
  }
}
```

```javascript
// 使用
const config = await requestWithCache({ url: `${API}/config` }, 10 * 60 * 1000)  // 配置类缓存 10 分钟
const list = await requestWithCache({ url: `${API}/list`, data: { page: 1 } })    // 默认 1 分钟

// 提交后失效相关缓存
await api.createOrder(payload)
invalidateCache('/orders')
```

> 内存缓存进程结束即失效；跨启动的缓存用 `wx.setStorage` 并自带 `expireAt` 字段，读取时校验过期。注意 Storage 总量上限 10MB。

### 页面间数据预加载

列表页在用户可能点击前，就把详情数据拉好：

```javascript
// utils/prefetch-store.js
const store = new Map()

export function prefetch(key, loader) {
  if (store.has(key)) return store.get(key)
  const p = loader().catch(() => null)
  store.set(key, p)
  setTimeout(() => store.delete(key), 30000)   // 30s 后自动清理，避免常驻内存
  return p
}

export function take(key) {
  const p = store.get(key)
  store.delete(key)
  return p
}
```

```javascript
// 列表页：长按 / 曝光 / touchstart 时预取
onItemTouchStart(e) {
  const { id } = e.currentTarget.dataset
  prefetch(`detail:${id}`, () => api.getDetail(id))
},

onItemTap(e) {
  const { id } = e.currentTarget.dataset
  wx.navigateTo({ url: `/packageGoods/pages/detail/detail?id=${id}` })
}
```

```javascript
// 详情页：优先消费预取结果
async onLoad({ id }) {
  const pending = take(`detail:${id}`)
  const detail = (pending && await pending) || await api.getDetail(id)
  this.setData({ detail, loading: false })
}
```

其他网络手段：

- **合并接口**：首屏多个小接口合成一个聚合接口，减少 RTT 和并发占用。
- **压缩响应**：服务端开 gzip；裁剪返回字段（前端用不到的不返回）。
- **失败重试**：只对幂等 GET 做 1～2 次退避重试，避免雪崩。
- **弱网降级**：`wx.getNetworkType` / `wx.onNetworkStatusChange`，2G/3G 下降低图片质量、关闭自动播放。

## 内存优化

### onUnload / detached 中彻底清理

```javascript
Page({
  onLoad() {
    this._timer = setInterval(() => this.tick(), 1000)
    this._countdown = setTimeout(() => this.expire(), 60000)

    this._onNetworkChange = (res) => this.handleNetwork(res)
    wx.onNetworkStatusChange(this._onNetworkChange)

    this._audio = wx.createInnerAudioContext()

    this._observer = wx.createIntersectionObserver(this)
    this._observer.relativeToViewport().observe('.lazy-item', () => {})

    eventBus.on('cart:update', this.onCartUpdate)
  },

  onUnload() {
    clearInterval(this._timer)
    clearTimeout(this._countdown)
    this._timer = this._countdown = null

    wx.offNetworkStatusChange(this._onNetworkChange)   // ✅ 必须传同一函数引用

    this._audio.stop()
    this._audio.destroy()
    this._audio = null

    this._observer.disconnect()
    this._observer = null

    eventBus.off('cart:update', this.onCartUpdate)

    this._allList = null                                // ✅ 释放大数组引用
    this._rawResponse = null
  }
})
```

自定义组件用 `lifetimes.detached` 做同样的清理。

| 泄漏源 | 症状 | 处理 |
|--------|------|------|
| `setInterval` / `setTimeout` 未清 | 页面关闭后仍在跑，回调里 setData 报错 | `onUnload` 中 clear |
| `wx.onXxx` 监听未解绑 | 每次进页面叠加一个监听，触发 N 次 | 成对调用 `wx.offXxx`，传相同引用 |
| 事件总线未 off | 同上，且持有已销毁页面实例 | `off` 时传相同的函数引用 |
| `InnerAudioContext` / `VideoContext` / `MapContext` | 音频后台继续播、内存不释放 | `stop()` + `destroy()` |
| `IntersectionObserver` | 观察器常驻 | `disconnect()` |
| 全局数组无限追加（日志、埋点队列） | 内存持续增长直至崩溃 | 设上限，超出丢弃或落盘 |
| Canvas / 大图缓存 | iOS 内存告警、页面被回收 | 及时清空引用，限制缓存条目数 |

### 用分页替代全量加载

```javascript
// ❌ 一次拿 5000 条，全部进 data
const all = await api.getAll()
this.setData({ list: all })

// ✅ 分页 + 虚拟列表；全量数据留在实例属性上
this._allList = this._allList.concat(page.list)
this.updateWindow(this._scrollTop)
```

配合上限策略：

```javascript
const MAX_KEEP = 300
if (this._allList.length > MAX_KEEP) {
  this._allList = this._allList.slice(-MAX_KEEP)      // 只保留最近 300 条
}
```

### 响应系统内存告警

```javascript
// app.js
onLaunch() {
  wx.onMemoryWarning((res) => {
    // res.level: 5(TRIM_MEMORY_RUNNING_MODERATE) / 10(LOW) / 15(CRITICAL)，iOS 无 level
    console.warn('内存告警', res.level)
    invalidateCache()                  // 清请求缓存
    imageCache.clear()                 // 清图片/数据缓存
  })
}
```

> iOS 上收到 CRITICAL 告警后若不释放内存，小程序会被系统直接杀掉，表现为「闪退回到聊天列表」。长列表 + 大图页面是重灾区。

## 性能优化检查清单

### 启动

- [ ] 主包 < 2MB，非 Tab 页全部下沉分包
- [ ] 配置 `preloadRule` 预下载高频分包（`network: "wifi"`）
- [ ] 外部直达页（活动/广告落地页）使用独立分包
- [ ] `App.onLaunch` 中并行发起首屏请求，页面 onLoad 直接消费
- [ ] 登录与首屏数据请求并行，不串行阻塞
- [ ] 启动路径无同步 API（`getSystemInfoSync` / 大数据 `getStorageSync`）
- [ ] 首屏使用骨架屏而非空白或 loading 图标
- [ ] 已开启代码压缩（`minified` / `minifyWXML` / `minifyWXSS`）
- [ ] 依赖按需引入，无 `moment`、全量 `lodash`

### 运行时（setData）

- [ ] 无循环内多次 setData，已合并为单次
- [ ] 使用数据路径 `'list[0].x'` 局部更新，无大数组全量回传
- [ ] 单次 setData 数据量 < 256KB，频率 < 10 次/秒
- [ ] 非渲染数据挂实例属性（`this._xxx`），不进 `data`
- [ ] `onPageScroll` / `bindscroll` / `bindinput` 已节流或防抖
- [ ] setData 前做差异判断，状态未变不调用
- [ ] 依赖渲染结果的逻辑放在 setData callback 中
- [ ] 纯 UI 联动优先用 WXS / Skyline worklet，不走 setData

### 渲染

- [ ] 单页节点数 < 1000，层级 < 20，无无意义嵌套
- [ ] `wx:for` 全部指定 `wx:key`
- [ ] 频繁切换用 `hidden`，低频重内容用 `wx:if`
- [ ] 列表超过 100 项使用分页；超过数百项使用虚拟列表 / `recycle-view` / Skyline `scroll-view type="custom"`
- [ ] 图片开启 `lazy-load`，使用 WebP 与 CDN 缩略图
- [ ] 图片容器固定尺寸，无加载后布局抖动
- [ ] WXML 中无复杂表达式计算
- [ ] 长任务已分片（`setTimeout` 让出主线程）

### 内存

- [ ] `onUnload` / `detached` 清理定时器、监听、Context、Observer
- [ ] `wx.offXxx` 传入与 `wx.onXxx` 相同的函数引用
- [ ] 音视频 Context 调用了 `stop()` + `destroy()`
- [ ] 大数组/大对象引用在页面卸载时置 null
- [ ] 全局缓存有条目上限或 TTL，不无限增长
- [ ] 监听 `wx.onMemoryWarning` 并释放缓存
- [ ] 真机测试内存占用 < 200MB，长时间滚动无持续增长

### 网络

- [ ] 无依赖关系的请求用 `Promise.all` 并行
- [ ] 配置类/低频变更接口有缓存（TTL + 并发去重）
- [ ] 相同请求并发时已去重
- [ ] 首屏聚合接口，避免十几个小请求争抢并发额度
- [ ] 高概率跳转的详情数据做了预取
- [ ] 弱网下有降级策略（低质量图、跳过非关键请求）
- [ ] 请求失败有重试上限与超时设置，不做无限重试

### 验证

- [ ] 用真机（含 Android 中低端机）而非模拟器测量
- [ ] 开发者工具 Audits 体验评分无高优问题
- [ ] `wx.getPerformance` 观测 `firstRender` / `evaluateScript` 达标
- [ ] 关键业务链路配置了 `wx.reportPerformance` 自定义测速
- [ ] 上线后在管理后台「性能分析」跟踪线上分位数

## 官方文档

- 性能与体验总览：https://developers.weixin.qq.com/miniprogram/dev/framework/performance/
- 启动性能优化建议：https://developers.weixin.qq.com/miniprogram/dev/framework/performance/tips/start.html
- 运行时性能优化建议：https://developers.weixin.qq.com/miniprogram/dev/framework/performance/tips/runtime.html
- 体验评分（Audits）：https://developers.weixin.qq.com/miniprogram/dev/devtools/audits.html
- 性能数据 `wx.getPerformance`：https://developers.weixin.qq.com/miniprogram/dev/api/base/performance/wx.getPerformance.html
- 自定义测速 `wx.reportPerformance`：https://developers.weixin.qq.com/miniprogram/dev/api/base/performance/wx.reportPerformance.html
- 分包加载：https://developers.weixin.qq.com/miniprogram/dev/framework/subpackages/
- 数据预拉取：https://developers.weixin.qq.com/miniprogram/dev/framework/ability/pre-fetch.html
