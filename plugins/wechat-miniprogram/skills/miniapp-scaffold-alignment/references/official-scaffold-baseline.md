# 官方项目结构基线

验证小程序项目结构时，以本文件作为基线参照。

## 官方规范来源

- 目录结构
- 全局配置与页面配置
- 代码构成
- 自定义组件
- TypeScript 支持
- 项目配置文件

## 基线规则

### 1. App 入口文件

小程序代码根目录必须包含 app 入口文件：

**JavaScript 项目：**
```
app.js      # 应用逻辑（必需）
app.json    # 全局配置（必需）
app.wxss    # 全局样式（可选）
```

**TypeScript 项目：**
```
app.ts      # 应用逻辑（必需）
app.json    # 全局配置（必需）
app.wxss    # 全局样式（可选）
```

TypeScript 项目编写 `app.ts`，但运行时仍依赖开发者工具的编译插件将其转为 `app.js`。

### 2. 页面文件集

每个页面由同名同路径的文件集组成：

**JavaScript 项目：**

| 文件 | 用途 | 必需 |
|------|------|------|
| `pageName.js` | 页面逻辑（Page 生命周期） | 是 |
| `pageName.wxml` | 页面结构（WXML 模板） | 是 |
| `pageName.wxss` | 页面样式 | 推荐 |
| `pageName.json` | 页面配置 | 推荐 |

**TypeScript 项目：**

| 文件 | 用途 | 必需 |
|------|------|------|
| `pageName.ts` | 页面逻辑（Page 生命周期） | 是 |
| `pageName.wxml` | 页面结构（WXML 模板） | 是 |
| `pageName.wxss` | 页面样式 | 推荐 |
| `pageName.json` | 页面配置 | 推荐 |

`app.json` 的 `pages` 数组必须列出所有实际存在的页面路径。路径格式为 `"pages/index/index"`（不含扩展名）。

### 3. 组件文件集

自定义组件同样由同名文件集组成：

**JavaScript 项目：**

| 文件 | 用途 | 必需 |
|------|------|------|
| `componentName.js` | 组件逻辑（Component 构造器） | 是 |
| `componentName.wxml` | 组件结构 | 是 |
| `componentName.wxss` | 组件样式 | 推荐 |
| `componentName.json` | 组件配置（必须含 `"component": true`） | 是 |

**TypeScript 项目：**

| 文件 | 用途 | 必需 |
|------|------|------|
| `componentName.ts` | 组件逻辑（Component 构造器） | 是 |
| `componentName.wxml` | 组件结构 | 是 |
| `componentName.wxss` | 组件样式 | 推荐 |
| `componentName.json` | 组件配置（必须含 `"component": true`） | 是 |

组件通过页面或全局 `usingComponents` 注册后使用。

### 4. project.config.json 职责

| 字段 | 说明 |
|------|------|
| `miniprogramRoot` | 指定小程序代码根目录（相对于仓库根目录） |
| `appid` | 小程序 AppID |
| `setting` | 开发者工具的编译和调试设置 |
| `compileType` | 编译类型（`miniprogram` 或 `plugin`） |

当仓库根目录同时包含非小程序代码（docs/、后端、工具）时，`miniprogramRoot` 决定开发者工具在哪里寻找 app 入口文件。省略时默认为仓库根目录。

### 5. TypeScript 约束

TypeScript 是编写便利，不是小程序的底层格式：

- 项目使用 `.ts` 编写时，`project.config.json` 必须配置：
  ```json
  {
    "setting": {
      "useCompilerPlugins": ["typescript"]
    }
  }
  ```
- 同一目录下不应同时存在同名的 `.ts` 和 `.js` 文件，否则可能出现编译优先级歧义
- `tsconfig.json` 应存在于小程序代码根目录，配置基础的编译选项

### 6. 典型目录结构

**JavaScript 项目：**
```
├── project.config.json
├── app.js
├── app.json
├── app.wxss
├── pages/
│   ├── index/
│   │   ├── index.js
│   │   ├── index.wxml
│   │   ├── index.wxss
│   │   └── index.json
│   └── logs/
│       ├── logs.js
│       ├── logs.wxml
│       ├── logs.wxss
│       └── logs.json
├── components/
│   └── my-component/
│       ├── my-component.js
│       ├── my-component.wxml
│       ├── my-component.wxss
│       └── my-component.json
└── utils/
    └── util.js
```

**TypeScript 项目：**
```
├── project.config.json
├── tsconfig.json
├── typings/               # 类型声明文件
│   └── index.d.ts
├── app.ts
├── app.json
├── app.wxss
├── pages/
│   ├── index/
│   │   ├── index.ts
│   │   ├── index.wxml
│   │   ├── index.wxss
│   │   └── index.json
│   └── logs/
│       ├── logs.ts
│       ├── logs.wxml
│       ├── logs.wxss
│       └── logs.json
├── components/
│   └── my-component/
│       ├── my-component.ts
│       ├── my-component.wxml
│       ├── my-component.wxss
│       └── my-component.json
└── utils/
    └── util.ts
```

**含 miniprogramRoot 的混合仓库：**
```
├── project.config.json    # miniprogramRoot: "miniprogram/"
├── docs/
├── server/
├── scripts/
└── miniprogram/
    ├── app.js (或 app.ts)
    ├── app.json
    ├── app.wxss
    ├── pages/
    └── components/
```

## 实际判断要点

- 提到 `app.ts` 但未说明 TypeScript 编译配置的规格是不完整的
- 列出页面目录但未说明匹配文件集的规格是不完整的
- 包含兄弟目录的仓库通常需要仓库根目录的 `project.config.json` 加上专用的 `miniprogramRoot`
- 存在 `tsconfig.json` 但没有 `useCompilerPlugins` 配置的 TypeScript 项目是不完整的
