# 微信小程序基础库版本兼容性指南

小程序运行在微信客户端提供的**基础库**之上。同一份代码在不同版本的基础库中可用的 API 并不相同，因此在使用较新的能力时必须做版本判断与降级处理，否则低版本用户会直接遇到 `xxx is not a function` 或静默失败。

---

## 常用 API 版本要求

| API | 最低基础库版本 | 说明 |
|---|---|---|
| `wx.request` | 1.0.0 | 发起 HTTPS 网络请求，全版本可用 |
| `wx.login` | 1.0.0 | 获取登录凭证 code，换取 openid/session_key |
| `wx.getSystemInfo` | 1.0.0 | 获取系统信息；2.20.1 起推荐改用 `wx.getSystemSetting` / `wx.getDeviceInfo` / `wx.getWindowInfo` / `wx.getAppBaseInfo` 等拆分接口 |
| `wx.chooseImage` | 1.0.0 | 选择图片；2.21.0 起标记为不再维护，推荐 `wx.chooseMedia` |
| `wx.getLocation` | 1.0.0 | 获取当前地理位置，需在 `app.json` 中声明 `requiredPrivateInfos` 与用户授权 |
| `wx.scanCode` | 1.0.0 | 调起客户端扫码界面 |
| `wx.getPhoneNumber` | 1.0.0 | 手机号快速验证；以 `button` 的 `open-type="getPhoneNumber"` 形式使用，需企业主体且已开通权限 |
| `wx.getSetting` | 1.2.0 | 获取用户当前的授权状态，用于判断是否需要引导重新授权 |
| `wx.authorize` | 1.2.0 | 提前发起授权请求；用户拒绝后需通过 `wx.openSetting` 引导 |
| `wx.getUserInfo` | 1.3.0 | 获取用户信息；自 2021-04-13 起**已逐步回收**，仅返回匿名数据（灰色头像、"微信用户"昵称），新项目不要再使用 |
| `wx.navigateToMiniProgram` | 1.3.0 | 跳转到其他小程序，需在 `app.json` 的 `navigateToMiniProgramAppIdList` 中声明目标 AppID |
| `wx.showShareMenu` | 1.4.0 | 显示当前页面的转发按钮；`withShareTicket` / `menus` 参数有更高版本要求 |
| `wx.pageScrollTo` | 1.4.0 | 将页面滚动到目标位置，可指定动画时长 |
| `wx.createSelectorQuery` | 1.4.0 | 节点查询，获取节点布局位置、尺寸等信息 |
| `wx.createIntersectionObserver` | 1.6.0 | 交叉状态监听，常用于图片懒加载、曝光埋点 |
| `wx.setTabBarBadge` | 1.9.0 | 为 tabBar 某一项右上角添加文本徽标 |
| `wx.getMenuButtonBoundingClientRect` | 2.1.0 | 获取胶囊按钮布局信息，自定义导航栏必用（同步接口） |
| `wx.cloud` | 2.2.3 | 云开发能力入口，使用前需 `wx.cloud.init()` |
| `wx.requestSubscribeMessage` | 2.4.4 | 订阅消息授权；必须由用户点击等手势事件触发 |
| `wx.onKeyboardHeightChange` | 2.7.0 | 监听键盘高度变化，用于聊天/输入类页面布局适配 |
| `wx.getRealtimeLogManager` | 2.7.1 | 实时日志管理器，日志可在小程序后台"实时日志"中查看 |
| `wx.chooseMedia` | 2.10.0 | 统一的图片/视频选择接口，替代 `wx.chooseImage` 和 `wx.chooseVideo` |
| `wx.getUserProfile` | 2.10.4 | 获取用户头像昵称的推荐方式，每次调用都会弹窗且必须由手势触发；2.27.1 起微信推荐改用「头像昵称填写能力」 |
| `wx.getPerformance` | 2.11.0 | 获取性能数据管理器，配合 `PerformanceObserver` 采集渲染/脚本耗时 |
| `wx.openCustomerServiceChat` | 2.12.0 | 打开微信客服会话（企业微信客服），需在后台绑定客服账号 |

> 提示：完整且权威的版本要求以官方文档每个 API 页面顶部的「基础库 x.x.x 开始支持」标注为准。新增 API 前先查文档，不要凭记忆。

---

## 版本兼容性检查

### 通用版本比较工具

```js
// utils/version.js

/**
 * 比较两个版本号，v1 > v2 返回 1，v1 < v2 返回 -1，相等返回 0
 */
function compareVersion(v1, v2) {
  const a = String(v1).split('.')
  const b = String(v2).split('.')
  const len = Math.max(a.length, b.length)

  while (a.length < len) a.push('0')
  while (b.length < len) b.push('0')

  for (let i = 0; i < len; i++) {
    const num1 = parseInt(a[i], 10)
    const num2 = parseInt(b[i], 10)
    if (num1 > num2) return 1
    if (num1 < num2) return -1
  }
  return 0
}

/**
 * 判断当前基础库版本是否 >= 目标版本
 */
function checkVersion(target) {
  const { SDKVersion } = wx.getSystemInfoSync()
  return compareVersion(SDKVersion, target) >= 0
}

module.exports = { compareVersion, checkVersion }
```

### 使用示例

```js
const { checkVersion } = require('../../utils/version.js')

Page({
  onLoad() {
    if (checkVersion('2.10.4')) {
      this.loginWithUserProfile()
    } else {
      // 低版本走服务端静默登录，不获取头像昵称
      this.loginSilently()
      wx.showModal({
        title: '提示',
        content: '当前微信版本过低，部分功能无法使用，请升级到最新微信版本后重试。',
        showCancel: false,
      })
    }
  },
})
```

