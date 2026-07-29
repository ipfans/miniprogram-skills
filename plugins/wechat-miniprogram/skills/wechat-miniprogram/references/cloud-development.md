# 微信小程序云开发与网络集成

云开发（CloudBase）提供云函数、云数据库、云存储三大能力，免鉴权直接在小程序端调用。本文覆盖云开发全流程模式，以及不使用云开发时的网络请求封装、授权与登录流程。

## 云开发初始化

云开发需要**基础库 2.2.3+**，且必须在使用任何 `wx.cloud.*` 接口前完成初始化。初始化只需一次，放在 `App.onLaunch` 中。

```javascript
// app.js
App({
  globalData: {
    userInfo: null,
    token: null,
    openid: null
  },

  onLaunch() {
    this.initCloud()
  },

  initCloud() {
    if (!wx.cloud) {
      console.error('云开发需要基础库 2.2.3 或以上版本')
      wx.showModal({
        title: '版本过低',
        content: '请更新微信客户端后重试',
        showCancel: false
      })
      return
    }

    wx.cloud.init({
      env: 'prod-env-id',   // 环境 ID，不要用 DYNAMIC_CURRENT_ENV 之外的魔法值
      traceUser: true       // 将用户访问记录到「用户管理」，便于运营查看
    })

    this.cloudReady = true
  }
})
```

### 多环境切换

开发/生产环境隔离，避免测试数据污染线上库：

```javascript
const ENV_MAP = {
  develop: 'dev-env-id',      // 开发版
  trial: 'dev-env-id',        // 体验版
  release: 'prod-env-id'      // 正式版
}

initCloud() {
  const { envVersion } = wx.getAccountInfoSync().miniProgram
  wx.cloud.init({
    env: ENV_MAP[envVersion] || ENV_MAP.release,
    traceUser: true
  })
}
```

| 配置项 | 说明 |
|--------|------|
| `env` | 环境 ID。传 `wx.cloud.DYNAMIC_CURRENT_ENV` 时云函数内使用当前所在环境 |
| `traceUser` | 是否记录用户访问，默认 `false`。开启后可在控制台「用户管理」查看 |

> 注意：`wx.cloud.init()` 是同步的，但不会校验 `env` 是否存在。环境 ID 写错只会在实际调用时报 `env not exists`。

## 云函数调用

`wx.cloud.callFunction` 自带用户身份（`event.userInfo.openId`），无需自行传 openid，也无需配置域名白名单。

### 统一封装

```javascript
// utils/cloud.js

/**
 * 调用云函数，自动处理 loading 与错误提示
 * @param {string} name 云函数名
 * @param {object} data 参数
 * @param {object} options { loading, loadingText, showError }
 */
function callFunction(name, data = {}, options = {}) {
  const {
    loading = true,
    loadingText = '加载中',
    showError = true
  } = options

  if (loading) {
    wx.showLoading({ title: loadingText, mask: true })
  }

  return wx.cloud
    .callFunction({ name, data })
    .then(res => {
      const result = res.result || {}
      // 约定云函数返回 { code, message, data }
      if (result.code !== 0) {
        const err = new Error(result.message || '请求失败')
        err.code = result.code
        throw err
      }
      return result.data
    })
    .catch(err => {
      console.error(`[cloud] ${name} 调用失败`, err)
      if (showError) {
        wx.showToast({
          title: err.message || '网络异常，请稍后重试',
          icon: 'none',
          duration: 2000
        })
      }
      throw err
    })
    .finally(() => {
      if (loading) wx.hideLoading()
    })
}

module.exports = { callFunction }
```

### 使用示例

```javascript
const { callFunction } = require('../../utils/cloud')

Page({
  async onLoad() {
    try {
      const user = await callFunction('getUser', { withProfile: true })
      this.setData({ user })
    } catch (err) {
      // 已统一提示，此处只做页面级降级
      this.setData({ loadFailed: true })
    }
  },

  async handleSubmit(e) {
    const order = await callFunction(
      'createOrder',
      { skuId: e.detail.skuId, count: 1 },
      { loadingText: '下单中' }
    )
    wx.navigateTo({ url: `/pages/order/detail?id=${order._id}` })
  }
})
```

### 云函数侧写法

