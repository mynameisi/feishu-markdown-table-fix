# Feishu Markdown 表格渲染修复

修复 Hermes Agent 向飞书发送 Markdown 时表格/代码块/标题显示为源码的问题。

## 问题

飞书原生不渲染 Markdown 的表格（`|...|`）、标题（`#`）、代码块（```），全部显示为纯文本源码。

## 解决方案

修改 `/opt/hermes/gateway/platforms/feishu.py`，将 Markdown 内容转为 **CardKit 2.0 interactive card**。

## 修改文件

仅修改一个文件：`feishu.py`

## 修改内容

### 1. 新增辅助函数（~line 610-758）

在文件中新增以下 5 个函数：

```python
def _split_by_code_fences(content: str) -> list:
    """Split markdown content by fenced code blocks, keeping fences intact."""
    parts: List[str] = []
    current: List[str] = []
    in_code_block = False

    for line in content.split("\n"):
        stripped = line.strip()
        if not in_code_block and _MARKDOWN_FENCE_OPEN_RE.match(stripped):
            if current:
                parts.append("\n".join(current))
                current = []
            in_code_block = True
            current.append(line)
        elif in_code_block and _MARKDOWN_FENCE_CLOSE_RE.match(stripped):
            current.append(line)
            in_code_block = False
            parts.append("\n".join(current))
            current = []
        else:
            current.append(line)

    if current:
        parts.append("\n".join(current))
    return parts


def _preprocess_card_markdown(text: str) -> str:
    """Adapt standard markdown for Feishu CardKit markdown tag.

    Feishu card ``markdown`` elements do not render ``#`` headers.  Convert
    them to bold text so headings remain visually distinct.
    """
    return re.sub(r"^(#{1,6})\s+(.+)$", r"**\2**", text, flags=re.MULTILINE)


def _parse_text_blocks(elements: List[Dict[str, Any]], text: str) -> None:
    """Parse a text segment, splitting tables into ``column_set`` card elements."""
    lines = text.split("\n")
    i = 0
    buf: List[str] = []

    def _flush_buf() -> None:
        nonlocal buf
        if not buf:
            return
        segment = "\n".join(buf)
        processed = _preprocess_card_markdown(segment)
        if processed.strip():
            elements.append({"tag": "markdown", "content": processed})
        buf = []

    while i < len(lines):
        stripped = lines[i].strip()
        # Detect table: a pipe-delimited line followed by a separator line.
        is_table_start = (
            stripped.startswith("|")
            and i + 1 < len(lines)
            and bool(re.match(r"^\|[-|: ]+\|$", lines[i + 1].strip()))
        )
        if is_table_start:
            _flush_buf()
            table_lines = [lines[i], lines[i + 1]]
            i += 2
            while i < len(lines) and lines[i].strip().startswith("|"):
                table_lines.append(lines[i])
                i += 1
            _append_table_element(elements, "\n".join(table_lines))
        else:
            buf.append(lines[i])
            i += 1

    _flush_buf()


def _append_table_element(elements: List[Dict[str, Any]], table_block: str) -> None:
    """Render a markdown table as CardKit 2.0 ``column_set`` layout.

    - Header row: grey background with bold text.
    - Data rows: alternating default / grey backgrounds.
    - ``hr`` dividers between header and data, and after the last row.
    """
    lines = [l.rstrip() for l in table_block.strip().split("\n") if l.strip()]
    if len(lines) < 2:
        return
    headers = [c.strip() for c in lines[0].split("|") if c.strip()]
    data_lines = lines[2:]  # skip header + separator
    n = len(headers)
    if n == 0:
        return

    # Header row
    hcols = [
        {
            "tag": "column",
            "width": "weighted",
            "weight": 1,
            "vertical_align": "top",
            "elements": [{"tag": "markdown", "content": f"**{h}**"}],
        }
        for h in headers
    ]
    elements.append({"tag": "column_set", "flex_mode": "none", "background_style": "grey", "columns": hcols})
    elements.append({"tag": "hr"})

    # Data rows with alternating background
    for idx, rl in enumerate(data_lines):
        cells = [c.strip() for c in rl.split("|") if c.strip()]
        cells = (cells + [""] * n)[:n]
        rcols = [
            {
                "tag": "column",
                "width": "weighted",
                "weight": 1,
                "vertical_align": "top",
                "elements": [{"tag": "markdown", "content": c or " "}],
            }
            for c in cells
        ]
        bg = "default" if idx % 2 == 0 else "grey"
        elements.append({"tag": "column_set", "flex_mode": "none", "background_style": bg, "columns": rcols})
    elements.append({"tag": "hr"})


