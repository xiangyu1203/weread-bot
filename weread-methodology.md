# 微信读书自动阅读机器人（weread-bot）——方法论与运维说明

> 仓库：**xiangyu1203/weread-bot**（主仓库，公开）
> 最后更新：2026-11-16

## 1. 项目简介

- **用途**：自动在微信读书里阅读，积累每日阅读时长。
- **来源**：公开 fork 自 `funnyzak/weread-bot`，在上游基础上按需定制。
- **每日阅读目标**：约 72.5 分钟。
- **运行方式**：全部跑在 GitHub Actions 上，无需自备服务器；每天自动运行，失败时推送通知。

## 2. 仓库与文件

- **主仓库**：`xiangyu1203/weread-bot`（公开 fork，须保持公开，见第 8 节）
- **上游**：`funnyzak/weread-bot`

关键文件：

| 文件 | 作用 |
|---|---|
| `weread-bot.py` | 阅读机器人主程序 |
| `.github/workflows/auto-reading.yml` | 自动阅读定时任务 |
| `.github/workflows/sync-fork.yml` | 每天同步上游更新 |
| `.github/workflows/keepalive.yml` | 保活，防止工作流被自动禁用 |
| `config.yaml` | 运行时由 workflow 动态生成，不入库 |
| `logs/run-history.json` | 运行历史记录，最多保留 50 条 |

仓库 Secrets（只放名字，值绝不写进代码或文档）：

- `WEREAD_CURL_STRING` —— 微信读书 cookie，核心凭证
- `TARGET_DURATION` —— 单次目标阅读时长
- `PUSHPLUS_TOKEN` —— 推送通知凭证

## 3. 定时任务一览

| 工作流 | 文件 | 设定时间（北京时间） | 实际运行时间（北京时间） | 作用 |
|---|---|---|---|---|
| Sync Fork | `sync-fork.yml` | 每天 06:00（UTC `0 22 * * *`） | 约 09:30–10:30 | 同步上游更新，须在阅读任务之前 |
| Auto Reading Bot | `auto-reading.yml` | 每天 07:00（UTC `0 23 * * *`）和 12:30（UTC `30 4 * * *`） | 上午约 10:30–11:30；下午约 15:30–16:00 | 自动阅读，每天两场 |
| Keepalive | `keepalive.yml` | 每 3 天 08:00（UTC `0 0 */3 * *`） | 约 14:00 | 提交时间戳保活，防止工作流被自动禁用 |

> GitHub 定时任务经常延迟 3–5 小时，实际时间比设定时间晚是常态，不是故障。

## 4. 阅读机制

- **凭证**：用 `WEREAD_CURL_STRING` 里的微信读书 cookie 登录后阅读。
- **单次目标时长**：约 72.5 分钟（取自 `TARGET_DURATION` secret），单次运行超时 120 分钟。
- **阅读模式**：默认 `smart_random`（智能随机选书、优先续读）。
- **按天累计**：机器人依据运行历史按天累计阅读时长；同一天内再次运行时，会补足未达到目标的部分。
- **补读**：某次运行被提前终止（如收到 SIGTERM），当天再手动触发一次即可补足剩余时长。实例：2026-11-07 上午只读了 23/72.5 分钟，当天补读后补满。
- **下午 12:30 第二场**：同一天内的第二次机会；上午未达标时下午补足，上午已达标时下午读得少。

## 5. 手动触发方法

1. 打开仓库 **Actions** 标签页。
2. 左侧选择 **Auto Reading Bot**。
3. 点击右上角 **Run workflow**。
4. （可选）填写参数，留空则用默认值 / secret：
   - `target_duration`：阅读时长（分钟），如 `10-20`；留空用 `TARGET_DURATION` secret
   - `reading_mode`：`smart_random`（默认）/ `sequential` / `pure_random`
   - `notify_enabled`：是否启用通知（默认 true）
   - `notify_only_on_failure`：仅失败时发送通知（默认 false）
   - `history_enabled`：是否启用执行历史持久化（默认 true）
   - `history_max_entries`：历史最多保留条数（默认 50）
5. 点击绿色 **Run workflow** 按钮开始运行。

## 6. 通知

- 通过 **PushPlus** 推送：成功和失败都会通知（除非设置了仅失败通知）。
- 失败时推送会带上原因，不静默。

## 7. 常见问题与处理

- **定时任务比设定时间晚几个小时**：正常现象，GitHub 定时任务常延迟 3–5 小时，不必处理。
- **工作流突然不触发了**：GitHub 会在 fork 长期无推送（约 28 天）后自动禁用工作流。去仓库 Actions 页面重新启用即可。已发生两次：2026-10-10、2026-11-07；2026-11-07 修复时加了 keepalive 保活，此后一般不会再发生。
- **阅读失败**：先检查 `WEREAD_CURL_STRING` 是否过期——cookie 失效是最常见原因，重新抓取新鲜 cookie 并更新到 Secrets。
- **单次阅读没读满目标时长**：手动再触发一次 Auto Reading Bot 补读（见第 5 节）。
- **收到 weread-bot-v2 的失败邮件**：v2 是未配 cookie 的旧副本，其工作流已于 2026-11-16 停用；不要用它，也不要把它和主仓库混淆。

## 8. 注意事项

- **必须保持公开**：fork 的可见性与上游绑定，不能转私有；私有仓库免费版每月仅 2,000 分钟 Actions 额度，而每天阅读约 75 分钟、每月约 2,250 分钟，会超额停跑。
- **不要删除或停用 `keepalive.yml`**：否则 fork 长期无推送会被 GitHub 再次自动禁用工作流。
- **不要改动 `sync-fork.yml` 的同步时间**：它必须在阅读任务之前运行。
- **凭证只放 Secrets**：cookie、token 等绝不写进仓库文件或本文档。
- **只用主仓库**：`xiangyu1203/weread-bot-v2`（私有副本）已停用，不要用它；所有操作只针对主仓库。
