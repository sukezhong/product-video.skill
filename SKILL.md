---
name: product-video-maker
description: 产品宣传视频制作全流程（基于 Remotion）。涵盖完整工作流、音画同步方法论、音量架构、场景节奏控制、过渡动画、手机 Mockup 排版。适用于 App 演示视频、产品宣传短视频。
---

# Remotion 宣传视频制作最佳实践

> 从 Mono 宣传视频 V1-V13 的 13 轮迭代中提炼。适用于 9:16 竖版产品宣传视频、App 演示视频、短视频广告。

## 第一性原理：音频驱动一切

视频制作的核心节奏由**音频**决定，不是由视觉决定。

```
VO 录制/TTS生成 -> ffmpeg 分析语音边界 -> 计算帧时间 -> 定义场景时长 -> 设定动画 delay -> 对齐 SFX -> 对齐 BGM ducking
```

**永远不要**：先设计动画时长，再让配音去填充。

---

## 0. 完整工作流（从零到成品）

### Phase 1: 内容策划（不写代码）

```
输入：产品定位、目标平台、目标受众
输出：V*-CONTENT-PLAN.md（确认后锁定）
```

1. **定义视频参数**：格式（9:16/16:9）、时长预算（短视频 45-65s）、FPS（30）
2. **设计场景大纲**：每个场景的目的、内容概要、预估时长
   - Hook (3-4s) -> Pain Point (8-12s) -> Brand Reveal (5s) -> Feature Demo x N -> CTA (8-12s)
3. **写配音文案**：每个场景的逐字稿，标注语气和节奏
4. **确定设计风格**：配色、字体、图标库、动画风格
5. **用户确认** -> 锁定 CONTENT-PLAN.md

### Phase 2: 音频制作（视觉之前）

```
输入：配音文案
输出：public/vo/*.mp3 + 每段 VO 的语音边界数据
```

1. **生成/录制 VO**
   - TTS：Fish Audio / ElevenLabs 等，生成后放 `public/vo/`
   - 真人录音：录制后转换格式 `ffmpeg -i input.m4a -ar 44100 -ac 1 output.wav`

2. **分析每段 VO 的语音边界**（关键步骤）
   ```bash
   for f in public/vo/*.mp3; do
     echo "=== $f ==="
     ffmpeg -i "$f" -af silencedetect=noise=-30dB:d=0.3 -f null - 2>&1 | grep silence
   done
   ```

3. **确定 playbackRate**
   - TTS 声音：1.3x-1.5x（TTS 普遍偏慢）
   - 真人录音：1.0x-1.3x
   - 试听确认自然度

4. **计算每段 VO 的实际帧数**
   ```
   实际秒数 = 原始秒数 / playbackRate
   所需帧数 = ceil(实际秒数 * FPS) + 10（余量）
   ```

5. **准备 BGM 和 SFX**
   - BGM：找一首风格匹配的背景音乐，时长 >= 视频总时长
   - SFX：准备 tap/pop/boom/whoosh/chime 等基础音效

### Phase 3: 时间轴规划（纸上算清楚再写代码）

```
输入：每段 VO 的帧数 + 场景大纲
输出：完整的帧级时间轴（写在代码注释里）
```

1. **排 VO 时间轴**：按场景顺序，每段 VO 之间留 gap（6-8帧），纯视觉场景不占 VO 时间
   ```
   S1 Hook VO:     frame 2-148     (146f @ 1.4x)
   S2 Dashboard:   -- 无 VO --     (70f 纯视觉)
   S3 Pain VO:     frame 208-528   (320f @ 1.3x)
   ...
   ```

2. **排场景时间轴**：场景比 VO 略宽（前后各多 5-10 帧），相邻场景重叠 10 帧
   ```
   S1  0-155
   S2  145-215    (重叠 S1 尾部 10f)
   S3  205-545    (重叠 S2 尾部 10f)
   ...
   ```

3. **从 VO 语音边界推导动画 delay**
   ```
   Pain VO 第一句话在原始音频 1.53s 开始
   playbackRate 1.3x -> 实际 1.53/1.3 = 1.18s
   Pain 场景 VO 从 global frame 208 开始 -> local frame 0
   local delay = 1.18 * 30 = 35
   ```

4. **排 FLASH_POINTS**：每个场景过渡点（后一个场景 from - 2）

