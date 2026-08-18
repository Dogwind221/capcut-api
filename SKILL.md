---
name: capcut-api
description: CapCut/剪映草稿生成 API（sun-guannan/CapCutAPI）的安装、启动、调用与排障。当用户提到 CapCut、剪映、草稿、dfd_、capcut_server、mcp_server、CapCutAPI、生成剪映草稿、视频草稿 API、剪映自动出片时触发。
---

# CapCut API（剪映草稿生成）

本地 HTTP/MCP 服务，通过 API 生成**剪映/CapCut 草稿**（`dfd_xxxx` 文件夹），
供 AI Agent 程序化创建视频项目。**非字节官方**。

## ⚠️ 核心限制（务必先告知用户）

- 只生成草稿，**不能自动渲染导出**——最后必须手动打开剪映点导出
- 草稿不会自动进剪映目录——必须手动复制 `dfd_xxxx` 到草稿目录
- 剪映更新大版本可能导致旧草稿 json 不兼容打不开——**务必备份草稿目录**
- 本机环境：Python 3.12.10 / FFmpeg 9.0 / Git 2.55 / 剪映专业版（均已在 PATH）

## 触发（DSH 场景）

- 用户提到 **CapCut / 剪映 / 草稿 / dfd_ / 剪映草稿 / 视频草稿 / 剪映自动出片 / CapCutAPI**
- 用户给出剪映草稿路径或 `dfd_xxx` 文件夹，要求分析/修改
- 用户要求「用 API 生成剪映草稿」「批量建草稿」
- 拖入剪映草稿文件（zip/文件夹）→ file-intake 可路由到本 skill 分析草稿结构

## DSH 使用（服务管理）

服务**默认常驻**（手动启动后保持运行）；新会话使用前先探测：

```powershell
# 服务是否在跑（端口 9001）
$conn = Get-NetTCPConnection -LocalPort 9001 -State Listen -ErrorAction SilentlyContinue
if ($conn) { "运行中 pid=$($conn.OwningProcess)" } else { "未运行" }

# 未运行则启动（后台、隐藏窗口）
Start-Process 'F:\dsh\capcut-api\CapCutAPI\venv-capcut\Scripts\python.exe' `
  -ArgumentList 'capcut_server.py' -WorkingDirectory 'F:\dsh\capcut-api\CapCutAPI' `
  -WindowStyle Hidden -RedirectStandardOutput 'F:\dsh\capcut-api\server.log' `
  -RedirectStandardError 'F:\dsh\capcut-api\server.err.log'
```

## DSH 会话内标准工作流

1. **探测服务**（见上）；未运行 → 启动，等 3-5 秒
2. **创建草稿**：`POST /create_draft`（width/height/fps；草稿暂存内存，返回 `draft_id`）
3. **落盘**：`POST /save_draft`（`draft_id` → 生成 `dfd_xxx` 文件夹，位于 `F:\dsh\capcut-api\CapCutAPI\`）
4. **复制进剪映**：`Copy-Item dfd_xxx → C:\Users\ASUS\Videos\JianyingPro\User Data\Projects\com.lveditor.draft\`
5. **打开剪映确认**：`Start-Process 'D:\剪映\JianyingPro\JianyingPro.exe'` → 草稿列表应出现 → **提醒用户手动导出**

## 安装（一次性）

```bash
git clone https://github.com/sun-guannan/CapCutAPI.git
cd CapCutAPI
python -m venv venv-capcut
venv-capcut\Scripts\activate        # 每次新终端都要激活
pip install -r requirements.txt      # HTTP API 基础依赖
# 如需 MCP（给 AI Agent 调用），额外：
pip install -r requirements-mcp.txt
copy config.json.example config.json
```

`config.json` 关键项：
- 国内剪映：`"is_capcut_env": false`（默认）
- CapCut 海外版：`"is_capcut_env": true`
- 默认端口 `9001`

## 启动

| 方式 | 命令 | 用途 |
|---|---|---|
| HTTP API | `python capcut_server.py` | 普通 REST 调用，访问 `http://127.0.0.1:9001` |
| MCP | `python mcp_server.py` | 给 AI Agent / MCP 客户端 |

