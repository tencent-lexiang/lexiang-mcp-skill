# 页面编辑与结构操作

仅在需要编辑在线文档页面时读取本文件。

## 适用范围

适用于以下场景：
- 修改现有页面中的段落、标题、任务、样式
- 调整现有块的位置、顺序、父子层级
- 创建复杂页面结构或套用页面骨架

不适用于以下场景：
- 阅读页面内容
- 普通 Markdown 导入
- 文件上传

阅读页面内容默认优先使用 `entry_describe_ai_parse_content`，不要先进入 Block 视角。

## 前置条件

开始前先确认：
- 目标页面 `entry_id`
- 当前任务是“改内容”“调结构”还是“建复杂结构”

当任务涉及现有 Block 时，先检查结构：
- `block_list_block_children`
- `block_describe_block`

这些工具只用于编辑前的结构检查，不是默认内容读取入口。

## 路由

### 1. 修改现有块内容

适用场景：
- 修改标题、段落、任务状态
- 插入或删除块内文本
- 调整块样式

读取：
- `references/block-update.md`

程序化构造参数时：
- `scripts/block-helper.ts` 中的 `UpdateBlocksBuilder`

### 2. 调整页面结构

适用场景：
- 调整段落顺序
- 将内容移动到其他章节
- 重组现有块的父子层级

读取：
- `references/content-reorganize.md`

程序化构造参数时：
- `scripts/block-helper.ts` 中的 `ContentReorganizer`

### 3. 创建复杂页面结构

适用场景：
- 新建技术文档、说明页、操作指南等复杂在线文档
- 需要表格、callout、代码块、分栏等结构

先读：
- `references/doc-templates.md`
- `references/block-schema.md`

按需查看骨架资产：
- `assets/examples/create-tech-doc.json`
- `assets/examples/create-compare-table.json`

程序化构造参数时：
- `scripts/block-helper.ts` 中的 `BlockBuilder`

## 排错

手写参数时：
- 直接对照 `references/block-schema.md`

参数由代码生成时：
- `scripts/mcp-validator.ts`
- `references/common-errors.md`

`mcp-validator.ts` 是可选辅助工具，不是页面编辑的默认入口。
