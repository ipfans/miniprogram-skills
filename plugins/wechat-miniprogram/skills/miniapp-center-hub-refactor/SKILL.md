---
name: miniapp-center-hub-refactor
description: 将分散的小程序顶层 Tab（首页、个人、消息、集成、设置等）重组为 hub 中心结构。Use when top-level navigation has scattered into too many tabs, high-frequency actions and low-frequency settings share the same page, pending work has no clear home, or multiple tabs link to the same destination because ownership is unclear.
tools:
  - Bash
  - Read
  - Edit
  - Glob
  - Grep
  - Write
slash_command: /miniapp-center-hub-refactor
---

# 小程序中心 Hub 重构

将分散的顶层 Tab 按**归属权**重组为 hub 结构——hub 聚合摘要与下一步操作，详情页保留需要独立表单、校验或多步交互的流程。

## 文档索引

根据需求快速定位（路径相对于 `references/`）：

| 我想要...                      | 查阅文件                      |
| ------------------------------ | ----------------------------- |
| 了解审计清单、分区规则和迁移步骤 | `center-hub-playbook.md`      |
| 查看触发与非触发 prompt 示例    | `example-prompts.md`          |

## 重构流程

1. 读取 `references/center-hub-playbook.md`。
2. **审计现有导航**：列出所有顶层 Tab、重复入口和详情页，填写审计表（入口 → 用户意图 → 使用频率 → 是否需要详情页 → hub 内归属分区）。完成标准：每个可达目标页均出现在审计表中。
3. **分离频率**：将高频操作流（待办、审核）与低频设置流（偏好、集成管理）标记为不同分区。完成标准：审计表中每行已标注"高频"或"低频"。
4. **设计 hub 结构**：确定 hub 名称及内部分区，每个分区拥有稳定归属（如 `待办`、`设置`、`状态`）。完成标准：分区列表已写出，且无"其他"或"杂项"兜底分区。
5. **编写迁移映射**：将每个旧入口映射到：保留顶层 / 移入 hub 摘要 / 保留为子详情页。完成标准：审计表中所有入口均已映射，无遗漏。
6. **执行迁移**：按 `references/center-hub-playbook.md` 的迁移清单逐项操作，同步更新导航配置、路由文档和冒烟检查。完成标准：旧入口均可通过新路径到达，旧详情页仍可访问。

## 核心规则

### 从用户问题出发，不从文件树出发

设计 hub 分区时回答两个问题：
- 现在需要处理什么
- 在哪里管理支撑性设置

### 归属权决定页面位置

按用户意图的归属分配页面，而非按旧页面的链接关系或组件复用。

### hub 聚合摘要，详情页保留流程

hub 展示摘要行（待办数、未读数、连接状态、同步模式）；当流程拥有独立表单状态、校验、筛选或多步交互时，保留为详情子页面。

### 分区必须有明确边界

给 hub 内部分区或 Tab 赋予显式归属，禁止把高频处理和低频设置混在一个无分隔的长滚动页面中。

### 迁移映射先于删除

在隐藏或删除旧导航前，先完成迁移映射并确认新入口可达。旧详情页保留到新 hub 导航验证通过。

## 输出格式

1. 当前导航问题
2. 提议的 hub 结构
3. 页面归属与迁移映射
4. 迁移顺序
