---
name: book-to-webpage
description: 把书（PDF/Markdown等）拆解成本地知识库，再据用户请求（全书概述/某主题/某章）生成可交互的HTML学习页。页面全程可点可探索。当用户想让一本书的某个主题以美观交互网页呈现、或做全书概览页时使用。
---

# book-to-webpage

把书变成可交互的学习页。**两阶段**：先拆全书成本地知识库（`<书名>.kb/`），再据请求生成单页 HTML。

## 哲学

- **展示层是核心差异化**：不是读书摘要，是把书中某主题做成可点击、可探索的页面。组件可交互，内容成叙事。
- **拆解逻辑复用 book-to-skill**：提取器和拆解流程整体复制，不重新发明；但产出本地知识库（`<书名>.kb/`），**不生成 skill、不安装到 ~/.claude/skills/**。
- **散布内容要全面聚合**：某主题散布全书时，用 Topic Index 定位所有相关章，全部纳入，不遗漏。

## 两种运行入口

1. **首次处理一本书**：跑阶段 1，生成 `<书名>.kb/` 知识库。
2. **已有知识库后生成网页**：跑阶段 2，按请求产出单页 HTML。

---

## 文件结构

本 skill 附带以下预制资产，Agent 在渲染时引用：

| 文件 | 作用 | 用法 |
|------|------|------|
| `templates/base.html` | ①骨架层：页面壳、CSS 变量、布局、移动端断点、所有组件基础样式 | 抄，替换 `{{...}}` 槽位 |
| `templates/components.md` | ②组件层：每个组件的 HTML/CSS/JS 范式 + 数据槽位 | 按 signature 挑范式，抄+填数据 |
| `scripts/extract.py` | 提取器入口（复制自 book-to-skill） | 阶段 1 跑 |
| `scripts/extractor/` | 提取器包（复制自 book-to-skill） | 阶段 1 用 |

---

## 阶段 1：全书拆解（复制 book-to-skill 流程，改产出物）

**前提：** 用户提供了书文件（PDF/Markdown/EPUB等）。

### Step 1.0 校验输入

支持格式同 book-to-skill：`.pdf`、`.epub`、`.docx`、`.txt`、`.md`、`.html`、`.rtf`、`.mobi`、`.azw`。

无输入则提示：`book-to-webpage 需要一本书的文件路径。`

### Step 1.1 提取全文

```bash
SCRIPT="$HOME/.claude/skills/book-to-webpage/scripts/extract.py"
python3 "$SCRIPT" <书文件路径> --mode text
```

产出 `<workdir>/book_skill_work/full_text.txt` + `metadata.json`。读 `metadata.json` 确认页数、字数、token、章节数。

如果提取失败，跑 `python3 "$SCRIPT" --check` 诊断依赖，给安装建议。

### Step 1.2 成本预估并确认

报页数/字数/预估 token 与价格，**等用户确认**再继续。

### Step 1.3 分析结构（复制 book-to-skill Step 3）

读 full_text 前 8000 字 + metadata 章节检测，识别：
- 书名、作者
- 章节结构（中文"第X章"或英文"Chapter N"、ToC）
- 核心领域

### Step 1.4 REPL 式切片拆解

**全书 >50k token 时禁止整体读入。** 用 grep 定位章节边界，sed 只切当前要写的那章：

```bash
grep -n -E "^\s*(第[一二三四五六七八九十百零0-9]+[章节回]|Chapter [0-9]+)" full_text.txt | head -60
sed -n '<start>,<end>p' full_text.txt
```

### Step 1.5 生成章节摘要（复制 book-to-skill Step 7 模板）

每章生成 `<书名>.kb/chapters/ch<NN>-<slug>.md`，字段：
- **Core Idea**（1-2 句）
- **Frameworks Introduced**（命名框架 + 何时用 + 怎么用，**保留作者精确命名**）
- **Key Concepts**（5-10 个术语，各一句定义）
- **Mental Models**（"用 X 当 Y"）
- **Anti-patterns**（要避免什么 + 为什么失败）
- **Worked Example**（重现作者亲算的一个例子）
- **Key Takeaways**（3-7 条）
- **Connects To**（关联其他章/外部概念）

自适应深度（默认中文社科书 = text + study = 1000-1800 token/章）：

| | reference | study |
|---|---|---|
| text | 800-1200 | 1000-1800 |
| technical | 1200-1800 | 2000-3000 |

### Step 1.6 生成支撑文件（复制 book-to-skill Step 8 规范）

- `glossary.md`：全部术语序排，`**术语** — 定义（第N章）`，≤1500 token
- `patterns.md`：全部技术/模式，`## 模式名 / 何时用 / 怎么做 / 权衡`，≤2000 token
- `cheatsheet.md`：**决策层**——决策规则（当X做Y因为Z）、决策树、权衡矩阵、阈值，≤1200 token

### Step 1.7 生成 INDEX.md（取代 book-to-skill 的 SKILL.md，不注册为 skill）

**关键改动**：book-to-skill 这里生成 SKILL.md；我们生成 `INDEX.md`（知识库导航）。含：
- 书名/作者/页数/章数/生成日期
- **Core Frameworks**（~2000 token：最重要命名框架）
- **Chapter Index**（表：章号→标题→关键框架）
- **Topic Index**（序排：术语/框架 → 出现在哪几章）★阶段 2 关键
- 支撑文件链接

### Step 1.8 质量规则

提取结构非摘要；保留作者精确命名；密度优先于完整度；从业者口吻（"用 X 当 Y"）；前置重要内容；章节按需加载；绝不照抄原文。

### 产物结构

```
<书名>.kb/
  INDEX.md          ← 知识库导航（常驻核心 ~4K）
  chapters/ch<NN>-*.md  ← 章节摘要
  glossary.md  patterns.md  cheatsheet.md
  full_text.md     ← 原文（供阶段 2 grep）
  metadata.json
```

**这是本地知识库，不写到 ~/.claude/skills/，不生成 SKILL.md。**

---

## 阶段 2：请求 → 网页

### 前提

阶段 1 已生成 `<书名>.kb/`。如果不存在，先跑阶段 1。

### 运行模式：常驻核心 + 按需深挖

INDEX.md 是常驻小核心（~4K），含 Core Frameworks + 章节索引 + 主题索引。章节文件按需加载，不全读。这与 book-to-skill 的查询机制一致；区别仅在我们把"文字回答"换成"交互 HTML"。

### Step 2.1 判定请求模式（三选一）

| 模式 | 触发条件 | 取材来源 | 产出 |
|------|---------|---------|------|
| ① 全书概述 | 用户说"概述全书/讲讲这本书"，无主题 | 读 INDEX.md Core Frameworks + cheatsheet | 概述页（骨架视图） |
| ② 主题聚合（默认） | 用户指定主题 / 说"做一页xxx" | 查主题索引 → 两级检索（§2.2） | 主题深度页 |
| ③ 单章深读（可选） | 用户指定章号 | 读该章摘要 + 该章原文 | 单章页 |

用户指定主题时，**先 grep full_text.md 验证书中确有相关内容**；命中 0 → 明确告知"本书未涉及该主题，建议尝试以下相关主题：…"，绝不瞎编。

### Step 2.2 主题聚合的两级检索（关键差异化）

```
第一级（结构化）：读 chapters/ 里主题索引指向的章节摘要
                  → 框架/概念/反模式等结构化要点
第二级（原汁原味）：grep full_text.md 原文散落表述
                  → 具体论述/作者原话/数据
合并去重重组 → 完整主题视图
```

**两级都要**：第一级保结构，第二级保全面。这是相对 book-to-skill（只读摘要）的强化，确保主题页囊括书中所有对该主题的论述。

### Step 2.3 重组（散碎 → 流畅叙事）

1. **去重**：同一观点在多章重复的，合并成一处
2. **归类**：按因果/对比/演进逻辑排，不按章节顺序
3. **改写**：用自己的话重述，但**保留作者精确命名**（框架名/术语不改）
4. **补衔接**：观点间加过渡句，形成可读叙事，不是观点清单

### Step 2.4 结构化为 JSON 内容契约

按模式产出 JSON。`meta.mode` 字段标识模式，决定后续组件挑选。

**theme 模式 JSON：**

```json
{
  "meta": {"book": "...", "author": "...", "mode": "theme", "theme": "...", "sources": ["ch03","ch05"]},
  "hero": {"eyebrow": "...", "title": "...", "lede": "...", "thesis": "..."},
  "sections": [
    {
      "id": "past", "label": "过去20年",
      "lead": "...",
      "notes": [{"title": "需求被释放", "body": "..."}],
      "signature": "causal_chain",
      "chain": [{"id": "tax", "label": "分税制", "body": "..."}]
    }
  ],
  "questions": [{"title": "...", "hint": "..."}]
}
```

- `sections[].signature`：决定该分区用哪个特色组件（`causal_chain` / `type_selector` / `matrix` / `timeline` / `decision_tree` / `questions`）
- `sections[].lead`：段首引语（衔接层，Agent 现写）

**overview 模式 JSON：**

```json
{
  "meta": {"book": "...", "author": "...", "mode": "overview"},
  "hero": {"eyebrow": "...", "title": "...", "thesis": "..."},
  "frameworks": [{"name": "土地财政", "oneLine": "...", "when": "..."}],
  "chapterMap": [{"ch": "ch01", "title": "...", "role": "..."}],
  "conceptMap": [{"term": "土地财政", "links": ["ch03","ch05"]}],
  "decisionRules": [{"when": "...", "do": "...", "because": "..."}]
}
```

每个顶层数组对应一个概述专用组件。

### Step 2.5 挑组件

**theme 模式：**
- **核心固定**（每页必有）：Hero + 一句话总论 + 分区导航（sticky Tab）
- **主题特色**（按 signature 挑 1-2 个）：
  - 因果链类 → `causal_chain`
  - 多并列类别 → `type_selector`
  - 多维权衡 → `matrix`
  - 历史演进 → `timeline`
  - 分支判断 → `decision_tree`
  - 读者自检 → `questions`
  - 多数主题配 2 个（一主一辅）

**overview 模式：**
- 核心固定 + 概述专用：`frameworks` 卡片网格 + `chapterMap` + `conceptMap` + `decisionRules`

从 `templates/components.md` 取对应范式，填入 JSON 数据。

### Step 2.6 三层渲染产出 HTML

**三层叠加（关键！组件不够，还要衔接层）：**

| 层 | 来源 | 谁做 | 占页面比例 |
|----|------|------|-----------|
| ① 骨架 | `templates/base.html` | **照抄**，替换 `{{lang}}`/`{{title}}` 等 | ~20% |
| ② 组件 | `templates/components.md` 对应范式 | **照抄 + 填数据** | ~35% |
| ③ 衔接 | Agent 现写，受下方规则约束 | **生成**：段首引语、过渡句、卡片标题重写 | ~45% |

**衔接层写作规则（必须遵守，决定页面质量）：**

1. **段首必有引语**：每个分区/Panel 开头一句提纲挈领的话，不直接堆组件
2. **卡片标题口语化重写**：不照搬原文小标题，改成读者一眼懂的短句（如"土地变成财源"而非"地方政府土地出让收入机制"）
3. **正文综合而非拼接**：grep 来的散碎原文必须去重合并改写成连贯段落，禁止直接堆多段原文
4. **组件间过渡**：组件前后用一两句话承接，不让交互组件孤立出现
5. **配色强调克制**：红字只用在 eyebrow 和总论标记，不滥用

**拼装流程：**

1. 读 `templates/base.html` 骨架
2. 设置 `{{lang}}`（中文书="zh-CN"）
3. 设置 `{{title}}` = `《书名》- 主题`
4. 把 Hero HTML 片段填入 `{{hero}}`
5. 把 Tab 按钮填入 `{{tabs}}`（`{{component_styles}}` 里设置 tabs 列数）
6. 把各 Panel HTML 填入 `{{panels}}`
7. 把 `{{component_styles}}` 设为组件专用 CSS
8. 把 `{{component_scripts}}` 设为所有交互 JS
9. 把 `{{footer_note}}` 设为脚注
10. 输出 `<书名>-<主题>.html`

### Step 2.7 自检

逐项验证：
- [ ] 所有 Tab 可切换、所有按钮可点且有响应
- [ ] 无残留占位符（无 `{{...}}`、TODO、xxx、空数据块）
- [ ] JSON 里每个声明的组件都真实渲染在 HTML 中
- [ ] 移动端（≤880px / ≤520px）不溢出
- [ ] 衔接层齐全（每分区有段首引语、卡片标题口语化、组件间有过渡）

### Step 2.8 清理工作目录

```bash
rm -rf "<workdir>/book_skill_work"
```

---

## 美学基线（锁定参考例子）

所有 HTML 统一套用以下设计语言，CSS 变量定义在 `templates/base.html`：

- **字体**：宋体正文（Songti SC / Noto Serif CJK SC / Georgia），PingFang SC 标题
- **色调**：暖纸色纸面 `#f8f2e7`，强调色红 `#b9422f` / 蓝 `#245f73` / 绿 `#4d6f48` / 金 `#c2872d`
- **背景**：径向渐变金色光晕 + 线性渐变纸面
- **单文件、零依赖**：CSS/JS 全部内联
- **移动端自适应**：≤880px 单列、取消 sticky；≤520px 字号缩减、chain 单列
- **无外部字体/图标/CDN**：仅用系统字体

---

## 错误处理

| 场景 | 处理 |
|------|------|
| PDF 多级回退全失败 | 报错 + 跑 `python3 scripts/extract.py --check` + 安装建议 |
| 提取出空文本 | 提示"可能是扫描版，建议用 Nutrient OCR" |
| 用户主题书里没有 | 明确告知"本书未涉及该主题" + 列相关候选（grep 相邻关键词） |
| 主题内容过少（<3处提及） | 提示"仅 N 处提及，内容较少，建议换主题或合并相关主题"，不硬凑 |

---

## 不在本次范围

- 语义搜索（当前只用关键词 grep）
- 更多组件 / 更复杂交互
- 多主题站点（当前只做单主题单页）
- 展示层换肤（当前锁定参考例子美学）
