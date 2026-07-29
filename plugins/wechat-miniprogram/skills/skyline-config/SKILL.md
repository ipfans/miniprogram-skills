---
name: skyline-config
description: Skyline JSON 配置规范：app.json 全局配置、页面 json 配置、project.config.json、混合渲染配置。
---

# Skyline JSON 配置规范

## 文档索引

根据需求快速定位（路径相对于 `references/`）：

| 我想要... | 查阅文档 |
|-----------|----------|
| 了解 app.json 中所有 Skyline 相关配置 | `app-config.md` |
| 了解页面级配置和混合渲染 | `page-config.md` |
| 配置开发者工具 | `project-config.md` |
| 查看完整配置模板 | `patterns.md` |

## 核心概念

### 三级配置层次

| 层级 | 文件 | 作用 | 关键配置 |
|------|------|------|----------|
| 全局 | `app.json` | 全局启用 Skyline | renderer, componentFramework, rendererOptions |
| 页面 | `页面.json` | 页面级配置/覆盖 | navigationStyle, disableScroll, renderer |
| 工具 | `project.config.json` | 开发者工具调试 | setting.skylineRenderEnable |

### 最小必需配置

```json
// app.json
{
  "renderer": "skyline",
  "componentFramework": "glass-easel",
  "lazyCodeLoading": "requiredComponents"
}
```

```json
// 每个页面的 .json
{
  "navigationStyle": "custom"
}
```

## 强制规则

### app.json 必须包含三项必需配置

```json
{
  "renderer": "skyline",
  "componentFramework": "glass-easel",
  "lazyCodeLoading": "requiredComponents"
}
```

缺少 `componentFramework` 或 `lazyCodeLoading` 都会导致编译错误。

### 每个页面的 json 必须配置 `"navigationStyle": "custom"`

```json
{
  "navigationStyle": "custom"
}
```

Skyline 不支持原生导航栏。缺少此配置会报错：`getAppConfig error: the "navigationStyle" configuration for the page should be set to "custom"`。

### 使用 scroll-view 替代页面级滚动时配置 `"disableScroll": true`

```json
{
  "navigationStyle": "custom",
  "disableScroll": true
}
```

未禁用页面滚动可能与 scroll-view 冲突。

### rendererOptions 应配置 defaultDisplayBlock 和 defaultContentBox 对齐 WebView 行为

```json
{
  "renderer": "skyline",
  "rendererOptions": {
    "skyline": {
      "defaultDisplayBlock": true,
      "defaultContentBox": true
    }
  }
}
```

不配置时 Skyline 默认 display:flex + border-box，与 WebView 的 block + content-box 不一致。

### 所有 Skyline 页面都必须声明 navigationStyle，不要在 Skyline 页面中依赖页面级全局滚动

Skyline 不支持页面级滚动，必须使用 `scroll-view` 组件。

## Quick Reference

### 必需配置速查

| 配置项 | 位置 | 值 | 级别 |
|--------|------|-----|------|
| `renderer` | app.json | `"skyline"` | 必需 |
| `componentFramework` | app.json | `"glass-easel"` | 必需 |
| `lazyCodeLoading` | app.json | `"requiredComponents"` | 必需 |
| `navigationStyle` | 页面 json | `"custom"` | 必需 |
| `disableScroll` | 页面 json | `true` | 推荐 |

### rendererOptions 配置速查

| 配置项 | 类型 | 默认值 | 推荐值 | 说明 |
|--------|------|--------|--------|------|
| `defaultDisplayBlock` | boolean | false | true | 默认 display:block（对齐 WebView） |
| `defaultContentBox` | boolean | false | true | 默认 box-sizing:content-box（对齐 WebView） |
| `tagNameStyleIsolation` | string | "isolated" | "legacy" | 标签选择器全局匹配（对齐 WebView） |
| `enableScrollViewAutoSize` | boolean | false | true | scroll-view 自动撑开高度 |
| `disableABTest` | boolean | false | true | 关闭 Skyline AB 实验，确保稳定性 |

### 场景决策表

| 场景 | 推荐配置 |
|------|----------|
| 新建 Skyline 项目 | 三项必需 + rendererOptions 全部推荐值 |
| WebView 迁移 | 三项必需 + rendererOptions 兼容配置 + disableABTest |
| 混合渲染 | app.json 不设 renderer，页面级单独设 `"renderer": "skyline"` |
| 仅部分页面用 Skyline | 页面 json 中设 `"renderer": "skyline"` 覆盖全局 |
