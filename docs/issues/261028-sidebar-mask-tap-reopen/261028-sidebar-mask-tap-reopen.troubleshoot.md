# 侧栏滑关后点屏幕又自动弹出 排查 (261028-sidebar-mask-tap-reopen)

> 日期：2026-10-28
> 状态：根因已锁定并复现，修复已验证

## 1. 现象

移动端实测：用手势左滑收起侧边栏后，在屏幕任意位置点一下，侧边栏又自己弹出来（非每次，但很常见）。用户操作路径里并没有再次触发右滑手势。

## 2. 复现

抽离真实插件代码到空白移动端页面（mock `ctx.layout.toggleSidebar` + 合成 AppFrame），用真实 Chromium 做命中和点击，不依赖 e2e 实例、不产生模型调用：

```bash
node /tmp/repro-mask.mjs
```

修复前输出（节选）：

```
after open:                             toggles=1 mask=true  shown=true
30ms after close:                       toggles=2 mask=true  shown=false pe=auto
...连续 DOM 变动 900ms 后:              toggles=4 mask=true  shown=false pe=auto hit=qol-drawer-mask
after tap under churn:                  toggles=5 mask=true  shown=true   ← 点屏幕把侧栏点开了
```

`hit=qol-drawer-mask` 表明点击命中的是遮罩；一次 tap 后 `collapsed` 从 true 翻回 false。

## 3. 根因

两个缺陷叠加，都在 `installSidebarOverlay`：

1. **隐藏态遮罩仍然可点。** `.qol-drawer-mask` 只有 `opacity: 0`，没有 `pointer-events: none`。`opacity: 0` 的元素依然参与命中测试（`display:none` / `visibility:hidden` / `pointer-events:none` 才不参与）。抽屉关闭后遮罩要留到淡出结束，这段时间它是一张全屏、看不见、可点击的膜。
2. **移除定时器会被 DOM 变动无限推迟。** `syncMask()` 是 MutationObserver 的回调，关闭态每进一次都会 `clearTimeout` 再重设 230ms 移除定时器。页面只要持续变动（会话流式输出、列表/状态更新），遮罩就永远removal不掉。

遮罩的 click 处理器又是 `toggleSidebar()`。于是：任意一次点击落在残留遮罩上 → 直接 toggle → 侧栏弹开。这就是"划走之后随便点一下又出来"。

补充：滑关动画的前 ~180ms 内，左半屏命中的还是正在退出的侧栏列（z-index 900 > 遮罩 890），只有右侧区域会命中遮罩；所以现象表现为"有时点一下才复现"，与用户描述的"非常常见"一致。

## 4. 修复

`lib/client.js`：

- CSS：`.qol-drawer-mask` 默认 `pointer-events: none`，只有 `.qol-mask-shown` 时恢复 `auto`。隐藏/淡出中的遮罩不再吃任何点击。
- 点击语义改为"关闭"而非"切换"：`if (sidebarOpen()) toggleSidebar(_maskCtx)`。遮罩的含义是"点抽屉外收起"；若一次点击在抽屉已关闭后才落地，不应反向打开。
- `syncMask()` 只在 `shown → hidden` 的那一次翻转时布置移除定时器，避免每次 DOM 变动重置计时；打开分支同样只在真正隐藏时重放 reflow，避免遮罩在每次页面变动时重新淡入闪一下。

## 5. 验证

`node /tmp/verify-mask.mjs`（真实 Chromium 命中测试）修复后 7/7：

- 展开时遮罩显示且拦截内容点击；
- 点击可见遮罩能关闭抽屉；
- 滑关后 30ms 内点击不再命中遮罩、不再弹回；
- 持续 DOM 变动下遮罩仍会在淡出后移除；
- 变动场景下点击不弹回；
- 右滑重新展开仍正常。

`npm run build` 通过。真机验收见 [validation](261028-sidebar-mask-tap-reopen.validation.md)。
