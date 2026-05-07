---
domain: cnki.net
aliases: [中国知网, CNKI]
updated: 2026-05-07
---

## 平台特征
- 搜索页面 `kns.cnki.net/kns8s/search` 可能触发腾讯滑块验证码（`#tcaptcha_transform_dy`），需要用户手动完成
- 验证码通过后页面标题变为"检索-中国知网"
- 论文详情页 URL 包含 `kcms2/article/abstract`
- PDF 下载链接为 `bar.cnki.net/bar/download/order?id=...`，需要正确的 Referer 头

## 有效模式

### 搜索论文
1. CDP Proxy `new` → `https://kns.cnki.net/kns8s/search`
2. `eval` 填入关键词并点击搜索按钮（`input.search-input` + `input.search-btn`）
3. 等待页面包含"条结果"后提取数据
4. 结果选择器：`.result-table-list tbody tr`，标题链接 `td.name a.fz14`

### 下载 PDF
**唯一可靠方式：`location.href` 跳转**
- 从论文详情页用 JS `window.location.href = pdfLink.href` 跳转到下载链接
- 这保留了 CNKI 需要的 Referer 头
- 直接 `new` 新标签页会被拦截："来源应用不正确(01)"
- `.btn-dlpdf a` 选择器获取 PDF 下载链接

### 导出到 Zotero（带 PDF）
**全流程（已验证可靠）**：
1. 搜索论文 → 获取 `href` 和基本信息
2. `navigate_page` 到详情页
3. CNKI Export API 获取完整元数据（`#export-id` + `GetExport`）
4. `zotero-mcp write_item(action: create)` 创建 Zotero 条目
5. `location.href` 触发 PDF 下载
6. 等待 15 秒下载完成
7. `find` + `cp` 处理中文文件名（智能引号问题）
8. `zotero-mcp write_item(action: importAttachment)` 附加 PDF → 需要 Zotero MCP Plugin v1.4.8+

### Zotero Connector（不可靠，不推荐）
- Connector 有时自动捕获元数据+PDF，有时不触发
- 触发条件不明确，依赖页面结构、Connector 状态等因素
- 不建议作为主要工作流，作为意外惊喜即可

## 已知陷阱
- **PDF 下载需要 Referer**：必须从详情页通过 `location.href` 跳转，不能开新标签页
- **中文文件名有智能引号**：CNKI 下载的 PDF 文件名可能包含 "" 等 Unicode 引号，bash 无法直接处理，需用 `find -exec cp`
- **验证码**：频繁操作后触发，需用户手动在 Chrome 中完成滑块验证
- **Zotero Connector 不可靠**：不要依赖它自动捕获，使用 `importAttachment` 作为确定性方案
- **`click()` 不触发下载**：JS 和真实鼠标点击都可能不生效，`location.href` 是最可靠的方式