```javascript
// cloudfunctions/getUser/index.js
const cloud = require('wx-server-sdk')
cloud.init({ env: cloud.DYNAMIC_CURRENT_ENV })

const db = cloud.database()

exports.main = async (event, context) => {
  // openid 由平台注入，前端伪造不了
  const { OPENID } = cloud.getWXContext()

  try {
    const { data } = await db.collection('users').where({ _openid: OPENID }).get()
    return { code: 0, data: data[0] || null }
  } catch (err) {
    console.error(err)
    return { code: 500, message: '查询失败' }
  }
}
```

> 云函数默认超时 3 秒、内存 256MB，可在 `config.json` 中调整。长耗时任务（导出、批量处理）应拆分或改用云托管。

## 云数据库操作

```javascript
const db = wx.cloud.database()
const _ = db.command    // 查询/更新操作符
```

### 查询

```javascript
// 条件查询 + 排序 + 字段裁剪 + 分页
const { data } = await db.collection('orders')
  .where({
    status: _.neq('cancelled'),       // 不等于
    amount: _.gte(100),               // 大于等于
    createdAt: _.gt(new Date('2026/01/01'))
  })
  .orderBy('createdAt', 'desc')
  .field({ _id: true, title: true, amount: true, status: true })  // 只取需要的字段
  .skip(page * 20)
  .limit(20)
  .get()
```

常用 `db.command` 操作符：

| 操作符 | 含义 | 示例 |
|--------|------|------|
| `_.eq` / `_.neq` | 等于 / 不等于 | `_.neq('cancelled')` |
| `_.gt` / `_.gte` | 大于 / 大于等于 | `_.gte(100)` |
| `_.lt` / `_.lte` | 小于 / 小于等于 | `_.lt(50)` |
| `_.in` / `_.nin` | 在 / 不在集合中 | `_.in(['paid', 'shipped'])` |
| `_.and` / `_.or` | 逻辑与 / 或 | `_.or([{ a: 1 }, { b: 2 }])` |
| `_.exists` | 字段是否存在 | `_.exists(true)` |
| `_.inc` / `_.mul` | 自增 / 自乘（更新用） | `_.inc(1)` |
| `_.push` / `_.pull` | 数组追加 / 移除（更新用） | `_.push('tag')` |

**分页与条数限制**：小程序端单次 `get()` 最多返回 **20 条**，云函数端最多 **100 条**（`limit` 上限 1000 但受返回体积限制）。大批量数据必须在云函数中分批拉取。

```javascript
// 统计总数用 count()，不要拉全量再取 length
const { total } = await db.collection('orders').where({ status: 'paid' }).count()
```

### 新增

```javascript
const res = await db.collection('orders').add({
  data: {
    title: '订单标题',
    amount: 199,
    status: 'pending',
    // 服务端时间，避免客户端时间被篡改或时区错乱
    createdAt: db.serverDate(),
    // 延后 1 小时的时间戳
    expireAt: db.serverDate({ offset: 60 * 60 * 1000 })
  }
})

console.log(res._id)   // 新记录 ID
```

> `_openid` 由数据库**自动写入**当前用户的 openid，不需要也不应该手动传。手动传入的 `_openid` 在小程序端会被忽略，只有云函数（管理端权限）才能指定。

### 更新

```javascript
// 更新单条：doc(id).update 只改传入字段，其余字段保留
await db.collection('orders').doc(orderId).update({
  data: {
    status: 'paid',
    paidAt: db.serverDate(),
    viewCount: _.inc(1)          // 原子自增，避免并发覆盖
  }
})

// set 会整条覆盖（未传字段被删除），谨慎使用
await db.collection('orders').doc(orderId).set({
  data: { title: '新标题', status: 'pending' }
})

// 批量更新只能在云函数中执行
// await db.collection('orders').where({ status: 'pending' }).update({ data: { status: 'expired' } })
```

### 删除

```javascript
// 删除单条
await db.collection('orders').doc(orderId).remove()

// 批量删除同样仅限云函数
// await db.collection('logs').where({ createdAt: _.lt(threshold) }).remove()
```

实践中很少物理删除，推荐软删除以便追溯：

```javascript
await db.collection('orders').doc(orderId).update({
  data: { deleted: true, deletedAt: db.serverDate() }
})
```

### 实时监听

