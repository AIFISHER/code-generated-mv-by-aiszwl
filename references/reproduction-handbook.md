# 《达拉崩吧》纸艺 RPG 代码 MV —— 完全复现手册

> 读者：要从零复现这支动画的 AI agent（一个“编排者/集成负责人” + 若干“执行者”子 agent）。
> 参考实现：`~/dalabengba-mv`（能访问就以它的代码和 docs 为准；本手册解释它是怎么一步步做出来的、为什么这么做、每一步怎么验收、踩过哪些坑）。
> 原则：本手册不包含歌词原文（版权）。画面里也不写歌词原句，只写名字、短对白和界面字。

---

## 0. 速览

| 项 | 内容 |
|---|---|
| 成品 | ① 原版（ilem 词曲，洛天依/言和演唱，3:28）1080p30 有声 MP4；② 周深 Live 版（歌手·当打之年，4:05）1080p30 有声 MP4；③ 单文件 HTML 播放器（可载入任意版本音频、打点对齐、LRC 字幕） |
| 技术路线 | 纯 Canvas 2D 程序化绘制（不用任何图片/视频素材）；每一帧是时间 T 的纯函数；Playwright 驱动 Chrome 逐帧截图；ffmpeg 合成 ProRes → H.264 |
| 风格 | 纸艺立体绘本（剪纸片 + 投影 + 纸纹 + 多平面视差）× 复古 RPG 界面（对话框、取名框、HP、LV UP、回合制） |
| 规模 | 约 4.6 万行 JS（core 1.0k / env 10.7k / rigs 9.2k / props 5.7k / ui 3.3k / fx 1.1k / scenes 14.5k / tools 0.5k）+ 约 9500 行设计与 API 文档 |
| 多智能体 | 约 48 个子 agent：分镜设计 6、基础资产 12（+12 收尾）、接缝合同 1、场景块 17 |
| 耗时（墙钟） | 约 14 小时（含两次网关故障造成的整批中断与重启）；纯渲染每版 11–12 分钟 |
| Token | 子 agent 合计约 2000–2500 万 |

### 0.1 管线总图

```mermaid
flowchart LR
  A[取时间轴<br/>网易云 LRC → cue 表] --> B[引擎 + 工具链<br/>src/core, tools]
  B --> C[导演任务书<br/>brief.md]
  C --> D[分镜设计 Workflow<br/>3 提案→2 评审→1 合成]
  D --> E[基础资产 12 组并行<br/>rigs/env/props/ui/fx + API 文档]
  E --> F[接缝合同<br/>_seams.js + drawSeam]
  E --> G[场景块 17 并行<br/>src/scenes/bXX]
  F --> G
  G --> H[集成验收<br/>check / seams / pops / cues]
  H --> I[出片<br/>HTML + MP4 + 混音]
```

### 0.2 实际时间线（供估算）

| 阶段 | 墙钟 | 说明 |
|---|---|---|
| 环境探测、取 LRC、引擎核心、工具、自检 | 约 40 分钟 | 编排者自己做 |
| 分镜设计 Workflow（6 agent） | 约 5.2 小时 | 提案各 90–105 分钟，评审 25–30 分钟，合成 3 小时（太慢，见第 12 节建议） |
| 基础资产第一轮（12 agent） | 85 分钟后全部死于网关 401 | 文件都已写到盘上，只缺文档 |
| 基础资产收尾（12 agent） | 24–46 分钟 | 先写 API 文档，再补完中断的改动 |
| 接缝合同（1 agent） | 与场景并行，约 3 小时 | |
| 场景块（17 agent） | 2–2.6 小时 | 1 块正常完成，16 块在收尾阶段死于网关 403（预算不足）；代码可用 |
| 集成修复 + HTML 打包 | 15 分钟 | 修了 1 个半截块 |
| 渲染原版 MP4 + 混音 | 11 分钟 + 1 分钟 | |
| 周深版：找音频 + 校准偏移 + 渲染 + 混音 | 约 25 分钟 | |

---

## 1. 总体思路（为什么这么做）

1. **一切都是 T 的纯函数**。`draw(g, T)` 只由时间决定：禁止 `Math.random()`、`Date.now()`、跨帧缓存状态；随机用固定种子哈希。好处：可任意跳帧、多进程分段并行渲染、像素级对比接缝、运动模糊子帧采样、改一处不影响别处。
2. **默认时间轴 + 时间映射（warp）**。所有镜头只按“默认秒”（原版 3:28 的 cue 时刻）编排；播放其它版本时，用分段线性 warp 把真实音频时间换算成默认秒。于是同一套动画能对齐原版、周深版、任意翻唱（播放器支持现场打点生成 warp）。
3. **文档驱动的契约**。先写导演任务书 → 多智能体出分镜定稿（含每个地点写死的坐标、每块的入场/出场状态）→ 资产规格 → 实际 API 文档 → 场景作者规范 → 接缝合同。几十个并行 agent 互相看不到代码，只能靠这些文档对齐。
4. **分块并行 + 接缝合同**。全片切 17 块（8–18 秒），每块一个 agent。块与块之间的交接分两类：有重叠的转场（由后一块画）、无重叠的“隐形接缝”（两块都 import 同一份接缝合同，在接缝时刻画成同一个状态）。
5. **自动化验收优先**。lint（语法 + 中文乱码 + 非确定性 API）、块覆盖检查、跳变检测、接缝 A|REF|B 像素对比、逐 cue 联系表。agent 必须“亲眼看图”（Read PNG）迭代至少两轮。
6. **编排者 = 集成负责人**。编排者不画画，负责：基础设施、拍板冲突、维护 `integration_notes.md`、向运行中的 agent 广播补充信息（SendMessage）、最后的集成修复与出片。

---

## 2. 环境

| 依赖 | 版本/路径 | 用途 |
|---|---|---|
| macOS + Apple M1 Pro（8 核 16 GB） | — | 17 个并行 agent + 渲染可行，swap 会到 4–5 GB |
| Node | v26 | 工具脚本、esbuild |
| playwright-core | ^1.63 | 驱动本机 Chrome |
| Chrome for Testing | `~/Library/Caches/ms-playwright/chromium-1234/chrome-mac-arm64/Google Chrome for Testing.app/Contents/MacOS/Google Chrome for Testing` | 无头渲染（`--use-angle=metal --ignore-gpu-blocklist`） |
| ffmpeg / ffprobe | 8.x | ProRes 分段、拼接、x264、混音、抽帧 |
| Python 3 + fonttools + brotli | `pip3 install --user fonttools brotli` | 字体子集化成 woff2（单文件 HTML） |
| esbuild | ^0.28（devDependency） | 打包单文件 HTML |
| yt-dlp | 可选 | 实测 B 站会 412，最后改用 B 站 API + curl |

```bash
mkdir -p ~/dalabengba-mv/{src/core,src/rigs,src/env,src/props,src/ui,src/fx,src/scenes,src/sheets,tools,assets/fonts,assets/audio,docs/{design,api,blocks,review},out,dist}
cd ~/dalabengba-mv && npm init -y && npm pkg set type=module
npm install playwright-core && npm install -D esbuild
```

### 2.1 字体（全部 OFL，google/fonts 仓库）

| 字体 | 用途 | 下载路径（`https://raw.githubusercontent.com/google/fonts/main/` + …） |
|---|---|---|
| 站酷庆科黄油体 ZCOOL QingKe HuangYou | 名字大字、标题（display） | `ofl/zcoolqingkehuangyou/ZCOOLQingKeHuangYou-Regular.ttf` |
| 站酷快乐体 ZCOOL KuaiLe | 对白、界面（play） | `ofl/zcoolkuaile/ZCOOLKuaiLe-Regular.ttf` |
| 站酷小薇 ZCOOL XiaoWei | 绘本旁白、片尾（serif） | `ofl/zcoolxiaowei/ZCOOLXiaoWei-Regular.ttf` |
| Fredoka 可变字重 300–700 | 数字、LV、HP（latin） | `ofl/fredoka/Fredoka%5Bwdth,wght%5D.ttf`（存为 Fredoka-Variable.ttf） |

下载后用 fontTools 检查四个长名字的每个字都在 cmap 里（三套中文字体全覆盖）。**四套字体都没有 ★ ♥ ♪ ✓ ✕ ▼ ▶ ①–④ 与箭头**，这些符号一律用路径画（`ui/kit.js` 的 `glyphIcon`）。「·」只有快乐体和 Fredoka 有。

---

## 3. 第一步：时间轴（cue 表）

### 3.1 取 LRC（网易云公开 API）

```bash
# 搜索版本与时长
curl -s -A "Mozilla/5.0" -H "Referer: https://music.163.com/" \
  "https://music.163.com/api/search/get?s=达拉崩吧&type=1&limit=20"
# 取歌词（带时间戳）
curl -s -A "Mozilla/5.0" -H "Referer: https://music.163.com/" \
  "https://music.163.com/api/song/lyric?id=<ID>&lv=1&tv=-1"
```

| 版本 | 网易云 id | 时长 | 用途 |
|---|---|---|---|
| 原版 ilem / 洛天依 / 言和 | 521493845 | 208.1 s | 默认时间轴（LRC 逐句时间戳） |
| 原版（ilem 2017–2018 作品合集里的同一录音） | 3392512852 | 208.07 s | 原版音频来源（521493845 是会员歌曲） |
| 周深 Live（歌手·当打之年 第 8 期） | 1434062381 | 245.6 s | 周深版 warp 锚点（LRC 只有逐行时间戳） |

