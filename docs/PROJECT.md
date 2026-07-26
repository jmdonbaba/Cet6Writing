# CET-6 作文助手 — 项目文档

## 目录

1. [项目概述](#1-项目概述)
2. [功能详述](#2-功能详述)
3. [架构设计](#3-架构设计)
4. [数据流](#4-数据流)
5. [Prompt 工程](#5-prompt-工程)
6. [状态管理与持久化](#6-状态管理与持久化)
7. [UI 组件树](#7-ui-组件树)
8. [安全与容错](#8-安全与容错)
9. [已知限制与改进方向](#9-已知限制与改进方向)

---

## 1. 项目概述

**CET-6 作文助手**是一款面向大学英语六级考生的纯前端写作备考工具。用户输入作文开头句后，工具调用 DeepSeek Chat API 生成符合六级评分标准的三段式高分范文，同时提供中文译文和深度词汇讲解，并支持作文和词汇的双轨收藏管理。

### 1.1 技术定位

- **类型**：单文件 Web 应用（SPA）
- **运行方式**：双击 `index.html` 即可在浏览器中使用
- **外部依赖**：仅 DeepSeek Chat API（网络请求）
- **本地依赖**：浏览器 localStorage（数据持久化）

### 1.2 适用场景

- CET-6 考生日常写作练习
- 范文积累与背诵
- 高级词汇的系统性学习
- 写作模板和句型的归纳

---

## 2. 功能详述

### 2.1 作文生成

**入口**：输入框 + 生成按钮，或点击 AI 推荐题目

**处理流程（三阶段流水线）**：

```
用户输入/点击句子
    ↓
【阶段 1】构建作文生成 Prompt（仅要求 essay + translation）
    ↓
调用 DeepSeek API（temperature=0.7, max_tokens=4096）
    ↓
解析 JSON → { essay, translation }
    ↓
【阶段 2】构建词汇提取 Prompt，传入作文全文
    ↓
调用 DeepSeek API（temperature=0.3, max_tokens=2048）
    ↓
解析 JSON → [{ word, usage }]（全部值得讲解的词，不设上限）
    ↓
【阶段 3】对每个词并行调用查词 API（每批 5 个并发）
    ↓
归一化词汇数据 → vocabulary 数组
    ↓
渲染到结果区
```

**生成内容**：
- `essay`：三段式英文作文，以给定句子开头，150–200 词
- `translation`：全文中文翻译
- `vocabulary`：作文中全部值得讲解的词汇详解数组（通过独立查词 API 逐一获取，不限数量）

**生成状态**：
- Loading 态：旋转动画 + 分阶段进度提示（撰写作文 → 分析词汇 → 详解词汇 N/M）
- 错误态：红色提示信息 + 控制台日志
- 生成中禁用按钮防重复提交

### 2.2 试卷格式展示

生成结果上方展示完整 CET-6 真题格式卡片：

```
Writing (30 minutes)
Directions: For this part, you are allowed 30 minutes to write
an essay that begins with the sentence "..." You can make
comments, cite examples or use your personal experiences to
develop your essay.
You should write at least 150 words but no more than 200 words.
You should copy the sentence given in quotes at the beginning
of your essay.
```

开头句在引号内高亮显示。作文正文开头句用蓝紫色背景 + 双引号包裹。

### 2.3 作文展示

**Tab 切换**（3 个标签页）：

| 标签 | 内容 |
|------|------|
| 作文 | 三段式英文范文，开头句高亮，底部显示词数 |
| 译文 | 三段落中文翻译 |
| 词汇 | 作文中全部高级词汇的卡片列表（不限数量） |

### 2.4 词汇讲解

每个词汇卡片包含：

```
┌───────────────────────────────────────────┐
│  remarkable  /r??mɑ?rk?bl/   adj.  [收藏] │
│  释义：显著的，非凡的                       │
│  文中用法："has gained remarkable momentum"│
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  │
│  1. The proliferation of misinformation   │
│     could exacerbate social divisions.    │
│     虚假信息的泛滥可能加剧社会分裂。         │
│  2. ...                                   │
│  3. ...                                   │
└───────────────────────────────────────────┘
```

**选词标准**：
1. 六级/考研/雅思级别高级词汇（如 exacerbate, imperative, formidable）
2. 地道的高级短语搭配（如 hinge on, grapple with）
3. 熟词的生僻义项（如 address 作"解决"）
4. 学术写作常用词（如 consequently, nevertheless）
5. **禁止**基础词汇（如 important, good, bad, make, do）

**例句要求**：
- 每个例句含其他高级词汇或复杂句式
- 适合直接用于六级作文
- 英文原文 + 中文翻译紧随其后

### 2.5 AI 推荐题目

- 页面加载时自动调用 DeepSeek API 生成 5 条 CET-6 风格开头句
- 点击「换一批」刷新
- 覆盖不同话题领域，尽量不重复
- 带防竞态保护（`fetchingExamples` 标志位）
- 加载失败显示重试提示

### 2.6 智能查词

**入口**：查词输入框 + 查词按钮（或回车）

**功能**：
1. **变形识别**：输入 `automating` → 识别为 automate 的现在分词 → 展示原形查询结果
2. **词典信息**：音标、词性、多释义（带词性标注）、其他变形
3. **例句**：3 个高难度例句 + 中文翻译
4. **收藏**：一键存入词汇收藏夹
5. **收藏状态同步**：已收藏的词显示「已收藏」按钮

**变形类型支持**：
| 类型 | 示例输入 | 识别结果 |
|------|----------|----------|
| 现在分词 | automating | automate |
| 过去式 | automated | automate |
| 过去分词 | written | write |
| 复数 | criteria | criterion |
| 比较级 | better | good |
| 最高级 | best | good |
| 三单 | runs | run |

### 2.7 作文收藏

**存储结构**：
```json
{
  "id": "唯一标识",
  "topic": "作文开头句",
  "essay": "完整英文作文",
  "translation": "中文译文",
  "vocabulary": [...],
  "pinned": false,
  "createdAt": "2026-07-23 14:30:00"
}
```

**操作**：
- **查看**：关闭弹窗，回填到结果区
- **置顶/取消置顶**：置顶项背景加深，始终排在列表最前；取消置顶后按原始收藏时间归位
- **删除**：带 confirm 确认
- 数据键名：`cet6_favorites`

### 2.8 词汇收藏

**存储结构**：
```json
{
  "id": "唯一标识",
  "word": "原形单词",
  "phonetic": "音标",
  "pos": "词性",
  "meaning": "释义合集",
  "definitions": ["n. 释义1", "v. 释义2"],
  "examples": [{"en": "英文", "cn": "中文"}, ...],
  "forms": "其他变形说明",
  "pinned": false,
  "createdAt": "2026-07-23 14:30:00"
}
```

**界面交互**：
- **紧凑列表**：显示词、音标、词性、简短释义
- **点击展开**：显示其他变形、3 条例句（含翻译）、收藏时间
- **三点菜单**：置顶/取消置顶 + 删除
- 数据键名：`cet6_vocab_favorites`

### 2.9 双收藏弹窗系统

- 主页面仅显示两个入口按钮：「词汇收藏 (N)」和「作文收藏 (N)」
- 点击按钮打开对应弹窗
- 弹窗支持：点击遮罩关闭、×按钮关闭、**Escape 键关闭**
- 弹窗内使用事件委托处理所有交互

---

## 3. 架构设计

### 3.1 整体架构

```
┌─────────────────────────────────────────┐
│              index.html                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │  HTML    │ │  CSS     │ │  JS      │ │
│  │  结构层   │ │  样式层  │ │  逻辑层   │ │
│  └──────────┘ └──────────┘ └──────────┘ │
│                     │                   │
│              ┌──────┴──────┐            │
│              │  DeepSeek   │            │
│              │  Chat API   │            │
│              └─────────────┘            │
│                     │                   │
│              ┌──────┴──────┐            │
│              │ localStorage│            │
│              └─────────────┘            │
└─────────────────────────────────────────┘
```

### 3.2 JS 内部模块划分

| 模块 | 行号范围 | 职责 |
|------|----------|------|
| 配置 | ~5 行 | API URL、localStorage 键名 |
| 工具函数 | ~30 行 | `esc()`、`escAttr()`、`fmtTime()`、`toast()`、`err()` |
| 例句（API 实时生成）| ~50 行 | `fetchExamples()` |
| DOM 引用 | ~20 行 | 所有 `getElementById` / `querySelectorAll` |
| API 调用 | ~200 行 | `generate()`（三阶段流水线）、`buildPrompt()`、`buildVocabExtractPrompt()`、`lookupWordData()` |
| 渲染层 | ~120 行 | `renderEssay()`、`renderTrans()`、`renderVocab()`、`showResult()` |
| 作文收藏 CRUD | ~80 行 | `getFavs()`、`setFavs()`、`addFav()`、`delFav()`、`pinFav()`、`viewFav()`、`renderFavs()` |
| 查词 | ~90 行 | `searchWord()` + 渲染 |
| 词汇收藏 CRUD | ~90 行 | `getVFavs()`、`setVFavs()`、`addVFav()`、`delVFav()`、`pinVFav()`、`renderVFavs()` |
| 弹窗管理 | ~30 行 | `openModal()`、`closeModal()`、`closeAllMenus()` |
| 事件绑定 | ~100 行 | 所有事件监听器 |
| 初始化 | ~5 行 | 页面加载时调用 |

### 3.3 CSS 变量系统

```css
--primary: #4f46e5;      /* 主色（蓝紫） */
--primary-hover: #4338ca; /* 主色悬浮 */
--primary-light: #eef2ff; /* 主色浅底 */
--bg: #f8fafc;            /* 页面背景 */
--card-bg: #ffffff;       /* 卡片背景 */
--text: #1e293b;          /* 正文色 */
--text-secondary: #64748b; /* 次级文字 */
--border: #e2e8f0;        /* 边框色 */
--danger: #ef4444;        /* 删除/错误色 */
--radius: 12px;           /* 圆角 */
--shadow: ...;            /* 投影 */
```

---

## 4. 数据流

### 4.1 作文生成流（三阶段）

```
Phase 1: buildPrompt(topic)
       → fetch(DEEPSEEK_URL, {model, messages, temperature:0.7, max_tokens:4096})
       → parseJSON → { essay, translation }

Phase 2: buildVocabExtractPrompt(essay)
       → fetch(DEEPSEEK_URL, {model, messages, temperature:0.3, max_tokens:2048})
       → parseJSON → [{ word, usage }]  (全部值得讲解的词汇)

Phase 3: 对每个词 lookupWordData(word)  （每批 5 个并发）
       → fetch(DEEPSEEK_URL, {model, messages, temperature:0.3, max_tokens:1024})
       → parseJSON → { word, phonetic, pos, definitions, forms, sentences }
       → 归一化 → { word, phonetic, pos, meaning, usage, examples }

showResult(topic, data)
  ├─ renderEssay(topic, data.essay)      → #tab-essay innerHTML
  ├─ renderTrans(data.translation)        → #tab-translation innerHTML
  ├─ renderVocab(data.vocabulary)         → #tab-vocabulary innerHTML
  ├─ updateFavBtn()                       → 刷新收藏按钮状态
  └─ switchTab('essay')                   → 默认展示作文
```

### 4.2 查词流

```
Input → searchWord(word)
      → fetch(DEEPSEEK_URL, {model, messages, temperature, max_tokens})
      → parseJSON(content)
      → info.baseForm → displayWord (原形优先)
      → render result (变形提示 + 词头 + 释义 + 变形 + 例句含翻译)
      → check favorite status → 设置按钮初始状态
```

### 4.3 收藏同步流

任一收藏操作（添加/删除）触发：

```
addFav / delFav / addVFav / delVFav
  → setFavs / setVFavs (localStorage 持久化)
  → renderFavs / renderVFavs (弹窗内列表刷新)
  → [作文] updateFavBtn() (结果区收藏按钮状态)
  → [词汇] updateAllVocabButtons()
       ├─ 查词结果按钮状态
       └─ 作文词汇栏所有按钮状态
```

---

## 5. Prompt 工程

### 5.1 作文生成 Prompt

```
角色：专业 CET-6 阅卷老师
硬性要求：
  - 三段式结构（Paragraph 1/2/3 标记）
  - 150-200 词
  - 给定句子开头，一字不改
输出：严格 JSON（仅含 essay + translation）
```

### 5.2 词汇提取 Prompt

```
角色：CET-6 英语教学专家
输入：完整作文全文
选词标准：
  - 六级/考研/雅思级 + 高级搭配 + 熟词僻义 + 学术词
  - 禁止基础词（important, good, bad 等）
  - 不限数量，全面覆盖所有值得讲解的词汇
输出：严格 JSON 数组 [{ word, usage }]
```

### 5.3 词汇详解 Prompt

```
角色：英语词典专家
输入：单个词汇
变形判断：先识别变形类型，还原为原形
词典信息：音标、词性、多释义、其他变形
例句要求：含高级词汇，适合作文，英文 + 中文翻译（必须恰好3个）
输出：严格 JSON
```

### 5.4 推荐题目生成 Prompt

```
角色：CET-6 命题专家
要求：
  - 5 个句子，覆盖不同话题
  - 15-30 词，语法正确，表达地道
  - 每个可引出观点讨论
输出：JSON 数组
温度：1.2（增加多样性）
```

### 5.5 查词 Prompt

```
角色：英语词典专家
变形判断：先识别变形类型，还原为原形
词典信息：音标、词性、多释义、其他变形
例句要求：含高级词汇，适合作文，英文 + 中文翻译
输出：严格 JSON
温度：0.3（保持准确）
```

---

## 6. 状态管理与持久化

### 6.1 localStorage 键

| 键名 | 数据类型 | 说明 |
|------|----------|------|
| `cet6_api_key` | `string` | DeepSeek API Key |
| `cet6_favorites` | `Array<EssayFav>` | 作文收藏列表 |
| `cet6_vocab_favorites` | `Array<VocabFav>` | 词汇收藏列表 |

### 6.2 读写安全

- **读取**：`getFavs()` / `getVFavs()` 带 try/catch + `Array.isArray` 类型校验，异常返回 `[]`
- **写入**：`setFavs()` / `setVFavs()` 带 try/catch，失败时 toast 提示用户
- **配额**：捕获 `QuotaExceededError`，提示清理空间

### 6.3 DOM 状态同步

不在 localStorage 中的运行时状态通过 DOM 属性同步：
- `$wordSearchResult._wordData`：当前查词结果的完整数据（搜索出错时设为 `null`）
- `.favorited` CSS class：按钮「已收藏」状态（视觉 + 逻辑双重意义）
- `.pinned` CSS class：置顶条目的视觉区分

---

## 7. UI 组件树

```
body
├── .container
│   ├── .header                     # 标题 + 副标题
│   ├── .card                       # 输入卡片
│   │   ├── .input-area             # 输入框 + 生成按钮
│   │   ├── .examples-header        # AI 推荐题目标题 + 换一批
│   │   └── .examples-list          # 5 条推荐题目
│   ├── .loading                    # 加载动画（条件显示）
│   ├── .error-msg                  # 错误提示（条件显示）
│   ├── .result-area                # 结果区（条件显示）
│   │   ├── .exam-format            # 试卷格式卡片
│   │   └── .card
│   │       ├── .result-header      # Tabs + 收藏按钮
│   │       ├── #tab-essay          # 作文内容
│   │       ├── #tab-translation    # 译文内容
│   │       └── #tab-vocabulary     # 词汇卡片列表
│   ├── .card                       # 查词卡片
│   │   ├── .input-area             # 查词输入 + 按钮
│   │   └── #wordSearchResult       # 查词结果
│   ├── .fav-btns                   # 收藏入口按钮
│   │   ├── #btnOpenVocabFav        # 词汇收藏按钮
│   │   └── #btnOpenEssayFav        # 作文收藏按钮
│   └── .footer                     # 页脚
├── .toast                          # Toast 通知
└── .modal-overlay                  # 弹窗遮罩
    ├── .modal-card#modalVocabFav   # 词汇收藏弹窗
    └── .modal-card#modalEssayFav   # 作文收藏弹窗
```

---

## 8. 安全与容错

### 8.1 XSS 防护

项目大量使用 `innerHTML` 渲染 AI 生成内容，所有用户输入和 API 返回均经过 HTML 转义：

- **`esc(str)`**：文本内容转义（`<` → `&lt;`，`>` → `&gt;`，`&` → `&amp;`）
- **`escAttr(str)`**：属性值转义（在上述基础上额外转义 `"` → `&quot;`，`'` → `&#39;`）
- 所有 `data-*` 属性使用 `escAttr()` 包裹 JSON 序列化数据

### 8.2 错误处理层次

| 层级 | 策略 | 用户感知 |
|------|------|----------|
| 网络错误 | try/catch + 用户友好提示 | Toast 或红色错误信息 |
| API 错误 | 检查 `res.ok` + 解析错误消息 | 显示具体 API 错误 |
| JSON 解析 | try/catch + 兜底（代码块剥离） | "格式异常，请重试" |
| localStorage 写入 | try/catch | "存储空间不足" toast |
| localStorage 读取 | try/catch + 类型校验 | 降级为空数组 |
| 按钮 JSON 解析 | try/catch（静默 + 关键操作 toast） | "收藏失败" toast |

### 8.3 竞态保护

- **作文生成**：按钮 disabled 防重复提交
- **查词**：`isSearching` 锁 + 按钮 disabled
- **推荐题目**：`fetchingExamples` 锁 + finally 解锁

### 8.4 数据完整性

- `getFavs()` / `getVFavs()` 解析后校验 `Array.isArray()`，非数组返回 `[]`
- 删除操作带 `confirm()` 确认
- 置顶/取消置顶通过 splice 操作保证数组一致性
- 取消置顶后按原始 `createdAt` 时间戳归位（`>` 严格比较保证同秒顺序不变）

---

## 9. 已知限制与改进方向

### 9.1 当前限制

| 限制 | 说明 |
|------|------|
| API 依赖 | 无网络或 API Key 无效时核心功能不可用 |
| 单文件体积 | 代码约 1250 行，维护成本随功能增长 |
| 无离线缓存 | 收藏可在本地查看，但生成功能必须在线 |
| 无用户系统 | 收藏数据绑定浏览器，换设备无法同步 |
| 无导出 | 不支持 PDF/打印导出 |

### 9.2 改进方向

- [ ] 作文/词汇收藏导出为 PDF 或 Markdown
- [ ] 收藏数据导入/导出（JSON 文件）
- [ ] 作文评分功能（AI 批改 + 分数预估）
- [ ] 写作模板库（按话题分类的句型模板）
- [ ] Service Worker 离线缓存（PWA）
- [ ] 可选云同步（WebDAV / GitHub Gist）
- [ ] 代码拆分（CSS/JS 外置，模块化管理）