def _append_code_block_element(elements: List[Dict[str, Any]], code_block: str) -> None:
    """Append a fenced code block as a ``div`` + ``lark_md`` card element."""
    code_stripped = code_block.strip()
    elements.append({"tag": "div", "text": {"tag": "lark_md", "content": code_stripped}})


def _parse_content_to_card_elements(content: str) -> List[Dict[str, Any]]:
    """Convert markdown content into a list of CardKit 2.0 card elements.

    1. Split by fenced code blocks so prose and code are handled separately.
    2. Within prose segments, detect tables and convert to ``column_set``.
    3. Non-table markdown goes through ``_preprocess_card_markdown`` (headers
       become bold) and is emitted as a ``markdown`` element.
    """
    elements: List[Dict[str, Any]] = []
    parts = _split_by_code_fences(content)

    for part in parts:
        if not part.strip():
            continue
        if _MARKDOWN_FENCE_OPEN_RE.match(part.strip()):
            _append_code_block_element(elements, part)
        else:
            _parse_text_blocks(elements, part)
    return elements
```

### 2. 修改 `_build_outbound_payload`（~line 4376）

替换原来的 `_build_outbound_payload` 方法：

```python
def _build_outbound_payload(self, content: str) -> tuple[str, str]:
    # When content contains markdown formatting (tables, headers, code
    # blocks, bold, etc.), use CardKit 2.0 interactive cards which render
    # all markdown elements properly.  The old "post" type's md elements
    # silently swallow tables and headers.
    if _MARKDOWN_TABLE_RE.search(content) or _MARKDOWN_HINT_RE.search(content):
        elements = _parse_content_to_card_elements(content)
        card = json.dumps(
            {
                "schema": "2.0",
                "config": {"wide_screen_mode": True},
                "body": {"elements": elements},
            },
            ensure_ascii=False,
        )
        return "interactive", card
    text_payload = {"text": content}
    return "text", json.dumps(text_payload, ensure_ascii=False)
```

### 3. 确保正则常量存在（~line 152-161）

文件顶部应已有这些正则（无需修改）：

```python
_MARKDOWN_HINT_RE = re.compile(
    r"(^#{1,6}\s)|(^\s*[-*]\s)|(^\s*\d+\.\s)|(^\s*---+\s*$)|(```)|(`[^\n`]+`)|(\*\*[^*\\n].+?\*\*)|(~~[^~\\n].+?~~)|(<u>.+?</u>)|(\*[^*\\n]+\*)|(\[[^\]]+\]\([^)]+\))|(^>\s)",
    re.MULTILINE,
)
_MARKDOWN_TABLE_RE = re.compile(r"^\|.*\|\n\|[-|: ]+\|", re.MULTILINE)
_MARKDOWN_FENCE_OPEN_RE = re.compile(r"^```([^\n`]*)\s*$")
_MARKDOWN_FENCE_CLOSE_RE = re.compile(r"^```\s*$")
```

## 重启网关

```bash
cat /opt/data/gateway_state.json   # 找到 PID
kill -TERM [PID]                     # 等自动重启后测试
```

## 验证

发送包含标题、表格、代码块、粗体、列表的消息，所有格式应正常渲染。

## 效果

| 渲染前 | 渲染后 |
|---|---|
| `\| col1 \| col2 \|` | 表头灰底加粗，数据行交替底色 |
| `# 标题` | **标题**（加粗显示） |
| ` ```code``` ` | 代码块 div + lark_md |

## License

MIT
