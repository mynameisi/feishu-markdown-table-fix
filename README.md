# feishu-markdown-table-fix 🔧

修复 Hermes Agent 向飞书发送 Markdown 时表格/代码块/标题显示为源码的问题。

## 问题

飞书原生不渲染 Markdown 的表格（`|...|`）、标题（`#`）、代码块（```），全部显示为纯文本源码。

## 解决方案

修改 `/opt/hermes/gateway/platforms/feishu.py`，将 Markdown 内容转为 **CardKit 2.0 interactive card**：

- **表格** → `column_set` 布局，表头灰底加粗，数据行交替底色
- **代码块** → `div` + `lark_md` 标签
- **标题** → 预处理为加粗文本

## 工作流

1. 按 `SKILL.md` 修改 `feishu.py`
2. 重启网关（`kill -TERM` + s6 自动拉起）
3. 发送 **Markdown 渲染测试** 标准模板到飞书 DM
4. 紧跟检查清单，请用户逐项确认渲染效果

**触发词：**「检测 markdown render」「检测一下我现在的 markdown render 怎么样了？」

## 文件

| 文件 | 说明 |
|:-----|:-----|
| `SKILL.md` | 完整修改步骤 + 重启流程 + 渲染测试模板 |
| `feishu.py` | 完整修改后文件 |
| `MODIFICATIONS.md` | 详细修改步骤 |
| `references/complete-code.md` | 代码参考 |

## License

MIT
