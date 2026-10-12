# 侧栏手势状态残留误开侧栏 修复总结 (261012-sidebar-gesture-stale)

> 日期：2026-10-12
> 影响文件：`lib/client.js`（`installGesture`）

## 背景

偶现问题：手势滑关侧边栏后，手势状态未正常结束，之后在界面上任意拖动会把侧边栏重新打开。怀疑与卡顿导致触摸事件丢失有关。

## 根本原因

`installGesture` 的 `drag` 状态只有 touchend / touchcancel 一条结束路径，而 `onTouchStart` 在多条跳过分支上会提前 return（不清理旧状态），`onTouchMove` 又只比对会复用的 `identifier` 且不校验功能开关。因此一次丢失的结尾事件会让手势永久武装，后续任意一次拖动都以**旧起点**计算位移，轻易越过 64px 阈值触发开合。

详见 [troubleshoot](261012-sidebar-gesture-stale.troubleshoot.md)。

## 修复内容

`lib/client.js` 中 `installGesture` 改为事件驱动的活性校验：

- 新增 `endGesture()` 统一收口状态清理；
- `onTouchStart` 第一行无条件结束上一次手势（新触摸即代表旧手势已终结，无论本次触摸是否被跳过）；
- `onTouchMove` 要求「仍只有本手势这一根手指 + identifier 匹配 + 功能仍开启」，否则丢弃手势并返回，绝不拿旧起点参与计算；
- 新增 `blur` / `pagehide` / `visibilitychange` 清理，覆盖切后台不给结尾事件的路径；
- disposer 注销新增监听并清空状态。

未引入超时/时间窗等启发式规则，避免误伤慢速滑动的正常手势。

## 验证

- 状态机回放（抽离真实 `installGesture` 源码 + 桩 DOM，含"不派发 touchend"的丢事件模拟）：修复前 3 项失败 / 8 项通过；修复后 **12/12 通过**（3 项丢事件用例 + 1 项切后台用例 + 8 项既有行为回归：右滑展开、左滑收起、反向不触发、竖直不抢滚动、阈值内只捕获不触发、左边缘让位系统手势、输入框不武装手势、连续两次滑动正常）。
- `npm run build`（`node --check lib/index.js && node --check lib/client.js`）通过。
- 真机现象待用户实测确认（详见验收标准）。
