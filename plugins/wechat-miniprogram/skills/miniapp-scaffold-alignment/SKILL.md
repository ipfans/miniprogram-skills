---
name: miniapp-scaffold-alignment
description: 验证微信小程序项目结构是否符合官方规范：project.config.json、app.json、页面/组件文件集、TypeScript 配置、仓库布局。Use when reviewing project scaffold, validating `miniprogramRoot`, checking page or component file sets, or setting up a new miniapp skeleton before feature work begins.
---

# 小程序项目结构验证

验证仓库结构是否符合微信小程序官方规范，兼容 JavaScript 和 TypeScript 项目。

## 文档索引

根据需求快速定位（路径相对于 `references/`）：

| 我想要...                      | 查阅文件                        |
| ------------------------------ | ------------------------------- |
| 了解官方项目结构基线规则       | `official-scaffold-baseline.md` |
| 查看验证 prompt 示例和评估说明 | `example-prompts.md`            |

## 验证流程

1. 读取 `references/official-scaffold-baseline.md`
2. 定位仓库根目录和小程序代码根目录（`miniprogramRoot`）
3. 逐项检查 `project.config.json`、`app.json`、页面路径、组件文件集、TypeScript 配置
4. 按输出格式报告结果

## 核心概念

### 仓库根目录 vs 小程序代码根目录

| 层级             | 说明                                                                     | 典型路径                      |
| ---------------- | ------------------------------------------------------------------------ | ----------------------------- |
| 仓库根目录       | 版本控制顶层，可能包含 docs/、后端代码、工具配置                         | `/`                           |
| 小程序代码根目录 | `project.config.json` 中 `miniprogramRoot` 指向的目录，包含 app 入口文件 | `/`、`/miniprogram/`、`/src/` |

当仓库包含非小程序的兄弟目录时，必须通过 `miniprogramRoot` 显式指定代码根目录。

### 文件集匹配规则

每个页面和组件必须拥有同名同路径的匹配文件集：

| 文件类型 | JS 项目 | TS 项目 | 必需               |
| -------- | ------- | ------- | ------------------ |
| 逻辑     | `.js`   | `.ts`   | 是                 |
| 结构     | `.wxml` | `.wxml` | 是                 |
| 样式     | `.wxss` | `.wxss` | 推荐               |
| 配置     | `.json` | `.json` | 页面推荐，组件必需 |

### TypeScript 项目识别

| 判断依据                                                        | 说明                       |
| --------------------------------------------------------------- | -------------------------- |
| 入口文件为 `app.ts`                                             | 项目使用 TypeScript 编写   |
| 存在 `tsconfig.json`                                            | TypeScript 编译配置        |
| `project.config.json` 中有 `useCompilerPlugins: ["typescript"]` | 开发者工具启用 TS 编译插件 |

三者必须一致——仅有 `app.ts` 但缺少编译插件配置是不完整的。

## 强制规则

### `app.json.pages` 必须与实际页面目录一一对应

```json
{
  "pages": ["pages/index/index", "pages/logs/logs"]
}
```

`pages` 数组中的每个路径必须对应真实存在的页面文件集。多余或缺失的路径都是错误。

### 自定义组件的 `.json` 必须声明 `"component": true`

```json
{
  "component": true
}
```

缺少此声明，框架不会将其识别为自定义组件。

### TypeScript 项目必须显式配置编译插件

```json
// project.config.json
{
  "setting": {
    "useCompilerPlugins": ["typescript"]
  }
}
```

仅编写 `.ts` 文件但未配置编译插件，开发者工具无法正确编译。同名 `.ts` 和 `.js` 文件并存时需要特别注意优先级。

### `miniprogramRoot` 必须指向包含 app 入口文件的目录

```json
// project.config.json
{
  "miniprogramRoot": "miniprogram/"
}
```

当省略时默认为仓库根目录。如果仓库根目录不包含 `app.js`/`app.ts` + `app.json`，必须显式设置。

## 验证清单

| #   | 检查项                     | 通过条件                                                       |
| --- | -------------------------- | -------------------------------------------------------------- |
| 1   | `project.config.json` 存在 | 文件存在于仓库根目录                                           |
| 2   | `miniprogramRoot` 指向正确 | 目标目录包含 app 入口文件                                      |
| 3   | app 入口文件完整           | JS: `app.js` + `app.json`；TS: `app.ts` + `app.json`           |
| 4   | `app.json.pages` 路径匹配  | 每个路径对应真实存在的页面文件集                               |
| 5   | 页面文件集完整             | 每个页面有匹配的逻辑 + 结构文件                                |
| 6   | 组件文件集完整             | 每个组件有匹配的逻辑 + 结构 + 配置（含 `component: true`）文件 |
| 7   | TS 编译配置一致            | 使用 `.ts` 时，`useCompilerPlugins` 包含 `"typescript"`        |
| 8   | 无同名 `.ts`/`.js` 冲突    | 同一目录下不存在同名的 `.ts` 和 `.js` 文件                     |

## 输出格式

验证完成后，按以下结构报告：

1. **符合规范的部分** — 已通过的检查项
2. **缺失或有风险的部分** — 未通过的检查项及具体问题
3. **推荐的结构决策** — 针对当前项目形态的最小调整建议
4. **第一步修改** — 最高优先级的单个修改操作