### 3.2 cue 设计规则

- 每句歌词一个 cue，id 是“段落字母 + 序号”，标签写**画面提示**（自己的话，不抄歌词）。
- 长名字按演唱分段拆成 a/b/c：勇者名 `N1a/N1b/N1c`（达拉崩吧 / 斑得贝迪 / 卜多比鲁翁）× 4 遍；龙名 `D1a/D1b`（昆图库塔卡提考特 / 苏瓦西拉松）× 3 遍。
- 纯音乐段单独一个 cue（`intro`、`I1` 间奏一、`I2` 间奏二、`OUT` 尾奏），段内节点用 `cue('I1') + 偏移`。
- 节拍：BPM≈120 只是估算，**没有实测音频时不做逐拍卡点，只卡 cue**。

完整 cue 表（`src/cues.js` 的 `CUE_LIST`，[id, 默认秒, 画面提示]）：

```js
['intro', 0.0, '前奏'],
['L01', 10.39, '很久以前的王国'], ['L02', 12.11, '巨龙突然现身'], ['L03', 14.18, '灾祸降临'], ['L04', 15.41, '掳走公主后消失'],
['L05', 18.0, '王国告急'], ['L06', 20.08, '寻找最勇敢的人'], ['L07', 22.11, '勇者赶到'], ['L08', 24.11, '勇者高喊'],
['L09', 25.35, '誓言：最好的剑'], ['L10', 28.17, '誓言：最高的山'], ['L11', 30.12, '誓言：最深的森林'], ['L12', 32.03, '誓言：带回公主'],
['L13', 34.0, '国王大喜'], ['L14', 36.11, '国王问名字'], ['L15', 38.11, '勇者想了想'], ['L16', 39.91, '开口报名'],
['N1a', 41.9], ['N1b', 42.84], ['N1c', 43.8], ['K1', 45.1, '国王要他再说一遍'],
['N2a', 45.94], ['N2b', 46.76], ['N2c', 47.71], ['K2', 49.31, '国王试着复述'],
['N3a', 49.83], ['N3b', 50.57], ['N3c', 51.77], ['H1', 53.18, '勇者：对！'],
['N4a', 54.0], ['N4b', 54.68], ['N4c', 55.71], ['I1', 56.89, '间奏一'],
['V01', 74.4, '英雄登场'], ['V02', 75.99, '跨上最快的马'], ['V03', 78.14, '带着众人的希望'], ['V04', 80.22, '出城'],
['V05', 82.04, '打怪'], ['V06', 83.83, '金币 ×12'], ['V07', 86.01, '伤痕累累'], ['V08', 88.01, '慢慢升级'],
['V09', 90.15, '偏远的美丽村庄'], ['V10', 91.97, '开遍宝箱'], ['V11', 94.04, '风霜一路'], ['V12', 95.85, '圣月光指路'],
['V13', 98.04, '闯进山洞'], ['V14', 99.97, '公主与巨龙'], ['V15', 102.1, '拔剑'], ['V16', 104.16, '巨龙开口'],
['D1a', 105.36], ['D1b', 107.89], ['Q1', 109.15, '勇者要它再说一遍'], ['D2a', 109.76], ['D2b', 111.69],
['Q2', 113.31, '勇者试着复述'], ['M1', 113.83, '听岔①：昆特牌 · 提琴'], ['M2', 115.21, '听岔②：烤蛋挞 · 苏打'], ['M3', 116.21, '听岔③：马拉松'],
['R1', 117.03, '巨龙：不对！'], ['D3a', 117.66], ['D3b', 119.65], ['I2', 120.97, '间奏二（大战）'],
['B01', 137.92, '于是——'], ['B02', 138.24, '勇者全名（出招）'], ['B03', 141.28, '砍向'], ['B04', 141.74, '龙全名（中招）'],
['B05', 145.23, '然后'], ['B06', 146.73, '龙全名（反扑）'], ['B07', 149.33, '咬'], ['B08', 150.52, '勇者全名（被咬）'],
['B09', 153.46, '最后'], ['B10', 153.78, '勇者全名（绝招）'], ['B11', 157.17, '战胜'], ['B12', 157.8, '龙全名（落败）'],
['E01', 161.13, '救出'], ['E02', 161.65, '公主全名'], ['E03', 165.08, '回到'], ['E04', 165.6, '城名'],
['E05', 168.83, '国王听说'], ['E06', 169.74, '勇者全名（捷报）'], ['E07', 173.05, '打败了'], ['E08', 173.82, '龙全名（捷报）'],
['E09', 177.0, '国王把——'], ['E10', 177.76, '公主全名（婚礼）'], ['E11', 180.99, '许配给'], ['E12', 181.66, '勇者全名（婚礼）'],
['O01', 185.47, '啦啦：两个短名字'], ['O02', 187.75, '童话般的幸福'], ['O03', 189.75, '有了孩子'], ['O04', 191.66, '孩子一天天长大'],
['O05', 193.73, '为免以后麻烦'], ['O06', 195.68, '孩子就叫王浩然'], ['O07', 197.77, '全名太难念'], ['O08', 199.66, '不想念一遍'],
['OUT', 201.6, '尾奏'], ['END', 208.1, '结束'],
```

可以写进画面的名字（`NAMES`）：勇者「达拉崩吧斑得贝迪卜多比鲁翁」、龙「昆图库塔卡提考特苏瓦西拉松」、公主「米娅莫拉苏娜丹妮谢莉红」（短名米娅）、城「蒙达鲁克硫斯伯古比奇巴勒城」、孩子「王浩然」。

**逐字时刻 NAME_TIMES**：段首字落在该段 cue，步长 = 段长 ÷ 字数（例如 N1a 0.235 s/字）；N4 印章段整段同时出现；没有分段 cue 的句子（名字之剑 B10、公主全名 E02/E10、城名 E04、勇者全名 E12、王浩然 O06）在分镜附录里写死首字时刻与步长。代码里一律调用 `NAME_TIMES('N1a')`，不手抄秒数。

### 3.3 warp（多版本对齐）

```js
// 锚点 {cueId: 该版本里的真实秒} → 真实秒 ↔ 默认秒，分段线性，两端斜率 1 外推
export function makeWarp(anchors = {}) {
  const pts = Object.entries(anchors).filter(([id]) => CUES[id] !== undefined)
    .map(([id, real]) => [real, CUES[id]]).sort((a, b) => a[0] - b[0]);
  if (!pts.length) return { toDefault: (t) => t, toReal: (T) => T, duration: DURATION };
  const map = (x, from, to) => {
    if (x <= pts[0][from]) return pts[0][to] + (x - pts[0][from]);
    for (let i = 1; i < pts.length; i++) {
      const a = pts[i - 1], b = pts[i];
      if (x <= b[from]) return a[to] + ((x - a[from]) / (b[from] - a[from] || 1)) * (b[to] - a[to]);
    }
    const z = pts[pts.length - 1];
    return z[to] + (x - z[from]);
  };
  return { toDefault: (t) => map(t, 0, 1), toReal: (T) => map(T, 1, 0), duration: map(DURATION, 1, 0) };
}
```

周深 Live 预设锚点（由其 LRC 逐行时间戳对应到 cue）：

```js
zhoushen: { intro: 0, L01: 19.43, L03: 23.48, L05: 27.28, L07: 31.22, L09: 34.7, L11: 39.45, L13: 43.03, L15: 47.42, L16: 50.4,
  K1: 54.55, N2a: 55.5, K2: 58.43, H1: 62.53, I1: 66.9, V01: 85.8, V03: 89.37, V05: 93.42, V07: 97.5, V09: 101.2,
  V11: 105.41, V13: 109.24, V15: 113.51, D1a: 116.75, Q1: 120.51, D2a: 121.31, Q2: 124.46, R1: 128.65, I2: 132.4,
  B01: 166.83, B02: 167.98, B03: 172.1, B04: 172.86, B05: 176.63, B06: 177.4, B07: 181.4, B08: 182.25, B09: 185.36,
  B10: 185.85, B11: 189.11, B12: 189.75, E01: 193.03, E03: 196.66, E05: 200.5, E06: 201.4, E07: 203.99, E08: 204.42,
  E09: 207.25, E11: 210.67, E12: 211.07, O01: 214.1, O02: 216.2, O03: 217.75, O05: 220.97, O07: 224.25, OUT: 228.5, END: 245.6 }
```

注意：Live 版大战间奏（I2→B01）有 34 秒，是原版的 2 倍，所以分镜要求纯音乐段的动作链“停在清楚的姿态上留保持、慢动作不低于 0.5 倍速”。

---

## 4. 第二步：引擎（`src/core`，约 1000 行）

### 4.1 文件

