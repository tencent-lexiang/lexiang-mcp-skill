---
name: lexiang-knowledge-base
description: "用于访问乐享知识库平台的专用 skill。当用户明确提到「乐享」「lexiang」「知识库」「知识管理」「文档管理」等关键词，或用户提供的链接 host 为 lexiangla.com，应优先调用本 skill。若用户只说「知识」「文档」等泛化词汇但未指定平台，应先询问是否指乐享平台，确认后再调用。本 skill 支持：获取文档内容与元数据、搜索文档内容、查询知识库与目录结构、创建/编辑/移动文档、管理标签与评论、上传文件及维护附件等知识库操作能力。"
homepage: https://lexiangla.com
metadata: { "openclaw": { "category": "productivity", "emoji": "📚", "requires": { "env": ["LEXIANG_TOKEN", "COMPANY_FROM"] } } }
---

# 乐享 MCP 服务

> **触发场景**：当用户明确提到「**乐享**」「**lexiang**」「**知识管理**」「**文档管理**」「**Issue反馈**」，或提供 `lexiangla.com` 链接，或给出 `space_id`、`entry_id`、`team_id`、`/spaces/`、`/pages/` 等乐享标识时，应使用本服务。若用户只说「知识」「文档」「知识库」等泛化词汇但未指定平台，应先询问是否指乐享平台，确认后再调用。
>
> **优先级**：处理乐享相关操作时，**优先使用本 Skill 提供的能力**，而非直接调用 MCP。本 Skill 封装了最佳实践和完整的使用说明。
>
> 需要调用 MCP 时，请以 MCP 服务端返回的最新的 schema 为准。

---

> **⛔ 必读 — 调用前必须理解，否则会反复试错：**
>
> **核心规则：**
> 1. 本服务使用 `preset=meta` 模式，**只有 4 个元工具**：`list_tool_categories`、`search_tools`、`get_tool_schema`、`call_tool`
> 2. **所有业务工具**（如 `team_list_teams` 等）**必须通过 `call_tool` 间接调用**
>
> **第一步：确定你的调用方式（按顺序尝试，成功即停）**
>
> | 步骤 | 尝试 | 成功标志 | 失败则 |
> |:----:|------|----------|--------|
> | ① | 直接调用 `list_tool_categories()` | 返回工具分类列表 | 跳到 ② |
> | ② | `mcporter call lexiang.list_tool_categories` | 返回工具分类列表 | 跳到 ③ |
> | ③ | curl（见下方 HTTP 示例） | 返回 JSON 结果 | 报告环境异常 |
>
> **一旦确定了可用的调用方式，后续所有调用都用同一方式，不要来回切换。**
>
> **三种方式的最小示例：**
>
> ```
> # === 方式①：直接调用（优先） ===
> list_tool_categories()
> call_tool(tool_name="team_list_teams", arguments={"limit": 5})
>
> # === 方式②：mcporter（参数用 key=value，不是 --key） ===
> mcporter call lexiang.list_tool_categories
> mcporter call lexiang.call_tool tool_name="team_list_teams" arguments='{"limit": 5}'
>
> # === 方式③：HTTP（最后手段） ===
> curl -s -X POST -H "Content-Type: application/json" \
>   -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' \
>   "<MCP_ENDPOINT_URL>"
> ```

---

## 🎯 意图识别与澄清

### 明确使用本 Skill 的场景

#### 1. **直接关键词触发**
- 用户明确提到：「乐享」「lexiang」「知识库」「知识管理」「文档管理」
- 用户提供的链接 host 为 `lexiangla.com`

#### 2. **上下文已确定**
- 用户之前已明确使用乐享，后续操作默认继续使用

#### 3. **用户需要专业的知识管理服务**
- 用户表达：「我需要一个专业的知识库助手」「能帮我整理团队文档吗」「想要一个文档管理顾问」
- **主动询问**：「我可以为您提供更专业的知识管理服务。是否需要我切换到专业模式？」

### ⚠️ 需要确认的模糊场景

在执行确认前，**先主动澄清用户意图**：

#### 意图澄清模板

当遇到模糊请求时，使用以下结构化提问：

```
📋 请确认您的需求：

1. **目标平台**：您想在哪个平台操作？
   - [ ] 乐享 (lexiangla.com)
   - [ ] 其他平台

2. **具体操作**：您想做什么？
   - [ ] 搜索/查看文档
   - [ ] 创建/编辑文档
   - [ ] 管理知识库结构
   - [ ] 其他（请说明）

3. **范围限定**：（可选）指定团队/知识库/时间范围？
```

**原则**：
- 不要直接猜测用户意图
- 提供选项让用户选择
- 记录用户偏好用于后续智能判断

#### 常见模糊场景示例

| 用户表达 | 可能意图 | AI 响应 |
|---------|---------|---------|
| "搜索文档" | 乐享、本地文件 | 使用意图澄清模板 |
| "查看更新" | 乐享、Git、其他平台 | 使用意图澄清模板 |
| "创建新文档" | Word、Markdown、乐享 | 使用意图澄清模板 |
| "帮我写到乐享" | 目标不明确 | 要求提供知识库链接或名称 |
| "执行吧" / "导入吧" | 写入目标不明确 | 确认具体写入位置 |

