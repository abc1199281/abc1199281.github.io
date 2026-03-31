# 任務清單：個人網站建置與內容遷移

## Phase 1：基礎設定

### T1.1 改寫 hugo.toml ✓
- [x] 設定 `baseURL`, `title`, `author`, `description`, `colorScheme = "auto"`
- [x] 設定 `[taxonomies]`：category, series, tag
- [x] 設定 `[languages.en]` 與 `[languages.zh-tw]`，各自設定 menu.main
- [x] 新增 `[[params.social]]`：GitHub, LinkedIn, Google Scholar, DSP Stack Exchange
- [x] 設定 `since = 2020`、`avatarURL = "images/avatar.jpg"`

### T1.2 新增 avatar 圖片 ✓
- [x] `static/images/` 目錄已建立
- [ ] 手動放入個人頭像（建議 400×400 px）至 `static/images/avatar.jpg`

### T1.3 設定 GitHub Actions 部署 ✓
- [x] 新增 `.github/workflows/deploy.yml`
- [x] 觸發條件：push to `master`
- [x] 步驟：checkout → Hugo build → deploy via GitHub Pages API
- [ ] 在 GitHub repo Settings → Pages → Source 選擇「GitHub Actions」

---

## Phase 2：核心靜態頁面

### T2.1 About 頁面（英文）`content/about.md`
內容包含：
- 個人簡介（研究背景、NCTU ECE）
- 研究領域：rPPG、biomedical signal processing、DSP
- 工作/學歷經歷（參考 LinkedIn）
- 目前職務或研究狀態

### T2.2 About 頁面（繁中）`content/about.zh-tw.md`
- 同上，繁體中文版本

### T2.3 Publications 頁面（英文）`content/publications.md`
- 說明研究方向（rPPG、heart rate、respiratory rate monitoring）
- 放置 Google Scholar profile 連結
- 列出主要論文（可從 Blogspot 的 Publication List 文章複製）

### T2.4 Publications 頁面（繁中）`content/publications.zh-tw.md`

### T2.5 Contact 頁面（英文）`content/contact.md`
- 聯絡方式
- 列出所有社群平台連結

### T2.6 Contact 頁面（繁中）`content/contact.zh-tw.md`

---

## Phase 3：Projects 頁面

### T3.1 建立 `data/projects.yaml`
依序新增以下 5 個 projects，格式參考 design.md 的 YAML schema：

| id | category | GitHub |
|---|---|---|
| rppg-research | research | （Scholar 連結，無 GitHub） |
| dsp-learning-path | educational | abc1199281/DSP_lab |
| systemc-notes | educational | abc1199281/systemc_notes |
| cmdmark | devtools | abc1199281/cmdmark |
| repo-lantern | devtools | abc1199281/repo-lantern |

每筆 entry 填寫：`title`, `description`, `description_zh`, `github`, `tags`, `blog_series`, `highlight`, `highlight_zh`

### T3.2 新增自訂 layout `layouts/projects.html`
- 繼承 `baseof.html`
- 讀取 `site.Data.projects`，依 `category` 分組（research → educational → devtools）
- 每個 category 渲染一組 project cards
- Card 顯示：title、description（依語言切換）、tags、GitHub 連結、blog_series 連結（若有）
- highlight 文字顯示於 card 底部

### T3.3 建立 Projects 頁面 Markdown
- `content/projects.md`（frontmatter 設定 `layout = "projects"`，英文標題）
- `content/projects.zh-tw.md`（frontmatter 設定 `layout = "projects"`，中文標題「作品集」）

---

## Phase 4：Blog 架構

### T4.1 確認 taxonomy 設定
- 確認 `hugo.toml` 的 `[taxonomies]` 包含 `series`, `category`, `tag`
- Hugo 預設會自動產生 `/categories/`、`/series/`、`/tags/` 列表頁

### T4.2 驗證 series 導覽功能
- 建立兩篇測試文章，同屬一個 series
- 確認文章底部出現「系列中的其他文章」導覽區塊
- 確認 hugo-coder 的 `layouts/_partials/posts/series.html` 正確渲染

---

## Phase 5：內容遷移

### T5.1 DSP Learning Path 系列（從 Blogspot 複製）

每篇需設定的 frontmatter：
```toml
title      = "[ChX] ..."
date       = "YYYY-MM-DD"
series     = ["DSP Learning Path"]
categories = ["tech"]
tags       = ["dsp", "matlab"]
```

- [x] `content/posts/dsp-ch4-adc-dac.md`
  - 來源：[Ch4 ADC/DAC] Simulation of ADC/DAC Process（Apr 17, 2020）
  - 加入 DSP_lab 連結：https://github.com/abc1199281/DSP_lab/tree/main/Ch4

- [x] `content/posts/dsp-ch5-group-delay.md`
  - 來源：[Chapter 5.1] Meaning of General Linear Phase or Group Delay（Apr 19, 2020）
  - 加入 DSP_lab 連結：https://github.com/abc1199281/DSP_lab/tree/main/Ch5

- [x] `content/posts/dsp-ch7-fft-zeroing.md`
  - 來源：[Ch7] Why Zeroing FFT Bins is Problematic（Apr 22, 2019）
  - 加入 DSP_lab 連結：https://github.com/abc1199281/DSP_lab/tree/main/Ch7

- [x] `content/posts/dsp-ch11-ar-model.md`
  - 來源：[Ch11] Parametric Signal Model (AR Model)（Apr 27, 2019）
  - 加入 DSP_lab 連結：https://github.com/abc1199281/DSP_lab/tree/main/Ch11

### T5.2 生活文章（從 Blogspot 複製）

每篇需設定的 frontmatter：
```toml
categories = ["life"]
tags       = ["travel", "manchester"]
```

- [x] `content/posts/life-manchester-stroll.md`
  - 來源：A leisurely stroll around the Manchester（Dec 29, 2022）

- [x] `content/posts/life-greenfield.md`
  - 來源：Sightseeing Adventure in a small town, Greenfield（Jan 2, 2023）

### T5.3 rPPG 競賽文章（改寫為 project highlight）
- 來源：Third Position in IEEE CVPR/CVPM2020 Challenge（Aug 31, 2020）
- 目標：不作為 blog post，內容整合至 `data/projects.yaml` 的 `rppg-research` entry
- highlight 填入：「3rd place @ IEEE CVPR/CVPM2020 · 125 participants」

---

## Phase 6：完善與驗證

### T6.1 本地預覽測試
```bash
hugo server -D
```
- 確認首頁、About、Projects、Publications、Blog、Contact 頁面正常渲染
- 切換語言（EN ↔ ZH-TW）確認多語系正常運作
- 確認 DSP series 文章底部導覽顯示正確

### T6.2 SEO 基本設定
- 確認每頁 `<title>` 與 `<meta description>` 正確
- 確認 `hugo.toml` 的 `description` 與 `keywords` 已填寫

### T6.3 部署驗證
- push to `master` → 確認 GitHub Actions 成功觸發
- 確認 `https://abc1199281.github.io/` 正常上線
- 確認中文路徑 `/zh-tw/` 正確運作

---

## 未來擴充（不在本次範圍）

- 新增留言功能（Giscus / Utterances，已在主題 layouts 中支援）
- 加入 Google Analytics 或其他 analytics
- 新增 rPPG 相關技術文章系列
- 新增 SystemC 教學系列（對應 systemc_notes repo）
- 新增 repo-lantern、cmdmark 的使用教學文章
