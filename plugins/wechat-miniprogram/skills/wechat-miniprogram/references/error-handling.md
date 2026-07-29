# 微信小程序错误处理与错误码

小程序的错误来源分散在三层：客户端 API（`wx.*` 的 `fail` 回调）、HTTP 请求（业务后端返回的状态码与业务码）、以及微信服务端接口（`errcode`/`errmsg`）。三者的错误结构完全不同，直接把原始错误抛给用户会得到 `request:fail timeout` 这类无法理解的文案。本文提供一套统一的收敛方案：把所有错误规范化成内部错误码，再统一决定提示、降级和上报行为。

## 目录

- [统一错误处理模板](#统一错误处理模板)
- [使用示例](#使用示例)
- [错误码对照表](#错误码对照表)
- [自定义错误码规范](#自定义错误码规范)
- [错误上报](#错误上报)

---

## 统一错误处理模板

建议放在 `utils/error-handler.js`，全局单例使用。

```js
// utils/error-handler.js

/**
 * 内部统一错误码
 */
const ERROR_CODES = {
  NETWORK_ERROR: 1001,      // 网络不可用 / 请求失败
  TIMEOUT_ERROR: 1002,      // 请求超时
  AUTH_DENIED: 2001,        // 用户拒绝授权
  AUTH_EXPIRED: 2002,       // 登录态过期，需重新登录
  PARAM_INVALID: 3001,      // 参数校验不通过
  RESOURCE_NOT_FOUND: 3002, // 资源不存在
  OPERATION_FAILED: 3003,   // 业务操作失败
  SYSTEM_ERROR: 5001,       // 服务端 / 系统异常
  UNKNOWN_ERROR: 9999       // 未归类错误
}

/**
 * 错误码 -> 用户可读文案
 */
const ERROR_MESSAGES = {
  [ERROR_CODES.NETWORK_ERROR]: '网络连接失败，请检查网络后重试',
  [ERROR_CODES.TIMEOUT_ERROR]: '请求超时，请稍后重试',
  [ERROR_CODES.AUTH_DENIED]: '需要您的授权才能继续',
  [ERROR_CODES.AUTH_EXPIRED]: '登录已过期，请重新登录',
  [ERROR_CODES.PARAM_INVALID]: '参数有误，请检查后重试',
  [ERROR_CODES.RESOURCE_NOT_FOUND]: '内容不存在或已被删除',
  [ERROR_CODES.OPERATION_FAILED]: '操作失败，请稍后重试',
  [ERROR_CODES.SYSTEM_ERROR]: '服务开小差了，请稍后重试',
  [ERROR_CODES.UNKNOWN_ERROR]: '出错了，请稍后重试'
}

class ErrorHandler {
  /**
   * 统一错误入口
   * @param {Error|Object} error 原始错误对象
   * @param {Object} options
   * @param {boolean} options.showToast  是否弹出提示，默认 true
   * @param {boolean} options.logError   是否上报，默认 true
   * @param {Function} options.fallback  降级处理函数，收到规范化后的错误
   * @param {string} options.scene       业务场景标识，便于上报归因
   * @returns {Object} 规范化后的错误 { code, message, raw }
   */
  static handle(error, options = {}) {
    const {
      showToast = true,
      logError = true,
      fallback = null,
      scene = ''
    } = options

    const normalized = ErrorHandler.parseError(error)

    if (showToast) {
      wx.showToast({
        title: normalized.message,
        icon: 'none',
        duration: 2000
      })
    }

    if (logError) {
      ErrorHandler.reportError(normalized, scene)
    }

    // 登录态过期统一跳转，避免每个调用点各写一遍
    if (normalized.code === ERROR_CODES.AUTH_EXPIRED) {
      ErrorHandler.redirectToLogin()
    }

    if (typeof fallback === 'function') {
      fallback(normalized)
    }

    return normalized
  }

  /**
   * 把任意来源的错误规范化为 { code, message, raw }
   */
  static parseError(error) {
    if (!error) {
      return ErrorHandler.build(ERROR_CODES.UNKNOWN_ERROR)
    }

    // 已经是内部规范化错误，直接透传
    if (typeof error.code === 'number' && error.message && error.raw !== undefined) {
      return error
    }

    // 业务层主动抛出的错误：{ code: 3001, message: 'xxx' }
    if (typeof error.code === 'number' && ERROR_MESSAGES[error.code]) {
      return ErrorHandler.build(error.code, error.message, error)
    }

    // wx.* API 的 fail 回调
    if (typeof error.errMsg === 'string') {
      return ErrorHandler.parseWxError(error)
    }

    // HTTP 响应
    if (typeof error.statusCode === 'number') {
      return ErrorHandler.parseHttpError(error)
    }

    return ErrorHandler.build(
      ERROR_CODES.UNKNOWN_ERROR,
      error.message || undefined,
      error
    )
  }

  /**
   * 解析 wx API 的 errMsg
   */
  static parseWxError(error) {
    const errMsg = error.errMsg || ''

    if (errMsg.includes('auth deny') || errMsg.includes('auth denied') ||
        errMsg.includes('cancel')) {
      return ErrorHandler.build(ERROR_CODES.AUTH_DENIED, undefined, error)
    }

    if (errMsg.includes('timeout')) {
      return ErrorHandler.build(ERROR_CODES.TIMEOUT_ERROR, undefined, error)
    }

    if (errMsg.includes('fail') &&
        (errMsg.includes('network') || errMsg.includes('ERR_FAILED') ||
         errMsg.includes('ssl') || errMsg.includes('socket') ||
         errMsg.includes('interrupted'))) {
      return ErrorHandler.build(ERROR_CODES.NETWORK_ERROR, undefined, error)
    }

    if (errMsg.includes('request:fail')) {
      return ErrorHandler.build(ERROR_CODES.NETWORK_ERROR, undefined, error)
    }

    return ErrorHandler.build(ERROR_CODES.UNKNOWN_ERROR, undefined, error)
  }

  /**
   * 解析 HTTP 状态码
   */
  static parseHttpError(error) {
    const status = error.statusCode
    const serverMsg = (error.data && error.data.message) || undefined

    if (status === 401) {
      return ErrorHandler.build(ERROR_CODES.AUTH_EXPIRED, serverMsg, error)
    }
    if (status === 403) {
      return ErrorHandler.build(ERROR_CODES.AUTH_DENIED, serverMsg, error)
    }
    if (status === 404) {
      return ErrorHandler.build(ERROR_CODES.RESOURCE_NOT_FOUND, serverMsg, error)
    }
    if (status === 400 || status === 422) {
      return ErrorHandler.build(ERROR_CODES.PARAM_INVALID, serverMsg, error)
    }
    if (status === 408 || status === 504) {
      return ErrorHandler.build(ERROR_CODES.TIMEOUT_ERROR, serverMsg, error)
    }
    if (status >= 500) {
      return ErrorHandler.build(ERROR_CODES.SYSTEM_ERROR, serverMsg, error)
    }
    if (status >= 400) {
      return ErrorHandler.build(ERROR_CODES.OPERATION_FAILED, serverMsg, error)
    }

    return ErrorHandler.build(ERROR_CODES.UNKNOWN_ERROR, serverMsg, error)
  }

  static build(code, message, raw = null) {
    return {
      code,
      message: message || ERROR_MESSAGES[code] || ERROR_MESSAGES[ERROR_CODES.UNKNOWN_ERROR],
      raw
    }
  }

  /**
   * 上报到实时日志（开发者工具「实时日志」可查）
   */
  static reportError(normalized, scene = '') {
    if (!wx.getRealtimeLogManager) return

    const logger = wx.getRealtimeLogManager()
    const pages = getCurrentPages()
    const current = pages.length ? pages[pages.length - 1] : null

    // filterMsg 支持在实时日志后台按关键字检索
    logger.addFilterMsg(`error_${normalized.code}`)
    logger.error('[error]', {
      code: normalized.code,
      message: normalized.message,
      scene,
      page: current ? current.route : 'unknown',
      raw: normalized.raw ? String(normalized.raw.errMsg || normalized.raw.message || '') : ''
    })
  }

  static redirectToLogin() {
    const pages = getCurrentPages()
    const current = pages.length ? pages[pages.length - 1] : null
    const from = current ? encodeURIComponent(current.route) : ''
    wx.navigateTo({ url: `/pages/login/index?from=${from}` })
  }

  /**
   * 包装异步函数，自动捕获并处理异常
   * @param {Function} fn 异步函数
   * @param {Object} options 同 handle 的 options
   * @returns {Function} 包装后的函数，出错时返回 null
   */
  static wrap(fn, options = {}) {
    return async function (...args) {
      try {
        return await fn.apply(this, args)
      } catch (error) {
        ErrorHandler.handle(error, options)
        return null
      }
    }
  }
}

ErrorHandler.CODES = ERROR_CODES

module.exports = { ErrorHandler, ERROR_CODES }
```

---

## 使用示例

### 模式一：try-catch + ErrorHandler.handle

适合需要在 catch 里做额外收尾（关闭 loading、恢复按钮状态）的场景。

```js
const { ErrorHandler, ERROR_CODES } = require('../../utils/error-handler')

Page({
  data: { list: [], loading: false },

  async loadList() {
    this.setData({ loading: true })
    try {
      const res = await api.getList({ page: 1 })
      this.setData({ list: res.data })
    } catch (error) {
      ErrorHandler.handle(error, {
        scene: 'list_page_load',
        fallback: () => this.setData({ list: [] })
      })
    } finally {
      this.setData({ loading: false })
    }
  }
})
```

### 模式二：ErrorHandler.wrap 包装

适合"出错就提示、无需额外收尾"的简单调用，省掉样板 try-catch。

```js
const { ErrorHandler } = require('../../utils/error-handler')

Page({
  onSubmit: ErrorHandler.wrap(async function (e) {
    const res = await api.submitOrder(e.detail.value)
    wx.showToast({ title: '提交成功', icon: 'success' })
    wx.navigateTo({ url: `/pages/order/detail?id=${res.orderId}` })
  }, { scene: 'order_submit' }),

  // 静默场景：只上报不打扰用户
  refreshBadge: ErrorHandler.wrap(async function () {
    const count = await api.getUnreadCount()
    this.setData({ badge: count })
  }, { showToast: false, scene: 'badge_refresh' })
})
```

### 模式三：按错误码分支处理

需要针对特定错误做差异化交互时，先规范化再分支，避免在业务代码里匹配 `errMsg` 字符串。

```js
const { ErrorHandler, ERROR_CODES } = require('../../utils/error-handler')

Page({
  async onChooseLocation() {
    try {
      const res = await wx.chooseLocation()
      this.setData({ address: res.address })
    } catch (error) {
      const normalized = ErrorHandler.parseError(error)

      switch (normalized.code) {
        case ERROR_CODES.AUTH_DENIED:
          // 用户拒绝授权：引导去设置页开启，不需要 toast
          wx.showModal({
            title: '需要位置权限',
            content: '请在设置中开启位置权限后重新选择',
            confirmText: '去设置',
            success: (r) => { if (r.confirm) wx.openSetting() }
          })
          break

        case ERROR_CODES.NETWORK_ERROR:
        case ERROR_CODES.TIMEOUT_ERROR:
          // 网络类错误：提供重试入口
          this.setData({ showRetry: true })
          ErrorHandler.handle(normalized, { scene: 'choose_location' })
          break

        default:
          ErrorHandler.handle(normalized, { scene: 'choose_location' })
      }
    }
  }
})
```

---

## 错误码对照表

### 全局错误码

| 错误码 | 含义 | 说明 |
| --- | --- | --- |
| -1 | 系统繁忙 | 微信服务端临时故障，稍后重试即可，不要立刻重试 |
| 0 | 请求成功 | 正常返回 |

### 登录相关

| 错误码 | 含义 | 处理建议 |
| --- | --- | --- |
| 40001 | AppSecret 错误或 access_token 无效 | 检查服务端 AppSecret 配置；确认使用的是小程序而非公众号凭证 |
| 40029 | code 无效 | `wx.login` 的 code 只能用一次且 5 分钟内有效，重新调用 `wx.login` |
| 40163 | code 已被使用 | 同一个 code 重复换取 session_key，检查是否有重复请求 |
| 41008 | 缺少 code 参数 | 客户端未把 code 传给服务端 |
| 42001 | access_token 超时 | 服务端刷新 access_token，注意全局缓存并提前刷新 |
| 42002 | refresh_token 超时 | 需要重新走完整授权流程 |

### 授权相关

| 错误码 | 含义 | 处理建议 |
| --- | --- | --- |
| 89019 | 用户未授权 / 缺少对应权限 | 引导用户 `wx.openSetting` 开启，或检查小程序类目是否具备该接口权限 |
| 50001 | 未授权命令 | 接口未开通或账号无权调用，去 MP 后台检查接口权限 |

### 频率限制

| 错误码 | 含义 | 处理建议 |
| --- | --- | --- |
| 45009 | 接口调用超过限制 | 加本地缓存与请求合并，避免重复调用；关注每日配额 |
| 45047 | 客服消息下发条数超过上限 | 单用户 24 小时内下发条数受限，改用订阅消息 |

### 网络请求错误（`errMsg` 关键字）

| errMsg 片段 | 含义 | 处理建议 |
| --- | --- | --- |
| `request:fail timeout` | 请求超时 | 提升 `timeout` 配置或做重试；提示用户网络较慢 |
| `request:fail ssl hand shake error` | SSL 握手失败 | 服务端证书过期/不完整，或用户设备时间不正确 |
| `request:fail -2 net::ERR_FAILED` | 请求失败 | 通常是域名未在 MP 后台配置 request 合法域名 |
| `request:fail createSocket ...` / `socket` | 建连失败 | 网络不可达、DNS 解析失败或服务端不可用 |
| `request:fail interrupted` | 请求被中断 | 页面卸载或用户切后台导致，一般可忽略不上报 |
| `request:fail url not in domain list` | 域名未配置 | 在 MP 后台「开发设置 - 服务器域名」添加，开发期可勾选「不校验合法域名」 |

### 云开发错误码

| 错误码 | 含义 | 处理建议 |
| --- | --- | --- |
| -401001 | 未登录 / 鉴权失败 | 检查 `wx.cloud.init` 的 env 与调用时机 |
| -402001 | 权限不足 | 检查数据库/存储的安全规则与集合权限设置 |
| -403001 | 请求被拒绝 | 常见于安全规则拦截或超出资源限制 |
| -404001 | 资源不存在 | 集合、文档、云函数或文件 ID 不存在 |
| -501001 | 云函数执行错误 | 查看云函数日志定位真实异常 |
| -502001 | 数据库请求失败 | 检查查询语句、索引与集合状态 |
| -503001 | 存储服务错误 | 检查文件 ID、文件大小与存储配额 |

### 支付相关

| 错误码 | 含义 | 处理建议 |
| --- | --- | --- |
| 1000 | 支付成功 | 以服务端支付回调为准，不要仅凭前端结果发货 |
| 1001 | 用户取消支付 | 正常行为，不要弹错误提示 |
| 1002 | 支付参数错误 | 检查服务端下单返回的 `timeStamp`/`nonceStr`/`package`/`paySign` |
| 1003 | 签名错误 | 校验签名算法、商户密钥与参数顺序 |
| 1004 | 支付超时 | 提示用户重新发起，服务端需查单确认最终状态 |

> 前端支付结果只用于交互提示，订单状态一律以服务端支付回调 + 主动查单为准。

---

## 自定义错误码规范

业务自定义错误码统一使用 **4 位 ABCD** 格式，便于按级别过滤告警、按模块归因。

```
A B CD
│ │ └── 序号（01-99），模块内自增
│ └──── 模块（0-4）
└────── 级别（1-5）
```

**A 位 — 错误级别**

| 值 | 级别 | 含义 |
| --- | --- | --- |
| 1 | 网络层 | 网络、超时、连接类问题 |
| 2 | 鉴权层 | 登录态、授权、权限 |
| 3 | 业务层 | 参数校验、业务规则不满足 |
| 4 | 数据层 | 数据不一致、解析失败 |
| 5 | 系统层 | 服务端异常、依赖不可用 |

**B 位 — 所属模块**

| 值 | 模块 |
| --- | --- |
| 0 | 通用/基础 |
| 1 | 用户 |
| 2 | 订单 |
| 3 | 支付 |
| 4 | 内容/其他业务 |

**CD 位 — 模块内序号**，从 `01` 递增到 `99`。

**示例**

| 错误码 | 拆解 | 含义 |
| --- | --- | --- |
| 1001 | 1-网络 / 0-通用 / 01 | 网络请求失败 |
| 2101 | 2-鉴权 / 1-用户 / 01 | 用户登录态过期 |
| 3201 | 3-业务 / 2-订单 / 01 | 订单不存在或已关闭 |
| 3202 | 3-业务 / 2-订单 / 02 | 库存不足，下单失败 |
| 3301 | 3-业务 / 3-支付 / 01 | 支付金额与订单金额不一致 |
| 5001 | 5-系统 / 0-通用 / 01 | 服务端内部错误 |

约定：
- 错误码一旦上线不可复用，废弃就废弃，避免历史日志歧义。
- 前后端共用同一张码表，建议由服务端维护并生成前端常量文件。
- `9999` 保留给未归类错误，出现即说明有未覆盖的错误来源，需要补充解析规则。

---

## 错误上报

`wx.getRealtimeLogManager` 适合排查单个用户的问题，但不便于做聚合统计。生产环境建议再加一层批量上报到自建服务：队列缓冲 + 定时/满量 flush，避免错误风暴打爆接口。

```js
// utils/error-reporter.js

class ErrorReporter {
  constructor(options = {}) {
    this.queue = []
    this.maxQueueSize = options.maxQueueSize || 20
    this.flushInterval = options.flushInterval || 10000
    this.reportUrl = options.reportUrl || 'https://api.example.com/log/error'
    this.timer = null
    this.startTimer()
  }

  startTimer() {
    if (this.timer) return
    this.timer = setInterval(() => this.flush(), this.flushInterval)
  }

  stopTimer() {
    if (!this.timer) return
    clearInterval(this.timer)
    this.timer = null
  }

  /**
   * 收集一条错误
   */
  report(error, extra = {}) {
    const pages = getCurrentPages()
    const current = pages.length ? pages[pages.length - 1] : null

    this.queue.push({
      message: (error && (error.message || error.errMsg)) || String(error),
      stack: (error && error.stack) || '',
      code: (error && error.code) || 0,
      timestamp: Date.now(),
      page: current ? current.route : 'unknown',
      system: this.getSystemInfo(),
      ...extra
    })

    // 队列满了立刻 flush，避免丢失
    if (this.queue.length >= this.maxQueueSize) {
      this.flush()
    }
  }

  getSystemInfo() {
    if (this._system) return this._system
    const device = wx.getDeviceInfo ? wx.getDeviceInfo() : {}
    const app = wx.getAppBaseInfo ? wx.getAppBaseInfo() : {}
    this._system = {
      brand: device.brand,
      model: device.model,
      system: device.system,
      platform: device.platform,
      sdkVersion: app.SDKVersion,
      version: app.version
    }
    return this._system
  }

  /**
   * 上报并清空队列
   */
  flush() {
    if (!this.queue.length) return

    const logs = this.queue.splice(0, this.queue.length)

    wx.request({
      url: this.reportUrl,
      method: 'POST',
      data: { logs },
      // 上报失败不再重试，避免错误上报本身产生错误的死循环
      fail: () => {}
    })
  }
}

const reporter = new ErrorReporter()

/**
 * 在 app.js 的 onLaunch 中调用一次
 */
function initGlobalErrorHandler() {
  // 未捕获的同步/运行时错误
  wx.onError((error) => {
    reporter.report(error, { type: 'js_error' })
  })

  // 未处理的 Promise rejection
  if (wx.onUnhandledRejection) {
    wx.onUnhandledRejection(({ reason }) => {
      reporter.report(reason, { type: 'unhandled_rejection' })
    })
  }

  // 页面不存在
  if (wx.onPageNotFound) {
    wx.onPageNotFound((res) => {
      reporter.report(new Error(`page not found: ${res.path}`), { type: 'page_not_found' })
    })
  }

  // 小程序退到后台时立即上报，避免进程回收丢日志
  if (wx.onAppHide) {
    wx.onAppHide(() => reporter.flush())
  }
}

module.exports = { reporter, initGlobalErrorHandler }
```

在 `app.js` 中接入：

```js
const { initGlobalErrorHandler } = require('./utils/error-reporter')

App({
  onLaunch() {
    initGlobalErrorHandler()
  }
})
```

**上报实践要点**

- **不要在 `fail` 回调里再次上报**：`flush` 的失败回调必须为空实现，否则错误上报失败会触发新的上报，形成死循环。
- **采样**：日活较大时对非致命错误做采样（如 10%），避免上报接口成为瓶颈。
- **去重**：相同 `message + page` 在短时间内只保留一条，防止循环中的错误刷屏。
- **切后台时 flush**：`onAppHide` 是最后的上报时机，小程序被回收后队列中的日志会丢失。
- **区分环境**：开发版/体验版建议直接 `console.error` 不上报，只在正式版启用上报。