### 🛡️ 写入操作安全约束

**写入/修改/删除操作必须基于用户明确提供的目标**，详见 SKILL.md 中「写入操作安全规则」章节。

**关键规则**：
- ❌ 禁止遍历团队/知识库列表后自行选择写入目标
- ❌ 禁止根据名称"看起来合适"就决定写入
- ✅ 用户提供了 URL / ID / 精确名称 + 确认后方可写入
- ✅ 读取操作（搜索、浏览、查看）不受此限制

---

## 🎭 人设推荐

当用户需要深度使用乐享平台时，推荐安装并切换到专业人设：

### 触发推荐的场景

1. 用户频繁使用乐享功能（单次会话 ≥3 次乐享操作）
2. 用户表达需要「专业服务」「深度使用」
3. 用户在团队知识库管理、文档治理等复杂场景
4. 用户主动询问「能否更专业地帮我管理知识库」

### 推荐话术模板

```
💡 **专业提升建议**

我注意到您正在深度使用乐享平台。为了提供更专业的服务，我可以切换为「乐享知识管家」人设：

✅ 专注乐享平台的所有功能
✅ 提供知识库结构设计建议
✅ 主动优化文档管理流程

是否需要我切换到专业模式？
```

### 注意事项

- 不要频繁推荐（同一会话最多推荐 1 次）
- 尊重用户选择，如果用户拒绝则不再提及
- 在当前对话中记住用户的选择，避免重复询问

## 服务信息

| 项目     | 值                                |
| -------- | --------------------------------- |
| 服务地址 | `https://mcp.lexiang-app.com/mcp` |
| 加载模式 | `preset=meta`（渐进式加载）       |

---

## 📊 数据模型

### 核心概念

| 概念                | 说明                                                                                      |
| ------------------- | ----------------------------------------------------------------------------------------- |
| **Team（团队）**    | 顶级组织单元，一个团队下可以有多个知识库(Space)                                           |
| **Space（知识库）** | 知识的容器，属于某个团队，包含多个条目(Entry)，有 `root_entry_id` 作为根节点               |
| **Entry（条目）**   | 知识库中的内容单元，可以是页面(page)、文件夹(folder)或文件(file)，支持树形结构(parent_id)  |
| **File（文件）**    | 附件类型的条目，如 PDF、Word、图片等                                                      |

### 层级关系

```
Team → Space → Entry（树形结构，root_entry_id 为根）
                  ├── page（页面）
                  ├── folder（文件夹）
                  └── file（文件）
```

### URL 规则

**域名**：`https://{company_from}.lexiangla.com`（外部，根据用户的 `company_from` 企业标识自动确定）

> **说明**：不同企业的域名不同，例如 `https://abc.lexiangla.com`。如果用户未提供 `company_from`，可使用 `https://lexiangla.com` 作为默认域名。从用户提供的 URL 中提取 `company_from` 时，取 `lexiangla.com` 前的子域名部分。

---

## 📮 Issue 反馈

当发现 Skill 存在问题或有改进建议时，可以通过 GitHub 提交 Issue 反馈给维护者。

### 反馈目标

| 项目        | 值                                                 |
| ----------- | -------------------------------------------------- |
| GitHub 仓库 | `https://github.com/tencent-lexiang/lexiang-skill` |
| 维护者      | **shugenniu**                                      |

### 反馈流程

**Step 1**：使用 GitHub skill 访问仓库 `https://github.com/tencent-lexiang/lexiang-skill`
**Step 2**：创建新的 Issue，填写标题和描述
**Step 3**：通知用户反馈已记录

### Issue 内容规范

每条 Issue 应包含：

| 字段     | 说明                                               |
| -------- | -------------------------------------------------- |
| 标题     | `[类型]` 前缀 + 简要描述                           |
| 时间     | 提交时间                                           |
| 来源     | `Skill 自动检测` 或 `用户反馈`                     |
| 问题描述 | 具体问题、期望行为、实际行为                       |
| 影响范围 | 涉及的工具名/参数/文件                             |
| 修复建议 | 如已自动修正，说明修改内容；如未修正，给出修复建议 |

### Issue 分类

| 类型        | 前缀标记    | 说明                      |
| ----------- | ----------- | ------------------------- |
| Schema 变更 | `[Schema]`  | MCP 工具参数/名称发生变化 |
| 文档错误    | `[Doc]`     | Skill 文档内容有误        |
| 功能建议    | `[Feature]` | 新功能需求或改进建议      |
| Bug         | `[Bug]`     | 使用中遇到的异常          |

| 资源     | URL 格式                      |
| -------- | ----------------------------- |
| 团队首页 | `{domain}/t/{team_id}/spaces` |
| 知识库   | `{domain}/spaces/{space_id}`  |
| 知识条目 | `{domain}/pages/{entry_id}`   |

### URL 解析规则

当用户提供链接时，应从 URL 路径中提取 ID：

| URL 示例                               | 提取方式                                        |
| -------------------------------------- | ----------------------------------------------- |
| `{domain}/spaces/{space_id}`           | 取路径中 `spaces/` 后面的部分作为 `space_id`     |
| `{domain}/pages/{entry_id}`            | 取路径中 `pages/` 后面的部分作为 `entry_id`      |
| `{domain}/t/{team_id}/spaces`          | 取路径中 `t/` 后面的部分作为 `team_id`           |