`watch()` 建立长连接，数据变更时推送。适合聊天、订单状态、协同编辑等场景。

```javascript
Page({
  onLoad() {
    this.watcher = db.collection('messages')
      .where({ roomId: this.options.roomId })
      .orderBy('createdAt', 'desc')
      .limit(20)
      .watch({
        onChange: snapshot => {
          // snapshot.type === 'init' 为首次全量快照
          // snapshot.docChanges 为增量变更列表
          this.setData({ messages: snapshot.docs })
        },
        onError: err => {
          console.error('监听失败', err)
          this.setData({ realtimeBroken: true })
        }
      })
  },

  onUnload() {
    // 必须关闭，否则连接泄漏且持续计费
    this.watcher && this.watcher.close()
  }
})
```

**限制**：单个小程序客户端最多 5 个监听器，监听结果集上限 5000 条，`where` 条件不支持部分复杂操作符。切页面务必在 `onUnload`/`detached` 中 `close()`。

## 云数据库安全规范

小程序端直连数据库很方便，但**权限校验完全依赖安全规则**。规则配错等于数据库裸奔。

### 安全规则配置

在云开发控制台为每个集合配置（「数据库 → 权限设置 → 自定义安全规则」）：

```json
{
  "read": "doc._openid == auth.openid",
  "write": "doc._openid == auth.openid"
}
```

常见规则模式：

```json
// 所有人可读，仅创建者可写（商品、文章）
{
  "read": true,
  "write": "doc._openid == auth.openid"
}

// 仅登录用户可读自己的数据（订单、地址）
{
  "read": "doc._openid == auth.openid",
  "write": "doc._openid == auth.openid"
}

// 前端完全禁止访问，只能通过云函数（支付流水、优惠券库存、用户余额）
{
  "read": false,
  "write": false
}

// 创建时强制写入者身份，且禁止改动关键字段
{
  "read": "doc._openid == auth.openid",
  "write": "doc._openid == auth.openid && request.auth.openid != null",
  "create": "request.resource.data._openid == auth.openid"
}
```

### 三条原则

**1. 敏感查询放云函数。** 涉及金额、库存、他人数据、聚合统计的读写，一律走云函数，集合权限设为 `false`。

**2. 用 `_openid` 隔离用户数据。** 每条用户数据都带 `_openid`，安全规则以此为唯一凭据。`_openid` 由平台注入，前端无法伪造。

**3. 不要依赖前端 `.where()` 做安全边界。** `.where()` 是查询条件，不是权限控制。攻击者可以直接改包、改代码去掉条件；真正拦截的只有安全规则和云函数。

### 错误 vs 正确

```javascript
// ❌ 错误：前端直查敏感集合，靠 where 限制范围
// 集合权限若是「所有用户可读」，去掉 where 就能拉到全部用户的余额
const { data } = await db.collection('user_wallet')
  .where({ _openid: app.globalData.openid })   // openid 来自前端，可被替换
  .get()

// ❌ 错误：前端直接扣款，金额由前端决定
await db.collection('user_wallet').doc(walletId).update({
  data: { balance: _.inc(-amount) }
})
```

```javascript
// ✅ 正确：集合权限设为 { "read": false, "write": false }，走云函数
const wallet = await callFunction('getWallet')

// ✅ 正确：扣款逻辑与校验全在云函数内完成
const result = await callFunction('payOrder', { orderId })
```

```javascript
// cloudfunctions/payOrder/index.js
const cloud = require('wx-server-sdk')
cloud.init({ env: cloud.DYNAMIC_CURRENT_ENV })
const db = cloud.database()
const _ = db.command

exports.main = async (event) => {
  const { OPENID } = cloud.getWXContext()   // 可信身份，前端伪造不了
  const { orderId } = event

  // 金额从数据库取，绝不信任前端传入的 amount
  const { data: order } = await db.collection('orders').doc(orderId).get()
  if (!order || order._openid !== OPENID) {
    return { code: 403, message: '无权操作该订单' }
  }
  if (order.status !== 'pending') {
    return { code: 400, message: '订单状态异常' }
  }

  // 条件更新保证余额充足，避免并发超扣
  const res = await db.collection('user_wallet')
    .where({ _openid: OPENID, balance: _.gte(order.amount) })
    .update({ data: { balance: _.inc(-order.amount) } })

  if (res.stats.updated === 0) {
    return { code: 400, message: '余额不足' }
  }

  await db.collection('orders').doc(orderId).update({
    data: { status: 'paid', paidAt: db.serverDate() }
  })

  return { code: 0, data: { orderId } }
}
```

