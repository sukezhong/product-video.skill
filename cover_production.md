---
name: Cover Production Guide
description: 社媒封面制作全流程 — 工具链、抠图方案、尺寸规范、渲染流程、设计元素复用
type: reference
originSessionId: 68ca0abd-a456-4b9f-9a0d-4645dab8640b
---
## 工具链

| 环节 | 工具 | 说明 |
|------|------|------|
| 设计 | Claude Design (claude.ai/design) | HTML/CSS 可视化设计，支持拖拽编辑、实时预览。比 Figma 和 Remotion 代码方式高效很多 |
| 抠图 | rembg + `birefnet-portrait` 模型 | 专门优化肖像分割，复杂背景也能干净去除。venv 在 `/tmp/rembg-env/` |
| 白色描边 | CSS `drop-shadow` 8方向叠加 | 比 PIL 生成的 stroke 更灵活，可在 HTML 层面调整 |
| 渲染 | Playwright + Chromium headless | viewport 设为封面尺寸，deviceScaleFactor: 2 输出 retina PNG |
| 字体 | Google Fonts: Noto Sans SC (500/700/800/900) | 需要 waitForTimeout(3000) 等字体加载 |

## 抠图方案对比（已验证）

| 模型 | 质量 | 适用场景 |
|------|------|---------|
| `u2net` (rembg 默认) | 差 | 简单纯色背景 |
| `u2net_human_seg` | 一般 | 中等复杂度背景 |
| **`birefnet-portrait`** | **好** | **复杂背景人像，推荐首选** |
| Claude Design 内置 heuristic | 差 | 只适合纯色背景，复杂场景无法使用 |
| iOS Photos "Lift Subject" | 极好 | 用户手动操作，最干净 |
| remove.bg | 极好 | 需要 API key 或手动上传 |

**Why:** birefnet-portrait 是 973MB 的专用模型，首次使用需要下载到 `~/.u2net/`。

**抠图代码：**
```python
from rembg import remove, new_session
from PIL import Image
session = new_session("birefnet-portrait")
inp = Image.open("input.jpg")
out = remove(inp, session=session)
out.save("output.png")
```

## 封面尺寸规范

| 平台 | 比例 | 设计尺寸 | 渲染输出 (2x) |
|------|------|---------|-------------|
| 小红书 / 抖音 / 快手 | 3:4 | 1080×1440 | 2160×2880 |
| 视频号 / B站 | 16:9 | 1920×1080 | 3840×2160 |

**How to apply:** 同一套设计元素，两个 HTML 文件分别排版。竖版元素上下排列，横版元素左右排列。

## 白色描边 CSS 方案

```css
filter:
  drop-shadow(0 -7px 0 #fff)
  drop-shadow(7px 0 0 #fff)
  drop-shadow(0 7px 0 #fff)
  drop-shadow(-7px 0 0 #fff)
  drop-shadow(5px 5px 0 #fff)
  drop-shadow(-5px 5px 0 #fff)
  drop-shadow(5px -5px 0 #fff)
  drop-shadow(-5px -5px 0 #fff)
  drop-shadow(-10px 22px 26px rgba(50, 35, 25, .22));
```

8 个方向的 drop-shadow 模拟均匀描边，最后一个做地面阴影。~7px 效果比较好。

## Playwright 渲染流程

1. 从 Claude Design 导出 zip（含 HTML + assets/）
2. 去掉 `editor.js`（Claude Design 编辑器脚本）和 `fit()` viewport 缩放脚本
3. Playwright: viewport = 设计尺寸, deviceScaleFactor = 2
4. `page.screenshot({ clip: { x:0, y:0, width, height } })`
5. 输出 PNG 即为 2x retina 高清图

**渲染脚本模板：** `/tmp/mono-cover-v2/render.mjs` 和 `render-16x9.mjs`

## Mono 封面设计元素

| 元素 | 说明 |
|------|------|
| 标题 | "你的专属 / 生活搭子" — 900 weight, 120-132px |
| App 图标 | Mono 猫头图标，圆角 + rotate(8deg) |
| 人物抠图 | 右侧，白色贴纸描边 + 地面阴影 |
| 聊天卡片 | Mono 响应卡片（记账/热量/日记三行 + 营养提示） |
| 语录气泡 | 深色圆角气泡，"午饭吃了麻辣烫，32块" |
| 标签 | "一句话 · 三件事"，黑底黄字强调 |
| 红点 | 标题末尾小红圆点，品牌标识 |
| 背景 | 暖米色渐变 + paper grain 纸纹纹理 |

## Claude Design 使用技巧

- 项目链接：`https://claude.ai/design/p/10c3a0da-714a-442a-aa10-f06cf62d7820`
- 用 Shift+Enter 换行，Enter 直接发送
- 上传图片直接粘贴 (Cmd+V) 到聊天框
- Claude Design 无法做 AI 抠图，需要先在本地处理好再上传
- 如果上下文过多会提示 "Start a new chat"，选 New chat 不会丢失文件
- 编辑模式 (Edit) 可以直接拖拽调整元素位置和大小

## 文件位置

| 文件 | 说明 |
|------|------|
| `public/cover-final.png` | 3:4 竖版封面 (2160×2880) |
| `public/cover-16x9-final.png` | 16:9 横版封面 (3840×2160) |
| `/tmp/mono-cover-v2/` | 完整设计文件（HTML + assets + 渲染脚本） |
| `public/me/me.jpg` | 人物原图 |
