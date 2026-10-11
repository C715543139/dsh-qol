# 检视报告

## 概要

检视范围：`lib/client.js`（FEATURES 定义 L81、CSS L812–886、JS L1779–2065 及 `onToggle` L2129–2136）、`README.md` / `README.en.md`（L25）、`docs/features/261011-jump-user-msg-pair/261011-jump-user-msg-pair.summary.md` 与 `.validation.md`。

整体评价：设计简洁克制，精准复用了既有 `.dsh-qol-jump-anchor` 零高 sticky 锚点、负 `margin-top` 精确抵消、水平内缩镜像（`syncJumpX`）等经过验证的机制；通过显式 `height: calc(2 * var(--dsh-qol-jump-half))` 防 `align-items: stretch` 压扁、用 `inset box-shadow` 替代 `border` 防 1px 高度溢出、用 `overflow: hidden` 裁剪胶囊圆角 hover 区域，几何与视觉细节考虑周全。代码无阻塞问题，存在若干边界细节与遗留清理建议。

## 需求对齐

- **上下连体按钮**：原有单个 `.dsh-qol-jump-user` 按钮已改造为 `.dsh-qol-jump-nav` 竖立胶囊容器（34×60px），内含上半 `.dsh-qol-jump-prev`（回到上一条用户消息）和下半 `.dsh-qol-jump-next`（回到下一条用户消息），完全符合需求。
- **宿主「回到底部」保持独立**：未合入宿主插槽，保留独立 sticky 锚点，宿主按钮可见时居其上方 8px，在底部 unmount 时回落占其原位（+16px），符合需求。
- **顶对齐与边界语义**：`jumpToPrevUserMsg` 保持原逻辑不变；新增 `jumpToNextUserMsg` 查找第一条 `top > viewTop + 16` 的用户消息并对齐到 `top - 16`，下方无用户消息或无用户行时滚到底部，含 `top <= viewTop` 的 down-only 守卫，与 prev 形成对称闭环。
- **文档同步**：`FEATURES` 文案、`README.md`、`README.en.md`、`summary.md`、`validation.md` 均已同步更新。

## 阻塞问题

无。

## 建议修改

| ID | 位置 | 问题 | 建议 |
| --- | --- | --- | --- |
| S-01 | `lib/client.js:2012, 2021` | `jumpToNextUserMsg()` 在「无用户消息行」或「下方已无用户消息」时调用 `scrollTo(scroller.scrollHeight)`，但缺少已在底部的短路守卫（对比 L2023 普通分支有 `if (top <= viewTop) return;`）。当视口已经在最底部（`viewTop >= scroller.scrollHeight - scroller.clientHeight - 1`）时，点击「下一条」仍会发起一次目标值等于 `scrollHeight` 的 `scrollTo({ behavior: "smooth" })` 调用。同时，传 `scroller.scrollHeight` 而非最大可滚位置 `floor = Math.max(0, scroller.scrollHeight - scroller.clientHeight)` 在部分浏览器实现下可能触发无意义的平滑滚动动画帧。 | 计算 `var floor = Math.max(0, scroller.scrollHeight - scroller.clientHeight);`，在滚到底部的两个分支中先判断 `if (viewTop >= floor - 1) return;`，再调用 `scrollTo(floor)`。 |
| S-02 | `lib/client.js:871`、`lib/client.js:318-323` | 当 `tap-feedback` 开启且在移动端（`@media (pointer: coarse)`）按压 `.dsh-qol-jump-nav button:active` 时，全局规则会对单个半胶囊 `<button>` 施加 `transform: scale(0.985) !important; filter: brightness(1.08) !important;`。由于下半按钮 `.dsh-qol-jump-next` 的中缝分隔线 `box-shadow: inset 0 1px 0` 画在按钮自身上，按压缩放可能让中缝或外缘在圆角胶囊内产生亚像素缩进/跳动，且 `corner-shape: round`（L872）写在无 `border-radius` 的子按钮上实为无效冗余。 | 移除 `.dsh-qol-jump-nav button` 上冗余的 `corner-shape: round;`（L872，外层 `.dsh-qol-jump-nav` 已有 `border-radius: 17px; overflow: hidden; corner-shape: round;`）；可选地为 `.dsh-qol-jump-nav button:active` 定制按压背景色或将分隔线改为伪元素/容器级绘制以保持连体按压时的视觉整体感。 |
| S-03 | `lib/client.js:2056-2062, 2129-2135` | 历史遗留但在本次触及区域旁：`installJumpUserMsg()` 注册了局部变量 `onResize` 监听 `window` 的 `resize` 事件，并在返回的 disposer 中移除；但 `onToggle("jump-user-msg", false)`（L2131–2135）手写了清理逻辑，**漏掉了 `window.removeEventListener("resize", onResize)`**，且再次打开开关又会再挂一个新的 `onResize` 造成监听器泄漏。 | 将 `onResize` 提升为模块级具名函数（如 `onJumpResize`），或在模块级保存 `_jumpDisposer`，在 `onToggle` 的 off 分支直接调用 `_jumpDisposer()`，消除重复清理代码与 `resize` 监听器泄漏。 |
| S-04 | `e2e/jump-user-msg.mjs:133`、`e2e/probe-chrome.mjs` | 已知未做项：现有 E2E 脚本仍在查询已删除的 `.dsh-qol-jump-user` 并断言 `height ≈ 34`，当前直接运行该 E2E 必败。 | 在合并或后续跑回归前，将选择器更新为 `.dsh-qol-jump-nav`（断言 `34×60`）及 `.dsh-qol-jump-prev` / `.dsh-qol-jump-next`（各 `34×30`），并补充下半按钮 `jumpToNextUserMsg` 的步进与到底部兜底断言。 |

