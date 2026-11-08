# dsh-redo（YZKZK01 fork）

> 基于 [Imzl-zl/dsh-redo](https://github.com/Imzl-zl/dsh-redo)（MIT，作者 Imzl-zl）的**自维护 fork**。
> 给发出的每条消息加上「编辑重发 / 重新生成」，并把**放弃编辑**的入口补上。
> 上游仓库与本 fork 的差异见下表；本仓库的改动全部落在 `src/`，`lib/` 是构建产物、不提交。

---

## 这个 fork 做了什么

| 能力 | 上游 0.4.1 | 本 fork |
|---|---|---|
| 编辑重发（撤回后从原位置重写） | ✅ | ✅ |
| 重新生成（原样重发，不经过输入框） | ✅ | ✅ |
| **取消重写**（点 ✎ 后反悔） | ❌ 消息行立刻隐藏，没有返回入口 | ✅ **消息行保持可见 + 动作行出现「取消重写」** |
| 长会话压缩后整段对话空白 | ❌（issue #1） | ✅ 回移 0.4.3 的区间修复 |
| 图片消息编辑/重试 | 0.2.1 上读不回（只传 id） | ✅ 双代兼容读取（ref → 失败回退 id） |

### 「取消重写」怎么用

1. 在消息动作行点 ✎（编辑重发）；
2. 消息**不会消失**，动作行末尾出现 **「取消重写」**，原文已回填输入框；
3. 点「取消重写」= 清空输入框 + 撤销宿主侧的 `/redo` 登记，消息原样留在时间线里；
4. 若直接改完发送，则按上游语义在该位置重写对话。

---

## 兼容性

| 宿主版本 | 状态 | 说明 |
|---|---|---|
| **DSH `0.2.1-rc.2`** | ✅ 主力验证 | 附件按 id 读取（回退路径） |
| **DSH `0.2.2-rc.1`** | ✅ 支持 | 附件按完整 `ImageAttachmentRef` 读取（优先路径） |
| 更早（`<0.2.1-rc.2`） | ❌ | `renderFactorySlot` / `UserContentData` 不存在 |
| 更新版本 | 未验证 | 见下方「维护节奏」 |

`package.json` 的 `engines.dsh` 保持 `>=0.2.1-rc.2 <0.3.0`：一次构建同时服务两代宿主，
图片读取在运行时按能力回退，不靠"装了哪个版本"分叉。

> 上游 0.4.2 把下界抬到 `>=0.2.2-rc.1`，因为它只用新版附件 API。本 fork 不抬下界，
> 改用「ref 优先、id 回退」，所以 0.2.1 也能用图片编辑。

---

## 安装

**方式一：插件页填仓库地址**

```
https://github.com/YZKZK01/dsh-redo
```

前提：仓库里要有构建产物 `lib/`。本仓库按上游约定**不提交 `lib/`**，所以这种方式会先构建；
如果安装器只取仓库内容而不构建，请改用方式二/三。

**方式二：本地目录（推荐自用）**

```sh
npm install
npm run build          # 产出 lib/index.js + lib/client.js
```

把整个目录（含 `lib/`、`cordis.patch.yml`、`package.json`）放进 profile 的 `vendor/`，
再让 `node_modules/<name>` 指向它（Windows 用 `mklink /J`），并在 `package.json` 里声明
`"dsh-redo": "file:./vendor/dsh-redo"` 且把 `dsh-redo` 加进 `dsh.profile.bundles`。

**方式三：npm 包（需要先发布）**

```sh
npm pack               # 生成 dsh-redo-0.4.1-fork.x.tgz
# 或发布到自己的 scope（不要占用上游的 dsh-redo 包名）
```

---

## 构建与验证

```sh
npm install
npm run typecheck      # tsc --noEmit
npm test               # node --test，直接跑 TS 源码
npm run build          # lib/index.js + lib/client.js
```

本 fork 当前状态：`typecheck` 通过、**37 项测试全过**、构建产物 `lib/client.js` ≈ 39.7 KB，
包含 `isRedoRewrite`（区间修复）、`cancelArm` / `cancelPending`（取消重写）、跨版本附件读取。

---

## 为什么没有整包升级到 0.4.3

| 版本 | `engines.dsh` | 原因 |
|---|---|---|
| 0.4.1 | `>=0.2.1-rc.2` | 当前基线 |
| 0.4.2 / 0.4.3 | `>=0.2.2-rc.1` | 只认完整 `ImageAttachmentRef`，旧宿主读不了 |

本 fork 的做法：**保持 0.4.1 的 API 面 + 回移 0.4.3 的区间修复 + 自己实现双代附件读取**。
这样在 DSH 还停在 0.2.1-rc.2 时也能拿到上游最有价值的那条修复（长会话不再空白）。

---

## 已知限制

- 输入框已有草稿、或该消息之后还有轮次时，点 ✎ 仍会先弹上游自带的确认框；确认后才进入"已武装"状态。
- 「取消重写」只清武装与输入框，不会恢复已被重发覆盖的旧内容（superseded 语义属上游设计）。
- `InputActions` 上没有 `focus()`，取消后焦点不会自动回到输入框。
- 文件（非图片）消息仍不支持编辑/重试（上游限制）。

---

## 维护节奏（DSH 一更新就跟着动）

每次 DSH 升级后按这个清单走一遍：

1. `npm view dsh-redo version` / 看上游 release：有没有新版本；
2. 有新版：下载对应 tag 源码，`git diff` 看 `src/` 变化，把本 fork 的三处改动重放：
   - `src/client/index.tsx`：ARMED store、`arm()` 不隐藏、动作行取消按钮、跨代附件读取；
   - `src/client/locales.ts`：`action.cancel`；
   - `src/client/intervals.ts`：`isRedoRewrite` 区间修复（若上游已含则跳过）；
3. `npm install && npm run typecheck && npm test && npm run build`；
4. 换 profile `vendor/` 里的 `lib/`，重启 DSH；
5. 验证三件事：✎ 后行还在 + 有「取消重写」；点击后草稿清空、行还在；压缩过的长会话不空白；
6. 顺手看 `engines.dsh` 是否仍覆盖当前宿主。

---

## 上游与许可

- 上游：https://github.com/Imzl-zl/dsh-redo （MIT，作者 Imzl-zl）
- 上游 issue #1：长会话压缩后 transcript 被清空（0.4.3 修复）
- 本 fork：https://github.com/YZKZK01/dsh-redo
- 许可：MIT，保留原版权与许可文本；本 fork 的改动同样以 MIT 提供。
