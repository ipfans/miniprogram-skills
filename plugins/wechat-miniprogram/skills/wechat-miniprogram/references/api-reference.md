# 微信小程序 API 速查手册

按分类整理的常用 API 用法速查。示例只保留关键参数，完整参数请查阅官方文档。

## 通用约定

- 绝大多数 `wx.xxx` 异步 API 都接受 `success` / `fail` / `complete` 回调。
- 基础库 2.10.2 起，多数异步 API 在**不传 success/fail/complete** 时返回 Promise。
- 判断 API 是否可用：`wx.canIUse('getUserProfile')`、`wx.canIUse('button.open-type.chooseAvatar')`。

```js
// 回调风格
wx.request({ url, success(res) {}, fail(err) {} })

// Promise 风格（不传回调即可）
const res = await wx.request({ url })
```

---

## 1. 基础 API

### 系统信息

```js
// 同步获取系统信息（已废弃但仍广泛使用，推荐拆分成下面几个新 API）
const info = wx.getSystemInfoSync()
// info.windowWidth / windowHeight  可用窗口尺寸（px）
// info.screenWidth / screenHeight  屏幕尺寸
// info.statusBarHeight             状态栏高度，做自定义导航栏必用
// info.safeArea                    安全区域 { top, bottom, left, right, width, height }
// info.pixelRatio                  设备像素比
// info.platform                    'ios' | 'android' | 'devtools'
// info.SDKVersion                  基础库版本

// 推荐的新版拆分 API（性能更好，2.20.1+）
const { windowWidth, windowHeight, statusBarHeight, safeArea } = wx.getWindowInfo()
const { platform, system, brand, model } = wx.getDeviceInfo()
const { SDKVersion, version, language } = wx.getAppBaseInfo()
const { bluetoothEnabled, locationEnabled } = wx.getSystemSetting()

// 计算自定义导航栏高度（常用套路）
const { statusBarHeight } = wx.getWindowInfo()
const menu = wx.getMenuButtonBoundingClientRect()  // 胶囊按钮位置
const navBarHeight = (menu.top - statusBarHeight) * 2 + menu.height
```

### 网络状态

```js
// 获取当前网络类型
wx.getNetworkType({
  success(res) {
    // res.networkType: 'wifi' | '2g' | '3g' | '4g' | '5g' | 'unknown' | 'none'
    console.log(res.networkType)
  }
})

// 监听网络状态变化
wx.onNetworkStatusChange((res) => {
  // res.isConnected 是否有网络连接
  // res.networkType  当前网络类型
  if (!res.isConnected) {
    wx.showToast({ title: '网络已断开', icon: 'none' })
  }
})

// 移除监听（页面卸载时调用，避免内存泄漏）
wx.offNetworkStatusChange(handler)
```

### 账号与启动参数

```js
// 获取小程序账号信息
const account = wx.getAccountInfoSync()
// account.miniProgram.appId    小程序 appId
// account.miniProgram.envVersion 'develop' | 'trial' | 'release'
// account.miniProgram.version  线上版本号（仅正式版有值）
// account.plugin.version       插件版本号（插件中调用时）

// 环境变量
wx.env.USER_DATA_PATH  // 本地文件系统用户目录

// 启动参数
const launch = wx.getLaunchOptionsSync()  // 冷启动参数
const enter = wx.getEnterOptionsSync()    // 本次启动参数（含热启动）
// path / query / scene / referrerInfo
```

---

## 2. 用户相关

### 登录

```js
// 获取登录凭证 code，5 分钟有效且只能用一次
wx.login({
  success(res) {
    // res.code 送到自己的服务端，换取 openid / session_key
    wx.request({
      url: 'https://api.example.com/login',
      method: 'POST',
      data: { code: res.code },
      success({ data }) {
        wx.setStorageSync('token', data.token)
      }
    })
  }
})

// 检查登录态是否过期
wx.checkSession({
  success() { /* session_key 未过期 */ },
  fail() { wx.login({ /* 重新登录 */ }) }
})
```

