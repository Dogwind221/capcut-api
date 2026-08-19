---
name: capcut-api
description: CapCut/剪映草稿生成 API（sun-guannan/CapCutAPI）的安装、启动、调用与排障；内置剪辑思维库与「需求→API 决策链」，能把用户的剪辑需求（卡点/转场/音乐/字幕/调色）准确转化为 capcut-api 调用序列。当用户提到 CapCut、剪映、草稿、dfd_、capcut_server、mcp_server、CapCutAPI、生成剪映草稿、视频草稿 API、剪映自动出片、剪辑思维、卡点剪辑、转场、字幕、BGM时触发。
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
- 用户描述剪辑需求（卡点/转场/字幕/音乐）要求自动出片
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

---

# 🧠 剪辑思维库

> 教程用 PR 演示，但**思维与原则软件无关**，以下均已映射到 capcut-api 的能力。
> 用途：把用户的模糊需求（"做一个燃的卡点视频"）转成准确的 API 调用序列。

## 1. 总纲：剪辑本质 = 讲故事（先定轴，再动手）

- 讲故事 = **讲清楚**（情绪 51% + 事件 23%）+ **讲有趣**（节奏 10%）
  （源自沃尔特·莫奇《眨眼之间》六大剪辑法则：情感 > 故事 > 节奏）
- **接任何需求先做三问**：
  1. **媒介**？（横屏 1920×1080 / 竖屏 1080×1920 → create_draft 的 width/height）
  2. **主题+情绪**？（热血燃 / 温情 / 悬疑 / 复古… → 决定音乐、画面、转场风格）
  3. **节奏**？（快节奏卡点 / 舒缓叙事 → 决定切点密度、转场用量）
- 创意剪辑核心原则：**反常规**——先了解常规做法，再反着做（例：高潮前不渐起而是突然停→留白）。

## 2. 需求 → API 决策链（核心转化思维）

用户说「我要……」，按下面 9 步决策（**先内容 → 再节奏 → 再情绪 → 再包装 → 最后动效**）：

| 步 | 决策问题 | API 端点 | 关键参数 |
|---|---|---|---|
| 1 画幅 | 横屏/竖屏/帧率 | `create_draft` | width=1920/1080, height=1080/1920, fps=25/30 |
| 2 内容 | 主素材有哪些？顺序？ | `add_video` / `add_image` | video_url/image_url, start, end, track_name |
| 3 节奏 | 切点在哪？（卡点/故事段） | `add_video` 多次 | target_start 精确对齐切点；speed 变速 |
| 4 段落衔接 | 段间要转场吗？什么档位？ | `add_video` transition | transition（枚举名）, transition_duration(默认0.5s) |
| 5 音乐 | 情绪曲？卡点曲？ | `add_audio` | audio_url, volume, target_start；track_name 分轨 |
| 6 声音细节 | 音效/人声填补留白？ | `add_audio`（多轨） | 音效轨 sfx / 人声轨 vocal / 音乐轨 music |
| 7 字幕 | 口播/歌词？ | `add_subtitle` | srt 内容或URL, font_size, transform_y 默认-0.8（底部安全区） |
| 8 花字标题 | 标题/强调文字？ | `add_text` | text, start, end, font, intro/outro_animation |
| 9 动效包装 | 入场/强调动画？ | `add_video_keyframe` | 批量 property_types/times/values |

**决策原则（来自教程）**：
- 转场使用准则：**换节奏/换音乐/换场景/另起段落时才用**；高级≠炫技，=合理+放在合适位置
- 卡点要**松弛有度**：不是每个重拍都切，从头卡到尾没人看
- 一首音乐从头铺到尾=平；选**有变化**的音乐（波形起伏多=剪辑可能性多）
- 节奏要握在剪辑手里，不被素材和音乐牵着走（可加音效/人声重建节奏）

## 3. 音乐思维（EP4-8 蒸馏）

- **找音乐**：渠道（罐头音乐=成品可购，注意版权；剪映 AI 生成音乐免版权）。
  「四问题检索法」：①作品类型（悬疑/纪实…）②音乐类型 ③最频繁乐器 ④感受 →
  用其中 1-3 个关键词组合检索。