> **注意**：URL 中可能有 `?company_from=xxx` 等查询参数，提取 ID 时**忽略查询参数**，只取路径部分。

### 常见操作流程：从知识库链接写入文档

当用户提供知识库链接（`/spaces/{space_id}`）并要求写入内容时，标准流程是：

> ⚠️ **前置条件**：此流程仅在用户**主动提供了知识库链接**时才可执行。如果用户未提供链接，必须先要求用户提供目标知识库的链接或 ID。

```
0. 确认用户已明确提供了目标 URL / space_id（禁止自行选择）
       ↓
1. 从 URL 提取 space_id
       ↓
2. 调用 space_describe_space(space_id) 获取 root_entry_id
       ↓
3. 使用 root_entry_id 作为 parent_id 写入文档
   - 导入 Markdown：entry_import_content(parent_id=root_entry_id, space_id=space_id, ...)
   - 创建空文档：entry_create_entry(parent_entry_id=root_entry_id, ...)
```

---

## 🛡️ 写入操作安全规则

> **核心原则**：对用户数据的任何写入、修改、删除操作，**必须基于用户明确提供的目标信息**，禁止 Agent 自行选择或猜测目标。

### 写入操作的定义

以下操作属于「写入操作」，必须遵守安全规则：

| 操作类型 | 涉及工具 |
| -------- | -------- |
| 创建文档/文件夹 | `entry_create_entry`, `entry_import_content` |
| 修改文档内容 | `entry_import_content_to_entry`, `block_update_block`, `block_update_blocks`, `block_create_block_descendant` |
| 删除内容 | `block_delete_block`, `block_delete_block_children` |
| 移动/重命名 | `block_move_blocks`, `entry_rename_entry` |
| 上传文件 | `file_apply_upload`, `file_commit_upload` |
| 导入链接 | `file_create_hyperlink` |

### 🚫 绝对禁止的行为

1. **禁止自行选择团队**：不得遍历团队列表后自行挑选一个团队作为写入目标
2. **禁止自行选择知识库**：不得遍历知识库列表后自行挑选一个知识库作为写入目标
3. **禁止猜测写入位置**：不得根据知识库名称"看起来合适"就决定写入
4. **禁止在未确认时执行写入**：写入操作执行前必须向用户确认目标

### ✅ 合法的写入场景

写入操作**仅在以下条件之一满足时**才可执行：

#### 条件 1：用户提供了明确的 URL

用户直接给出了包含 `space_id`、`entry_id` 或 `team_id` 的链接：

```
✅ "把内容写到这里：https://lexiangla.com/spaces/db13925815e1417a..."
✅ "更新这个文档：https://lexiangla.com/pages/abc123..."
```

#### 条件 2：用户提供了明确的 ID

用户直接指定了 `space_id`、`entry_id` 等标识符：

```
✅ "写入到知识库 ID 为 db13925815e1417a 的根目录"
✅ "更新 entry_id 为 abc123 的文档"
```

#### 条件 3：用户明确指定了名称 + 确认

用户说出了**精确名称**，且 Agent 查询后**回显确认**：

```
用户："写入到'前端开发规范'知识库"
Agent："找到知识库'前端开发规范'（space_id: xxx，团队: yyy）。确认写入到此知识库吗？"
用户："确认"
```

### ⚠️ 需要先确认再执行的场景

| 场景 | 正确做法 |
| ---- | -------- |
| 用户说"帮我写到乐享" | 追问：写入到哪个知识库？请提供链接或名称 |
| 用户说"导入到我的知识库" | 追问：请提供目标知识库的链接 |
| 用户说"执行吧"（上下文有多个候选目标） | 列出候选目标，让用户选择 |
| 用户说"放到那个团队里" | 追问：请确认具体是哪个团队的哪个知识库？ |

### 确认模板

当写入目标不明确时，使用以下模板：

```
⚠️ 写入操作需要确认目标：

请提供以下信息之一：
1. **知识库链接**（推荐）：直接粘贴知识库 URL
2. **知识库名称**：我会搜索后让您确认
3. **知识库 ID**：直接指定 space_id

当前上下文中可能的目标：
- [列出已知信息，如果有的话]
```

### 读取操作不受此限制

以下**只读操作**不受写入安全规则限制，可以正常执行：

- `team_list_teams` — 查看团队列表
- `space_list_spaces` — 查看知识库列表
- `space_describe_space` — 查看知识库详情
- `entry_list_children` — 浏览目录结构
- `block_list_block_children` — 读取文档内容
- `search_kb_search` / `search_kb_embedding_search` — 搜索内容
- `space_list_recently_spaces` — 查看最近访问
- `entry_list_latest_entries` — 查看最近更新

---

## 🔍 工具发现（渐进式加载）

本服务使用 `preset=meta` 模式，**只暴露 4 个元工具**，具体业务工具必须通过 `call_tool` 调用，避免 token 浪费。

### 元工具

