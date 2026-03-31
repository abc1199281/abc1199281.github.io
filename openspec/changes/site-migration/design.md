# 技術設計：個人網站建置與內容遷移

## Repo 架構

此 repo（`abc1199281.github.io`）本身是 `hugo-coder` 主題的 fork，`go.mod` 宣告為 `module github.com/luizdepra/hugo-coder`。

採用「theme-in-repo」模式：主題檔案（`layouts/`、`assets/`）直接放在 repo 根目錄，不另外引入外部主題。只需在根目錄新增 `content/`、`data/` 並正確設定 `hugo.toml` 即可運作。

```
abc1199281.github.io/          ← repo 根目錄 = Hugo site 根目錄
├── layouts/                   ← hugo-coder 主題 layouts（已存在）
├── assets/                    ← 主題 CSS/JS（已存在）
├── hugo.toml                  ← 需要完整改寫（目前僅有 baseURL）
├── content/                   ← 需要新建（目前不存在）
├── data/                      ← 需要新建
│   └── projects.yaml
└── static/
    └── images/
        └── avatar.jpg         ← 需要新增
```

---

## hugo.toml 設定架構

以 `exampleSite/hugo.toml` 為參考基礎，改寫根目錄的 `hugo.toml`：

```toml
baseURL = "https://abc1199281.github.io/"
title = "Po-Wei Huang"
theme = ""                      # theme-in-repo 模式不需指定
languageCode = "en"
defaultContentLanguage = "en"
defaultContentLanguageInSubdir = false   # 英文保持在根目錄

[taxonomies]
category = "categories"
series   = "series"
tag      = "tags"

[params]
author      = "Po-Wei Huang"
description = "Signal Processing Engineer & Developer"
keywords    = "blog,signal processing,DSP,rPPG,developer"
info        = ["Signal Processing", "Software Engineer"]
avatarURL   = "images/avatar.jpg"
colorScheme = "auto"
since       = 2020

[[params.social]]  # GitHub, LinkedIn, Scholar, DSP SE
...

[languages.en]
languageName = ":uk:"
[[languages.en.menu.main]]  # About, Projects, Publications, Blog, Contact

[languages.zh-tw]
languageName = ":taiwan:"
[[languages.zh-tw.menu.main]]  # 關於, 作品集, 論文, 部落格, 聯絡
```

---

## 多語系實作

### URL 結構

```
英文（預設語言）  →  /about/  /projects/  /posts/
繁體中文         →  /zh-tw/about/  /zh-tw/projects/  /zh-tw/posts/
```

`defaultContentLanguageInSubdir = false`：英文保持在根目錄，中文加 `/zh-tw/` 前綴。

### 核心頁面雙語檔案對應

```
content/
├── about.md               ← 英文版
├── about.zh-tw.md         ← 繁中版
├── projects.md            ← 英文版（layout = "projects"）
├── projects.zh-tw.md      ← 繁中版（layout = "projects"）
├── publications.md
├── publications.zh-tw.md
├── contact.md
└── contact.zh-tw.md
```

### 部落格文章

每篇文章只有一個語言版本，在 frontmatter 標記語言即可，不強制翻譯：

```toml
# content/posts/dsp-ch4-adc-dac.md
[params]
language = "en"
```

---

## Projects 頁面：data-driven 設計

### 資料來源

```
data/projects.yaml   ←  唯一資料來源
                         新增 project = 新增一筆 YAML entry
```

### YAML Schema

```yaml
- id: repo-lantern
  category: devtools          # research | educational | devtools
  title: "repo-lantern"
  description: "AI-powered codebase analyzer that turns complex repos into step-by-step narratives."
  description_zh: "AI 驅動的程式碼分析工具，將複雜 repo 轉化為循序漸進的說明文件。"
  github: "https://github.com/abc1199281/repo-lantern"
  tags: ["Python", "LangGraph", "AI", "CLI"]
  blog_series: ""             # 有對應的 blog series 時填入 series slug
  highlight: "109 commits · multi-language output"
  highlight_zh: "109 commits · 多語言輸出"
```

### 自訂 Layout

在 `content/projects.md` 指定自訂 layout：

```toml
# content/projects.md frontmatter
layout = "projects"
```

新增 `layouts/projects.html`，覆蓋預設 `single.html`：

```html
{{ define "content" }}
  <!-- 讀取 site.Data.projects，依 category 分組渲染 project cards -->
  <!-- 支援雙語：根據 .Site.Language.Lang 選擇 description 或 description_zh -->
{{ end }}
```

分組順序固定為：`research` → `educational` → `devtools`

---

## Blog Taxonomy 設計

```toml
# hugo.toml
[taxonomies]
category = "categories"   # tech | life
series   = "series"       # DSP Learning Path（文章排序用）
tag      = "tags"         # dsp, rppg, matlab, python, travel, ...
```

### DSP 系列文章 frontmatter 範例

```toml
title   = "[Ch4] Simulation of ADC/DAC Process"
date    = "2020-04-17"
series  = ["DSP Learning Path"]
categories = ["tech"]
tags    = ["dsp", "matlab", "sampling"]
```

`series` taxonomy 讓讀者可在文章底部看到系列導覽（上一篇 / 下一篇），對應 DSP_lab 的 chapter 結構。

---

## 部署設定

使用 GitHub Actions 自動部署：

```
.github/workflows/deploy.yml
  trigger: push to master
  steps:
    1. checkout (with submodules)
    2. setup Hugo (extended, 最新版)
    3. hugo build → public/
    4. deploy to gh-pages branch（或直接用 GitHub Pages from master /docs）
```

建議使用 `gh-pages` branch 做為 GitHub Pages source，`master` 保持為開發 branch。

---

## 目錄結構（完整實作後）

```
abc1199281.github.io/
├── hugo.toml
├── go.mod
├── content/
│   ├── about.md / about.zh-tw.md
│   ├── projects.md / projects.zh-tw.md
│   ├── publications.md / publications.zh-tw.md
│   ├── contact.md / contact.zh-tw.md
│   └── posts/
│       ├── dsp-ch4-adc-dac.md
│       ├── dsp-ch5-group-delay.md
│       ├── dsp-ch7-fft-zeroing.md
│       ├── dsp-ch11-ar-model.md
│       ├── life-manchester-stroll.md
│       └── life-greenfield.md
├── data/
│   └── projects.yaml
├── static/
│   └── images/
│       └── avatar.jpg
├── layouts/
│   ├── projects.html          ← 新增（覆蓋）
│   └── ... (existing theme layouts)
├── assets/
├── openspec/
└── .github/
    └── workflows/
        └── deploy.yml
```
