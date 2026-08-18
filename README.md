# capcut-api

CapCut / 剪映草稿生成 API 使用 Skill（基于 [sun-guannan/CapCutAPI](https://github.com/sun-guannan/CapCutAPI)）。

通过本地 HTTP/MCP 服务，用 API 程序化生成**剪映/CapCut 草稿**（`dfd_xxxx` 文件夹），
供 AI Agent 创建视频项目。**非字节官方**。

## ⚠️ 核心限制

- 只生成草稿，**不能自动渲染导出**——最后必须手动打开剪映点导出
- 草稿不会自动进剪映目录——必须手动复制 `dfd_xxxx` 到草稿目录
- 剪映更新大版本可能导致旧草稿 json 不兼容打不开——**务必备份草稿目录**

## 快速开始

```bash
git clone https://github.com/sun-guannan/CapCutAPI.git
cd CapCutAPI
python -m venv venv-capcut
venv-capcut\Scripts\activate
pip install -r requirements.txt
copy config.json.example config.json   # 国内剪映 is_capcut_env: false
python capcut_server.py                # HTTP 服务 http://127.0.0.1:9001
```

### 工作流

```bash
# 1. 创建草稿（返回 draft_id）
curl -X POST http://127.0.0.1:9001/create_draft -H "Content-Type: application/json" -d '{"width":1080,"height":1920,"fps":30}'

# 2. 落盘生成 dfd_xxx 文件夹
curl -X POST http://127.0.0.1:9001/save_draft -H "Content-Type: application/json" -d '{"draft_id":"dfd_cat_xxx"}'

# 3. 复制草稿到剪映草稿目录
Copy-Item "dfd_cat_xxx" "C:\Users\<用户名>\Videos\JianyingPro\User Data\Projects\com.lveditor.draft" -Recurse

# 4. 打开剪映 → 草稿列表出现 → 手动导出
```

## 草稿目录

- Windows 剪映：`C:\Users\<用户名>\Videos\JianyingPro\User Data\Projects\com.lveditor.draft\`
- Windows CapCut 国际版：JianyingPro 换成 `CapCut`
- macOS：`/Users/<用户名>/Movies/JianyingPro/User Data/Projects/com.lveditor.draft/`

## 费用

- 本地部署 **¥0**（开源项目 + Python/FFmpeg/Git 免费 + 剪映个人版免费）
- 可选：剪映会员（高级特效/导出，约 ¥25/月）；云服务器 24h 在线（约 ¥30-100/月）

完整说明见 [SKILL.md](./SKILL.md)。

## 核心优点

- **本地部署 ¥0**：开源项目 + Python/FFmpeg/Git 免费 + 剪映个人版免费，无服务器、无订阅
- **程序化生成标准剪映草稿**：API 一键产出完整 `dfd_xxx` 草稿结构（draft_content.json / Timelines / Resources 等 20+ 文件），剪映打开即见
- **完整工作流实测**：create_draft → save_draft → 复制草稿目录 → 剪映识别，全链路验证通过
- **双模式**：HTTP API（REST）与 MCP（AI Agent 直调）两种接入方式
- **双环境适配**：国内剪映（`is_capcut_env: false` / jianying_pro_10）与海外 CapCut 均可
- **配套排障手册**：SKILL.md 收录常见踩坑（虚拟环境/FFmpeg PATH/版本兼容/草稿备份）与本机部署记录
