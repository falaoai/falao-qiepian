---
name: falao-qiepian
description: 直播切片全流程自动化工具。输入直播回放视频，自动转录→分析→切片→花字→多机位→产品弹窗→音效→去重→输出成品。适用：女装/服装直播切片、带货视频二创。
version: 1.2
homepage: https://github.com/falaoai/falao-qiepian
metadata:
  requires:
    - ffmpeg
    - python >= 3.10
    - Pillow (pip install Pillow)
    - openai-whisper (pip install openai-whisper)
    - Node.js 18+
    - Chrome/Chromium
  optional:
    - HyperFrames (首次使用时 npm install hyperframes)
---

# 直播切片 Skill

一站式直播回放 → 成品切片流水线。

## 触发词

直播切片、剪直播、切片去重、带货视频、女装切片、服装二创、批量切片

## 前置依赖

- `ffmpeg` / `ffprobe` 在 PATH
- `python` 3.10+ + `PIL` (Pillow)
- `whisper` (openai-whisper, `python -m whisper`)
- HyperFrames (npm 包，`npx hyperframes`)
- Chrome/Chromium (HyperFrames 渲染引擎)

首次使用务必检查依赖：
```bash
ffmpeg -version                         # 必须
python -c "from PIL import Image"       # 必须，缺则 pip install Pillow
python -m whisper --help | head -1      # 必须，缺则 pip install openai-whisper
node --version                          # 必须，>= 18
"/c/Program Files/Google/Chrome/Application/chrome.exe" --version  # 必须
```
HyperFrames 会在第一次跑花字步骤时自动 npm install，无需提前安装。

## 完整流水线

### 第 1 步：素材准备

```bash
# 1.1 提取音频
ffmpeg -y -i <直播.ts> -vn -acodec pcm_s16le -ar 16000 -ac 1 edit/audio.wav

# 1.2 ffprobe 获取视频参数
ffprobe -v error -show_entries format=duration -show_entries stream=width,height <直播.ts>
```

### 第 2 步：语音转文字

```bash
python -m whisper edit/audio.wav --model small --language zh --output_format json --output_dir edit/
```

输出：`edit/audio.json`（带词级时间戳）

### 第 3 步：LLM 分析选段 + 提取重点词时间戳

读取 `edit/audio.json`，完成两件事：

**① 选高价值片段**（用于切片截取）

| 类型 | 关键词信号 | 价值 |
|------|-----------|------|
| 产品展示 | 这件/这套/看一下/新款 | 高 |
| 上身效果 | 穿起来/上身/给你们看 | 最高 |
| 面料卖点 | 面料/材质/纯棉/真丝 | 高 |
| 价格亮点 | 价格/只要/划算/才 | 最高 |
| 促单话术 | 链接/下单/橱窗/库存 | 最高 |

**② 提取重点词时间戳**（用于花字定位）

从选定片段的转录文本中，提取 3-7 个重点词，每个词输出：
```
词: "95%棉"
开始时间: 3.8s（用 whisper segment 的 start）
结束时间: 6.5s
位置: 左侧中间          ← 固定：画面左半区，垂直居中
```

重点词选取优先级：面料成分 > 支数/品质 > 价格 > 促单口号 > 上身效果

输出：选定片段的起止时间 + 重点词列表（词/开始时间/结束时间）

### 第 4 步：基础切片

```bash
# 从原视频截取片段（根据第3步的时间戳）
ffmpeg -y -ss <start> -i <直播.ts> -t <duration> -c:v libx264 -preset fast -crf 18 -r 30 -c:a aac -b:a 128k edit/clip_raw.mp4
```

### 第 5 步：去重 + 多机位