## 云存储

云存储用 `fileID`（形如 `cloud://env.xxx/path/file.png`）标识文件。`image`、`video` 等组件的 `src` 可直接填 `fileID`，会自动解析。

### 上传

```javascript
// utils/storage.js

function genCloudPath(filePath, dir = 'images') {
  const ext = filePath.match(/\.(\w+)$/)?.[1] || 'png'
  const unique = `${Date.now()}-${Math.random().toString(36).slice(2, 10)}`
  return `${dir}/${unique}.${ext}`
}

function uploadFile(filePath, dir = 'images', onProgress) {
  const task = wx.cloud.uploadFile({
    cloudPath: genCloudPath(filePath, dir),
    filePath
  })

  if (onProgress) {
    task.onProgressUpdate(res => onProgress(res.progress))
  }

  return task.then(res => res.fileID)
}

module.exports = { uploadFile, genCloudPath }
```

> 文件名必须唯一。同名 `cloudPath` 会**直接覆盖**已有文件——用时间戳 + 随机串生成，不要用原始文件名。

### 下载

```javascript
const { tempFilePath } = await wx.cloud.downloadFile({ fileID })

// 下载后可保存到相册（需 scope.writePhotosAlbum 授权）
await wx.saveImageToPhotosAlbum({ filePath: tempFilePath })
```

### 删除

```javascript
const res = await wx.cloud.deleteFile({
  fileList: [fileID1, fileID2]    // 单次最多 50 个
})

res.fileList.forEach(item => {
  if (item.status !== 0) console.warn('删除失败', item.fileID, item.errMsg)
})
```

小程序端删除受存储安全规则限制，通常只允许删自己上传的文件；批量清理应在云函数中执行。

### 获取临时链接

`fileID` 无法在小程序外使用（如分享到外部、传给第三方接口），需换成 HTTPS 临时链接。

```javascript
const res = await wx.cloud.getTempFileURL({
  fileList: [
    { fileID, maxAge: 60 * 60 }   // 有效期秒数，默认 2 小时，最长 24 小时
  ]
})

const url = res.fileList[0].tempFileURL
```

单次最多 50 个 `fileID`。链接会过期，**不要把 tempFileURL 存进数据库**——数据库里存 `fileID`，展示时再换取。

### 完整示例：选图 → 上传 → 入库

```javascript
const { uploadFile } = require('../../utils/storage')

Page({
  data: { images: [], uploading: false },

  async chooseAndUploadImage() {
    if (this.data.uploading) return

    // 1. 选择图片（wx.chooseMedia 需基础库 2.10.0+，替代已废弃的 wx.chooseImage）
    const { tempFiles } = await wx.chooseMedia({
      count: 9 - this.data.images.length,
      mediaType: ['image'],
      sourceType: ['album', 'camera'],
      sizeType: ['compressed']       // 压缩图，显著减小体积
    })

    this.setData({ uploading: true })
    wx.showLoading({ title: '上传中 0%', mask: true })

    try {
      // 2. 并发上传
      const fileIDs = await Promise.all(
        tempFiles.map(file =>
          uploadFile(file.tempFilePath, 'posts', progress => {
            wx.showLoading({ title: `上传中 ${progress}%`, mask: true })
          })
        )
      )

      // 3. 写入数据库
      const db = wx.cloud.database()
      await db.collection('posts').add({
        data: {
          images: fileIDs,
          createdAt: db.serverDate()
        }
      })

      this.setData({ images: this.data.images.concat(fileIDs) })
      wx.showToast({ title: '上传成功' })
    } catch (err) {
      console.error('上传失败', err)
      wx.showToast({ title: '上传失败，请重试', icon: 'none' })
    } finally {
      wx.hideLoading()
      this.setData({ uploading: false })
    }
  },

  // 删除已上传的图片，同时清理云存储
  async removeImage(e) {
    const { index } = e.currentTarget.dataset
    const fileID = this.data.images[index]

    const images = this.data.images.filter((_, i) => i !== index)
    this.setData({ images })

    wx.cloud.deleteFile({ fileList: [fileID] }).catch(err => {
      console.warn('云存储清理失败，将由定时任务回收', err)
    })
  }
})
```

