# lexiang-skills

**乐享知识库 MCP Skill（v2.2.3）**

给 AI Agent 用的乐享知识库操作手册。Agent 读取 `SKILL.md` 识别意图，再按需加载 `references/` 里对应的参考文档，通过乐享 MCP 完成搜索、写入、编辑、文件、表格和外部导入。

面向乐享 **v2**。`whoami()` 返回 `sub_server_type=v1`（或用户明确在操作「乐享社区」）时，应切换到 `lexiang-v1-docs`，本 Skill 不适用。

---

## 能做什么

| 能力 | 说明 |
|------|------|
| 搜索与阅读 | 关键词召回、语义切片检索、目录浏览；PDF / PPT / Word / 图片走服务端 AI 解析 |
| 新建与导入 | Markdown / HTML 新建文档、写入已有页面、公众号链接收藏；未指定目标时可写入个人知识库 |
| 页面编辑 | 已有页面的 Block 增删改移，优先用页面级更新 |
| 文件 | PDF、Word、图片等二进制文件的三步上传与下载 |
| 智能表格 | 用 SQL DDL 管 schema，并增删查改记录与视图 |
| 草稿 | Markdown 草稿的保存、查看、发布、删除 |
| 评论 | 查看页面评论（只读） |
| 外部导入 | 腾讯会议录制搜索、导入、重新导入 |
| 连接配置 | Token 手动配置，以及 WorkBuddy / QClaw 等内置连接器的 OAuth |

---

## 快速开始

### WorkBuddy、QClaw 等内置连接器

客户端已内置乐享连接器时，**不要手写 `mcp.json`**。连接器会覆盖手动安装的 MCP。

1. 在客户端「集成」/「连接器」里找到「乐享」
2. 点击「授权」完成 OAuth
3. 用 `whoami()` 确认已连通

连接断开或 401 时，在同一入口点「重新授权」，不要改配置文件。

### 其他平台（Cursor、Claude、OpenClaw 等）

