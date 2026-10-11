# 上一条/下一条用户消息连体按钮 — 总结

日期：2026-10-11 · 状态：实现完成，静态校验通过，待用户实机验证

## 需求（用户 2026-10-11 提出）

> 现在右上（实为右下）那个回到上一条，把它改造成一个上下连体按钮，上边上半是回到上一条，下半是回到下一条。系统自带的那个"回到底部"还是留单独。

要点：
- 原来单个圆形「上一条」按钮 → 改为**上下连体**两段：上半 = 上一条，下半 = 下一条。
- 宿主自带的「回到底部」**保持独立**，不与我们的按钮合并。

## 交付内容（全部在 `lib/client.js`）

局部改动，沿用 `jump-user-msg` 这个 feature id / 开关 / 属性闸门 `html[data-qol-jump-user-msg]`：

1. **DOM 结构**：`div.dsh-qol-jump-anchor`（sticky 定位锚点，机制与几何完全不变）内由单个 `button.dsh-qol-jump-user` 换成 `div.dsh-qol-jump-nav`（连体容器）+ 两个按钮 `.dsh-qol-jump-prev` / `.dsh-qol-jump-next`。二者共用 `makeJumpHalf(cls, label)` 工厂 + `JUMP_CHEVRON_SVG` 常量（上 chevron，下一半 CSS `transform: rotate(180deg)` 翻转）。
2. **CSS 连体胶囊**：`--dsh-qol-jump-half: 30px`；容器 34px 宽 × `calc(2 * half)` = 60px 高、`flex-direction: column`、`border-radius: 17px`、`overflow: hidden`，上半 30px 高、下半 30px 高 —— 合起来是一个竖立胶囊，读作一个控件。
   - **负边距必须抵消容器真实高度**：锚点是 `height: 0` 的 sticky flex 容器，原来靠子元素 `margin-top: -34px` 把自己拉出零高盒子；现在改成 `margin-top: calc(-2 * var(--dsh-qol-jump-half))`。**同时必须给容器显式 height**（`calc(2 * var(--dsh-qol-jump-half))`）——否则容器高 `auto` 会被锚点的 `align-items: stretch` 拉成 0 高，配合 `overflow: hidden` 直接压扁（原单按钮靠显式 34px 高天然免疫）。
   - **分隔细线用 inset box-shadow，不用 border**：border 会占 1px 布局，破坏上面 `-2 * half` 的抵消（总高变 61px）。`.dsh-qol-jump-next { box-shadow: inset 0 1px 0 var(--dsw-alias-border-l3) }`。
3. **布局不变式**：锚点 `bottom: calc(composer + 58px)` = 预留宿主按钮一整格（16px 下边距 + 34px 按钮 + 8px 间距）。**该偏移恒定**，宿主按钮在不在下方都一样（见下方"追加"）；`syncJumpX()` 实测水平内缩量镜像为 `padding-right`。
4. **「下一条」定位 `jumpToNextUserMsg`**（与 prev 的当前行语义镜像）：
   - 当前行优先取顶对齐行（top ∈ `viewTop ± 16`），否则取视口中心之上的最后一条用户消息（阅读焦点）；目标永远是当前行的**后一条**。
   - 不再使用“第一条 top > `viewTop + 16`”作为目标。旧规则会在当前消息刚好位于视口顶部下方 30~50px 时只做 17~34px 微移，看起来像按钮无反应。
   - 没有中心之上的消息时，第一条用户消息是目标；越过最后一条时翻到**最底部**（镜像 prev 越过首行翻到最顶）。
   - DOM 里没有用户消息行（懒加载窗口 / 纯 agent 会话）→ 同样翻到最底部；密集行仍跳过会 clamp 的落点。
5. **点击分派**：`onJumpClick` 改用 `closest(".dsh-qol-jump-nav button")`，按 `classList.contains("dsh-qol-jump-next")` 分派 next / prev；capture 阶段监听、`isOn` 守卫、`isOn` off 时移除监听与按钮的逻辑不变。
6. **文案**：FEATURES 行 label 改「上一条/下一条用户消息」；`README.md` / `README.en.md` 对应行同步。

## 宿主事实（沿用，未变）