## 常用 API

**创建草稿**（简单测试）：
```bash
curl -X POST http://127.0.0.1:9001/create_draft ^
  -H "Content-Type: application/json" ^
  -d "{\"name\":\"测试项目\",\"width\":1080,\"height\":1920,\"fps\":30}"
```

**工作流**：
1. 调 API 创建草稿 → 本地生成 `dfd_xxxx` 文件夹
2. 复制 `dfd_xxxx` → 剪映草稿目录（见下）
3. 打开剪映 → 草稿列表出现项目 → 手动导出

## 草稿目录

- Windows 剪映：`C:\Users\<用户名>\Videos\JianyingPro\User Data\Projects\com.lveditor.draft\`
- Windows CapCut 国际版：JianyingPro 换成 `CapCut`
- macOS：`/Users/<用户名>/Movies/JianyingPro/User Data/Projects/com.lveditor.draft/`

## 常见踩坑

- 虚拟环境忘激活 → 依赖报错：`venv-capcut\Scripts\activate`
- Python 3.13 会报错（建议 3.10-3.11；本机 3.12.10 可用）
- FFmpeg 不在 PATH → 素材功能失败
- 草稿不会自动同步进剪映目录 → 必须手动复制
- 剪映大版本更新 → 旧草稿 json 不兼容 → 提前备份
- 代码拉取走代理：`$env:HTTPS_PROXY='http://127.0.0.1:7897'`
- DSH 会话内服务未运行会直接报连接错误 → 先按「DSH 使用」启动再调 API

## 本机部署位置（2026-08-17 已实测部署）

- 项目目录：`F:\dsh\capcut-api\CapCutAPI`
- 虚拟环境：`F:\dsh\capcut-api\CapCutAPI\venv-capcut`（Python 3.12.10，依赖已装）
- 服务端口：9001（启动：`venv-capcut\Scripts\python.exe capcut_server.py`）
- config.json：`draft_profile: "jianying_pro_10"`、`is_capcut_env: false`（本机剪映专业版 10.x，装在 `D:\剪映\JianyingPro`）
- 剪映草稿目录：`C:\Users\ASUS\Videos\JianyingPro\User Data\Projects\com.lveditor.draft`

### 实测工作流（已验证）

```powershell
# 1. 创建草稿（返回 draft_id，草稿暂存内存缓存）
curl -X POST http://127.0.0.1:9001/create_draft -H "Content-Type: application/json" -d '{"width":1080,"height":1920,"fps":30}'
# → {"draft_id":"dfd_cat_xxx",...}

# 2. 落盘生成 dfd_xxx 文件夹（默认保存到 capcut_server.py 所在目录）
curl -X POST http://127.0.0.1:9001/save_draft -H "Content-Type: application/json" -d '{"draft_id":"dfd_cat_xxx"}'

# 3. 复制草稿到剪映草稿目录
Copy-Item "F:\dsh\capcut-api\CapCutAPI\dfd_cat_xxx" "C:\Users\ASUS\Videos\JianyingPro\User Data\Projects\com.lveditor.draft" -Recurse

# 4. 打开剪映 → 草稿列表出现 → 手动导出
Start-Process 'D:\剪映\JianyingPro\JianyingPro.exe'
```

### 费用

- **本地部署费用：¥0**（开源项目 + Python/FFmpeg/Git 免费 + 剪映个人版免费）
- 可选：剪映会员（高级特效/导出，约 ¥25/月）；云服务器 24h 在线（约 ¥30-100/月）
- `draft_domain`（capcutapi.top）仅用于远程预览下载器，本地 `save_draft` 不依赖，`is_upload_draft: false` 不上传
