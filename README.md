# miniprogram-skills

微信小程序开发 skills，适用于 **Claude Code** 和 **Codex**，或者其他的 AI Coding Agent。

## 安装

### Claude Code

```bash
# 添加 marketplace
/plugin marketplace add ipfans/miniprogram-skills

# 安装插件
/plugin install wechatide-skill@miniprogram-skills
/plugin install wechat-miniprogram@miniprogram-skills
```

### Codex

```bash
# 添加 marketplace
codex plugin marketplace add ipfans/miniprogram-skills

# 安装插件
codex plugin install wechatide-skill
codex plugin install wechat-miniprogram
```

或在 ChatGPT 桌面端 → Plugins → 选择 miniprogram-skills marketplace → 安装。

## 插件列表

| 插件 | 版本 | 说明 |
|---|---|---|
| **wechatide-skill** | 0.3.6 | 微信开发者工具工作流：小程序/小游戏的创建与导入、编译预览上传、登录与项目管理、页面自动化、调试取证、云开发，以及开发者工具的下载安装更新 |
| **wechat-miniprogram** | 1.0.0 | 微信小程序开发框架 — WXML/WXSS/WXS 模板、组件、API、云开发，以及 Skyline 渲染引擎 |

## 添加新插件

1. 在 `plugins/` 下创建插件目录：

```
plugins/my-plugin/
├── .claude-plugin/
│   └── plugin.json          # Claude Code 清单
├── .codex-plugin/
│   └── plugin.json          # Codex 清单（含 interface 对象）
├── skills/
│   └── my-skill/
│       └── SKILL.md
├── SKILL.md                 # 根 skill 入口（可选）
└── skill.yaml               # 版本元数据（可选）
```

2. 在两个 marketplace 文件中注册：

   - `.claude-plugin/marketplace.json`（Claude Code）— 使用 `"source": "./plugins/my-plugin"` 字符串格式
   - `.agents/plugins/marketplace.json`（Codex）— 使用 `"source": { "source": "local", "path": "./plugins/my-plugin" }` 对象格式，并包含 `policy` 和 `category`

## License

MIT
