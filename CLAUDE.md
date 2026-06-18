# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

西语学习12周打卡工具 — 单文件 SPA，vanilla JS + CSS，零依赖，部署在 GitHub Pages。目标：2026年9月SIELE B2备考。

## 开发命令

没有 build/lint/test。直接浏览器打开 `index.html` 即可。

JS 语法检查：把相关代码段提取到 `/tmp/check.js`，用 `node --check /tmp/check.js` 验证。修改变位引擎后必须跑这个，因为单文件报错行号难定位。

## 架构

单文件 `index.html`（~3700行），7个视图通过 `switchView()` 单页切换，sidebar 做12周导航。

### 数据层

全部持久化在 `localStorage`，5个 key：
- `spanish_checkins` — 每日打卡（`{ "2026-05-09": { escuchar: true, hablar: true, ... } }`）
- `spanish_journal_v2` — 学习笔记（多条目版）
- `spanish_summaries` — AI学习汇总
- `spanish_conj_errors` — 变位错题记录
- `spanish_vocab_practice` — 遗忘词练数据（`{ practices: [...], wordBank: [...] }`）

导出/导入通过 JSON blob 下载/上传，key 映射在 `ALL_DATA_KEYS`。数据迁移用 IIFE（`migrateCheckins`, `migrateJournal`），在脚本加载时自动执行。

### 关键函数

- `dateKey(date)` → `"2026-05-09"`，`today()` → 今天0点（本地时间）
- `getWeekInfo(date)` → `{ weekNum: 1-12, dayOfWeek: 0-6 }`，基于 `START_DATE = 2026-05-08`
- `WEEK_DATA[]` — 12周每日任务定义，`tasks` 数组每项对应一个 `CATEGORIES` key
- `CATEGORIES` — 7项分类：escuchar/hablar/leer/escribir/vocabulario/gramatica/revisar
- `escapeHtml(s)` — HTML 转义，数据渲染时使用
- `escapeHtml` 是正确函数名，忘了/写成 `escHtml` 会找不到

### 变位引擎（~550行，整个项目最复杂的部分）

核心设计：**模式驱动**，不手写每个动词的 42 种变位。

- `REGULAR_ENDINGS` — 规则词尾表（8时态 × 3类动词）
- `conjugate(verbObj, personId, tenseId)` — 核心函数：取动词对象 → 应用模式 → 返回正确变位
- 动词对象由 `buildVerbEntry(infinitive, meaning, pattern)` 构建，pattern 标签（如 `"ar"`, `"stem-change:e→ie"`, `"irregular"`, `"tener"`, `"ducir"` 等20+种）触发不同的 override 生成逻辑。返回值包含 `tags` 数组（如 `["e→ie","yo-irreg","pret-irreg"]`），用于多维过滤。
- `IRREG_PARTICIPLES` — 不规则过去分词映射表（~30个：hecho/dicho/escrito/visto/puesto/vuelto/abierto/muerto/roto…），`getIrregParticiple()` 查询。`buildVerbEntry` 自动检测不规则分词并生成 participio overrides + `"participio-irreg"` tag。
- 特殊变位类（SC: stem-changing）处理 `e→ie/i`, `o→ue/u`, `e→i` 等
- 正字法修正：`c→qu`, `g→gu`, `z→c`, `gu→gü` 等
- 完全不规则动词通过 `overrides` 逐时态/人称指定
- `VERB_PATTERN_LIST` — ~290个动词的清单（不定式、中文释义、模式标签）+ `IRREG_VERBS`（ser/ir，含 tags）
- 练习状态：`quizState` 全局对象，`generateQuestion()` 随机出题（排除已掌握、超纲时态），`checkAnswer()` 判对错并记录
- 人称选择：`pickWeightedPerson()` — vosotros(id=5) 权重 1，其余 6，出题概率 1/31

**时态**：`TENSES` 数组共 8 个时态，按周渐进开放。当前（第5周）已开：presente_ind + indefinido + imperfecto + futuro + condicional + presente_subj + imperfecto_subj + participio。默认练习只开 presente_ind + indefinido，用户手动点按钮增减。

**分类过滤器**：三行筛选项，基于 `tags` 数组过滤动词池（`getFilteredVerbPool()`）
- 词干：e→ie / o→ue / e→i / u→ue
- 不规则：yo（yo不规则）/ pret（indefinido不规则）/ fut（将来/条件不规则）/ 完全（多时态不规则）
- 其他：规则 / 正字法 / 分词（有不规则过去分词的动词）

**注意：变位引擎可能存在 bug。**生成每日报告或汇总时，先抽查几个动词变位答案是否正确，再引用。发现 bug 直接在 `conjugate()` 或对应动词的 override 里修复。修改后必须 `node --check` 验证语法。

**声明顺序注意事项**：`IRREG_PARTICIPLES` 和 `getIrregParticiple()` 必须在 `VERB_DB` 之前声明，因为 `VERB_DB` 构建时 `buildVerbEntry` 就会调用它们。搞反了会导致 TDZ ReferenceError，整个页面白屏。

### 视图

`currentView` 变量控制，取值：`"today"` / `"week"` / `"progress"` / `"conjugation"` / `"journal"` / `"summary"` / `"vocab"`。`updateView()` 根据 `currentView` 显示/隐藏对应 div。

### 遗忘词练（`vocab` 视图，2026-06-05 新增，持续迭代）