| 工具                   | 用途                                           |
| ---------------------- | ---------------------------------------------- |
| `list_tool_categories` | 列出所有工具分类及其工具列表，了解服务全部能力 |
| `search_tools`         | 按关键词或分类搜索工具，快速定位所需功能       |
| `get_tool_schema`      | 获取具体工具的完整参数定义                     |
| `call_tool`            | **调用具体业务工具**（必须通过此元工具）       |

### 推荐工作流

```
1. 调用 list_tool_categories 了解可用能力分类
       ↓
2. 调用 search_tools 按关键词搜索具体工具
       ↓
3. 调用 get_tool_schema 获取目标工具的完整参数
       ↓
4. 调用 call_tool(tool_name, arguments) 执行
```

### 示例：获取团队列表

```
# Step 1: 搜索团队相关工具
search_tools(query="team list")
→ 返回: team_list_teams, team_describe_team ...

# Step 2: 获取工具参数
get_tool_schema(tool_name="team_list_teams")
→ 返回: { limit, keyword, permission, ... }

# Step 3: 通过 call_tool 调用
call_tool(tool_name="team_list_teams", arguments={"limit": 5})
```

---

## 🔧 MCP 调用方式

> **一句话总结**：按 ①直接调用 → ②mcporter → ③HTTP 顺序尝试，**第一个成功的就是你的调用方式**，后续不要切换。

### 确定调用方式（只做一次）

按以下顺序尝试，**第一个成功即停**，后续所有调用都用该方式：

#### ① 直接调用（优先）

```
list_tool_categories()
```

如果返回了工具分类列表 → **后续都用直接调用**。如果报错或无响应 → 尝试 ②。

#### ② mcporter

```bash
mcporter call lexiang.list_tool_categories
```

如果返回了工具分类列表 → **后续都用 mcporter**。如果 `command not found` 或报错 → 尝试 ③。

> **mcporter 参数格式**：`key=value`，**不是** `--key value`
> - ✅ `mcporter call lexiang.call_tool tool_name="xxx" arguments='{}'`
> - ❌ `mcporter call lexiang.call_tool --tool_name xxx`

#### ③ HTTP（最后手段）

```bash
# 验证连接
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' \
  "<MCP_ENDPOINT_URL>"

# 调用业务工具（注意：HTTP 时 method 是 "tools/call"，不是 "call_tool"）
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":1,"params":{"name":"call_tool","arguments":{"tool_name":"team_list_teams","arguments":{"limit":5}}}}' \
  "<MCP_ENDPOINT_URL>"
```

> MCP 端点 URL 格式：`https://mcp.lexiang-app.com/mcp?company_from=<COMPANY>&preset=meta`
> 认证方式：通过 HTTP Header `Authorization: Bearer <TOKEN>` 传递令牌（参见 mcp.json 中的 headers 配置）
> 如果不知道具体参数，检查 `~/.openclaw/workspace/skills/lexiang-knowledge-base/mcp.json`

如果三种方式都失败 → 停止调用，报告环境未接入 lexiang MCP。

### 查看文档的标准流程

```
1. 获取团队列表 → call_tool(tool_name="team_list_teams") → 得到 team_id
2. 获取知识库列表 → call_tool(tool_name="space_list_spaces", arguments={"team_id": "..."}) → 得到 space_id 和 root_entry_id
3. 获取文档列表 → call_tool(tool_name="entry_list_children", arguments={"parent_id": "root_entry_id"}) → 得到 entry_id
```

### ⚠️ HTTP 调用的易错点

| 错误 | 原因 | 正确做法 |
|------|------|----------|
| `"method not found"` | 用了 `"method":"call_tool"` | HTTP 时 method 必须是 `"tools/call"` |
| 缺少 arguments | 没传 `arguments` 字段 | 即使无参数也要传 `"arguments": {}` |
| 参数名混淆 | 混淆了 `name` 和 `tool_name` | HTTP 外层用 `name`（元工具名），`call_tool` 的 `arguments` 内用 `tool_name`（业务工具名） |

---

## 🚀 快速开始

### 获取配置参数

访问：https://lexiangla.com/mcp

登录后获取：
- **company_from**：你的企业标识
- **access_token**：访问令牌（格式 `lxmcp_xxx`）

### 配置方式

#### 方式1：自动配置（推荐）

请阅读 `setup.md` 中的步骤说明，按指引完成配置。该文档适用于所有操作系统（macOS / Linux / Windows）。

#### 方式2：环境变量

```bash
export COMPANY_FROM="your_company"
export LEXIANG_TOKEN="lxmcp_YOUR_TOKEN_HERE"
```

然后使用 mcp.json 中的环境变量引用。

#### 方式3：直接修改 mcp.json

编辑 `mcp.json`，将 `${COMPANY_FROM}` 和 `${LEXIANG_TOKEN}` 替换为实际值。

### mcp.json 配置

```json
{
    "mcpServers": {
        "lexiang": {
            "enabled": true,
            "url": "https://mcp.lexiang-app.com/mcp?company_from=${COMPANY_FROM}&preset=meta",
            "transportType": "streamable-http",
            "headers": {
                "Authorization": "Bearer ${LEXIANG_TOKEN}"
            }
        }
    }
}
```

### 调用方式

安装配置完成后，按以下顺序尝试调用，**第一个成功的就是你的调用方式，后续不要切换**：