- **用音乐**：
  - 开场 3 式：①指数淡化渐起（音量由弱到强）②先音效+人声再起音乐 ③配合画面动势直接起
  - 高潮「打破预期」：音乐将到顶时突然停顿（滞空/留白）→ 后续高潮更猛烈（让子弹飞一会儿）
  - 衔接过渡：滞空处加音效+人声台词填补；两曲叠加=前曲音量渐弱+后曲渐强（交叉淡化）；
    结尾=渐弱退场 或 删掉重复小节直接接原曲结尾
- **API 映射**：
  - 音量淡入淡出：`add_video_keyframe` 的 `volume` 属性做在音频轨上
    （volume=0 → 1.0 渐入；1.0 → 0 渐出；K 帧时间按音乐波形对齐）
  - 音效/人声：额外 `add_audio`，`target_start` 对齐滞空区间，track_name 分开
  - 变速：`add_video` 的 `speed` 参数

## 4. 卡点思维（EP13 蒸馏）

- 卡点本质 = **变化**（画面/动作/灯光/镜头运动都是节奏点）；剪辑工作=重构节奏
- 找鼓点：**看音频波形起伏的起始位置**（放大音轨）
- 节奏点类型清单：
  - 声音：音乐重拍、编曲细节（合成器/弦乐/钢琴键音）、音效、人声台词
  - 画面：人物动作（眨眼/抬手/跺脚）、物体运动（抛物/开车）、灯光变化、
    镜头运动（推拉摇移/甩镜/变速）、后期变化（位置/分屏/文字快闪/特效转场）
- **API 映射**（API 无节拍检测端点，需外部计算）：
  ```python
  # 用 librosa 检测重拍（faster-whisper 环境需 pip install librosa；或用剪映 AI 标记后人工导出）
  import librosa
  y, sr = librosa.load('bgm.mp3')
  tempo, beats = librosa.beat.beat_track(y=y, sr=sr)
  beat_times = librosa.frames_to_time(beats, sr=sr).tolist()  # 切点秒数列表
  ```
  然后把 `beat_times` 转成 `add_video` 序列：第 i 段 `target_start=beat_times[i]`、
  `start=beat_times[i]`、`end=beat_times[i+1]`。**按结构留呼吸**（跳 1-2 个拍不切）。

## 5. 转场思维（EP14 + EP19-27 蒸馏）

### 转场五等级（按用户需求档次选择）
| 档位 | 手法 | API 实现 |
|---|---|---|
| 青铜 | 系统默认：闪白/闪黑(1-2帧)/交叉溶解 | `add_video` transition="交叉溶解" 类；闪白闪黑用 add_effect 白光/黑场 |
| 白银 | 效果+关键帧：位移/缩放/模糊/分屏 | `add_video_keyframe` scale/position + `add_effect` 模糊 |
| 黄金 | 遮罩转场/眨眼转场 | mask_type+mask_*（近似）；眨眼=黑场+椭圆遮罩+羽化扩展关键帧 |
| 星耀 | 动态拼贴拉近（镜像补黑边） | scale 缩小 + position 位移关键帧（API 无镜像，用放大+位移近似） |
| 王者 | 晃动模糊（位置弹跳幅度递减+运动模糊） | position_x/position_y 弹跳关键帧序列（幅度递减）+ 模糊特效 |

### 教程 9 大高级转场 → API 枚举直接对应（先 GET 枚举确认存在）
| 教程 | 效果 | `get_transition_types` 枚举 |
|---|---|---|
| EP19 旋转模糊开场 | 缩放150-200%+旋转+模糊→还原（1秒） | `旋转模糊`；关键帧 uniform_scale/rotation |
| EP20 曝光缩放转场 | 变焦溶解+镜头模糊+灰光（7帧） | `曝光拉丝` / `曝光摇镜` / `长曝光` / `快速缩放` |
| EP21 弹性旋转拉镜 | 鼓点±10帧+缩放/旋转/纵向弹跳 | `中心旋转` / `旋转模糊` + position_y 弹跳关键帧 |
| EP22 亮度转场 | 12帧重叠+亮度100→0 | brightness 关键帧 1.0→0.0（视频段属性） |
| EP23 光线摆动转场 | 光线扫过画面 | `扫光` / `光束` / `斜向闪光` / `星光叠化` / `泛光` / `白光快闪` |
| EP24 曝光抖动转场 | 抖动+模糊+缩放旋转+曝光闪 | `抖动` / `微抖动` / `抖动放大` / `收缩抖动` / `曝光拉丝` |
| EP25 3D翻转转场 | 基本3D旋转89°/-89°+底层赛博素材 | `_3D空间` / `空间翻转` / `镜像翻转` / `空间旋转` |
| EP26 内吸转场 | 镜头扭曲-89+亮度100+模糊40（20帧） | `吸入` / `星星吸入` |
| EP27 镜面破碎转场 | 定格帧+破碎效果（匹配画面动作） | `玻璃破碎` / `玻璃破碎_II` / `镜像翻转` |