---

## 兼容性处理最佳实践

### 1. 优先使用 `wx.canIUse` 而非版本号

`wx.canIUse` 可以精确到**接口、参数、返回值字段、组件属性**级别，比手写版本号更准确，也不会因为版本号记错而误判。

```js
// 接口是否存在
wx.canIUse('getUserProfile')

// 接口的某个入参
wx.canIUse('request.object.timeout')

// 回调返回值中的字段
wx.canIUse('getSystemInfo.return.safeArea')

// 组件属性
wx.canIUse('button.open-type.getPhoneNumber')
```

### 2. 渐进式降级：新 API → 旧 API → 用户提示

按能力从高到低依次尝试，保证任何版本的用户都有一条可走的路径。

```js
function chooseImages(count = 9) {
  return new Promise((resolve, reject) => {
    // 一级：2.10.0+ 统一媒体选择接口
    if (wx.canIUse('chooseMedia')) {
      wx.chooseMedia({
        count,
        mediaType: ['image'],
        success: (res) => resolve(res.tempFiles.map((f) => f.tempFilePath)),
        fail: reject,
      })
      return
    }

    // 二级：1.0.0+ 旧接口降级
    if (wx.canIUse('chooseImage')) {
      wx.chooseImage({
        count,
        success: (res) => resolve(res.tempFilePaths),
        fail: reject,
      })
      return
    }

    // 三级：能力完全不可用，明确提示用户升级
    wx.showModal({
      title: '无法选择图片',
      content: '当前微信版本不支持该功能，请升级微信后重试。',
      showCancel: false,
    })
    reject(new Error('chooseImage/chooseMedia not supported'))
  })
}
```

### 3. 用 `try-catch` 包裹同步接口

同步接口（如 `wx.getMenuButtonBoundingClientRect`、`wx.getPerformance`）在低版本中可能不存在或直接抛错，必须兜底。

```js
function getMenuButtonRect() {
  try {
    return wx.getMenuButtonBoundingClientRect()
  } catch (err) {
    // 低版本兜底：使用经验值，避免自定义导航栏错位
    return { top: 26, height: 32, width: 87, right: 368 }
  }
}

function getPerformanceObserver() {
  if (typeof wx.getPerformance !== 'function') return null
  try {
    return wx.getPerformance()
  } catch (err) {
    return null
  }
}
```

### 4. 其他实践要点

- **能力检测放在调用点附近**，不要在 `app.js` 里一次性判断后全局缓存布尔值——部分能力还依赖运行时权限与账号配置。
- **降级路径必须真实可用**，不要写一个空的 `else` 分支假装兼容；无法降级时要给出明确的用户提示。
- **异步接口统一用 `fail` 回调兜底**，即使版本满足，也可能因用户拒绝授权、账号未开通权限而失败。
- **在真机上验证低版本表现**，微信开发者工具可在「详情 → 本地设置 → 调试基础库」中切换版本进行回归测试。

---

## 基础库版本设置

在 `project.config.json` 中通过 `libVersion` 字段指定开发与调试使用的基础库版本：

```json
{
  "libVersion": "3.8.12",
  "setting": {
    "es6": true,
    "minified": true
  }
}
```

线上用户实际可运行的**最低基础库版本**需在「微信公众平台 → 管理 → 版本管理 → 基础库最低版本设置」中配置。低于该版本的用户打开小程序时，会被引导升级微信客户端。设置得越高，可用 API 越多、兼容代码越少，但会损失一部分用户。

---

## 版本分布参考

以下为线上基础库版本占比（数据更新于 **2026-07-29**）：

| 基础库版本 | 总体占比 |
|-----------|---------|
| 3.17.0 | 33.62% |
| 3.16.2 | 55.10% |
| 3.15.3 | 4.79% |
| 3.14.3 | 0.45% |
| 3.13.2 | 0.69% |
| 3.12.1 | 0.47% |
| 3.11.3 | 0.53% |
| 3.10.3 | 0.75% |
| 3.9.3 | 0.39% |
| 3.8.12 | 1.10% |
| 3.7.12 | 0.31% |
| < 3.7 | ~1.7% |

### 累计覆盖率

- **3.16.2 及以上**：约 **88.7%** 的用户
- **3.15.3 及以上**：约 **93.5%** 的用户
- **3.8.12 及以上**：约 **98%** 的用户

### 建议

- **推荐将最低基础库版本设置为 `3.8.12`**，可覆盖绝大多数用户（约 98%），同时避免为不足 2% 的长尾版本编写大量兼容代码。
- 若产品需要使用较新的能力且能接受少量用户流失，可上调至 `3.15.3`（覆盖约 93.5%）。
- 注意：基础库版本号目前已进入 **3.x 系列**，旧文档中常见的 "2.x" 版本号只用于描述 API 的**最低支持版本**（如 `wx.getUserProfile` 需 2.10.4），并不代表线上仍有大量 2.x 用户。实际上，当前线上 3.7 以下版本合计仅约 1.7%，因此表格中标注的 2.x 级别 API 在绝大多数设备上都已可用；真正需要重点关注的是 3.x 时代新增的能力。
- 版本分布会随时间快速变化，发版前建议到「微信公众平台 → 管理 → 版本管理 → 基础库最低版本设置」页面查看最新的实时数据，不要长期沿用本文档中的静态快照。
