# 从长视频到社媒图文：端到端工作流

> 适用：从一个频道的批量长视频里，筛出值得讲的几支，切成可传播的片段，配上真实视频帧做成的长图，再产出一篇可直接发布的社媒文案。
> 配套：[YouTube 素材获取手册](youtube-source-cookbook.md)（凭据、字幕、视频流与取帧）。

## 流程总览

| 阶段 | 做什么 | 关键依据 | 产出 |
|---|---|---|---|
| 0 环境 | 确认凭据与工具链 | Cookie、yt-dlp、ffmpeg、Python 3.10+ / Pillow / OpenCV | 自检通过 |
| 1 选题 | 取频道最新视频并筛选 | 官方频道 RSS，拿真实上传日期 | 3–5 篇候选 |
| 2 文字稿 | 元数据 + 官方字幕 | 见素材获取手册 §1 | 带时间戳台词文本 |
| 3 切片 | 按主题切段 + 定位金句 | 时间点全部来自官方字幕 | 切片表 + 高光点 |
| 4 配图 | 真实帧渲染长图 | 候选帧 → 睁眼评分 → `render-script` | 1–3 张竖版图 |
| 5 文案 | 社媒成稿 | 要点标注时间点 | 可复制发布的文案 |
| 6 交付 | 汇总 + 质检 + 归档 | 见下方质检清单 | 主文档 + 成图 |

## 阶段要点

### 1. 选题

用官方频道 RSS 而不是第三方节目单——后者日期常不准，会选到旧内容。

```
https://www.youtube.com/feeds/videos.xml?channel_id=<channel_id>
```

筛选规则：

- 取最近 7–10 天、时长 5–25 分钟的内容；
- 排除纯音乐/表演/片头等非演讲内容；
- 题材敏感的一律跳过，并在交付里注明跳过原因；
- **与往期查重**：每次执行前先读上一期的执行记录，避免重复推荐。

### 2. 文字稿

字幕解析成 `[mm:ss.xx] 台词` 后，用关键词定位金句时间点。滚动字幕必须去重，否则同一句会重复出现、时间轴错乱。

拿不到字幕时：切片时间轴只能是估算，**必须在交付中显式标注口径与误差**，不得伪装成精确值。

### 3. 切片

- 按主题/观点切 8–10 段，每段给：时间区间、小标题、核心要点（含字幕原句时间点）；
- 另挑 3 个最适合剪短视频的高光点：时间点 + 内容 + 适用账号类型；
- 金句表列出「时间点 + 原句 + 讲者」，全部可回官方字幕核验。

### 4. 配图

真实视频帧优先，流程：

1. 下视频（见素材获取手册 §2）；
2. 围绕每个台词点取候选帧（±0.8 秒），先程序化排除黑帧、转场、相邻帧差异过大的帧；
3. **睁眼评分**：主画面必须用评分最高的帧，`eyes ≤ 1` 判为眨眼，`score = 0` 多为远景空镜，都要换帧；
4. 渲染：

```bash
python native_subtitle_stitch.py render-script video.mp4 \
  --script script.json --out out.jpg --aspect 3:4 --width 1440
```

台词用官方字幕核对后的译写，并在交付中标注「后期绘制字幕，非画面烧录」。
无法下载视频时再退到「AI 生成底图 + 后期叠字」，同样要注明非真实帧。

### 5. 文案

标题抓眼球；正文提炼 3–5 个要点并标注时间点；结尾 3–5 个话题标签；约 300 字，可直接复制发布。

### 6. 交付质检清单

- [ ] 引语、数据、播放量均可溯源，无编造；缺失项留 `—`
- [ ] 时间轴来自官方字幕；估算项已标注口径
- [ ] 成图无黑帧、无转场、字幕完整、主画面人物睁眼
- [ ] 脚本字幕模式已明确标注为后期绘制
- [ ] 主文档包含英文原文对照，便于复核
- [ ] 执行记录已归档（便于下期查重）

## 内容质量铁律

1. 绝不编造引语、数据、播放量。
2. 时间轴优先官方字幕，估算必须显式标注。
3. 敏感题材主动跳过并注明原因。
4. 每期与往期查重。
5. 每张成图必须过睁眼与画面质检。
6. 消耗 AI 生图额度前先告知预计成本。

## 可复用提示词

自包含，换频道即可复用；路径按本机实际情况替换。

