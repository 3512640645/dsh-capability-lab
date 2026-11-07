# dsh-capability-lab

> **DSH 能力工具包** —— 把「让 AI 操控电脑」从"能点对"推进到**可量化、可回归、可复现**；
> 并附一条**场景化视频流水线**（场景 JSON → 双画幅成片 → 规格驱动配乐 → 字幕 → 质检）。
>
> 它不是"一个能点鼠标的脚本"。它的重点是**判据**：先定义"什么叫达标"，再让每次改动都能被同一个尺子量。

---

## 它解决什么问题

大多数"AI 操控电脑"的做法停在演示：点一下、截个图，看着能跑。
一旦要它**连续操作**（瞄准、连击、按节奏出招），就会暴露三类问题：

| 问题 | 表现 | 本包的应对 |
|---|---|---|
| **通道选错** | 用元素通道做游戏操作，或用真实指针做批量表单 | 两条通道分别量化，给"何时用哪个"的选择指南 |
| **延迟说不清** | 把"插件整条调用耗时"当成"投递延迟"，于是冤枉调度器、又放过插件 | **三层延迟口径**：调度精度 / 注入延迟 / 端到端，必须同批同源并报 loss |
| **"达标"没定义** | 今天 84.9%、明天 98.1%，全看分母怎么算 | `CONTROL-BENCH.md`：分级判据 + 口径硬要求 + 踩坑记录 |

---

## 目录

```
dist/dsh-capability-lab-0.1.0.tgz   可直接安装的插件包（0.86 MiB）
docs/                               四个技能 + 能力判据（在线可读）
  pc-control.md        真实指针操控：平滑轨迹 / 按键保持 / 定时序列 / 常驻注入器
  pc-elements.md       元素优先自动化：先定位元素再动手，自带动作后校验
  control-bench.md     怎么给自己评级：L1/L2/L3 + 三层延迟口径
  video-pipeline.md    怎么用场景 JSON 出片（五步命令 + 质检指标）
  CONTROL-BENCH.md     能力判据全文（含踩坑与修订史）
video/                              介绍片（76 秒）
  plugin-intro-16x9.mp4 / -9x16.mp4          双画幅
  plugin-intro-16x9-subbed.mp4               烧字幕版
  plugin-intro-contact.png                   接触表（12 帧）
  plugin-intro.srt                           字幕
LICENSE
```

安装包内含工具源码副本：`aim/`（输入引擎 + 视觉闭环）、`winctl/`（元素优先自动化）、
`arena/`（自建靶场）、`engine/`（视频引擎）、`audio/`（规格驱动配乐）。

---

## 安装

**方式一：插件包**（推荐）

```powershell
# 在 DSH Desktop 的插件管理里安装本仓库 releases 或 dist/ 下的 .tgz
# 或在 DSH 会话中让它安装本地 tgz
```

安装后插件会把自己的 4 个技能落地到 `$DSH_HOME/skills`，**重启 DSH 后**新会话即可看到。

> 实现细节：宿主的技能扫描只认 `.dsh/skills`、`.agents/skills`、`$DSH_HOME/skills`，
> **包内 `skills/` 不会被自动扫描**，因此插件在自己的 `apply()` 里做落地。

**方式二：只拿工具**

```powershell
# 从 tgz 里解出 tools/ 即可独立使用（不依赖 DSH 插件机制）
```

---

## 能力现状（实测，非宣传）

### 操控（真实指针通道，自建靶场 30 秒一局）

| 判据 | 要求 | 实测 |
|---|---|---|
| 命中数 | ≥ 20 / 30s | **124–129** ✅ |
| 命中率 | ≥ 90% | **99.22%**（763/769） ✅ |
| 平均反应时间 | ≤ 700ms | **222.9–230.6ms** ✅ |
| 目标会动/会缩时 | 命中率 ≥ 70% | **100%**（240px/s，半径 40→18px） ✅ |

### 延迟（三层口径，同批同源）

| 层级 | 要求 | 实测 |
|---|---|---|
| ① 调度精度 | \|dev\| 中位 ≤15ms / 最大 ≤50ms | 中位 **11.3ms** / 最大 **42.1ms** ✅ |
| ② 注入延迟 | 中位 ≤5ms、p90 ≤10ms、loss=0 | 键盘 **1.41ms** ✅ / **鼠标 7.79ms ❌** / raw 通道 **loss 11.8% ❌** |
| ③ 端到端 | 中位 ≤30ms / 最大 ≤80ms | **28.5ms** / **67.0ms** ✅ |

### 未达标项（如实列出）

- **鼠标注入延迟 7.79ms > 5ms**：瞄准类玩法受此限制
- **raw 通道有丢失**（34 个事件丢 4 个，11.8%）：走 RawInput/DirectInput 的目标可能收不到后台注入
- **元素通道单步延迟**：办公类任务 p50 1270ms，其中包含校验等待，不是纯动作耗时
- **真机通道没有 CI 覆盖**：真实指针打靶需要独占桌面，回归总闸只跑离线项
- **视觉闭环频率**：旧记录的 468ms 找不到可复现出处，当前**无法判定**

---

## 视频流水线

场景 JSON 同时驱动**画面与声音**（画声同源），一份 spec 出双画幅成片。

```powershell
# 1) 画面 → 无声 MP4（--html-only 只出 HTML，秒级）
node engine/render.mjs --spec engine/scenes/plugin-intro.json --aspect 16:9 --fps 30 --port 9555 `
     --out out/plugin-intro-16x9-silent.mp4 --frames frames/plugin-intro-16x9

# 2) 配乐（读同一份 spec 的 audio 块，内置 13 项质检）
python tools/make-audio.py --spec engine/scenes/plugin-intro.json --out out/plugin-intro-audio.wav

# 3) 字幕
node engine/subs.mjs --spec engine/scenes/plugin-intro.json --out out/plugin-intro

# 4) 混流
ffmpeg -y -i out/plugin-intro-16x9-silent.mp4 -i out/plugin-intro-audio.wav `
       -c:v copy -c:a aac -shortest out/plugin-intro-16x9.mp4

# 5) 质检（容器 + 音画 + 响度）
node engine/qa.mjs --video out/plugin-intro-16x9.mp4 --expect-duration 76
```

介绍片的实测数字：76.000s / 30fps / 2280 帧，母带 **13/13** 质检、LRA 9.5 LU，字幕 27 条覆盖 87%。

---

## 三条设计经验（都是用事故换来的）

1. **插件整条调用耗时 ≠ 投递延迟。**
   某插件 `keypress` 整条调用 296–604ms，而投递本体只有 **0.17–0.33ms** —— 时间全花在"注入前"
   （前台切换 sleep、逐字符 sleep）。把前者当后者写进判据，会同时冤枉调度器、又放过插件。

2. **脏临时目录会拖慢整条闭环。**
   截图工具每拍往 `%TEMP%` 写文件，攒到 1827 个/83MB 后，截图耗时从 324ms 涨到 189ms 的**反向**
   逐局上扬。清理 + 每拍即删后，整片反应时间从 709ms 降到 224ms。

3. **"达标"必须绑定口径与批次。**
   同一个成功率，换一种同样合理的口径能从 51.6% 变到 98.1%（46 个百分点），
   而分母还会随语料增长漂移。**不写清"谁进分母、哪条通道、哪一批"的成功率没有判定效力。**

---

## 许可

MIT © YG
