# 微信小程序测试与调试指南

小程序的测试体系分为三层：**单元测试**（miniprogram-simulate，跑在 Node 环境，验证组件与工具函数逻辑）、**自动化测试**（miniprogram-automator，驱动真实开发者工具，验证页面跳转与交互流程）、**真机调试与性能测试**（真机调试 2.0、实时日志、Performance API，验证线上真实表现）。

## 目录

- [1. 单元测试](#1-单元测试)
- [2. 自动化测试](#2-自动化测试)
- [3. 真机调试](#3-真机调试)
- [4. 性能测试](#4-性能测试)
- [5. 调试技巧](#5-调试技巧)

---

## 1. 单元测试

`miniprogram-simulate` 在 Node 环境模拟小程序的组件运行时，配合 Jest 即可对自定义组件和工具函数做单元测试，无需启动开发者工具。

### 1.1 安装依赖

```bash
npm install --save-dev jest miniprogram-simulate
# 若组件使用 ES6+ 语法需要转译
npm install --save-dev babel-jest @babel/core @babel/preset-env
```

`package.json` 中添加脚本：

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}
```

### 1.2 Jest 配置

在项目根目录创建 `jest.config.js`：

```js
module.exports = {
  testEnvironment: 'jsdom',
  moduleFileExtensions: ['js', 'json'],
  testMatch: [
    '**/test/**/*.test.js',
    '**/__tests__/**/*.js'
  ],
  collectCoverageFrom: [
    'miniprogram/components/**/*.js',
    'miniprogram/utils/**/*.js',
    '!**/node_modules/**'
  ],
  coverageDirectory: 'coverage',
  setupFiles: ['<rootDir>/test/setup.js'],
  transform: {
    '^.+\\.js$': 'babel-jest'
  }
}
```

> `testEnvironment` 必须是 `jsdom`：miniprogram-simulate 依赖 DOM API 来模拟渲染树。
> `collectCoverageFrom` 只统计业务源码，排除 `node_modules` 与测试文件本身。

`test/setup.js` 用于注入 Jest 环境下缺失的全局对象（`wx` 上的部分 API 需要手动 mock）：

```js
global.wx = global.wx || {}

wx.getStorageSync = jest.fn(() => '')
wx.setStorageSync = jest.fn()
wx.showToast = jest.fn()
wx.request = jest.fn()
```

### 1.3 组件测试

被测组件 `components/counter/index.js`：

```js
Component({
  properties: {
    count: { type: Number, value: 0 },
    disabled: { type: Boolean, value: false }
  },
  data: {
    label: '点击'
  },
  methods: {
    onTap() {
      if (this.data.disabled) return
      const next = this.data.count + 1
      this.setData({ count: next })
      this.triggerEvent('change', { count: next })
    }
  }
})
```

对应测试 `test/counter.test.js`：

```js
const simulate = require('miniprogram-simulate')
const path = require('path')

describe('counter 组件', () => {
  let id

  beforeAll(() => {
    // 加载组件，返回组件 id
    id = simulate.load(path.join(__dirname, '../components/counter/index'))
  })

  test('按 properties 渲染初始值', () => {
    const comp = simulate.render(id, { count: 5 })
    comp.attach(document.createElement('parent-wrapper'))

    expect(comp.data.count).toBe(5)
    expect(comp.instance.data.label).toBe('点击')
  })

  test('点击后计数加一并派发 change 事件', () => {
    const comp = simulate.render(id, { count: 0 })
    comp.attach(document.createElement('parent-wrapper'))

    const handler = jest.fn()
    comp.instance.triggerEvent = handler

    const btn = comp.querySelector('.counter-btn')
    btn.dispatchEvent('tap')

    expect(comp.data.count).toBe(1)
    expect(handler).toHaveBeenCalledWith('change', { count: 1 })
  })

  test('disabled 状态下点击无效', () => {
    const comp = simulate.render(id, { count: 0, disabled: true })
    comp.attach(document.createElement('parent-wrapper'))

    comp.querySelector('.counter-btn').dispatchEvent('tap')

    expect(comp.data.count).toBe(0)
  })

  test('querySelectorAll 匹配多个子节点', () => {
    const comp = simulate.render(id, { count: 3 })
    comp.attach(document.createElement('parent-wrapper'))

    const items = comp.querySelectorAll('.item')
    expect(items.length).toBe(3)
  })
})
```

常用 API：

| API | 说明 |
| --- | --- |
| `simulate.load(path)` | 加载组件，返回组件 id；也可传 `{ tagName, template, usingComponents }` 定义内联组件 |
| `simulate.render(id, properties)` | 用给定 properties 渲染组件实例 |
| `comp.attach(parentNode)` | 挂载到 DOM，触发 `attached` / `ready` 生命周期 |
| `comp.detach()` | 卸载组件，触发 `detached` |
| `comp.data` | 组件当前 data |
| `comp.instance` | 组件实例，可访问 `setData`、自定义方法 |
| `comp.querySelector(selector)` | 按 class / tag 查找单个节点 |
| `comp.querySelectorAll(selector)` | 查找全部匹配节点 |
| `node.dispatchEvent(name, detail)` | 派发事件，如 `tap`、`input`、`change` |
| `simulate.sleep(ms)` | 等待若干毫秒（setData 异步刷新后再断言） |

> `setData` 是异步的：断言渲染结果前用 `await simulate.sleep(10)`，断言 `comp.data` 则可以同步进行。

### 1.4 工具函数测试

`utils/format.js`：

```js
function formatTime(date) {
  const pad = (n) => `${n}`.padStart(2, '0')
  return [date.getFullYear(), pad(date.getMonth() + 1), pad(date.getDate())].join('/') +
    ' ' +
    [pad(date.getHours()), pad(date.getMinutes()), pad(date.getSeconds())].join(':')
}

function debounce(fn, wait = 300) {
  let timer = null
  return function (...args) {
    if (timer) clearTimeout(timer)
    timer = setTimeout(() => {
      timer = null
      fn.apply(this, args)
    }, wait)
  }
}

module.exports = { formatTime, debounce }
```

测试 `test/format.test.js`：

```js
const { formatTime, debounce } = require('../utils/format')

describe('formatTime', () => {
  test('补零格式化到秒', () => {
    expect(formatTime(new Date(2024, 0, 5, 9, 8, 7))).toBe('2024/01/05 09:08:07')
  })

  test('两位数月份日期不额外补零', () => {
    expect(formatTime(new Date(2024, 10, 25, 23, 59, 59))).toBe('2024/11/25 23:59:59')
  })
})

describe('debounce', () => {
  beforeEach(() => jest.useFakeTimers())
  afterEach(() => jest.useRealTimers())

  test('多次调用只执行最后一次', () => {
    const fn = jest.fn()
    const debounced = debounce(fn, 300)

    debounced('a')
    debounced('b')
    debounced('c')
    expect(fn).not.toHaveBeenCalled()

    jest.advanceTimersByTime(300)
    expect(fn).toHaveBeenCalledTimes(1)
    expect(fn).toHaveBeenCalledWith('c')
  })

  test('间隔超过 wait 时分别执行', () => {
    const fn = jest.fn()
    const debounced = debounce(fn, 300)

    debounced()
    jest.advanceTimersByTime(300)
    debounced()
    jest.advanceTimersByTime(300)

    expect(fn).toHaveBeenCalledTimes(2)
  })

  test('保留调用时的 this', () => {
    const fn = jest.fn()
    const ctx = { name: 'ctx', run: debounce(fn, 100) }

    ctx.run()
    jest.advanceTimersByTime(100)

    expect(fn.mock.instances[0]).toBe(ctx)
  })
})
```

> 涉及定时器的逻辑（debounce / throttle / 轮询 / 倒计时）一律使用 `jest.useFakeTimers()` + `jest.advanceTimersByTime()`，避免测试挂在真实等待上。

---

## 2. 自动化测试

`miniprogram-automator` 通过开发者工具的自动化端口驱动真实小程序运行，适合做端到端的页面流程测试。

### 2.1 安装与准备

```bash
npm install --save-dev miniprogram-automator jest
```

前置条件：

1. 开发者工具中开启 **设置 → 安全设置 → CLI/HTTP 调用功能**。
2. 关闭正在占用项目的开发者工具窗口（同一项目不能被重复打开）。
3. 记录开发者工具 CLI 路径：
   - macOS：`/Applications/wechatwebdevtools.app/Contents/MacOS/cli`
   - Windows：`C:\Program Files (x86)\Tencent\微信web开发者工具\cli.bat`

自动化测试耗时较长，单独用一份 Jest 配置 `jest.e2e.config.js`：

```js
module.exports = {
  testEnvironment: 'node',
  testMatch: ['**/e2e/**/*.test.js'],
  testTimeout: 60000,
  maxWorkers: 1
}
```

> `maxWorkers: 1`：开发者工具同一时间只能被一个自动化会话控制，并发会导致连接失败。

### 2.2 测试脚本示例

`e2e/login.test.js`：

```js
const automator = require('miniprogram-automator')
const path = require('path')

let miniProgram

describe('登录与首页流程', () => {
  beforeAll(async () => {
    miniProgram = await automator.launch({
      cliPath: '/Applications/wechatwebdevtools.app/Contents/MacOS/cli',
      projectPath: path.join(__dirname, '..'),
      // 可选：projectConfig 中已有 appid 时无需再传
      timeout: 60000
    })
  }, 60000)

  afterAll(async () => {
    await miniProgram && miniProgram.close()
  })

  beforeEach(async () => {
    // 每个用例都从首页干净状态开始
    await miniProgram.reLaunch('/pages/index/index')
    await miniProgram.currentPage()
  })

  test('首页标题正确渲染', async () => {
    const page = await miniProgram.currentPage()
    expect(page.path).toBe('pages/index/index')

    const title = await page.$('.page-title')
    expect(await title.text()).toBe('欢迎使用')
  })

  test('搜索框输入后跳转到结果页', async () => {
    const page = await miniProgram.currentPage()

    const input = await page.$('.search-input')
    await input.input('小程序')

    const btn = await page.$('.search-btn')
    await btn.tap()

    // 等待跳转完成
    await page.waitFor(500)
    const result = await miniProgram.currentPage()
    expect(result.path).toBe('pages/search/index')

    const items = await result.$$('.result-item')
    expect(items.length).toBeGreaterThan(0)
  })

  test('列表加载完成后显示数据', async () => {
    const page = await miniProgram.currentPage()

    // 等待选择器出现，而不是硬编码 sleep
    await page.waitFor('.list-item')

    const items = await page.$$('.list-item')
    expect(items.length).toBe(10)
    expect(await items[0].attribute('data-id')).toBe('1')
  })

  test('直接调用页面方法并校验 data', async () => {
    const page = await miniProgram.currentPage()

    await page.callMethod('loadMore')
    await page.waitFor(300)

    const data = await page.data()
    expect(data.list.length).toBe(20)
    expect(data.hasMore).toBe(true)
  })

  test('详情页返回后回到列表页', async () => {
    await miniProgram.navigateTo('/pages/detail/index?id=1')
    let page = await miniProgram.currentPage()
    expect(page.path).toBe('pages/detail/index')
    expect(page.query).toEqual({ id: '1' })

    await miniProgram.navigateBack()
    page = await miniProgram.currentPage()
    expect(page.path).toBe('pages/index/index')
  })

  test('切换 tabBar 到我的页面', async () => {
    await miniProgram.switchTab('/pages/mine/index')
    const page = await miniProgram.currentPage()
    expect(page.path).toBe('pages/mine/index')
  })
})
```

运行：

```bash
npx jest --config jest.e2e.config.js
```

### 2.3 API 速查

**automator / miniProgram（小程序级）**

| API | 说明 |
| --- | --- |
| `automator.launch({ cliPath, projectPath })` | 启动开发者工具并打开项目，返回 miniProgram 实例 |
| `automator.connect({ wsEndpoint })` | 连接已启动的开发者工具（复用已有窗口） |
| `miniProgram.reLaunch(url)` | 关闭所有页面，打开指定页面 |
| `miniProgram.navigateTo(url)` | 保留当前页面，跳转到新页面 |
| `miniProgram.redirectTo(url)` | 关闭当前页面，跳转到新页面 |
| `miniProgram.navigateBack()` | 返回上一页 |
| `miniProgram.switchTab(url)` | 切换到 tabBar 页面 |
| `miniProgram.currentPage()` | 获取当前页面对象（含 `path`、`query`） |
| `miniProgram.pageStack()` | 获取页面栈 |
| `miniProgram.callWxMethod(method, ...args)` | 调用 wx API，如 `callWxMethod('setStorageSync', 'k', 'v')` |
| `miniProgram.mockWxMethod(method, result)` | mock wx API 返回值（如登录态、定位） |
| `miniProgram.evaluate(fn, ...args)` | 在小程序上下文执行函数 |
| `miniProgram.systemInfo()` | 获取系统信息 |
| `miniProgram.close()` | 关闭小程序与开发者工具连接 |

**page（页面级）**

| API | 说明 |
| --- | --- |
| `page.$(selector)` | 获取单个元素 |
| `page.$$(selector)` | 获取元素数组 |
| `page.waitFor(condition)` | 等待：数字=毫秒，字符串=等待选择器出现，函数=等待返回真值 |
| `page.callMethod(name, ...args)` | 调用页面 Page 中定义的方法 |
| `page.data(path?)` | 获取页面 data，可传路径如 `'list[0].name'` |
| `page.setData(obj)` | 设置页面 data |
| `page.size()` | 页面尺寸 |
| `page.scrollTop()` | 页面滚动位置 |

**element（元素级）**

| API | 说明 |
| --- | --- |
| `element.tap()` | 点击 |
| `element.longpress()` | 长按 |
| `element.input(text)` | 输入文本（input / textarea） |
| `element.text()` | 获取节点文本 |
| `element.attribute(name)` | 获取节点属性 |
| `element.property(name)` | 获取节点属性值（自定义组件为 properties） |
| `element.style(name)` | 获取样式值 |
| `element.trigger(type, detail)` | 触发自定义事件 |
| `element.size()` / `element.offsetLeft()` / `element.offsetTop()` | 尺寸与位置 |
| `element.callMethod(name, ...args)` | 调用自定义组件方法 |
| `element.data(path?)` | 获取自定义组件 data |

> 优先用 `page.waitFor('.selector')` 或 `page.waitFor(() => ...)` 而不是固定 `waitFor(1000)`，可显著降低 flaky 率。

---

## 3. 真机调试

### 3.1 开启真机调试 2.0

1. 开发者工具工具栏点击 **真机调试**。
2. 调试模式选择 **真机调试 2.0**（相比 1.0 支持断点调试、Sources 面板、更完整的 Network 面板）。
3. 手机微信扫描弹出的二维码，保持工具窗口不关闭。
4. 工具中会打开调试器窗口，可用 Console / Sources / Network / Storage 面板直接调试真机运行的代码。

常见问题：

- 手机与电脑需在同一网络，或允许工具的网络访问；企业网络的隔离策略常导致连接失败。
- 真机调试下 `console.log` 输出在调试器 Console，不在开发者工具主窗口。
- 编译模式选择「普通编译」以外的自定义编译条件时，真机调试会以该条件启动。

### 3.2 实时日志（wx.getRealtimeLogManager）

真机调试只能覆盖开发期，线上问题需要实时日志。日志在 mp 后台「开发 → 运维中心 → 实时日志」查看，保留 7 天。

`utils/logger.js`：

```js
const logManager = wx.getRealtimeLogManager ? wx.getRealtimeLogManager() : null

function withMeta(args) {
  return [
    {
      page: getCurrentPages().slice(-1)[0]?.route || 'unknown',
      ts: Date.now()
    },
    ...args
  ]
}

const logger = {
  info(...args) {
    console.log(...args)
    logManager && logManager.info(...withMeta(args))
  },

  warn(...args) {
    console.warn(...args)
    logManager && logManager.warn(...withMeta(args))
  },

  error(...args) {
    console.error(...args)
    logManager && logManager.error(...withMeta(args))
  },

  // 给当前页面日志打标记，方便在后台按 filterMsg 精确检索
  setFilterMsg(msg) {
    logManager && logManager.setFilterMsg(msg)
  },

  addFilterMsg(msg) {
    logManager && logManager.addFilterMsg(msg)
  }
}

module.exports = logger
```

使用：

```js
const logger = require('../../utils/logger')

Page({
  onLoad(options) {
    logger.setFilterMsg(`order:${options.id}`)
    logger.info('订单页加载', options)
  },

  async fetchOrder(id) {
    try {
      const res = await request(`/order/${id}`)
      logger.info('订单加载成功', { id, status: res.status })
    } catch (err) {
      logger.error('订单加载失败', { id, message: err.message })
    }
  }
})
```

要点：

- 单条日志上限 5KB，单页面日志总量上限 5MB，超出会被截断，不要打印整个大对象。
- `error` 级别日志才会在实时日志后台标红并支持按错误检索，异常路径务必用 `error`。
- `setFilterMsg` 覆盖当前页面的检索标记，`addFilterMsg` 追加（最多 5 条），用业务 id 做标记可快速定位单个用户的问题。
- 实时日志只在真机生效，开发者工具中 `wx.getRealtimeLogManager` 返回的实例调用无效果（因此要做存在性判断）。

### 3.3 远程调试辅助工具

真机上无法直接看变量时，可用一个轻量的调试面板把关键状态回显到界面或日志：

`utils/remote-debug.js`：

```js
const logger = require('./logger')

const remoteDebug = {
  enabled: false,

  init() {
    const { envVersion } = wx.getAccountInfoSync().miniProgram
    // 体验版 / 开发版开启，正式版关闭
    this.enabled = envVersion !== 'release'
    return this.enabled
  },

  // 打印当前页面栈与栈顶页面 data
  dumpPages() {
    if (!this.enabled) return
    const pages = getCurrentPages()
    logger.info('页面栈', pages.map((p) => p.route))
    const top = pages[pages.length - 1]
    top && logger.info('栈顶 data', top.data)
  },

  // 打印设备与运行环境信息
  dumpEnv() {
    if (!this.enabled) return
    const info = wx.getSystemInfoSync()
    logger.info('运行环境', {
      brand: info.brand,
      model: info.model,
      system: info.system,
      SDKVersion: info.SDKVersion,
      version: info.version
    })
  },

  // 打印本地存储占用
  dumpStorage() {
    if (!this.enabled) return
    const { keys, currentSize, limitSize } = wx.getStorageInfoSync()
    logger.info('本地存储', { keys, currentSize, limitSize })
  }
}

module.exports = remoteDebug
```

在 `app.js` 中初始化：

```js
const remoteDebug = require('./utils/remote-debug')

App({
  onLaunch() {
    remoteDebug.init()
    remoteDebug.dumpEnv()
  }
})
```

---

## 4. 性能测试

### 4.1 性能数据采集（wx.getPerformance）

`wx.getPerformance()` 提供小程序运行期的性能条目，通过 `createObserver` 订阅可以持续监控渲染与脚本耗时。

`utils/perf.js`：

```js
const SLOW_THRESHOLD = 100 // ms

function startPerfObserver() {
  if (!wx.getPerformance) return null

  const performance = wx.getPerformance()
  const observer = performance.createObserver((entryList) => {
    entryList.getEntries().forEach((entry) => {
      const { entryType, name, duration, startTime, path } = entry

      if (duration > SLOW_THRESHOLD) {
        console.warn(
          `[perf] 慢${entryType === 'render' ? '渲染' : '脚本'}: ${name} ` +
          `耗时 ${duration}ms`,
          { path, startTime }
        )
      }
    })
  })

  // render: 渲染耗时（首次渲染、setData 更新渲染）
  // script: 脚本注入与执行耗时
  observer.observe({ entryTypes: ['render', 'script'] })

  return observer
}

module.exports = { startPerfObserver }
```

在 `app.js` 中启动：

```js
const { startPerfObserver } = require('./utils/perf')

App({
  onLaunch() {
    this.perfObserver = startPerfObserver()
  },

  onHide() {
    // 不再需要时断开，避免持续回调开销
    this.perfObserver && this.perfObserver.disconnect()
  }
})
```

关键条目说明：

| entryType | name | 含义 |
| --- | --- | --- |
| `render` | `firstRender` | 页面首次渲染耗时，直接影响用户感知的打开速度 |
| `render` | `firstPaint` | 首次绘制时间点 |
| `script` | `evaluateScript` | 脚本注入耗时，与代码包体积正相关 |
| `navigation` | `route` | 路由跳转耗时 |
| `loadPackage` | 分包名 | 分包加载耗时 |

> `firstRender` 超过 500ms、单次 `render` 超过 100ms 通常意味着 setData 数据量过大或渲染节点过多，应优先排查 setData 载荷。

### 4.2 内存监控

```js
function checkMemory() {
  if (!wx.getPerformance) return null

  const memory = wx.getPerformance().memory
  if (!memory) return null // 部分平台不支持

  const toMB = (bytes) => +(bytes / 1024 / 1024).toFixed(2)
  const usage = {
    used: toMB(memory.usedJSHeapSize),
    total: toMB(memory.totalJSHeapSize),
    limit: toMB(memory.jsHeapSizeLimit)
  }
  usage.ratio = +(usage.used / usage.limit).toFixed(3)

  if (usage.ratio > 0.8) {
    console.warn('[perf] JS 堆内存占用过高', usage)
  }

  return usage
}
```

配合系统级的内存告警一起使用：

```js
App({
  onLaunch() {
    // level: 5(TRIM_MEMORY_RUNNING_MODERATE) / 10(LOW) / 15(CRITICAL)
    wx.onMemoryWarning((res) => {
      console.error('[perf] 收到内存告警', { level: res.level, ...checkMemory() })
      // 释放可重建的缓存：图片缓存、长列表数据、已离屏的 canvas 等
      releaseCaches()
    })
  }
})
```

内存排查思路：进入/退出页面若干次后对比 `usedJSHeapSize`，持续增长不回落通常意味着页面销毁时没有清理定时器、事件监听、`createSelectorQuery` 的持有引用或全局缓存。

---

## 5. 调试技巧

### 5.1 按环境开启 vConsole

真机上没有开发者工具控制台时，vConsole 是查看日志最直接的方式。用 `envVersion` 判断只在开发版开启，避免正式版给用户看到调试面板。

```js
App({
  onLaunch() {
    const { envVersion } = wx.getAccountInfoSync().miniProgram

    // develop=开发版, trial=体验版, release=正式版
    if (envVersion === 'develop') {
      wx.setEnableDebug({ enableDebug: true })
    } else if (envVersion === 'release') {
      wx.setEnableDebug({ enableDebug: false })
    }
  }
})
```

> 用户也可以手动开启：小程序右上角胶囊 → 关于 → 右上角菜单 → 开发调试。
> `wx.setEnableDebug` 会在下次冷启动后完全生效，调用后建议提示用户重启小程序。

### 5.2 网络请求监控

覆写 `wx.request` 统一记录耗时、状态码和失败原因，是定位接口问题最省事的做法。

`utils/request-monitor.js`：

```js
const logger = require('./logger')

const SLOW_REQUEST = 1000 // ms

function installRequestMonitor() {
  const originalRequest = wx.request

  Object.defineProperty(wx, 'request', {
    writable: true,
    value(options) {
      const start = Date.now()
      const { url, method = 'GET' } = options

      const wrap = (name, extra) => (res) => {
        const duration = Date.now() - start
        const record = { url, method, duration, ...extra(res) }

        if (name === 'fail' || res.statusCode >= 400) {
          logger.error('[request] 失败', record)
        } else if (duration > SLOW_REQUEST) {
          logger.warn('[request] 慢请求', record)
        } else {
          logger.info('[request] 完成', record)
        }

        options[name] && options[name](res)
      }

      return originalRequest.call(wx, {
        ...options,
        success: wrap('success', (res) => ({ statusCode: res.statusCode })),
        fail: wrap('fail', (err) => ({ errMsg: err.errMsg }))
      })
    }
  })
}

module.exports = { installRequestMonitor }
```

在 `app.js` 最顶部调用 `installRequestMonitor()`，确保在任何业务请求发起前完成覆写。

> 只在非正式版安装监控，或把日志级别下调，避免正式环境产生大量实时日志占用配额。

### 5.3 页面生命周期监控

包装全局 `Page` 构造器，统一记录生命周期与页面停留时长，可以快速发现「onLoad 里做了重活」「页面没有正确 onUnload」这类问题。

`utils/page-monitor.js`：

```js
const logger = require('./logger')

const LIFECYCLE = ['onLoad', 'onShow', 'onReady', 'onHide', 'onUnload']

function installPageMonitor() {
  const originalPage = Page

  Page = function (options) {
    const wrapped = { ...options }

    LIFECYCLE.forEach((hook) => {
      const original = options[hook]

      wrapped[hook] = function (...args) {
        const route = this.route || this.__route__
        const start = Date.now()

        if (hook === 'onLoad') {
          this.__enterTime = start
          logger.info(`[page] ${route} onLoad`, args[0] || {})
        } else if (hook === 'onUnload') {
          const stay = Date.now() - (this.__enterTime || start)
          logger.info(`[page] ${route} onUnload`, { stayMs: stay })
        } else {
          logger.info(`[page] ${route} ${hook}`)
        }

        const result = original && original.apply(this, args)

        const cost = Date.now() - start
        if (cost > 100) {
          logger.warn(`[page] ${route} ${hook} 耗时 ${cost}ms`)
        }

        return result
      }
    })

    return originalPage(wrapped)
  }
}

module.exports = { installPageMonitor }
```

在 `app.js` 中于所有页面注册前安装：

```js
const { installPageMonitor } = require('./utils/page-monitor')
const { installRequestMonitor } = require('./utils/request-monitor')

installPageMonitor()
installRequestMonitor()

App({
  onLaunch() { /* ... */ }
})
```

> 同类包装也适用于 `Component`（监控 `attached` / `detached`）。生产环境建议只保留 `onLoad` 与异常路径的日志，全量生命周期日志仅在开发版开启。

### 5.4 其他实用调试手段

| 手段 | 用途 |
| --- | --- |
| 开发者工具 → Wxml 面板 | 实时查看渲染树与节点样式，定位布局问题 |
| 开发者工具 → AppData 面板 | 直接编辑页面 data，快速验证不同数据下的渲染结果 |
| 开发者工具 → Sensor 面板 | 模拟地理位置与重力感应 |
| 开发者工具 → Network 面板 | 查看请求与 socket；注意云开发调用需在 Cloud 面板查看 |
| 开发者工具 → Storage 面板 | 查看与清理本地缓存，复现「缓存脏数据」类问题 |
| 自定义编译条件 | 直接以指定页面 + 启动参数启动，免去手动点击路径 |
| 代码依赖分析 | 查看各文件体积占比，定位主包超限元凶 |
| 体验评分（Audits） | 自动跑出性能、体验、最佳实践评分与具体改进项 |

---

## 参考

- 单元测试：[miniprogram-simulate](https://github.com/wechat-miniprogram/miniprogram-simulate)
- 自动化测试：[小程序自动化 SDK](https://developers.weixin.qq.com/miniprogram/dev/devtools/auto/)
- 实时日志：[wx.getRealtimeLogManager](https://developers.weixin.qq.com/miniprogram/dev/api/base/debug/wx.getRealtimeLogManager.html)
- 性能监控：[wx.getPerformance](https://developers.weixin.qq.com/miniprogram/dev/api/base/performance/wx.getPerformance.html)
- 真机调试：[真机调试 2.0](https://developers.weixin.qq.com/miniprogram/dev/devtools/remote-debug.html)
