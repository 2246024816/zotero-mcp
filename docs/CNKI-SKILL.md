---
name: cnki
description: CNKI (中国知网) literature toolkit — search, download PDF, and import to Zotero with attachments. Uses CDP Proxy for browser automation and zotero-mcp for library management.
argument-hint: "[CNKI task: search/download/export to zotero + query or URL]"
---

# CNKI Research Toolkit

Search CNKI (中国知网), download papers, and import them to Zotero **with PDF attachments**.

## Prerequisites

| Component | Purpose |
|-----------|---------|
| Zotero 7+ | Literature library |
| Zotero MCP Plugin v1.4.8+ | LLM ↔ Zotero bridge (includes `importAttachment`) |
| Node.js 22+ | CDP Proxy runtime |
| Chrome + remote debugging | Browser automation |
| CDP Proxy (web-access skill) | curl → Chrome bridge |
| CNKI login | Required for PDF download |

## Intent Routing

| User intent | Section |
|-------------|---------|
| Search papers by keyword | Basic Search |
| Extract paper metadata | Parse Search Results |
| Next/previous/page, sort | Pagination & Sorting |
| Export single paper to Zotero (with PDF) | Export to Zotero |
| Download PDF/CAJ | Paper Download |
| Browse journal issues | Journal TOC |
| Search journals, check indexing | Journal Search / Indexing |

## Global Rules

### CDP Proxy API

All browser operations use the CDP Proxy via `curl`:

```bash
# Open new tab
curl -s "http://localhost:3456/new?url=URL"

# Evaluate JS in page
curl -s -X POST "http://localhost:3456/eval?target=TARGET_ID" -d 'javascript code'

# Navigate existing tab
curl -s "http://localhost:3456/navigate?target=TARGET_ID&url=URL"

# Click element
curl -s -X POST "http://localhost:3456/click?target=TARGET_ID" -d 'selector'

# Take screenshot
curl -s "http://localhost:3456/screenshot?target=TARGET_ID&file=/tmp/name.png"

# Close tab
curl -s "http://localhost:3456/close?target=TARGET_ID"
```

### Key Rules

- **Prefer `navigate_page` to detail URLs over clicking links** — clicking opens new tabs and wastes tool calls.
- **For PDF download: use `location.href` jump**, not `click()` — it preserves the CNKI referrer header. Opening `bar.cnki.net` directly in a new tab is blocked ("来源应用不正确").
- **Captcha**: check `#tcaptcha_transform_dy` with `getBoundingClientRect().top >= 0`. Hidden SDK nodes at `top: -1000000px` are not active captchas. If active, ask user to solve in Chrome.
- **Chinese filenames**: CNKI downloads PDFs with smart quotes (""). Use `find` + `-exec cp` to handle them safely.
- **Wait for page loads**: use async `evaluate_script` with polling patterns (check for "条结果" text), not separate `wait_for`.

---

# Basic Search

## Steps

### 1. Open CNKI

```bash
curl -s "http://localhost:3456/new?url=https://kns.cnki.net/kns8s/search"
# Returns: {"targetId":"..."}
```

Wait 3-5s for page load.

### 2. Search + extract results (single evaluate_script)

Replace `KEYWORDS` with actual search terms:

