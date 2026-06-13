# NEXUS · WebAudio 合成器工作站

> 一个专业级、**零依赖、单文件**的网页音乐合成器工作站。纯原生 [Web Audio API](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Audio_API)，无任何框架 / CDN / npm 包——整个工作站就是一个 [`synth.html`](./synth.html)（约 1600 行，HTML+CSS+JS 全内联）。

浏览器打开即用，无需安装、无需联网。

![tech](https://img.shields.io/badge/WebAudio-native-3dd2c0) ![deps](https://img.shields.io/badge/dependencies-0-blue) ![file](https://img.shields.io/badge/single%20file-synth.html-orange)

---

## 特性

**合成引擎（减法合成）**
- 3 个振荡器（sine / square / sawtooth / triangle）+ 每振荡器 **Unison**（supersaw 厚度）
- Sub 振荡器 + 白噪音发生器
- BiquadFilter（lowpass / highpass / bandpass / notch），带共振
- 双 ADSR 包络：振幅 + 滤波器（含包络调制量）
- per-voice LFO，可路由到 **音高 / 截止频率 / 音量**
- glide（滑音）、pitch bend

**效果链**（串联）
- 失真（WaveShaper，4× 过采样降混叠）
- Bitcrusher（AudioWorklet 内联，零依赖）
- 合唱（短延迟 + LFO）
- 立体声延迟（反馈环）
- **程序生成 IR 的卷积混响**（无需下载 impulse response 文件）
- 三段 EQ + 主限幅

**工作站功能**
- 复音（最多 16 音）+ voice stealing
- 步进音序器（16/32 步，6 轨，swing，look-ahead 精确调度）
- 预设系统（localStorage 保存 + 5 个出厂预设 + JSON 导入导出）
- 实时可视化：示波器 + 频谱分析仪 + VU 表
- Web MIDI 输入（note / pitch bend）
- MediaRecorder 录音导出
- 自定义旋钮（拖拽 / 滚轮 / 双击重置 / Shift 精细）
- 虚拟键盘 + QWERTY 映射、XY Pad、弯音 / 调制轮

## 快速开始

直接双击 `synth.html`，或：

```bash
git clone https://github.com/niasand/webaudio-synth.git
open webaudio-synth/synth.html   # macOS
```

点「▸ 启动音频引擎」（浏览器策略要求一次用户手势），然后弹奏。

### 键盘映射

| 低八度 | `Z` `S` `X` `D` `C` `V` `G` `B` `H` `N` `J` `M` |
|--------|--------------------------------------------------|
| 高八度 | `Q` `2` `W` `3` `E` `R` `5` `T` `6` `Y` `7` `U` |
| 切八度 | 界面 `− / +` 按钮 |

> 接了 MIDI 键盘？直接弹，自动识别。

## 出厂预设

`INIT` · `WARM PAD` · `ACID LEAD` · `SUB BASS` · `PLUCK`

## 技术亮点

- **ConstantSource 加和调制**：滤波器截止频率由基础值 + 包络 + LFO 三个 ConstantSource 经 Gain 加和注入 `filter.frequency`，杜绝 AudioParam 自动值冲突（减法合成器最易踩的坑）。
- **零依赖混响**：用指数衰减噪声 + 立体声去相关 + 一阶低通，程序合成 impulse response 喂给 ConvolverNode。
- **爆音消除**：统一的 `ramp()` helper（`cancelScheduledValues` → 锚定 → `linearRamp`/`setTargetAtTime`），voice release 先平滑淡出再 `stop()`。
- **AudioWorklet 内联**：Bitcrusher 的 DSP 代码以 Blob URL 注入 `audioWorklet.addModule`，无需独立 .js 文件。
- **音序器 look-ahead 调度**：25ms 轮询、100ms 预读窗口，用 `ctx.currentTime + delta` 精确排程，避免 `setTimeout` 抖动。

## 文件

| 文件 | 说明 |
|------|------|
| `synth.html` | 工作站本体（唯一可运行产物） |
| `issue.md` | 开发踩坑记录（autoplay policy、AudioParam 冲突等） |
| `.mygoal/state.md` | 目标追踪与验证证据 |

## 兼容性

Chrome / Edge 桌面端一等公民（AudioWorklet、Web MIDI 全支持）。Safari / iOS 二档（MIDI 不可用，回退虚拟键盘）。

## License

[MIT](./LICENSE)
