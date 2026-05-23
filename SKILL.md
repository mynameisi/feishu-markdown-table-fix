---
name: feishu-markdown-table-fix
description: "Fix Hermes Agent Feishu Markdown rendering: convert MD tables/code blocks/headers to CardKit 2.0 interactive cards."
tags: [hermes, feishu, markdown, cardkit, render, gateway]
version: "1.0.0"
created: "2026-05-23"
---

# Feishu Markdown 表格渲染修复

修复 Hermes Agent 发送到飞书时 Markdown 格式（标题、表格、代码块）显示为源码而非渲染结果的问题。

## 问题

发送 Markdown 到飞书时，标题（#）、表格（|...|）、代码块（```）显示为源码而不是渲染结果。

## 修复路径

`/opt/hermes/gateway/platforms/feishu.py`

## 修改步骤

### Step 1: 修改 `_build_outbound_payload`

改为使用 interactive card 发送 Markdown 内容：

```python
def _build_outbound_payload(self, content):
    if _MARKDOWN_HINT_RE.search(content) or _MARKDOWN_TABLE_RE.search(content):
        elements = self._parse_content_to_card_elements(content)
        card = json.dumps({
            "schema": "2.0",
            "config": {"wide_screen_mode": True},
            "body": {"elements": elements}
        }, ensure_ascii=False)
        return "interactive", card
    return "text", json.dumps({"text": content}, ensure_ascii=False)
```

### Step 2: 修改 `_parse_content_to_card_elements`

先拆分代码块，再调用 `_parse_text_blocks` 处理表格：

```python
def _parse_content_to_card_elements(self, content):
    elements = []
    parts = re.split(r'(```\n.*?\n```)', content, flags=re.DOTALL)
    for part in parts:
        if not part.strip():
            continue
        if part.startswith("```"):
            self._append_code_block_element(elements, part)
        else:
            self._parse_text_blocks(elements, part)
    return elements
```

### Step 3: 新增 `_parse_text_blocks`

用 regex 检测表格块，表格走 `_append_table_element`，其他走 markdown tag：

```python
def _parse_text_blocks(self, elements, text):
    tbl = re.compile(r'(^|.+|\n|[-:| ]+|\n(?:|.+|\n?)*)', re.MULTILINE)
    for part in tbl.split(text):
        if not part.strip():
            continue
        if re.match(r'^|.+|\n|[-:| ]+|\n?', part, re.MULTILINE):
            self._append_table_element(elements, part)
        else:
            processed = self._preprocess_card_markdown(part)
            if processed.strip():
                elements.append({"tag": "markdown", "content": processed})
```

### Step 4: 新增 `_append_table_element`

将 Markdown 表格解析为 CardKit 2.0 column_set 布局：
- 表头行：灰底 + 粗体
- 数据行：交替底色（default/grey）
- 表头和表尾加 hr 分隔线

```python
@staticmethod
def _append_table_element(elements, table_block):
    lines = [l.rstrip() for l in table_block.strip().split('\n') if l.strip()]
    if len(lines) < 2:
        return
    headers = [c.strip() for c in lines[0].split('|') if c.strip()]
    data_lines = lines[2:]
    n = len(headers)

    # header row
    hcols = [{"tag": "column", "width": "weighted", "weight": 1,
              "vertical_align": "top",
              "elements": [{"tag": "markdown", "content": f"**{h}**"}]}
             for h in headers]
    elements.append({"tag": "column_set", "flex_mode": "none",
                      "background_style": "grey", "columns": hcols})
    elements.append({"tag": "hr"})

    # data rows with alternating background
    for i, rl in enumerate(data_lines):
        cells = [c.strip() for c in rl.split('|') if c.strip()]
        cells = (cells[:n] + [''] * n)[:n]
        rcols = [{"tag": "column", "width": "weighted", "weight": 1,
                  "vertical_align": "top",
                  "elements": [{"tag": "markdown", "content": c or " "}]}
                 for c in cells]
        bg = "default" if i % 2 == 0 else "grey"
        elements.append({"tag": "column_set", "flex_mode": "none",
                          "background_style": bg, "columns": rcols})
        elements.append({"tag": "hr"})
```

### Step 5: 确保辅助方法存在

- `_append_code_block_element`：代码块转为 div + lark_md
- `_preprocess_card_markdown`：`#` 标题转为加粗

### Step 6: 重启网关

```bash
cat /opt/data/gateway_state.json   # 找到 PID
kill -TERM [PID]                     # 等自动重启后测试
```

## 验证

发送包含标题、表格、代码块、粗体、列表的消息，所有格式应正常渲染。

## 注意事项

- 仅修改 `feishu.py`，不涉及其他文件
- CardKit 2.0 schema 是飞书新版消息卡片格式
- 交替底色提升表格可读性
- 代码块保持原样渲染，不走 markdown tag