```javascript
(async () => {
  const query = "KEYWORDS";

  // Wait for search input
  await new Promise((r, j) => {
    let n = 0;
    const c = () => { if (document.querySelector('input.search-input')) r(); else if (++n > 30) j('timeout'); else setTimeout(c, 500); };
    c();
  });

  // Fill and submit
  const input = document.querySelector('input.search-input');
  input.value = query;
  input.dispatchEvent(new Event('input', { bubbles: true }));
  document.querySelector('input.search-btn')?.click();

  // Wait for results
  await new Promise((r, j) => {
    let n = 0;
    const c = () => { if (document.body.innerText.includes('条结果')) r(); else if (++n > 30) j('timeout'); else setTimeout(c, 500); };
    c();
  });

  // Extract results
  const rows = document.querySelectorAll('.result-table-list tbody tr');
  const checkboxes = document.querySelectorAll('.result-table-list tbody input.cbItem');
  const results = Array.from(rows).map((row, i) => ({
    n: i + 1,
    title: row.querySelector('td.name a.fz14')?.innerText?.trim() || '',
    href: row.querySelector('td.name a.fz14')?.href || '',
    authors: Array.from(row.querySelectorAll('td.author a.KnowledgeNetLink')).map(a => a.innerText?.trim()).join('; '),
    journal: row.querySelector('td.source a')?.innerText?.trim() || '',
    date: row.querySelector('td.date')?.innerText?.trim() || '',
    citations: row.querySelector('td.quote')?.innerText?.trim() || '',
    downloads: row.querySelector('td.download')?.innerText?.trim() || ''
  }));

  return {
    query,
    total: document.querySelector('.pagerTitleCell')?.innerText?.match(/([\d,]+)/)?.[1] || '0',
    page: document.querySelector('.countPageMark')?.innerText || '1/1',
    results
  };
})()
```

### 3. Report

Present as a numbered list:
```
Search CNKI for "KEYWORDS": found {total} results (page {page}).

1. {title}
   Authors: {authors} | Journal: {journal} | Date: {date}
   Citations: {citations} | Downloads: {downloads}
```

---

# Parse Search Results

If already on a results page, extract structured data:

```javascript
(() => {
  const rows = document.querySelectorAll('.result-table-list tbody tr');
  const results = Array.from(rows).map((row, i) => ({
    number: i + 1,
    title: row.querySelector('td.name a.fz14')?.innerText?.trim() || '',
    url: row.querySelector('td.name a.fz14')?.href || '',
    authors: Array.from(row.querySelectorAll('td.author a.KnowledgeNetLink')).map(a => a.innerText?.trim()),
    journal: row.querySelector('td.source a')?.innerText?.trim() || '',
    date: row.querySelector('td.date')?.innerText?.trim() || '',
    citations: row.querySelector('td.quote')?.innerText?.trim() || '',
    downloads: row.querySelector('td.download')?.innerText?.trim() || ''
  }));
  return {
    papers: results,
    totalCount: (document.querySelector('.pagerTitleCell')?.innerText?.match(/([\d,]+)/) || [])[1] || 'unknown',
    pageInfo: document.querySelector('.countPageMark')?.innerText || ''
  };
})()
```

## Verified Selectors

| Data | Selector | Notes |
|------|----------|-------|
| Search input | `input.search-input` | id=`txt_search` |
| Search button | `input.search-btn` | |
| Result rows | `.result-table-list tbody tr` | |
| Title link | `td.name a.fz14` | href to detail page |
| Authors | `td.author a.KnowledgeNetLink` | |
| Journal | `td.source a` | |
| Date | `td.date` | |
| Citations | `td.quote` | |
| Downloads | `td.download` | |
| Total count | `.pagerTitleCell` | "共找到 X 条结果" |
| Page indicator | `.countPageMark` | "1/42" |

---

# Pagination & Sorting

Single `evaluate_script` call — replace `ACTION` with `"next"`, `"previous"`, `"page 3"`, or sort with `"date"`, `"citations"`, `"downloads"`.

```javascript
(async () => {
  const action = "next"; // or "date", "citations", "downloads"
  const prevMark = document.querySelector('.countPageMark')?.innerText;

  if (action === 'next') {
    Array.from(document.querySelectorAll('.pages a')).find(a => a.innerText.trim() === '下一页')?.click();
  } else if (action === 'previous') {
    Array.from(document.querySelectorAll('.pages a')).find(a => a.innerText.trim() === '上一页')?.click();
  } else {
    // Sorting: FFD=relevance, PT=date, CF=citations, DFR=downloads
    const idMap = { date: 'PT', citations: 'CF', downloads: 'DFR' };
    document.querySelector('#orderList li#' + idMap[action])?.click();
  }

  await new Promise((r, j) => {
    let n = 0;
    const c = () => { if (document.querySelector('.countPageMark')?.innerText !== prevMark) r(); else if (++n > 30) j('timeout'); else setTimeout(c, 500); };
    setTimeout(c, 1000);
  });

  return {
    total: document.querySelector('.pagerTitleCell')?.innerText?.match(/([\d,]+)/)?.[1] || '0',
    page: document.querySelector('.countPageMark')?.innerText || '?',
    url: location.href
  };
})()
```