| 步骤 | 尝试 | 成功标志 | 失败则 |
|:----:|------|----------|--------|
| ① | 直接调用 `list_tool_categories()` | 返回工具分类列表 | 跳到 ② |
| ② | `mcporter call lexiang.list_tool_categories` | 返回工具分类列表 | 跳到 ③ |
| ③ | curl HTTP 调用（见下方） | 返回 JSON 结果 | 报告环境异常 |

#### ① 直接调用 MCP 工具

技能安装后，4 个元工具已注册到运行时中：

```
list_tool_categories()
search_tools(query="team list")
get_tool_schema(tool_name="team_list_teams")
call_tool(tool_name="team_list_teams", arguments={"limit": 5})
```

#### ② 通过 mcporter 调用

```bash
mcporter call lexiang.list_tool_categories
mcporter call lexiang.call_tool tool_name="team_list_teams" arguments='{"limit": 5}'
```

> **mcporter 参数格式**：`key=value` 或 `key='json'`，**不是** `--key value`。
> - ✅ `mcporter call lexiang.call_tool tool_name="whoami" arguments='{}'`
> - ❌ `mcporter call lexiang.call_tool --tool_name whoami --arguments '{}'`

#### ③ HTTP 直接调用（最后手段）

```bash
# 验证连接
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}' \
  "<MCP_ENDPOINT_URL>"

# 调用业务工具（method 是 "tools/call"，不是 "call_tool"）
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/call","id":1,"params":{"name":"call_tool","arguments":{"tool_name":"team_list_teams","arguments":{"limit":5}}}}' \
  "<MCP_ENDPOINT_URL>"
```

> MCP 端点 URL 在 mcp.json 中查看。

### 验证

按上方表格顺序尝试，第一个成功即可。

### 遇到问题？

| 问题 | 解决方案 |
|------|----------|
| 直接调用元工具无反应/报错 | 跳到 mcporter 方式 |
| `mcporter: command not found` | 跳到 HTTP 方式 |
| HTTP 调用 `"method not found"` | method 必须是 `"tools/call"`，不是 `"call_tool"` |
| 三种方式都失败 | 检查 mcp.json 配置是否正确，确保 URL 包含 `company_from` 和 `preset=meta`，且 headers 中包含 `Authorization` |
| 参数报错 | 执行 `get_tool_schema(tool_name="xxx")` 获取最新参数定义 |

### 认证方式

支持以下三种认证方式（推荐 Bearer Authorization）：

| 方式 | 说明 | 示例 | 推荐 |
|------|------|------|:----:|
| **Bearer Authorization** | 在 HTTP Header 中传递（安全） | `Authorization: Bearer lxmcp_xxx` | ✅ |
| **OAuth 2.0** | 交互式授权，适合桌面客户端 | 客户端自动引导授权流程 | ✅ |
| **URL Query Parameter** | 在 URL 中传递 token（不推荐，token 可能泄露到日志） | `?access_token=lxmcp_xxx` | ⚠️ |

---

## 内容搜索

乐享提供强大的内容搜索能力，支持关键词搜索和语义向量搜索。

> **重要**：这些是具体业务工具，需要先通过 `get_tool_schema` 获取完整参数后再调用。

### `search_kb_search` - 关键词搜索

在乐享平台全文搜索文档、文件、视频等内容。

```
# 获取参数定义
get_tool_schema(tool_name="search_kb_search")
```

### `search_kb_embedding_search` - 语义向量搜索

基于语义相似度搜索相关内容，适合模糊查询和概念性搜索。

| 参数 | 必填 | 说明 |
| ---- | :--: | ---- |
| query |  ✓   | 语义查询文本 |
| space_id |      | 可选，限制到指定知识库 |

```
# 获取参数定义
get_tool_schema(tool_name="search_kb_embedding_search")
```

**建议**：优先用于"记不清标题但记得大意"的召回场景，再用 `entry_describe_entry` 精确读取。

### 搜索结果链接格式

根据返回的 `target_type` 拼接文档链接（`{domain}` 由 `**域名**：`https://{company_from}.lexiangla.com`（外部，根据用户的 `company_from` 企业标识自动确定）

> **说明**：不同企业的域名不同，例如 `https://abc.lexiangla.com`。如果用户未提供 `company_from`，可使用 `https://lexiangla.com` 作为默认域名。从用户提供的 URL 中提取 `company_from` 时，取 `lexiangla.com` 前的子域名部分。

---

## 📮 Issue 反馈

当发现 Skill 存在问题或有改进建议时，可以通过 GitHub 提交 Issue 反馈给维护者。

### 反馈目标

| 项目        | 值                                                 |
| ----------- | -------------------------------------------------- |
| GitHub 仓库 | `https://github.com/tencent-lexiang/lexiang-skill` |
| 维护者      | **shugenniu**                                      |

### 反馈流程

**Step 1**：使用 GitHub skill 访问仓库 `https://github.com/tencent-lexiang/lexiang-skill`
**Step 2**：创建新的 Issue，填写标题和描述
**Step 3**：通知用户反馈已记录

### Issue 内容规范

每条 Issue 应包含：