```bash
# 5.1 生成3个机位角度（A:中景 B:近景 C:特写）
# 机位A - 中景（原构图）
ffmpeg -y -i edit/clip_raw.mp4 -vf "scale=1080:1920" -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 128k edit/cam_A.mp4

# 机位B - 近景（放大15%，居中）
ffmpeg -y -i edit/clip_raw.mp4 -vf "scale=1242:2208:flags=lanczos,crop=1080:1920:81:144" -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 128k edit/cam_B.mp4

# 机位C - 特写（放大30%，偏上）
ffmpeg -y -i edit/clip_raw.mp4 -vf "scale=1404:2496:flags=lanczos,crop=1080:1920:60:250" -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 128k edit/cam_C.mp4

# 5.2 多机位切换 + 溶解转场（用Python生成filter_complex）
# 模板：每2-3秒切换机位，xfade=fade:0.3s 过渡
# 参见下方"多机位切换脚本"
```

### 第 6 步：花字字幕（HyperFrames GSAP）

```bash
# 6.1 初始化 HyperFrames 项目
mkdir -p edit/animations/slot_1 && cd edit/animations/slot_1
npm init -y && npm install hyperframes

# 6.2 创建 index.html（花字叠加层）
# 模板见下方"花字模板"
# 关键参数：1080x1920, 30fps, data-composition-id="huazi"

# 6.3 渲染
npx hyperframes lint . && npx hyperframes render . -o render.webm --format webm

# 6.4 验证渲染结果（关键！没字幕 90% 是这个原因）
ls -lh render.webm
ffprobe -v error -show_entries stream=codec_name,width,height edit/animations/slot_1/render.webm
# ⚠️ 如果 render.webm < 50KB，说明渲染失败：
#   1. 检查 GSAP CDN：curl -I https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js
#   2. CDN 挂了就换本地：npm install gsap，改 HTML src 为 ./node_modules/gsap/dist/gsap.min.js
#   3. 重新渲染
```

### 第 7 步：产品截图弹窗

```bash
# 7.1 从原视频截取产品画面
ffmpeg -y -ss <产品出现时间> -i <直播.ts> -vframes 1 -q:v 2 edit/product_frame.jpg

# 7.2 用 Python/PIL 做圆形白边
python -c "
from PIL import Image, ImageDraw
img = Image.open('edit/product_frame.jpg')
w, h = img.size
size = min(w, h) // 2
crop = img.crop((w//4, h//6, w//4+size, h//6+size)).resize((390,390), Image.LANCZOS)
mask = Image.new('L', (390,390), 0)
ImageDraw.Draw(mask).ellipse((0,0,390,390), fill=255)
result = Image.new('RGBA', (400,400), (0,0,0,0))
ImageDraw.Draw(result).ellipse((0,0,400,400), fill=(255,255,255,255))
inner = Image.new('RGBA', (390,390))
inner.paste(crop, (0,0)); inner.putalpha(mask)
result.paste(inner, (5,5), inner)
result.save('edit/product_circle.png')
"

# 7.3 合成时用 ffmpeg overlay（弹窗位置可调）
# overlay=x:(H-h)/2  → 画面居中竖直
# overlay=60:(H-h)/2  → 左边居中
# enable='between(t,<start>,<end>)'
```

### 第 8 步：弹出音效

```bash
# 8.1 生成"噗"音效
python -c "
import struct, wave, math
sr = 44100
dur = 0.18
samples = [int(32767 * math.exp(-t/sr*20) * 0.8 * math.sin(2*math.pi*(800-400*t/(sr*dur))*t/sr)) for t in range(int(sr*dur))]
with wave.open('edit/pop.wav','w') as w:
    w.setnchannels(1); w.setsampwidth(2); w.setframerate(sr)
    w.writeframes(struct.pack('<' + 'h'*len(samples), *samples))
"

# 8.2 合成时混入音效
# [pop.a]adelay=<ms>|<ms>,volume=0.5[pops]
# [main.a][pops]amix=2:duration=first:weights=1 0.3,alimiter=limit=0.95[a]
```

### 第 9 步：最终合成

