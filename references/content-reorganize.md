# 场景：内容重组

使用 MoveBlocks 调整文档结构，将块移动到新位置。

前置条件：
- 已确认目标页面 `entry_id`
- 已确认待移动的 `block_id` 列表
- 已确认目标父节点 `parent_block_id`
- 需要先检查结构时，先读 `references/page-edit.md`

## 移动块 API

```
业务工具：`block_move_blocks`
Arguments: {
  "entry_id": "<entry_id>",
  "block_ids": ["block_1", "block_2", "block_3"],
  "parent_block_id": "<目标父块 ID>",
  "after": "<插入位置，某块之后，可选>"
}
```

**限制**: 单次最多移动 20 个块

---

## 参数说明

| 参数 | 说明 |
|------|------|
| `entry_id` | 文档 entry_id |
| `block_ids` | 要移动的块 ID 数组，按顺序移动 |
| `parent_block_id` | 目标父节点块 ID |
| `after` | 插入到此块之后，为空则插入到开头 |

---

## 使用场景

### 将分散内容整合到同一章节

```
业务工具：`block_move_blocks`
Arguments: {
  "entry_id": "doc123",
  "block_ids": ["para_1", "para_2", "list_1"],
  "parent_block_id": "section_h2",
  "after": "intro_callout"
}
```

### 调整段落顺序

```
业务工具：`block_move_blocks`
Arguments: {
  "entry_id": "doc123",
  "block_ids": ["para_3"],
  "parent_block_id": "root_block",
  "after": "para_1"
}
```

---

## 注意事项

1. **所有块只能移动到同一个目标父节点**
2. **目标父节点不能是叶子节点类型**，包括：
   - h1, h2, h3, h4, h5（标题块）
   - code（代码块）
   - image（图片块）
   - attachment（附件块）
   - video（视频块）
   - divider（分割线）
   - mermaid、plantuml（图表块）
3. 移动操作会保持块的子孙结构
4. 程序化构造移动计划时，可使用 `scripts/block-helper.ts` 中的 `ContentReorganizer`