**组合思维**：高级转场=多效果叠加+关键帧+缓动曲线；效果参数值需按曲线**预计算**后批量传入
（API 关键帧是线性插值，缓入缓出要在值上做补偿——中间密两端疏）。

## 6. 字幕规范（EP16-17 蒸馏）

- 唱词位置：**安全区内、画面中心偏下**（transform_y 默认 -0.8 已对齐）；不要太贴底（要有呼吸感）
- **单行**、断句干净（按句/按词语断，不从中间断字）
- **不要标点符号**（商业标准：除书名号引号）
- 电视标准约 14 字/行；短视频可放宽（看甲方）
- 字体：不用细线尖锐体（如宋体），常用黑体系（微软雅黑/阿里普惠体，注意版权）；
  白字加轻描边/阴影/背景突出
- 花字（装饰字幕）→ `add_text` 做艺术处理；唱词 → `add_subtitle`
- **API 映射**：`add_subtitle` 传 SRT；生成 SRT 用 whisper 转写口播→按句切分→去除标点→
  font="思源粗宋"(默认)或黑体系, font_size=5.0左右, transform_y=-0.8, border_width 轻描边

## 7. 调色思维（EP12 蒸馏，API 能力有限，明确边界）

- 调色分级：一级=校正（亮度/对比/饱和/白平衡）；二级=风格化（S曲线/RGB曲线）；局部=HSL 辅助
- **API 支持**：`add_video_keyframe` 的 `brightness`/`contrast`/`saturation`（-1~1，视频段）
- **API 不支持**：色温/白平衡/HSL 选区/曲线 → 用 `add_effect`（scene 滤镜类：`泛光`/`复古`/`胶片` 等 912 种）近似
- 原则：先校正再风格化；保护原素材（剪映用「自定义调节」层，API 层面对整段 K 帧）

## 8. 爆款拆解与创意（EP18 蒸馏）

- 短视频四文体：说明文（讲清楚）/议论文（观点）/记叙文（vlog：清楚+有趣）/散文诗（混剪：感觉）→
  **先归文体再决定技术投入**（说明文不需要花哨转场）
- 拉片方法论（拆任何爆款）：逐镜头剪开 → 记录每镜头时长（爆款常 <1秒）→ 听鼓点位置 →
  看画面何时切（卡哪个重音）→ 看人声/音乐进出点 → 得到**可复制的参数模板**
- Casey Neistat 技巧清单（新手友好）：音效卡点、画画/写字代替特效、空镜转场、黑场过渡
  （切场景+切音乐+新章节）、亮度转场、手写体字幕、倒放镜头、年度叠加混剪

---

# 常用 API

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

## 端点速查（全部 POST JSON，响应 `{success, output, error}`）

| 端点 | 用途 | 核心参数 |
|---|---|---|
| `/create_draft` | 建草稿（暂存内存） | width, height, fps, name |
| `/add_video` | 加视频段 | video_url, start, end, target_start, duration, speed, scale_x/y, transform_x/y, transition, transition_duration, volume, mask_type+mask_*, background_blur, track_name, relative_index |
| `/add_audio` | 加音频段 | audio_url, start, end, target_start, volume, speed, duration, effect_type, effect_params, track_name |
| `/add_image` | 加图片段 | image_url, start, end, intro/outro/combo_animation, transition, mask_*, background_blur |
| `/add_text` | 加文字 | text, start, end, font, color, size, vertical, border_*, background_*, shadow_*, intro/outro_animation, text_styles |
| `/add_subtitle` | 加字幕（SRT） | srt（内容或URL）, time_offset, font, font_size, bold, italic, underline, font_color, border_*, background_*, transform_y(默认-0.8), track_name |
| `/add_video_keyframe` | 关键帧（单/批量） | track_name, property_type+time+value 或 property_types+times+values（等长列表） |
| `/add_effect` | 加特效 | effect_type, effect_category(scene/character), start, end, params, track_name |
| `/add_sticker` | 加贴纸 | resource_id, start, end, transform_* |
| `/save_draft` | 落盘 | draft_id → dfd_xxx 文件夹 |
| `/query_draft_status` | 查状态 | draft_id |
| `/generate_draft_url` | 生成下载链接 | draft_id |