- 锚点必须直挂 `[data-conversation-scroll]`（`--dsh-composer-height` 只在该子树可见）。
- 宿主「回到底部」按钮：插槽 `[class$="_toBottomSlot"]` + `button[class$="_toBottom"]`，`!atBottom` 才渲染；CSS Modules hash 前缀随版本变，一律后缀匹配 + aria-label 兜底。
- 用户可见消息 DOM：`[data-chat-flow-kind="user"]` 与运行中插入/排队输入的 `[data-chat-flow-kind="steering"]`；导航两者都纳入候选。

## 静态验证（已完成）

- `npm run build`（`node --check lib/client.js` + `index.js`）通过，无语法错误。
- **静态几何夹具复核**（复刻宿主 slot/button 结构 + 本插件 CSS，google-chrome 无头）：nav 34×60、上下半各 34×30、nav 底边距宿主按钮顶边 **8px**、右边缘与宿主按钮对齐（`navRight == hostRight`）、无 pageerror。截图确认为竖立胶囊 + 中间细线 + 上下 chevron。
  - 夹具与截图：`preview-static-harness.html` / `preview-static-harness.png`（**静态夹具渲染，非真机截图**）。
- **走位逻辑断言**（node 脚本从 `lib/client.js` 抽取 `scrollChatTo` / `jumpToPrevUserMsg` / `jumpToNextUserMsg` 真实源码 + 假 scroller 驱动，8/8 通过）：到底 no-op、顶部落到第一条线下行、逐次步进 `184→684→1184→1684→2000`（每次恰好一条、末尾落底）、prev 顶对齐回环、两方向"无用户行"兜底。脚本为一次性临时件（未入库）。

## 追加：为宿主「回到底部」预留位置，消除位置跳动（用户 2026-10-11 反馈）

> 现在留那个系统的回到底部，它在我往上滑了一段时间才出现，然后它那个位置就会跳动，然后就会导致用户操作可能会误触。

现象与根因：宿主「回到底部」是 `!atBottom` 才挂载的（往上滑一点才出现、滚回底部就整个 unmount）。旧 CSS 用 `:not(:has([class$="_toBottomSlot"]))` 跟着它切换锚点偏移 —— 宿主按钮在时 `+58px`、宿主按钮没了掉到 `+16px`。于是**在底部附近来回滚动时，我们的连体按钮会在 42px 的两个位置间瞬间跳变**，手指正好瞄着旧位置就会点空或点到别的按钮（宿主按钮本身也在这一刻出现/消失，两个控件同时换位）。

修法：**恒定预留宿主按钮那一格**——删掉 `:has` 那条切换规则，锚点永远 `bottom: calc(composer + 58px)`。宿主按钮在时仍精确保持 8px 间距；宿主按钮不在（已在底部）时，连体按钮停在原位不动，下方留出的是宿主按钮的那 42px 空档。代价是滚到底部时按钮下方多出一段留白，换来的是一次都不跳。

顺带收益：不再依赖 `:has([class$="_toBottomSlot"])` 这个宿主 CSS Modules 类名后缀（宿主改 hash 时旧写法会静默退化到 +16px 并与宿主按钮重叠），少一个版本脆弱点。

复核（静态夹具，同一页面内先量后删槽）：槽在位 `gap = 8px`（`navBottom 411 / hostTop 419`）；JS 删掉 `.EvIC1a_toBottomSlot` 后 `navTop/navBottom` 完全不变（351/411）。

## 检视与处理后（reviewer，条件准入）

`261011-jump-user-msg-pair.review.md`：0 阻塞 / 4 建议 / 2 非阻塞。已处理 S-01（到底 no-op 守卫，改传 `floor`）、S-02（删掉子按钮上冗余的 `corner-shape`；按压亚像素缩放判定为不可见、不特殊处理）、S-03（新增 `_jumpDisposer`，`onToggle` off 复用同一 disposer，修掉历史遗留的 `resize` 监听泄漏）、N-02（提炼 `scrollChatTo` 共用）；S-04 按用户决定本次不做，N-01 不处理。详见 review 文末"处理后记"。

## 未做（用户已拍板）

- **e2e 本次不动**（用户 2026-10-11 选择"先不动 e2e"）：`e2e/jump-user-msg.mjs` / `e2e/probe-chrome.mjs` 仍在查旧类名 `.dsh-qol-jump-user`，其中的 34×34 圆形与 8px 间距断言按旧结构编写，直接跑必败。下次动 e2e 时需更新为 `.dsh-qol-jump-nav`（34×60）+ `.dsh-qol-jump-prev` / `.dsh-qol-jump-next`（各 34×30），并补「下一条」的步进与到底兜底断言。
