# 知网论文一键导入 Zotero —— 完整安装教程

## 这套东西能做什么

**效果：** 你对 AI 助手说"去知网搜索这篇论文，导入到我的 Zotero 里"，30 秒后论文的元数据和 PDF 就整整齐齐出现在 Zotero 里了。

**原理：** 一个 AI 助手 → 操控浏览器打开知网 → 搜索论文 → 下载 PDF → 写入 Zotero。全程自动。

---

## 安装清单（共 4 样东西）

| 序号 | 要装的 | 做什么用 | 难度 |
|------|--------|----------|------|
| 1 | Zotero 7 | 文献管理软件 | 简单 |
| 2 | Zotero MCP 插件 | 让 AI 能操作 Zotero | 简单 |
| 3 | Chrome 远程调试 | 让 AI 能操控浏览器 | 简单（点一下） |
| 4 | 知网登录 | 下载 PDF 的权限 | 已有的话跳过 |

如果某一步不会操作，**直接把这段话复制给 AI 助手，让它帮你**：

> "帮我安装以下内容：1) Zotero 7 文献管理软件；2) Zotero MCP 插件，下载地址 https://github.com/2246024816/zotero-mcp/releases/download/v1.4.8/zotero-mcp-plugin-v1.4.8.xpi；3) 开启 Chrome 远程调试。我没有技术背景，请用最简单的语言指导我每一步。"

---

## 第一步：安装 Zotero 7

1. 打开 https://www.zotero.org/download/
2. 下载 Mac 版（或 Windows 版）
3. 安装后打开一次，确认能正常运行
4. 保持 Zotero 在后台运行（不用管它，开着就行）

> **验证：** 菜单栏看到 Zotero 图标就是装好了。

---

## 第二步：安装 Zotero MCP 插件

1. 下载这个文件：  
   https://github.com/2246024816/zotero-mcp/releases/download/v1.4.8/zotero-mcp-plugin-v1.4.8.xpi
2. 打开 Zotero → 顶部菜单栏点"工具" → "附加组件"
3. 点右上角的齿轮图标 ⚙ → "Install Add-on From File..."
4. 选择刚才下载的 `.xpi` 文件
5. 看到"Zotero MCP Plugin 1.4.8"出现在列表中 → 成功
6. **重启 Zotero**

> **验证：** 重启后，浏览器打开 http://localhost:23120/mcp 看到 JSON 数据就是装好了。（不要让代理上）

---

## 第三步：开启 Chrome 远程调试

1. 打开 Chrome 浏览器（你日常用的那个）
2. 在地址栏输入：`chrome://inspect/#remote-debugging`
3. 勾选 ☑️ **"Allow remote debugging for this browser instance"**
4. 看到弹出对话框，点"允许"或"确认"
5. 关闭这个标签页，正常使用 Chrome 即可

> **注意：** Chrome 每次重启后这个设置会自动关闭，AI 助手操作前需要重新勾选。

> **验证：** 不需要验证。以后 AI 助手连不上时会提醒你。

---

## 第四步：确认知网登录状态

1. 在 Chrome 中打开 https://kns.cnki.net
2. 看右上角，有没有显示你的用户名或机构（如"四川大学图书馆"）
3. 如果没有 → 点"登录"→ 选"机构登录"→ 通过学校账号登录
4. 确保能搜索论文并看到 PDF 下载按钮

> **大部分高校用户已经通过学校网络自动登录了，这一步通常不需要操作。**

---

## 完成！试试你的第一个自动化

装完之后，**把下面这句话发给你的 AI 助手**：

> "帮我去知网搜索'人工智能对国际关系的影响'这篇论文，下载并导入到我的 Zotero 中。"

看看效果。

---

## 常见问题

### Q: AI 说"Chrome 远程调试未开启"
**A:** 重新做第三步。Chrome 重启后这个设置会消失。

### Q: AI 说"验证码"  
**A:** 看你的 Chrome 浏览器，知网弹了一个滑块验证，手动拖一下。然后告诉 AI "已解决，继续"。

### Q: AI 说"Zotero MCP 连接失败"
**A:** 确认 Zotero 在运行（看菜单栏有没有 Zotero 图标）。如果刚装了插件，试试重启 Zotero。

### Q: 下载的 PDF 文件名是乱码
**A:** 不影响使用。文件内容是正常的，只是知网给的中文文件名偶尔有奇怪的引号。

### Q: 我用的不是 Proma，是其他 AI 工具
**A:** 只要是支持 MCP 协议的 AI 工具（Claude Desktop、Cline 等），都可以用这个插件。把 Zotero MCP 的 endpoint `http://localhost:23120/mcp` 配置进工具的 MCP 设置就行。不会配的话让 AI 帮你配。

### Q: 我想把论文批量导入，一次几十篇
**A:** 可以。对 AI 说："帮我把搜索结果的第 1-20 篇都导入到 Zotero"。AI 会逐篇处理。

---

## 技术架构（给想看原理的人）

```
你的 Chrome  ←→  CDP Proxy  ←→  AI 助手  ←→  Zotero MCP 插件  ←→  Zotero
    ↑                                                              ↑
    └── 知网搜索、下载 PDF ──────────────→  元数据 + PDF ─────────→ 导入条目
```

每个组件都是一个小工具，各司其职。你不需要理解它们内部怎么工作，装好就能用。
