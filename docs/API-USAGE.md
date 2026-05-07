# Zotero MCP Plugin — 新增工具使用说明

## 版本

v1.4.10（基于 cookjohn/zotero-mcp v1.4.7 修改）

## 新增工具

### 1. importAttachment — 导入本地 PDF 到已有条目

**用途：** 把下载好的 PDF 文件附加到 Zotero 条目中。

**调用：**
```json
{
  "action": "importAttachment",
  "parentKey": "3XZAX5E3",
  "filePath": "/tmp/paper.pdf",
  "importMode": "import"
}
```

**参数说明：**

| 参数 | 必填 | 说明 |
|------|------|------|
| `action` | ✅ | 固定为 `"importAttachment"` |
| `parentKey` | ✅ | 目标 Zotero 条目的 key（如 "3XZAX5E3"） |
| `filePath` | ✅ | 本地 PDF 文件的**绝对路径** |
| `importMode` | ❌ | `"import"` 复制到 Zotero storage（默认）；`"link"` 链接外部文件不复制 |

**返回示例：**
```json
{
  "success": true,
  "data": {
    "parentKey": "3XZAX5E3",
    "attachmentKey": "86WZA2CH",
    "path": "/Users/xxx/Zotero/storage/86WZA2CH/paper.pdf",
    "contentType": "application/pdf",
    "importMode": "import"
  }
}
```

**支持的格式（自动检测）：** PDF、DOCX、DOC、XLSX、PPTX、TXT、HTML、MD、RTF、ODT

**注意事项：**
- CNKI 下载的 PDF 文件名可能包含智能引号（`""`），建议先用 `find -exec cp` 复制到 `/tmp/` 再导入
- 需要 Zotero 在运行，且写权限已开启（默认开启）

---

### 2. delete_item — 删除条目

**用途：** 删除重复或垃圾条目。默认移到废纸篓（可恢复），可选永久删除。

**调用：**
```json
{
  "itemKeys": ["SY68CJES", "4FFPCW5G"],
  "permanent": false
}
```

**参数说明：**

| 参数 | 必填 | 说明 |
|------|------|------|
| `itemKeys` | ✅ | 要删除的条目 key 数组，支持批量 |
| `permanent` | ❌ | `true` 永久删除；`false/不传` 移到废纸篓（可在 Zotero 废纸篓恢复） |

**返回示例：**
```json
{
  "success": true,
  "data": {
    "permanent": false,
    "results": [
      {"key": "SY68CJES", "success": true, "title": "21世纪以来中国的南亚研究——邱永辉教授访谈"},
      {"key": "4FFPCW5G", "success": true, "title": "21世纪以来中国的南亚研究——邱永辉教授访谈"}
    ],
    "deletedCount": 2,
    "totalCount": 2
  }
}
```

**注意事项：**
- 先确认再删，特别是 `permanent: true` 不可逆
- 建议先列出重复条目清单，确认后再批量删除

---

## 原有工具（写操作）

| 工具 | 用途 |
|------|------|
| `write_item` (create) | 创建新条目（设置元数据、作者、标签） |
| `write_item` (reparent) | 把已有附件重新挂到其他条目下 |
| `write_metadata` | 更新已有条目的元数据字段 |
| `write_note` | 创建/修改/追加笔记 |
| `write_tag` | 添加/删除/替换标签 |

## 完整工作流示例：知网论文 → Zotero

```
1. write_item(create)     → 创建条目，获得 itemKey
2. 通过 CDP 下载 PDF       → 文件在 ~/Downloads/
3. write_item(importAttachment) → 附加 PDF 到条目
4. search_library         → 检查重复
5. delete_item            → 删除重复项
```

## 安装

1. 下载 https://github.com/2246024816/zotero-mcp/releases/latest
2. Zotero → 工具 → 附加组件 → ⚙ → Install Add-on From File
3. 选择下载的 `.xpi` 文件
4. 重启 Zotero

## 构建（开发者）

```bash
git clone https://github.com/2246024816/zotero-mcp.git
cd zotero-mcp/zotero-mcp-plugin
npm install
npx zotero-plugin build
# 输出: .scaffold/build/zotero-mcp-plugin.xpi
```
