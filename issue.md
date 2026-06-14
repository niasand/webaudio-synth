# NEXUS WebAudio 合成器 — 踩坑记录 (issue.md)

> 跨 session / 跨 Agent 的项目记忆。git log 记"改了什么"，这里记"为什么这么改"和"下次怎么避免"。

## [2026-06-13] CDP 自动化验证 WebAudio：autoplay policy 与 tab 漂移

**现象**：用 cdp-bridge 自动化验证 `synth.html` 时，AudioContext 始终 `suspended`，模拟 keydown/click 后 VU 表无反应（无声音）。

**根因**：
1. **autoplay policy**：CDP 通过 `Runtime.evaluate` 执行的 JS（`element.click()`、`dispatchEvent(KeyboardEvent)`）是 **untrusted**，不产生 "transient/sticky user activation"。因此 `AudioContext.resume()` 的 Promise 在无真实手势时长期 pending，ctx 保持 suspended。
2. **`await ctx.resume()` 阻塞**：最初 `AudioEngine.init()` 里 `await this.ctx.resume()`，因上述 pending 导致整个 init（及后续 `UI.build`）永不返回——表现为 `oscUnits=0`、`keys=0`，但 **无任何 JS 报错**（极具迷惑性）。
3. **CDP 多 tab 漂移**：环境有 17 个 tab，cdp-bridge 的活动 tab 不稳定（`active_tab:null`），trusted `Input.dispatchMouseEvent` 偶尔落到错误 tab，`window.__NEXUS` 时有时无。

**修复方案**：
- `init()` 中 resume 改 **fire-and-forget**（不 await）：`if(ctx.state!=='running') ctx.resume().catch(()=>{})`。
- 增加 `ensureRunning()`，在 `VoiceManager.noteOn` 入口调用——用户真实手势触发 noteOn 时顺便 resume。
- 暴露 `window.__NEXUS` 调试接口（无副作用），便于自动化读取内部 ctx 状态。
- 验证改用 **OfflineAudioContext 离线渲染**（不受 autoplay policy 限制），用真实 `DSP.makeIR` + 完整信号链渲染，检查 RMS/peak 非零，确证合成逻辑正确。

**涉及文件**：`synth.html`（`AudioEngine.init`/`ensureRunning`、`VoiceManager.noteOn`、调试接口）

**验证证据**：
- UI 全区域视觉正常（视觉模型逐区确认：顶栏/3OSC/滤波/双ADSR/LFO/6效果/可视化/音序器/键盘/弯音轮/XYPad，无渲染缺陷）。
- OfflineAudioContext 渲染：`rms=0.071`、`peak=0.25`、`hasAudio=true`、`distCurve=2048` 非零。
- `node --check` 语法通过；浏览器运行 `log:[]` 零错误。

**教训（通用铁律，不限于本 bug）**：
- **WebAudio 自动化验证不能依赖实时 AudioContext**——autoplay policy 让 CDP/无头环境的 ctx 永远 suspended。用 **OfflineAudioContext 离线渲染** 验证信号链，它绕过策略且能跑完整 DSP。
- **`await ctx.resume()` 是隐患**：在无手势环境会永久 pending，阻塞后续逻辑。Web Audio 初始化中 resume 应 fire-and-forget，真实激活留给用户交互入口（noteOn/click）。
- **CDP 多 tab 环境不可靠**：操作前先 `get_tabs` 拿明确 tab_id，所有 `execute_js/wait/batch` 都带 `switch_tab_id`/`tab_id` 锁定，否则活动 tab 漂移导致读错页面。
- **file:// 页面 cdp-bridge 注入失败**：扩展默认无 file:// 权限（注入脚本读不到 DOM），改用本地 HTTP server（`python3 -m http.server`）走 `http://localhost`。

## [2026-06-13] 自审发现的 5 处代码 bug（编写后立即修复）

**现象**：首版写完后自审发现逻辑缺陷，浏览器加载前修复。

1. `Voice` 的 LFO→pitch 路由：`lfoToPitch` 是 `build()` 内局部变量未存到 `this`，导致 `updateParam('lfo.toPitch')` 是坏占位代码 → 改为 `this.lfoToPitch` 成员。
2. `XYPad` 覆盖了 knob 的 `onChange`，丢掉 cutoff/reso 的 State 同步与 liveUpdate → 改为 wrap（保留原 onChange 再叠加 place）。
3. 虚拟键盘 `z`/`x` 既映射音符又切八度，冲突 → 移除键绑定的八度切换，八度走 UI 按钮。
4. `Visualizer` 初始 canvas 尺寸可能为 0 → draw 开头自适应 resize。
5. splash 提示文字与实际 QWERTY 映射不符 → 修正。

**教训**：大单文件写完务必做一次静态自审（grep 局部变量泄漏、onChange 覆盖、键映射冲突），配合 `node --check` 兜底语法。

## [2026-06-14] CSS grid 布局溢出覆盖：矮窗口下音序器盖住键盘栏

**现象**：用户报告"左下角 UI 被覆盖"。在标准尺寸（1600×980 等）肉眼看不出问题，视觉模型也判断正常。

**根因**：`.workspace`（grid 布局）没有 overflow 处理。窗口较矮时，网格内容总高度（实测 scrollH 765px）超过可用高度（~450px），默认 `overflow:visible` 让底部行（音序器的 step grid）**视觉溢出到下方的键盘栏区域**造成"覆盖"。注意：键盘与左侧八度/弯音轮其实从未重叠（DOM rect 一直没重叠），是音序器格子越界——极具迷惑性。

**修复**（2 行 CSS）：
- `.workspace { overflow:auto }`：内容超高时内部滚动，不溢出到键盘栏。
- seq 行 `grid-template-rows: auto auto minmax(190px,1fr)`：保证音序器最小高度，防 1fr 过度压缩。

**涉及文件**：`synth.html`（`.workspace` 规则）

**验证证据**：Playwright `getBoundingClientRect` 实测——1366×640 下 `.seq-grid` 底部从 592px（越界压住键盘栏 top 518px，+74px）→ 501px（留 17px 间隙）。1280×600 / 1600×980 同样不再重叠。

**教训（通用铁律）**：
- **"UI 被覆盖"类 bug，先用 `getBoundingClientRect` 量 rect 找越界元素，别只靠肉眼看/截图猜**——本案肉眼和视觉模型在标准尺寸都看不出，rect 才定位到 seq-grid 越界。
- **flex/grid 容器要显式设 overflow**：内容可能超高的布局区（工作台、面板网格），默认 `overflow:visible` 会让子内容溢出覆盖相邻 flex 项。用 `overflow:auto`（滚动）或 `hidden`（裁剪）兜底。
- **响应式要测极端尺寸**：标准尺寸正常的布局，在矮窗口（高 600–640）/窄窗口/移动端可能崩。验收用 Playwright 跑一组尺寸（1366×640、1280×600、375×812）截图 + 量 rect。
- **grid 行用 minmax 防 1fr 过度压缩**：纯 `1fr` 空间不足时压到 0、内容溢出；`minmax(minH, 1fr)` 保底。
