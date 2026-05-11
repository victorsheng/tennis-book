# Pages Bilingual Library Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current GitHub Pages list with a light-theme bilingual tennis PDF library that supports language switching, search, category filtering, and Raw PDF downloads.

**Architecture:** Keep the site as a single static `docs/index.html` file with inline CSS and vanilla JavaScript. Store all book metadata in a local `BOOKS` array, render the cards client-side, and persist the language choice in `localStorage`.

**Tech Stack:** HTML, CSS, vanilla JavaScript, GitHub Pages, Poppler tools (`pdftotext`, `pdfinfo`) for source sampling, Python's built-in static server for local QA.

---

## Scope Check

The approved spec covers one static GitHub Pages page and one book metadata set. It does not require a backend, build system, separate data files, PDF processing pipeline, or cover image generation, so it fits one implementation plan.

## File Structure

- Modify: `docs/index.html`
  - Owns page markup, light theme styling, bilingual UI strings, book metadata, rendering, search, filter, and Raw PDF link generation.
- Read: `docs/superpowers/specs/2026-05-11-pages-bilingual-library-design.md`
  - Source of approved design requirements.
- Read: repository root `*.pdf`
  - Source material for conservative book descriptions.

No new production files are required.

## Task 1: Confirm PDF Inventory And Metadata Evidence

**Files:**
- Read: `*.pdf`
- Modify: none

- [ ] **Step 1: Confirm the PDF inventory**

Run:

```bash
rg --files -g '*.pdf' | sort
```

Expected: exactly these 17 files are printed:

```text
[网球技术精解全书].《网球》杂志.著.插图版.pdf
tennis淺談網球步法訓練.pdf
中国网协CTN网球等级智能测评指南.pdf
实用网球技巧提升200  _13225880.pdf
德约科维奇  一发制胜  我的14天身心逆转计划_13648186.pdf
拉法纳达尔自传_13400544.pdf
牵伸解剖指南.pdf
独自上场_13077137.pdf
精准拉伸：疼痛消除和损伤预防的针对性练习.pdf
網球截擊技術及雙打「搶打戰術」之應用分析.pdf
网球制胜：实用技战术图解=Winning Tennis_13735994.pdf
网球压力训练_上册.pdf
网球压力训练_下册.pdf
网球提升  基础技巧与实战策略_13831982.pdf
网球运动教程.陶志翔.pdf
网球运动系统训练13800928.pdf
费德勒传 光影中的网坛传奇.pdf
```

- [ ] **Step 2: Extract first-page samples where text is available**

Run:

```bash
mkdir -p /tmp/tennis-book-samples
for file in *.pdf; do pdftotext -f 1 -l 12 "$file" "/tmp/tennis-book-samples/${file%.pdf}.txt"; done
```

Expected: `/tmp/tennis-book-samples` contains one `.txt` file per PDF. Some scanned PDFs may produce mostly form-feed characters; that is acceptable and means the metadata should stay conservative.

- [ ] **Step 3: Check page counts for link and inventory sanity**

Run:

```bash
for file in *.pdf; do printf '%s | ' "$file"; pdfinfo "$file" | awk -F': *' '/^Pages:/ {print $2}'; done
```

Expected: every file prints a page count and no command errors.

- [ ] **Step 4: Compare the planned metadata against extracted evidence**

Use the text samples when available, especially for translated or anatomy/stretching books. For scanned PDFs, rely on the title and keep the description broad. The implementation should use the `BOOKS` entries in Appendix A unless the extracted text reveals an obvious mismatch; in that case, adjust only the affected summary before committing.

## Task 2: Replace `docs/index.html` With The Bilingual Library Page

**Files:**
- Modify: `docs/index.html`

- [ ] **Step 1: Replace the current page**

Use `apply_patch` to replace the contents of `docs/index.html` with the complete HTML in Appendix A.

- [ ] **Step 2: Validate JavaScript syntax**

Run:

```bash
node - <<'NODE'
const fs = require('fs');
const html = fs.readFileSync('docs/index.html', 'utf8');
const scripts = [...html.matchAll(/<script>([\s\S]*?)<\/script>/g)].map((m) => m[1]);
if (scripts.length !== 1) {
  throw new Error(`Expected 1 inline script, found ${scripts.length}`);
}
new Function(scripts[0]);
console.log('inline script syntax ok');
NODE
```

Expected:

```text
inline script syntax ok
```

- [ ] **Step 3: Commit the page implementation**

Run:

```bash
git add docs/index.html
git commit -m "feat: redesign Pages library"
```

Expected: commit succeeds with only `docs/index.html` staged.

## Task 3: Verify Browser Behavior

**Files:**
- Modify: `docs/index.html` only if verification reveals a defect

- [ ] **Step 1: Start a local static server**

Run:

```bash
python3 -m http.server 8000 --directory docs
```

Expected: server starts and prints a line containing `Serving HTTP on`.

- [ ] **Step 2: Open the page**

Open:

```text
http://localhost:8000/
```

Expected: light theme renders, the hero section says `网球电子书与训练资料库`, the toolbar is visible, and the page shows 17 book cards.

- [ ] **Step 3: Verify language switching**