5. **排 SFX 时间轴**：每个 SFX 对齐到 global frame 的动画事件
   ```
   Pain line1 出现 = global 205 + local 35 = 240 -> tap SFX at 240
   ```

6. **算总时长** -> 更新 Root.tsx DURATION_SECONDS

### Phase 4: 代码实现

```
输入：完整时间轴
输出：可渲染的 Remotion 项目
```

按以下顺序实现（依赖关系从上到下）：

1. **Root.tsx** — Composition 注册（尺寸、FPS、总帧数）
2. **styles.ts** — 设计 token（颜色、字体、spacing）
3. **PhoneMockup.tsx** — 手机壳组件（如果需要展示 App 界面）
4. **各场景组件** — 每个场景一个 React 组件，内部用 `useCurrentFrame()` + `useAppleFade` 驱动动画
5. **DashboardScreens.tsx** — 产品界面模拟组件（纯视觉场景内容）
6. **PromoVideo.tsx（主编排文件）**：
   - 写 VO_RANGES（BGM ducking 用）
   - 写 getBgmVolume 函数
   - 排 VO Sequences（from + durationInFrames + playbackRate）
   - 排 Scene Sequences（from + durationInFrames）
   - 排 SFX（SoundDesign 组件）
   - 排 TransitionFlashes

### Phase 5: 渲染与迭代

```
输入：完整代码
输出：最终 MP4
```

1. **快速验证构建**
   ```bash
   npx remotion render PromoVideo --frames=0-5
   ```

2. **完整渲染**
   ```bash
   npx remotion render PromoVideo --gl=angle
   open out/PromoVideo.mp4
   ```

3. **Review -> 单变量迭代**
   - 每轮只改一个维度（音量 / 时间轴 / 动画 / 文案大小）
   - 渲染 -> 用户看 -> 反馈 -> 修改 -> 渲染
   - 参考「9.3 用户反馈的优先级映射」定位问题

4. **常见迭代轮次**（经验值）：
   - Round 1-2: 时间轴和音画同步（最重要，先搞对）
   - Round 3: 音量平衡（BGM/SFX/VO 相对比例）
   - Round 4: 视觉打磨（字体大小、间距、动画节奏）
   - Round 5: 最终微调（CTA hold 时间、过渡流畅度）

### 工作流检查清单

在每个 Phase 结束时自检：

**Phase 2 结束后**：
- [ ] 每段 VO 都跑了 silencedetect，语音边界数据已记录
- [ ] playbackRate 已试听确认
- [ ] BGM 时长 >= 视频预估总时长

**Phase 3 结束后**：
- [ ] 所有 VO Sequence from 值已确定
- [ ] 所有 Scene Sequence from/duration 已确定，相邻重叠 10f
- [ ] 所有动画 delay 从语音边界推导（不是猜的）
- [ ] FLASH_POINTS 与场景过渡点对齐
- [ ] SFX 时间与动画事件对齐
- [ ] 总帧数已算好

**Phase 5 每轮迭代后**：
- [ ] 只改了一个维度
- [ ] 改 Scene from 时同步了：VO from、SFX at、FLASH_POINTS、envelope duration
- [ ] 改 VO playbackRate 时同步了：durationInFrames、动画 delay、VO_RANGES

## 1. 音画同步方法论（最重要）

### 1.1 分析 VO 语音边界

```bash
# 分析 VO 文件中每段语音的起止时间
ffmpeg -i vo/02-pain.mp3 -af silencedetect=noise=-30dB:d=0.3 -f null - 2>&1 | grep silence

# 输出示例：
# [silencedetect] silence_end: 1.53 | silence_duration: 1.53
# [silencedetect] silence_start: 3.83
# [silencedetect] silence_end: 4.28 | silence_duration: 0.45
```

解读：第一段语音 = 1.53s ~ 3.83s，第二段 = 4.28s ~ 下一个 silence_start

### 1.2 换算成帧号

```
帧号 = (原始时间 / playbackRate) * FPS

例：原始时间 1.53s，playbackRate 1.3x，FPS 30
帧号 = (1.53 / 1.3) * 30 = 35.3 -> 35
```

### 1.3 设定动画 delay

将每段语音的**开始帧**作为对应动画元素的 delay：

