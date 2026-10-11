# 上一条按钮连续点只上移一小段 — 总结

日期：2026-10-12

## 修复

`lib/client.js` 的 `jumpMessageRows()` 现在跳过宿主保留在 DOM 但不显示的行（折叠 process 成员、rewind 替换掉的历史）：

```js
if (el.closest("[hidden]") || el.getClientRects().length === 0) continue;
```

原因见 [troubleshoot](261012-jump-ghost-rows.troubleshoot.md)：隐藏行没有盒模型，`getBoundingClientRect()` 恒为 `(0,0,0,0)`，使 `topOf()` 变成一个与滚动位置无关的常数，上一条按钮每次都对齐同一个“幽灵行”，于是永远只上移同一小段。

## 验证

- 真实 DOM 复现（5 条可见用户行 + 1 条 `display:none` 用户行）：
  - 修复前连续 4 次位移 `-93, -93, -93, -93`（复现用户现象）；
  - 修复后 `-266, -306, -296, 0`（逐条上跳并到顶）。
- 既有走位断言未回归：`exact.mjs` 3/3（下一条的顶对齐落点）、`edge.mjs`（上下 clamp 边界）、`t.mjs` 其余 6 项。
- `npm run build` 通过。
- 待用户刷新 4180 实机确认。