Click `English`.

Expected: the hero title becomes `Tennis Books & Training Library`, category buttons and card summaries switch to English, and the active language button changes state.

Click `中文`.

Expected: the same UI switches back to Chinese without a page reload.

- [ ] **Step 4: Verify search**

Search for:

```text
费德勒
```

Expected: exactly the Federer biography card remains.

Clear the search and search for:

```text
stretching
```

Expected: `Prescriptive Stretching` and `Stretching Anatomy Guide` remain.

- [ ] **Step 5: Verify category filtering**

Click the `名将传记` category.

Expected: the Agassi, Nadal, Djokovic, and Federer cards remain.

Click `全部`.

Expected: all 17 cards return.

- [ ] **Step 6: Verify empty state**

Search for:

```text
zzzz-not-a-book
```

Expected: no cards render and the empty state appears in the active language.

- [ ] **Step 7: Verify Raw PDF link generation**

Run:

```bash
node - <<'NODE'
const fs = require('fs');
const html = fs.readFileSync('docs/index.html', 'utf8');
const match = html.match(/const BOOKS = (\[[\s\S]*?\]);/);
if (!match) throw new Error('BOOKS array not found');
const books = Function(`return ${match[1]}`)();
const base = 'https://raw.githubusercontent.com/victorsheng/tennis-book/master/';
for (const name of [
  '[网球技术精解全书].《网球》杂志.著.插图版.pdf',
  '精准拉伸：疼痛消除和损伤预防的针对性练习.pdf',
  '网球制胜：实用技战术图解=Winning Tennis_13735994.pdf'
]) {
  const book = books.find((item) => item.file === name);
  if (!book) throw new Error(`Missing ${name}`);
  console.log(base + encodeURIComponent(book.file));
}
NODE
```

Expected: three encoded `https://raw.githubusercontent.com/victorsheng/tennis-book/master/` URLs print, with Chinese characters encoded as `%` sequences.

- [ ] **Step 8: Commit verification fixes if needed**

If any browser check required a change, run:

```bash
git add docs/index.html
git commit -m "fix: polish Pages library interactions"
```

Expected: commit succeeds. If no fixes were needed, skip this step.

## Task 4: Final Checks

**Files:**
- Modify: none unless checks reveal a defect

- [ ] **Step 1: Check whitespace and conflict markers**

Run:

```bash
git diff --check HEAD
```

Expected: no output.

- [ ] **Step 2: Check repository status**

Run:

```bash
git status --short
```

Expected: no untracked or modified files, unless the current session intentionally keeps the implementation plan uncommitted.

- [ ] **Step 3: Final response evidence**

Report the exact commits created, the local URL used for QA, and the verification checks that passed.

