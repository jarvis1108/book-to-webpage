# 组件范式库（②组件层）

每个组件是一段可复制的 HTML/CSS/JS，带 `{{...}}` 数据槽位。Agent 按 JSON 内容契约的 `signature`/请求模式挑选对应范式，填入数据，拼进 base.html。

美学锁定：宋体正文（Songti SC / Noto Serif CJK SC）、PingFang SC 标题、暖纸色调（见 base.html CSS 变量）。

---

## 核心固定组件（每页必有）

### Hero + 一句话总论

```html
<section class="hero" aria-labelledby="title">
  <div>
    <p class="eyebrow">{{eyebrow}}</p>
    <h1 id="title">{{title}}</h1>
    <p class="lede">{{lede}}</p>
  </div>
  <aside class="thesis">
    <strong>一句话总论</strong>
    <span>{{thesis}}</span>
  </aside>
</section>
```

### 分区导航（Sticky Tab）

`{{component_styles}}` 里插入：`.tabs{grid-template-columns:repeat({{tab_count}},1fr);}`

```html
<button class="tab" aria-selected="{{first}}" data-tab="{{id}}">{{label}}</button>
```

`{{component_scripts}}` 里插入：

```js
document.querySelectorAll(".tab").forEach(t=>{
  t.onclick=()=>{
    document.querySelectorAll(".tab").forEach(x=>x.setAttribute("aria-selected","false"));
    document.querySelectorAll(".panel").forEach(p=>p.classList.remove("active"));
    t.setAttribute("aria-selected","true");
    document.getElementById(t.dataset.tab).classList.add("active");
  };
});
```

---

## 主题特色组件（theme 模式，按主题性质挑 1-2 个）

### signature="causal_chain" — 因果链探索器

适用：因→果链条主题。每个节点一个按钮，点击展开解释。

`{{component_styles}}` 里插入：`.chain{grid-template-columns:repeat({{chain_count}},1fr);}`

```html
<div class="chain" aria-label="因果链">
  {{#chain_items}}
  <button class="chain-step {{#first}}active{{/first}}" data-step="{{id}}">{{label}}</button>
  {{/chain_items}}
</div>
<div class="explain" id="chainText">{{chain_first_body}}</div>
```

`{{component_scripts}}` 里插入：

```js
const steps={ {{#chain_items}}"{{id}}":"{{body_escaped}}"{{^last}},{{/last}}{{/chain_items}} };
document.querySelectorAll(".chain-step").forEach(b=>{
  b.onclick=()=>{
    document.querySelectorAll(".chain-step").forEach(x=>x.classList.remove("active"));
    b.classList.add("active");
    document.getElementById("chainText").textContent=steps[b.dataset.step];
  };
});
```

### signature="type_selector" — 类型选择器

适用：多并列类别，点类别出分析。

```html
<div class="split">
  <div class="city-buttons" aria-label="{{selector_label}}">
    {{#options}}
    <button class="city-btn {{#first}}active{{/first}}" data-city="{{id}}">{{label}}</button>
    {{/options}}
  </div>
  <article class="result" id="cityResult">
    <div class="label">{{result_label}}</div>
    <h3>{{first_title}}</h3>
    <p>{{first_body}}</p>
    <p><strong>看点：</strong>{{first_watch}}</p>
  </article>
</div>
```

`{{component_scripts}}` 里插入：

```js
const cities={ {{#options}}"{{id}}":{title:"{{title_escaped}}",body:"{{body_escaped}}",watch:"{{watch_escaped}}"}{{^last}},{{/last}}{{/options}} };
document.querySelectorAll(".city-btn").forEach(b=>{
  b.onclick=()=>{
    document.querySelectorAll(".city-btn").forEach(x=>x.classList.remove("active"));
    b.classList.add("active");
    const c=cities[b.dataset.city];
    document.getElementById("cityResult").innerHTML='<div class="label">{{result_label}}</div><h3>'+c.title+'</h3><p>'+c.body+'</p><p><strong>看点：</strong>'+c.watch+'</p>';
  };
});
```

### signature="matrix" — 对比矩阵

适用：多维权衡类主题。打分表或特征对比。

