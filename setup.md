# Setup Guide - Lexiang MCP Skill (ClawHub Edition)

> 本文档描述配置步骤，AI Agent 应读取后自动完成配置。

## 前置条件

用户需要提供两个参数（如果用户未提供，引导用户访问 https://lexiangla.com/mcp 获取）：

| 参数 | 说明 | 格式示例 |
|------|------|----------|
| `company_from` | 企业标识 | `your_company` |
| `access_token` | 访问令牌 | `lxmcp_xxx` |

## 配置步骤

### Step 1: 获取用户参数

向用户询问 `company_from` 和 `access_token`。如果用户不清楚，引导其访问：

```
https://lexiangla.com/mcp
```

登录后即可看到配置信息。

**校验规则**：两个参数都不能为空。

### Step 2: 确定配置文件路径

配置文件路径为：

| 操作系统 | 路径 |
|----------|------|
| macOS / Linux | `~/.mcporter/mcporter.json` |
| Windows | `%USERPROFILE%\.mcporter\mcporter.json` |
| WSL | `~/.mcporter/mcporter.json`（Linux 侧路径） |

### Step 3: 检查已有配置

如果配置文件已存在，**提示用户确认是否覆盖**，不要静默覆盖。

### Step 4: 创建目录并写入配置

1. 如果 `.mcporter` 目录不存在，创建它
2. 将以下 JSON 写入配置文件（替换 `{company_from}` 和 `{access_token}` 为用户提供的实际值）：

```json
{
  "mcpServers": {
    "lexiang": {
      "url": "https://mcp.lexiang-app.com/mcp?company_from={company_from}&preset=meta",
      "transportType": "streamable-http",
      "headers": {
        "Authorization": "Bearer {access_token}"
      }
    }
  }
}
```

**安全说明**：access_token 通过 HTTP Authorization header 传递，避免在 URL 中暴露（URL 可能被记录到日志、Referer 头中）。

**编码要求**：文件必须以 UTF-8 无 BOM 编码保存。

### Step 5: 确认结果

配置写入后，告知用户配置文件的完整路径，并提示配置完成。

## 注意事项

- 不要在输出中回显 access_token 的完整值（安全考虑）
- 如果已有配置文件包含其他 mcpServers 条目，应合并而非覆盖整个文件
- Windows 环境下注意路径分隔符使用 `\`
