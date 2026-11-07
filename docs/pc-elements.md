---
name: pc-elements
description: 元素优先的 Windows 桌面自动化——用 winctl 的 UIA 定位 + 元素 invoke + 多通道校验，配套 selfcheck/bench 与收紧扫描的提速参数，附 10 个实战坑
whenToUse: 目标是标准控件、原生对话框、记事本/计算器这类有 UIA 树的应用，或需要「点完能自证生效」时
---

# pc-elements · 元素优先的桌面自动化

来自 `D:\dsh-tools\winctl\`（约 14KB 文档 + 62KB 基准脚本）的核心结论：**元素优先，坐标兜底；动作之后必须校验。**

## 1. 五条设计原则（每条都对应一次真实翻车）

| # | 原则 | 来自哪次翻车 |
|---|---|---|
| 1 | 元素优先，坐标兜底 | 盲点像素把 `dsh-video` 点成 `ecloud`（差 20px 点到隔壁行） |
| 2 | 真实点击前必须先激活窗口 | `ForegroundLockTimeout=200000`；一次真实点击落到盖在上面的资源管理器上，连点三下全空 |
| 3 | 打开目录用 invoke 或「选中+Enter」，不要模拟双击 | 模拟双击间隔 ≈500ms = 系统双击阈值 → 被当成两次单击，文件夹永远打不开 |
| 4 | 动作之后必须校验 | "命令发出去了" ≠ "生效了"；旧坐标通道自报 100%，实际一半任务没生效 |
| 5 | 校验通道要有优先级 | 记事本中文 OCR 把"探针"认成"深钅十"，而 UIA 文本是精确的；只信 OCR 会把成功判成失败 |

## 2. 最小可用流程（照抄）

```js
import { connect, windowBy } from '…/tools/winctl/mcp.mjs';
import { locate, waitForAnyText, activate, invokeElement, typeAndVerify, pressKeys } from '…/tools/winctl/index.mjs';

const client = await connect();                                  // 一次握手 ~180ms，长连接多次调用
const win = await windowBy(client, { title: '计算器' });          // 窗口枚举自带 1.5s 缓存
const btn = await locate(client, win.nativeWindowHandle, '等于'); // 精确优先、子串兜底、默认重试 8 次
await invokeElement(client, win.nativeWindowHandle, btn);         // 不依赖像素与前台
const ok = await waitForAnyText(client, win.nativeWindowHandle, '56'); // 多通道校验：UIA → OCR 拼接 → 标题
await client.close();
```

定位输入类控件的正确写法（别靠名字猜）：

```js
// 坑：查「文件名」命中的是旁边的标签（set_value 报"值为只读"），真正的输入框叫「文件名:」
await locate(client, hwnd, '文件名', { controlType: 'Edit', maxDepth: 5, maxNodes: 400, treeTimeoutMs: 1500 });
```

## 3. 先自检环境，再跑基准

```powershell
$node = 'C:\Users\35126\AppData\Roaming\dsh-desktop\harness\.desktop-bin\node.cmd'
$w = 'D:\dsh-plugin\tools\winctl'          # 装到 profile 后换成 node_modules\dsh-capability-lab\tools\winctl