```html
<table class="matrix">
  <thead>
    <tr><th>选项</th>{{#dimensions}}<th>{{.}}</th>{{/dimensions}}</tr>
  </thead>
  <tbody>
    {{#rows}}
    <tr><td>{{option}}</td>{{#scores}}<td>{{.}}</td>{{/scores}}</tr>
    {{/rows}}
  </tbody>
</table>
```

交互增强（hover 高亮行）：`{{component_scripts}}` 里插入：

```js
document.querySelectorAll(".matrix tbody tr").forEach(r=>{
  r.onmouseenter=()=>r.style.background="rgba(194,135,45,.12)";
  r.onmouseleave=()=>r.style.background="";
});
```

### signature="timeline" — 时间线

适用：历史演进类主题。

```html
<div class="timeline">
  {{#events}}
  <div class="timeline-item">
    <div class="time">{{time}}</div>
    <h3>{{title}}</h3>
    <p>{{body}}</p>
  </div>
  {{/events}}
</div>
```

### signature="decision_tree" — 决策树

适用：分支判断类主题。

```html
<div class="decision">
  {{#branches}}
  <details {{#first_open}}open{{/first_open}}>
    <summary>{{question}}</summary>
    <div class="branch">
      <p><strong>{{choice_label}}</strong>：{{reasoning}}</p>
      {{#sub_branches}}<p>{{label}}：{{body}}</p>{{/sub_branches}}
    </div>
  </details>
  {{/branches}}
</div>
```

### signature="questions" — 问题清单

适用：读者自检/行动指南。

```html
<div class="questions">
  {{#questions}}
  <article class="question">
    <div>
      <h3>{{title}}</h3>
      <p>{{hint}}</p>
    </div>
  </article>
  {{/questions}}
</div>
```

---

## 概述专用组件（仅 overview 模式用）

### 核心框架卡片网格

从 INDEX.md Core Frameworks 取材。

```html
<div class="grid">
  {{#frameworks}}
  <article class="note">
    <h3>{{name}}</h3>
    <p>{{oneLine}}</p>
    <p class="when">适用：{{when}}</p>
  </article>
  {{/frameworks}}
</div>
```

### 章节结构图

从 Chapter Index 取材，按 Part 分组。

```html
<div class="chapter-list">
  {{#chapters}}
  <div class="chapter-row">
    <span class="ch-label">{{label}}</span>
    <div>
      <h3>{{title}}</h3>
      <p class="ch-role">{{role}}</p>
    </div>
  </div>
  {{/chapters}}
</div>
```

### 概念星图

从 Topic Index 取材。点概念按钮高亮关联章节。

```html
<div class="concept-grid">
  {{#concepts}}
  <button class="concept-btn" data-links="{{links_attr}}">{{term}}</button>
  {{/concepts}}
</div>
```

`{{component_scripts}}` 里插入：

```js
document.querySelectorAll(".concept-btn").forEach(b=>{
  b.onclick=()=>{
    b.classList.toggle("active");
    // 高亮关联章节行（用 data-links 属性匹配 .chapter-row 的 data-ch）
    const links=(b.dataset.links||"").split(",");
    document.querySelectorAll(".chapter-row").forEach(r=>{
      r.style.background=links.includes(r.dataset.ch)&&b.classList.contains("active")
        ?"rgba(194,135,45,.15)":"";
    });
  };
});
```

### 决策规则速览

从 cheatsheet.md 取材。

```html
<ul class="rules">
  {{#decisionRules}}
  <li>
    <strong>当 {{when}}</strong> → {{do}}
    <em>（{{because}}）</em>
  </li>
  {{/decisionRules}}
</ul>
```

---

## 页面拼装流程

1. 读 base.html 骨架
2. 判定请求模式（overview / theme / chapter）
3. 按 JSON 的 `signature` / 模式挑选对应组件范式
4. 填入 JSON 数据，生成 HTML 片段
5. 写衔接层（段首引语、过渡句、卡片标题口语化重写）
6. 用 `{{component_styles}}` 插入组件专用 CSS（tabs 列数、chain 列数…）
7. 用 `{{component_scripts}}` 插入组件交互 JS
8. 拼成单文件，输出 `<书名>-<主题>.html`