### 用户信息

```js
// 获取用户信息（必须由用户点击等手势触发，不能在生命周期里直接调用）
wx.getUserProfile({
  desc: '用于完善会员资料',  // 必填，声明用途，展示在授权弹窗上
  success(res) {
    // res.userInfo: { nickName, avatarUrl, gender, country, province, city }
    this.setData({ userInfo: res.userInfo })
  }
})
```

> 注意：2022-10-25 起 `wx.getUserProfile` 返回的昵称为「微信用户」、头像为灰色默认图。
> 新方案是用**头像昵称填写能力**：

```html
<!-- 头像 -->
<button open-type="chooseAvatar" bind:chooseavatar="onChooseAvatar">
  <image src="{{avatarUrl}}" />
</button>
<!-- 昵称 -->
<input type="nickname" placeholder="请输入昵称" bindchange="onNickChange" />
```

```js
onChooseAvatar(e) {
  const { avatarUrl } = e.detail  // 临时文件路径，需自行上传到服务端
  this.setData({ avatarUrl })
}
```

### 授权与设置

```js
// 查询用户已授权的权限
wx.getSetting({
  success(res) {
    // res.authSetting: { 'scope.userLocation': true, 'scope.writePhotosAlbum': false }
    if (!res.authSetting['scope.userLocation']) {
      // 未授权，引导用户开启
    }
  }
})

// 主动发起授权（只能对未授权的 scope 调用；用户拒绝过则直接 fail）
wx.authorize({
  scope: 'scope.userLocation',
  success() { /* 已授权 */ },
  fail() {
    // 用户之前拒绝过，只能引导去设置页
    wx.showModal({
      title: '需要位置权限',
      content: '请在设置中开启位置权限',
      success(res) {
        if (res.confirm) wx.openSetting()
      }
    })
  }
})

// 打开设置页，让用户手动开关权限（必须由点击触发）
wx.openSetting({
  success(res) {
    console.log(res.authSetting)
  }
})
```

常用 scope：`scope.userLocation`（位置）、`scope.record`（录音）、`scope.camera`（摄像头）、
`scope.writePhotosAlbum`（保存到相册）、`scope.address`（通讯地址）、`scope.invoiceTitle`（发票抬头）。

---

## 3. 网络请求

### HTTP 请求

```js
wx.request({
  url: 'https://api.example.com/users',   // 必须是 https 且在域名白名单内
  method: 'GET',                          // GET/POST/PUT/DELETE/...
  data: { page: 1, size: 20 },
  header: { 'content-type': 'application/json', Authorization: 'Bearer xxx' },
  timeout: 10000,
  success(res) {
    // res.statusCode / res.data / res.header
    console.log(res.data)
  },
  fail(err) { console.error(err.errMsg) },
  complete() { wx.hideLoading() }
})

// 中断请求
const task = wx.request({ url: '...' })
task.abort()
```

### 封装成 Promise（推荐做法）

```js
// utils/request.js
const BASE_URL = 'https://api.example.com'

export function request(options) {
  return new Promise((resolve, reject) => {
    wx.request({
      ...options,
      url: BASE_URL + options.url,
      header: {
        'content-type': 'application/json',
        Authorization: `Bearer ${wx.getStorageSync('token') || ''}`,
        ...options.header
      },
      success(res) {
        if (res.statusCode >= 200 && res.statusCode < 300) {
          resolve(res.data)
        } else if (res.statusCode === 401) {
          wx.redirectTo({ url: '/pages/login/login' })
          reject(res)
        } else {
          wx.showToast({ title: res.data?.message || '请求失败', icon: 'none' })
          reject(res)
        }
      },
      fail: reject
    })
  })
}

export const get = (url, data) => request({ url, method: 'GET', data })
export const post = (url, data) => request({ url, method: 'POST', data })
```