```
你是内容运营助手。任务：从指定频道筛选最新发布的 3–5 个优质长视频，产出「精选清单 + 切片方案 + 竖版配图 + 社媒文案」一整套可发布交付物。无人值守运行，遇到缺失信息自行做合理假设并在交付中注明，不要停下来问。

【固定环境，每次运行先确认】
- 平台 Cookie 文件路径（缺失则走降级路径并在交付中说明）
- 调用 yt-dlp 前必须先执行 unset NODE_OPTIONS
- yt-dlp / python / ffmpeg 使用本机绝对路径
- 工作目录：subs/ 字幕、video/ 视频、output/ 成品、scripts_json/ 台词 JSON

【阶段 1｜选题】
1. 取频道官方 RSS 拿真实上传日期
2. 取最近 7–10 天、时长 5–25 分钟的内容，优先播放量高者
3. 排除非演讲内容、敏感题材，并与往期执行记录查重
4. 选出 3–5 篇，每篇给出：标题、讲者、时长、完整链接、推荐理由、内容讲解与代表性金句

【阶段 2｜元数据与文字稿】
1. yt-dlp --cookies <cookie> --js-runtimes node --extractor-args "youtube:player_client=web_safari" 取元数据
2. 同命令加 --write-subs --write-auto-subs --sub-langs "en.*" 下载字幕
3. 解析为 [mm:ss.xx] 台词文本，滚动字幕去重
4. 降级：拿不到字幕时用官方页描述，此时切片只能是估算并显式标注

【阶段 3｜切片】
1. 挑传播力最高的一支作为切片对象
2. 按主题切 8–10 段：时间区间 + 小标题 + 核心要点（含原句时间点）
3. 另给 3 个高光点：时间点 + 内容 + 适用账号类型
4. 金句表：时间点 + 原句 + 讲者，全部可回官方字幕核验

【阶段 4｜配图】
1. yt-dlp -g -f 94 取 m3u8 → ffmpeg -i <m3u8> -c copy 下载视频（不要用 hlsnative）
2. 围绕台词点取候选帧 ±0.8 秒，程序化排除黑帧与转场
3. 用人脸+眼睛检测给候选帧打睁眼分，主画面用最高分帧；eyes ≤ 1 判眨眼需换帧
4. render-script 渲染 3:4 1440×1920，台词为核对后的译写，标注为后期绘制字幕
5. 生成 1–3 张并逐张质检

【阶段 5｜社媒文案】
标题抓眼球，正文 3–5 个要点并标注时间点，结尾 3–5 个标签，约 300 字，可直接复制发布

【阶段 6｜交付】
1. 汇总为 Markdown 主文档：精选、切片、金句表、配图说明（含原文对照）、文案、踩坑记录
2. 呈现主文档与全部配图
3. 写执行记录（只记高层结论，便于下期查重）

【铁律】
- 绝不编造引语、数据、播放量；缺失留 —
- 时间轴优先官方字幕，估算必须显式标注
- 每张成图必须过睁眼与画面质检
- 消耗 AI 生图额度前先告知预计成本
```

## 附录：睁眼评分脚本

按候选时间点批量取帧并打分，输出按分数降序，挑最高分做主画面。

```python
import subprocess, os, sys
import cv2

FF = "<ffmpeg 绝对路径>"
V  = "<视频路径>"
TMP = "_eye"
os.makedirs(TMP, exist_ok=True)

eye  = cv2.CascadeClassifier(os.path.join(cv2.data.haarcascades, "haarcascade_eye.xml"))
face = cv2.CascadeClassifier(os.path.join(cv2.data.haarcascades, "haarcascade_frontalface_default.xml"))

cands = [float(x) for x in sys.argv[1:]]
rows = []
for t in cands:
    f = os.path.join(TMP, f"e_{t:.2f}.jpg")
    subprocess.run([FF, "-y", "-ss", str(t), "-i", V, "-frames:v", "1", "-q:v", "2", f],
                   capture_output=True)
    img = cv2.imread(f)
    gray = cv2.equalizeHist(cv2.cvtColor(img, cv2.COLOR_BGR2GRAY))
    score, note = -1.0, "no-face"
    for (x, y, w, h) in face.detectMultiScale(gray, 1.12, 5, minSize=(48, 48)):
        roi = gray[y:y + int(h * 0.62), x:x + w]
        eyes = eye.detectMultiScale(roi, 1.10, 6, minSize=(14, 14))
        area = sum(e[2] * e[3] for e in eyes)
        s = len(eyes) * 1000 + area
        if s > score:
            score, note = s, f"face{w}x{h} eyes={len(eyes)} area={area}"
    rows.append((t, score, note))

for t, s, n in sorted(rows, key=lambda r: -r[1]):
    print(f"t={t:8.2f}  score={s:8.0f}  {n}")
```

用法：`python eye_score.py 0.4 0.8 1.2 2.0 2.8`
判读：`score = 0` 多为远景或无人脸；`eyes = 1` 大概率是眨眼或侧脸；取 `eyes ≥ 2` 且分数最高者。
依赖：`pip install "opencv-python-headless==4.10.0.84"`（5.x 已移除 `CascadeClassifier`）。