## 非阻塞问题

| ID | 位置 | 问题 | 建议 |
| --- | --- | --- | --- |
| N-01 | `lib/client.js:1872-1873` | 按钮 `aria-label` 与 `title` 硬编码为中文 `"上一条用户消息"` / `"下一条用户消息"`（沿用原实现风格），英文界面下 tooltip 仍显示中文。 | 与插件内其它硬编码中文文案保持一致即可；若未来统一做 i18n 再一并提取。 |
| N-02 | `lib/client.js:1912, 1983, 2003` | `jumpToPrevUserMsg` 与 `jumpToNextUserMsg` 中重复编写了 `prefers-reduced-motion` 媒体查询检测与 `try { scroller.scrollTo(...) } catch (err) { scroller.scrollTop = top; }` 包装。 | 可提炼一个内部小辅助函数 `scrollChatTo(scroller, top)` 供两处共用，减少重复代码。 |

## 准入结论

**结论**：`条件准入`

**说明**：功能实现与需求完全对齐，CSS 零高锚点抵消与连体胶囊几何设计扎实，无阻塞缺陷。建议在合并前或后续迭代中顺手补上 `jumpToNextUserMsg` 到底部时的 no-op 守卫（S-01）并同步更新 E2E 脚本选择器（S-04）。

## 处理后记（作者回应，2026-10-11）

| ID | 处理 | 说明 |
| --- | --- | --- |
| S-01 | **已修** | `jumpToNextUserMsg` 改为先算 `floor = max(0, scrollHeight - clientHeight)`，两个"翻到底部"分支统一走 `scrollToBottom()`：`viewTop >= floor - 1` 直接 return，否则 `scrollChatTo(scroller, floor)`（不再传 `scrollHeight`）。 |
| S-02 | **部分采纳** | 已删除 `.dsh-qol-jump-nav button` 上冗余的 `corner-shape: round`（外层容器已持有）。按压 `scale(0.985)` 的亚像素缩进未另作处理：两半与容器同底色，`overflow:hidden` 内露出的仍是同色，视觉上不可见；为一枚连体控件定制 `:active` 会与全局 `tap-feedback` 规则打架，不划算。 |
| S-03 | **已修** | 新增模块级 `_jumpDisposer`，`installJumpUserMsg()` 把自己的 disposer 登记进去；`onToggle` off 分支改为 `_jumpDisposer()`，消除了重复清理代码与 `resize` 监听器泄漏（disposer 内置自注销，重入/重开幂等）。 |
| S-04 | **本次不做（用户决定）** | 用户 2026-10-11 明确选择"先不动 e2e"。失效点已记入 `summary.md` / `validation.md` 的待跟进项。 |
| N-01 | 不处理 | 沿用插件内其余硬编码中文文案的风格，i18n 统一时再提取。 |
| N-02 | **已修** | 提炼 `scrollChatTo(scroller, top)`（reduced-motion 判定 + `try/catch` 兜底）供 prev 两处、next 一处共用，prev 的走位算法本身未改。 |

改后复验：`npm run build`（`node --check`）通过；从 `lib/client.js` 抽取 `scrollChatTo` / `jumpToPrevUserMsg` / `jumpToNextUserMsg` 真实源码 + 假 scroller 做走位断言（8/8 通过）：到底 no-op、顶部落点、逐次步进 `184→684→1184→1684→2000`、prev 回环、无用户行的双向兜底。
