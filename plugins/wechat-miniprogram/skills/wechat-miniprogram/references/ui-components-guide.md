# 微信小程序 UI 组件库选型指南

小程序原生组件（`view` / `button` / `input` 等）只提供最基础的能力，实际项目通常需要引入 UI 组件库来补齐样式规范与复杂交互。本文对比三个主流方案并给出选型建议。

## 对比总览

| 维度 | WeUI | TDesign | CloudBase Agent UI |
|---|---|---|---|
| 维护方 | 微信官方 | 腾讯多 BG 联合 | 腾讯云开发（CloudBase） |
| 定位 | 原生体验，与微信视觉一致 | 企业级设计体系，多端统一 | AI 对话 / Agent 交互 |
| 引入方式 | 扩展库（推荐）或 npm | npm | npm |
| 包体积影响 | **0**（扩展库不计入包体积） | 约 100KB（按需引入后） | 约 50KB |
| 最低基础库 | 2.2.3+ | 2.6.0+ | 2.13.0+ |
| 组件数量 | 基础常用组件 | 丰富（60+） | 对话场景专用 |
| 主题定制 | 有限 | 强（CSS 变量 + 设计令牌） | 支持样式覆盖 |
| TypeScript | 部分 | 完整类型定义 | 完整类型定义 |

三者不互斥：可以用 TDesign 搭主体界面，同时引入 CloudBase Agent UI 处理 AI 对话页面。

---

## WeUI

微信官方出品的基础样式库，与微信客户端原生视觉保持一致。

### 特点

- **扩展库引入不占包体积**：通过 `useExtendedLib` 引入时，代码由微信客户端提供，不计入小程序 2MB 的分包体积限制。这是 WeUI 相对其他方案最大的优势。
- **原生视觉一致**：按钮、表单、弹窗、Toast 等的样式和交互与微信内置界面完全统一，用户无学习成本。
- **官方维护**：跟随基础库同步更新，兼容性风险最低。

### 安装方式一：扩展库（推荐）

在 `app.json` 中声明：

```json
{
  "useExtendedLib": {
    "weui": true
  }
}
```

这一行即可完成引入，无需 npm、无需构建，且不占用包体积。

### 安装方式二：npm

需要修改组件源码或锁定版本时使用：

```bash
npm install weui-miniprogram
```

安装后在微信开发者工具中执行「工具 → 构建 npm」。

### 组件注册

在需要使用的页面 `page.json` 中按需注册：

```json
{
  "usingComponents": {
    "mp-dialog": "weui-miniprogram/dialog/dialog",
    "mp-toptips": "weui-miniprogram/toptips/toptips",
    "mp-cells": "weui-miniprogram/cells/cells",
    "mp-cell": "weui-miniprogram/cell/cell"
  }
}
```

页面中使用：

```html
<mp-dialog title="提示" show="{{showDialog}}" bindbuttontap="onTapDialogButton" buttons="{{buttons}}">
  <view>确定要删除这条记录吗？</view>
</mp-dialog>
```

> 注意：即使使用扩展库方式引入，`usingComponents` 的路径依然写 `weui-miniprogram/...`，无需改动。

### 官方链接