```tsx
// BAD: 拍脑袋猜的 delay
const painLines = [
  { text: '记账App不知道你在减肥', delay: 10 },
  { text: '饮食App不知道你肠胃不好', delay: 65 },
]

// GOOD: 从 ffmpeg silencedetect 推导的 delay
const painLines = [
  { text: '记账App不知道你在减肥', delay: 35 },   // 语音 1.53s/1.3x*30
  { text: '饮食App不知道你肠胃不好', delay: 99 },  // 语音 4.28s/1.3x*30
]
```

### 1.4 验证同步

渲染后逐帧检查关键节点：
- 文字出现帧 vs VO 开始说该句话的帧
- 允许误差：+/-3 帧（0.1s）
- 超过 +/-5 帧用户就能感知到不同步

---

## 2. 场景节奏控制

### 2.1 两类场景，两种节奏

| 场景类型 | 特征 | 时长策略 | 典型帧数 |
|---------|------|---------|---------|
| **有 VO 场景** | 配音驱动 | VO 结束后 +10~15 帧即切 | 由 VO 决定 |
| **纯视觉场景** | 无配音，展示内容 | 让用户有时间"看" | 70-120 帧 (2.3-4s) |

**核心规则**：
- 有 VO 的场景不要留大段空白（用户会觉得"卡了"）
- 纯视觉场景不要一闪而过（用户来不及看内容）
- CTA 场景所有元素出完后必须 hold 3-4 秒

### 2.2 场景 Envelope 模式

每个场景用 `useSceneEnvelope` 控制进出：

```tsx
const useSceneEnvelope = (
  localFrame: number, sceneDuration: number, fadeIn = 10, fadeOut = 15,
) => {
  const enter = interpolate(localFrame, [0, fadeIn], [0, 1], clamp)
  const exit = interpolate(localFrame, [sceneDuration - fadeOut, sceneDuration], [1, 0], clamp)
  return enter * exit  // 0->1->1->0 的包络线
}
```

**关键**：`sceneDuration` 必须和 Sequence 的 `durationInFrames` 一致，否则淡出时间不对。

### 2.3 场景重叠过渡

相邻场景 Sequence 重叠 10 帧，配合白色 flash：

```tsx
{/* S3 结束 545, S4 开始 535 -> 重叠 10f */}
<Sequence from={205} durationInFrames={340}><PainScene /></Sequence>
<Sequence from={535} durationInFrames={155}><BrandScene /></Sequence>
```

Flash 点 = 后一个场景开始前 2 帧：

```tsx
const FLASH_POINTS = [143, 203, 533, 678, ...] as const

// 白色脉冲
if (frame >= p && frame <= p + 6) {
  opacity = interpolate(frame, [p, p+2, p+6], [0, 0.18, 0], clamp)
}
```

---

## 3. 音量架构

### 3.1 三层音频

```
Layer 1: BGM（持续）  -- 音量随 VO 自动 ducking
Layer 2: VO（分段）   -- 0.85~0.95，是锚点
Layer 3: SFX（点状）  -- 与动画节点对齐
```

### 3.2 BGM Ducking

定义所有 VO 出现的帧范围，BGM 在 VO 期间降低：

```tsx
const VO_RANGES: readonly [number, number][] = [
  [2, 155],       // scene 1 VO
  [208, 530],     // scene 3 VO
  // ...
]

const getBgmVolume = (f: number): number => {
  for (const [s, e] of VO_RANGES) {
    if (f >= s - 8 && f <= e + 8) {
      if (f < s) return interpolate(f, [s-8, s], [1.6, 1.15], clamp)  // ramp down
      if (f > e) return interpolate(f, [e, e+8], [1.15, 1.6], clamp)  // ramp up
      return 1.15  // ducked
    }
  }
  return 1.6  // full volume (no VO)
}
```

### 3.3 音量 > 1.0 是合法的

Remotion `<Audio volume>` 支持大于 1.0 的值（信号放大）：

```tsx
// OK: 完全合法
<Audio src={staticFile('bgm.mp3')} volume={1.6} />

// BAD: 不要人为限制
const vol = Math.min(volume * boost, 1.0)  // 这会让 BGM/SFX 永远太小声
```

推荐基线：
- BGM 无 VO：1.4~1.6
- BGM 有 VO：1.0~1.2
- VO：0.85~0.95
- SFX：基础 0.6~0.9 x SFX_BOOST (1.5~2.0)

### 3.4 SFX 时间对齐

SFX 必须对齐到**全局帧**的动画节点：