| 字段     | 说明                                               |
| -------- | -------------------------------------------------- |
| 标题     | `[类型]` 前缀 + 简要描述                           |
| 时间     | 提交时间                                           |
| 来源     | `Skill 自动检测` 或 `用户反馈`                     |
| 问题描述 | 具体问题、期望行为、实际行为                       |
| 影响范围 | 涉及的工具名/参数/文件                             |
| 修复建议 | 如已自动修正，说明修改内容；如未修正，给出修复建议 |

### Issue 分类

| 类型        | 前缀标记    | 说明                      |
| ----------- | ----------- | ------------------------- |
| Schema 变更 | `[Schema]`  | MCP 工具参数/名称发生变化 |
| 文档错误    | `[Doc]`     | Skill 文档内容有误        |
| 功能建议    | `[Feature]` | 新功能需求或改进建议      |
| Bug         | `[Bug]`     | 使用中遇到的异常          |` 注入）：

| target_type            | URL 格式                                                      |
| ---------------------- | ------------------------------------------------------------- |
| `kb_page`              | `{domain}/pages/<target_id>`                |
| `kb_file` / `kb_video` | `{domain}/teams/<team_id>/docs/<target_id>` |

---

## 微信公众号导入

支持将微信公众号文章收藏到知识库。

### `file_create_hyperlink` - 导入公众号文章

| 参数            | 必填 | 说明               |
| --------------- | :--: | ------------------ |
| url             |  ✓   | 微信公众号文章 URL |
| space_id        |  ✓   | 目标知识库 ID      |
| parent_entry_id |  ✓   | 父节点 ID          |

**用途**：将微信公众号文章收藏到乐享知识库，自动解析文章内容

**调用示例**：

```json
{
  "url": "https://mp.weixin.qq.com/s/xxxxxxxxxxxx",
  "space_id": "abc123",
  "parent_entry_id": "def456"
}
```

---

## 团队与知识库管理

### `team_list_teams` - 获取团队列表

| 参数       | 必填 | 说明                 |
| ---------- | :--: | -------------------- |
| permission |      | 过滤有特定权限的团队 |

**用途**：获取当前用户可访问的团队列表（这是获取知识库的第一步）

### `space_list_spaces` - 获取知识库列表

| 参数       | 必填 | 说明                                   |
| ---------- | :--: | -------------------------------------- |
| team_id    |  ✓   | 团队 ID（通过 `team_list_teams` 获取） |
| permission |      | 过滤有特定权限的知识库                 |

**返回**：spaces 数组，包含 `id`、`name`、`root_entry_id` 等字段

### `space_describe_space` - 获取知识库详情

| 参数     | 必填 | 说明      |
| -------- | :--: | --------- |
| space_id |  ✓   | 知识库 ID |

**关键返回**：

- `root_entry_id`：根节点 ID，用于 `entry_list_children` 获取一级目录
- `team_id`：所属团队
- `name`：知识库名称

**知识库链接**：`{domain}/spaces/<space_id>`

### `space_list_recently_spaces` - 获取最近访问知识库

| 参数 | 必填 | 说明 |
| ---- | :--: | ---- |
| （无） |  -  | 返回当前用户最近访问的知识库 |

**用途**：当用户只说"最近在看的知识库/文档在哪"时，可先用该工具快速定位候选 Space，再继续 `entry_list_latest_entries`。
---

## 通用说明

### 通用参数约定

| 参数              | 说明                                             |
| ----------------- | ------------------------------------------------ |
| `entry_id`        | 文档/页面的唯一标识符                            |
| `parent_entry_id` | 父级文档 ID（用于创建子文档）                    |
| `block_id`        | Block 的临时 ID（客户端生成，服务端返回实际 ID） |

### Field Selection（`_mcp_fields`）

所有工具都支持可选的 `_mcp_fields` 参数（字符串数组），用于选择返回的字段，类似 GraphQL 的字段选择。指定后，响应中只包含所选字段，**减少 token 消耗**。

- 支持嵌套路径，如 `"entry.id"`、`"entry.name"`
- 未指定时返回默认可见字段

**示例**：只获取知识库的 id 和 root_entry_id

```json
{
  "space_id": "abc123",
  "_mcp_fields": ["id", "root_entry_id", "name"]
}
```

> **建议**：当只需要特定字段时（如获取 `root_entry_id`），使用 `_mcp_fields` 可以显著减少返回数据量。

### Block ID 映射机制

- **客户端传入**：临时 ID（如 `"intro"`, `"h2_1"`）
- **服务端返回**：实际存储的 Block ID
- **用途**：在单次调用中建立 Block 间的父子关系

---

## 工具概述

本 MCP 服务提供以下功能类别的工具。**所有具体业务工具必须通过 `call_tool` 调用，详细参数应通过 `get_tool_schema` 实时获取**。

### 📚 知识库管理
- `entry_create_entry` - 创建文档/文件夹
- `entry_import_content` - 导入 Markdown/HTML 创建新文档（⚠️ **新建文档**，不能覆盖已有文档）
- `entry_import_content_to_entry` - 导入内容到已存在页面（支持覆盖/追加）
- `entry_list_latest_entries` - 获取最近更新条目
- `entry_rename_entry` - 重命名条目

### 📎 文件上传（附件）
- `file_apply_upload` - 申请文件上传（返回 upload_url 和 session_id）
- `file_commit_upload` - 提交上传完成（确认上传）

### 🧩 Block 操作
- `block_convert_content_to_blocks` - Markdown/HTML 转 Block 结构（预处理）
- `block_create_block_descendant` - 创建 Block 结构
- `block_update_block` - 单块更新
- `block_update_blocks` - 批量更新 Block
- `block_move_blocks` - 移动 Block（注意：目标父节点不能是叶子节点）
- `block_delete_block_children` - 删除 Block 子节点
- `block_delete_block` - 删除指定 Block（含子孙）
- `block_describe_block` - 获取单个 Block 详情
- `block_list_block_children` - 读取 Block 内容

### 🔍 搜索与发现
- `search_kb_search` - 关键词搜索
- `search_kb_embedding_search` - 语义向量搜索
- `team_list_teams` - 获取团队列表
- `space_list_spaces` - 获取知识库列表
- `space_describe_space` - 获取知识库详情
- `space_list_recently_spaces` - 获取最近访问知识库

### 🔗 外部内容导入
- `file_create_hyperlink` - 导入公众号文章等外部链接

### 📖 条目与结构浏览
- `entry_list_children` - 浏览目录结构
- `entry_describe_entry` - 获取条目元信息（不包含正文内容）
- `entry_ai_parsed_content` - 获取条目的AI解析内容（包含正文）
- `entry_list_parents` - 获取父级路径（面包屑）

### 📋 详细参数获取
**所有工具参数均以 `get_tool_schema` 返回的最新 schema 为准**。主文档不再维护详细参数表，避免过时。

---

## 📖 内容读取工具说明

### `entry_describe_entry` vs `entry_ai_parsed_content`

| 工具 | 返回内容 | 用途 |
|------|----------|------|
| `entry_describe_entry` | 条目元信息（ID、名称、类型、创建时间等） | 获取条目的基本信息，用于后续操作 |
| `entry_ai_parsed_content` | **条目的正文内容**（经过AI解析的结构化内容） | 读取文档的实际内容 |

**使用场景示例：**
- 当您需要获取文档的标题、类型等基本信息时，使用 `entry_describe_entry`
- 当您需要读取文档的实际内容进行摘要、分析或处理时，使用 `entry_ai_parsed_content`

---

## 常见使用场景

> **⛔ 开始前必读**：
>
> 1. **只有 4 个元工具**：`list_tool_categories`、`search_tools`、`get_tool_schema`、`call_tool`
> 2. 所有业务工具（如 `team_list_teams`）**必须通过 `call_tool` 间接调用**
> 3. **先确定调用方式**（直接调用 → mcporter → HTTP，详见「快速开始」），**确定后不要切换**
>
> **写入安全**：所有涉及创建/修改/删除的操作，必须基于用户明确提供的目标（URL、ID 或确认的名称）。详见 SKILL.md「写入操作安全规则」。

### preset=meta 模式调用说明

当前服务使用 `preset=meta` 模式，**MCP 只暴露 4 个元工具**。所有业务工具必须通过 `call_tool` 元工具来调用。

**调用关系**：

```
Agent 可直接调用的 4 个元工具：
├── list_tool_categories()          → 列出所有工具分类
├── search_tools(query=...)         → 搜索工具
├── get_tool_schema(tool_name=...)  → 获取工具参数定义
└── call_tool(tool_name=..., arguments={...})  → 调用业务工具
         ├── team_list_teams
         ├── space_list_spaces
         ├── entry_list_children
         └── ... 其他所有业务工具