```
index.html            加载 /src/main.js（?capture=1 逐帧模式；?sheet=名 只渲染 src/sheets/名.js；?only=b05 只加载某块；?preset=zhoushen；?nograin=1；?stamp=1）
src/main.js           加载字体 → 建引擎 → window.renderFrame(t) / window.duration / window.engine；非 capture 时启动播放器
src/story.js          await loadBlocks() 汇总 17 块的镜头；cuts: [18.0]
src/scenes/index.js   逐块动态 import（某块语法错只让那一块空着，控制台报 [block load failed]）
src/core/engine.js    调度、图层、遮罩、后期、子帧运动模糊、renderProbe
src/core/paper.js     PAL 调色板 + 纸艺路径/填充/渐变/光效
src/core/camera.js    多平面视差机位、震动、漂移
src/core/particles.js 无状态粒子
src/core/texture.js   纸纹（底层 multiply 用、图层 source-atop 用）、胶片颗粒
src/core/fonts.js     FontFace 注册 + 预热（canvas 首次用到某字体会先回退一帧）
src/core/ease.js      缓动（outBack/outExpo/spring/glide/whip/snap/brand/ez(t,a,b,fn)）
src/core/util.js      clamp/lerp/seg/envelope/pulse/hit/hits/wobble/osc/squash/stepIndex/hash1/hash2/mulberry32/noise1/fbm1/rgba/mixHex
```

### 4.2 镜头（shot）约定

```js
// src/scenes/b05_names.js 默认导出镜头数组
export default [{
  id: 'b05_n1',                     // 块号前缀，全片唯一
  start: cue('N1a'), end: cue('K1'),// 默认秒，激活区间 [start, end)；相邻镜头可重叠做转场
  z: 0,                             // 可选，越大越上（转场镜头 z:1）
  draw(g, T, lt, ctx) { /* g 已是逻辑 1920×1080 坐标；T 默认秒；lt = T - start */ },
}];
```

- 每个镜头自己画满整屏；多镜头同时激活按 z、再按全片顺序叠加。
- **转场由后一个镜头负责画**（后镜头 start 早于前镜头 end，重叠期内用 `ctx.mask` 揭示自己或画擦屏物；前镜头照常画）。

### 4.3 ctx（每帧）

| 字段 | 说明 |
|---|---|
| `ctx.layer(g, o, fn)` | 离屏图层：fn(lg) 在新画布上画，再整体合成回 g。`o`: `shadow`(数字=纸片高度，自动换算 blur/dx/dy) / `texture`(纸纹强度，source-atop 只作用于有内容处) / `texOffset` / `tint:{color,alpha,mode}` / `alpha` / `blend` / `blur` / `sat` / `bright` / `contrast` / `filter` |
| `ctx.mask(g, maskFn, drawFn, o)` | 遮罩：drawFn 画内容，maskFn 只 fill 保留区域（destination-in）；`feather` 羽化、`invert` 挖洞 |
| `ctx.fx` | 整帧后期（每帧复位）：`shake:[x,y]`、`zoom`、`rot`、`sat/bright/contrast/hue/blur`、`tint/tintAlpha/tintMode`、`flash/flashColor`、`fade/fadeColor`、`paper 0.22`、`vignette 0.3`、`grain 0.045`、`letterbox 0..1(2.39:1)` |
| `ctx.paperPattern(g)` | 纸纹图案 |

### 4.4 渲染管线（engine.render）

```
render(t):  T = warp.toDefault(t)
  samples==1:  renderOnce(T) = drawShots(T) → compose(fx)          → finish(fx)
  samples>1 :  在快门区间内取 n 个子帧（夹在当前硬切段内，不跨 story.cuts），逐个 renderOnce 后按 1/(i+1) 累加 → finish
drawShots:  底色 → 活跃镜头按 z 排序依次 draw（每个镜头 save/restore；报错画红条不中断）→ story.post
compose:    场景缓冲 → 输出：机位（震动/推近/滚转，自动放大避免露边）+ CSS filter 调色 + 叠色 + 闪白（screen）
finish:     纸纹 multiply → 暗角 multiply → 胶片颗粒 overlay（每秒换 12 次，4 张预生成）→ 淡出 → 宽银幕黑边 → 时间戳
renderProbe(T, draw):  只用给定 draw 渲染一帧（接缝验收用）
```

图层池：离屏画布按需扩容、用完归还；每个镜头结束检查池子是否泄漏。性能预算：每帧全屏缓冲（layer + mask）≤ 8，含重叠期两块相加；不要逐小形状 `shadowBlur`。

### 4.5 多平面视差机位

```js
export const cam = (x = 960, y = 540, zoom = 1, rot = 0) => ({ x, y, zoom, rot });
// depth：1 = 主体层；0.2–0.6 远景；1.2–1.6 前景（配 blur 3–5）
export function applyCam(g, c, depth = 1) {
  const z = 1 + (c.zoom - 1) * depth;
  g.translate(960, 540); if (c.rot) g.rotate(c.rot); g.scale(z, z);
  g.translate(-(960 + (c.x - 960) * depth), -(540 + (c.y - 540) * depth));
}
// toScreen(c, depth, x, y) 世界→屏幕；lerpCam(a,b,t)（zoom 按几何插值）；shake(T, at, {amp,decay,freq,seed})；drift(T,{amp,speed,seed})
```

### 4.6 纸艺原语（`paper.js`）

- 路径：`rr` 圆角矩形；`blob(cx,cy,rx,ry,{seed,amp,n,rot,freq})` 手剪椭圆（5 个正弦叠加的稳定边缘起伏）；`poly(points,{seed,amp,step,round})` 手剪多边形（直边细分 + 法向抖动）；`smooth(points)` Catmull-Rom → Bezier；`ribbon(points, widths)` 粗细变化条带（围巾、火舌、头发）。
- 填充：`cut(g, path, fill, {shadow, stroke, rim, rimW, alpha, blend})`（纸片投影 + 顶边切口亮边）；`shade(g, path, color, x0,y0,x1,y1, a0,a1)`（形状内明暗渐变）。
- 光效：`glow`（screen 柔光斑）、`rays`（放射光芒）、`sparkle`（四角星）、`line`（手绘线、虚线、生长）。
- 调色板 PAL（全片只用这些色，深浅用 mixHex/rgba）：

```js
paper '#F6EDDA' paper2 '#EADCBF' kraft '#D5BC8E' kraftDark '#B39565' ink '#2C2233' inkSoft '#56495E' shadow '#2B1A24' white '#FFFDF7'
red '#D8473D' redDark '#9F2C2C' redDeep '#6E1F24' gold '#F2B634' goldDark '#C98A1B' goldLight '#FFE28E'
skyDay '#8FD0F2' skyDayLow '#DDF1F7' skyDusk '#F4A26F' skyDuskHigh '#7E5C9F' skyNight '#28325F' skyNightHigh '#131937' skyDoom '#6B2B4A' skyDoomLow '#E0674A'
meadow '#93CF72' grass '#62AE5E' grassDark '#3E8A4F' forest '#2F6B52' forestDeep '#1E4A3C' leafLight '#B9E08A'
earth '#B57D4C' wood '#8C5B3C' woodDark '#5F3C28' sand '#E6C892' path '#E9D3A4'
stone '#CFC6B6' stone2 '#ABA192' stoneDark '#71685D' roof '#4E6FA8' roofRed '#C3533F'
water '#58AACB' waterDeep '#2F6F99' snow '#F8FBFF' ice '#C3E5F3' snowShade '#B9CFE6' mountain '#7C8FB8' mountainFar '#A9B8D6' mountainDark '#4F5F86'
skin '#F8CDA6' skinShade '#E7A985' blush '#F2918C' heroBlue '#3F70B8' heroBlueDark '#2B4F87' scarf '#E2493E' scarfDark '#A9302D' heroHair '#7B4A2B' boot '#6A4431'
steel '#E0E9EF' steelDark '#9BADB9' leather '#9B6A42' princess '#F39DB8' princessDark '#D86C94' princessLight '#FFD3E0' hairGold '#F5C744' hairGoldDark '#D39A22'
robe '#C8413B' robeDark '#8E2A2A' ermine '#FFF8EC' beard '#F6F2EA' horse '#FAF6EE' horseShade '#DCD3C4' mane '#E9C46A'
dragon '#6C4E9C' dragonDark '#473271' dragonDeep '#2E2050' dragonBelly '#72D3C3' dragonWing '#8E6CC0' dragonEye '#FFD24A'
fire '#FF8B3D' fire2 '#FFD65E' fireDeep '#E2462B' ember '#FFB347' moon '#FFF5D8' moonGlow '#FFE7A3' magic '#C2AEFF' crystal '#7FE3E0'
slime '#8FDB6B' slimeDark '#4FA24B' bat '#5A4A7A' coin '#F7C443' coinDark '#D0922A' heart '#F25C6E'
```

### 4.7 无状态粒子（`particles.js`）

- `burst(g,T,{at,seed,count,x,y,spread,speed,angle,gravity,drag,life,spin,size}, draw)`：一次性爆发。位置解析解：`k = (1-e^{-drag·age})/drag`，`p = p0 + v·k + ½·g·age²`。
- `stream(g,T,{start,end,rate,emit(tBorn,i,r)}, draw)`：持续发射，只遍历仍存活的出生序号区间。
- `field(g,T,{count,x,y,w,h,vx,vy,sway})`：循环飘落（雪、花瓣、萤火），`fract(r + v·T/w)·w`。
- 所有随机量 = `hash2(seed·97 + k·13.37, i)`，所以任何时刻都能直接算出来。

### 4.8 纹理