```bash
ffmpeg -y \
  -i edit/clip_multicam.mp4 \      # 多机位+溶解
  -i edit/animations/slot_1/render.webm \  # 花字
  -i edit/product_circle.png \     # 产品图
  -i edit/pop.wav \                # 音效
  -filter_complex "
    # similarity=0.5 更宽容，避免背景不是纯黑时抠不干净导致字幕消失
    [1:v]colorkey=0x000000:0.5:0.2[huazi];
    [0:v][huazi]overlay=0:0[base];
    [base][2:v]overlay=60:(H-h)/2:enable='between(t,4.0,6.5)'[v];
    [3:a]adelay=4050|4050|9000|9000|27650|27650,volume=0.5[pops];
    [0:a][pops]amix=2:duration=first:weights=1 0.3,alimiter=0.95[a]
  " \
  -map "[v]" -map "[a]" \
  -c:v libx264 -preset fast -crf 18 -c:a aac -b:a 128k \
  edit/FOLOERGE_最终.mp4
```

## 多机位切换脚本

用 Python 生成 ffmpeg 的 `filter_complex`，实现多机位 + 溶解转场：

```python
segments = [
    (0, 0, 2.8),     # 0=cam_A中景
    (1, 3.0, 4.8),   # 1=cam_B近景
    (2, 5.0, 14.8),  # 2=cam_C特写
    (1, 15.0, 17.8),
    (0, 18.0, 27.8),
    (2, 28.0, 30.4),
]

parts = []
out = '[v0]'; a_pads = ''; td = 0; xd = 0.3  # xfade duration
xfade_chain = ''
for i, (cam, s, e) in enumerate(segments):
    d = e - s
    parts.append(f'[{cam}:v]trim={s}:{e},setpts=PTS-STARTPTS[v{i}]')
    parts.append(f'[{cam}:a]atrim={s}:{e},asetpts=PTS-STARTPTS[a{i}]')
    a_pads += f'[a{i}]'
    if i > 0:
        off = td - i * xd
        xfade_chain += f'{out}[v{i}]xfade=transition=fade:duration={xd}:offset={off:.1f}[x{i}];'
        out = f'[x{i}]'
    td += d

filt = ';'.join(parts) + ';' + xfade_chain + f'{a_pads}concat=n={len(segs)}:v=0:a=1[aout]'
```

然后 `-map "[x5]" -map "[aout]"` 分别映射视频和音频。

## 花字模板（HyperFrames）

**两种花字并存**，分别打在不同位置：

| 类型 | 位置 | 效果 | 用途 |
|------|------|------|------|
| 底部字幕 | bottom:240-280px, 居中 | 弹跳/弹性/滑入轮换 | 主播整句旁白 |
| 左侧重点词 | left:50px, top:800-1000px（主播左中） | 极简逐显（字依次淡入上滑+金线） | 主播说的核心卖点词 |

---

### 底部字幕模板（弹跳/弹性/滑入轮换）

**注意**：text-shadow 用简洁黑色描边，不要用红色大光晕，避免"红色乱七八糟"。

```html
  <!-- 底部字幕 Slot：弹跳/弹性轮换 -->
  <div id="b1" class="clip" data-start="0.0" data-duration="2.5" data-track-index="0"
    style="position:absolute;bottom:280px;left:0;right:0;display:flex;justify-content:center;">
    <div id="b1-t" style="font:bold 46px 'Microsoft YaHei',sans-serif;color:white;
      -webkit-text-stroke: 1px rgba(0,0,0,0.6);
      text-shadow: 0 2px 4px rgba(0,0,0,0.5);">
      95%棉 + 5%氨纶
    </div>
  </div>
```

底部字幕动画轮换（避免重复感）：
```js
  tl.from("#b1-t", {y:30, opacity:0, duration:0.35, ease:"back.out(1.7)"}, 0.0);   // 弹跳
  tl.from("#b2-t", {scale:0.3, opacity:0, duration:0.45, ease:"elastic.out(1,0.4)"}, 4.08); // 弹性
  tl.from("#b3-t", {x:-40, opacity:0, duration:0.35, ease:"power2.out"}, 9.3);        // 滑入
```

---

### 左侧重点词模板（极简逐显，v1.1 新增）

