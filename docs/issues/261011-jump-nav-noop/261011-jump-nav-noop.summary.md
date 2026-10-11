# 连体消息导航按钮跳过 steering 消息 — 总结

日期：2026-10-11

## 修复

宿主把运行中插入或排队的用户输入标记为 `data-chat-flow-kind="steering"`，插件原先只选择 `data-chat-flow-kind="user"`，导致该消息从上一条和下一条两个方向都不进入候选序列。现新增 `jumpMessageRows(scroller)`，统一选择 `user` 与 `steering`，prev/next 共用。

## 验证

- `npm run build` 通过。
- 静态回归确认两个导航方向都调用 `jumpMessageRows(scroller)`，选择器同时包含 `user` 与 `steering`。
- 待用户刷新 4180 后确认此前被跳过的 `# UI Debug Snapshot` 消息可从上下两个方向到达。