## Appendix A: Complete `docs/index.html`

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <meta name="description" content="tennis-book 网球电子书与训练资料库，收集技术、训练、传记与运动科学资料。" />
  <title>tennis-book · 网球资料库</title>
  <style>
    :root {
      --bg: #f5f7f3;
      --surface: #ffffff;
      --surface-soft: #eef4ee;
      --border: #d8e2d9;
      --border-strong: #b9cbbb;
      --text: #17231b;
      --muted: #607064;
      --muted-strong: #405347;
      --green: #1f7a45;
      --green-dark: #155b33;
      --blue: #315caa;
      --amber: #a85d18;
      --rose: #9a4057;
      --shadow: 0 18px 45px rgba(31, 58, 39, 0.08);
    }

    * {
      box-sizing: border-box;
    }

    html {
      scroll-behavior: smooth;
    }

    body {
      margin: 0;
      min-height: 100vh;
      font-family: ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", sans-serif;
      background: var(--bg);
      color: var(--text);
      line-height: 1.6;
    }

    a {
      color: inherit;
    }

    button,
    input {
      font: inherit;
    }

    .page {
      width: min(1120px, calc(100% - 32px));
      margin: 0 auto;
      padding: 20px 0 42px;
    }

    .topbar {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 16px;
      padding: 14px 0;
    }

    .brand {
      display: inline-flex;
      align-items: center;
      gap: 10px;
      min-width: 0;
      text-decoration: none;
    }

    .brand-mark {
      display: inline-flex;
      width: 34px;
      height: 34px;
      align-items: center;
      justify-content: center;
      border-radius: 8px;
      background: var(--green);
      color: #ffffff;
      font-weight: 800;
    }

    .brand-name {
      display: block;
      font-weight: 800;
      line-height: 1.2;
    }

    .brand-subtitle {
      display: block;
      color: var(--muted);
      font-size: 0.78rem;
      line-height: 1.3;
    }

    .top-actions {
      display: flex;
      align-items: center;
      justify-content: flex-end;
      gap: 10px;
      flex-wrap: wrap;
    }

    .count-pill,
    .lang-switch {
      border: 1px solid var(--border);
      background: rgba(255, 255, 255, 0.78);
      box-shadow: 0 8px 24px rgba(31, 58, 39, 0.05);
    }

    .count-pill {
      border-radius: 999px;
      color: var(--muted-strong);
      padding: 7px 11px;
      font-size: 0.86rem;
      white-space: nowrap;
    }

    .lang-switch {
      display: inline-flex;
      align-items: center;
      gap: 2px;
      border-radius: 999px;
      padding: 3px;
    }

    .lang-button {
      border: 0;
      border-radius: 999px;
      background: transparent;
      color: var(--muted);
      cursor: pointer;
      padding: 6px 10px;
      transition: background 0.15s ease, color 0.15s ease;
    }

    .lang-button[aria-pressed="true"] {
      background: var(--green);
      color: #ffffff;
    }

    .hero {
      display: grid;
      grid-template-columns: minmax(0, 1.2fr) minmax(280px, 0.8fr);
      gap: 28px;
      align-items: stretch;
      padding: 34px;
      margin-top: 18px;
      border: 1px solid var(--border);
      border-radius: 8px;
      background: rgba(255, 255, 255, 0.82);
      box-shadow: var(--shadow);
    }

    .eyebrow {
      margin: 0 0 10px;
      color: var(--green);
      font-weight: 800;
      font-size: 0.82rem;
    }

    h1 {
      margin: 0;
      max-width: 760px;
      font-size: clamp(2rem, 5vw, 4.2rem);
      line-height: 1.05;
    }

    .intro {
      max-width: 720px;
      margin: 18px 0 0;
      color: var(--muted-strong);
      font-size: clamp(1rem, 2vw, 1.12rem);
    }

    .hero-actions {
      display: flex;
      gap: 10px;
      flex-wrap: wrap;
      margin-top: 24px;
    }

    .button-link {
      display: inline-flex;
      align-items: center;
      justify-content: center;
      min-height: 40px;
      border-radius: 8px;
      border: 1px solid var(--border);
      padding: 8px 14px;
      text-decoration: none;
      font-weight: 750;
      transition: transform 0.15s ease, border-color 0.15s ease, background 0.15s ease;
    }

    .button-link:hover {
      transform: translateY(-1px);
      border-color: var(--border-strong);
    }

    .button-primary {
      border-color: var(--green);
      background: var(--green);
      color: #ffffff;
    }

    .button-secondary {
      background: #ffffff;
      color: var(--muted-strong);
    }

    .hero-panel {
      display: grid;
      gap: 12px;
    }

    .stat-grid {
      display: grid;
      grid-template-columns: repeat(2, minmax(0, 1fr));
      gap: 12px;
    }

    .stat,
    .path-panel,
    .toolbar,
    .book-card,
    .empty-state {
      border: 1px solid var(--border);
      border-radius: 8px;
      background: var(--surface);
    }

    .stat {
      padding: 16px;
    }

    .stat-value {
      display: block;
      font-size: 2rem;
      line-height: 1;
      font-weight: 850;
    }

    .stat-label {
      display: block;
      margin-top: 8px;
      color: var(--muted);
      font-size: 0.9rem;
    }

    .path-panel {
      padding: 16px;
    }

    .path-title {
      margin: 0 0 10px;
      color: var(--muted-strong);
      font-weight: 800;
      font-size: 0.92rem;
    }

    .chip-row {
      display: flex;
      gap: 8px;
      flex-wrap: wrap;
    }

    .chip {
      display: inline-flex;
      align-items: center;
      border-radius: 999px;
      background: var(--surface-soft);
      color: var(--muted-strong);
      padding: 5px 9px;
      font-size: 0.82rem;
      font-weight: 650;
    }

    .section-head {
      display: flex;
      align-items: flex-end;
      justify-content: space-between;
      gap: 16px;
      margin: 34px 0 14px;
    }

    h2 {
      margin: 0;
      font-size: clamp(1.35rem, 3vw, 2rem);
      line-height: 1.2;
    }

    .section-note {
      margin: 6px 0 0;
      color: var(--muted);
    }

    .result-count {
      color: var(--muted-strong);
      font-weight: 750;
      white-space: nowrap;
    }

    .toolbar {
      display: grid;
      grid-template-columns: minmax(240px, 1fr) auto;
      gap: 14px;
      align-items: center;
      padding: 14px;
      margin-bottom: 16px;
    }

    .search-field {
      display: flex;
      align-items: center;
      gap: 10px;
      min-width: 0;
      border: 1px solid var(--border);
      border-radius: 8px;
      background: #fbfdfb;
      padding: 9px 12px;
    }

    .search-label {
      color: var(--muted);
      font-weight: 750;
      white-space: nowrap;
    }

    .search-field input {
      width: 100%;
      min-width: 80px;
      border: 0;
      outline: 0;
      background: transparent;
      color: var(--text);
    }

    .filter-row {
      display: flex;
      justify-content: flex-end;
      gap: 8px;
      flex-wrap: wrap;
    }

    .filter-button {
      border: 1px solid var(--border);
      border-radius: 999px;
      background: #fbfdfb;
      color: var(--muted-strong);
      cursor: pointer;
      padding: 8px 11px;
      font-weight: 750;
      transition: background 0.15s ease, border-color 0.15s ease, color 0.15s ease;
    }

    .filter-button[aria-pressed="true"] {
      border-color: var(--green);
      background: var(--green);
      color: #ffffff;
    }

    .book-grid {
      display: grid;
      grid-template-columns: repeat(3, minmax(0, 1fr));
      gap: 14px;
    }

    .book-card {
      display: flex;
      flex-direction: column;
      min-height: 250px;
      padding: 16px;
      box-shadow: 0 10px 28px rgba(31, 58, 39, 0.05);
    }

    .book-topline {
      display: flex;
      align-items: center;
      justify-content: space-between;
      gap: 8px;
      margin-bottom: 12px;
    }

    .category-label {
      font-weight: 800;
      font-size: 0.82rem;
    }

    .category-technique {
      color: var(--green);
    }

    .category-training {
      color: var(--blue);
    }

    .category-body {
      color: var(--amber);
    }

    .category-biography {
      color: var(--rose);
    }

    .category-assessment {
      color: #53616f;
    }

    .file-type {
      flex: 0 0 auto;
      border: 1px solid var(--border);
      border-radius: 999px;
      color: var(--muted);
      padding: 3px 7px;
      font-size: 0.74rem;
      font-weight: 800;
    }

    .book-title {
      margin: 0;
      font-size: 1.04rem;
      line-height: 1.35;
    }

    .book-summary {
      flex: 1;
      margin: 10px 0 12px;
      color: var(--muted-strong);
      font-size: 0.94rem;
    }

    .tag-row {
      display: flex;
      gap: 6px;
      flex-wrap: wrap;
      margin-bottom: 14px;
    }

    .tag {
      border-radius: 999px;
      background: #edf3ef;
      color: var(--muted-strong);
      padding: 4px 7px;
      font-size: 0.76rem;
      font-weight: 700;
    }

    .download-row {
      display: flex;
      gap: 8px;
      align-items: center;
      justify-content: space-between;
      margin-top: auto;
    }

    .download-link {
      display: inline-flex;
      align-items: center;
      justify-content: center;
      min-height: 36px;
      border-radius: 8px;
      background: var(--green);
      color: #ffffff;
      padding: 7px 11px;
      text-decoration: none;
      font-weight: 800;
    }

    .audience {
      color: var(--muted);
      font-size: 0.82rem;
      text-align: right;
    }

    .empty-state {
      display: none;
      padding: 32px;
      text-align: center;
      color: var(--muted-strong);
    }

    .empty-state.is-visible {
      display: block;
    }

    footer {
      margin-top: 34px;
      padding-top: 20px;
      border-top: 1px solid var(--border);
      color: var(--muted);
      font-size: 0.9rem;
    }

    footer a {
      color: var(--green-dark);
      font-weight: 750;
    }

    .visually-hidden {
      position: absolute;
      width: 1px;
      height: 1px;
      padding: 0;
      margin: -1px;
      overflow: hidden;
      clip: rect(0, 0, 0, 0);
      white-space: nowrap;
      border: 0;
    }

    @media (max-width: 900px) {
      .hero,
      .toolbar {
        grid-template-columns: 1fr;
      }

      .book-grid {
        grid-template-columns: repeat(2, minmax(0, 1fr));
      }

      .filter-row {
        justify-content: flex-start;
      }
    }

    @media (max-width: 640px) {
      .page {
        width: min(100% - 24px, 1120px);
        padding-top: 12px;
      }

      .topbar,
      .section-head {
        align-items: flex-start;
        flex-direction: column;
      }

      .top-actions {
        justify-content: flex-start;
      }

      .hero {
        padding: 22px;
      }

      .stat-grid,
      .book-grid {
        grid-template-columns: 1fr;
      }

      .download-row {
        align-items: flex-start;
        flex-direction: column;
      }

      .audience {
        text-align: left;
      }
    }
  </style>
