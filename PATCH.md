# 本 fork 的改动（源码级）

基于上游 `Imzl-zl/dsh-redo` **v0.4.1**（MIT，作者 Imzl-zl）。改动全部落在 `src/`，`lib/` 是
`npm run build` 的产物，**不提交**（与上游一致，`.gitignore` 已忽略）。

## 1. 新增：「取消重写」

点 ✎（编辑重发）之后，原来的实现会把那条消息**立刻从界面上隐藏**（`rangeFeed.markPending`），
于是"放弃这次编辑"没有任何入口——消息没了，输入框里是回填的原文，只能硬着头皮发出去。

现在：

- 进入重写后 **消息行保持可见**（不再本地隐藏）；
- 该消息的动作行末尾出现 **「取消重写」**；
- 点击 = 清空输入框 + `armedSet(sessionId, undefined)` + 给宿主发 `/redo off`（宿主本来就支持该命令），
  消息原样留在时间线里。

涉及文件：

| 文件 | 改动 |
|---|---|
| `src/client/index.tsx` | 新增 `ARMED` 外部 store（`armedGet/armedSet/armedSubscribe/armedRevision`）；`arm()` 不再调 `markPending`；`RangeFeed` 增加 `cancelPending`；动作行条件渲染取消按钮；注入 `isPending / cancelArm / subscribeArmed / armedTarget` |
| `src/client/locales.ts` | 新增 `action.cancel`（`取消重写` / `Cancel rewrite`） |

## 2. 回移上游 0.4.3 的修复（transcript 不再被清空）

上游 0.4.3（Unreleased 段，issue #1）修复了：把**所有** `surfaceOp: replace` 都当成自己的撤回区间，
于是压缩（compaction）、system-prompt 刷新、结果裁剪、上下文注入写出的 replace 也会把普通 Turn 隐藏，
长会话压缩后整段对话变空。

该修复与新版附件 API 无关，**可以安全回移到 0.4.1**：

- `src/client/intervals.ts` 换成 0.4.3 版本（新增 `isRedoRewrite`：只有空 content 的 `system/message` 才算本插件的撤回）；
- `test/intervals.test.ts` 换成 0.4.3 版本，锁定"harness 的 replace 不隐藏 Turn"。

## 3. 跨版本附件读取（0.2.1 ↔ 0.2.2）

上游 0.4.2 的图片修复依赖 `ImageAttachmentRef`，而 0.2.1 只认 attachment id。本 fork 不抬
`engines.dsh`，改成运行时两代兼容：

```ts
const viaRef = await session.readAttachment(image)      // 0.2.2-rc.1+
if (viaRef.ok) return viaRef
return session.readAttachment(image.attachmentId)       // 0.2.1 回退
```

`arm()`（编辑回填图片草稿）与 `resend()`（重新生成带图）两条路径都走这个 helper。

## 4. 为什么没有直接升到 0.4.3

上游 0.4.2 把 `engines.dsh` 抬到了 `>=0.2.2-rc.1`，原因是 `ImageAttachmentRef` 的附件读取 API 只在
0.2.2-rc.1 起才完整（旧版只接受 attachmentId）。而当前宿主是 **DSH 0.2.1-rc.2**。

所以本 fork 的策略是：

- 保持 `engines.dsh: >=0.2.1-rc.2 <0.3.0`（与 0.4.1 一致）；
- 只回移与附件 API 无关的 0.4.3 区间修复；
- **不**回移 0.4.2 的图片重读修复——那需要升级宿主，等 DSH 升到 0.2.2-rc.1 再跟随。

## 5. 构建与验证

```sh
npm install
npm run typecheck   # 通过
npm test            # 37 项全过（0.4.1 的 34 + 回移的 3 项区间用例）
npm run build       # 产出 lib/index.js + lib/client.js（不提交）
```

本 fork 实际安装用的构建：`lib/client.js` 39,114 字节，含 `isRedoRewrite`、`cancelArm`、`cancelPending`；
包版本标记为 `0.4.1-fork.2`。

## 6. 已知限制

- 输入框已有草稿、或该消息之后仍有轮次时，点 ✎ 仍会先弹上游自带的确认框；确认后才进入"已武装"状态，
  才会出现「取消重写」（这是上游既有交互，本次未改动）。
- 「取消重写」只清武装与输入框，不会恢复已被重发覆盖的旧内容（superseded 语义属上游设计）。
- 未处理：`InputActions` 上没有 `focus()`，取消后焦点不会自动回到输入框。
