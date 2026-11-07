---
name: video-pipeline
description: 场景化视频流水线——写一份场景 JSON，用 build.mjs 五步出片（配乐/字幕/渲染/混流/质检），一次出 16:9 与 9:16 双画幅，含质检指标与「画声同源」写法
whenToUse: 要出一部带配乐与字幕的成片、或要改画面节奏而不想回去数秒数对齐音频时
---

# video-pipeline · 场景 JSON 出片

核心理念：**把"做一部片子"从手写每个元素的动画数学，变成写一份场景描述 + 跑一条命令。**
场景 JSON 是**唯一时间轴来源**：画面（`sections`）与声音（`audio`）写在同一个文件里，所以改画面节奏时音频不会掉队。

## 1. 一条命令出片（五步全部由同一份 JSON 驱动）

```powershell
$node = 'C:\Users\35126\AppData\Roaming\dsh-desktop\harness\.desktop-bin\node.cmd'
$e = 'D:\dsh-plugin\tools\engine'        # 装到 profile 后换成 node_modules\dsh-capability-lab\tools\engine
$py = 'D:\dsh-plugin\tools\audio\make-audio.py'

& $node "$e\build.mjs" --spec "$e\scenes\dsh-phone-access.json" --aspect 16:9 --aspect 9:16
```

| 步骤 | 输入 | 产物 | 自检 |
|---|---|---|---|
| 1 配乐 | `spec.audio`（和弦段 + 事件表） | `out/<name>-audio.wav` | 响度/真峰/频段/空洞/侧链 **7 项**（v5 起 10 项） |
| 2 字幕 | spec 里的文字元素 | `out/<name>.srt` / `.vtt` | 条数/覆盖率/最长一条 |
| 3 渲染 | `spec.sections` | `<name>-<tag>-silent.mp4` | 逐帧由 `seek(t)` 计算 |
| 4 混流 | 画面 + 配乐 | `<name>-<tag>.mp4` | — |
| 5 质检 | 成片 | `<name>-<tag>-qa.json` + 接触表 | **6 项** |

单跑某一步：

```powershell
& $node "$e\render.mjs" --spec "$e\scenes\dsh-phone-access.json" --aspect 16:9 --html-only   # 只出 HTML 调画面
& $node "$e\render.mjs" --spec "$e\scenes\dsh-phone-access.json" --aspect 9:16               # 只渲染某画幅
& python "$py" --spec "$e\scenes\dsh-phone-access.json" --out out\x.wav                      # 只出配乐
& $node "$e\subs.mjs"   --spec "$e\scenes\dsh-phone-access.json" --out out\x                  # 只出字幕
& $node "$e\qa.mjs"     --video out\x.mp4 --expect-duration 45                               # 只质检
& $node "$e\build.mjs"  --spec ... --aspect 16:9 --burn-subs                                  # 额外产出烧录字幕版
& $node "$e\compare.mjs" --a old.mp4 --b new.mp4 --label-a A --label-b B --out cmp.mp4        # 对比样片
```

**双画幅**：组件不写死坐标，接收 `ctx.aspect` 后分支（`pipeline` 横排一列一个 → 竖排一行一个；
卡片纵排、手机居中、标题收字号；光点横屏沿 X、竖屏沿 Y）。所以 `--aspect 9:16` 不是裁切，是重排。

**性能**：串行双画幅 8.9 分钟 → 并行 6.0 分钟（瓶颈是逐帧渲染 1350 帧×2 ≈ 5 分钟）。
并行有个坑：两个渲染进程共用 CDP 端口 9555 与配置目录时，第二个实例**静默失败**（产物缺失但退出码不报错，
旧的旧文件还在，肉眼看不出来）。修法：端口/配置目录按画幅派生 + `build.mjs` 产物校验
（文件存在 + 帧目录帧数 ≥98% 预期 + `mtime >= 本次开始时间`，防止"沿用旧文件"的假成功）。

## 2. 场景 JSON 怎么写

```jsonc
{
  "title": "片名", "duration": 45.0, "fps": 30,
  "style": "classic",                        // 省略 = anthropic（默认规范档）；classic = 旧观感，用于 A/B
  "bg": "both",                              // mesh | noise | both（默认）| none
  "audio": {
    "key": "A minor", "target_lufs": -15,
    "sections": [ { "start": 4.2, "dur": 7.4, "chord": [174.61,220,261.63], "root": 174.61, "arp": [] } ],
    "events": [
      { "t": 8.20, "type": "impact", "amp": 0.20 },
      { "t": 8.22, "type": "bell", "freq": 146.83, "pan": -0.3 },
      { "t": 13.40, "type": "ramp", "count": 9, "span": 2.15, "tone0": 1000, "tone1": 2300 }
    ]
  },
  "sections": [
    { "id": "problem", "name": "问题", "start": 4.2, "end": 11.4,
      "layers": [
        { "type": "pipeline", "props": { "from": "A", "to": "B" } },
        { "type": "caption", "props": { "items": [ { "in": 8.8, "text": "文字" } ] } }
      ] }
  ]
}
```