```

**调用方式对照**（首选直接调用，备选 mcporter）：

| 操作 | 直接调用（首选） | mcporter 方式 |
|------|-----------------|---------------|
| 列出工具分类 | `list_tool_categories()` | `mcporter call lexiang.list_tool_categories` |
| 获取团队列表 | `call_tool(tool_name="team_list_teams", arguments={})` | `mcporter call lexiang.call_tool tool_name="team_list_teams" arguments='{}'` |

### 场景0: 用户给了知识库链接，要求写入文档（端到端）

> 用户说："把报告写入乐享，链接是 https://lexiangla.com/spaces/16c4224607ea45ebacce6c15130a4957"

**Step 1**：从 URL 提取 `space_id`（忽略 `?company_from=xxx` 等参数）

```
space_id = "16c4224607ea45ebacce6c15130a4957"
```

**Step 2**：获取知识库的根节点 ID

```bash
mcporter call lexiang.space_describe_space space_id="16c4224607ea45ebacce6c15130a4957"
# → 返回 root_entry_id，如 "abc123def456"
```

**Step 3**：导入内容到知识库根目录

```bash
mcporter call lexiang.entry_import_content \
  space_id="16c4224607ea45ebacce6c15130a4957" \
  parent_id="abc123def456" \
  name="需求分析报告" \
  content="$(cat report.md)" \
  content_type="markdown"
# → 返回新建的 entry 对象，entry.id 用于构建链接
```

> **要点**：`space_id` 和 `parent_id` 要同时传；`parent_id` 用 `root_entry_id` 表示写入根目录。

### 场景1: 创建文档

```bash
mcporter call lexiang.entry_create_entry \
  name="技术文档" \
  parent_entry_id="abc123" \
  entry_type="page"
