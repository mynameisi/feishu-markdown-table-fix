# Complete Modified Code Sections for feishu.py

All functions are module-level (not methods on a class) except `_build_outbound_payload` which is a method on the Feishu adapter class.

## Regex Constants (line ~152)

Already present in feishu.py — no modification needed:

```python
_MARKDOWN_HINT_RE = re.compile(
    r"(^#{1,6}\s)|(^\s*[-*]\s)|(^\s*\d+\.\s)|(^\s*---+\s*$)|(```)|(`[^\n`]+`)"
    r"|(\*\*[^*\\n].+?\*\*)|(~~[^~\\n].+?~~)|(<u>.+?</u>)|(\*[^*\\n]+\*)"
    r"|(\[[^\]]+\]\([^)]+\))|(^>\s)",
    re.MULTILINE,
)
_MARKDOWN_TABLE_RE = re.compile(r"^\|.*\|\n\|[-|: ]+\|", re.MULTILINE)
_MARKDOWN_FENCE_OPEN_RE = re.compile(r"^```([^\n`]*)\s*$")
_MARKDOWN_FENCE_CLOSE_RE = re.compile(r"^```\s*$")
```

## _split_by_code_fences (line ~610)

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
```

## _preprocess_card_markdown (line ~637)

```python
def _preprocess_card_markdown(text: str) -> str:
    """Adapt standard markdown for Feishu CardKit markdown tag.

    Feishu card ``markdown`` elements do not render ``#`` headers.  Convert
    them to bold text so headings remain visually distinct.
    """
    return re.sub(r"^(#{1,6})\s+(.+)$", r"**\2**", text, flags=re.MULTILINE)
```

## _parse_text_blocks (line ~646)

```python
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
```

## _append_table_element (line ~685)

```python
def _append_table_element(elements: List[Dict[str, Any]], table_block: str) -> None:
    """Render a markdown table as CardKit 2.0 ``column_set`` layout."""
    lines = [l.rstrip() for l in table_block.strip().split("\n") if l.strip()]
    if len(lines) < 2:
        return
    headers = [c.strip() for c in lines[0].split("|") if c.strip()]
    data_lines = lines[2:]
    n = len(headers)
    if n == 0:
        return

    # Header row
    hcols = [
        {
            "tag": "column", "width": "weighted", "weight": 1,
            "vertical_align": "top",
            "elements": [{"tag": "markdown", "content": f"**{h}**"}],
        }
        for h in headers
    ]
    elements.append({"tag": "column_set", "flex_mode": "none",
                      "background_style": "grey", "columns": hcols})
    elements.append({"tag": "hr"})

    # Data rows with alternating background
    for idx, rl in enumerate(data_lines):
        cells = [c.strip() for c in rl.split("|") if c.strip()]
        cells = (cells + [""] * n)[:n]
        rcols = [
            {
                "tag": "column", "width": "weighted", "weight": 1,
                "vertical_align": "top",
                "elements": [{"tag": "markdown", "content": c or " "}],
            }
            for c in cells
        ]
        bg = "default" if idx % 2 == 0 else "grey"
        elements.append({"tag": "column_set", "flex_mode": "none",
                          "background_style": bg, "columns": rcols})
    elements.append({"tag": "hr"})
```

## _append_code_block_element (line ~734)

```python
def _append_code_block_element(elements: List[Dict[str, Any]], code_block: str) -> None:
    """Append a fenced code block as a ``div`` + ``lark_md`` card element."""
    code_stripped = code_block.strip()
    elements.append({"tag": "div", "text": {"tag": "lark_md", "content": code_stripped}})
```

## _parse_content_to_card_elements (line ~740)

```python
def _parse_content_to_card_elements(content: str) -> List[Dict[str, Any]]:
    """Convert markdown content into a list of CardKit 2.0 card elements."""
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

## _build_outbound_payload method (line ~4376)

This is a method on the Feishu adapter class (not module-level):

```python
def _build_outbound_payload(self, content: str) -> tuple[str, str]:
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