</head>
<body>
  <div class="page">
    <header class="topbar">
      <a class="brand" href="#library" aria-label="tennis-book">
        <span class="brand-mark">T</span>
        <span>
          <span class="brand-name">tennis-book</span>
          <span class="brand-subtitle" data-i18n="brandSubtitle">Tennis reading archive</span>
        </span>
      </a>
      <div class="top-actions">
        <span class="count-pill" data-i18n-count="topCount">17 份 PDF</span>
        <div class="lang-switch" aria-label="Language">
          <button class="lang-button" type="button" data-lang="zh" aria-pressed="true">中文</button>
          <button class="lang-button" type="button" data-lang="en" aria-pressed="false">English</button>
        </div>
      </div>
    </header>

    <main>
      <section class="hero" aria-labelledby="hero-title">
        <div>
          <p class="eyebrow" data-i18n="eyebrow">个人网球资料馆</p>
          <h1 id="hero-title" data-i18n="heroTitle">网球电子书与训练资料库</h1>
          <p class="intro" data-i18n="heroIntro">整理网球技术、专项训练、比赛心理、名将传记与运动科学资料，给学习者一条清晰的阅读入口。</p>
          <div class="hero-actions">
            <a class="button-link button-primary" href="#library" data-i18n="browseAction">浏览书目</a>
            <a class="button-link button-secondary" href="https://github.com/victorsheng/tennis-book" rel="noopener noreferrer" data-i18n="repoAction">GitHub 仓库</a>
          </div>
        </div>
        <aside class="hero-panel" aria-label="Library overview">
          <div class="stat-grid">
            <div class="stat">
              <span class="stat-value" id="stat-books">17</span>
              <span class="stat-label" data-i18n="statBooks">PDF 资料</span>
            </div>
            <div class="stat">
              <span class="stat-value" id="stat-categories">5</span>
              <span class="stat-label" data-i18n="statCategories">主题分类</span>
            </div>
          </div>
          <div class="path-panel">
            <p class="path-title" data-i18n="pathTitle">学习入口</p>
            <div class="chip-row" id="path-chips"></div>
          </div>
        </aside>
      </section>

      <section id="library" aria-labelledby="library-title">
        <div class="section-head">
          <div>
            <h2 id="library-title" data-i18n="libraryTitle">书目资料库</h2>
            <p class="section-note" data-i18n="libraryNote">按书名、主题、标签或简介快速找到下一份资料。</p>
          </div>
          <div class="result-count" id="result-count" aria-live="polite"></div>
        </div>

        <div class="toolbar">
          <label class="search-field">
            <span class="search-label" data-i18n="searchLabel">搜索</span>
            <input id="search-input" type="search" autocomplete="off" aria-label="Search books" />
          </label>
          <div class="filter-row" id="category-filters" aria-label="Book categories"></div>
        </div>

        <div class="book-grid" id="book-grid"></div>
        <div class="empty-state" id="empty-state">
          <h3 data-i18n="emptyTitle">没有找到匹配资料</h3>
          <p data-i18n="emptyBody">换一个关键词，或切回“全部”分类再试。</p>
        </div>
      </section>
    </main>

    <footer>
      <p data-i18n="footerText">本站由 GitHub Pages 托管。PDF 通过 Raw 链接从仓库直链下载；请遵守版权，仅限个人学习使用。</p>
    </footer>

    <noscript>
      <p>此页面需要 JavaScript 才能显示搜索、筛选和完整书目。你仍可在 GitHub 仓库中查看 PDF 文件。</p>
    </noscript>
  </div>

  <script>
    (function () {
      const OWNER = "victorsheng";
      const REPO = "tennis-book";
      const BRANCH = "master";
      const RAW_BASE = `https://raw.githubusercontent.com/${OWNER}/${REPO}/${BRANCH}/`;
      const LANG_KEY = "tennis-book-lang";

      const CATEGORY_ORDER = ["all", "technique", "training", "body", "biography", "assessment"];

      const CATEGORIES = {
        all: { zh: "全部", en: "All" },
        technique: { zh: "技术与战术", en: "Technique & Tactics" },
        training: { zh: "训练与心理", en: "Training & Mindset" },
        body: { zh: "身体与科学", en: "Body & Science" },
        biography: { zh: "名将传记", en: "Player Stories" },
        assessment: { zh: "规则与测评", en: "Rules & Assessment" }
      };

      const PATH_CHIPS = {
        zh: ["技术精进", "专项训练", "身体维护", "冠军故事"],
        en: ["Technique", "Training", "Body care", "Champion stories"]
      };

      const UI = {
        zh: {
          brandSubtitle: "Tennis reading archive",
          topCount: "17 份 PDF",
          eyebrow: "个人网球资料馆",
          heroTitle: "网球电子书与训练资料库",
          heroIntro: "整理网球技术、专项训练、比赛心理、名将传记与运动科学资料，给学习者一条清晰的阅读入口。",
          browseAction: "浏览书目",
          repoAction: "GitHub 仓库",
          statBooks: "PDF 资料",
          statCategories: "主题分类",
          pathTitle: "学习入口",
          libraryTitle: "书目资料库",
          libraryNote: "按书名、主题、标签或简介快速找到下一份资料。",
          searchLabel: "搜索",
          emptyTitle: "没有找到匹配资料",
          emptyBody: "换一个关键词，或切回“全部”分类再试。",
          footerText: "本站由 GitHub Pages 托管。PDF 通过 Raw 链接从仓库直链下载；请遵守版权，仅限个人学习使用。",
          download: "下载 PDF",
          result: (count, total) => `显示 ${count} / ${total} 本`
        },
        en: {
          brandSubtitle: "Tennis reading archive",
          topCount: "17 PDFs",
          eyebrow: "Personal tennis archive",
          heroTitle: "Tennis Books & Training Library",
          heroIntro: "A curated collection of tennis technique, structured practice, match mindset, player stories, and body-care resources for focused study.",
          browseAction: "Browse books",
          repoAction: "GitHub repo",
          statBooks: "PDF resources",
          statCategories: "Topics",
          pathTitle: "Study paths",
          libraryTitle: "Library",
          libraryNote: "Search by title, topic, tag, or summary to find the next resource.",
          searchLabel: "Search",
          emptyTitle: "No matching resources",
          emptyBody: "Try another keyword or switch back to All.",
          footerText: "Hosted with GitHub Pages. PDFs are downloaded through Raw links from the repository; please respect copyright and use them for personal study only.",
          download: "Download PDF",
          result: (count, total) => `${count} of ${total} shown`
        }
      };

      const BOOKS = [
        {
          file: "[网球技术精解全书].《网球》杂志.著.插图版.pdf",
          title: { zh: "网球技术精解全书", en: "Complete Illustrated Guide to Tennis Technique" },
          category: "technique",
          tags: { zh: ["技术图解", "动作细节", "基础到进阶"], en: ["illustrated", "stroke mechanics", "fundamentals"] },
          summary: { zh: "以插图和分解动作梳理网球关键技术，适合校准击球动作与基础框架。", en: "An illustrated technique reference for studying stroke mechanics and building a reliable fundamentals framework." },
          audience: { zh: "技术学习", en: "Technique study" }
        },
        {
          file: "tennis淺談網球步法訓練.pdf",
          title: { zh: "浅谈网球步法训练", en: "A Brief Guide to Tennis Footwork Training" },
          category: "training",
          tags: { zh: ["步法", "移动", "训练"], en: ["footwork", "movement", "practice"] },
          summary: { zh: "聚焦网球移动与步法训练，适合作为场上移动节奏和位置意识的补充资料。", en: "A compact resource on tennis movement and footwork practice for improving court positioning and movement rhythm." },
          audience: { zh: "移动训练", en: "Movement work" }
        },
        {
          file: "中国网协CTN网球等级智能测评指南.pdf",
          title: { zh: "中国网协 CTN 网球等级智能测评指南", en: "CTA CTN Intelligent Rating Assessment Guide" },
          category: "assessment",
          tags: { zh: ["等级测评", "CTN", "标准"], en: ["rating", "CTN", "assessment"] },
          summary: { zh: "中国网协 CTN 等级测评参考资料，适合理解网球水平评估标准和测试流程。", en: "A China Tennis Association CTN reference for understanding player rating standards and assessment flow." },
          audience: { zh: "水平测评", en: "Rating reference" }
        },
        {
          file: "实用网球技巧提升200  _13225880.pdf",
          title: { zh: "实用网球技巧提升 200", en: "200 Practical Tennis Improvement Tips" },
          category: "technique",
          tags: { zh: ["技巧", "实用练习", "提升"], en: ["tips", "practice ideas", "improvement"] },
          summary: { zh: "围绕常见技术与实战问题提供大量技巧建议，适合按问题查找改进方向。", en: "A practical tip collection for common technical and match-play problems, useful for targeted improvement." },
          audience: { zh: "日常提升", en: "Everyday improvement" }
        },
        {
          file: "德约科维奇  一发制胜  我的14天身心逆转计划_13648186.pdf",
          title: { zh: "德约科维奇：一发制胜", en: "Serve to Win: Djokovic's 14-Day Plan" },
          category: "biography",
          tags: { zh: ["德约科维奇", "饮食", "身心"], en: ["Djokovic", "nutrition", "mind-body"] },
          summary: { zh: "德约科维奇分享饮食、恢复、专注与生活习惯，呈现其身心状态转变的方法。", en: "Djokovic shares nutrition, recovery, focus, and lifestyle habits behind his mind-body reset." },
          audience: { zh: "冠军习惯", en: "Champion habits" }
        },
        {
          file: "拉法纳达尔自传_13400544.pdf",
          title: { zh: "拉法纳达尔自传", en: "Rafa: My Story" },
          category: "biography",
          tags: { zh: ["纳达尔", "传记", "竞争心态"], en: ["Nadal", "memoir", "competitive mindset"] },
          summary: { zh: "通过纳达尔的成长和职业经历，理解顶尖球员的训练文化、意志和比赛心态。", en: "A memoir of Nadal's development and career, revealing the training culture and mindset behind elite competition." },
          audience: { zh: "球员故事", en: "Player story" }
        },
        {
          file: "牵伸解剖指南.pdf",
          title: { zh: "牵伸解剖指南", en: "Stretching Anatomy Guide" },
          category: "body",
          tags: { zh: ["解剖", "柔韧性", "拉伸"], en: ["anatomy", "mobility", "stretching"] },
          summary: { zh: "从肌肉和关节结构理解拉伸动作，适合补充网球训练中的柔韧性和身体维护。", en: "An anatomy-oriented stretching reference for understanding muscles, joints, mobility, and body maintenance." },
          audience: { zh: "身体维护", en: "Body care" }
        },
        {
          file: "独自上场_13077137.pdf",
          title: { zh: "独自上场", en: "Open: An Autobiography" },
          category: "biography",
          tags: { zh: ["阿加西", "自传", "心理"], en: ["Agassi", "autobiography", "inner game"] },
          summary: { zh: "阿加西自传，以个人视角呈现职业网球、压力、身份和自我重建。", en: "Agassi's autobiography, tracing professional tennis, pressure, identity, and personal reinvention from the inside." },
          audience: { zh: "传记阅读", en: "Memoir reading" }
        },
        {
          file: "精准拉伸：疼痛消除和损伤预防的针对性练习.pdf",
          title: { zh: "精准拉伸：疼痛消除和损伤预防的针对性练习", en: "Prescriptive Stretching" },
          category: "body",
          tags: { zh: ["疼痛缓解", "损伤预防", "针对性拉伸"], en: ["pain relief", "injury prevention", "targeted stretching"] },
          summary: { zh: "按肌肉和疼痛问题组织针对性拉伸练习，适合运动后的恢复和常见不适管理。", en: "A targeted stretching guide organized around muscles and pain patterns, useful for recovery and common discomfort management." },
          audience: { zh: "恢复拉伸", en: "Recovery work" }
        },
        {
          file: "網球截擊技術及雙打「搶打戰術」之應用分析.pdf",
          title: { zh: "网球截击技术及双打抢打战术之应用分析", en: "Volley Technique and Doubles Poaching Tactics" },
          category: "technique",
          tags: { zh: ["截击", "双打", "战术"], en: ["volley", "doubles", "poaching"] },
          summary: { zh: "分析截击技术和双打抢打战术的应用，适合提升网前处理与双打意识。", en: "An analysis of volley technique and doubles poaching tactics for improving net play and doubles awareness." },
          audience: { zh: "双打战术", en: "Doubles tactics" }
        },
        {
          file: "网球制胜：实用技战术图解=Winning Tennis_13735994.pdf",
          title: { zh: "网球制胜：实用技战术图解", en: "Winning Tennis: Practical Technique and Tactics Diagrams" },
          category: "technique",
          tags: { zh: ["技战术", "图解", "实战"], en: ["tactics", "diagrams", "match play"] },
          summary: { zh: "用图解方式连接技术动作和比赛战术，适合把训练内容转化为场上决策。", en: "A diagram-based bridge between technique and tactics, useful for turning practice into better match decisions." },
          audience: { zh: "实战策略", en: "Match strategy" }
        },
        {
          file: "网球压力训练_上册.pdf",
          title: { zh: "网球压力训练（上册）", en: "Tennis Pressure Training, Vol. 1" },
          category: "training",
          tags: { zh: ["压力训练", "比赛心理", "练习设计"], en: ["pressure training", "mindset", "drills"] },
          summary: { zh: "围绕压力情境设计训练，帮助球员把技术稳定性带入更接近比赛的环境。", en: "A pressure-based practice resource for helping players carry technical stability into match-like situations." },
          audience: { zh: "比赛训练", en: "Match practice" }
        },
        {
          file: "网球压力训练_下册.pdf",
          title: { zh: "网球压力训练（下册）", en: "Tennis Pressure Training, Vol. 2" },
          category: "training",
          tags: { zh: ["压力训练", "进阶练习", "实战"], en: ["pressure training", "advanced drills", "competition"] },
          summary: { zh: "延续压力训练主题，补充更多练习组织和实战化训练思路。", en: "A continuation of pressure training with additional practice structures and competition-oriented ideas." },
          audience: { zh: "进阶训练", en: "Advanced practice" }
        },
        {
          file: "网球提升  基础技巧与实战策略_13831982.pdf",
          title: { zh: "网球提升：基础技巧与实战策略", en: "Tennis Improvement: Basic Skills and Match Strategy" },
          category: "technique",
          tags: { zh: ["基础技巧", "实战策略", "提升"], en: ["basic skills", "strategy", "improvement"] },
          summary: { zh: "结合基础技术和实战策略，适合从动作学习过渡到比赛应用的阶段。", en: "Combines basic skills with match strategy for players moving from technique learning into practical play." },
          audience: { zh: "基础进阶", en: "Fundamentals plus" }
        },
        {
          file: "网球运动教程.陶志翔.pdf",
          title: { zh: "网球运动教程", en: "Tennis Course" },
          category: "technique",
          tags: { zh: ["教程", "基础", "教学"], en: ["course", "fundamentals", "teaching"] },
          summary: { zh: "网球教学型资料，覆盖运动基础、技术学习和课程化训练内容。", en: "A teaching-oriented tennis course covering fundamentals, technique learning, and structured instruction." },
          audience: { zh: "系统入门", en: "Structured basics" }
        },
        {
          file: "网球运动系统训练13800928.pdf",
          title: { zh: "网球运动系统训练", en: "Systematic Training for Tennis" },
          category: "training",
          tags: { zh: ["系统训练", "专项能力", "体能"], en: ["systematic training", "specific conditioning", "fitness"] },
          summary: { zh: "围绕网球专项能力建立系统训练框架，适合规划体能、技术和训练周期。", en: "A systematic tennis training reference for planning conditioning, technical work, and practice cycles." },
          audience: { zh: "训练规划", en: "Training plans" }
        },
        {
          file: "费德勒传 光影中的网坛传奇.pdf",
          title: { zh: "费德勒传：光影中的网坛传奇", en: "Roger Federer: A Legend in Light and Shadow" },
          category: "biography",
          tags: { zh: ["费德勒", "传记", "职业生涯"], en: ["Federer", "biography", "career"] },
          summary: { zh: "回顾费德勒的职业轨迹、比赛风格和时代影响，适合理解现代网球传奇。", en: "A biographical portrait of Federer's career, playing style, and influence on the modern tennis era." },
          audience: { zh: "传记阅读", en: "Player story" }
        }
      ];

      let currentLang = localStorage.getItem(LANG_KEY) === "en" ? "en" : "zh";
      let currentCategory = "all";

      const nodes = {
        html: document.documentElement,
        translatable: document.querySelectorAll("[data-i18n]"),
        topCount: document.querySelector("[data-i18n-count='topCount']"),
        langButtons: document.querySelectorAll(".lang-button"),
        pathChips: document.getElementById("path-chips"),
        categoryFilters: document.getElementById("category-filters"),
        searchInput: document.getElementById("search-input"),
        resultCount: document.getElementById("result-count"),
        bookGrid: document.getElementById("book-grid"),
        emptyState: document.getElementById("empty-state"),
        statBooks: document.getElementById("stat-books"),
        statCategories: document.getElementById("stat-categories")
      };

      function normalize(value) {
        return String(value || "").trim().toLocaleLowerCase();
      }

      function getSearchText(book) {
        return normalize([
          book.file,
          book.title.zh,
          book.title.en,
          CATEGORIES[book.category].zh,
          CATEGORIES[book.category].en,
          book.summary.zh,
          book.summary.en,
          book.audience.zh,
          book.audience.en,
          ...book.tags.zh,
          ...book.tags.en
        ].join(" "));
      }

      function setLanguage(lang) {
        currentLang = lang === "en" ? "en" : "zh";
        localStorage.setItem(LANG_KEY, currentLang);
        applyLanguage();
        render();
      }

      function applyLanguage() {
        const ui = UI[currentLang];
        nodes.html.lang = currentLang === "zh" ? "zh-CN" : "en";
        document.title = currentLang === "zh" ? "tennis-book · 网球资料库" : "tennis-book · Tennis Library";

        nodes.translatable.forEach((node) => {
          const key = node.dataset.i18n;
          node.textContent = ui[key];
        });

        nodes.topCount.textContent = ui.topCount;
        nodes.statBooks.textContent = BOOKS.length;
        nodes.statCategories.textContent = CATEGORY_ORDER.length - 1;

        nodes.langButtons.forEach((button) => {
          const active = button.dataset.lang === currentLang;
          button.setAttribute("aria-pressed", String(active));
        });

        nodes.pathChips.innerHTML = "";
        PATH_CHIPS[currentLang].forEach((label) => {
          const chip = document.createElement("span");
          chip.className = "chip";
          chip.textContent = label;
          nodes.pathChips.appendChild(chip);
        });

        renderCategoryButtons();
      }

      function renderCategoryButtons() {
        nodes.categoryFilters.innerHTML = "";
        CATEGORY_ORDER.forEach((key) => {
          const button = document.createElement("button");
          button.type = "button";
          button.className = "filter-button";
          button.dataset.category = key;
          button.setAttribute("aria-pressed", String(key === currentCategory));
          button.textContent = CATEGORIES[key][currentLang];
          button.addEventListener("click", () => {
            currentCategory = key;
            renderCategoryButtons();
            render();
          });
          nodes.categoryFilters.appendChild(button);
        });
      }

      function getFilteredBooks() {
        const query = normalize(nodes.searchInput.value);
        return BOOKS.filter((book) => {
          const matchesCategory = currentCategory === "all" || book.category === currentCategory;
          const matchesSearch = !query || getSearchText(book).includes(query);
          return matchesCategory && matchesSearch;
        });
      }

      function createBookCard(book) {
        const card = document.createElement("article");
        card.className = "book-card";

        const topLine = document.createElement("div");
        topLine.className = "book-topline";

        const category = document.createElement("span");
        category.className = `category-label category-${book.category}`;
        category.textContent = CATEGORIES[book.category][currentLang];

        const type = document.createElement("span");
        type.className = "file-type";
        type.textContent = "PDF";

        topLine.append(category, type);

        const title = document.createElement("h3");
        title.className = "book-title";
        title.textContent = book.title[currentLang];

        const summary = document.createElement("p");
        summary.className = "book-summary";
        summary.textContent = book.summary[currentLang];

        const tags = document.createElement("div");
        tags.className = "tag-row";
        book.tags[currentLang].forEach((label) => {
          const tag = document.createElement("span");
          tag.className = "tag";
          tag.textContent = label;
          tags.appendChild(tag);
        });

        const bottom = document.createElement("div");
        bottom.className = "download-row";

        const download = document.createElement("a");
        download.className = "download-link";
        download.href = RAW_BASE + encodeURIComponent(book.file);
        download.download = "";
        download.rel = "noopener noreferrer";
        download.textContent = UI[currentLang].download;

        const audience = document.createElement("span");
        audience.className = "audience";
        audience.textContent = book.audience[currentLang];

        bottom.append(download, audience);
        card.append(topLine, title, summary, tags, bottom);
        return card;
      }

      function render() {
        const filteredBooks = getFilteredBooks();
        nodes.resultCount.textContent = UI[currentLang].result(filteredBooks.length, BOOKS.length);
        nodes.bookGrid.innerHTML = "";
        filteredBooks.forEach((book) => nodes.bookGrid.appendChild(createBookCard(book)));
        nodes.emptyState.classList.toggle("is-visible", filteredBooks.length === 0);
      }

      nodes.langButtons.forEach((button) => {
        button.addEventListener("click", () => setLanguage(button.dataset.lang));
      });

      nodes.searchInput.addEventListener("input", render);

      applyLanguage();
      render();
    })();
  </script>
</body>
</html>
```