**定位**：主播左侧中间，文字左对齐（不居中）

```html
  <!-- 重点词：面料成分（主播左侧中间，极简逐显） -->
  <div id="s1" class="clip" data-start="3.8" data-duration="2.5" data-track-index="1"
    style="position:absolute;left:50px;top:880px;">
    <div id="s1-t" style="display:inline-flex;gap:2px;align-items:baseline;flex-wrap:nowrap;">
      <span class="kd" style="font:900 20px 'Microsoft YaHei';color:#FFD700;opacity:0;">95%</span>
      <span class="kd" style="font:900 17px 'Microsoft YaHei';color:#ffffff;opacity:0;margin-left:2px;">棉</span>
      <span class="kd" style="font:300 15px 'Microsoft YaHei';color:#666;opacity:0;margin:0 4px;">+</span>
      <span class="kd" style="font:900 20px 'Microsoft YaHei';color:#FF6B9D;opacity:0;">5%</span>
      <span class="kd" style="font:900 17px 'Microsoft YaHei';color:#ffffff;opacity:0;">氨纶</span>
    </div>
    <div id="s1-line" style="height:1px;width:0;background:linear-gradient(90deg,transparent,#FFD700,transparent);margin-top:5px;"></div>
  </div>
```

**极简逐显 JS**（每个字依次淡入上滑 + 底部金线展开）：
```js
  // 对所有重点词 slot 执行
  ['s1','s2','s3','s4','s5'].forEach(function(id){
    var chars = document.querySelectorAll('#' + id + ' .kd');
    var line = document.getElementById(id + '-line');
    var startTime = parseFloat(document.getElementById(id).dataset.start);

    chars.forEach(function(ch, i){
      tl.to(ch, { opacity:1, y:0, duration:0.18, ease:"power2.out" }, startTime + i * 0.08);
    });
    tl.to(line, { width:'100%', duration:0.4, ease:"power2.out" }, startTime + chars.length * 0.08);
    tl.to('#' + id, { opacity:0, duration:0.25, ease:"power2.in" }, startTime + 2.0);
  });
```

---