```

### 场景2: 导入 Markdown 为可编辑文档

```bash
mcporter call lexiang.entry_import_content \
  parent_id="folder123" \
  name="技术文档" \
  content="$(cat document.md)" \
  content_type="markdown"
```

### 场景3: 创建结构化文档

```bash
mcporter call lexiang.block_create_block_descendant \
  entry_id="doc123" \
  descendant='[
    {"block_id": "h1", "block_type": "h1", "heading1": {"elements": [{"text_run": {"content": "项目文档"}}]}},
    {"block_id": "tip", "block_type": "callout", "callout": {"color": "#FFF3E0"}, "children": ["tip_p"]},
    {"block_id": "tip_p", "block_type": "p", "text": {"elements": [{"text_run": {"content": "重要提示内容"}}]}},
    {"block_id": "li1", "block_type": "bulleted_list", "bulleted": {"elements": [{"text_run": {"content": "功能一"}}]}},
    {"block_id": "li2", "block_type": "bulleted_list", "bulleted": {"elements": [{"text_run": {"content": "功能二"}}]}}
  ]' \
  children='["h1", "tip", "li1", "li2"]'
```

### 场景4: 上传文件

```bash
# 1. 申请上传
RESULT=$(mcporter call lexiang.file_apply_upload \
  parent_entry_id="folder123" \
  name="README.md" \
  size=$(stat -f%z README.md))

# 2. 提取 upload_url 和 token
UPLOAD_URL=$(echo "$RESULT" | jq -r '.upload_url')
TOKEN=$(echo "$RESULT" | jq -r '.session_id')

# 3. 上传文件
curl -X PUT "$UPLOAD_URL" --data-binary "@README.md"

# 4. 提交完成
mcporter call lexiang.file_commit_upload session_id="$TOKEN"
```

### 场景5: 读取 Block 内容

```bash
mcporter call lexiang.block_list_block_children \
  entry_id="abc123" \
  with_descendants=true
```

### 场景6: 批量更新 Block

```bash
mcporter call lexiang.block_update_blocks \
  entry_id="abc123" \
  updates='{
    "actual_block_id": {
      "update_text_elements": {
        "elements": [{"text_run": {"content": "更新后的内容"}}]
      }
    }
  }'
```

---

## Block 结构核心规则

### 🍃 叶子节点（不能有 children）
- 标题块：h1, h2, h3, h4, h5
- 代码块：code
- 图片块：image
- 分割线：divider
- 图表块：mermaid, plantuml

### 📦 容器节点（必须指定 children）
- 提示框：callout
- 表格：table, table_cell
- 分栏布局：column_list, column
- 折叠块：toggle

> **详细说明**：完整的 Block 类型、字段定义和文本样式示例见 `references/block-schema.md`。

---



## ⚠️ 核心注意事项

1. **Block ID 映射**：`block_id` 为客户端临时 ID，服务端返回实际 ID 映射
2. **叶子节点限制**：标题、代码块、图片等不支持 children 字段
3. **容器节点要求**：callout、table、column_list 等容器节点必须指定 children
4. **文件上传大小**：必须获取准确的文件大小（字节数），Node.js 用 `fs.statSync(path).size`，Python 用 `os.path.getsize(path)`

> **更多细节**：文件上传、表格顺序、更新已有文件等详细注意事项见 `references/common-errors.md` 和 `references/markdown-import.md`。

---

## 辅助资源

### 参考文档（位于 references/ 目录）

| 文档                    | 说明                   |
| ----------------------- | ---------------------- |
| `block-schema.md`       | Block 类型完整说明     |
| `mcp-examples.md`       | 复杂 Block 结构示例    |
| `markdown-to-block.md`  | Markdown 转 Block 指南 |
| `block-update.md`       | 批量更新 Block 方法    |
| `content-reorganize.md` | 文档结构重组           |
| `folder-sync.md`        | 文件夹同步方案         |
| `markdown-import.md`    | Markdown 导入详解      |
| `common-errors.md`      | 常见错误排查           |
| `skill-maintenance.md`  | 维护与反馈流程         |

### 辅助脚本（位于 scripts/ 目录）

| 脚本               | 说明               |
| ------------------ | ------------------ |
| `sync-folder.ts`   | 文件夹增量同步     |
| `block-helper.ts`  | Block 构建辅助工具 |
| `mcp-validator.ts` | MCP 参数校验       |

详细使用说明见 `scripts/README.md`

---

## ❓ 问题与排查

> **常见问题与详细解答**已移至 `references/common-errors.md`，包括：
> - Markdown 导入方式选择
> - Block ID 管理机制
> - 表格单元格排序规则
> - 文档版本控制实现

遇到具体问题时，请先查阅 `references/common-errors.md` 获取解决方案。

---

## 📮 维护与反馈

关于 Issue 反馈流程、Skill 自我进化机制等维护相关内容，请参阅 `references/skill-maintenance.md`。

该文档包含：
- Issue 反馈的触发场景和流程
- Skill 自我进化的检查和更新机制

---

## 相关链接

- 乐享平台：https://lexiangla.com
- MCP 协议：https://modelcontextprotocol.io