- [微信官方文档](https://developers.weixin.qq.com/miniprogram/dev/platform-capabilities/extended/weui/)
- [GitHub](https://github.com/wechat-miniprogram/weui-miniprogram)
- [快速开始](https://wechat-miniprogram.github.io/weui/docs/quickstart.html)

---

## TDesign

腾讯多个事业群联合推出的企业级设计体系在小程序端的实现。

### 特点

- **企业级设计规范**：有完整的设计令牌（Design Token）体系，覆盖颜色、字号、间距、圆角、阴影，适合需要统一视觉语言的中大型项目。
- **多端统一**：同一套设计规范同时提供 Web（Vue/React）、小程序、Flutter 实现，跨端产品能保持界面一致。
- **组件丰富且可定制**：60+ 组件，覆盖导航、表单、数据展示、反馈、业务组件（日历、上传、级联选择等）；通过 CSS 变量可深度定制主题。
- **TypeScript 支持**：提供完整类型定义，配合 TS 项目有良好的类型提示。

### 安装

```bash
npm install tdesign-miniprogram
```

安装后在微信开发者工具中执行「工具 → 构建 npm」。

若项目根目录没有 `package.json`，先执行 `npm init -y`。

### 全局注册

在 `app.json` 中注册常用组件（也可在各页面 `page.json` 中按需注册）：

```json
{
  "usingComponents": {
    "t-button": "tdesign-miniprogram/button/button",
    "t-cell": "tdesign-miniprogram/cell/cell",
    "t-cell-group": "tdesign-miniprogram/cell-group/cell-group",
    "t-input": "tdesign-miniprogram/input/input",
    "t-toast": "tdesign-miniprogram/toast/toast"
  }
}
```

页面中使用：

```html
<t-button theme="primary" size="large" bind:tap="onSubmit">提交</t-button>

<t-cell-group>
  <t-cell title="用户名" note="{{userName}}" arrow />
  <t-cell title="手机号" note="{{phone}}" arrow />
</t-cell-group>
```

### 主题定制

在 `app.wxss` 中覆盖 CSS 变量：

```css
page {
  --td-brand-color: #0052d9;
  --td-radius-default: 12rpx;
}
```

### 官方链接

- [官网](https://tdesign.tencent.com/)
- [GitHub](https://github.com/Tencent/tdesign-miniprogram)

---

## CloudBase Agent UI

腾讯云开发提供的 AI 对话组件库，用于快速在小程序中接入 Agent 交互界面。

### 特点

- **开箱即用的 AI 对话能力**：提供完整的聊天界面（消息列表、输入框、打字机效果、思考过程展示），无需自己实现流式渲染与滚动逻辑。
- **对接混元 / DeepSeek 等模型**：通过云开发 AI 能力调用腾讯混元、DeepSeek 等大模型，也支持自建 Agent。
- **富交互能力**：内置文件上传、语音输入、联网搜索等扩展功能，可按需开关。
- **Agent 工具调用**：支持展示 Agent 的工具调用（Function Calling）过程和中间结果，适合做带工具能力的智能助手。

### 安装

```bash
npm install @cloudbase/ai-agent-ui
```

安装后在微信开发者工具中执行「工具 → 构建 npm」。

### 配置

**1. 配置 request 合法域名**

在微信公众平台「开发管理 → 开发设置 → 服务器域名」中，把云开发环境的接口域名加入 request 白名单。使用 `wx.cloud` 调用时通常无需额外配置，但涉及文件上传、联网搜索等能力时需要补充相关域名。

**2. 初始化云开发**

在 `app.js` 中：

```js
App({
  onLaunch() {
    wx.cloud.init({
      env: 'your-env-id',
      traceUser: true,
    });
  },
});
```

**3. 注册组件并使用**

`page.json`：

```json
{
  "usingComponents": {
    "agent-ui": "@cloudbase/ai-agent-ui/agent-ui/index"
  }
}
```

`page.wxml`：

```html
<agent-ui agentConfig="{{agentConfig}}" showBotAvatar="{{true}}" />
```

`page.js`：

```js
Page({
  data: {
    agentConfig: {
      botId: 'your-bot-id',
      allowUploadFile: true,
      allowWebSearch: true,
      allowVoice: true,
    },
  },
});
```

### 官方链接

- [文档](https://docs.cloudbase.net/ai/agent-ui/agent-ui-mp)
- [GitHub](https://github.com/TencentCloudBase/cloudbase-agent-ui)

---

## 选型建议

### 按场景

| 场景 | 推荐 | 理由 |
|---|---|---|
| 快速原型 / 工具类小程序 | **WeUI** | 一行配置即可用，0 包体积，无构建步骤 |
| 企业级 / 中大型业务 | **TDesign** | 组件齐全、设计规范完整、主题可定制、TS 类型完善 |
| AI 对话 / 智能客服 / Agent 助手 | **CloudBase Agent UI** | 对话界面开箱即用，直接对接云开发 AI 能力 |
| 多端统一（Web + 小程序） | **TDesign** | 同一套设计体系有多端实现，视觉与交互可对齐 |
| 界面要求与微信高度一致 | **WeUI** | 官方出品，样式与微信客户端完全统一 |

### 按包体积

小程序主包上限 2MB、总包（含分包）上限 20MB，组件库体积需要重点关注：

- **包体积极度敏感**（主包已接近上限）→ 选 WeUI 扩展库方式，占用为 0。
- **可接受约 100KB** → TDesign，务必按页面 `usingComponents` 按需引入，不要在 `app.json` 中全量注册未使用的组件。
- **仅对话页需要** → CloudBase Agent UI 放进**分包**，避免约 50KB 计入主包。

### 注意事项

**基础库版本**

- 在 `app.json` 中通过 `"libVersion"` 或在微信公众平台设置最低基础库版本。
- 引入组件库前，先确认目标用户的基础库覆盖率（微信公众平台「设置 → 基础库版本分布」可查），低于组件库要求的版本会直接报错或渲染异常。
- 三者最低要求：WeUI 2.2.3+、TDesign 2.6.0+、CloudBase Agent UI 2.13.0+。

**包体积控制**

- **按需引入**：只在实际使用的页面 `page.json` 中注册组件，避免在 `app.json` 全量注册。
- **善用分包**：把只在特定业务用到的组件库放进对应分包，减轻主包压力。
- **构建 npm 后检查**：npm 方式引入的组件库会生成 `miniprogram_npm/` 目录，用开发者工具「详情 → 基本信息 → 本地代码大小」核对实际增量。
- **不要混用过多库**：同时引入 WeUI 和 TDesign 会造成视觉风格割裂，且重复占用体积；除非有明确理由（如仅用 CloudBase Agent UI 补齐对话能力），否则主体界面只选一套。
