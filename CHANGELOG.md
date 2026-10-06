# 更新记录

## v2.1 — 跳转停留时间改为 5 秒 + 两处修复

> 改动文件：`prank/index.html`、`README.md`、`CHANGELOG.md`（原版 `index.html` 未改动）

### 1. 停留时间 1.5 秒 → 5 秒

- `prank/index.html` 里的 `REDIRECT.delayMs` 由 `1500` 改为 `5000`（见 [prank/index.html:357](prank/index.html#L357)）。
- 效果：踩雷后棋盘停留 **5 秒**，状态栏从 `5.0 秒后自动跳转…` 倒数到 `0.0 秒后自动跳转…`，再执行跳转。

### 2. 修复：倒计时期间"用时"会继续往上跳

上一版里，状态栏的倒计时每 100ms 重绘一次，而用时是实时算的，导致踩雷后显示的用时还在涨
（例如明明只玩了 2 秒，倒计时到一半却显示"用时 5 秒"）。

改用**终局快照** `finalElapsed`：在 `boom()` / `checkWin()` 里把用时冻结下来，之后所有重绘都读这个快照。
现在倒计时全过程用时保持不变，`__minesweeper.state().elapsed` 在终局后也不再增长。

### 3. 修复：抽签袋"跨轮"仍可能连续重复

v2 的文档里写了"绝不连续跳到同一个链接"，但抽签袋只保证**一轮内**不重复，
上一轮的最后一个和下一轮的第一个仍有 1/7 的概率撞上——这是文档的过度承诺，属于真 bug。

现在洗新的一轮时，如果**袋尾（即下一轮第一个要发出的）**与上一轮最后一个相同，就把它和袋内随机一个位置交换。
这样连续两次相同的情况在**跨轮边界上也被消除**，文档里的说法才真正成立。

### 4. 其他

- 想临时改时长不用动代码，网址后面加参数即可，例如 `?delay=2000`。
- 抽签袋随机、取消按钮、`?target=blank`、URL 参数等其余行为不变。

### 5. 回归结果

| 套件 | 结果 |
| --- | --- |
| 游戏本体 | ✅ 64/64 |
| 跳转彩蛋 | ✅ 40/40（随机性检查加强到 **28 次 / 4 轮**，每轮内部不重复 + 跨轮不重复） |

## v2 — 新增「失败后自动跳转」彩蛋

> 改动文件：`prank/index.html`（在原版 `index.html` 的基础上新增，共 +140 行左右）
> 仓库结构：原版 `index.html` 保持字节级不变，彩蛋版独立放在 `prank/` 子目录
> 回归结果（v2 当时的版本）：游戏本体 64 项 + 跳转彩蛋 41 项 = **105 项断言全部通过**（v2.1 调整后为 64 + 40）

### 一、新增了什么

**踩到雷之后，棋盘展示完爆炸现场，隔一小段时间自动随机跳转到 7 条 b23.tv 链接中的一条。**

新增的 7 条链接（顺序即 `REDIRECT.urls` 数组顺序）：

| # | 短链 |
| --- | --- |
| 1 | https://b23.tv/gJKzzAV |
| 2 | https://b23.tv/J72tQfZ |
| 3 | https://b23.tv/tzoU9cp |
| 4 | https://b23.tv/aHy587q |
| 5 | https://b23.tv/zbn0prs |
| 6 | https://b23.tv/JDrqc8U |
| 7 | https://b23.tv/niH5vnR |

7 条短链均已实测可正常解析（HTTP 200，分别指向 7 个不同的 B 站视频）。

### 二、具体改了什么

| 位置 | 行号 | 内容 |
| --- | --- | --- |
| 样式 | [prank/index.html:164](prank/index.html#L164) | 新增 `.cancel-jump` 按钮样式，以及状态栏倒计时时的呼吸动画 `ms-jump-pulse` |
| 结构 | [prank/index.html:316](prank/index.html#L316) | HUD 里新增 `<button id="cancelJump" hidden>⛔ 停止跳转</button>` |
| 配置 | [prank/index.html:346](prank/index.html#L346) | 新增 `REDIRECT` 配置对象 + URL 参数覆盖逻辑 |
| 状态 | [prank/index.html:402](prank/index.html#L402) | 新增运行时变量：抽签袋、定时器、预开窗口等 |
| 开新局 | [prank/index.html:519](prank/index.html#L519) | `newGame()` 里调用 `cancelRedirect()`，重开/换难度即取消待跳转 |
| 失败钩子 | [prank/index.html:635](prank/index.html#L635) | `boom()` 末尾调用 `scheduleRedirect()` |
| 核心逻辑 | [prank/index.html:639-728](prank/index.html#L639-L728) | 新增 `pickRedirectUrl()` / `cancelRedirect()` / `scheduleRedirect()` / `doRedirect()` 四个函数 |
| 事件 | [prank/index.html:821](prank/index.html#L821) | 「停止跳转」按钮的点击处理 |
| 调试接口 | [prank/index.html:898](prank/index.html#L898) | `window.__minesweeper` 新增 `redirect`、`pendingJump()`、`cancelJump()` |

`boom()` 里只加了一行，其余全部是新增函数，**原有游戏逻辑一行未改**，所以扫雷本身的行为与 v1 完全一致。

### 三、配置项（都在 `index.html` 顶部，直接改数字/字符串即可）

```js
var REDIRECT = {
  enabled: true,        // 总开关。改成 false 就完全没有跳转
  urls: [ /* 7 条链接 */ ],
  delayMs: 5000,        // 踩雷后停留多久再跳，单位毫秒（v2.1 起为 5 秒，v2 时为 1500）
  target: 'self',       // 'self' 当前标签页跳走 | 'blank' 新标签页打开
  shuffleBag: true,     // 抽签袋随机（见下）
  cancellable: true     // 是否显示「停止跳转」按钮
};
```

几条最常改的：

- **想更狠**：`cancellable: false`（不给取消按钮）+ `delayMs: 600`（几乎秒跳）。
- **想温和一点**：`target: 'blank'`，只开新标签页，玩家原来的游戏页面还在。
- **想彻底关掉**：`enabled: false`。

### 四、URL 参数（不用改代码，加在网址后面即可）

| 参数 | 作用 | 例子 |
| --- | --- | --- |
| `?redirect=off` | 关闭跳转 | `.../minesweeper/?redirect=off` |
| `?redirect=on` | 强制开启（覆盖默认关闭） | |
| `?delay=3000` | 改停留时长 | `?delay=3000` |
| `?target=blank` | 新标签页打开 | `?target=blank` |
| `?cancel=off` | 不给取消按钮 | `?cancel=off` |

可以组合：`?target=blank&delay=800&cancel=off`。
分享"安全版"给不想被整的人时，发带 `?redirect=off` 的链接就行。

### 五、随机是怎么做的（"随机播放"）

用的是**抽签袋（shuffle bag）**，不是每次独立随机：

1. 每次需要跳转时，如果袋子空了，就把 7 条链接**洗牌一次**装进袋子；
2. 从袋子末尾取一条（`pop`），取走就不再放回；
3. 袋子空了再洗下一轮。

好处：既保证随机，又保证 **7 条链接在一轮内各出现一次**；v2.1 起进一步保证**跨轮也不会连着两次跳到同一个链接**。纯随机会出现"连着三次都是同一个视频"的尴尬情况。

实测 28 次（4 轮）的抽签结果，每轮内部不重复，且轮与轮的交界处也不重复：

```
第 1 轮： zbn0prs → niH5vnR → gJKzzAV → aHy587q → JDrqc8U → J72tQfZ → tzoU9cp
第 2 轮： gJKzzAV → aHy587q → zbn0prs → tzoU9cp → niH5vnR → J72tQfZ → JDrqc8U
第 3 轮： gJKzzAV → J72tQfZ → JDrqc8U → aHy587q → tzoU9cp → niH5vnR → zbn0prs
第 4 轮： J72tQfZ → niH5vnR → JDrqc8U → tzoU9cp → aHy587q → zbn0prs → gJKzzAV
```

### 六、交互细节

- 踩雷后状态栏实时显示倒计时：`💥 踩到雷了！用时 12 秒 —— 1.4 秒后自动跳转…`，并伴随呼吸闪烁，旁边出现「⛔ 停止跳转」。
- **只在失败时触发**，胜利永远不跳。
- 点「重新开始」或切换难度会自动取消未执行的跳转。
- `target: 'blank'` 时会在**点击的同一个手势内**先把空白标签页开出来，倒计时结束后再让它导航——否则延迟打开的窗口会被浏览器弹窗拦截。
- 默认的 `target: 'self'` 走 `location.href`，属于正常页面导航，弹窗拦截器拦不住。

> ⚠️ 一点提醒：「停止跳转」按钮是故意留的（`cancellable: true`）。它是这个整活功能唯一的"逃生出口"，也让手机用户不至于被卡住。想要纯粹的整蛊效果，把它设为 `false` 即可——这个开关就在配置对象里。

### 七、回归测试

新增 41 项断言全部通过（v2.1 加强随机性检查后为 40 项），覆盖：

| 检查项 | 结果 |
| --- | --- |
| `?redirect=off` 时踩雷不跳转、无按钮 | ✅ |
| 默认开启时的倒计时文案、按钮出现、秒数递减 | ✅ |
| 点「停止跳转」后取消，且不再离开页面 | ✅ |
| 「重新开始」/ 换难度清空待跳转 | ✅ |
| 胜利不触发跳转 | ✅ |
| 14 次跳转覆盖 7 条链接各 2 次、无连续重复 | ✅ |
| `?target=blank` 打开新标签页且原页保留 | ✅ |
| 真实导航到 b23.tv（拦截目标域名验证） | ✅ |
| `?cancel=off` 不显示按钮但照常跳转 | ✅ |
| 原有扫雷功能无回归（64 项） | ✅ |
