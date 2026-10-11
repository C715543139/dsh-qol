# 上一条按钮连续点只上移一小段（隐藏幽灵行）

日期：2026-10-12

## 现象

点击「上一条」有时只向上移动一小段；**继续点仍然只移动同样的一小段**（不是偶发，而是每次位移几乎相同），要一直点到更上面一条消息才恢复正常的逐条跳转。

## 根因

`jumpMessageRows()` 之前用裸选择器取候选行：

```js
scroller.querySelectorAll('[data-chat-flow-kind="user"], [data-chat-flow-kind="steering"]')
```

宿主会把**不显示但保留在 DOM 里**的行一起返回，例如折叠的 turn-process 成员、以及被 rewind 替换掉的历史（`hidden` / `hidden="until-found"`）。这类隐藏行没有盒模型：

- `display:none` 的元素 `getBoundingClientRect()` 恒为 `(0,0,0,0)`；
- 而 `topOf()` 的算法是 `rect.top - scRect.top + scrollTop`，于是该行算出来的“内容坐标”永远是 `scrollTop - scRect.top`（约等于视口顶部上方一个 header 高度），**与滚动位置无关**。

上一条的走位里，“已滚进该行 → 上滚对齐它的顶部”这一分支因此每次都以这个幽灵行为目标：落点 = `幽灵 top - 16`，即每次固定上移约 header 高度 + 16px。点多少次都是同一段位移，直到焦点行变化到别的行。

宿主自己的行运算就带 `:not([hidden]):not([hidden] *)` 过滤，本插件漏了这一步。

## 复现与验证（真实 DOM + 真实抽取函数）

在真实 Chromium DOM 里放 5 条可见用户行 + 1 条 `display:none` 的用户行（头部 77px，滚动容器 500px），从 `scrollTop=868` 连续调用 4 次从 `lib/client.js` 抽出的 `jumpToPrevUserMsg`：

| 版本 | 4 次位移 | 结果 |
| --- | --- | --- |
| 修复前 | `-93, -93, -93, -93` | 每次同样一小段，无限重复 |
| 修复后 | `-266, -306, -296, 0` | 正常逐条上跳并到顶 |

从 `scrollTop=500` 起同样是 `-204, -296, 0`（修复后）对照旧版 `-93, -93, -314, 0`。

## 修复

`jumpMessageRows()` 过滤掉没有实际盒子的行：

```js
if (el.closest("[hidden]") || el.getClientRects().length === 0) continue;
```

- `closest("[hidden]")` 覆盖宿主用 `hidden` / `hidden="until-found"` 隐藏的行（含隐藏祖先）；
- `getClientRects().length === 0` 覆盖一切 `display:none`（属性或 CSS 造成），从根上杜绝“零矩形幽灵行”。

与宿主自己的行查询过滤语义一致。

## 验收标准

1. 在被隐藏历史（rewind 替换）之后的区块连续点击「上一条」：每次都应跳到上一条用户消息，不应出现连续多次、每次只挪一小段的情况。
2. 常规长会话中「上一条 / 下一条」仍逐条移动并顶对齐。
3. `npm run build` 通过。
