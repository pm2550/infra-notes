# CLAUDE.md — 行为规则

## 文件操作
- **修改文件必须用 Edit 工具**，不许用 bash sed/awk/python 写文件
- **新建文件用 Write 工具**，不许用 bash echo/cat heredoc
- bash 只用于系统命令和无法用内置工具完成的操作
