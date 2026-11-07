---
name: pc-control
description: 桌面真实指针操控——先按目标形状选通道（元素 vs 真实指针），再用 input.mjs 平滑轨迹/按键保持/定时序列，用 vision.mjs 截图找靶并换算坐标，含三层延迟口径警告
whenToUse: 需要真的移动鼠标、真的按键（画布/游戏/自绘 UI/视频帧/无 UIA 的目标），或需要拟人轨迹与真实输入延迟测量时
---

# pc-control · 桌面真实指针操控

本技能只管一件事：**让指针像手一样动过去，并且知道这一动到底花了多少时间**。
元素通道（UIA invoke）不在这里，见技能 `pc-elements`。

## 0. 先决策：走哪条通道（不要混着谈成绩）

| 目标形态 | 走哪条 | 理由 |
|---|---|---|
| 标准控件（按钮/输入框/列表项，有 automationId/name） | **元素通道** | 更快、不抢前台、能自证；见 `pc-elements` |
| 自绘画布 / 游戏 / 图片 / 视频帧 / 无 UIA 的自绘 UI | **真实指针**（本技能） | 只依赖像素 |
| 需要测「真实输入延迟」或拟人轨迹 | **真实指针** | `Invoke` 不产生真实鼠标事件，测不了延迟 |
| 目标会动 | 元素优先，打不到再兜像素 | 元素在动也 invoke 得中；像素要提前量 |

选错通道的代价是**数字不可比**：靶场里元素通道的"反应时间"≈1.5s 是整环开销（find+click+校验），
纯 Invoke 分发只要 ~170ms；真实指针整环 ≈470–480ms。两者不能放进同一张表比大小。

## 1. 运行前提（Windows 上照抄）

```powershell
$env:TEMP='D:\dsh-tmp'; $env:TMP='D:\dsh-tmp'          # 后端截图落这里，目录不存在会直接失败
New-Item -ItemType Directory -Force 'D:\dsh-tmp' | Out-Null
$pkg = 'D:\dsh-plugin'                                  # 本插件根；装到 profile 后换成 node_modules\dsh-capability-lab
$node = 'C:\Users\35126\AppData\Roaming\dsh-desktop\harness\.desktop-bin\node.cmd'
```

所有示例**默认 dry-run**：先看它想点什么，确认后再加 `--live`。

## 2. 五分钟跑通（不改任何文件）

```powershell
# ① 离线图找靶（完全不碰桌面）
& $node "$pkg\tools\aim\examples\aim-at.mjs" --image D:\dsh-arena\probe\targets-b.png --min-area 100

# ② 真机只读截屏找靶（仍然不点）
& $node "$pkg\tools\aim\examples\aim-at.mjs" --scope desktop --max-width 1440 --color e8452c --min-area 300 --max-area 20000

# ③ 打一局自建靶场（dry-run，不拉窗口、不动鼠标）
& $node "$pkg\tools\aim\examples\play-arena.mjs" --cycles 4 --min-area 100

# ④ 真的动鼠标（桌面必须归你独占）
& $node "$pkg\tools\aim\examples\play-arena.mjs" --live --duration 10 --seed 20261107
```

dry-run 的自证方式（跑前跑后各读一次真实指针）：

```powershell
& $node "$pkg\tools\aim\input.mjs" --pointer   # 例：x=913 y=205；跑完还是同一个值就证明没发键鼠
```

## 3. 直接用 API（最小闭环）

核心契约只有一条：

```js
// screen = bounds.origin + imagePixel / imageScale
import { loadImage, detectTargets, resolveCapture, targetToClick } from '…/tools/aim/vision.mjs';
import { createEngine, readPointerPosition } from '…/tools/aim/input.mjs';

// A. 截图由 wcu 后端给（meta 里有 path / bounds / imageScale），vision.mjs 自己不带截屏能力
const capture = resolveCapture({ bounds: { x:0, y:0, width:1440, height:900 }, imageScale: 1 });

// B. 找靶：显式给颜色，别用 'auto'（整屏 auto 会把壁纸当靶）
const img = loadImage(shotPath);
const [t] = detectTargets(img, { color: 'e8452c', tolerance: 60, minArea: 300, maxArea: 20000, sort: 'area' });
const click = targetToClick(t, capture, null, { point: 'center', radiusScale: 0 });

// C. 动指针：live 之前先只读一次真实指针当锚点（否则首段会从屏幕左上角划一条长线）
const ptr = readPointerPosition({});
const engine = createEngine({ sender: 'mcp', rawKey: null, pointer: ptr, stepMs: 15, curvature: 0.18 });
await engine.moveTo(click.click.screenX, click.click.screenY, 320);
await engine.click(click.click.screenX, click.click.screenY, { durationMs: 180, settleMs: 30 });
engine.close();
```