```js
// 页面中使用
import { get, post } from '../../utils/request'

async onLoad() {
  try {
    const list = await get('/users', { page: 1 })
    this.setData({ list })
  } catch (e) {
    console.error(e)
  }
}
```

### 文件上传

```js
const task = wx.uploadFile({
  url: 'https://api.example.com/upload',
  filePath: tempFilePath,      // 本地文件临时路径
  name: 'file',                // 服务端接收的字段名
  formData: { userId: '123' }, // 额外的表单字段
  header: { Authorization: 'Bearer xxx' },
  success(res) {
    // 注意：res.data 是字符串，需要自己 JSON.parse
    const data = JSON.parse(res.data)
  }
})

// 上传进度
task.onProgressUpdate((res) => {
  console.log(res.progress)        // 百分比 0-100
  console.log(res.totalBytesSent)  // 已上传字节数
})
task.abort()  // 取消上传
```

### 文件下载

```js
wx.downloadFile({
  url: 'https://example.com/file.pdf',
  success(res) {
    if (res.statusCode === 200) {
      // res.tempFilePath 临时路径，小程序重启后可能失效
      wx.saveFile({
        tempFilePath: res.tempFilePath,
        success({ savedFilePath }) { console.log(savedFilePath) }
      })
      // 或直接打开文档
      wx.openDocument({ filePath: res.tempFilePath, fileType: 'pdf' })
    }
  }
})
```

### WebSocket

```js
const socket = wx.connectSocket({
  url: 'wss://example.com/ws',
  header: { 'content-type': 'application/json' },
  protocols: ['protocol1']
})

socket.onOpen(() => {
  socket.send({ data: JSON.stringify({ type: 'ping' }) })
})
socket.onMessage((res) => {
  const msg = JSON.parse(res.data)
})
socket.onError((err) => console.error(err))
socket.onClose((res) => {
  // res.code / res.reason，可在此实现断线重连
})
socket.close({ code: 1000, reason: 'normal' })
```

---

## 4. 数据存储

单个 key 上限 1MB，同一小程序所有数据上限 10MB。

```js
// 同步（简单场景优先用同步，代码更直观）
wx.setStorageSync('user', { id: 1, name: 'Tom' })
const user = wx.getStorageSync('user')     // 不存在时返回 ''
wx.removeStorageSync('user')
wx.clearStorageSync()

// 异步（大数据量时避免阻塞主线程）
wx.setStorage({
  key: 'user',
  data: { id: 1 },
  success() {}
})
wx.getStorage({ key: 'user', success(res) { console.log(res.data) } })
wx.removeStorage({ key: 'user' })
wx.clearStorage()

// 查看存储信息
const info = wx.getStorageInfoSync()
// info.keys        当前所有 key 数组
// info.currentSize 当前占用空间 KB
// info.limitSize   限制空间 KB

wx.getStorageInfo({ success(res) { console.log(res.keys) } })
```

带过期时间的简单封装：

```js
export function setCache(key, data, ttl = 3600 * 1000) {
  wx.setStorageSync(key, { data, expire: Date.now() + ttl })
}

export function getCache(key) {
  const cache = wx.getStorageSync(key)
  if (!cache || Date.now() > cache.expire) {
    wx.removeStorageSync(key)
    return null
  }
  return cache.data
}
```

---

## 5. 位置相关

```js
// 获取当前位置（需在 app.json 中配置 requiredPrivateInfos: ["getLocation"]）
wx.getLocation({
  type: 'gcj02',       // 'wgs84' 返回 GPS 坐标；'gcj02' 可用于 wx.openLocation
  altitude: false,     // 传 true 会更耗时
  isHighAccuracy: false,
  success(res) {
    // res.latitude / res.longitude / res.speed / res.accuracy
  },
  fail(err) {
    // 用户拒绝或未开启系统定位
  }
})

// 打开地图让用户选点
wx.chooseLocation({
  success(res) {
    // res.name 位置名称 / res.address 详细地址 / res.latitude / res.longitude
  }
})

// 用微信内置地图查看位置
wx.openLocation({
  latitude: 39.908,
  longitude: 116.397,
  scale: 18,           // 缩放级别 5-18
  name: '天安门',
  address: '北京市东城区'
})

// 持续监听位置变化
wx.startLocationUpdate({ success() {} })
wx.onLocationChange((res) => console.log(res.latitude, res.longitude))
wx.offLocationChange()
wx.stopLocationUpdate()
```

