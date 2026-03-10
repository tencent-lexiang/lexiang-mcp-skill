# 乐享 MCP 常见错误

仅在调用报错或程序化参数需要预检查时读取本文件。

- 页面结构规则以 `references/block-schema.md` 为准
- 参数由代码生成时，可使用 `scripts/mcp-validator.ts`

## 高频错误速查

| 场景 | 常见问题 | 处理 |
|------|----------|------|
| 文件上传 | 缺少 `size` | 传文件大小（字节数） |
| 更新文件 | `parent_entry_id` 或 `file_id` 传错 | 更新时 `parent_entry_id` 为当前文件 `entry_id`，`file_id` 来自 `entry_describe_entry.target_id` |
| 页面结构创建/更新 | 叶子节点带 `children` | 对照 `references/block-schema.md` 的叶子节点规则 |
| 页面结构创建/更新 | 容器块缺少 `children` | 对照 `references/block-schema.md` 的容器块规则 |

## 文件上传

### 缺少 `size`

`file_apply_upload` 必须传文件大小。无论上传 Markdown、附件还是更新文件版本，都不能省略 `size`。

### 更新文件参数混淆

| 场景 | `parent_entry_id` | `file_id` |
|------|-------------------|-----------|
| 新建文件 | 父目录 `entry_id` | 不传 |
| 更新文件 | 当前文件自己的 `entry_id` | 必传 |

获取 `file_id` 的方式：
1. 调用 `entry_describe_entry`
2. 从返回值读取 `entry.target_id`

## 页面结构错误

这类错误不在本文件重复 schema，直接对照 `references/block-schema.md`。

| 现象 | 常见原因 | 先检查 |
|------|----------|--------|
| 标题、代码块、分割线报 `children` 错误 | 叶子节点不能带 `children` | `支持的块类型`、`注意事项` |
| `callout`、`table`、`column_list` 等结构不完整 | 容器块必须带 `children` | `支持的块类型`、`特殊块结构` |
| 表格、分栏内容顺序异常 | `children` 顺序或子块类型不对 | `特殊块结构` |

## 校验工具

仅当参数由代码生成时再使用 `scripts/mcp-validator.ts`。

- 上传参数预检查：缺少 `size`、更新场景缺少 `file_id`
- 页面结构预检查：叶子/容器节点约束、常见结构错误