```tsx
// 场景从 frame 205 开始，第一行文字 delay 35
// -> SFX tap 应在 205 + 35 = 240
<Sfx at={240} src="tap.mp3" volume={0.65} />
```

---

## 4. 动画模式

### 4.1 Apple 风格淡入

```tsx
const useAppleFade = (localFrame: number, delay: number, dur = 15) => ({
  opacity: interpolate(localFrame - delay, [0, dur], [0, 1], clamp),
  y: interpolate(localFrame - delay, [0, dur + 3], [14, 0], clamp),
})
```

使用：元素从下方 14px 淡入上滑，自然且不突兀。

### 4.2 内容出现顺序

```
标题/主文案 -> 停 8-15 帧 -> 手机 mockup -> 停 -> 内容填充
```

不要所有元素同时出现，有节奏感。

### 4.3 CTA Hold 时间

CTA 场景特殊处理——最后一个元素出完后，保持静止画面 3-4 秒：

```tsx
// 最后一个元素在 frame 168 出完
// 白屏淡出推迟到 frame 280
const finalFade = interpolate(frame, [280, 310], [0, 1], clamp)
```

### 4.4 CalloutCard 浮动卡片（V13 新增）

用于纯视觉 Dashboard 场景——观众来不及从完整界面中找到重点时，提取关键内容到浮动大卡片。

```tsx
const CalloutCard: React.FC<{
  frame: number
  appearAt: number
  children: React.ReactNode
}> = ({ frame, appearAt, children }) => {
  const calloutOpacity = interpolate(frame, [appearAt, appearAt + 6], [0, 1], clamp)
  const calloutScale = interpolate(frame, [appearAt, appearAt + 8], [0.88, 1], clamp)
  const dimOpacity = interpolate(frame, [appearAt, appearAt + 6], [0, 0.35], clamp)
  if (frame < appearAt) return null
  return (
    <>
      {/* 半透明遮罩让背景变暗 */}
      <div style={{ position: 'absolute', inset: 0, background: '#000', opacity: dimOpacity, zIndex: 20 }} />
      {/* 浮动内容卡片 */}
      <div style={{ position: 'absolute', inset: 0, display: 'flex', alignItems: 'center', justifyContent: 'center', zIndex: 21, opacity: calloutOpacity }}>
        <div style={{
          background: '#fff', borderRadius: 20, padding: '28px 32px',
          borderLeft: '4px solid #e07a7a',
          boxShadow: '0 12px 48px rgba(0,0,0,0.18)',
          transform: `scale(${calloutScale})`, maxWidth: 460,
        }}>
          {children}
        </div>
      </div>
    </>
  )
}
```

**使用场景**：
- 前一个场景（聊天）说了"午饭麻辣烫 32 块"，下一个 Dashboard 场景要突出"麻辣烫已记录 + ¥32"
- 纯展示场景 90 帧（3s），callout 在 frame 18 弹入，给观众 ~2s 阅读时间

**配合音效**：callout 出现时触发 chime SFX（SFX at = scene_start + appearAt）

---

## 5. 手机 Mockup 排版

### 5.1 Scale 与可读性的平衡

```tsx
<PhoneMockup delay={10} scale={1.05}>
  <ChatScreen />
</PhoneMockup>
```

- `scale > 1.1`：手机占据太多画面，字幕被遮挡
- `scale < 0.95`：手机太小，内部内容看不清
- **推荐 scale：1.0~1.08**

### 5.2 字幕避让

手机下方必须留足字幕空间：

```tsx
// 字幕容器
<div style={{ marginBottom: 28 }}>字幕文字</div>
```

### 5.3 Dashboard 场景用更大 scale

纯展示的 Dashboard 场景可以用 1.15，因为没有额外字幕：

```tsx
<div style={{ transform: 'scale(1.15)', transformOrigin: 'center top' }}>
  <PhoneMockup delay={3}>
    <DashboardContent />
  </PhoneMockup>
</div>
```

---

## 6. VO 管理

### 6.1 PlaybackRate 分层

```
TTS 生成的 VO：playbackRate 1.3~1.5（TTS 普遍偏慢）
真人录音的 VO：playbackRate 1.0~1.3（已有自然节奏）
```

### 6.2 VO Sequence 布局

每段 VO 独立 Sequence，gap 6-8 帧避免重叠：