> `app.json` 配置示例：
> ```json
> {
>   "requiredPrivateInfos": ["getLocation", "chooseLocation", "onLocationChange"],
>   "permission": {
>     "scope.userLocation": { "desc": "你的位置信息将用于展示附近的门店" }
>   }
> }
> ```

---

## 6. 媒体相关

### 图片

```js
// 选择图片（推荐用 chooseMedia，同时支持图片和视频）
wx.chooseMedia({
  count: 9,                       // 最多可选数量
  mediaType: ['image'],           // 'image' | 'video' | 'mix'
  sourceType: ['album', 'camera'],
  sizeType: ['compressed'],       // 'original' | 'compressed'
  camera: 'back',
  success(res) {
    // res.tempFiles: [{ tempFilePath, size, fileType, width, height }]
    const paths = res.tempFiles.map(f => f.tempFilePath)
  }
})

// 旧版 API（仍可用）
wx.chooseImage({ count: 9, success(res) { console.log(res.tempFilePaths) } })

// 预览图片（支持双指缩放、长按保存）
wx.previewImage({
  current: urls[0],  // 当前显示的图片链接
  urls: urls         // 需要预览的图片链接列表
})

// 压缩图片
wx.compressImage({
  src: tempFilePath,
  quality: 80,        // 0-100，仅对 jpg 有效
  compressedWidth: 800,
  success(res) { console.log(res.tempFilePath) }
})

// 保存到相册（需要 scope.writePhotosAlbum 授权）
wx.saveImageToPhotosAlbum({
  filePath: tempFilePath,
  success() { wx.showToast({ title: '已保存' }) }
})

// 获取图片信息
wx.getImageInfo({
  src: 'https://example.com/a.png',
  success(res) { console.log(res.width, res.height, res.orientation) }
})
```

### 视频

```js
wx.chooseVideo({
  sourceType: ['album', 'camera'],
  maxDuration: 60,       // 最长拍摄时间（秒）
  camera: 'back',
  compressed: true,
  success(res) {
    // res.tempFilePath / res.duration / res.size / res.width / res.height
  }
})

// 控制页面上的 <video> 组件
const ctx = wx.createVideoContext('myVideo', this)
ctx.play()
ctx.pause()
ctx.seek(30)
ctx.requestFullScreen({ direction: 90 })
```

### 录音

```js
const recorder = wx.getRecorderManager()

recorder.start({
  duration: 60000,          // 最长录音时长 ms，最大 600000
  sampleRate: 44100,
  numberOfChannels: 1,
  encodeBitRate: 192000,
  format: 'mp3'             // 'mp3' | 'aac' | 'wav' | 'PCM'
})

recorder.onStart(() => console.log('开始录音'))
recorder.onStop((res) => {
  // res.tempFilePath / res.duration / res.fileSize
})
recorder.onError((err) => console.error(err))
recorder.onFrameRecorded((res) => console.log(res.frameBuffer))

recorder.pause()
recorder.resume()
recorder.stop()
```

### 音频播放

```js
const audio = wx.createInnerAudioContext()
audio.src = 'https://example.com/music.mp3'
audio.autoplay = false
audio.loop = false

audio.play()
audio.pause()
audio.stop()
audio.seek(30)          // 跳转到 30 秒

audio.onPlay(() => {})
audio.onTimeUpdate(() => console.log(audio.currentTime, audio.duration))
audio.onEnded(() => {})
audio.onError((err) => console.error(err.errCode, err.errMsg))

audio.destroy()         // 页面卸载时务必销毁
```

