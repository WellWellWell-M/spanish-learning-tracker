# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

西语学习12周打卡工具 — 单文件 SPA，vanilla JS + CSS，零依赖，部署在 GitHub Pages。目标：2026年9月SIELE B2备考。

## 开发命令

没有 build/lint/test。直接浏览器打开 `index.html` 即可。

JS 语法检查：把相关代码段提取到 `/tmp/check.js`，用 `node --check /tmp/check.js` 验证。修改变位引擎后必须跑这个，因为单文件报错行号难定位。

## 架构

单文件 `index.html`（~2876行），CSS ~800行 + JS ~2000行。6个视图通过 `switchView()` 单页切换，sidebar 做12周导航。

### 数据层

全部持久化在 `localStorage`，4个 key：
- `spanish_checkins` — 每日打卡（`{ "2026-05-09": { escuchar: true, hablar: true, ... } }`）
- `spanish_journal_v2` — 学习笔记（多条目版）
- `spanish_summaries` — AI学习汇总
- `spanish_conj_errors` — 变位错题记录

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

`currentView` 变量控制，取值：`"today"` / `"week"` / `"progress"` / `"conjugation"` / `"journal"` / `"summary"`。`updateView()` 根据 `currentView` 显示/隐藏对应 div。

## 注意事项

- **用户数据在浏览器 localStorage，不能从代码推算天数**。等用户发每日报告校准。
- 修改 `WEEK_DATA` 的任务结构时，需要同步更新 `CATEGORIES` 和 `migrateCheckins` 的 key 映射。
- 变位引擎改完后，用 `node --check` 验证语法，再在浏览器里手动测试几个动词确认。
- Git remote: `git@github.com:wellwellwell-m/spanish-learning-tracker.git`，push 后 GitHub Pages 自动部署。

## 用户诉求

- **正向反馈**：对用户的打卡行为和学习进展多给积极反馈，帮助保持动力。
- **主动建议**：关注打卡数据中的薄弱环节（如某类别长期未完成、变位错题集中），主动提出优化建议。
- **最终目标**：一切改动和建议以帮助用户 2026 年 9 月考过 SIELE B2 为准绳。
