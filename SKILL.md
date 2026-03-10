---
name: lexiang-knowledge-base
description: 用于乐享知识库相关任务，当用户明确提及「乐享」或 lexiang、提供 lexiangla.com 链接，或提出在乐享中搜索、阅读、创建、更新、整理、上传内容等操作需求时使用。
---

# 乐享知识库 Skill

## MCP 接入

依赖乐享 MCP 服务：
- 服务地址：`https://mcp.lexiang-app.com/mcp`
- 推荐模式：`preset=meta`
- 配置文件：`mcp.json`
- 初始化脚本：`setup.sh`

**调用通道**：优先使用当前环境已提供的原生乐享 MCP；若原生不可用，则通过 MCPorter 调用。

**核心原则**：以 MCP 服务返回的最新 schema 为准，按需读取 reference。若当前环境只显示元工具，视为 `preset=meta` 的正常行为。

## Meta 模式工作流

标准执行顺序：

1. 判断任务类型：搜索、读取、写入、结构整理、附件上传
2. 如果任务目标不清楚，先澄清平台、对象和动作（见"任务识别规则"）
3. 根据用户意图，调用合适的工具。如果有需要，则可以通过 `search_tools` 或 `list_tool_categories` 进行业务工具的查找。
4. 使用 `get_tool_schema` 获取目标工具的最新参数定义
5. 使用 `call_tool(tool_name, arguments)` 执行业务工具
6. 使用 `_mcp_fields` 收窄返回字段，减少噪音

**约束**：先发现工具 → 读 schema → 执行。不硬编码参数表，不混淆元工具和业务工具。

**脚本使用**：仅在文件上传、目录同步或程序化构造复杂 Block 参数时使用脚本，其他任务优先直接调用 MCP 工具。

## 任务识别与权限规则

### 目标澄清优先级

优先确认三要素：
- **平台**：是否在乐享操作（避免因"知识库""文档"等泛词误触发）
- **对象**：团队、知识库、页面、文件（要求链接、ID 或精确名称）
- **动作**：搜索、读取、创建、更新、删除、上传

### 写入操作安全规则

**写入操作包括**：创建页面/文件夹、导入内容、更新 Block、删除/移动、上传附件。

**允许写入的条件**（满足其一）：
- 用户提供明确 URL
- 用户提供明确 ID
- 用户提供精确名称，且查询后由用户确认目标

**严格禁止**：
- 遍历后自行挑选目标写入
- 根据名称"看起来合适"直接决定
- 目标不明确时执行创建、覆盖、删除、移动

**目标不明确时**：说明缺少目标，要求用户提供链接、ID 或精确名称。

**读取操作**：不受上述限制，可先搜索、浏览和查看。

## 核心数据模型

### 最小概念

| 概念 | 说明 | 层级关系 |
|------|------|----------|
| `team` | 顶级组织单元 | `team → space → entry (page/folder/file)` |
| `space` | 知识库，包含 `root_entry_id` | |
| `entry` | 页面、文件夹或文件 | |

### URL 解析

从用户提供的乐享链接提取 ID（忽略查询参数）：

| URL 形态 | 提取字段 |
|----------|----------|
| `.../spaces/{space_id}` | `space_id` |
| `.../pages/{entry_id}` | `entry_id` |
| `.../t/{team_id}/spaces` | `team_id` |

### 根节点定位流程

用户给出知识库链接需要根目录 ID 时：
1. 从 URL 提取 `space_id`
2. 调用 `space_describe_space` 获取 `root_entry_id`
3. 将 `root_entry_id` 作为根目录入口

## 常见任务执行流程

### 1. 搜索和定位

**场景**：找内容、找页面、找最近更新。

**工具选择**：
- **精确查找**（查找指定团队、知识库、条目）→ `search_kb_search`
- **语义检索**（RAG 召回内容切片）→ `search_kb_embedding_search`

**使用场景**：
- 查找指定团队、知识库、条目，用 `search_kb_search` 定位。
  - 查最近创建、最近更新的文档
  - 查指定用户创建的文档
  - 查找某个知识库
- 在知识库中检索相关内容，以回答用户问题，使用`search_kb_embedding_search`。

### 2. 从链接读取

**场景**：用户提供 `lexiangla.com` 链接。

**流程**：
1. 按上述 URL 解析规则提取 ID
2. 若是 `space_id` 且需根节点 → 按"根节点定位流程"处理
3. 若是 `entry_id` → 直接读取条目

**读取工具**：
- **阅读理解**（默认）→ `entry_describe_ai_parse_content`
- **浏览子条目** → `entry_list_children`
- **编辑前结构检查**（仅在需要更新 Block 时）→ `block_list_block_children` / `block_describe_block`

### 3. 写入内容（导入页面、上传文件、同步目录）

**场景**：
- 导入 Markdown 为可在线协同编辑的页面
- 原样上传 Markdown 文件或其他附件
- 同步本地目录到知识库
- 导入公众号内容

**流程**：
- 先校验写入目标是否明确（见"任务识别规则"）
- Markdown 转换为页面 → `entry_import_content`
- 单文件原样上传或更新版本 → `file_apply_upload` → HTTP PUT → `file_commit_upload`
- 批量上传优先使用 `scripts/upload-files.py`
- 本地目录增量同步时，读取 `references/folder-sync.md` 并使用 `scripts/sync-folder.ts`
- 更新已有文件时，先用 `entry_describe_entry` 读取 `target_id` 作为 `file_id`
- 导入公众号内容为在线页面 → `file_create_hyperlink`


### 4. 页面编辑与结构操作

**场景**：更新已有页面内容、移动重组结构、创建复杂在线文档结构。

**流程**：优先确认目标页面 ID，然后读取 `references/page-edit.md`。页面编辑相关的具体分流、结构检查、骨架示例和辅助工具都由该文档统一说明。

## References 导航

按需读取：
- 页面编辑、结构调整、复杂页面创建 → `references/page-edit.md`
- 本地目录增量同步 → `references/folder-sync.md`
- 调用报错或程序化参数排错 → `references/common-errors.md`