---

## 7. 交互反馈

```js
// 轻提示（不带 icon 时最多两行文字，用 icon:'none' 可显示较长文本）
wx.showToast({
  title: '操作成功',
  icon: 'success',   // 'success' | 'error' | 'loading' | 'none'
  duration: 1500,
  mask: true         // 是否显示透明蒙层，防止触摸穿透
})
wx.hideToast()

// 模态对话框
wx.showModal({
  title: '提示',
  content: '确定要删除吗？',
  showCancel: true,
  cancelText: '取消',
  confirmText: '确定',
  confirmColor: '#576B95',
  success(res) {
    if (res.confirm) { /* 点了确定 */ }
    else if (res.cancel) { /* 点了取消 */ }
  }
})

// 底部操作菜单（最多 6 项）
wx.showActionSheet({
  itemList: ['拍照', '从相册选择', '删除'],
  itemColor: '#000000',
  success(res) {
    console.log(res.tapIndex)  // 点击的序号，从 0 开始
  },
  fail() { /* 用户取消 */ }
})

// 加载提示（必须手动 hideLoading，且与 showToast 共用一个层）
wx.showLoading({ title: '加载中', mask: true })
wx.hideLoading()
```

Promise 化封装：

```js
export const confirm = (content, title = '提示') =>
  new Promise((resolve) => {
    wx.showModal({ title, content, success: (res) => resolve(res.confirm) })
  })

// 使用
if (await confirm('确定要删除吗？')) {
  await deleteItem(id)
}
```

---

## 8. 导航

### 页面跳转

```js
// 保留当前页，跳转到新页面（页面栈最多 10 层）
wx.navigateTo({
  url: '/pages/detail/detail?id=1&from=list',
  success(res) {
    // 向被打开页面传数据（需配合被打开页的 eventChannel）
    res.eventChannel.emit('initData', { data: 'hello' })
  }
})

// 关闭当前页，跳转到新页面（不能跳 tabBar）
wx.redirectTo({ url: '/pages/detail/detail?id=1' })

// 跳转到 tabBar 页面（会关闭所有非 tabBar 页面，url 不能带参数）
wx.switchTab({ url: '/pages/index/index' })

// 关闭所有页面，打开某个页面（可跳 tabBar）
wx.reLaunch({ url: '/pages/index/index?id=1' })

// 返回上一页
wx.navigateBack({ delta: 1 })

// 获取当前页面栈
const pages = getCurrentPages()
const current = pages[pages.length - 1]
const prev = pages[pages.length - 2]
```

页面间通信（`navigateTo` 的 eventChannel）：

```js
// A 页面
wx.navigateTo({
  url: '/pages/detail/detail',
  events: {
    acceptDataFromDetail(data) { console.log(data) }   // 接收 B 传回的数据
  },
  success(res) { res.eventChannel.emit('initData', { id: 1 }) }
})

// B 页面
onLoad() {
  const ch = this.getOpenerEventChannel()
  ch.on('initData', (data) => this.setData({ id: data.id }))
  ch.emit('acceptDataFromDetail', { result: 'ok' })
}
```

### 导航栏

```js
wx.setNavigationBarTitle({ title: '商品详情' })

wx.setNavigationBarColor({
  frontColor: '#ffffff',    // 只能是 #ffffff 或 #000000
  backgroundColor: '#1989fa',
  animation: { duration: 300, timingFunc: 'easeIn' }
})

wx.showNavigationBarLoading()
wx.hideNavigationBarLoading()

// 隐藏返回首页按钮（仅在从分享等场景进入时出现）
wx.hideHomeButton()
```

### TabBar