三步学习流程：**Claude 生成阅读文章 → 用户用遗忘词造句 → Claude 批改 → 页面展示批改结果**。用故事记忆法替代死记硬背，文章做 SIELE B1+-B2 水平。

三个子选项卡（`vocabSwitchTab()`）：
- **当前练习** (`practice`) — 显示阅读文章（`**粗体**` 渲染） + 词语标签 + 造句输入区 / 批改结果卡片
- **历史** (`history`) — 过往练习列表，每条可 ✕ 删除，多条时出现「清空全部历史」
- **词库** (`wordBank`) — 自动汇总。创建练习时单词自动进入词库（去重），记录 `firstFrom`（首次来源）+ `appearances`（出现次数）+ `lastSeen`。手动添加也支持

**初始化文本格式**（Claude 生成，用户粘贴到 `vocabInitInput`）：
```
---practice---
title: 练习标题
words: palabra1|释义1, palabra2|释义2, ...
===
文章内容（markdown 纯文本，**粗体** 会被渲染）
```
解析函数：`vocabCreatePractice()`。自动将 `\r\n` 规范为 `\n`。如果粘贴内容以 `---单词---` 开头且不含 `---practice---`，自动识别为批改结果，调用 `vocabParseCorrections()`。

**注意**：文章与 header 的分隔符是独立成行的 `===`，不是 `---`（避免与文章内 markdown 横线冲突）。

**批改结果格式**（Claude 返回，用户粘贴到同一输入框）：
```
---marchar---
原句: xxx
改句: yyy   （或 (✓ 正确)）
说明: zzz   （可选）
---
```
解析后逐词展示原句 vs 改后句卡片，正确标绿✓、有错标红✗。

**造句暂存**：用户在 `practiceSentencesInput` 中写 `word: sentence` 格式，点「💾 暂存」→ 自动复制到剪贴板（`navigator.clipboard.writeText`），发给 Claude 批改。

**数据存储**：`VOCAB_KEY = "spanish_vocab_practice"`，结构 `{ practices: [{ id, date, title, words: [{word, meaning}], article, sentences: [{word, text}], corrections: [{word, original, corrected, note}], status: "writing"|"corrected" }], wordBank: [{word, meaning, firstFrom, appearances, lastSeen}] }`。包含在 `ALL_DATA_KEYS` 中以支持导出导入。

**每日报告集成**：`dailyReport()` 函数会检查当天是否有遗忘词练习，自动列出练习标题、词单、造句进度、批改结果。

**初始化文本文件**：`init_practice_01.txt` ~ `init_practice_11.txt` 存放练习的初始化文本。**每期必须包含**：文章后附 `### Vocabulario en contexto`（每个词在文中的用法实例 + 核心搭配），init 文件开头附完整的词源/同源/钩子分析。不能只写故事不发词分析。

## 注意事项

- **用户数据在浏览器 localStorage，不能从代码推算天数**。等用户发每日报告校准。
- 修改 `WEEK_DATA` 的任务结构时，需要同步更新 `CATEGORIES` 和 `migrateCheckins` 的 key 映射。
- 变位引擎改完后，用 `node --check` 验证语法，再在浏览器里手动测试几个动词确认。
- Git remote: `https://github.com/WellWellWell-M/spanish-learning-tracker.git`，push 后 GitHub Pages 自动部署。

## 用户学习偏好

- **正向反馈**：对用户的打卡行为和学习进展多给积极反馈，帮助保持动力。
- **主动建议**：关注打卡数据中的薄弱环节（如某类别长期未完成、变位错题集中），主动提出优化建议。
- **最终目标**：一切改动和建议以帮助用户 2026 年 9 月考过 SIELE B2 为准绳。
- **词源解释**：用户喜欢词源故事（如 banco 从长椅变成银行的典故），解释生词时优先用英语同源词 + 拉丁语根。
- **SIELE 阅读**：阅读文章控制在 B1+-B2 难度，目标词在文中以 `**粗体**` 出现。长度灵活——用户嫌太长就缩短（第6期只250词），丰富时也可写800+词。
- **变位练习**：用户不喜欢 vosotros，已将其出题概率从 1/6 降到 1/31。分词只练不规则的。推荐按 e→ie→o→ue→e→i→pret→fut→完全 顺序逐维过滤练习。
- **遗忘词造句**：一次性批量写 + 批量批改。批改格式：`---单词--- 原句/改句/说明 ---`。
- **文章风格**：用户偏好短小有趣的故事（100-200字），避免冗长说教。时态尽量只用 presente + indefinido，不要用复杂时态让用户认不出词。禁止缺席父亲回归、替酒鬼开脱、受害者背锅等令用户反感的模板。
- **猫王宇宙**：用户喜欢「会计表姐 + 称霸街区的猫」系列故事（第13-14期）。后续遗忘词故事优先用这个设定连载下去，角色已有：会计表姐、猫王、第一人称叙述者。
- **每期练习必须包含**：(1) 聊天里的词源/同源/钩子分析；(2) init 文件里的 `Vocabulario en contexto` 用法解析 + 核心搭配；(3) 文章后面附英文版翻译，帮助用户在两种语言间对照理解
- **本地优先**：目前本地 `file://` 打开为主，出门前导出数据到线上。
- **每日报告**：点📋日报告会汇总打卡+笔记+变位错题+遗忘词练进度。用户粘贴日报后 Claude 生成总结，用户再粘贴到 📊 学习汇总。