### 完整 index.html 结构（底部 + 左侧重点词并存）

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<style>
  * { margin:0; padding:0; box-sizing:border-box; }
  /* 必须用纯黑背景！transparent 在 HyperFrames 渲染时经常失效，
     导致背景不是 #000000，colorkey 抠不干净，字幕直接消失 */
  body { background: #000000; overflow: hidden; }
</style>
</head>
<body>
<!-- data-width / data-height 必须从原视频动态读取，不能硬编码！
     先用 ffprobe 获取，再写入 HTML -->
<div id="root" class="clip" data-composition-id="huazi" data-start="0" data-duration="28"
  data-width="{{VIDEO_WIDTH}}" data-height="{{VIDEO_HEIGHT}}" data-track-index="0">

  <!-- 底部字幕（track 0）：弹跳/弹性轮换 -->
  <div id="b1" class="clip" data-start="0.0" data-duration="2.5" data-track-index="0"
    style="position:absolute;bottom:280px;left:0;right:0;display:flex;justify-content:center;">
    <div id="b1-t" style="font:bold 46px 'Microsoft YaHei',sans-serif;color:white;
      -webkit-text-stroke: 1px rgba(0,0,0,0.6);
      text-shadow: 0 2px 4px rgba(0,0,0,0.5);">
      95%棉 + 5%氨纶
    </div>
  </div>

  <!-- 左侧重点词（track 1）：极简逐显 -->
  <div id="s1" class="clip" data-start="3.8" data-duration="2.5" data-track-index="1"
    style="position:absolute;left:50px;top:880px;">
    <div id="s1-t" style="display:inline-flex;gap:2px;align-items:baseline;">
      <span class="kd" style="font:900 20px 'Microsoft YaHei';color:#FFD700;opacity:0;">95%</span>
      <span class="kd" style="font:900 17px 'Microsoft YaHei';color:#fff;opacity:0;">棉</span>
    </div>
    <div id="s1-line" style="height:1px;width:0;background:linear-gradient(90deg,transparent,#FFD700,transparent);margin-top:5px;"></div>
  </div>

</div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.5/gsap.min.js"></script>
<script>
(function(){
  // 错误保护：如果 GSAP 加载失败，至少让字幕直接显示
  window.addEventListener('error', function(e){
    console.error('JS Error:', e.message);
    // 兜底：直接显示所有字幕元素
    document.querySelectorAll('.kd').forEach(function(el){ el.style.opacity = 1; el.style.transform = 'none'; });
  });

  window.__timelines = window.__timelines || {};
  var tl = gsap.timeline();

  // 底部字幕：弹跳入场
  tl.from("#b1-t", {y:30,opacity:0,duration:0.35,ease:"back.out(1.7)"}, 0.0);

  // 左侧重点词：极简逐显（先 set 初始 y:15+opacity:0，再 to y:0+opacity:1，才有上滑效果）
  var chars = document.querySelectorAll('#s1 .kd');
  chars.forEach(function(ch){ gsap.set(ch, {y:15, opacity:0}); });  // ← 关键：先设初始状态
  chars.forEach(function(ch,i){
    tl.to(ch, {opacity:1, y:0, duration:0.22, ease:"power2.out"}, 3.8 + i*0.09);
  });
  tl.to('#s1-line', {width:'100%',duration:0.4,ease:"power2.out"}, 3.8 + chars.length*0.09);
  tl.to('#s1', {opacity:0,duration:0.3}, 5.6);

  window.__timelines["huazi"] = tl;
})();
</script>
</body>
</html>
```

花字配色（两种共用）：
- 数字/百分比：金色 #FFD700，font:900 20px
- 中文材质名：白色 #ffffff，font:900 17px  
- 连接符/标点：深灰 #666，font:300 15px
- 强调词：粉色 #FF6B9D，font:900 20px
- 底部字幕描边：`-webkit-text-stroke: 1px rgba(0,0,0,0.6)` + `text-shadow: 0 2px 4px rgba(0,0,0,0.5)`（简洁黑色描边，不用红色光晕）
- 左侧重点词底部金线：linear-gradient(transparent, #FFD700, transparent)

## 去重手段汇总

| 手段 | 实现 | 效果 |
|------|------|------|
| 多机位切换 | 3 角度交替 + xfade | ⭐⭐⭐⭐⭐ |
| 溶解转场 | xfade=fade:0.3 | ⭐⭐⭐⭐ |
| 画面镜像 | hflip 轮换 | ⭐⭐⭐ |
| 画面裁切 | 不同 crop 区域 | ⭐⭐⭐⭐ |
| 变速（Todo） | setpts 加速/减速 | ⭐⭐⭐⭐⭐ |
| 随机调色（Todo） | eq 亮度/饱和度微调 | ⭐⭐⭐⭐ |

## 输出检查

```bash
ffprobe -v error -show_entries format=duration,size <output.mp4>
# 预期：duration≈30s, size≈15-19MB, 1080x1920
```

## 已知限制

- zoompan（动态推拉）太慢，暂用多机位模拟。真正平滑推拉需 Remotion
- HyperFrames 音频处理不稳定，音效走 ffmpeg 混音
- colorkey 抠黑可能吃掉深色衣服细节，已换 ffmpeg overlay 直叠产品图

## 目录结构

```
<video_dir>/
├── <source.ts>              原始直播回放
└── edit/
    ├── audio.wav             提取的音频
    ├── audio.json            Whisper 转录
    ├── clip_raw.mp4          基础切片
    ├── cam_A/B/C.mp4         3 个机位
    ├── clip_multicam.mp4     多机位+溶解
    ├── product_circle.png    产品圆图
    ├── pop.wav               弹出音效
    ├── animations/
    │   └── slot_1/
    │       ├── index.html    花字模板
    │       └── render.webm   花字渲染
    └── <output>.mp4          最终成品
```