```js
wx.setTabBarBadge({ index: 2, text: '99+' })   // 右上角文字徽标
wx.removeTabBarBadge({ index: 2 })

wx.showTabBarRedDot({ index: 1 })              // 红点
wx.hideTabBarRedDot({ index: 1 })

wx.setTabBarItem({
  index: 0,
  text: '首页',
  iconPath: '/images/home.png',
  selectedIconPath: '/images/home-active.png'
})

wx.setTabBarStyle({
  color: '#7A7E83',
  selectedColor: '#1989fa',
  backgroundColor: '#ffffff',
  borderStyle: 'black'
})

wx.hideTabBar({ animation: true })
wx.showTabBar({ animation: true })
```

---

## 9. 界面

### 下拉刷新

```js
// 页面 json 中开启：{ "enablePullDownRefresh": true }
Page({
  onPullDownRefresh() {
    this.loadData().then(() => {
      wx.stopPullDownRefresh()   // 必须手动停止
    })
  },
  onReachBottom() {              // 上拉触底，配合 onReachBottomDistance 使用
    this.loadMore()
  }
})

// 代码触发下拉刷新
wx.startPullDownRefresh()
wx.stopPullDownRefresh()
```

### 滚动

```js
// 滚动到页面指定位置
wx.pageScrollTo({
  scrollTop: 0,
  duration: 300      // 0 表示无动画，立即跳转
})

// 滚动到指定选择器位置（2.23.1+）
wx.pageScrollTo({ selector: '#section-3', duration: 300 })

// 获取节点位置信息
wx.createSelectorQuery()
  .select('#target')
  .boundingClientRect((rect) => {
    console.log(rect.top, rect.height)
  })
  .exec()

// 组件内需加 .in(this)
this.createSelectorQuery().in(this).select('.item').boundingClientRect().exec()
```

### 动画

```js
const animation = wx.createAnimation({
  duration: 400,
  timingFunction: 'ease',   // linear | ease | ease-in | ease-out | ease-in-out | step-start | step-end
  delay: 0,
  transformOrigin: '50% 50% 0'
})

animation.opacity(0.5).translateY(100).rotate(45).scale(1.2).step()
animation.opacity(1).translateY(0).step({ duration: 200 })

this.setData({ animationData: animation.export() })  // export 后动画队列会被清空
```

```html
<view animation="{{animationData}}">动画元素</view>
```

> 复杂/高帧率动画优先用 CSS animation 或 WXS 响应事件，避免频繁 `setData` 引发的通信开销。

---

## 10. 设备

### 屏幕

```js
wx.getScreenBrightness({ success(res) { console.log(res.value) } })  // 0-1
wx.setScreenBrightness({ value: 0.8 })
wx.setKeepScreenOn({ keepScreenOn: true })    // 保持常亮（仅当前小程序生效）

// 监听截屏
wx.onUserCaptureScreen(() => console.log('用户截屏了'))
```

### 振动

```js
wx.vibrateShort({ type: 'medium' })   // 15ms，type: 'heavy' | 'medium' | 'light'
wx.vibrateLong()                      // 400ms
```

### 剪贴板

```js
wx.setClipboardData({
  data: 'https://example.com',
  success() { /* 会自动弹出「内容已复制」提示 */ }
})

wx.getClipboardData({ success(res) { console.log(res.data) } })
```

### 拨打电话

```js
wx.makePhoneCall({ phoneNumber: '10086' })
```

### 扫码

```js
wx.scanCode({
  onlyFromCamera: false,                  // true 则只允许相机扫码，不允许相册
  scanType: ['qrCode', 'barCode'],        // 还支持 'datamatrix' | 'pdf417'
  success(res) {
    // res.result 扫码内容 / res.scanType 码类型 / res.charSet 字符集
  }
})
```

### 其他常用

```js
// 网络加速度、陀螺仪、罗盘
wx.startAccelerometer({ interval: 'normal' })
wx.onAccelerometerChange((res) => console.log(res.x, res.y, res.z))
wx.stopAccelerometer()

wx.onCompassChange((res) => console.log(res.direction))

// 拨号盘、日历、联系人等
wx.addPhoneContact({ firstName: '张三', mobilePhoneNumber: '13800138000' })

// WiFi、蓝牙、NFC 请查阅官方文档
```

