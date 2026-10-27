# 侧栏滑关后点屏幕又自动弹出 修复总结 (261028-sidebar-mask-tap-reopen)

> 日期：2026-10-28
> 影响文件：`lib/client.js`

## 背景

移动端手势左滑收起侧边栏后，在屏幕上随便点一下，侧边栏常会自己重新弹出。

## 根本原因

抽屉遮罩（`.qol-drawer-mask`）在隐藏态只是 `opacity: 0`，仍然可被点击；它的移除定时器又会被 MutationObserver 的每一次 DOM 变动重置，页面持续更新时遮罩会一直留在 DOM 里。遮罩的 click 处理器使用 `toggleSidebar()`，于是任何落在残留遮罩上的点击都会把侧栏重新展开。

详见 [troubleshoot](261028-sidebar-mask-tap-reopen.troubleshoot.md)。

## 修复内容

- 遮罩隐藏态 `pointer-events: none`，仅 `.qol-mask-shown` 时恢复 `auto`；
- 遮罩点击从"切换"改为"收起"（`sidebarOpen()` 为真才关闭）；
- `syncMask()` 只在 shown→hidden 翻转时布置移除定时器，且打开时仅在遮罩确实隐藏时重放淡入。

修复保持最小化，仅动 `lib/client.js` 的遮罩 CSS 与 `installSidebarOverlay`，未改动手势状态机与其它功能。

## 验证

- 真实 Chromium 命中测试（抽离真实插件代码 + mock ctx）：修复前可稳定复现"点击残留遮罩 → 侧栏弹开"；修复后 7/7 通过（展开态遮罩拦截、遮罩点按关闭、滑关后点击不弹回、持续 DOM 变动下遮罩正常移除、右滑展开正常）。
- `npm run build` 通过。
- 真机行为待用户实测确认。
