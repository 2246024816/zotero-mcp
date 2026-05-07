# CNKI → Zotero 全自动文献导入系统

## 概述

实现了一个从知网搜索论文到 Zotero 完整导入（含 PDF 附件）的全自动工作流。LLM Agent 通过 CDP Proxy 操控用户 Chrome 浏览器访问知网，搜索、下载论文 PDF，再通过扩展后的 Zotero MCP Plugin 自动创建条目并附加 PDF。

## 动机

传统文献管理流程：
```
手动搜索知网 → 手动下载PDF → 手动创建Zotero条目 → 手动拖入PDF → 手动填写元数据
```

我们实现的全自动流程：
```
用户说"下载这篇论文" → Agent 自动完成所有步骤 → 结果在 Zotero 里等着
```

中间经历了多次踩坑才找到可靠的实现路径。

## 核心组件

```
┌─────────────────────────────────────────────────────────┐
│                      LLM Agent                           │
│  (Proma Agent / Claude Code / 任意 MCP 客户端)            │
└──────┬──────────────┬──────────────────┬────────────────┘
       │ curl         │ MCP (JSON-RPC)   │ MCP (JSON-RPC)
       ▼              ▼                  ▼
┌──────────┐  ┌──────────────┐  ┌──────────────────┐
│CDP Proxy │  │Zotero MCP    │  │web-access skill   │
│ :3456    │  │ :23120       │  │(CDP Proxy 管理)   │
│          │  │ (Zotero内)   │  │                   │
│ WS↔Chrome│  │              │  │                   │
└────┬─────┘  └──────┬───────┘  └──────────────────┘
     │               │
     ▼               ▼
┌──────────┐  ┌──────────────┐
│ Chrome   │  │   Zotero 7   │
│ + 知网    │  │              │
│ + 登录态  │  │ + SQLite DB  │
│ + PDF下载 │  │ + Storage/   │
└──────────┘  └──────────────┘
```

### 组件清单

| 组件 | 作用 | 安装方式 |
|------|------|----------|
| **Zotero 7+** | 文献库平台 | zotero.org 下载 |
| **Zotero MCP Plugin v1.4.8+** | LLM → Zotero 桥梁（含 importAttachment） | 我们的修改版 |
| **Node.js 22+** | CDP Proxy 运行时 | nvm install 22 |
| **Chrome** | 浏览器自动化载体 | 系统自带浏览器 |
| **CDP Proxy** | curl ↔ Chrome WebSocket 转发 | web-access skill 自带 |
| **CNKI 登录态** | PDF 下载权限 | 用户手动登录一次 |

## 已验证的工作流

### 完整流程（单篇论文，约60秒）

```
Step 1: 搜索论文
  CDP Proxy /new → kns.cnki.net/kns8s/search
  CDP Proxy /eval → 填入关键词，点击搜索
  等待页面出现"条结果"
  提取结果列表（标题、URL、作者、期刊、日期）

Step 2: 获取元数据
  CDP Proxy /navigate → 论文详情页
  提取 #export-id
  调用 CNKI Export API:
    POST https://kns.cnki.net/dm8/API/GetExport
    body: { filename, displaymode: "GBTREFER,elearning,EndNote", uniplatform: "NZKPT" }
  解析 ELEARNING 字段 → 标题、作者、期刊、年份、摘要、关键词、ISSN、DOI

Step 3: 创建 Zotero 条目
  zotero-mcp write_item(action: create, itemType: journalArticle, ...)
  → 返回 itemKey

Step 4: 下载 PDF
  从详情页 .btn-dlpdf a 获取 PDF 下载链接
  用 window.location.href = pdfLink 跳转（保留 Referer）
  等待 15 秒
  find ~/Downloads -name "*.pdf" -mmin -2 找到文件
  find -exec cp {} /tmp/paper.pdf 处理中文文件名

Step 5: 附加 PDF
  zotero-mcp write_item(action: importAttachment, parentKey, filePath)
  → PDF 复制到 ~/Zotero/storage/{attachmentKey}/
  → SQLite 创建附件记录
  → Zotero UI 实时显示

Step 6: 清理
  CDP Proxy /close → 关闭知网标签页
  rm /tmp/paper.pdf
```

### 关键实现细节

#### PDF 下载：location.href 是唯一可靠方式

CNKI 的 PDF 下载服务 `bar.cnki.net/bar/download/order` 要求请求必须带有正确的 Referer 头（来自详情页）。

