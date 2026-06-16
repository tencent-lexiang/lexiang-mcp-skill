# lexiang-skills

**乐享知识库 MCP Skill（v2.1.0）**

为 AI Agent 提供乐享知识库的全功能操作能力，包括搜索阅读、文档写入、Block 编辑、文件上传、外部导入等。

---

## 快速开始

### WorkBuddy 用户

WorkBuddy 已内置乐享连接器，**无需手动配置**：

1. 在 WorkBuddy「集成」页面找到「乐享」连接器
2. 点击「授权」完成 OAuth 登录，连接器自动激活

### 其他平台（OpenClaw、Claude 等）

访问 [https://lexiangla.com/mcp](https://lexiangla.com/mcp) 获取 `COMPANY_FROM` 和 `LEXIANG_TOKEN`，填入 `mcp.json`：

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

---

## 目录结构

```
lexiang-skills/
├── SKILL.md                  # 顶层路由入口（意图识别 → 参考文件）
├── mcp.json                  # MCP 配置模板
├── README.md                 # 本文件
├── assets/                   # 静态资源
├── references/               # 参考文档（19 份）
│   ├── base.md               # 数据模型、URL 规则、完整安全规则、工具发现
│   ├── setup.md              # Token 配置、续期、WorkBuddy OAuth、故障排查
│   ├── search.md             # 关键词/语义搜索、内容读取、目录浏览
│   ├── writer.md             # 新建文档、导入内容、公众号收藏
│   ├── blocks.md             # 已有页面的 Block 级增删改移
│   ├── files.md              # 二进制文件上传/下载（三步流程）
│   ├── connectors.md         # 腾讯会议录制导入、iWiki 文档迁移
│   ├── index.md              # 完整索引 + 按场景推荐加载顺序
│   ├── block-schema.md       # Block 类型完整字段定义
│   ├── block-update.md       # 批量更新 Block 方法
│   ├── common-errors.md      # 常见错误排查（高频错误速查表）
│   ├── content-reorganize.md # 文档结构重组方案
│   ├── doc-templates.md      # 文档类型与大纲模板
│   ├── folder-sync.md        # 文件夹同步方案
│   ├── markdown-import.md    # Markdown 导入详解
│   ├── markdown-to-block.md  # Markdown 转 Block 指南
│   ├── mcp-examples.md       # 复杂 Block 结构示例
│   ├── skill-maintenance.md  # Skill 维护指南
│   └── theme-config.md       # 主题配色配置
└── scripts/                  # 辅助脚本
    ├── upload-files.py       # 批量文件上传（支持单文件/文件夹/并行/dry-run）
    ├── sync-folder.ts        # 文件夹增量同步到乐享知识库
    └── test_upload_files.py  # 上传脚本测试
```

---

## 模块说明

根据用户意图，Agent 读取 `references/` 下对应的参考文件：

| 用户意图 | 参考文件 | 典型触发词 |
|---------|---------|-----------|
| 配置乐享 / 401 错误 / Token 过期 | `references/setup.md` | 「配置乐享」「token 过期」「连不上」「401」 |
| 搜索 / 查找 / 阅读 / 浏览 | `references/search.md` | 「找一下」「搜索」「读一下这个页面」 |
| 创建文档 / 写入 / 保存 / 导入 | `references/writer.md` | 「写到乐享」「创建文档」「保存到知识库」 |
| 编辑已有页面 / Block 操作 | `references/blocks.md` | 「修改这个页面」「加个标题」「删掉这段」 |
| 上传/下载文件 | `references/files.md` | 「传个 PDF」「上传文件」「下载这个文件」 |
| 导入腾讯会议 / iWiki 迁移 | `references/connectors.md` | 「导入会议录制」「迁移 iWiki 文档」 |
| 数据模型 / URL 规则 / 安全规则 | `references/base.md` | 由上述模块内部引用 |

> 完整路由规则和易混淆场景见 `SKILL.md`

---

## 核心规则

- **写入安全**：必须基于用户明确提供的目标（URL/ID/确认），禁止自行遍历或猜测
- **链接生成**：使用 `whoami().company.company_domain` 作为域名；顶级域名需追加 `?company_from=`
- **401 处理**：不重试，引导用户续期（点续期按钮即可恢复，无需重新配置）
- **强制检索**：用户提到「乐享里的文档/内容」时，必须先搜索再回答，禁止凭空生成

> 完整规则详见 `SKILL.md` 和 `references/base.md`

---

## 辅助脚本

| 脚本 | 说明 |
|------|------|
| `scripts/upload-files.py` | 批量文件上传（支持单文件/文件夹/并行/dry-run） |
| `scripts/sync-folder.ts` | 文件夹增量同步到乐享知识库 |
| `scripts/test_upload_files.py` | 上传脚本测试 |

```bash
# 单文件上传
python scripts/upload-files.py --files doc.md --entry-id <parent_entry_id>

# 文件夹批量上传（5 并行）
python scripts/upload-files.py --folder ./docs --entry-id <parent_entry_id> --parallel 5
```

---

## 参考文档

### 主参考文档（按意图路由加载）

| 文档 | 说明 |
|------|------|
| `references/base.md` | 数据模型、URL 规则、完整安全规则、工具发现 |
| `references/setup.md` | Token 配置、续期、WorkBuddy OAuth、故障排查 |
| `references/search.md` | 关键词/语义搜索、内容读取、目录浏览 |
| `references/writer.md` | 新建文档、导入内容、公众号收藏 |
| `references/blocks.md` | 已有页面的 Block 级增删改移 |
| `references/files.md` | 二进制文件上传/下载（三步流程） |
| `references/connectors.md` | 腾讯会议录制导入、iWiki 文档迁移 |
| `references/index.md` | 完整索引 + 按场景推荐加载顺序 |

### 补充参考文档

| 文档 | 说明 |
|------|------|
| `references/block-schema.md` | Block 类型完整字段定义 |
| `references/mcp-examples.md` | 复杂 Block 结构示例 |
| `references/markdown-to-block.md` | Markdown 转 Block 指南 |
| `references/block-update.md` | 批量更新 Block 方法 |
| `references/content-reorganize.md` | 文档结构重组方案 |
| `references/folder-sync.md` | 文件夹同步方案 |
| `references/markdown-import.md` | Markdown 导入详解 |
| `references/common-errors.md` | 常见错误排查（高频错误速查表） |
| `references/doc-templates.md` | 文档类型与大纲模板 |
| `references/theme-config.md` | 主题配色配置 |
| `references/skill-maintenance.md` | Skill 维护指南 |

---

## 相关链接

- 乐享平台：https://lexiangla.com
- 获取 MCP 配置：https://lexiangla.com/mcp
- MCP 协议：https://modelcontextprotocol.io