& $node "$w\selfcheck.mjs"                        # 把隐含假设变成可读事实
& $node "$w\bench.mjs" --repeat 3                 # 全套 9 类任务（自动拉起记事本/计算器）
& $node "$w\bench.mjs" --only dialog-control      # 单任务
& $node "$w\bench.mjs" --only x --channel winctl  # 只跑 winctl 通道（隔离排查）
& $node "$w\diag.mjs"                             # 把元素名、OCR 原文原样打出来
```

基准自报口径（**成功率由基准统一裁决，不采信通道自报**）：

| 环节 | A 通道（坐标+不校验） | B 通道（winctl：元素+校验+重试） |
|---|---|---|
| 总体 | 71% / 1548ms | **100% / 3307ms** |
| 原生对话框写入 | **0%**（文字压根没落进输入框，却报 ok） | **100%** |
| 滚动后确认内容变化 | **0%**（scroll 调用 1ms 返回，界面没动，照样报 ok） | **100%** |

B 通道更慢是正常的：它把"确认真的生效"算进了耗时。

## 4. 提速：瓶颈是调用次数，不是等待逻辑

同一个原生对话框任务实测 **9.3s → 5.1s（快 45%）**，三招：

1. **收紧扫描范围**：`find` 默认 `treeTimeoutMs=5000` 且尽量扫全树（单次 0.3–0.9s 的主因）；
   浅层目标用 `maxDepth:5, maxNodes:400, treeTimeoutMs:1500` 就够。
2. **`controlType` 限定**（见第 2 节）。
3. **`set_value` + 服务端 `wait_for` 校验**：写值不用"点击聚焦+模拟键盘"；注意 `wait_for` 默认作用域是
   `active_window`，要么先激活目标窗口，要么显式 `scope:'desktop'`。

阶段耗时对照（同一台机器）：窗口枚举 1243ms；激活 1726→**0ms**（已在前台就跳过）；
定位输入框 2741→**1620ms**；写入 1123→**311ms**；校验 2646→**770ms**。

## 5. 十个实战坑（摘要，全文见 `tools/winctl/README.md`）

1. `waitForText` 逐词匹配 —— OCR 把中文按字切词（"自检"→"自"+"检"），永远匹配不上；改成整窗文字拼成一串再匹配。
2. 计算器查 `"7"` 命中的是显示屏"显示为 7"，真实元素名是中文 `七/八/乘以/等于`。
3. 只信 OCR 做校验 → 升级为多通道（UIA 文本 → OCR 拼接 → 窗口标题）。
4. 目标应用没开就跳过 → 加 `ensureApp()`，没有窗口就自己启动并等它出现。
5. 缓存门控引入的回归：OCR 改成"仅当 UIA 一无所获时才用"后，对话框任务 100%→0%（Edit 控件的值不在 UIA `Name` 里）→ 改成"常开 + 只做节流"。**优化必须复测。**
6. **并发不是提速手段**：`Promise.any` 并发两条等待条件反而更慢（4.7s），因为 MCP 传输是串行的 → 改成串行感知（零等待本地检查 → 单条件等 → 复检）。
7. 大头是每次调用本身 300–900ms（全树扫描），不是等待逻辑。
8. 子串匹配命中"错的同名邻居"（查 `上传` 命中侧栏「最近上传」）→ `locate()` 精确优先、子串兜底；
   且必须**先限定可点击元素（有 id），再挑精确匹配**（第一版顺序写反，把计算器从 100% 打到 0%）。
9. GPU 合成的 Flutter 窗口（天翼云盘）OCR 整条失效（截屏全黑），UIA 树间歇可读（有时 129 节点、有时 0）→ 点击发得出去但**无法校验**；改走非 UI 路径或要求人在屏幕前。
10. 控件名带标点且旁边有"看起来一样"的标签 → 用 `controlType:'Edit'` 限定。

## 6. 已知限制（如实）

* Flutter/Electron 自绘窗口 UIA 稀疏，首次查询常返回空节点，重试有时能唤醒，但不保证成功。
* Win11 资源管理器左树不通过 UIA 暴露 `TreeItem`（实测 0 个），`tree-expand` 任务在本机**自判无效**而非白送通过。
* Windows 键被 wcu 插件设计性屏蔽（Win+R / Win+D 做不到），只能换入口。
* 窗口不在前台时 UIA 树变窄 → `setValueVerified()` 逐级放宽（收紧扫描+控件类型 → 默认范围+控件类型 → 默认范围+名称）。
* 原生对话框 OCR 校验偏慢（~2.6s/次），下一步改事件驱动。
* 多显示器与 DPI：本机 1440×900 @96dpi（缩放 1.0），换算位保留但**未在缩放环境实测**。
* 基准覆盖 9 类任务（8 类可度量、1 类平台限制自判无效）；表格控件、拖拽、树控件仍未覆盖。
