# 乐享搜索与浏览

> **基础知识**：数据模型、URL 规则、工具发现方式见 `references/base.md`。
> **前置条件**：本 skill 需要已配置乐享 MCP 连接。如未配置，请先读取 `references/setup.md`。
> **遇到 401 错误**：不要重试，读取 `references/setup.md` 引导用户续期（点击续期按钮即可恢复，无需重新配置）。

---

## 工具概览

### 🔍 搜索与发现
- `search_kb_search` — 实体召回（可搜索 space、entry、team）
- `search_kb_embedding_search` — RAG 切片召回（语义检索 + 重排，返回文档切片，用于回答用户问题）
- `search_kb_chunk_search` — 切片检索（按关键词检索文档切片，用于回答用户问题）
- `lexiang_search` — 综合搜索（doc/space/team/all，**只返回条目列表，不含内容切片**）
- `team_list_teams` — 获取团队列表
- `team_describe_team` — 获取团队详情
- `team_list_frequent_teams` — 获取常用团队列表
- `space_list_spaces` — 获取知识库列表
- `space_describe_space` — 获取知识库详情（返回 `root_entry_id`）
- `space_list_recently_spaces` — 获取最近访问知识库

### 📖 条目与结构浏览
- `entry_list_children` — 浏览目录结构
- `entry_describe_entry` — 获取条目元信息（不含正文）
- `entry_describe_ai_parse_content` — 获取 AI 解析内容（含正文；**所有条目类型均适用**，含 PDF/PPT/Word/图片等文件附件）
- `entry_list_parents` — 获取父级路径（面包屑）
- `entry_list_latest_entries` — 获取最近更新条目
- `entry_list_recently_entries` — 获取当前用户最近浏览的条目
- `lexiang_fetch` — 按资源类型读取单个资源（entry/space/team/file/block）

### 💬 评论查看
- `comment_list_comments` — 获取条目评论列表
- `comment_describe_comment` — 获取评论详情

---

## 内容搜索

### 实体召回 vs RAG 切片召回

| 工具 | 定位 | 适用场景 | 参数 |
|------|------|----------|------|
| `search_kb_search` | 实体召回 | 查找具体的知识库(space)、文档(entry)、团队(team) | `keyword="xxx"` |
| `search_kb_embedding_search` | RAG 切片召回 | 基于语义检索 + 重排，返回与问题最相关的文档切片，直接用于回答用户问题 | `filters={"keyword": "xxx"}` |
| `search_kb_chunk_search` | 切片检索 | 按关键词检索文档切片，直接用于回答用户问题 | 以 `get_tool_schema` 返回为准 |
| `lexiang_search` | 实体/条目召回 | 快速搜索 doc/space/team/all，**只返回条目列表，不含正文与切片** | `query="xxx", type="all"` |

>⚠️ **参数注意**：`search_kb_embedding_search` 的关键词参数为 `filters.keyword`（嵌套字段），**不是**顶层 `keyword`。

**使用策略**：
- 用户想**了解某个问题的答案** → 用 `search_kb_embedding_search` / `search_kb_chunk_search` 获取切片直接回答
- 用户想**找某个文档/知识库/团队** → 用 `search_kb_search` 或 `lexiang_search` 召回条目
- 如需查看完整文档，再用 `entry_describe_ai_parse_content` 精确读取

> 🚨 **入口选择提醒**：`lexiang_search` 返回的是**条目列表**而非内容切片。
> 不要因为它调用省事（代理模式下只需一跳）就用它回答内容类问题——那样拿不到切片，
> 后续只能整篇解析（`entry_describe_ai_parse_content` / `file_download_file`），效率低且浪费上下文。

> 💡 **调用方式**：`lexiang_search` 为元工具，可直接调用；`search_kb_*` 为业务工具，
> 若当前只有元工具（代理模式）需经 `call_tool(tool_name=...)` 代理调用（见 `references/base.md`）。

###搜索结果链接格式

链接拼接规则见 `references/base.md`（`{domain}` 来源、三级域名判断、`company_from` 获取优先级）。

根据返回的 `target_type` 拼接路径部分（搜索独有）：

| target_type | 路径格式 |
|-------------|----------|
| `kb_page` | `/pages/<target_id>` |
| `kb_file` / `kb_video` | `/teams/<team_id>/docs/<target_id>` |

---

## 📖 内容读取

| 工具 | 返回内容 | 用途 |
|------|----------|------|
| `entry_describe_entry` | 条目元信息（ID、名称、类型等） | 获取基本信息 |
| `entry_describe_ai_parse_content` | **条目 AI 解析内容**（在线文档为正文；文件附件为 OCR/解析文本，PPT 逐页带 page-number 标记与图片描述） | 读取实际内容进行分析，**所有条目类型均适用** |
| `lexiang_fetch` | 任意资源详情 | 快捷读取单个 entry/space/team/file/block（entry 不含正文） |

> 🚨 **文件内容读取路径**：分析/引用文件附件（PDF/PPT/Word/图片等）内容时，
> **直接用 `entry_describe_ai_parse_content(entry_id=<文件条目ID>)` 获取解析文本**，
> **禁止**先 `file_download_file` 下载再本地解析。下载仅用于需要原始二进制的场景
> （本地再编辑、格式转换、版式还原），见 `references/files.md`。

---

## 💬 查看评论

当用户要查看某个页面的评论时：

```
Step 1: comment_list_comments(target_type="kb_entry", target_id=<entry_id>)
  → 返回评论列表

Step 2（可选）: comment_describe_comment(target_type="kb_entry", target_id=<entry_id>)
  → 返回评论详情（content为特殊格式，注意解析）
```

>⚠️ 评论的 `content` 字段不是普通 HTML，需要特别注意解析方式。

---

## ⚠️ 核心注意事项

1. **`_mcp_fields` 优化**：所有工具支持 `_mcp_fields` 参数选择返回字段，减少 token 消耗
2. 参数不确定时用 `get_tool_schema(tool_name="xxx")` 获取定义；若当前只暴露元工具（代理模式），业务工具需经 `call_tool(tool_name=...)` 代理调用（见 `references/base.md`）

---

## 参考文档

| 文档 | 说明 |
|------|------|
| `references/common-errors.md` | 常见错误排查 |
| `references/base.md` | 数据模型、URL 规则、工具发现方式 |
| `references/comment.md` | 评论功能详细说明 |