- 底层纸纹：白底 + 70 个低频斑驳（3×3 平铺绘制保证无缝）+ 900 根纤维短线 + 细颗粒，整帧 multiply 0.22。
- 图层纸纹：透明底的暗/亮纤维与斑驳，用 source-atop 只叠在有内容的地方（**最初用 multiply 整层叠，把整个画面压灰了，这是第一个视觉 bug**）。
- 胶片颗粒：50% 灰为中性的噪声，overlay 0.045，每秒换 12 次。

---

## 5. 第三步：工具链（`tools/`）

| 工具 | 命令 | 作用 / 实现要点 |
|---|---|---|
| `server.mjs` | — | 本地静态服务（no-store，mime 含 ttf/woff2/mp3） |
| `chrome.mjs` | — | 导出 CHROME 路径（单独成文件，避免 import render.mjs 时把它当脚本执行） |
| `render.mjs stills` | `--only b05 --times 42,43.1 --outdir out/review/b05` | 全分辨率 PNG |
| `render.mjs sheet` | `--only b05 --from 41.8 --to 57 --every 0.25 --cols 8 --tile 300 --out …jpg` | 联系表（带时间戳与激活镜头 id） |
| `render.mjs cues` | `--from 0 --to 208.1 --cols 8 --tile 320 --out …jpg` | 每个 cue 后 0.12 s 一帧，左上印 cue id + 画面提示：**逐句检查“这一句的画面”最有用的图** |
| `render.mjs video` | `--fps 30 --dpr 2 --workers 5 [--preset zhoushen] [--prores] [--audio x.mp3] --out …mp4` | N 个 Chrome 各渲一段：renderFrame(t) → JPEG q95 截图 → ffmpeg image2pipe → ProRes 422HQ 分段 → concat → x264 slow CRF16 → mp4。1080p 2× 超采样，3:28 约 11 分钟 |
| `check.mjs sheets / blocks` | `blocks --list b05 --samples 12` | 一个浏览器依次加载每张模型图/每个块，在若干时刻渲染，汇总页面报错与空镜（NO SHOT） |
| `pops.mjs` | `--only b05 --from 41.9 --to 56.9` | 低分辨率逐帧渲染（关颗粒），相邻帧差异突增且不在计划硬切处 → JUMP |
| `seams.mjs` | `--list "b04|b05"` | 隐形接缝验收：A=全片 T−1/30、REF=drawSeam 参考、B=全片 T，输出平均像素差与 A|REF|B 对照图 |
| `build.mjs` | `node tools/build.mjs` | 收集 src 所有用字 → pyftsubset 子集化 4 套字体为 woff2 → base64 内嵌；静态 import 17 块；esbuild 打包；拼成单 HTML（约 3 MB） |
| `lint.sh` | `./tools/lint.sh src/scenes/b05*` | `node --check` + **U+FFFD 乱码检查** + 禁用 Math.random( / Date.now( / new Date(（播放器除外）；可只查指定文件 |

注意：Playwright `page.evaluate` 传字符串函数不会带参数调用，要传真正的函数。

---

## 6. 第四步：导演任务书（`docs/brief.md`）

编排者在设计阶段前写好，作为所有 agent 的最高准则。要点：

1. **硬性约束**：纯代码；画面严格跟歌词 cue（关键动作误差 ≤0.15 s）；不内嵌歌词原文（只允许四个长名字 + 王浩然、≤5 字短对白/拟声、自拟界面字、听岔五名词、片尾署名）；确定性；分镜连贯（不许无动机跳切和交叉溶解）。
2. **风格**：世界是一本会动的纸艺立体书（平涂色块、手剪边、纸片投影、纸纤维、顶边亮边、多平面视差）；RPG 是界面语言（全部做成奶油色卡纸 + 墨线 + 缝线 + 角饰，不是像素风）；精致 = 配色克制、焦点明确、光有方向（左上主光，右下投影）、运动全缓动、无死帧、守安全边距（左右 96、上下 72）。
3. **角色**（Q 版约 2.6 头身，剪纸木偶关节）：勇者（栗发呆毛、超长红围巾 = 长名字的视觉隐喻、蓝短袍、背最好的剑）；国王（矮圆、过大金王冠会滑落、白胡子、红袍貂皮领）；公主米娅（金色麻花辫、粉裙、机灵会自救）；巨龙（紫色长蛇形、**身体 13 节、背板上逐字亮出名字**、爱面子蠢萌）；白马；市民/卫兵/书记官/传令官/史莱姆/蝙蝠；孩子王浩然。
4. **世界地理**：王城 → 草原 → 村庄 → 风雪山口 → 月光山岗 → 龙山山洞；去程左→右、归程右→左；**地图 + 红色虚线**是全片导航道具。
5. **色彩剧本**：按段落给主色（封面深红烫金 → 黎明 → 危急紫红 → 灰蓝告急 → 牛皮纸蜡笔 → 王座厅红金 → 草绿 → 金色午后 → 冷蓝雪 → 深蓝月夜 → 茄紫山洞 → 火焰橙 vs 冷紫 → 剧场红丝绒 → 粉金黎明 → 春日粉彩 → 金色夕阳 → 星空 → 合书）。
6. **故事节拍**：逐段写“画面必须表达的内容”（不是字幕）。
7. **运动原则**：缓动、预备、跟随余振、错峰 30–80 ms、机位永远有漂移、冲击帧（闪白 + 放射线 + 震动）、无状态粒子。
8. **连贯手段**：绘本开合、地图红线、屏幕方向、红围巾线索、形状匹配剪辑（月亮↔金币、洞口↔龙眼…）、统一角色比例、有动机的转场。

---

## 7. 第五步：分镜设计（多智能体 Workflow，6 agent）

### 7.1 编排

```
Propose（并行 3）：同一任务、三种立场
  A 摄影/连贯派：全片当一条连续的纸上长镜头；转场动机、交接状态、视差层次、颜色桥
  B 喜剧/动态字体派：名字太长的喜剧；每遍名字的排版、表演、RPG 梗、逐遍升级
  C 制作设计/技术派：可实现性与复用；固定布局坐标、动作/资产清单、分包建议
Judge（并行 2）：
  评审1 导演/观众视角：歌词贴合、连贯、笑点、情绪曲线、高潮华丽度
  评审2 制作/一致性视角：Canvas 2D 能否做精致、复用、风格统一、分块交接、性能
  每份提案 5 维度打分；列 12–20 个最佳点子、8–12 个必须避免的问题、定稿骨架建议
Synthesize（1）：总导演按评审合成定稿，产出 storyboard.md、assets.md、blocks.json
```

结论（两份评审一致）：**戏用 B，连贯用 A，施工用 C**。

### 7.2 定稿 storyboard.md 的结构（约 1050 行）

0. 定稿说明：来源表、拍板的取舍表（例：告急在阳台、书的开场不做俯仰转正、N3 气球字字序正确、N4 金印卡在 cue 上、间奏一每一步由上一步的物件变来、旅途一镜到底拆两块接缝藏在大橡树后、D3 石碑墙、大战留在洞内五拍、婚礼阳台外景只绕主堡尖塔一圈、尾声复用王座厅夜景、名字之剑向右横长再竖起劈下、取名面板 O06 收拢到 3 格）。
1. 视觉主张；2. 色彩剧本（段落 × 主色 × 光与后期 × 颜色桥）。
3. 全片规则：坐标与机位、角色标准比例（勇者 s=1 高 260，中景 zoom 1.6 ≈ 420，特写 zoom 3.4 头 ≈ 360）、景别、反应格（国王格 CU-K / 勇者格 CU-H 构图固定，K1↔Q1、K2↔Q2 左右镜像）、冲击分级（全片最大一次在 B11）、方向规则（名字一律从左往右读，龙头在左）、字形规则、**字幕带**（有人声段落 y ≥ 920 只放装饰）、计时铁律、转场铁律、性能预算。
4. 各地点固定布局（坐标写死）：BOOK、CASTLE（13 座塔 x = 300 + 110·i、圆窗 (960,430) r70、阳台…）、PLAZA、MAP（三形态共用一组 (u,v)）、THRONE（王座 x=1580、玫瑰窗 (1580,250) r150…）、JOURNEY（区间与遮挡物时刻、journeyCam 关键帧）、CAVE（鸟笼吊点 (1320,0)、STANDOFF 构图）、THEATER、片名页。
5. 17 个制作块：每块写文件名、范围、cue、一句话、入场状态、逐镜头分镜（精确到 cue 与名字分段）、出场状态、交接方式。
6. 名字梗总表；7. 贯穿道具与伏笔；8. cue 覆盖检查表；9. 块间交接一览 + 共享函数表；10. 风险与降级；附录 A NAME_TIMES。

### 7.3 17 块与交接

| 块 | 文件 | 默认秒 | cue | 入场交接 |
|---|---|---|---|---|
| b01 | b01_book.js | [0, 10) | intro | 片头 |
| b02 | b02_dragon.js | [9.5, 20.1) | L01–L05 | ←b01 重叠 9.50–10.00 天幕卡矩形遮罩（逐像素一致）；块内 18.00 唯一硬切 |
| b03 | b03_oath.js | [19.95, 34.1) | L06–L12 | ←b02 竖向甩镜帘 19.95–20.10 |
| b04 | b04_ask.js | [33.6, 41.9) | L13–L16 | ←b03 心形遮罩 33.60–34.10 |
| b05 | b05_names.js | [41.9, 56.89) | N1a…N4c | ←b04 隐形接缝 41.90 |
| b06 | b06_title.js | [56.89, 74.45) | I1 | ←b05 隐形接缝 56.89 |
| b07 | b07_journey1.js | [74.1, 90.0) | V01–V08 | ←b06 红虚线合拢成围巾 74.10–74.45 |
| b08 | b08_journey2.js | [90.0, 99.6) | V09–V13 | ←b07 隐形接缝 90.00（大橡树后） |
| b09 | b09_cave.js | [98.6, 109.15) | V14–D1b | ←b08 杏仁洞口遮罩 98.60–99.60 |
| b10 | b10_dragonname.js | [109.0, 120.97) | Q1–D3b | ←b09 横向甩镜帘 109.00–109.15 |
| b11 | b11_battle.js | [120.97, 137.92) | I2 | ←b10 隐形接缝 120.97 |
| b12 | b12_theater1.js | [137.4, 150.52) | B01–B07 | ←b11 黑边长成台口 137.40–137.92 |
| b13 | b13_namesword.js | [150.52, 165.35) | B08–E02 | ←b12 隐形接缝 150.52 |
| b14 | b14_return.js | [165.0, 177.2) | E03–E08 | ←b13 巨大围巾擦屏 165.00–165.35 |
| b15 | b15_wedding.js | [176.8, 185.47) | E09–E12 | ←b14 玫瑰窗内外匹配 176.80–177.20 |
| b16 | b16_family.js | [185.47, 197.77) | O01–O06 | ←b15 隐形接缝 185.47 |
| b17 | b17_close.js | [197.77, 208.1) | O07–END | ←b16 隐形接缝 197.77；208.10 关灯淡出 |

三大高潮：间奏一片头（56.89–74.40）、名字之剑（153.46–157.80）、婚礼（176.80–185.47）。

### 7.4 名字梗（逐遍升级，每遍 c 段是爆点）

- 勇者名 ×4：N1 撑破对话框、「翁」撞上镜头 → N2 挤爆吊线 8 格取名面板（「字数超限！」13/8，崩断）→ N3 国王吹出气球字把自己吊起 → N4 金印三连砸在 cue 上，三行印章字越拉越大，名签更新为全名。
- 龙名 ×3：D1 龙身 13 节背板逐字点亮 → D2 名字变火焰字连珠炮（点着勇者呆毛）→ 听岔做成 RPG 道具栏（昆特牌·提琴 / 烤蛋挞·苏打 / 马拉松五星传说级），R1 炸刺烧掉 → D3 石碑字砸成两层墙，I2 巨龙撞穿自己的名字开打。
- 纸剧场：名字牌窗口 5 字，全名走马灯，进度 = 本 cue 已过时间 / 本 cue 时长。
- 名字之剑：13 字从名字牌剥下横拼成剑刃，向右长、捅破侧幕、竖起撕开顶幕、劈下（全片最大冲击）；龙名字牌逐字弹飞一节龙身，缩成小蜥蜴。
- 城名：开场就立着的 13 座塔，E04 一旗一字。婚礼：公主竖幅变舞龙，勇者横幅由纸鸽从左往右拉开、绕尖塔系蝴蝶结（城堡包成礼物）。
- 王浩然：N2 那块 8 格面板原样降下，全场严阵以待 → 三个字刚好装下 → 空格折起收拢 → 盖对勾章 → 全体泄气，王冠自己回正 → 孩子一拽「全名」小签，射出无限长卷轴 → 讲故事的手合上书。

---

## 8. 第六步：基础资产（12 组并行）

### 8.1 分组

| 组 | 文件 | 内容 |
|---|---|---|
| ui | ui/type.js、ui/kit.js | 剪纸字（paperGlyph/glyphRow/typeOn/glyphsOnPath/glyph13/marquee/NAME_TIMES/titleGlyphs/scrollInk、14 种 style）；纸艺 RPG 界面（glyphIcon 17 个符号、对话框、气泡、取名面板、HUD、经验条、LV UP、金币计数、名字牌走马灯、战斗菜单、回合横幅、伤害数字、道具栏、角色卡…，全部 y ≤ 900） |
| hero | rigs/hero.js、props/scarf.js、props/sword.js | 勇者 59 个动作 16 个表情、围巾（拖尾/缺口/破洞/补丁/冻住/金边/沿围巾写字/擦屏/红带/书签/红绳）、剑与名字之剑 |
| horse | rigs/horse.js、rigs/monsters.js | 白马（步态相位 = 世界 x ÷ 步幅，不打滑；saddle 锚点）、史莱姆/蝙蝠/纸鸟/纸鸽/蜗牛/鸡/蝴蝶/萤火 |
| royals | rigs/king.js、rigs/princess.js、props/royal.js | 国王 29 动作（王冠五态是情绪刻度）、公主 20 动作、王冠/小皇冠/金印/印迹/权杖/钟/花环/摇篮/灯笼 |
| dragon | rigs/dragon.js | 64 点脊线样条、13 节沿弧长等距、背板挂字、15 个姿态、盘柱前后分层、逐节弹飞缩小、剪影、剧场木偶版、小蜥蜴 |
| folk | rigs/folk.js | 12 个市民变体、卫兵、书记官、传令官、孩子四阶段；批量人群函数；detail=0 远景版 |
| castle | env/castle.js、env/book.js | 王城外景 7 个时段/着火/烽火/塔旗写字/婚礼装饰/纸天撕裂/横幅绕塔；书桌与绘本、立体书机关、书内推镜与王城世界的换算 |
| plaza_throne | env/plaza.js、env/throne.js | 城下广场；王座厅（4 种光照、17 个预设机位、圣旨、玫瑰窗） |
| cave_theater | env/cave.js、props/cave.js、env/theater.js | 龙洞（5 种光照、名字墙、STANDOFF）、鸟笼/金币/石碑；纸剧场（黑边长成台口、主幕、台口咬合、撕幕、theaterState 共享状态） |
| map_journey | env/map.js、env/journey.js | 蜡笔/羊皮纸/小地图三形态共用 (u,v)；旅途长卷一镜到底（journeyCam 关键帧、遮挡物换区、heroJourneyState、hudState） |
| props | props/paper.js、village.js、misc.js | 告示、卷轴、圣旨、纸卷、纸卷塔、无限卷轴、片尾贴签、宝箱、听岔五图标、火焰字/火柱、风滚草、花瓣环、横幅、蝴蝶结、讲故事人的手、粒子 draw 回调 |
| fx_player | fx/transitions.js、ui/player.js | 冲击帧分级、速度线、甩镜帘、形状揭示（心/圆/杏仁）、翻页、燃烧边、撕纸、撕纸分屏、左冷右暖、灰烟；播放器（载入音频/LRC、打点对齐、清晰度、音频偏移） |

### 8.2 统一约定（写进 assets.md 第 0 节，所有组遵守）

- 签名 `drawX(g, o)`；`x,y` 为**脚底中心**（世界坐标，调用方已 applyCam）；`s` 缩放（1 = 全景标准高度）；`face` ±1 镜像（文字附件不镜像）；`t` 秒驱动呼吸/眨眼/飘动；`pose`/`expr` 枚举，`pose` 可传 `{from, to, k}` 关节角插值；`joints` 关节角增量（度）；`detail` 0 = 远景简化；`silhouette` + `keepColor`；`rod`（纸剧场木杆）；`squash`。返回锚点对象（head/mouth/handR/swordTip/scarfEnd/saddle…）。
- 函数内部 save/restore；**不在函数里调用 ctx.layer/ctx.mask**（图层由镜头决定，按 `layer` 参数分次画）。
- 布景：`drawXxx(g, T, cam, { layer: 'sky'|'far'|'mid'|'main'|'near'|'fg', ... })`，函数内部自己 applyCam；布局坐标导出为常量、预设机位导出为 `CAMS`。
- **统一脸部画法**（保证全员同一画风）：脸 skin 圆；眼 ink 竖椭圆（宽 ≈0.09×头宽、高 ≈0.13×头宽）+ 左上白色高光；腮红 blush 椭圆 α≈0.45；嘴 ink 弧线，张嘴填 redDeep + heart 舌；眉 ink 短弧；鼻最多一个 skinShade 小点。
- 纸艺质感：角色不描黑边，每个部件一片剪纸，部件间细微错位投影，顶边亮边，明暗用 shade，投影一律右下。
- 每个模块配模型图 `src/sheets/<名>.js`（与镜头同约定），渲染后亲眼检查至少 2 轮；交付 `docs/api/<组>.md`。

### 8.3 API 文档模板（8 节）

1 导出一览（名称｜类型｜用途）；2 坐标、尺寸与锚点约定；3 函数详解（签名、参数表含默认值、返回值）；4 枚举全表（每项一句话 + 用在哪个块）；5 推荐用法（3–6 段可复制代码，含 ctx.layer 设置）；6 与 assets.md 的差异；7 已知限制；8 模型图清单。

### 8.4 美术一致性检查（编排者做）

写一张 `_lineup.js` 模型图：全部角色 s=1 站一排（脚底线、260 身高线）+ 王座厅 TR-W 里试摆 + 王城阳台 CA-BAL s=0.26 试摆。本次发现的问题：勇者长围巾默认平铺在脚底高度，阳台上会“悬空伸出平台”、红地毯上红配红看不清 → 写进场景规范（用 `scarf.len/trail/to` 控制）。

---

## 9. 第七步：接缝合同（1 agent，与场景并行）

7 处隐形接缝：b04|b05 41.90、b05|b06 56.89、b07|b08 90.00、b10|b11 120.97、b12|b13 150.52、b15|b16 185.47、b16|b17 197.77。

`src/scenes/_seams.js`：

```js
export const SEAMS = {
  'b04|b05': {
    T: cue('N1a'), loc: 'THRONE', cam: CAMS.TR_2S, drift: { amp, speed, seed },
    env: { light: 'sunny', ... }, layers: { ... },
    actors: [{ fn: 'drawHero', params: { x, y, s, face, pose, expr, scarf, ... } }, ...],
    props: [...], ui: [{ fn: 'dialogBox', params: { ... } }], fx: { sat, vignette },
    motion: '接缝前后在发生什么', notes: '...',
  },
  // ...
};
export function drawSeam(g, T, ctx, key) { /* 参考实现：按合同画完整一帧 */ }
```

两边作者都 `import { SEAMS, drawSeam }`、**从 SEAMS 读值不要硬编码**，在接缝时刻落到/起步于合同状态（离开 0.3 s 以上自由运动）。用 `tools/seams.mjs` 验收：差值 < 0.01 看不出、0.01–0.03 轻微、> 0.03 要对齐。判读时注意：**快速移动的遮挡物、设计好的闪白也会产生大差值**（本次 b07|b08 是大橡树每帧移动 158 px，b10|b11 是撞墙闪白，都不是错位）。

---

## 10. 第八步：场景块（17 agent 并行）

### 10.1 场景作者规范（`docs/scene_guide.md` 要点）

1. 必读顺序：brief → storyboard（自己那一块整节 + 相邻块交接 + 规则章节）→ engine.md → 用到的 docs/api → seams → 模型图 final_*.png。
2. 文件：`src/scenes/bXX_*.js` 导出镜头数组，私有辅助放 `src/scenes/bXX/`；块覆盖范围严格等于 blocks.json 的 [start, end)；**不许改任何共享文件**（缺姿势用 joints 微调或在块内自己画，写进“需求”）。
3. 计时：每个 cue 一个重音动作，落在 cue 后 0–0.15 s；预备在前、余波在后；名字逐字用 NAME_TIMES；走马灯按本 cue 进度。
4. 接缝与转场：隐形接缝按合同；重叠转场由后一块画、前一块照常画到 end；全片唯一硬切 18.00。
5. “精致”十条：构图焦点、≥3 层景深视差 + 前景虚化、光向统一、色彩剧本、运动缓动与跟随、机位永远有漂移、表演（每个 cue 表情变化、不说话的角色也在反应）、文字规则（≥48 px、静止可读 ≥0.25 s、符号路径画）、字幕带、没有死帧。
6. 自检：lint → check.mjs blocks → cues 联系表 → sheet 联系表 → stills → pops → seams；**每张图都要用 Read 打开亲眼看**；至少完整迭代 2 轮。
7. 交付：代码 + `docs/blocks/bXX.md`（镜头表、与分镜差异、接缝/转场自检、已知问题、对基础模块的需求、**进度日志**）+ `out/review/bXX/` 下的图。

### 10.2 工作节奏（写进每个 builder 的 prompt）

- A 粗排约 30 分钟：所有镜头搭出来，每个 cue 有对的画面、卡点正确、没有空镜。
- B 精修约 60 分钟：表演、层次、光色、粒子冲击、转场交接、字幕带。
- C 打磨约 30 分钟：逐 cue 全分辨率检查、pops、性能预算、再看一轮。
- 每阶段更新进度日志（中断后接手的人能继续）；不要等相邻块。

### 10.3 集成负责人在场景阶段做的事

- 维护 `docs/integration_notes.md`：资产作者交付时报告的坑和拍板的决定，按“全局 / 按块”记录，场景作者开工先读（见第 12.3 节清单）。
- 用 SendMessage 向运行中的 builder 补发信息（例：UI 组交付后告诉 b07/b08 `drawHUD(g, { ...hudState(T), t: T })`；告诉 b12/b13 theaterState 的三个坑；告诉所有接缝块有 seams.mjs 可用）。
- 裁定冲突（例：b15 报告接缝合同与已拍板事项 4 处冲突 → 公主横幅 princessLight 底 + red 字；勇者横幅路径改为左起短引线 → 13 字从左往右 → 绕尖塔；王冠尾声 12°；阳台站位 935/985/1035 避开花箱 → 同时通知合同作者、b15、b16）。
- 用户催交付时广播收尾截止时间：“停止新打磨，只保证全覆盖、无报错、接缝对齐、删临时文件、写交付文档；收不了尾就回退到最近能渲染的版本”。

---

## 11. 第九步：集成、验收、出片

### 11.1 检查顺序

```bash
./tools/lint.sh src/scenes
node tools/check.mjs blocks --samples 12            # 17 块：报错 / 空镜
node tools/seams.mjs                                 # 7 处接缝
node tools/pops.mjs --from 0 --to 208.1              # 全片跳变（可选）
node tools/render.mjs cues --from 0 --to 208.1 --cols 8 --tile 320 --out out/review/final_cues.jpg   # 逐句画面总览
```

本次结果：16/17 干净；b09 有两个“计划中但没写完”的函数（agent 中断时正在重构：调用已写好、函数体没写）。修法：读上下文的状态函数，按资产 API 补写——`drawCageAndPrincess`（鸟笼 back → 公主绕吊钩随笼摆动 → 鸟笼 front）与 `drawLitPlates`（按 drawDragon 返回的 plates 坐标重画亮起的背板：金边圆片 + 深色面 + glyph13 亮字）。

### 11.2 出片

```bash
node tools/build.mjs                                                   # dist/dalabengba-mv.html（单文件，可离线播放）
node tools/render.mjs video --fps 30 --dpr 2 --workers 5 --out "dist/达拉崩吧-纸艺MV_1080p.mp4"          # 原版 208.1 s
node tools/render.mjs video --preset zhoushen --fps 30 --dpr 2 --workers 5 --out "dist/…_周深版_noaudio.mp4"   # 周深版 245.6 s
```

### 11.3 原版音频

- `https://music.163.com/song/media/outer/url?id=521493845.mp3` 会 302 到 404 页（会员歌曲）；`/api/song/enhance/player/url?ids=[id]&br=320000` 返回 code -110。
- ilem 2017–2018 作品合集里的同一录音（id 3392512852，208.07 s）可取到 320k URL → 下载 → 与 LRC 时长对上 → 混音：

```bash
ffmpeg -i video.mp4 -i original.mp3 -map 0:v:0 -map 1:a:0 -c:v copy -af apad -c:a aac -b:a 256k -shortest -movflags +faststart out.mp4
```

### 11.4 周深版音频（含偏移校准）

1. 网易云 Live 曲目是会员（-110）。B 站 `yt-dlp` 会 412；改为：先 curl 访问 `https://www.bilibili.com/` 拿 cookie → `api.bilibili.com/x/web-interface/view?bvid=…` 取 cid → `api.bilibili.com/x/player/playurl?bvid=…&cid=…&fnval=16` 取 dash 音频 baseUrl → 带 `Referer: https://www.bilibili.com/video/<BV>` 下载。
2. **偏移校准**：同一场演出的多个上传起点不同（官方视频上传多了约 0.7 s 片头、尾部也剪短了）。做法：下载 3–4 个来源，解码成 8 kHz 单声道，算 10 ms RMS 包络，归一化后在 ±3 s 内找互相关最大的滞后；时长与网易云原曲（245.6 s）一致、且彼此对齐的那几个就是母带起点，选码率最高的一个。

```python
import array, math
def env(path, hop=80):   # 8 kHz s16le → 10 ms RMS 包络，归一化
    a = array.array('h'); a.frombytes(open(path,'rb').read())
    out = [math.sqrt(sum(v*v for v in a[i:i+hop:4])/(hop/4)) for i in range(0, len(a)-hop, hop)]
    m = sum(out)/len(out); sd = math.sqrt(sum((v-m)**2 for v in out)/len(out)) or 1
    return [(v-m)/sd for v in out]
def best_lag(A, B, maxlag=300):   # 返回 B 相对 A 的滞后（×10 ms）
    N = min(len(A), len(B)) - 400; best = None
    for L in range(-maxlag, maxlag + 1):
        s = n = 0
        for i in range(300, N, 3):
            j = i + L
            if 0 <= j < len(B): s += A[i]*B[j]; n += 1
        if best is None or s/n > best[1]: best = (L, s/n)
    return best
# ffmpeg -i x.m4a -ac 1 -ar 8000 -f s16le x.raw
```

3. 视频与音频并行：渲染不依赖音频（偏移可在混音时用 `-itsoffset` / `adelay` 补），所以先开渲染、同时找音频；用后台命令“等渲染进程退出 → 立即混音 → 删中间文件 → open 打开”，零等待交付。

### 11.5 版权提示

音频归原权利人（原曲 ilem；演唱 洛天依/言和、周深；节目方）。有声 MP4 供个人观看；公开发布请署名原曲与演唱，并遵守平台二创规则。画面不含歌词原文。

---

## 12. 踩坑全集

### 12.1 编排与网关

- **网关 401（鉴权）/ 403（预算不足）会让所有后台 agent 在同一时刻全部死掉**。资产阶段 12 个、场景阶段 16 个先后整批中断。对策：
  - 派活时要求“**先写交付文档 / 进度日志，再做精修**”；
  - 中断后从 agent 转录尾部提取最后几条消息（它正在做什么、改到哪），写进接手 agent 的 prompt；
  - 接手 agent 先跑批量检查（check.mjs）确认现状，再补完半截改动；
  - **建议 `git init` 并在每个阶段提交**，半截编辑可以直接回退（本项目没用 git，b09 只能手工补函数）。
- Workflow 的后台 agent 在 VS Code 对话里看不见；用户要看过程时，用并行 Agent 调用（可展开），或每阶段主动汇报。
- 单个 Workflow 并发上限 = min(16, CPU−2)（8 核即 6）；需要更多并发就开多个 Workflow 或用并行 Agent 调用。
- 设计合成 agent 跑了 3 小时：合成任务太大。建议拆成“合成全局规则 + 地点布局”与“逐块分镜”两步，或按块分给多个合成者。
- 用户催交付时，砍掉独立评审轮，改为编排者自动化验收 + 只修坏块。若时间允许，推荐流水线：builder → 独立 reviewer（对照歌词与分镜挑 P0/P1/P2）→ fixer → 复验。

### 12.2 中文乱码与工具

- **中文经 Write / 补丁写入时会间歇性出现 U+FFFD**（agent 和编排者自己写的文件都中过，连 python 替换脚本里的中文搜索串也会坏）。对策：lint 必查 `\xef\xbf\xbd`；修复用“上下文 + 正则”替换（`re.sub('前文\\ufffd+后文', '前文X后文', s)`），必要时用 `\u` 转义写字。
- `window.duration` 一开始返回整首歌长度，模型图检查只采到 0.5 s 有效帧 → 改成 `warp.toReal(story.duration)`。
- 胶片颗粒每 1/12 s 换一次，不同时刻的两帧整屏差 ±10 → 像素对比一律 `--nograin`。
- `check.mjs` 默认抽样要落在模型图时长内；按真实时间轴取时刻的模型图（book/journey/map）早段空白要补“索引页”，否则报 EMPTY。
- Playwright `page.evaluate(字符串)` 不会把参数传给字符串函数。
- render 的 `--only` 让单块渲染互不干扰；块动态加载让某块语法错只影响自己。

### 12.3 素材/API 坑（integration_notes 的精华）

- 勇者长围巾：窄平台会悬空、红毯上红配红 → 用 `scarf.len/trail/to/ground` 控制；转身一周时围巾在半程翻边（跳舞镜头自己画围巾）。
- 勇者朝左走/骑时 `phase` 不随 face 翻转（马的会）→ 传 `phase = face × 路程 ÷ 步幅`。
- 白马必须单独一个 ctx.layer（边缘明暗只叠在图层内已有内容上）；同层顺序：马 → 后座公主 → 勇者 → 缰绳。
- 整帧 `ctx.fx.sat` 会把警报红也降掉 → 引擎加了按图层的 `sat/bright/contrast/filter`。
- `whipCurtain` 方向：要“速度线带由右往左”写 `from:'right'`（写 `dir:'left'` 是反的）。
- 道具栏燃烧只用 `burnEdge`，不要同时用 itemBar 自带 burn（两条燃烧线对不齐）。
- `drawHUD` 要补 `t`：`drawHUD(g, { ...hudState(T), t: T })`。
- theaterState：160.4 后 `peeled` 仍是 13、`pulse` 用 `tf.s` 叠加、`menu.show/drop` 已缓动过不要再缓动。
- 洞口=龙眼匹配：两边都用 `journey.caveMouthPath` 同一形状（杏仁巨眼轮廓两端尖角略不同）。
- 洞穴光照 glint→lit 亮度从 0.56 跳到 1，要用 `{from,to,k}` 渐变。
- b11→b12：剧场背景片不画前景层、饱和度低 10% → b11 在 137.0–137.40 淡出前景层、sat 渐变到 0.9。
- 分镜给的个别机位会切掉人（`TR_KING2S`、E01 的 `cam(1300,420,1.5)` 看不到地面）→ 场景作者按实际构图调。
- 王城阳台：自带红灯笼（别再画一盏）；左侧花箱挡人到胸口 → 站位 935/985/1035。
- 国王王冠：E09 之后歪 12°，O06 自己回正；单飞的王冠用 drawKing 返回的 crownBase/crownRot/crownS 接 drawCrown。
- 旅途：main 层不画史莱姆/宝箱/村民（只给位置）；事件时刻有浮点尾数，判断写 `T >= 字段`。
- 快乐体「鲁」内白极窄，48 px 以下偏糊。
- 插值姿势两端手持物不同，道具在 k=0.5 处跳变。

### 12.4 音频

- 网易云会员歌曲 outer URL 404、player API -110；同一录音可能在别的专辑里免费。
- B 站 yt-dlp 412 → 用网页 API + cookie + Referer。
- 不同上传起点差约 0.7 s → 多来源互相关校准（11.4）。

---

## 13. 复现清单（按顺序打勾）

1. [ ] 装环境、下字体、验字形覆盖（第 2 节）。
2. [ ] 取 LRC，写 `src/cues.js`（CUE_LIST、NAMES、PRESETS、makeWarp），生成 `docs/cues.md`（第 3 节）。
3. [ ] 写引擎 `src/core/*`、入口、播放器基础版、`tools/*`，用一个自检镜头渲染测试帧，确认纸纹/投影/字体/粒子/运动模糊/单文件打包都正常（第 4–5 节）。
4. [ ] 写 `docs/brief.md`、`docs/engine.md`（第 6 节）。
5. [ ] 分镜设计：3 提案 → 2 评审 → 1 合成，产出 storyboard.md / assets.md / blocks.json；编排者读一遍定稿的全局规则和 1–2 个块，确认可执行（第 7 节）。
6. [ ] 预先建好 17 个空块文件与动态加载的 `scenes/index.js`（场景作者不碰它）。
7. [ ] 基础资产 12 组并行（第 8 节），每组：实现 → 模型图 → 看图迭代 2 轮 → API 文档；编排者做阵容一致性检查，把各组报告的坑写进 integration_notes。
8. [ ] 接缝合同（第 9 节），与场景并行启动即可，场景作者先做块中间部分。
9. [ ] 写 `docs/scene_guide.md`，17 个 builder 并行（第 10 节），编排者持续补发信息、裁定冲突。
10. [ ] （推荐）每块独立评审 + 修复。
11. [ ] 集成验收（第 11.1 节），修坏块，全片联系表人工过一遍。
12. [ ] 打包 HTML、渲染 MP4、配音频（原版、周深版），抽帧检查，交付。

预估：顺利的话约 8–10 小时墙钟（本次 14 小时含两次整批中断）；子 agent 约 50 个；token 约 2000–2500 万。

---

## 附录 A：目录结构（参考实现）

```
dalabengba-mv/
  index.html  package.json  复现全流程.md
  src/
    main.js  story.js  cues.js
    core/  engine.js paper.js camera.js particles.js texture.js fonts.js ease.js util.js
    rigs/  hero.js horse.js king.js princess.js dragon.js folk.js monsters.js
    env/   book.js castle.js plaza.js throne.js map.js journey.js cave.js theater.js roundwindow.js
    props/ scarf.js sword.js paper.js royal.js cave.js village.js misc.js
    ui/    type.js kit.js player.js
    fx/    transitions.js
    scenes/ index.js _seams.js _kit_test.js b01_book.js … b17_close.js  b01/ … b16/（各块私有辅助）
    sheets/ <模块>.js（资产模型图）  seams.js  _lineup.js
  tools/  server.mjs chrome.mjs render.mjs check.mjs pops.mjs seams.mjs build.mjs lint.sh
  assets/ fonts/（4 套 OFL 字体）  audio/（混音用，不进单文件 HTML）
  docs/   brief.md cues.md engine.md storyboard.md assets.md blocks.json scene_guide.md integration_notes.md seams.md
          design/（三份提案 + 两份评审）  api/（12 份 API 文档）  blocks/（各块交付文档）
  out/review/  模型图、联系表、接缝对照、关键帧
  dist/   达拉崩吧-纸艺MV.html  达拉崩吧-纸艺MV_1080p.mp4  达拉崩吧-纸艺MV_周深版_1080p.mp4
```

---

## 附录 B：Prompt 模板

以下是实际使用过的 prompt 的精简版（`{…}` 处替换）。所有执行者 prompt 都要包含：项目根目录、必读文档、只能改哪些文件、纯函数规则、lint（含乱码）、看图迭代、交付格式、返回结构化简报。

### B.1 分镜提案（3 个，换立场）

```
你在为《达拉崩吧》制作一支纯代码 MG 动画 MV（Canvas 2D 程序化绘制，纸艺立体绘本 × 复古 RPG 风格）。项目根目录 {ROOT}。
必读：docs/brief.md（导演任务书）、docs/cues.md（提示点表）、docs/engine.md（引擎 API）。
硬性规则：画面严格跟 cue 走、只卡 cue 不卡拍；不写整句歌词原文（只可写四个长名字、王浩然、≤5 字短对白/拟声、自拟界面字、听岔五名词）；中文写完 grep U+FFFD 必须为 0；只写文档不写代码。
你是分镜师「{立场名}」（方案 {X}）。{立场说明：A 连贯派 / B 喜剧字体派 / C 制作技术派，见第 7.1 节}
写一版完整分镜到 docs/design/proposal_{X}.md：
1 一句话主张 + 核心卖点 ≤8；2 色彩剧本；3 逐镜头分镜表（覆盖 0–208.1 不留空档；每镜：镜头号、cue+默认秒起止、时长、地点、景别与机位运动、画面内容与动作（精确到 cue/名字分段）、画面文字、图层与景深、出镜转场（类型+动机+由谁负责））；
4 连贯设计（地图路线、方向、围巾线索、形状匹配、机位体系、各地点固定布局）；5 名字梗设计（逐遍升级、卡 a/b/c）；6 最难做的 5 个镜头及规避办法。
返回：文件路径、一句话主张、最得意的 5 个点子。
```

### B.2 评审（2 个，换视角）

```
{同上背景}。你是评审 {N}，视角：{导演/观众：歌词贴合、连贯、笑点、情绪曲线、高潮 | 制作/一致性：能否做精致、复用、风格统一、分块交接、性能}。
完整阅读三份提案；每份按歌词贴合/连贯/趣味/精致可实现/一致性打分（1–10）并写理由；
列出：应进入定稿的最佳点子 12–20 条（出处、时间段、理由）、必须避免的问题 8–12 条、定稿骨架建议（以哪份为底、嫁接哪些点子）。写入 docs/design/judge_{N}.md。
```

### B.3 总导演合成

```
{同上背景}。你是总导演。完整阅读三份提案与两份评审，按评审意见合成最终定稿，产出：
(1) docs/storyboard.md：视觉主张、色彩剧本、全片机位体系与角色标准比例、各重复地点的固定布局（关键物体世界坐标）；把全片切成 14–17 个制作块（每块 8–18 秒，起止落在 cue 或自然剪辑点），每块写块 id、代码文件名、起止、cue、逐镜头分镜、入场状态、出场状态、与相邻块的交接方式（相邻块出入场逐项对得上）；名字梗总表；附录 NAME_TIMES。
(2) docs/assets.md：基础资产规格（每个模块：文件、导出函数签名、参数表、坐标约定、需要的姿态/变体（从定稿分镜统计）、验收标准）。
(3) docs/blocks.json：[{id,file,title,start,end,cues,summary,entry,exit,handoff_next}]。
要求：每个 cue 都有画面；名字逐遍升级；三大高潮；全片连贯。
```

### B.4 基础资产作者（12 组）

```
你是《达拉崩吧》纯代码 MG 动画 MV 的基础资产作者（Canvas 2D、纸艺立体绘本 × 复古 RPG）。项目根目录 {ROOT}。
必读：docs/brief.md（风格与角色）、docs/engine.md、docs/assets.md（第 0 节通用约定 + 你负责的章节）、docs/storyboard.md（第 2–4 节；grep 第 5 节里用到你资产的镜头）。
硬规则：只写你负责的文件、自己的模型图 src/sheets/<名>.js、文档 docs/api/<组>.md；不改 src/core、其它组、src/scenes；纯函数（参数含 t 决定画面）；函数内部 save/restore，不调 ctx.layer/ctx.mask；只用 PAL；纸艺质感（不描黑边、剪纸部件、细微投影、顶边亮边、shade 明暗、投影右下）；统一脸部画法（见第 8.2 节）；木偶支持 o.joints、o.detail、pose {from,to,k}；文字走 type.js；符号路径画；每次写完 ./tools/lint.sh。
流程：实现 → 模型图（PAL.paper 底、标注、网格展示全部 pose/表情/状态 + 动态循环）→ node tools/render.mjs stills/sheet --sheet <名> 渲染 → Read 亲眼对照验收标准，至少完整迭代 2 轮 → 写 API 文档（第 8.3 节 8 节模板）。精致是第一要求。
你负责：「{组名}」；文件 {文件}；规格 {章节}；模型图 {名}；要点 {本组要点}。
完成后返回：文件、模型图路径、API 文档、与规格的差异、遗留问题。
```

### B.5 收尾接手（中断后续做）

```
你接手《达拉崩吧》纸艺 MV 的一组基础资产。上一位作者已完成大部分实现并迭代了 2–3 轮，但在收尾时因网关错误中断，没来得及写文档。你的任务是“收尾 + 交付”，不是重做。
现状（集成负责人已核对）：{check.mjs 结果}；docs/api/ 为空——这是最重要的交付物。
步骤：1 date 记开始时间，约 40 分钟；2 读本组源码与模型图；3 立即写 API 文档初稿（防止再次中断）；4 处理上一位作者中断时正在做的事：{从转录尾部提取的最后几条消息}，确认改动完整、渲染并亲眼检查；5 补齐场景马上会用到的小缺口（不改签名与默认行为）；6 删除本组临时文件 {列表}；7 lint + check 通过，渲染 final_*.png，文档定稿。
```

### B.6 接缝合同作者

```
你是“接缝合同”作者。全片 17 块由 17 位作者同时编写，其中 7 处是隐形接缝（同地点、同机位、不重叠），前一块最后一帧与后一块第一帧必须几乎像素一致。两边作者看不到彼此代码，所以先把每个接缝时刻的完整画面状态写成代码合同。
交付：src/scenes/_seams.js（SEAMS 7 项：T（cue()+偏移）、loc、cam、drift、env、layers、actors[{fn,params}]、props、ui、fx、motion、notes；drawSeam(g,T,ctx,key) 参考实现，只 import 基础模块与 cues.js）；src/sheets/seams.js 模型图（每个接缝一段，亲眼检查）；docs/seams.md（每个接缝的画面、字段含义、前一块如何落到、后一块如何起步、验收方法、基础模块问题）。
状态必须符合 storyboard 对该时刻的描述，没写死的细节由你拍板，选最利于两边衔接、最好看的。
```

### B.7 场景块作者（17 个，按块填差异）

```
你是《达拉崩吧》纸艺 MV（Canvas 2D 纯代码动画，1920×1080）的场景块作者，负责制作块 {bXX}「{标题}」。项目根目录 {ROOT}。全片 17 块由 17 位作者同时编写；基础资产已完成并有 API 文档。用户的要求是：画面按歌词来、精致、分镜连贯。
开工前必读：docs/scene_guide.md（逐条遵守）→ docs/integration_notes.md（全局 + 你这一块）→ storyboard「### {bXX} ·」整节及规则章节 → 用到的 docs/api/*.md 与模型图。
你的块：文件 {file}；范围 [{start}, {end})；cue：{cues}。
交接：{入场：由你画什么转场/隐形接缝合同；出场：由谁画、你照常画到 end}。
重点：{本块的高潮/卡点/易错点}。风险：{storyboard 第 10 节对应条目}。
接缝合同（如有）：{接缝 key 与时刻}；合同未出现前先做远离接缝的部分；出现后严格对齐并用 node tools/seams.mjs 自检；收尾时仍无合同就按 storyboard 做并写明“接缝待对齐”。
工作方式：A 粗排约 30 分钟 → B 精修约 60 分钟 → C 打磨约 30 分钟；每阶段更新 docs/blocks/{bXX}.md 进度日志；不要等相邻块。
硬规则：只写 {file}、src/scenes/{bXX}/、docs/blocks/{bXX}.md、out/review/{bXX}/；不改共享文件；纯函数；不写歌词原句；字幕带；NAME_TIMES；cue()+偏移；lint 与 check.mjs blocks 必须通过。
返回：镜头数与概要、与分镜差异、接缝/转场自检结论、已知问题、对基础模块的需求、3–5 张关键帧路径。
```

### B.8（推荐，本次因赶工未执行）独立评审 + 修复

```
你是块 {bXX} 的独立评审兼修复者（不是原作者）。对照 storyboard 与 brief，渲染 cues / sheet / 关键帧 / seams / pops，逐 cue 检查：画面是否就是那一句在讲的事、重音是否落在 cue 后 0–0.15 s、构图层次光色表演、文字规则与字幕带、接缝与转场、报错与空镜。
先把全部问题按 P0（错画面/错时间/接缝破/违规/报错）、P1（明显不精致）、P2（锦上添花）写进 docs/review/{bXX}.md，再按优先级修复，修完复验并更新 docs/blocks/{bXX}.md。只改本块文件。
```

---

## 附录 C：交付物

| 文件 | 说明 |
|---|---|
| `dist/达拉崩吧-纸艺MV.html` | 单文件播放器（约 3 MB，字体子集内嵌）：载入音频、版本预设、T 打点对齐、LRC 字幕、清晰度、音频偏移；空格播放、←/→ ±1 s、[ ] 上/下一个 cue、H 隐藏界面、F 全屏 |
| `dist/达拉崩吧-纸艺MV_1080p.mp4` | 原版 208.1 s，1080p30 H.264 + AAC 256k |
| `dist/达拉崩吧-纸艺MV_周深版_1080p.mp4` | 周深 Live 245.6 s，1080p30，按周深版 warp 渲染 |
| `out/review/final_cues.jpg` | 全片逐句画面联系表（96 个 cue） |