---

## 11. 转发分享

```js
Page({
  // 分享给朋友：需页面右上角菜单或 <button open-type="share"> 触发
  onShareAppMessage(res) {
    if (res.from === 'button') {
      console.log(res.target)   // 触发分享的按钮
    }
    return {
      title: '快来看看这个商品',
      path: '/pages/detail/detail?id=1&from=share',  // 必须是绝对路径
      imageUrl: 'https://example.com/share.png'      // 5:4 比例，不填则截取页面
    }
  },

  // 分享到朋友圈（需先支持 onShareAppMessage；仅 Android 支持）
  onShareTimeline() {
    return {
      title: '快来看看这个商品',
      query: 'id=1',                                 // 只能传 query，不能传 path
      imageUrl: 'https://example.com/share.png'      // 1:1 比例
    }
  },

  // 收藏
  onAddToFavorites(res) {
    return {
      title: '商品详情',
      imageUrl: 'https://example.com/fav.png',
      query: 'id=1'
    }
  }
})
```

```js
// 显示/隐藏「转发」按钮
wx.showShareMenu({ withShareTicket: true, menus: ['shareAppMessage', 'shareTimeline'] })
wx.hideShareMenu()

// 主动拉起转发面板（2.32.3+，需用户点击触发）
wx.shareAppMessage({ title: '标题', path: '/pages/index/index' })
```

---

## 12. 性能监控

```js
const performance = wx.getPerformance()

// 查询已有的性能数据
const entries = performance.getEntries()
entries.forEach(e => {
  console.log(e.entryType, e.name, e.duration, e.startTime)
})
// entryType: 'navigation'（页面切换）| 'render'（渲染）| 'script'（脚本）| 'loadPackage'

// 监听性能数据上报
const observer = performance.createObserver((list) => {
  list.getEntries().forEach(entry => {
    console.log(entry.name, entry.duration)
  })
})
observer.observe({ entryTypes: ['render', 'script', 'navigation'] })
observer.disconnect()

// 自定义打点（2.9.2+，id 需先在 mp 后台注册，value 单位 ms）
wx.reportPerformance(1001, 350)
```

```js
// 上报自定义分析事件
wx.reportEvent('event_id', { key: 'value' })

// 内存告警
wx.onMemoryWarning((res) => {
  console.log(res.level)  // 5 trim / 10 low / 15 critical
  // 主动释放缓存、销毁不用的实例
})
```

---

## 13. 更新管理

```js
// 通常写在 app.js 的 onLaunch 中
const updateManager = wx.getUpdateManager()

updateManager.onCheckForUpdate((res) => {
  console.log(res.hasUpdate)   // 是否有新版本
})

updateManager.onUpdateReady(() => {
  wx.showModal({
    title: '更新提示',
    content: '新版本已就绪，是否重启应用？',
    showCancel: false,
    success(res) {
      if (res.confirm) {
        updateManager.applyUpdate()   // 强制重启并应用新版本
      }
    }
  })
})

updateManager.onUpdateFailed(() => {
  wx.showModal({
    title: '更新失败',
    content: '请删除当前小程序，重新搜索打开',
    showCancel: false
  })
})

// 兼容判断
if (!wx.canIUse('getUpdateManager')) {
  wx.showModal({ title: '提示', content: '当前微信版本过低，请升级微信' })
}
```

---

## 常用 Promise 封装工具

```js
// utils/wx-promise.js
// 把回调式 API 统一转成 Promise（基础库较低时兜底用）
export function promisify(fn) {
  return (options = {}) =>
    new Promise((resolve, reject) => {
      fn({ ...options, success: resolve, fail: reject })
    })
}

export const getLocation = promisify(wx.getLocation)
export const chooseMedia = promisify(wx.chooseMedia)
export const scanCode = promisify(wx.scanCode)
```

---

## 参考资料

- [微信小程序官方 API 文档](https://developers.weixin.qq.com/miniprogram/dev/api/)