打开 [https://lexiangla.com/mcp](https://lexiangla.com/mcp)，登录后拿到 `COMPANY_FROM` 和 `LEXIANG_TOKEN`（`lxmcp_xxx`），写入 `mcp.json`：

```json
{
  "mcpServers": {
    "lexiang": {
      "enabled": true,
      "url": "https://mcp.lexiang-app.com/mcp?company_from=你的COMPANY_FROM",
      "transportType": "streamable-http",
      "headers": {
        "Authorization": "Bearer 你的LEXIANG_TOKEN"
      }
    }
  }
}
```

仓库里的 [`mcp.json`](mcp.json) 使用 `${COMPANY_FROM}`、`${LEXIANG_TOKEN}` 占位，部署时替换成真实值。Token 过期后在乐享侧点续期即可，不必重写整份配置。401 时停止重试，按 [`references/setup.md`](references/setup.md) 处理。

---

## 意图路由

Agent 只加载当前任务需要的参考文件。完整规则、易混淆场景和跨模块顺序见 [`SKILL.md`](SKILL.md)，按场景的加载顺序见 [`references/index.md`](references/index.md)。

| 用户意图 | 参考文件 | 典型触发 |
|---------|---------|---------|
| 配置、401、Token 过期、切换企业 | [`references/setup.md`](references/setup.md) | 「配置乐享」「连不上」「401」 |
| 搜索、阅读、浏览、分析文件内容 | [`references/search.md`](references/search.md) | 「找一下」「读一下这个页面」「这份 PDF 讲了什么」 |
| 新建文档、导入、保存到知识库 | [`references/writer.md`](references/writer.md) | 「写到乐享」「存到我的知识库」 |
| 改已有页面、Block 增删改移 | [`references/blocks.md`](references/blocks.md) | 「改这个页面」「加个标题」「删掉这段」 |
| 上传或下载文件本体 | [`references/files.md`](references/files.md) | 「传个 PDF」「下载这个文件」 |
| 导入腾讯会议录制 | [`references/connectors.md`](references/connectors.md) | 「把会议录制导入知识库」 |
| 智能表格、结构化数据 | [`references/smartsheet.md`](references/smartsheet.md) | 「智能表格」「新增一行」「查记录」 |
| 草稿、发布 | [`references/draft.md`](references/draft.md) | 「先存草稿」「发布草稿」 |
| 查看评论 | [`references/comment.md`](references/comment.md) | 「看看评论」「有没有讨论」 |
| 数据模型、URL、安全规则、工具发现 | [`references/base.md`](references/base.md) | 由上面的模块引用 |

容易判错的几条：

- `/pages/...` 且要求修改 → `blocks.md`；只读总结 → `search.md`
- `/spaces/...` 且要求新建 → `writer.md`
- 分析附件内容用 `entry_describe_ai_parse_content`；只有要原始文件时才下载
- 创建 Markdown 文本文档走 `writer.md`；上传 PDF / Word / 图片走 `files.md`

---

## 核心规则

- **写入目标必须明确**：URL、ID，或用户确认过的名称。禁止遍历团队/知识库后自行挑一个写入。
- **个人知识库例外**：用户说「保存到知识库 / 个人知识库」但没指定具体库，且 `whoami()` 返回了个人知识库时，可以直接写入。
- **链接用企业域名**：取 `whoami().company.company_domain`。三级域名直接拼路径；顶级域名（如 `lexiangla.com`）追加 `?company_from=`。`company_from` 不能当成子域名。禁止用 MCP endpoint 拼用户链接。
- **先检索再回答**：用户同时提到乐享/知识库，以及文档、内容、资料、笔记、上次写的、写作风格时，先搜索或读取真实内容。
- **大批量分批**：文件或条目超过 20 个时每批不超过 20，先告诉用户数量和策略。
- **401 不重试**：内置连接器引导重新授权；其他平台引导续期。

细则在 [`SKILL.md`](SKILL.md) 和 [`references/base.md`](references/base.md)。调用参数以 MCP 实时 schema 为准（`get_tool_schema`）；若客户端只暴露元工具，业务工具经 `call_tool` 代理。

---

## 目录结构

```
lexiang-skills/
├── SKILL.md                      # Agent 入口：意图识别与路由
├── mcp.json                      # MCP 配置模板
├── README.md
├── references/                   # 参考文档（22 份，按需加载）
│   ├── base.md                   # 数据模型、URL、安全规则、工具发现
│   ├── setup.md                  # Token、OAuth、续期、故障排查
│   ├── search.md                 # 搜索、阅读、文件 AI 解析、目录
│   ├── writer.md                 # 新建、导入、公众号收藏、个人知识库
│   ├── blocks.md                 # 已有页面的 Block 操作
│   ├── files.md                  # 二进制上传/下载（申请 → PUT → 确认）
│   ├── connectors.md             # 腾讯会议录制导入
│   ├── smartsheet.md             # 智能表格
│   ├── draft.md                  # Markdown 草稿
│   ├── comment.md                # 评论查看
│   ├── index.md                  # 索引与推荐加载顺序
│   ├── block-schema.md           # Block 类型字段
│   ├── block-update.md           # 批量更新 Block
│   ├── markdown-to-block.md      # Markdown 转 Block
│   ├── markdown-import.md        # Markdown 导入方式选择
│   ├── mcp-examples.md           # 复杂 Block 结构示例
│   ├── content-reorganize.md     # 用移动 Block 重组文档
│   ├── doc-templates.md          # 文档类型与大纲模板
│   ├── folder-sync.md            # 本地文件夹增量同步方案
│   ├── theme-config.md           # 主题配色配置说明
│   ├── common-errors.md          # 高频错误速查
│   └── skill-maintenance.md      # Skill 自查与反馈
└── scripts/
    ├── upload-files.py           # 生成批量上传计划
    ├── sync-folder.ts            # 生成文件夹增量同步计划
    ├── test_upload_files.py      # 上传脚本测试
    └── README.md
```

补充文档在主流程里按需再读，不必和路由文件一起加载。

---

## 辅助脚本

脚本扫描本地文件并**打印 MCP 调用计划**。实际上传、建目录由 Agent 按计划调用乐享 MCP 完成（文件上传固定为 `file_apply_upload` → HTTP PUT → `file_commit_upload`）。

依赖：

```bash
pip install aiohttp requests          # upload-files.py；无 aiohttp 时回退到 requests
npm install -g ts-node typescript     # sync-folder.ts
npm install @types/node
```

```bash
# 生成单文件 / 多文件上传计划（默认扩展名 .md .txt .pdf）
python scripts/upload-files.py --files doc.md --entry-id <parent_entry_id>

# 扫描文件夹，5 路并行计划；--dry-run 只出计划
python scripts/upload-files.py --folder ./docs --entry-id <parent_entry_id> \
  --parallel 5 --extensions .md .pdf --output plan.json --dry-run

# 文件夹增量同步计划（状态默认写在 .lexiang-sync-state.json）
npx ts-node scripts/sync-folder.ts --local ./docs --entry-id <parent_entry_id> --dry-run
npx ts-node scripts/sync-folder.ts --local ./docs --entry-id <parent_entry_id> \
  --space-id <space_id> --state-file .lexiang-sync-state.json
```

`sync-folder.ts` 会跳过 `node_modules/`、`.git/` 等，默认识别 `.md`、`.markdown`、`.txt`、`.json`、`.pdf`、`.doc`、`.docx`。新文件走三步上传，内容变更走文件版本更新，新目录创建 folder。删除不会自动同步到远端。

更多参数见 [`scripts/README.md`](scripts/README.md)。

---

## 相关链接

- 乐享平台：https://lexiangla.com
- 获取 MCP 配置：https://lexiangla.com/mcp
- MCP 协议：https://modelcontextprotocol.io