WXML 中直接用 `fileID` 渲染：

```xml
<view class="image-grid">
  <image
    wx:for="{{images}}"
    wx:key="*this"
    src="{{item}}"
    mode="aspectFill"
    bindtap="previewImage"
    data-index="{{index}}"
  />
</view>
```

## 网络请求封装

不使用云开发时，`wx.request` 需要自行封装：域名必须在「开发设置 → 服务器域名」中配置 HTTPS 白名单（每月最多改 5 次）。

```javascript
// utils/request.js

const BASE_URL = 'https://api.example.com'
const TIMEOUT = 10000

class Request {
  constructor(options = {}) {
    this.baseURL = options.baseURL || BASE_URL
    this.timeout = options.timeout || TIMEOUT
    this.isRedirecting = false    // 防止 401 并发时重复跳转
  }

  getHeader(custom = {}) {
    const header = {
      'content-type': 'application/json',
      ...custom
    }
    const token = wx.getStorageSync('token')
    if (token) {
      header.Authorization = `Bearer ${token}`
    }
    return header
  }

  handleUnauthorized() {
    if (this.isRedirecting) return
    this.isRedirecting = true

    wx.removeStorageSync('token')
    wx.removeStorageSync('userInfo')

    wx.showToast({ title: '登录已过期，请重新登录', icon: 'none' })
    setTimeout(() => {
      wx.navigateTo({
        url: '/pages/login/login',
        complete: () => { this.isRedirecting = false }
      })
    }, 1500)
  }

  request(options) {
    const { url, method = 'GET', data, header, showError = true } = options

    return new Promise((resolve, reject) => {
      wx.request({
        url: /^https?:\/\//.test(url) ? url : this.baseURL + url,
        method,
        data,
        header: this.getHeader(header),
        timeout: this.timeout,

        success: res => {
          const { statusCode, data: body } = res

          if (statusCode === 200) {
            resolve(body)
            return
          }

          if (statusCode === 401) {
            this.handleUnauthorized()
            reject(new Error('未授权'))
            return
          }

          const message = {
            403: '没有权限访问',
            404: '请求的资源不存在',
            500: '服务器错误，请稍后重试'
          }[statusCode] || `请求失败（${statusCode}）`

          if (showError) wx.showToast({ title: message, icon: 'none' })
          const err = new Error(message)
          err.statusCode = statusCode
          reject(err)
        },

        fail: err => {
          const message = err.errMsg.includes('timeout')
            ? '请求超时，请检查网络'
            : '网络异常，请稍后重试'
          if (showError) wx.showToast({ title: message, icon: 'none' })
          reject(new Error(message))
        }
      })
    })
  }

  get(url, data, options = {}) {
    return this.request({ url, method: 'GET', data, ...options })
  }

  post(url, data, options = {}) {
    return this.request({ url, method: 'POST', data, ...options })
  }

  put(url, data, options = {}) {
    return this.request({ url, method: 'PUT', data, ...options })
  }

  delete(url, data, options = {}) {
    return this.request({ url, method: 'DELETE', data, ...options })
  }
}

module.exports = new Request()
```

### 使用示例

```javascript
// api/user.js
const request = require('../utils/request')

module.exports = {
  getProfile: () => request.get('/user/profile'),
  updateProfile: (data) => request.put('/user/profile', data),
  getOrders: (page) => request.get('/orders', { page, size: 20 })
}
```

```javascript
// pages/profile/profile.js
const userApi = require('../../api/user')

Page({
  async onLoad() {
    try {
      const profile = await userApi.getProfile()
      this.setData({ profile })
    } catch (err) {
      this.setData({ error: true })
    }
  }
})
```

> `wx.request` 最大并发数为 **10**，超出会排队。批量请求建议分批或合并接口。

## 用户授权处理

`wx.authorize` 只在用户**首次**被询问时弹窗；用户拒绝后再次调用会直接失败，必须引导到 `wx.openSetting` 手动打开。