* **事件类型**：`whoosh`（过场气流）· `bell`（铃音）· `click`（界面点按）· `impact`（低频下坠）·
  `riser`（上行张力）· `series`（等间隔一串）· `ramp`（音高上行一串）。转场音（riser/impact/落音）
  在段落边界自动排布，不需要手写。
* **元素只给"进入时刻"**，其余交给运行时：

| data 属性 | 含义 | 默认 |
|---|---|---|
| `data-in` / `data-out` | 进入 / 退出时刻（秒） | 必填 / 99（不退出） |
| `data-fade` / `data-fade-out` | 渐变时长 | 0.5 / 0.4 |
| `data-dy` / `data-scale` / `data-rotate` | 入场位移(px) / 缩放起点 / 起始旋转(deg) | 18 / 1 / 0 |
| `data-ease` | `outCubic`/`outExpo`/`inOutCubic`/`spring`/`linear` | outCubic |
| `data-trail` + `data-trail-axis` + `data-trail-amp` | 入场拖尾 `ghost`（卡片残影）/ `ink`（文字彗尾）、轴向、强度 | 无 |

* **段落门控**：`section.start/end` 会在渲染管线最外层再乘一层透明度 —— 门控必须做在最外层，
  散在各自元素里会漏掉光点、抽屉这类"轨道元素"，出现"游离的蓝点"。
* **内容密度审计**：同一帧同时运动元素 ≤3（超了只告警不改时间轴，排期属于内容决策）。
  产物里可 grep 当前生效值：`<!-- style-active anthropic | bg=… grid=.16 noise=.025 pad=5/3/2.4 … -->`。

**内置 10 个组件**：`hud`（段落名+进度条，`persist:true` 常驻）· `title`（默认逐字错峰，中英混排不破字）·
`cards`（竖屏自动纵排）· `pipeline`（链路图+流动光点）· `bullets` · `phone`（可带抽屉开合）· `outro` ·
`caption` · `timeline`（节点按 `t` 点亮，`even:true` 改等距）· `stats`（大数字滚动计数）。

## 3. 质检指标怎么读

**音频 10 项**（`tools/audio/make-audio.py` 落 `out/qa-audio-spec.json`，前 7 项是基线）：

| 检查 | 判据 | 含义 |
|---|---|---|
| 响度 | 在 spec 目标 ±1.5 LUFS 内 | 成片整体响度对齐平台口径 |
| 真峰 | ≤ −1 dBTP | 防削波/编码后爆音 |
| 低频（80–250Hz） | 占比 20–50% | 有没有"闷"或"空" |
| 中频（1k–4k） | 占比 10–30% | 人声/主旋律可听度 |
| 空气感（10k+） | ≥1.5% | 高频是否死板 |
| 无空洞 | 最长静音 <400ms | 断带检测 |
| 侧链避让 | 最深 ≤ −3dB | 音效有没有把铺底压开 |
| 动态 LRA | ≥5.0 LU（v4 基线 4.0） | 段落起伏，不是一条平线 |
| 段落能量分层 | 副歌−主歌 ≥2.0 dB | 结构编排是否真的分层 |
| 转场自动排布 | ≥3 个 riser/impact/落音 | 段落边界有没有转场 |

**视频 6 项**（`engine/qa.mjs` 落 `*-qa.json` + 接触表）：分辨率符合预期 · 时长符合预期 ·
帧率 ≥24fps（不要掉帧顿感）· **无 ≥0.5s 纯黑段**（引擎漏画会表现为黑场）· 接触表已生成且 12 格均为真帧
（人眼复核节奏）· 混音响度在 −16～−12 LUFS；外加 A/V 双流时长一致（`|Δ| ≤ 100ms`）。

`qa.mjs` 还能用夹具回归自证：同相夹具必须 PASS、反相与退化夹具必须 FAIL（判据 + 趋势双断言）——
**一个"总是通过"的检查项比没有检查更危险**，所以夹具反向也要验。

## 4. 已知限制（如实）

* 成片正片很重（一局双画幅 ≈9 分钟、帧渲染占 7 分钟），**生产流程不要放进回归总闸的默认档**。
* `.build/*.html` 可单独打开调试；改 `lib.mjs` 前先备份，回滚就是 `Copy-Item .backup\lib.mjs.*.bak lib.mjs -Force`。
* 字幕是从画面 `caption` 层抽出来的，所以烧录字幕版与画面内说明**文字重复**；要消除需给 spec 加"字幕变体"。
* `compare.mjs` 的标签里 `:` `,` `'` 在 ffmpeg 滤镜图里是语法字符，写 `16:9 横屏` 这种标签必炸 —— 脚本里做了转义。
* 组件库仍缺：真正的折线/柱状图表、代码块逐行高亮、对比滑动。
* 依赖 ffmpeg/ffprobe：本机放在 `D:\dsh-video\tools\ffmpeg\`；本插件**未打包 ffmpeg**（三个 exe 合计 ~500MB），
  请自备并让脚本能找到。
