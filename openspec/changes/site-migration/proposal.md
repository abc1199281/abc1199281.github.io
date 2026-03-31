# 提案：個人網站建置與內容遷移

## 概述

將散落在 Blogspot、Google Scholar、LinkedIn、GitHub 等平台的個人資訊與技術文章，統一遷移至 `abc1199281.github.io`，建立一個兼具作品集（Portfolio）與技術/生活部落格（Blog）的中英雙語個人網站。

---

## 背景

目前個人資訊分散在多個平台，缺乏統一的入口：

| 平台 | 內容 | 問題 |
|---|---|---|
| poweidsplearningpath.blogspot.com | DSP 教學系列、rPPG 研究筆記 | 無法客製化、SEO 差 |
| poweihuangblog.blogspot.com | 生活旅遊文章 | 與技術內容分離 |
| Google Scholar | 學術論文清單 | 無法整合個人品牌 |
| LinkedIn | 職涯經歷 | 非技術受眾導向 |
| GitHub repos | 多個 side projects | 各自獨立、無彙整入口 |

目前的 GitHub Pages 使用 `hugo-coder` 主題，但尚無任何實際內容。

---

## 目標

1. 建立統一的個人網站，涵蓋作品集與部落格兩個獨立區塊
2. 遷移 Blogspot 文章至 Hugo 格式
3. 彙整所有 GitHub side projects 至 Portfolio 頁面
4. 支援中英雙語（繁體中文 / English）
5. 架構具備彈性，未來可持續新增文章與 projects

---

## 非目標

- 不重新設計或替換 `hugo-coder` 主題
- 不完整翻譯每一篇部落格文章（文章各自使用適合的語言即可）
- 不自行架設論文資料庫（Publications 頁面只連結至 Google Scholar）
- 不搬移 DSP Stack Exchange 的回答（只放個人頁面連結）

---

## 網站架構

```
abc1199281.github.io/
│
├── /about/             個人簡介（中英雙語）
│
├── /projects/          作品集（單頁，data-driven，依分類分組）
│   ├── Research        rPPG Research + CVPR/CVPM2020
│   ├── Educational     DSP Learning Path、SystemC Notes
│   └── Developer Tools cmdmark、repo-lantern
│
├── /publications/      學術論文（連結至 Google Scholar）
│
├── /posts/             部落格
│   ├── series: DSP Learning Path  (技術系列)
│   └── category: life             (生活文章)
│
└── /contact/           聯絡與社群連結
```

---

## 雙語策略

採三層設計，兼顧完整度與維護成本：

```
Layer 1 — UI 與導覽列（全翻譯）
  hugo.toml 設定 [languages.en] + [languages.zh-tw]
  導覽列顯示語言切換按鈕

Layer 2 — 核心頁面（全翻譯）
  about.md       ↔  about.zh-tw.md
  projects.md    ↔  projects.zh-tw.md
  publications.md ↔ publications.zh-tw.md

Layer 3 — 部落格文章（各自語言，不強制翻譯）
  每篇文章以最自然的語言撰寫
  frontmatter 標記語言供讀者參考
```

---

## Projects 資料結構

Projects 頁面採用 Hugo data file 驅動，新增 project 只需在 YAML 加入一筆資料：

```yaml
# data/projects.yaml
- id: repo-lantern
  category: devtools          # research | educational | devtools
  title: "repo-lantern"
  description: "AI-powered codebase analyzer..."
  description_zh: "AI 驅動的程式碼分析工具..."
  github: "https://github.com/abc1199281/repo-lantern"
  tags: ["Python", "LangGraph", "AI"]
  blog_series: ""
  highlight: "109 commits"
```

---

## 內容遷移清單

### 從 Blogspot 遷移（需手動 copy 為 Markdown）

| 來源文章 | 目標路徑 | 分類 |
|---|---|---|
| [Ch4] ADC/DAC Simulation | `content/posts/dsp-ch4-adc-dac.md` | series: DSP Learning Path |
| [Ch5.1] General Linear Phase | `content/posts/dsp-ch5-group-delay.md` | series: DSP Learning Path |
| [Ch7] Why Zeroing FFT Bins | `content/posts/dsp-ch7-fft-zeroing.md` | series: DSP Learning Path |
| [Ch11] AR Model | `content/posts/dsp-ch11-ar-model.md` | series: DSP Learning Path |
| Manchester city stroll | `content/posts/life-manchester-stroll.md` | category: life |
| Greenfield, Manchester | `content/posts/life-greenfield.md` | category: life |

### 從 Blogspot 遷移至 Portfolio（改寫為 project 格式）

| 來源文章 | 目標 |
|---|---|
| Publication List | `data/projects.yaml` (rPPG entry) + `content/publications.md` |
| CVPR/CVPM2020 3rd Place | `data/projects.yaml` (rPPG entry) |

### 手動建立

| 頁面 | 內容來源 |
|---|---|
| `content/about.md` | LinkedIn 職涯、學歷、研究背景 |
| `data/projects.yaml` | 所有 GitHub repos 資訊 |
| `content/publications.md` | Google Scholar 連結 |
| `hugo.toml` | 網站設定、社群連結、語言設定 |

---

## 社群連結（Social Bar）

| 平台 | 連結 |
|---|---|
| GitHub | https://github.com/abc1199281 |
| LinkedIn | https://www.linkedin.com/in/powei-huang-b24061104/ |
| Google Scholar | https://scholar.google.com/citations?user=sklHe5cAAAAJ |
| DSP Stack Exchange | https://dsp.stackexchange.com/users/41818/po-wei-huang |

---

## 相關資源

- 主題文件：hugo-coder (`layouts/`, `exampleSite/hugo.toml`)
- DSP 程式碼：https://github.com/abc1199281/DSP_lab
- SystemC 筆記：https://github.com/abc1199281/systemc_notes
- CLI 工具：https://github.com/abc1199281/cmdmark
- AI 文件工具：https://github.com/abc1199281/repo-lantern
