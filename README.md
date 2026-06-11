# feishu-markdown-table-fix

修复 Hermes Agent 向飞书发送 Markdown 时表格/代码块/标题显示为源码的问题。

## 问题

飞书原生不渲染 Markdown 的表格（`|...|`）、标题（`#`）、代码块（```），全部显示为纯文本源码。

## 解决方案

修改 `/opt/hermes/gateway/platforms/feishu.py`，将 Markdown 内容转为 **CardKit 2.0 interactive card**。

## 工作流

1. 按 `SKILL.md` 修改 `feishu.py`
2. 重启网关（`kill -TERM` + s6 自动拉起）
3. 发送 **Markdown 渲染测试** 标准模板到飞书 DM 验证

## 文件

| 文件 | 说明 |
|:-----|:-----|
| `SKILL.md` | 完整修改步骤 + 重启流程 + 渲染测试模板 |
| `references/complete-code.md` | 完整代码参考 |
