# feishu-markdown-table-fix 🔧

修复 Hermes Agent 向飞书发送 Markdown 时表格/代码块/标题显示为源码的问题。

## 问题

飞书原生不渲染 Markdown 的表格（`|...|`）、标题（`#`）、代码块（```），全部显示为纯文本源码。

## 解决方案

修改 `/opt/hermes/gateway/platforms/feishu.py`，将 Markdown 内容转为 **CardKit 2.0 interactive card**：

- **表格** → `column_set` 布局，表头灰底加粗，数据行交替底色
- **代码块** → `div` + `lark_md` 标签
- **标题** → 预处理为加粗文本

## 修改文件

仅修改一个文件：`feishu.py`

| 方法 | 操作 |
|---|---|
| `_build_outbound_payload` | 改为检测 Markdown 并走 interactive card |
| `_parse_content_to_card_elements` | 拆分代码块，调用 text blocks |
| `_parse_text_blocks` | 新增：检测表格 vs 普通文本 |
| `_append_table_element` | 新增：MD table → CardKit column_set |

## License

MIT
