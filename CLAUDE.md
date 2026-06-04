# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

西语学习12周打卡工具 — 单文件 SPA，vanilla JS + CSS，零依赖，部署在 GitHub Pages。目标：2026年9月SIELE B2备考。

## 开发命令

没有 build/lint/test。直接浏览器打开 `index.html` 即可。

JS 语法检查：把相关代码段提取到 `/tmp/check.js`，用 `node --check /tmp/check.js` 验证。修改变位引擎后必须跑这个，因为单文件报错行号难定位。

## 架构

单文件 `index.html`（~3470行），CSS ~950行 + JS ~2500行。7个视图通过 `switchView()` 单页切换，sidebar 做12周导航。

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

### 变位引擎（~500行，整个项目最复杂的部分）

核心设计：**模式驱动**，不手写每个动词的 42 种变位。

- `REGULAR_ENDINGS` — 规则词尾表（7时态 × 3类动词）
- `conjugate(verbObj, personId, tenseId)` — 核心函数：取动词对象 → 应用模式 → 返回正确变位
- 动词对象由 `buildVerbEntry(infinitive, meaning, pattern)` 构建，pattern 标签（如 `"ar"`, `"stem-change:e→ie"`, `"irregular"`）触发不同的 override 生成逻辑
- 特殊变位类（SC: stem-changing）处理 `e→ie/i`, `o→ue/u`, `e→i` 等
- 正字法修正：`c→qu`, `g→gu`, `z→c`, `gu→gü` 等
- 完全不规则动词通过 `overrides` 逐时态/人称指定
- `VERB_PATTERN_LIST` — 287个动词的清单（不定式、中文释义、模式标签）
- 练习状态：`quizState` 全局对象，`generateQuestion()` 随机出题（排除已掌握、超纲时态），`checkAnswer()` 判对错并记录

时态按周渐进开放（`TENSES[].week`），当前默认只开 `presente_ind` + `indefinido`。

**注意：变位引擎可能存在 bug。**生成每日报告或汇总时，先抽查几个动词变位答案是否正确，再引用。发现 bug 直接在 `conjugate()` 或对应动词的 override 里修复。

### 视图

`currentView` 变量控制，取值：`"today"` / `"week"` / `"progress"` / `"conjugation"` / `"journal"` / `"summary"` / `"vocab"`。`updateView()` 根据 `currentView` 显示/隐藏对应 div。

### 遗忘词练（`vocab` 视图，2026-06-05 新增）

三步学习流程：**Claude 生成阅读文章 → 用户用遗忘词造句 → Claude 批改 → 页面展示批改结果**。

三个子选项卡（`vocabSwitchTab()`）：
- **当前练习** (`practice`) — 显示阅读文章 + 词语标签 + 造句输入区 / 批改结果卡片
- **历史** (`history`) — 过往练习列表，点击可回看
- **词库** (`wordBank`) — 纯词表，批量添加

**初始化文本格式**（Claude 生成，用户粘贴到 `vocabInitInput`）：
```
---practice---
title: 练习标题
words: palabra1|释义1, palabra2|释义2, ...
---
文章内容（markdown 纯文本）
```
解析函数：`vocabCreatePractice()`。如果粘贴内容以 `---单词---` 开头且不含 `---practice---`，自动识别为批改结果，调用 `vocabParseCorrections()`。

**批改结果格式**（Claude 返回，用户粘贴到同一输入框）：
```
---marchar---
原句: xxx
改句: yyy   （或 (✓ 正确)）
说明: zzz   （可选）
---
```
解析后逐词展示原句 vs 改后句卡片，正确标绿✓、有错标红✗。

**造句暂存**：用户在 `practiceSentencesInput` 中写 `word: sentence` 格式，点暂存后复制发给 Claude 批改。`vocabSaveSentences()` 按 `word: text` 格式解析每行。

**数据存储**：`VOCAB_KEY = "spanish_vocab_practice"`，结构 `{ practices: [{ id, date, title, words: [{word, meaning}], article, sentences: [{word, text}], corrections: [{word, original, corrected, note}], status: "writing"|"corrected" }], wordBank: [{word, meaning}] }`。包含在 `ALL_DATA_KEYS` 中以支持导出导入。

**用户学习偏好**：
- 不喜欢抽卡式复习，偏好**故事记忆法**（用遗忘词编荒诞故事）和**语境造句**
- 阅读文章做成 SIELE B1+-B2 水平，目标词在文中以粗体出现
- 每个词需要词源分析 + 英语同源 + 钩子 + 常用搭配 + 记法
- 造句一次性批量写完发给 Claude 批改，不一词一词来
- 最近用户在旅途中，本地 `file://` 打开为主，出门前再导出数据到线上

## 注意事项

- **用户数据在浏览器 localStorage，不能从代码推算天数**。等用户发每日报告校准。
- 修改 `WEEK_DATA` 的任务结构时，需要同步更新 `CATEGORIES` 和 `migrateCheckins` 的 key 映射。
- 变位引擎改完后，用 `node --check` 验证语法，再在浏览器里手动测试几个动词确认。
- Git remote: `https://github.com/WellWellWell-M/spanish-learning-tracker.git`，push 后 GitHub Pages 自动部署。

## 用户诉求

- **正向反馈**：对用户的打卡行为和学习进展多给积极反馈，帮助保持动力。
- **主动建议**：关注打卡数据中的薄弱环节（如某类别长期未完成、变位错题集中），主动提出优化建议。
- **最终目标**：一切改动和建议以帮助用户 2026 年 9 月考过 SIELE B2 为准绳。