```tsx
<Sequence from={688} durationInFrames={140}>
  <Audio src={staticFile('vo/04a-demo.mp3')} playbackRate={1.2} />
</Sequence>
<Sequence from={827} durationInFrames={92}>  {/* gap 8f */}
  <Audio src={staticFile('vo/04a-user.mp3')} playbackRate={1.4} />
</Sequence>
```

### 6.3 VO 时长计算

```
VO 原始时长(s) / playbackRate * FPS = 所需帧数
durationInFrames = 所需帧数 + 10（留余量避免截断）
```

---

## 7. 项目结构

```
promo-video/
  src/
    Root.tsx              -- Composition 定义（尺寸、帧率、总时长）
    PromoVideo.tsx        -- 主编排：VO + SFX + Scene Sequences
    styles.ts             -- 颜色、字体、设计 token
    components/
      PhoneMockup.tsx     -- iPhone 手机壳组件
      DashboardScreens.tsx  -- 产品截图/模拟页面
  public/
    vo/                   -- VO 音频文件
    *.mp3                 -- SFX 音效（tap, pop, boom, whoosh, chime）
    bgm.mp3              -- 背景音乐
    icon.jpg             -- 产品 icon
  out/
    PromoVideo.mp4        -- 渲染输出
```

---

## 8. 渲染与调试

### 8.1 快速测试

```bash
# 只渲染前 5 帧，验证构建
npx remotion render PromoVideo --frames=0-5

# 渲染指定片段（调试某个场景）
npx remotion render PromoVideo --frames=200-550
```

### 8.2 完整渲染

```bash
npx remotion render PromoVideo --gl=angle
```

### 8.3 多格式

在 Root.tsx 注册多个 Composition（9:16 + 16:9）：

```tsx
<Composition id="PromoVideo" width={1080} height={1920} ... />
<Composition id="PromoVideo-16x9" width={1920} height={1080} ... />
```

---

## 9. 迭代工作流

### 9.1 单变量迭代

每轮只改一个维度，渲染验证后再改下一个：

```
Round 1: 音量调整 -> 渲染 -> 确认
Round 2: 时间轴调整 -> 渲染 -> 确认
Round 3: 动画同步 -> 渲染 -> 确认
```

**不要**一次改音量+时间轴+动画——出问题时无法定位。

### 9.2 变更同步清单

改一个值时，检查所有引用它的地方：

| 改了什么 | 必须同步检查 |
|---------|------------|
| Scene from/duration | VO Sequence from、SFX at、FLASH_POINTS、scene envelope duration |
| VO playbackRate | VO durationInFrames、动画 delay、VO_RANGES |
| PhoneMockup scale | 字幕 marginBottom、周边元素间距 |
| 总时长 | Root.tsx DURATION_SECONDS、CTA finalFade |

### 9.3 用户反馈的优先级映射

| 用户反馈 | 本质问题 | 修什么 |
|---------|---------|-------|
| "音画不同步" | 动画 delay 和 VO 对不上 | ffmpeg 分析, 重算 delay |
| "太快了看不清" | 纯视觉场景太短 | 增加 durationInFrames |
| "配音后面空了一段" | VO 结束后场景还在跑 | 减少 durationInFrames |
| "BGM/SFX 太小" | 音量不够 | volume > 1.0，去掉 Math.min 限制 |
| "文字被遮住了" | 元素层叠冲突 | 减小 scale 或增加 margin |
| "最后太快了" | CTA 没有 hold 时间 | 延长场景 + 推迟 finalFade |
| "看不出重点" | Dashboard 没有视觉引导 | 添加 CalloutCard 浮动卡片 |

---

## 10. 反模式速查

| 反模式 | 后果 | 正确做法 |
|-------|------|---------|
| 用"估算"定动画 delay | 音画不同步 | ffmpeg silencedetect 分析 |
| Math.min(vol, 1.0) | BGM/SFX 永远太小声 | 允许 > 1.0 |
| Dashboard 场景 45 帧 | 用户来不及看 | 最少 70 帧，带 callout 建议 90 帧 |
| CTA 出完立即淡出 | 用户看不清 URL | hold 90+ 帧 |
| 一次改多个维度 | 无法定位问题 | 单变量迭代 |
| 改 Scene from 不改 SFX | SFX 错位到别的场景 | 全局同步清单 |
| VO Sequence 无 gap | 音频重叠杂音 | gap 6-8 帧 |
| Dashboard 整页缩放找重点 | 看不清到底突出什么 | CalloutCard 提取内容到浮动卡片 |