## 关键帧属性（add_video_keyframe 支持）
- `position_x` / `position_y`：位移（单位=半画布宽/高，右移/上移为正，范围-10~10）
- `rotation`：旋转（顺时针角度）
- `scale_x` / `scale_y` / `uniform_scale`：缩放（1.0=不缩放；uniform_scale 与 xy 互斥）
- `alpha`：不透明度（1.0=不透明，仅视频段）
- `saturation` / `contrast` / `brightness`：色彩（-1~1，仅视频段）
- `volume`：音量（1.0=原音量，音频段/视频段均可 → **用做淡入淡出**）

## 枚举端点（先用 GET 查真实名字，别猜）
| 端点 | 数量 | 用途 |
|---|---|---|
| `/get_transition_types` | 362 | 转场名（如 旋转模糊/玻璃破碎/吸入/曝光拉丝/抖动） |
| `/get_mask_types` | 6 | 线性/镜面/圆形/矩形/爱心/星形 |
| `/get_audio_effect_types` | 106 | 音频特效（人声变声/8bit等） |
| `/get_font_types` | 335 | 字体 |
| `/get_intro_animation_types` | 95 | 视频入场动画 |
| `/get_outro_animation_types` | 72 | 视频出场动画 |
| `/get_combo_animation_types` | 123 | 组合动画 |
| `/get_text_intro_types` | 144 | 文字入场 |
| `/get_text_outro_types` | 97 | 文字出场 |
| `/get_text_loop_anim_types` | 92 | 文字循环动画 |
| `/get_video_scene_effect_types` | 912 | 场景特效（模糊/光效/滤镜类） |
| `/get_video_character_effect_types` | 227 | 人物特效 |

---

# 端到端示例：用户说「做一个竖屏燃向卡点视频，3个镜头，配BGM和音效」

```powershell
# 0. 探测服务（见上）；未运行先启动

# 1. 画幅：竖屏短视频
curl -X POST http://127.0.0.1:9001/create_draft -H "Content-Type: application/json" -d '{"width":1080,"height":1920,"fps":30,"name":"燃向卡点"}'
# → {"success":true,"output":{"draft_id":"dfd_cat_xxx"}}  ← 记下 draft_id

# 2. 分析 BGM 重拍（外部计算，先下载音乐并转 16k wav）
#    python: import librosa; beats = librosa.frames_to_time(librosa.beat.beat_track(...)[1])
#    假设重拍在 [0.0, 1.2, 2.4, 3.6, 4.8] 秒；选 0/1.2/2.4 三个点做切点，3.6 留白（松弛有度）

# 3. 三个镜头依次上轨，切点=重拍；中间段用转场
curl -X POST http://127.0.0.1:9001/add_video -H "Content-Type: application/json" -d '{"draft_id":"dfd_cat_xxx","video_url":"C:/clips/shot1.mp4","start":0,"end":1.2,"target_start":0,"transition":"抖动","transition_duration":0.4}'
curl -X POST http://127.0.0.1:9001/add_video -H "Content-Type: application/json" -d '{"draft_id":"dfd_cat_xxx","video_url":"C:/clips/shot2.mp4","start":0,"end":1.2,"target_start":1.2,"transition":"曝光拉丝","transition_duration":0.4}'
curl -X POST http://127.0.0.1:9001/add_video -H "Content-Type: application/json" -d '{"draft_id":"dfd_cat_xxx","video_url":"C:/clips/shot3.mp4","start":0,"end":1.2,"target_start":2.4}'

# 4. BGM 轨 + 音效轨（分开，音量淡入）
curl -X POST http://127.0.0.1:9001/add_audio -H "Content-Type: application/json" -d '{"draft_id":"dfd_cat_xxx","audio_url":"C:/music/bgm.mp3","track_name":"music","volume":0.8}'
curl -X POST http://127.0.0.1:9001/add_audio -H "Content-Type: application/json" -d '{"draft_id":"dfd_cat_xxx","audio_url":"C:/sfx/whoosh.mp3","target_start":3.5,"track_name":"sfx","volume":1.0}'

# 5. 弹跳+缩放关键帧（转场处弹性感；幅度递减=缓动近似）
curl -X POST http://127.0.0.1:9001/add_video_keyframe -H "Content-Type: application/json" -d '{"draft_id":"dfd_cat_xxx","track_name":"video_main","property_types":["uniform_scale","uniform_scale","uniform_scale"],"times":[1.0,1.2,1.4],"values":["1.15","1.0","1.05"]}'

# 6. 字幕（无标点、单行、底部安全区）
curl -X POST http://127.0.0.1:9001/add_subtitle -H "Content-Type: application/json" -d '{"draft_id":"dfd_cat_xxx","srt":"1\n00:00:00,000 --> 00:00:01,200\n开 局 直 接 燃\n\n","font_size":5.0,"transform_y":-0.8}'

# 7. 落盘 → 复制到剪映 → 打开确认
curl -X POST http://127.0.0.1:9001/save_draft -H "Content-Type: application/json" -d '{"draft_id":"dfd_cat_xxx"}'
Copy-Item "F:\dsh\capcut-api\CapCutAPI\dfd_cat_xxx" "C:\Users\ASUS\Videos\JianyingPro\User Data\Projects\com.lveditor.draft" -Recurse
Start-Process 'D:\剪映\JianyingPro\JianyingPro.exe'
```

