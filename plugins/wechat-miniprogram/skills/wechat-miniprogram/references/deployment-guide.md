# 微信小程序发布与部署指南

本文覆盖小程序从合规准备到上线发布的完整链路：备案、隐私保护指引、域名配置、发布流程、CI/CD 自动化以及常见问题排查。

## 目录

1. [备案指引](#1-备案指引2023-年-9-月起强制)
2. [用户隐私保护指引](#2-用户隐私保护指引2024-年起强制)
3. [域名配置](#3-域名配置)
4. [发布流程](#4-发布流程)
5. [CI/CD 自动化](#5-cicd-自动化)
6. [常见问题](#6-常见问题)

---

## 1. 备案指引（2023 年 9 月起强制）

自 2023 年 9 月起，所有新上架的小程序必须完成 ICP 备案后才能提交审核并发布。存量小程序也需在规定期限内补全备案，否则会被下架。

### 1.1 所需材料

| 主体类型 | 必备材料 |
| --- | --- |
| 企业 / 个体工商户 | 营业执照原件照片、法定代表人身份证正反面、主体负责人与小程序负责人身份证及手机号、办公场所照片（部分省份要求） |
| 个人 | 身份证正反面、本人手持身份证照片、常用手机号与邮箱 |
| 事业单位 / 社会组织 | 统一社会信用代码证书、法人证书、负责人身份证 |
| 特殊行业 | 前置审批文件（见下表） |

**需要前置审批的行业**：

| 行业 | 所需前置审批 |
| --- | --- |
| 新闻资讯 | 互联网新闻信息服务许可证 |
| 网络出版 / 小说 | 网络出版服务许可证 |
| 网络游戏 | 网络文化经营许可证 + 版号 |
| 网络表演 / 直播 | 网络文化经营许可证 |
| 医疗器械 / 药品 | 医疗器械经营许可证、药品经营许可证 |
| 金融 / 支付 | 金融业务相关牌照 |
| 教育培训 | 办学许可证 |

> 前置审批文件必须在提交备案前取得，且经营范围需与小程序服务类目一致。

### 1.2 备案流程

```
准备材料
   ↓
登录微信公众平台 → 设置 → 基本设置 → 小程序备案
   ↓
填写主体信息、小程序信息、负责人信息，上传证件
   ↓
平台初审（1-3 个工作日）
   ↓
工信部管局审核（约 20 个工作日）
   ↓
备案成功，获得备案号（如：粤ICP备2023xxxxxx号-1X）
```

**各阶段说明**：

1. **准备材料** — 证件照片需清晰、四角完整、无反光，建议使用原件拍摄而非复印件。
2. **填写信息** — 小程序名称需与备案信息一致；服务内容描述要具体，避免"提供各类服务"这类模糊表述。
3. **平台初审** — 微信侧核验材料完整性与真实性，驳回后可修改重提，不影响管局审核排队。
4. **管局审核** — 由小程序主体所在省份通信管理局审核。期间可能会收到短信/电话核验，需及时响应，超时未响应会被驳回。
5. **备案成功** — 收到通知后，备案号会展示在公众平台的备案信息页。

### 1.3 重要注意事项

- **备案期间可以开发和真机调试，但无法提交代码审核**。开发者工具、体验版正常可用，"提交审核"按钮会被禁用。
- **必须在小程序内展示备案号**。通常放在「关于我们」「设置」页面底部，或首页页脚。未展示会在审核时被驳回。
- **信息变更需重新备案**。主体名称、小程序名称、负责人、服务内容发生变更时，须提交变更备案，否则原备案失效。
- **备案与认证是两回事**。企业主体还需完成微信认证（300 元/年）才能使用支付、部分开放接口等能力。
- **一个主体可备案多个小程序**，备案号后缀依次为 `-1X`、`-2X`、`-3X`。

### 1.4 展示备案号的代码示例

`pages/about/about.wxml`：

```html
<view class="about-page">
  <view class="content">
    <!-- 页面主体内容 -->
  </view>

  <view class="footer">
    <text class="copyright">© 2024 某某科技有限公司</text>
    <text
      class="icp-number"
      bindtap="onTapIcp"
    >{{icpNumber}}</text>
  </view>
</view>
```

`pages/about/about.js`：

```js
Page({
  data: {
    icpNumber: '粤ICP备2023123456号-1X',
  },

  onTapIcp() {
    wx.setClipboardData({
      data: 'https://beian.miit.gov.cn',
      success: () => {
        wx.showToast({ title: '备案查询网址已复制', icon: 'none' })
      },
    })
  },
})
```

`pages/about/about.wxss`：

```css
.footer {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40rpx 0;
}

.copyright,
.icp-number {
  font-size: 24rpx;
  color: #999;
  line-height: 40rpx;
}
```

> 也可以把备案号抽成全局配置，在 `app.js` 的 `globalData` 中维护，避免多页面重复硬编码。

---

## 2. 用户隐私保护指引（2024 年起强制）

自 2023 年 9 月 15 日起小程序需配置《小程序用户隐私保护指引》，2024 年起未配置的小程序调用隐私接口会直接失败。核心规则：**调用涉及用户隐私的接口前，必须先让用户同意隐私协议**。

### 2.1 需要声明的隐私接口

| 隐私类型（`requiredPrivateInfos` / 权限项） | 涉及接口或组件 | 说明 |
| --- | --- | --- |
| 用户信息 | `wx.getUserProfile`、`wx.getUserInfo` | 昵称、头像等 |
| 手机号 | `<button open-type="getPhoneNumber">`、`wx.getPhoneNumber` | 需企业主体 + 认证 |
| 位置信息 | `wx.getLocation`、`wx.onLocationChange`、`wx.startLocationUpdate`、`wx.chooseLocation`、`wx.choosePoi` | 需在 `requiredPrivateInfos` 中显式声明 |
| 后台定位 | `wx.startLocationUpdateBackground` | 需额外申请权限 |
| 相册 | `wx.chooseImage`、`wx.chooseMedia`、`wx.chooseVideo`、`wx.saveImageToPhotosAlbum`、`wx.saveVideoToPhotosAlbum` | 读写相册 |
| 摄像头 | `<camera>` 组件、`wx.createCameraContext`、`wx.scanCode` | 拍摄与扫码 |
| 麦克风 | `wx.startRecord`、`wx.getRecorderManager`、`<live-pusher>` | 录音 |
| 剪贴板 | `wx.getClipboardData`、`wx.setClipboardData` | 读写剪贴板 |
| 蓝牙 | `wx.openBluetoothAdapter` 及相关接口 | |
| 通讯录 | `wx.addPhoneContact`、`wx.chooseContact` | |
| 日历 | `wx.addPhoneRepeatCalendar`、`wx.addPhoneCalendar` | |
| 微信运动 | `wx.getWeRunData` | 步数数据 |
| 发票 | `wx.chooseInvoiceTitle`、`wx.chooseInvoice` | |
| 设备信息 | `wx.getDeviceInfo` 中的部分字段 | |

> 完整清单以官方《小程序用户隐私保护指引》为准，可能随基础库版本更新增删。

### 2.2 配置步骤

**第一步：在微信公众平台配置隐私协议**

登录 [微信公众平台](https://mp.weixin.qq.com) → 设置 → 服务内容与声明 → 用户隐私保护指引 → 填写并提交审核（约 1-3 个工作日）。

需逐项勾选实际会收集的信息类型，并说明用途。**勾选范围必须覆盖代码中实际调用的接口**，否则接口调用会返回 `fail api scope is not declared in the privacy agreement`。

**第二步：在 `app.json` 中声明权限说明与隐私接口**

```json
{
  "pages": ["pages/index/index"],
  "permission": {
    "scope.userLocation": {
      "desc": "用于展示附近的门店并计算配送距离"
    },
    "scope.userFuzzyLocation": {
      "desc": "用于推荐所在城市的服务内容"
    },
    "scope.writePhotosAlbum": {
      "desc": "用于将生成的图片保存到您的相册"
    },
    "scope.record": {
      "desc": "用于录制语音留言"
    }
  },
  "requiredPrivateInfos": [
    "getLocation",
    "chooseLocation",
    "onLocationChange",
    "startLocationUpdate",
    "chooseAddress"
  ]
}
```

要点：

- `permission.scope.*.desc` 是授权弹窗中展示给用户的理由，需具体说明用途，长度不超过 30 个字符。
- `requiredPrivateInfos` 用于声明**地理位置相关接口**，未声明的接口调用会直接失败。数组最多 5 项。
- 使用云开发时，若在云函数中调用相关能力，同样需要在小程序端声明。

**第三步：实现隐私授权弹窗**

从基础库 2.32.3 起，可通过 `wx.onNeedPrivacyAuthorization` 拦截隐私接口调用并弹出自定义协议弹窗。

### 2.3 隐私弹窗组件完整实现

**目录结构**：

```
components/privacy-popup/
├── index.js
├── index.json
├── index.wxml
└── index.wxss
```

`components/privacy-popup/index.json`：

```json
{
  "component": true,
  "usingComponents": {}
}
```

`components/privacy-popup/index.js`：

```js
let privacyResolves = new Set()
let closeOtherPagePopUpHooks = new Set()

// 全局只注册一次，避免多页面重复挂载导致弹窗叠加
if (wx.onNeedPrivacyAuthorization) {
  wx.onNeedPrivacyAuthorization((resolve, eventInfo) => {
    privacyResolves.add(resolve)
    closeOtherPagePopUpHooks.forEach((hook) => hook(eventInfo))
  })
}

Component({
  properties: {
    // 隐私协议名称，通常为《XX小程序用户隐私保护指引》
    privacyContractName: {
      type: String,
      value: '《用户隐私保护指引》',
    },
  },

  data: {
    visible: false,
  },

  lifetimes: {
    attached() {
      this.closeOtherPagePopUp = () => {
        this.setData({ visible: true })
      }
      closeOtherPagePopUpHooks.add(this.closeOtherPagePopUp)

      if (wx.getPrivacySetting) {
        wx.getPrivacySetting({
          success: (res) => {
            if (res.needAuthorization) {
              this.setData({
                privacyContractName: res.privacyContractName || this.data.privacyContractName,
              })
            }
          },
        })
      }
    },

    detached() {
      closeOtherPagePopUpHooks.delete(this.closeOtherPagePopUp)
    },
  },

  methods: {
    // 用户点击「同意」——必须由 open-type="agreePrivacyAuthorization" 的 button 触发
    handleAgree() {
      this.setData({ visible: false })
      privacyResolves.forEach((resolve) => resolve({ event: 'agree' }))
      privacyResolves.clear()
      this.triggerEvent('agree')
    },

    // 用户点击「拒绝」
    handleDisagree() {
      this.setData({ visible: false })
      privacyResolves.forEach((resolve) => resolve({ event: 'disagree' }))
      privacyResolves.clear()
      this.triggerEvent('disagree')
    },

    // 打开隐私协议全文
    openPrivacyContract() {
      wx.openPrivacyContract({
        fail: () => {
          wx.showToast({ title: '隐私协议打开失败', icon: 'error' })
        },
      })
    },
  },
})
```

`components/privacy-popup/index.wxml`：

```html
<view class="privacy-mask" wx:if="{{visible}}">
  <view class="privacy-dialog">
    <view class="privacy-title">用户隐私保护提示</view>

    <view class="privacy-desc">
      感谢您使用本小程序。在使用前，请阅读并同意
      <text class="privacy-link" bindtap="openPrivacyContract">{{privacyContractName}}</text>。
      当您点击同意并开始使用产品服务时，即表示您已理解并同意该条款内容。
    </view>

    <view class="privacy-actions">
      <button class="btn btn-cancel" bindtap="handleDisagree">拒绝</button>
      <button
        class="btn btn-confirm"
        open-type="agreePrivacyAuthorization"
        bindagreeprivacyauthorization="handleAgree"
      >同意并继续</button>
    </view>
  </view>
</view>
```

> **关键点**：「同意」按钮必须使用 `open-type="agreePrivacyAuthorization"`，并监听 `bindagreeprivacyauthorization` 事件。仅用 `bindtap` 调用 `resolve` 不会真正授予授权，隐私接口依然会失败。

`components/privacy-popup/index.wxss`：

```css
.privacy-mask {
  position: fixed;
  inset: 0;
  z-index: 9999;
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(0, 0, 0, 0.5);
}

.privacy-dialog {
  width: 580rpx;
  padding: 48rpx 40rpx 32rpx;
  border-radius: 24rpx;
  background: #fff;
}

.privacy-title {
  font-size: 34rpx;
  font-weight: 600;
  text-align: center;
  margin-bottom: 32rpx;
}

.privacy-desc {
  font-size: 28rpx;
  line-height: 48rpx;
  color: #666;
  margin-bottom: 40rpx;
}

.privacy-link {
  color: #576b95;
}

.privacy-actions {
  display: flex;
  gap: 24rpx;
}

.btn {
  flex: 1;
  height: 80rpx;
  line-height: 80rpx;
  font-size: 30rpx;
  border-radius: 12rpx;
  margin: 0;
}

.btn::after {
  border: none;
}

.btn-cancel {
  background: #f5f5f5;
  color: #666;
}

.btn-confirm {
  background: #07c160;
  color: #fff;
}
```

**在页面中使用**：

`pages/index/index.json`：

```json
{
  "usingComponents": {
    "privacy-popup": "/components/privacy-popup/index"
  }
}
```

`pages/index/index.wxml`：

```html
<privacy-popup bindagree="onPrivacyAgree" />

<button bindtap="handleGetLocation">获取当前位置</button>
```

`pages/index/index.js`：

```js
Page({
  handleGetLocation() {
    // 若用户尚未授权，会自动触发 onNeedPrivacyAuthorization，弹出隐私弹窗
    wx.getLocation({
      type: 'gcj02',
      success: (res) => {
        console.log(res.latitude, res.longitude)
      },
      fail: (err) => {
        console.error('定位失败', err)
      },
    })
  },

  onPrivacyAgree() {
    console.log('用户已同意隐私协议')
  },
})
```

**主动查询授权状态**：

```js
wx.getPrivacySetting({
  success: (res) => {
    // res.needAuthorization: 是否需要弹出授权
    // res.privacyContractName: 隐私协议名称
    if (!res.needAuthorization) {
      // 已授权，可直接调用隐私接口
    }
  },
})
```

---

## 3. 域名配置

小程序发起的所有网络请求，目标域名必须在公众平台预先配置，否则真机上会报 `url not in domain list`。

### 3.1 服务器域名类型

登录公众平台 → 开发管理 → 开发设置 → 服务器域名。

| 类型 | 用途 | 数量上限 |
| --- | --- | --- |
| request 合法域名 | `wx.request` 发起的 HTTPS 请求 | 200 个 |
| socket 合法域名 | `wx.connectSocket` 建立的 WSS 连接 | 200 个 |
| uploadFile 合法域名 | `wx.uploadFile` 上传文件 | 200 个 |
| downloadFile 合法域名 | `wx.downloadFile` 下载文件 | 200 个 |
| udp 合法域名 | `wx.createUDPSocket` | 200 个 |
| tcp 合法域名 | `wx.createTCPSocket` | 200 个 |

**修改频次限制：每个自然月最多修改 5 次**（各类型合计），月初重置。因此不要频繁改动，建议一次性把测试/预发/生产域名配齐。

### 3.2 域名要求

- **必须使用 HTTPS / WSS**，且服务器需支持 **TLS 1.2 及以上**（TLS 1.0/1.1 已被停止支持）。
- **域名必须已完成 ICP 备案**，备案主体不要求与小程序主体一致，但需为有效备案。
- **不支持 IP 地址和 localhost**，必须是域名。
- **不支持端口号**（`https://api.example.com:8443` 无效），HTTPS 请使用默认 443 端口。
- **不支持 IP 直连与内网域名**。
- 域名配置**不含路径**，只填写到域名层级，例如填 `https://api.example.com` 而不是 `https://api.example.com/v1`。
- 支持配置**二级/三级域名**，但**不支持通配符**（不能填 `*.example.com`）。
- SSL 证书必须有效且被系统信任，自签证书会被拒绝。

**验证 TLS 版本**：

```bash
# 检查服务器是否支持 TLS 1.2
openssl s_client -connect api.example.com:443 -tls1_2 </dev/null

# 检查证书链是否完整
openssl s_client -connect api.example.com:443 -showcerts </dev/null
```

### 3.3 业务域名（web-view）

若使用 `<web-view>` 组件加载网页，需在「业务域名」中单独配置。

配置步骤：

1. 下载平台提供的校验文件（如 `xxxxxxxx.txt`）。
2. 将文件放到域名根目录，确保 `https://example.com/xxxxxxxx.txt` 可直接访问（不能重定向、不能加鉴权）。
3. 回到公众平台点击「保存并提交」完成校验。

要求：

- 域名必须已备案，且校验文件需长期保留（后续复验会再次访问）。
- 业务域名同样有修改次数限制（每月 50 次），且需管理员扫码确认。
- `<web-view>` 仅**非个人主体**小程序可使用。

### 3.4 开发环境跳过校验

本地开发时无需配置域名，可在微信开发者工具中关闭校验：

**详情 → 本地设置 → 勾选「不校验合法域名、web-view（业务域名）、TLS 版本以及 HTTPS 证书」**

也可通过项目配置文件设置：

`project.config.json`：

```json
{
  "appid": "wxxxxxxxxxxxxxxxxx",
  "projectname": "my-miniprogram",
  "setting": {
    "urlCheck": false,
    "es6": true,
    "enhance": true,
    "minified": true,
    "postcss": true
  }
}
```

> ⚠️ **该设置仅在开发者工具中生效**。体验版和正式版真机上始终强制校验域名，因此上线前务必在真机预览下完整回归一遍网络请求。

---

## 4. 发布流程

### 4.1 发布前检查清单

**代码审查**

- [ ] 移除所有 `console.log` 调试输出与测试代码
- [ ] 移除硬编码的测试账号、测试数据、mock 开关
- [ ] 检查是否有未使用的页面、组件、图片资源（影响包体积）
- [ ] 确认主包体积 < 2MB，总包体积 < 20MB
- [ ] 敏感信息（AppSecret、私钥）不得出现在小程序端代码中

**功能测试**

- [ ] 覆盖所有核心业务流程（登录、下单、支付、退款等）
- [ ] 在 iOS 与 Android 真机上分别验证
- [ ] 测试弱网、断网、请求超时下的表现
- [ ] 首次进入（无缓存）与二次进入（有缓存）均验证
- [ ] 分享卡片、扫码进入、小程序码进入等入口路径均可用
- [ ] 页面回退栈正确，无白屏与死循环跳转

**合规检查**

- [ ] 备案已完成，备案号已在小程序内展示
- [ ] 隐私保护指引已配置并通过审核
- [ ] 服务类目与实际功能一致
- [ ] 涉及特殊行业的资质已上传
- [ ] 无诱导分享、诱导关注、外部支付跳转等违规内容
- [ ] 用户协议、隐私政策入口可访问

**体验优化**

- [ ] 首屏加载时间可接受，长耗时操作有 loading 提示
- [ ] 所有异常路径有明确的错误提示，而非静默失败
- [ ] 空状态、加载中、错误状态三态齐全
- [ ] 按钮点击有反馈，避免重复提交
- [ ] 适配不同屏幕尺寸与刘海屏安全区

**配置检查**

- [ ] 服务器域名已配置正确（生产环境）
- [ ] 业务域名已配置（若使用 web-view）
- [ ] `app.json` 中 `permission` 与 `requiredPrivateInfos` 完整
- [ ] 小程序头像、名称、简介、截图已更新
- [ ] 客服消息、消息推送配置正常

### 4.2 版本发布流程

```
开发者工具「上传」→ 填写版本号与项目备注
   ↓
公众平台 → 版本管理 → 开发版本 → 提交审核
   ↓
选择服务类目、填写功能页面路径、测试账号
   ↓
微信审核（1-7 个工作日，通常 1-3 天）
   ↓  加急审核：约 2 小时（每年 5 次机会，仅限紧急修复）
审核通过 → 版本管理 → 审核版本 → 发布
   ↓
可选：全量发布 / 分阶段发布（灰度）
```

**各环节说明**：

| 环节 | 说明 |
| --- | --- |
| 上传 | 版本号建议遵循语义化版本（如 `1.2.0`），备注写清本次改动，便于回溯 |
| 体验版 | 上传后可将某个开发版本设为体验版，供内部测试（最多 100 名体验成员） |
| 提交审核 | 需填写功能页面路径与测试账号；若功能需登录，务必提供可用测试账号，否则大概率被驳回 |
| 加急审核 | 在提审页面勾选，每个自然年 5 次，用于修复线上严重问题 |
| 发布 | 审核通过后需手动点击发布，不会自动上线 |
| 分阶段发布 | 可按 1% / 5% / 10% / 20% / 50% 逐步放量，期间可随时「停止发布」回退，适合大版本 |
| 版本回退 | 版本管理页可回退到上一个线上版本，仅保留最近的历史版本 |

### 4.3 版本管理与灰度判断

通过 `wx.getAccountInfoSync` 可在运行时获取当前小程序的版本信息，用于埋点上报、灰度开关或环境判断。

```js
const accountInfo = wx.getAccountInfoSync()

const { appId, envVersion, version } = accountInfo.miniProgram
// appId:      小程序 AppID
// envVersion: 'develop' | 'trial' | 'release'（开发版 / 体验版 / 正式版）
// version:    线上版本号，开发版和体验版为空字符串

console.log(appId, envVersion, version)
```

**按环境切换 API 域名**：

`utils/env.js`：

```js
const HOST_MAP = {
  develop: 'https://api-dev.example.com',
  trial: 'https://api-staging.example.com',
  release: 'https://api.example.com',
}

const { envVersion } = wx.getAccountInfoSync().miniProgram

export const API_HOST = HOST_MAP[envVersion] || HOST_MAP.release
export const IS_PRODUCTION = envVersion === 'release'
```

> 三个环境的域名都需要在公众平台的 request 合法域名中配置，否则体验版真机会请求失败。

**版本更新提示**：小程序发布新版本后，用户端不会立即生效，需通过更新管理器主动提示。

```js
// app.js
App({
  onLaunch() {
    this.checkUpdate()
  },

  checkUpdate() {
    if (!wx.canIUse('getUpdateManager')) return

    const updateManager = wx.getUpdateManager()

    updateManager.onCheckForUpdate((res) => {
      if (!res.hasUpdate) return
    })

    updateManager.onUpdateReady(() => {
      wx.showModal({
        title: '更新提示',
        content: '新版本已经准备好，是否重启应用？',
        success: (res) => {
          if (res.confirm) {
            updateManager.applyUpdate()
          }
        },
      })
    })

    updateManager.onUpdateFailed(() => {
      wx.showModal({
        title: '更新失败',
        content: '新版本下载失败，请删除当前小程序后重新搜索打开',
        showCancel: false,
      })
    })
  },
})
```

---

## 5. CI/CD 自动化

微信官方提供 [`miniprogram-ci`](https://developers.weixin.qq.com/miniprogram/dev/devtools/ci.html) 包，可在没有开发者工具 GUI 的环境中完成上传、预览、构建 npm 等操作，适合接入 CI 流水线。

### 5.1 准备工作

1. 登录公众平台 → 开发管理 → 开发设置 → **小程序代码上传**，生成并下载代码上传密钥（`private.<appid>.key`）。
2. 在同一页面配置 **IP 白名单**：填入 CI 机器的出口 IP；若使用 GitHub Actions 等动态 IP 环境，需关闭 IP 白名单限制。
3. **密钥文件绝不能提交到仓库**，应通过 CI Secret 注入。

### 5.2 安装

```bash
npm install miniprogram-ci --save-dev
```

### 5.3 上传脚本

`scripts/upload.js`：

```js
const ci = require('miniprogram-ci')
const path = require('path')

const APPID = process.env.MP_APPID
const PRIVATE_KEY_PATH = process.env.MP_PRIVATE_KEY_PATH || path.resolve(__dirname, '../private.key')
const VERSION = process.env.MP_VERSION || require('../package.json').version
const DESC = process.env.MP_DESC || `CI 自动上传 ${new Date().toLocaleString('zh-CN')}`

async function main() {
  const project = new ci.Project({
    appid: APPID,
    type: 'miniProgram',
    projectPath: path.resolve(__dirname, '..'),
    privateKeyPath: PRIVATE_KEY_PATH,
    ignores: ['node_modules/**/*', '.git/**/*', 'scripts/**/*'],
  })

  const result = await ci.upload({
    project,
    version: VERSION,
    desc: DESC,
    setting: {
      es6: true,            // 转换 ES6 语法
      es7: true,            // 增强编译
      minify: true,         // 压缩代码
      minifyJS: true,
      minifyWXML: true,
      minifyWXSS: true,
      codeProtect: false,   // 代码保护（会增加包体积，按需开启）
      autoPrefixWXSS: true, // 自动补全 WXSS 前缀
      disableUseStrict: false,
    },
    robot: 1,               // 机器人编号 1-30，用于区分不同流水线来源
    onProgressUpdate: (task) => {
      if (typeof task === 'string') {
        console.log(task)
      } else if (task.message) {
        console.log(`[${task.status}] ${task.message}`)
      }
    },
  })

  console.log('上传成功')
  console.log(`版本：${VERSION}`)
  if (result && result.subPackageInfo) {
    result.subPackageInfo.forEach((pkg) => {
      console.log(`  ${pkg.name}: ${(pkg.size / 1024).toFixed(2)} KB`)
    })
  }
}

main().catch((err) => {
  console.error('上传失败：', err)
  process.exit(1)
})
```

`package.json`：

```json
{
  "scripts": {
    "upload": "node scripts/upload.js"
  }
}
```

### 5.4 生成预览二维码

`scripts/preview.js`：

```js
const ci = require('miniprogram-ci')
const path = require('path')

async function main() {
  const project = new ci.Project({
    appid: process.env.MP_APPID,
    type: 'miniProgram',
    projectPath: path.resolve(__dirname, '..'),
    privateKeyPath: process.env.MP_PRIVATE_KEY_PATH,
    ignores: ['node_modules/**/*'],
  })

  await ci.preview({
    project,
    desc: '预览版本',
    setting: { es6: true, es7: true, minify: true },
    qrcodeFormat: 'image',
    qrcodeOutputDest: path.resolve(__dirname, '../preview.jpg'),
    pagePath: 'pages/index/index',
    searchQuery: '',
    robot: 2,
  })

  console.log('预览二维码已生成：preview.jpg')
}

main().catch((err) => {
  console.error(err)
  process.exit(1)
})
```

### 5.5 其他常用能力

```js
// 构建 npm（使用了 npm 依赖时必须在上传前执行）
await ci.packNpm(project, {
  ignores: ['pack_npm_ignore_list'],
  reporter: (info) => console.log(info),
})

// 代码质量分析：产出包体积、分包信息等
const res = await ci.getDevSourceMap({ project, robot: 1, sourceMapSavePath: './sourcemap.zip' })

// 上传云开发云函数
await ci.cloud.uploadFunction({
  project,
  env: 'prod-xxxxx',
  name: 'login',
  path: './cloudfunctions/login',
  remoteNpmInstall: true,
})
```

### 5.6 GitHub Actions 工作流

`.github/workflows/deploy.yml`：

```yaml
name: Deploy MiniProgram

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:
    inputs:
      desc:
        description: '版本备注'
        required: false
        default: '手动触发上传'

jobs:
  upload:
    name: Upload to WeChat
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Write private key
        run: |
          printf '%s' "${{ secrets.MP_PRIVATE_KEY }}" > "${RUNNER_TEMP}/private.key"
          chmod 600 "${RUNNER_TEMP}/private.key"

      - name: Resolve version
        id: version
        run: echo "value=${GITHUB_REF_NAME#v}" >> "$GITHUB_OUTPUT"

      - name: Build npm
        run: npm run build:npm --if-present

      - name: Upload miniprogram
        env:
          MP_APPID: ${{ secrets.MP_APPID }}
          MP_PRIVATE_KEY_PATH: ${{ runner.temp }}/private.key
          MP_VERSION: ${{ steps.version.outputs.value }}
          MP_DESC: ${{ github.event.inputs.desc || github.event.head_commit.message }}
        run: npm run upload

      - name: Cleanup key
        if: always()
        run: rm -f "${RUNNER_TEMP}/private.key"
```

**配置 Secrets**：仓库 Settings → Secrets and variables → Actions，添加：

| Secret | 说明 |
| --- | --- |
| `MP_APPID` | 小程序 AppID |
| `MP_PRIVATE_KEY` | 代码上传密钥文件的完整内容（含 `-----BEGIN RSA PRIVATE KEY-----` 头尾） |

**注意事项**：

- GitHub Actions 的 runner IP 是动态的，需在公众平台**关闭上传密钥的 IP 白名单**，或改用自托管 runner 并固定出口 IP。
- CI 只能完成「上传」，**提交审核与发布仍需在公众平台手动操作**（官方未开放审核发布 API 给普通小程序）。
- `robot` 编号用于区分不同流水线（如 1 = 生产、2 = 预发、3 = 开发），在版本管理页可看到来源，便于排查。
- 密钥写入临时目录而非工作区，避免被误打包或缓存。

---

## 6. 常见问题

### 6.1 审核被驳回的常见原因

| 驳回原因 | 具体表现 | 解决办法 |
| --- | --- | --- |
| 功能不完整 | 页面空白、功能未完成、有"敬请期待"占位、点击无响应 | 补全功能后再提审；未完成的入口先隐藏而非留占位 |
| 内容违规 | 涉及色情低俗、赌博、虚假宣传、医疗夸大宣传、政治敏感 | 移除违规内容，参考《微信小程序平台运营规范》 |
| 诱导分享 | "分享得积分""邀请 3 人解锁""不转发不能继续" | 移除强制或利益诱导的分享逻辑，分享应为用户自愿行为 |
| 未配置隐私协议 | 调用了隐私接口但未在公众平台配置用户隐私保护指引 | 配置并通过隐私指引审核后重新提审 |
| 未备案 | 未完成 ICP 备案或未在小程序内展示备案号 | 完成备案，并在页面展示备案号 |
| 类目不符 | 实际功能超出所选服务类目范围（如选"工具"却做电商） | 调整服务类目或缩减功能范围；特殊类目需补充资质 |
| 缺少测试账号 | 核心功能需登录但未提供测试账号 | 提审时在备注中提供可用测试账号与验证码获取方式 |
| 支付违规 | 引导用户使用微信支付以外的支付方式，或跳转外部收银台 | 移除外部支付入口 |
| 用户信息违规 | 强制授权才能使用基础功能 | 改为按需授权，未授权时提供降级体验 |
| 后台交互异常 | 审核时接口报错、服务不可用 | 提审期间保持后端服务与测试数据可用 |

**驳回后的处理**：在公众平台「版本管理 → 审核版本」查看具体驳回理由与截图，修复后重新提交。多次因同一问题被驳回可能触发限制，建议逐条对照说明修改。若确认为误判，可在驳回详情页申诉。

### 6.2 域名相关报错排查

| 报错信息 | 原因 | 解决办法 |
| --- | --- | --- |
| `url not in domain list` | 域名未在公众平台配置 | 添加到对应类型的合法域名；开发时可临时关闭校验 |
| `request:fail -202` / `ERR_CERT_*` | SSL 证书无效、过期或链不完整 | 更新证书，补全中间证书链 |
| `request:fail ssl hand shake error` | 服务器 TLS 版本过低 | 升级到 TLS 1.2+ |
| 开发者工具正常，真机失败 | 工具关闭了域名校验，真机强制校验 | 补配域名后重新预览验证 |
| `web-view` 页面无法打开 | 业务域名未配置或校验文件失效 | 重新上传校验文件并保存业务域名 |
| 域名配置后仍失败 | 配置有缓存延迟 | 等待几分钟，重启开发者工具与微信客户端后重试 |
| 本月修改次数已用完 | 每月 5 次修改上限 | 等待下月，或提前规划一次性配齐全部环境域名 |

**排查顺序建议**：

1. 确认真机报错文案（用 `vConsole` 或真机调试查看）。
2. 在公众平台核对域名是否**完全一致**（协议、有无 `www` 前缀都算不同）。
3. 用 `curl -v https://api.example.com` 验证证书与连通性。
4. 检查是否配置到了正确的类型（`uploadFile` 的域名不会对 `wx.request` 生效）。

### 6.3 发布相关限制

| 限制项 | 数值 |
| --- | --- |
| 服务器域名修改 | 每月 5 次 |
| 业务域名修改 | 每月 50 次 |
| 加急审核 | 每年 5 次 |
| 体验成员 | 最多 100 人 |
| 开发者数量 | 认证后最多 20 人（未认证 10 人） |
| CI 机器人 | 1-30 号 |
| 主包体积 | ≤ 2MB |
| 单个分包体积 | ≤ 2MB |
| 总包体积 | ≤ 20MB |
| 小程序名称修改 | 认证账号每年 2 次 |
| 服务类目修改 | 每月 1 次 |

### 6.4 其他常见问题

**Q：提交审核后可以撤回吗？**
可以。在「版本管理 → 审核版本」点击「撤回审核」，每天有撤回次数限制（通常 1 次）。撤回后需重新提审并重新排队。

**Q：审核通过后忘记发布会怎样？**
审核版本会一直保留，不会自动上线，也不会过期，随时可以点击发布。

**Q：发布后发现严重问题怎么办？**
优先在「版本管理」使用**版本回退**回到上一个线上版本（立即生效），再修复代码走加急审核。若使用了分阶段发布，直接点「停止发布」即可。

**Q：新版本发布后用户为什么还是旧版？**
微信客户端有本地缓存，用户下次冷启动才会拉取新版本。需通过 `wx.getUpdateManager` 主动检测并提示重启（见 4.3 节）。

**Q：备案还没通过，能先做开发吗？**
可以。开发、真机调试、体验版都不受影响，仅「提交审核」被限制。

**Q：个人主体有哪些限制？**
不能使用 `<web-view>`、微信支付、部分开放接口；可选服务类目大幅受限（不能做电商、医疗、金融、社区等）。

---

## 相关文档

- [微信公众平台](https://mp.weixin.qq.com)
- [小程序备案指引](https://developers.weixin.qq.com/miniprogram/product/record/)
- [用户隐私保护指引](https://developers.weixin.qq.com/miniprogram/dev/framework/user-privacy/)
- [小程序 CI](https://developers.weixin.qq.com/miniprogram/dev/devtools/ci.html)
- [小程序平台运营规范](https://developers.weixin.qq.com/miniprogram/product/)
- 本地参考：[api.md](api.md) · [framework.md](framework.md) · [cloud.md](cloud.md)
