# Product Video Maker

> 装到 Claude Code，一句话生成产品宣传视频。不需要 Premiere / After Effects。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Skill-blue.svg)](https://claude.ai)
[![Remotion](https://img.shields.io/badge/Remotion-4.0+-purple.svg)](https://www.remotion.dev/)
[![ffmpeg](https://img.shields.io/badge/ffmpeg-required-green.svg)](https://ffmpeg.org/)

Product Video Maker 是一个 **Claude Code 产品宣传视频制作 skill**。

它会从内容策划、配音制作、帧级音画同步、动画编排到最终渲染，
帮你用纯代码做出一条 45-65 秒的产品宣传视频（9:16 竖版 / 16:9 横版），
输出 MP4 文件，直接可发。

装好后直接说：

```
帮我做一条产品宣传视频
```

它会引导你走完整个流程：
内容策划 → 配音 → 时间轴 → 代码 → 渲染 → 迭代。

---

## 核心方法论：音频驱动一切

从 13 轮真实迭代（V1 → V13）中提炼的第一原则：

**视频节奏由音频决定，不是由视觉决定。**

```
配音录制 → ffmpeg 语音边界分析 → 计算帧时间 → 场景时长 → 动画 delay → SFX 对齐 → BGM ducking
```

每一个动画 delay 都是从语音边界数学推导的 —— 不是估的。

---

## 覆盖内容

| 主题 | 内容 |
|------|------|
| 音画同步 | ffmpeg silencedetect → 帧精度动画 delay |
| 场景节奏 | 有 VO / 纯视觉两类节奏，场景重叠过渡，CTA hold |
| 音量架构 | 三层音频（BGM ducking + VO + SFX），支持 volume > 1.0 |
| 动画模式 | Apple 风格淡入、CalloutCard 浮动卡片、场景包络线 |
| 手机 Mockup | scale 与可读性平衡、字幕避让 |
| 迭代工作流 | 单变量迭代、变更同步清单、反馈→修复映射表 |
| 反模式速查 | 7 个常见坑和正确做法 |
| 封面制作 | 社媒封面（rembg 抠图 + Playwright 渲染） |

---

## 安装

```bash
# 克隆到 Claude Code skills 目录
git clone https://github.com/sukezhong/product-video-maker.git ~/.claude/skills/product-video-maker
```

然后在 Claude Code 中调用：

```
/product-video-maker
```

### 前置依赖

- [Node.js](https://nodejs.org/) 18+
- [Remotion](https://www.remotion.dev/) (`npm create video@latest`)
- [ffmpeg](https://ffmpeg.org/)（音频分析）
- TTS 服务（Fish Audio / ElevenLabs）或真人录音

---

## 工作流总览

```
Phase 1  内容策划        定义视频参数、场景大纲、配音文案、设计风格
                         ↓
Phase 2  音频制作        生成 VO → ffmpeg 分析语音边界 → 确定 playbackRate
                         ↓
Phase 3  时间轴规划      排 VO/场景/SFX 帧级时间轴（纸上算清楚再写代码）
                         ↓
Phase 4  代码实现        Remotion React 组件 + Apple 风格动画
                         ↓
Phase 5  渲染与迭代      单变量改动 → 渲染 → 看 → 反馈 → 修 → 再渲染
```

---

## 快速上手

```bash
# 分析配音语音边界
ffmpeg -i vo/02-pain.mp3 -af silencedetect=noise=-30dB:d=0.3 -f null - 2>&1 | grep silence

# 测试渲染（前 5 帧）
npx remotion render PromoVideo --frames=0-5

# 完整渲染
npx remotion render PromoVideo --gl=angle
```

---

## 项目结构

```
promo-video/
  src/
    Root.tsx                — Composition 定义（尺寸、帧率、总时长）
    PromoVideo.tsx          — 主编排（VO + SFX + 场景 Sequences）
    styles.ts               — 设计 token（颜色、字体、spacing）
    components/
      PhoneMockup.tsx       — iPhone 手机壳组件
      DashboardScreens.tsx  — 产品界面模拟
  public/
    vo/                     — 配音音频
    *.mp3                   — 音效（tap, pop, whoosh, chime）
    bgm.mp3                 — 背景音乐
  out/
    PromoVideo.mp4          — 渲染输出
```

---

## 背景

从一个真实产品宣传视频的 13 轮迭代中提炼。Skill 里的每一条规则，都是因为踩过那个坑才写上去的。

## License

MIT
