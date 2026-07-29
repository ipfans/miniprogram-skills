---
name: skyline
description: "WeChat Mini Program Skyline rendering engine. Use when developing with Skyline renderer, including components (scroll-view, swiper, draggable-sheet), WXSS styles, worklet animations, custom routes/transitions, scroll APIs, and Skyline configuration/migration."
version: 1.0.0
---

# Skyline 渲染引擎技能

微信小程序 Skyline 渲染引擎开发指南，涵盖组件、样式、动画、路由、滚动 API 和配置迁移。

## 适用场景

- 使用 Skyline 渲染引擎开发微信小程序
- 使用 Skyline 专属组件（scroll-view 增强、draggable-sheet、share-element 等）
- 实现 worklet 动画（SharedValue、timing、spring、decay）
- 实现自定义路由与页面转场（routeBuilder、open-container）
- 检查 WXSS/CSS 属性在 Skyline 下的兼容性
- 在 app.json / page.json 中配置 Skyline
- 从 WebView 迁移到 Skyline

## 文档索引

用 `Read` 工具按需加载对应模块的参考文档。

| 我想要... | 查阅模块 |
|-----------|----------|
| 了解 Skyline 架构、性能优势、迁移指南 | `../skyline-overview/SKILL.md` |
| 配置 app.json / page.json 启用 Skyline | `../skyline-config/SKILL.md` |
| 使用 scroll-view、swiper、draggable-sheet 等组件 | `../skyline-components/SKILL.md` |
| 检查 CSS 属性/值在 Skyline 下是否支持 | `../skyline-wxss/SKILL.md` |
| 实现 worklet 动画（SharedValue、timing、spring） | `../skyline-worklet/SKILL.md` |
| 实现自定义路由、预设路由、容器转场 | `../skyline-route/SKILL.md` |
| 使用 ScrollViewContext / DraggableSheetContext API | `../skyline-scroll-api/SKILL.md` |

## 快速上手

Skyline 的最小必需配置见 `../skyline-config/SKILL.md`，核心要点：

- `app.json` 中设置 `"renderer": "skyline"` 和 `"componentFramework": "glass-easel"`
- Skyline 页面默认不可滚动，必须用 `<scroll-view type="list" scroll-y>` 包裹内容