| 想要的 | 用什么 |
|---|---|
| 平滑移动（无 duration 的后端） | `planMove({from,to,durationMs})` → 贝塞尔 + 缓动，多段发出 |
| 按住一段时间 | `keyHold(key, holdMs)`（走 `keybd_event`；没有 rawKey 会降级成只按一下并标 `degraded:true`） |
| 组合键 | `chord(['ctrl','s'], {holdMs})` / `planChord` |
| 连按 | `planKeyRepeat({key,count,intervalMs,holdMs})` |
| 整段定时序列 | `planSchedule(actions, [{anchor}])` + `engine.schedule(actions)`（各段起点链式衔接上一段落点） |
| 自测 | `& $node "$pkg\tools\aim\self-test.mjs"`（纯函数，虚拟时钟，不碰桌面） |

失败语义记住一条：**不抛异常，看 `_run` 的 `ok / failed / attempts / error`**，`ok = failed <= allowFailures`。

## 4. ⚠️ 三层延迟口径（写错这一条，所有数字作废）

`CONTROL-BENCH.md` 定的硬要求：定时序列偏差必须分**三层**报，三层数字要来自**同一批实验**（注明 run id 与数据文件）。

| 口径 | 定义 | 判据（同批实测） |
|---|---|---|
| ① 调度精度 | `(实际k−实际0) − (计划k−计划0)`，报 **\|dev\|** | 中位 ≤15ms、最大 ≤50ms（实测 11.260 / 42.102 ✅） |
| ② 注入延迟 | 「发出调用 → 目标收到」，同机 QPC，**不含调度等待**，按 键盘/鼠标 × 消息/raw 四类分列并报 loss | 键盘 med 1.414 / 鼠标 med 7.787 ❌（鼠标**未达标**）；raw 无数据 |
| ③ 端到端 | 靶场打点时刻 − 客户端计划时刻（跨进程/跨时钟） | 中位 ≤30ms、最大 ≤80ms（实测 28.525 / 67.025 ✅） |

**最容易犯的错**：把**插件整条调用耗时**（键 296–604ms / 鼠 73–446ms，前置成本占 99.5%）当投递延迟写进任一层。
它不是投递延迟，禁止出现在 ①②③ 里。只报其中一层也算不合格。

其他必须写清的口径：只报 `|dev|`（不含符号）；`find` 默认扫全树 0.3–0.9s，高频闭环必须收紧
`maxDepth/maxNodes/treeTimeoutMs`；截屏冷启动 ~1.9s、热 267–490ms，decode 1440px 103–159ms、
720px 31–40ms、360px 6–11ms —— 报"闭环频率"时必须说明画幅与降采样档。

## 5. 高频踩坑（都真踩过）

1. `color:'auto'` 在整屏截图上会猜出壁纸色 → 真机**显式给色**（靶场红 = `e8452c`）。
2. 同色小碎块是干扰（HUD 红字）→ 用 `--min-area` 压掉；`circularity` 在极小连通域上会 >1，别当唯一判据。
3. 降采样双刃剑：`--max-width 720` 让 decode 从 103ms 掉到 31ms，但 56px 的靶只剩 28px。
4. 拿图片像素当屏幕坐标只在 1:1 全屏截图时恰好成立；窗口裁剪/降采样后整体偏。
5. 真实点击会**抢前台**：目标被压住时报 `POINT_COVERED`；共用桌面跑 `--live` 会打断别人。
6. `createEngine({pointer:{x:0,y:0}})` 会让引擎认为锚点已知 → live 首段从左上角划长线。**live 一律 `pointer:null` 再只读一次**。
7. dry sender 拦不住原生按键通道：`createEngine({sender:'dry'})` 后 `keyHold('a')` 仍会真的按 A 键 —— 示例里显式传 `rawKey: null`。
8. `estimateActionMs('drag')` 少算 50ms，用它做预算会系统性偏乐观。

## 6. 边界（工程事实，不是免责声明）

真实指针通道就是 `SetCursorPos`/`SendInput` 级别的正常输入，**它不绕过任何反作弊**。
联机游戏的自动化判定是账号级不可逆的，拿它当靶子会污染自己的账号数据；
本插件全部验证都在自建靶场（`pc-elements` 技能里的 arena）与自己的测试机上完成。