---

# Export to Zotero (Recommended: metadata + PDF)

The validated workflow that reliably produces a complete Zotero item with PDF:

## Single Paper Export

### Step 1: Search and find the paper

Use Basic Search to locate the paper. Extract `href` and basic metadata from results.

### Step 2: Navigate to detail page, get metadata

```bash
curl -s "http://localhost:3456/navigate?target=TARGET_ID&url=PAPER_HREF"
```

Wait 6s, then extract metadata:

```javascript
(async () => {
  const exportId = document.querySelector('#export-id')?.value;
  const resp = await fetch('https://kns.cnki.net/dm8/API/GetExport', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({ filename: exportId, displaymode: 'GBTREFER,elearning,EndNote', uniplatform: 'NZKPT' })
  });
  const data = await resp.json();
  const result = { title: document.querySelector('.brief h1')?.innerText?.trim() || '' };
  for (const item of data.data) { result[item.mode] = item.value[0]; }
  const body = document.body.innerText;
  result.issn = body.match(/ISSN[：:]\s*(\S+)/)?.[1] || '';
  result.doi = body.match(/DOI[：:]\s*(\S+)/)?.[1] || '';
  result.hasPDF = !!document.querySelector('.btn-dlpdf a');
  result.pdfHref = document.querySelector('.btn-dlpdf a')?.href || null;
  return result;
})()
```

### Step 3: Create Zotero item via MCP

From the ELEARNING field, extract: title, authors, journal, year, date, issue, pages, keywords, abstract, ISSN, DOI, organization.

Use `mcp__zotero-mcp__write_item`:
```
action: create
itemType: journalArticle
creators: [{creatorType: "author", name: "作者名"}]
fields: {title, abstractNote, date, issue, pages, publicationTitle, ISSN, DOI, url, language: "zh-CN"}
tags: [keyword1, keyword2, ...]
```

Returns `itemKey` for the new entry.

### Step 4: Download PDF

**Use `location.href` jump** (preserves CNKI referrer):

```javascript
(() => {
  const pdfLink = document.querySelector('.btn-dlpdf a');
  window.location.href = pdfLink.href;
  return { navigating: true };
})()
```

Wait 15s for download to complete. Find the file with `find ~/Downloads -name "*.pdf" -mmin -2`.

**Handle special characters** in Chinese filenames (smart quotes):

```bash
find ~/Downloads -name "*keyword*" -mmin -3 -type f -exec cp {} /tmp/paper.pdf \;
```

### Step 5: Attach PDF to Zotero item

Use `mcp__zotero-mcp__write_item` (v1.4.8+):
```
action: importAttachment
parentKey: {itemKey from step 3}
filePath: /tmp/paper.pdf
importMode: import   # copies to Zotero storage
```

Returns `attachmentKey` — done!

### Step 6: Clean up

```bash
curl -s "http://localhost:3456/close?target=TARGET_ID"
rm /tmp/paper.pdf
```

---

## Batch Export (Multiple Papers)

Loop Steps 2-5 for each paper. Use `navigate_page` (not click) to go to each detail page.

**Important**: Navigate in the SAME tab (sequential `navigate_page` calls) — each page load is a fresh context.

---

# Paper Download (PDF only)

When user only wants the PDF file (not Zotero import):