# 安装（一次性）

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

# 启动

| 方式 | 命令 | 用途 |
|---|---|---|
| HTTP API | `python capcut_server.py` | 普通 REST 调用，访问 `http://127.0.0.1:9001` |
| MCP | `python mcp_server.py` | 给 AI Agent / MCP 客户端 |

# 草稿目录

- Windows 剪映：`C:\Users\<用户名>\Videos\JianyingPro\User Data\Projects\com.lveditor.draft\`
- Windows CapCut 国际版：JianyingPro 换成 `CapCut`
- macOS：`/Users/<用户名>/Movies/JianyingPro/User Data/Projects/com.lveditor.draft/`

# 常见踩坑

- 虚拟环境忘激活 → 依赖报错：`venv-capcut\Scripts\activate`
- Python 3.13 会报错（建议 3.10-3.11；本机 3.12.10 可用）
- FFmpeg 不在 PATH → 素材功能失败
- 草稿不会自动同步进剪映目录 → 必须手动复制
- 剪映大版本更新 → 旧草稿 json 不兼容 → 提前备份
- 代码拉取走代理：`$env:HTTPS_PROXY='http://127.0.0.1:7897'`
- DSH 会话内服务未运行会直接报连接错误 → 先按「DSH 使用」启动再调 API
- 转场/特效名字必须先 GET 枚举确认，写错名字会静默跳过（`add_video_track` 对未知转场仅告警）
- **PowerShell 传中文 JSON 会编码损坏**（实测：`Invoke-RestMethod -Body (@{transition='旋转模糊'} | ConvertTo-Json)` 中文字符变 `????` 导致 API 报 Unsupported transition）→ 中文参数一律用 Python requests 调用（见下），或先用 `[System.Text.Encoding]::UTF8` 处理字节
  ```python
  # 推荐：venv python 调用（中文安全）
  import requests
  requests.post('http://127.0.0.1:9001/add_video', json={
      "draft_id": "dfd_cat_xxx", "video_url": "C:/clips/shot.mp4",
      "start": 0, "end": 2, "transition": "旋转模糊", "transition_duration": 0.5,
      "mask_type": "圆形", "mask_size": 0.8})
  ```
- 关键帧是线性插值：教程里的缓入缓出曲线要用「值中间密两端疏」的方式预计算补偿
- 节拍检测 API 无内置 → 用 librosa 或剪映 AI 标记外部计算后传入

# 本机部署位置（2026-08-17 已实测部署）

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
