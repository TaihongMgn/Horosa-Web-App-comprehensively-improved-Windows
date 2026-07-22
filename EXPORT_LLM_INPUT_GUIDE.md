# Horosa「导出完整 LLM 输入」功能 — 构建与部署指南

> 本文档面向自动化 Agent 或开发者，描述如何从零构建并部署该功能到已安装的 Horosa 桌面版。

## 功能说明

在 AI 分析页面的输入框工具栏新增一个导出按钮（export 图标），点击后将**即将发送给大模型的全部内容**导出为 `.txt` 文件，包含：

- 元信息（Provider 类型、Base URL、模型名、思考档位、providerOptions）
- 完整的 Messages 数组（system prompt + 对话历史 + 当前用户输入）
- 上下文层明细（buildContextLayers 裁剪后保留的每一层的 key/title/字数）

## 代码改动

仅修改一个文件：

```
astrostudyui/src/components/aianalysis/AIAnalysisMain.js
```

改动内容（+88 行）：

1. **新增 `exportFullLLMInput()` 异步函数**（在 `handleStopStreaming` 之后）
   - 调用 `buildResolvedPrompt()` 走与 `handleSend` 完全相同的 prompt 组装链路
   - 构建与真实发送一致的 `chatMessages` 数组
   - 还原 `providerOptions`（temperature / top_p / thinking level）
   - 格式化为可读文本并通过 `saveBlobToBrowser()` 下载

2. **输入框工具栏新增按钮**（在"添加图片"按钮之后、"停止生成"按钮之前）
   - 图标：`<XQIcon name="export" />`
   - Tooltip：「导出完整 LLM 输入（系统提示+挂载上下文+对话+参数）」
   - 生成中禁用（`disabled={sending}`）

## 构建步骤

### 前置条件

- Node.js >= 16（推荐 18+）
- npm >= 8
- Git

### 构建前端

```bash
cd <repo>/local/workspace/Horosa-Web-55c75c5b088252fbd718afeffa6d5bcb59254a0c/astrostudyui
npm install
npm run build:file
```

产物在 `astrostudyui/dist-file/`（约 388 个文件）。

> 注意：`npm run build:file` 使用 `scripts/umi-runner.js`（Windows 适配版），自动设置 `NODE_OPTIONS=--openssl-legacy-provider`。

## 部署步骤（热替换到已安装的 Horosa 桌面版）

### 关键路径

Horosa 桌面版（Electron）的前端**实际加载路径**不是安装目录，而是：

```
%LOCALAPPDATA%\HorosaDesktop\embedded-runtime\{payloadId}\rt\bundle\dist-file\
```

其中 `{payloadId}` 取自安装目录下的 manifest：

```
D:\Program Files\Horosa\resources\rt\payload-manifest.json → "payloadId" 字段
```

### 部署脚本（PowerShell，无需管理员权限）

```powershell
# 1. 读取 payloadId
$manifest = Get-Content 'D:\Program Files\Horosa\resources\rt\payload-manifest.json' | ConvertFrom-Json
$payloadId = $manifest.payloadId

# 2. 确定目标路径
$dst = "$env:LOCALAPPDATA\HorosaDesktop\embedded-runtime\$payloadId\rt\bundle\dist-file"
$src = '<repo>\local\workspace\Horosa-Web-55c75c5b088252fbd718afeffa6d5bcb59254a0c\astrostudyui\dist-file'

# 3. 关闭 Horosa（文件占用）
Get-Process -Name 'Horosa' -ErrorAction SilentlyContinue | Stop-Process -Force
Get-Process -Name 'java' -ErrorAction SilentlyContinue | Where-Object { $_.Path -like '*HorosaDesktop*' } | Stop-Process -Force
Get-Process -Name 'python' -ErrorAction SilentlyContinue | Where-Object { $_.Path -like '*HorosaDesktop*' } | Stop-Process -Force
Start-Sleep -Seconds 3

# 4. 替换
Remove-Item -Recurse -Force $dst
Copy-Item -Recurse -Force $src $dst

# 5. 清 Chromium 缓存（防旧 index.html 缓存）
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\HorosaDesktop\Cache" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\HorosaDesktop\Code Cache" -ErrorAction SilentlyContinue

Write-Host "Done. Restart Horosa to use the new feature."
```

### 常见陷阱

| 问题 | 原因 | 解决 |
|------|------|------|
| 替换后按钮不出现 | 替换了 `D:\Program Files\Horosa\resources\rt\rt\bundle\dist-file`（错误位置） | 必须替换 `embedded-runtime\{payloadId}\rt\bundle\dist-file` |
| Permission denied | Horosa 进程未关闭，文件被占用 | 先杀 Horosa + java + python 进程 |
| 替换后仍显示旧版 | Chromium 磁盘缓存了旧 index.html | 删除 `%LOCALAPPDATA%\HorosaDesktop\Cache` 和 `Code Cache` |
| 软件自动更新后功能消失 | 更新器重新下载 payload 覆盖了 dist-file | 重新执行部署脚本 |
| `npm run build:file` 失败 | Windows 下缺少 `umi-runner.js`（Mac 的 build 脚本用 bash export） | 本仓库已包含 Windows 适配版 `scripts/umi-runner.js` |
| git clone 报 Filename too long | Java jnilib.dSYM 路径超限 | `git config core.longpaths true` 后重新 checkout |
| grep 报 "Binary file matches" | 仓库 .js 文件含特殊字符被误判 | 加 `-a` flag：`grep -a` |

## 回滚

原版前端备份位置（如果之前备份过）：直接重新运行 Horosa 的「检查更新」即可恢复官方版本。

## 技术架构备注

- Prompt 组装**全部在前端**完成（`AIAnalysisMain.js` 的 `buildResolvedPrompt()`）
- 后端 `AIAnalysisProxyService.java` 仅是透明代理，不参与 prompt 构建
- 上下文层顺序：system(100) → source(95) → dayBoundaryRule(94) → technique:*(93-idx) → bundle-system(92) → template(90) → retrieved-context(80) → material(70) → recent-history(60)
- 裁剪预算由 `contextCharBudgetForModel()` 按模型窗口动态计算（默认 20000 字符，大窗口模型最高 60000）
- 稳定层与挥发层交界插入 `[[__CACHE_BP__]]` 标记供 provider 前缀缓存