1. Navigate to detail page
2. Check `#tcaptcha_transform_dy` for captcha
3. Check login status: `.downloadlink.icon-notlogged`
4. Use `location.href` to the PDF link (NOT click, NOT new tab)
5. Wait for download in `~/Downloads/`

## Verified Selectors

| Element | Selector | Notes |
|---------|----------|-------|
| PDF download | `.btn-dlpdf a` | href to `bar.cnki.net/bar/download/order` |
| CAJ download | `.btn-dlcaj a` | |
| Not logged in | `.downloadlink.icon-notlogged` | |
| Title | `.brief h1` | |

---

# Paper Detail Extraction

Extract complete metadata from a CNKI paper detail page:

```javascript
(() => {
  const brief = document.querySelector('.brief');
  if (!brief) return { error: 'Paper detail section not found' };

  const title = brief.querySelector('h1')?.innerText?.trim()?.replace(/\s*网络首发\s*$/, '');
  const authorH3s = brief.querySelectorAll('h3.author');
  const authors = [];
  if (authorH3s[0]) {
    authorH3s[0].querySelectorAll('a').forEach(a => {
      authors.push({ name: a.innerText?.replace(/\d+$/, '').trim() });
    });
  }
  const affiliations = [];
  if (authorH3s.length > 1) {
    authorH3s[1].querySelectorAll('a').forEach(a => affiliations.push(a.innerText?.trim()));
  }

  return {
    title,
    authors,
    affiliations,
    abstract: document.querySelector('.abstract-text')?.innerText?.trim() || '',
    keywords: Array.from(document.querySelectorAll('p.keywords a')).map(a => a.innerText?.replace(/;$/, '').trim()),
    fund: document.querySelector('p.funds')?.innerText?.trim() || '',
    journal: document.querySelector('.doc-top a')?.innerText?.trim() || '',
    pubInfo: document.querySelector('.head-time')?.innerText?.trim() || ''
  };
})()
```

---

# Journal Operations

## Journal Search

Navigate to `https://navi.cnki.net/knavi`, search by name/ISSN/CN.

## Journal Indexing

Check which databases index a journal. Navigate to journal detail page (`navi.cnki.net/knavi/detail`), extract indexing tags (北大核心, CSSCI, CSCD, SCI, EI, etc.) and impact factors.

## Journal TOC

Browse journal issue table of contents. Select year/issue from `#yearissue0 dl.s-dataList`, extract papers from `#CataLogContent dd.row`.

---

# Captcha Detection

```javascript
const cap = document.querySelector('#tcaptcha_transform_dy');
if (cap && cap.getBoundingClientRect().top >= 0) {
  return { error: 'captcha', message: '请在 Chrome 中手动完成滑块验证' };
}
```

Tencent captcha SDK preloads DOM at `top: -1000000px` (not active). Only act when `top >= 0` (visible).

---

# CNKI Export API Reference

| Parameter | Value | Source |
|-----------|-------|--------|
| API URL | `https://kns.cnki.net/dm8/API/GetExport` | Fixed |
| filename | Encrypted ID | Detail: `#export-id`, Results: `input.cbItem` value |
| displaymode | `GBTREFER,elearning,EndNote` | Comma-separated |
| uniplatform | `NZKPT` | Required |

---

# Zotero MCP API Reference

## Create Item

```
mcp__zotero-mcp__write_item({
  action: "create",
  itemType: "journalArticle",
  creators: [{ creatorType: "author", name: "作者名" }],
  fields: {
    title, abstractNote, date, issue, pages,
    publicationTitle, ISSN, DOI, url, language: "zh-CN"
  },
  tags: ["keyword1", "keyword2"]
})
// → { itemKey: "ABC123" }
```

## Import PDF Attachment (v1.4.8+)

```
mcp__zotero-mcp__write_item({
  action: "importAttachment",
  parentKey: "ABC123",
  filePath: "/tmp/paper.pdf",
  importMode: "import"   // or "link" for linked file
})
// → { attachmentKey: "XYZ789" }
```
