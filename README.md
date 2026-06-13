# book-to-webpage

把书（PDF/Markdown 等）变成单个自包含的交互式 HTML 学习页。

## 用法（在 Claude Code 里）
- "用 book-to-webpage 处理 xxx.pdf" → 拆全书成知识库 + 引导生成主题页
- "概述一下这本书" → 全书概览页
- "就'房地产'这个主题做一页" → 单主题深度页

## 依赖
- Python 3
- PDF 解析（任一）：`brew install poppler`（pdftotext，推荐）或 `pip3 install pypdf2 pdfminer.six`
- 跑 `python3 scripts/extract.py --check` 查看各格式解析器是否就绪

## 说明
阶段 1 的提取器与拆解流程复制自 book-to-skill，但产出本地知识库（非 skill）。