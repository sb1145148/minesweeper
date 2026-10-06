# 更新记录

本项目的版本划分：

| 版本 | 内容 | 文件 |
| --- | --- | --- |
| **v1.0** | 纯扫雷，无任何彩蛋代码 | `main` 分支的 `index.html` |
| **v2.0** | 在 v1.0 基础上增加「失败后自动跳转」彩蛋 | 本分支（`v2`）的 `index.html` |

两个分支的 `index.html` 是各自独立的完整单文件：`main` 分支那份与 v1.0 发布时字节级一致（sha256 `bd1400649722fc63`），从未被改动。

---

## v2.0 — 新增「失败后自动跳转」彩蛋

> 本分支的 `index.html` 在 v1.0 基础上 +140 行左右，独立自包含，不依赖 v1.0 的文件

### 一、新增了什么

**踩到雷之后，棋盘展示完爆炸现场，停留 1.5 秒，自动随机跳转到 7 条 b23.tv 链接中的一条。**

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
| 样式 | [index.html:164](index.html#L164) | 新增 `.cancel-jump` 按钮样式，以及状态栏倒计时时的呼吸动画 `ms-jump-pulse` |
| 结构 | [index.html:316](index.html#L316) | HUD 里新增 `<button id="cancelJump" hidden>⛔ 停止跳转</button>` |
| 配置 | [index.html:346](index.html#L346) | 新增 `REDIRECT` 配置对象 + URL 参数覆盖逻辑 |
| 状态 | [index.html:398](index.html#L398) | 新增终局用时快照 `finalElapsed` |
| 状态 | [index.html:402](index.html#L402) | 新增运行时变量：抽签袋、定时器、预开窗口等 |
| 开新局 | [index.html:519](index.html#L519) | `newGame()` 里调用 `cancelRedirect()`，重开/换难度即取消待跳转 |
| 失败钩子 | [index.html:635](index.html#L635) | `boom()` 末尾调用 `scheduleRedirect()`（仅新增一行） |
| 核心逻辑 | [index.html:639-728](index.html#L639-L728) | 新增 `pickRedirectUrl()` / `cancelRedirect()` / `scheduleRedirect()` / `doRedirect()` 四个函数 |
| 事件 | [index.html:821](index.html#L821) | 「停止跳转」按钮的点击处理 |
| 调试接口 | [index.html:898](index.html#L898) | `window.__minesweeper` 新增 `redirect`、`pendingJump()`、`cancelJump()` |

`boom()` 里只加了一行，其余全部是新增函数，**原有游戏逻辑一行未改**，所以扫雷本身的行为与 v1.0 完全一致。

### 三、配置项（在 `index.html` 顶部，直接改数字/字符串即可）

```js
var REDIRECT = {
  enabled: true,        // 总开关。改成 false 就完全没有跳转
  urls: [ /* 7 条链接 */ ],
  delayMs: 1500,        // 踩雷后停留多久再跳，单位毫秒（当前 1.5 秒）
  target: 'self',       // 'self' 当前标签页跳走 | 'blank' 新标签页打开
  shuffleBag: true,     // 抽签袋随机（见第五节）
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
| `?redirect=off` | 关闭跳转 | `.../v2/?redirect=off` |
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
3. 袋子空了再洗下一轮；洗新轮时如果袋尾（下一轮第一个要发的）与上一次相同，就把它换到袋内其他位置。

好处：既保证随机，又保证 **7 条链接在一轮内各出现一次**，而且**跨轮也不会连着两次跳到同一个链接**。纯随机会出现"连着三次都是同一个视频"的尴尬情况。

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
- 倒计时里的"用时"是**踩雷瞬间的冻结值**，不会随着倒计时重绘而继续增长（早期版本有这个问题，已修）。
- `target: 'blank'` 时会在**点击的同一个手势内**先把空白标签页开出来，倒计时结束后再让它导航——否则延迟打开的窗口会被浏览器弹窗拦截。
- 默认的 `target: 'self'` 走 `location.href`，属于正常页面导航，弹窗拦截器拦不住。

> ⚠️ 一点提醒：「停止跳转」按钮是故意留的（`cancellable: true`）。它是这个整活功能唯一的"逃生出口"，也让手机用户不至于被卡住。想要纯粹的整蛊效果，把它设为 `false` 即可——这个开关就在配置对象里。

### 七、回归测试

| 套件 | 结果 |
| --- | --- |
| v1.0 游戏本体 | ✅ 64/64 |
| v2.0 游戏本体 | ✅ 64/64 |
| v2.0 跳转彩蛋 | ✅ 40/40（随机性检查为 **28 次 / 4 轮**：每轮内部不重复 + 跨轮不重复） |
| 主题跟随 DSH 令牌 | ✅ |

跳转彩蛋部分覆盖：

| 检查项 | 结果 |
| --- | --- |
| `?redirect=off` 时踩雷不跳转、无按钮 | ✅ |
| 默认开启时的倒计时文案、按钮出现、秒数递减 | ✅ |
| 点「停止跳转」后取消，且不再离开页面 | ✅ |
| 「重新开始」/ 换难度清空待跳转 | ✅ |
| 胜利不触发跳转 | ✅ |
| 28 次跳转覆盖 7 条链接各 4 次、无连续重复 | ✅ |
| `?target=blank` 打开新标签页且原页保留 | ✅ |
| 真实导航到 b23.tv（拦截目标域名验证） | ✅ |
| `?cancel=off` 不显示按钮但照常跳转 | ✅ |
| 原有扫雷功能无回归 | ✅ |