| 方式 | 结果 |
|------|------|
| `click()` | ❌ 有时触发，有时不触发（不可靠） |
| `clickAt()` (真实鼠标事件) | ❌ 同上 |
| CDP `/new` 开新标签 | ❌ "来源应用不正确(01)" — 无 Referer |
| `window.location.href = pdfLink` | ✅ 保留 Referer，100% 可靠 |

#### 中文文件名处理

CNKI 下载的 PDF 文件名常包含 Unicode 智能引号（如 `""`），这些在终端/curl 中会丢失或乱码。

```bash
# ❌ 不行 — 智能引号在 bash 中丢失
cp ~/Downloads/实施"一带一路"..._唐鹏琪.pdf /tmp/paper.pdf

# ✅ 可行 — find 保留原始字节
find ~/Downloads -name "*一带一路*" -exec cp {} /tmp/paper.pdf \;
```

#### Zotero MCP Plugin 扩展

原版 `write_item` 只支持两个 action：`create` 和 `reparent`。
我们新增了 `importAttachment`，调用 Zotero 原生的 `Attachments.importFromFile()` API。

```typescript
// src/modules/streamableMCPServer.ts (约 60 行新增)
case 'importAttachment': {
  const file = Zotero.File.pathToFile(filePath);
  const attachment = await Zotero.Attachments.importFromFile({
    file,
    parentItemID: parentItem.id,
    contentType  // 根据扩展名自动检测
  });
  return { attachmentKey: attachment.key, path: attachment.getFilePath() };
}
```

#### Zotero Connector 为何不可靠

Zotero 浏览器扩展的捕获机制依赖以下条件：
1. 页面 DOM 结构匹配 CNKI translator 脚本的解析规则
2. 扩展的 `webRequest` API 拦截到 PDF 下载
3. 没有内部缓存/状态导致的跳过

在实际测试中，同一个会话内第一次导航到详情页时 Connector 可能触发，后续就不触发。触发条件不透明，因此 **不作为主要方案**。

## 给新用户的分发方案

### 必要文件

| 文件 | 位置 | 说明 |
|------|------|------|
| `zotero-mcp-plugin-v1.4.8.xpi` | Zotero 扩展 | 修改版 MCP 插件 |
| `zotero-mcp-importAttachment.patch` | 任意 | Git patch，方便自行构建 |
| `cnki/SKILL.md` | ~/.proma/.../skills/cnki/ | CNKI skill |
| `web-access/skills/` | ~/.claude/skills/web-access/ | CDP Proxy |

### 安装步骤

1. 安装 Zotero 7+
2. 在 Zotero 中安装 `zotero-mcp-plugin-v1.4.8.xpi`（工具 → 附加组件 → 从文件安装）
3. 重启 Zotero
4. 确认 Zotero MCP 运行在 `localhost:23120`
5. Node.js 22+ + Chrome 远程调试授权
6. 部署 CNKI skill 到 Proma skills 目录
7. 确认 CDP Proxy 可用
8. 在 Chrome 中登录知网

### 开发者自行构建

```bash
git clone https://github.com/cookjohn/zotero-mcp.git
cd zotero-mcp
git apply zotero-mcp-importAttachment.patch
cd zotero-mcp-plugin
npm install
npx zotero-plugin build
# 输出: .scaffold/build/zotero-mcp-plugin.xpi
```

## 设计决策记录

### 为什么不用 Headless Chrome？

用户的 Chrome 天然携带登录态。Headless 模式下知网登录态难以维持，且验证码处理更复杂。CDP Proxy 方案直接操作用户日常浏览器，登录态和扩展天然可用。

### 为什么不用 Zotero Web API？

Zotero Web API 需要 API Key + 在线同步，且文件上传走 Zotero 云存储（有限额）。本地 API（Connector/MCP Plugin）直接写 SQLite + 本地文件系统，零延迟无限制。

### 为什么不用 Python 脚本？

原 SKILL.md 的 `push_to_zotero.py` 方案：导出 API 数据 → 写 JSON 文件 → Python 解析 → POST Connector API。问题是 **无 PDF 支持**，且多一个依赖（Python 环境）。MCP Plugin 扩展后，两步搞定：`create` → `importAttachment`。

## 未来方向

1. **批量导入优化**：目前逐篇下载等待 15 秒。可考虑打开多个详情页标签页并行下载。
2. **自动重命名**：利用 Zotero 的 `getFileBaseNameFromItem()` 给附件规范命名（如"作者 - 年份 - 标题.pdf"）
3. **DOI 自动补全**：对非知网来源的 PDF，通过 DOI → Crossref API 补全元数据
4. **合并到上游**：向 `cookjohn/zotero-mcp` 提交 PR，让 `importAttachment` 进入主线
