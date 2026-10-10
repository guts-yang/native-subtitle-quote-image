# YouTube 素材获取手册

> 适用：把 YouTube 上的公开视频作为本 Skill 的输入源，需要拿到**元数据、字幕轨、完整视频流与精确帧**。
> 前提：视频是你有权访问的公开内容；使用的 Cookie 是**你本人账号**导出、并经你明确授权的凭据。不使用任何绕过付费墙、地区限制或 DRM 的手段。

## 0. 授权凭据（Cookie）

YouTube 对自动化请求会返回「Sign in to confirm you're not a bot」。取得授权后，用你自己账号导出的 Netscape 格式 Cookie 文件即可正常解析，无需 PO Token 之类的额外提供器。

```bash
# 导出后固定存放，例如
~/.workbuddy/credentials/youtube.com_cookies.txt
```

- 导出一次即可复用数月；长期项通常有效期约 1 年，短期项约 1 年以内，建议 1–3 个月更新一次。
- 禁止把 Cookie 文件提交进仓库（放进 `.gitignore` 之外的安全位置即可，本手册不涉及提交）。
- 无 Cookie 时：仍可拿到部分元数据；拿不到字幕与视频流，此时不要反复重试，直接走降级路径（见 §5）。

## 1. 元数据与字幕

```bash
unset NODE_OPTIONS                       # 关键：某些环境的 node shim 会拦截挑战求解
yt-dlp --cookies "$COOKIE" \
       --js-runtimes node \
       --extractor-args "youtube:player_client=web_safari" \
       --skip-download --write-subs --write-auto-subs --sub-langs "en.*" \
       -o "subs/%(id)s.%(ext)s" "<video_url>"
```

- `web_safari` 客户端在本场景实测最稳；`web` / `mweb` / `ios` 等可能卡在 PO Token。
- 把 VTT 解析成 `[mm:ss.xx] 台词` 文本并**对滚动字幕去重**，得到可检索的时间轴，后续金句定位全靠它。

字幕兜底路径（字幕接口报 PO Token 缺失时）：带 Cookie 抓取视频页 HTML → 正则取 `captionTracks` 里带签名的 `timedtext` URL → 追加 `&fmt=json3` 直接请求，请求头带 `Referer` 与桌面浏览器 UA。

## 2. 完整视频流

`web_safari` + Cookie + node 可直接拿到 HLS 直链，**不需要额外 PO Token 提供器**。

```bash
yt-dlp --cookies "$COOKIE" --js-runtimes node \
       --extractor-args "youtube:player_client=web_safari" \
       -g -f 94 "<video_url>" > stream.txt      # 取 m3u8 直链

ffmpeg -i "$(cat stream.txt)" -c copy -bsf:a aac_adtstoasc video.mp4
```

实测：20 分钟 480p 片源约 12 秒完成（约 64 MB）。

> ⚠️ 不要用 yt-dlp 内置 hlsnative 分片下载：它边下边删分片，在带删除保护的执行环境里会被中断（下到一半停住）。用 ffmpeg 直连 m3u8 输出单文件最稳。

## 3. 精确取帧

```bash
ffmpeg -ss <秒> -i "$(cat stream.txt)" -frames:v 1 -q:v 2 frame.jpg
```

HLS 直链 seek 很快（单帧约 2 秒）；若已下载完整 mp4，直接对本地文件取帧更快，也便于 Skill 的 `sample` / `render-script` 复用。

## 4. 主画面选帧：闭眼/空镜自动剔除

人物闭眼或镜头切到远景，会让主画面废掉。用 OpenCV 的人脸 + 眼睛级联分类器给候选帧打分：

```python
import cv2, os
face = cv2.CascadeClassifier(os.path.join(cv2.data.haarcascades, "haarcascade_frontalface_default.xml"))
eye  = cv2.CascadeClassifier(os.path.join(cv2.data.haarcascades, "haarcascade_eye.xml"))

# score = 检出眼数 × 1000 + 眼睛总面积；越大越可能是睁眼正脸
# 判定：eyes ≤ 1 或 score = 0 → 眨眼 / 非正脸，换帧
```

完整脚本见 [selection-to-post-workflow.md](selection-to-post-workflow.md#附录睁眼评分脚本)。实测能准确识别出人工判定的眨眼帧（该帧仅检出 1 只眼），也能识别出「镜头切远景导致画面里没有人物」的情况。

## 5. 降级路径与已知坑

| 现象 | 原因 | 处理 |
|---|---|---|
| `Access to this API has been restricted` | 环境的 `NODE_OPTIONS` 挂了 node shim，拦截挑战求解 | 先 `unset NODE_OPTIONS` |
| 报 PO Token 缺失 | 客户端不合适 | 换 `player_client=web_safari`；字幕走 §1 兜底路径 |
| 下载中途停住 | hlsnative 边下边删分片触发删除保护 | 改用 ffmpeg 直连 m3u8 |
| `module 'cv2' has no attribute 'CascadeClassifier'` | 装到了 OpenCV 5.x，已移除级联分类器 | 装 4.x：`pip install "opencv-python-headless==4.10.0.84"` |
| pip 装包十分钟无进展 | 默认源慢 | 换镜像源（如清华源），实测 16 秒完成 |
| 频道列表取不到真实上传日期 | 第三方节目单日期不准 | 用官方频道 RSS：`https://www.youtube.com/feeds/videos.xml?channel_id=<channel_id>` |

## 6. 合规边界

- 仅使用**本人账号**凭据，且仅用于自己有权访问的内容。
- 不绕过付费墙、地区限制、年龄限制中的付费部分或 DRM。
- 成图若使用脚本字幕模式，必须标注「后期绘制字幕，非画面烧录」；台词须能回到官方字幕或权威来源复核。