```javascript
// utils/permission.js

const SCOPE_NAMES = {
  'scope.userLocation': '地理位置',
  'scope.userFuzzyLocation': '模糊地理位置',
  'scope.writePhotosAlbum': '保存到相册',
  'scope.camera': '摄像头',
  'scope.record': '麦克风',
  'scope.werun': '微信运动步数',
  'scope.address': '通讯地址',
  'scope.invoiceTitle': '发票抬头'
}

class Permission {
  /**
   * 检查并申请授权
   * @param {string} scope 如 'scope.userLocation'
   * @returns {Promise<boolean>} 是否已获授权
   */
  async check(scope) {
    const name = SCOPE_NAMES[scope] || '相关'

    const { authSetting } = await wx.getSetting()

    // 已授权
    if (authSetting[scope] === true) return true

    // 从未询问过：直接申请，此时会弹出系统授权框
    if (authSetting[scope] === undefined) {
      try {
        await wx.authorize({ scope })
        return true
      } catch (err) {
        return this.guideToSetting(scope, name)
      }
    }

    // 曾被拒绝（authSetting[scope] === false）：只能去设置页
    return this.guideToSetting(scope, name)
  }

  async guideToSetting(scope, name) {
    const { confirm } = await wx.showModal({
      title: '需要授权',
      content: `请在设置中开启「${name}」权限，以便正常使用该功能`,
      confirmText: '去设置'
    })

    if (!confirm) return false

    const { authSetting } = await wx.openSetting()
    return authSetting[scope] === true
  }
}

module.exports = new Permission()
```

### 使用示例

```javascript
const permission = require('../../utils/permission')

Page({
  async getLocation() {
    const granted = await permission.check('scope.userLocation')
    if (!granted) {
      wx.showToast({ title: '未授权定位，已使用默认城市', icon: 'none' })
      this.setData({ city: '北京' })
      return
    }

    const { latitude, longitude } = await wx.getLocation({ type: 'gcj02' })
    this.setData({ latitude, longitude })
    this.loadNearbyShops(latitude, longitude)
  }
})
```

> 使用 `scope.userLocation` 必须在 `app.json` 的 `requiredPrivateInfos` 中声明 `getLocation`，并在隐私保护指引中说明用途，否则接口直接调用失败。

## 登录流程

### 完整登录

`wx.login` 拿到的 `code` 有效期 5 分钟且只能用一次，必须由后端（或云函数）换取 `openid` / `session_key`——**绝不能把 AppSecret 放在小程序端**。

```javascript
// utils/auth.js
const request = require('./request')

/**
 * 静默登录：换取业务 token
 */
async function login() {
  const { code } = await wx.login()

  // 后端用 code 调 code2Session，返回业务 token
  const { token, userInfo, expiresIn } = await request.post('/auth/login', { code })

  wx.setStorageSync('token', token)
  wx.setStorageSync('tokenExpireAt', Date.now() + expiresIn * 1000)
  if (userInfo) wx.setStorageSync('userInfo', userInfo)

  const app = getApp()
  app.globalData.token = token
  app.globalData.userInfo = userInfo

  return { token, userInfo }
}

/**
 * 检查登录态：本地 token 未过期 + 微信会话未失效 + 服务端校验通过
 */
async function checkLoginStatus() {
  const token = wx.getStorageSync('token')
  const expireAt = wx.getStorageSync('tokenExpireAt')

  if (!token || (expireAt && Date.now() > expireAt)) {
    return false
  }

  // 微信侧 session_key 可能已失效
  try {
    await wx.checkSession()
  } catch (err) {
    return false
  }

  // 服务端二次校验，防止 token 被后台吊销
  try {
    const { valid, userInfo } = await request.get(
      '/auth/verify',
      {},
      { showError: false }
    )
    if (valid) {
      getApp().globalData.userInfo = userInfo
      return true
    }
  } catch (err) {
    return false
  }

  return false
}

/**
 * 确保已登录，未登录则自动静默登录
 */
async function ensureLogin() {
  if (await checkLoginStatus()) return true
  try {
    await login()
    return true
  } catch (err) {
    console.error('登录失败', err)
    return false
  }
}

function logout() {
  wx.removeStorageSync('token')
  wx.removeStorageSync('tokenExpireAt')
  wx.removeStorageSync('userInfo')
  const app = getApp()
  app.globalData.token = null
  app.globalData.userInfo = null
}

module.exports = { login, checkLoginStatus, ensureLogin, logout }
```

