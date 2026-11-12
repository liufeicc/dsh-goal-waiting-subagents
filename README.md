# dsh-goal-waiting-subagents

## 目的

目标栏在「目标 active + 已 armed + 主 agent 空闲」时，如果还有子代理在跑，显示：

```
等待 N 个子代理完成
```

鼠标悬停显示说明：子代理仍在运行，目标将在它们结束后继续。

这样就不会再把「goal-round-driver 正在等子代理」误判成「目标没生效 / 不会自动唤起」。

## 背景

`goal-round-driver` 在排下一轮前会执行：

```js
await subagents.waitForChildren(agent)
```

只要还有后代子代理在推进，就不会开新一轮。这是 README 里写明的策略，不是死锁：

- 子代理结束后，它的结算消息会唤醒父会话；
- 即使它没发消息，`waitForChildren` 返回后驱动器也会立刻排下一轮。

问题在可观测性：这段时间里目标栏仍然显示「进行中的目标」，主 agent 又是空闲的，看起来像目标失效。典型现场：

- 父会话 `session-d95d95a9…` 在 22:06:32 正常结束第 27 轮；
- 21:56:13 派出的子代理 `d63b7bb3`（收口 intro1 剩余微差）一直在跑；
- 目标投影是 `active`、`roundsStarted=13`，目标栏显示暂停按钮（armed），但没有任何新一轮。

## 改动

只改**浏览器端**的目标栏插件，不碰宿主、不重启发服务器、不需要重启正在运行的 agent。

### 数据来源

用 Host 会话列表里每个 Session 的实时 `running` 位：

```js
ctx.sessions.list.getSnapshot().byId
```

只统计 `parentId` 能沿父子链追溯到当前会话、且 `running === true` 的直接/嵌套后代。选它而不是从事件流里数 `subagent/catalog` / `subagent-settled`，是因为：

- 已结算、已取消的子代理会留下 `subagent/catalog`，但 `running` 已经变 false；事件流里 5e2f5511、dd66136d 这类被取消的子代理可能永远等不到 `subagent-settled` 消息，会误报；
- Host 列表的 `running` 由 `Agent.status === "running"` 直接计算，是实时权威值；
- 未打开的冷子会话也在列表里，`running` 为 false，不会误报。

### 显示条件

```js
goal.phase === "active" && activation === "armed" && waitingSubagents > 0
```

`runningSubagents()` 在主 agent 自己 `running` 时直接返回 0，因此正常跑轮次时不会显示等待。

### 文件

- `dsh-client-ui-goal/lib/client.js`
  - `createGoalActivationSource()` 新增 `runningSubagents()` / `publishWaiting()`；
  - 订阅 `ctx.sessions.list`，子代理 running 变化时只更新 `waitingSubagents`，不覆盖 activation；
  - `GoalActivationSnapshot` 增加 `waitingSubagents`；
  - `GoalBar` 新增等待标签和 tooltip，`GoalDock` 把字段透传下去；
  - zh / en 字典新增 `phase.active.waiting.one|other|tooltip`。
- `dsh-client-ui-goal/lib/types/client/*.d.ts`：同步类型。
- `dsh-goal/lib/types/types.d.ts`：给 `GoalActivationChanged.goal` 预留可选的 `waitingSubagents`，便于以后宿主端直接下发精确值；当前客户端不依赖它。

### 行为

- 子代理在跑：`等待 1 个子代理完成` / `等待 2 个子代理完成`，暂停按钮、编辑、清除保持可用；
- 子代理结束：计数归零，标签恢复为「进行中的目标」，随后 goal-round-driver 排下一轮；
- 主 agent 开始跑新一轮：立即恢复为「进行中的目标」；
- 目标 paused / disarmed / blocked / complete：保持原来的标签，不显示等待。

## 应用

推荐用工作区的统一脚本（幂等，已应用会自动跳过）：

```bash
/home/liufei/data/claude-workspace/dsh/apply-dsh-patches.sh
```

只应用本补丁时：

```bash
BASE="$(npm root -g)/@deepseek-ai/dsh/node_modules/@deepseek-ai"
patch -p1 -d "$BASE" < /home/liufei/data/claude-workspace/dsh/patches/dsh-goal-waiting-subagents/index.patch
```

### 生效方式（重要）

这是纯客户端 bundle。宿主不需要重启，但浏览器缓存要处理：

插件 URL 形如

```
/plugins/??@deepseek-ai/dsh-client-ui-goal/client.js&rev=<版本号>
```

响应头是 `cache-control: public, max-age=31536000, immutable`。应用补丁会改变文件 mtime，`client-hmr` 通常会在 1~2 秒内重算 `rev`；此时普通刷新就能拿到新 bundle。如果 `rev` 没变、刷新后仍是旧行为，再强制刷新：

- Electron/Chrome：`Ctrl+Shift+R`（或 `Ctrl+F5`）；
- 不行就关闭 DSH Desktop 再打开（会重启宿主，正在运行的 agent 会被中断）；
- 或者打开开发者工具 → Network 勾选 Disable cache 后刷新。

强制刷新后，新的 bundle 会以同一个 `rev` 缓存下来，之后普通刷新就是新代码。

## 备份与撤销

应用前先备份 `dsh-client-ui-goal/lib/client.js`。本补丁只改客户端文件，可随时反向撤销：

```bash
patch -R -p1 -d "$BASE" < index.patch
```

## 升级注意

DSH 升级会覆盖 `node_modules` 下的文件。`apply-dsh-patches.sh` 会遍历 `patches/*/`，升级流程已接入的话会自动重放本补丁；未接入时手动跑一次即可。

## 已知边界

- 计数是「Host 列表里 running 的后代」。子代理在两轮之间短暂 idle（还没结算）时可能显示 0，随后又变回 1；这是实时状态，不是持久计数。
- 只统计能沿 `parentId` 链追溯到的后代；如果某个子会话因持久化损坏不在列表里，它不会被计入（但 goal 驱动器仍可能等它）。
- `mode: external` 的子代理没有本地子 Session 行，无法从列表计数；它们由外部运行器承载，目标栏此时不会显示等待数。
- 这是展示层补丁，不改变 `goal-round-driver` 的等待策略。