在 `app.js` 中启动时执行：

```javascript
// app.js
const auth = require('./utils/auth')

App({
  globalData: { userInfo: null, token: null },

  async onLaunch() {
    this.initCloud()
    // 用 Promise 暴露登录态，页面可 await
    this.loginReady = auth.ensureLogin()
  }
})
```

```javascript
// 页面中等待登录完成
Page({
  async onLoad() {
    const ok = await getApp().loginReady
    if (!ok) {
      wx.navigateTo({ url: '/pages/login/login' })
      return
    }
    this.loadData()
  }
})
```

### 获取用户昵称头像

`wx.getUserProfile` **必须由用户点击按钮触发**（不能在 `onLoad` 等生命周期中调用），且 `desc` 参数必填。

```xml
<button bindtap="handleGetUserProfile">授权获取头像昵称</button>
```

```javascript
Page({
  async handleGetUserProfile() {
    // 必须在点击事件回调中同步调用，异步链路中调用会失败
    try {
      const { userInfo } = await wx.getUserProfile({
        desc: '用于完善会员资料'     // 必填，会展示在授权弹窗中
      })

      this.setData({ userInfo })
      wx.setStorageSync('userInfo', userInfo)
      await request.post('/user/profile', userInfo)
    } catch (err) {
      // 用户点了「拒绝」
      wx.showToast({ title: '已取消授权', icon: 'none' })
    }
  }
})
```

> **重要变更**：自基础库 2.27.1 起，`wx.getUserProfile` 与 `wx.getUserInfo` 返回的是**匿名数据**（灰色头像 + 昵称「微信用户」）。新项目应改用**头像昵称填写能力**：

```xml
<button open-type="chooseAvatar" bindchooseavatar="onChooseAvatar">
  <image src="{{avatarUrl}}" />
</button>
<input type="nickname" placeholder="请输入昵称" bindblur="onNicknameBlur" />
```

```javascript
Page({
  async onChooseAvatar(e) {
    const { avatarUrl } = e.detail    // 本地临时路径，需自行上传
    const fileID = await uploadFile(avatarUrl, 'avatars')
    this.setData({ avatarUrl: fileID })
  },

  onNicknameBlur(e) {
    this.setData({ nickname: e.detail.value })
  }
})
```

### 手机号获取

手机号需通过 `open-type="getPhoneNumber"` 按钮获取，且**必须是已认证的非个人主体小程序**：

```xml
<button open-type="getPhoneNumber" bindgetphonenumber="onGetPhoneNumber">
  微信手机号快捷登录
</button>
```

```javascript
Page({
  async onGetPhoneNumber(e) {
    if (e.detail.errMsg !== 'getPhoneNumber:ok') {
      wx.showToast({ title: '已取消', icon: 'none' })
      return
    }
    // code 交给后端换取手机号（新版方式，基础库 2.21.2+）
    const { phone } = await request.post('/auth/phone', { code: e.detail.code })
    this.setData({ phone })
  }
})
```

## 检查清单

| 项 | 要求 |
|----|------|
| 云开发初始化 | `App.onLaunch` 中执行，先判断 `wx.cloud` 是否存在 |
| 环境隔离 | 按 `envVersion` 切换开发/生产环境 ID |
| 云函数返回 | 统一 `{ code, message, data }` 结构 |
| 敏感集合权限 | 设为 `{ "read": false, "write": false }`，只走云函数 |
| 用户数据隔离 | 安全规则用 `doc._openid == auth.openid` |
| 金额/库存 | 一律在云函数校验，不信任前端传参 |
| 时间字段 | 用 `db.serverDate()`，不用 `new Date()` |
| 文件名 | 时间戳 + 随机串，避免覆盖 |
| `tempFileURL` | 不入库，展示时现取 |
| `watch()` | `onUnload` 中必须 `close()` |
| `wx.getUserProfile` | 仅按钮点击触发，`desc` 必填；新项目改用头像昵称填写能力 |
| 请求域名 | HTTPS + ICP 备案，白名单每月限改 5 次 |

相关文档：`api-reference.md`（wx.* 接口速查）、`error-handling.md`（错误码对照）、`deployment-guide.md`（域名与合规配置）。
