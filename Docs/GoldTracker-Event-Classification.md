# Gold Tracker 事件到分类的映射一览（当前实现）

## AC_LOGTYPE 定义与本地化

| AC_LOGTYPE | 英文显示名（English） | 中文显示名（zhCN） | 备注 |
| --- | --- | --- | --- |
| TRANSMO | Transmogrify | 幻化 | 仅零售版；来自 `TRANSMOGRIFY` 全局常量 |
| GARRISON | Garrison / Order Hall Missions | 要塞 / 职业大厅任务 | 仅零售版；由 `GARRISON_LOCATION_TOOLTIP .. " / " .. ORDER_HALL_MISSIONS` 组合 |
| LFG | LFD, LFR and Scen. | 随机地城、团队与事件 | 仅零售版；使用本地化键 `L["LFD, LFR and Scen."]` |
| BARBER | Barbershop | 理发店 | 仅零售版；`BARBERSHOP` |
| GUILD | Guild | 公会 | 仅零售版；`GUILD` |
| TRAIN | Training Costs | 训练费用 | 经典/零售通用；`L["Training Costs"]` |
| TAXI | Taxi Fares | 飞行花费 | 经典/零售通用；`L["Taxi Fares"]` |
| TRADE | Trade Window | 交易 | 经典/零售通用；`L["Trade Window"]` |
| AH | Auctions | 拍卖 | 经典/零售通用；`AUCTIONS` |
| MERCH | Merchants | 商人 | 经典/零售通用；`L["Merchants"]` |
| REPAIRS | Repair Costs | 修理花费 | 经典/零售通用；`L["Repair Costs"]` |
| MAIL | Mail | 邮寄 | 经典/零售通用；`L["Mail"]` |
| QUEST | Quests | 任务 | 经典/零售通用；`QUESTS_LABEL`（客户端内置本地化） |
| LOOT | Loot | 战利品 | 经典/零售通用；`LOOT`（客户端内置本地化） |
| OTHER | Unknown | 未知 | 经典/零售通用；`L["Unknown"]`（当 `AC_LOGTYPE == ""` 时回落） |
| VOID | Void Storage | 虚空仓库 | 零售 10.0 起已移除；经典等分支可能保留 |

本文档汇总 `Core/Core.lua` 中的上下文猜测分类逻辑：当某些 UI/系统事件触发时，会将 `AC_LOGTYPE` 设定为某个分类。真正记账发生在 `PLAYER_MONEY`（或特殊的 `CHAT_MSG_MONEY` 强制路径）时，入账时读取“当时的” `AC_LOGTYPE`。若 `AC_LOGTYPE == ""`，`updateLog()` 会回落为 `OTHER`。

- 核心文件：`Core/Core.lua`
- 事件列表定义：`Core/Constants.lua` 中的 `constants.events`
- 分类列表定义：`Core/Constants.lua` 中的 `constants.logtypes` 与 `constants.onlineData`

> 注：以下表格涵盖零售分支与经典分支的共通与特有事件；若某事件未出现在你的客户端版本中，可忽略。

## 事件 → AC_LOGTYPE 映射表

| 事件（Event） | 分类（AC_LOGTYPE） | 说明 | 代码位置（参考） |
| --- | --- | --- | --- |
| GARRISON_MISSION_FINISHED | GARRISON | 要塞/职业大厅任务 | `Core/Core.lua` → `AccountantClassic_OnEvent` |
| GARRISON_ARCHITECT_OPENED | GARRISON | 要塞/职业大厅任务 | 同上 |
| GARRISON_MISSION_NPC_OPENED | GARRISON | 要塞/职业大厅任务 | 同上 |
| GARRISON_SHIPYARD_NPC_OPENED | GARRISON | 要塞/职业大厅任务 | 同上 |
| GARRISON_ARCHITECT_CLOSED | 清空为 "" | 关闭后清空，后续若立即入账会回落到 `OTHER` | 同上 |
| GARRISON_MISSION_NPC_CLOSED | 清空为 "" | 同上 | 同上 |
| GARRISON_SHIPYARD_NPC_CLOSED | 清空为 "" | 同上 | 同上 |
| LFG_COMPLETION_REWARD | LFG | 随机地城/团本/事件奖励 | 同上 |
| BARBER_SHOP_OPEN | BARBER | 理发店 | 同上 |
| BARBER_SHOP_SUCCESS | BARBER | 理发店 | 同上 |
| BARBER_SHOP_FORCE_CUSTOMIZATIONS_UPDATE | BARBER | 理发店 | 同上 |
| BARBER_SHOP_COST_UPDATE | BARBER | 理发店 | 同上 |
| TRANSMOGRIFY_OPEN | TRANSMO | 幻化 | 同上 |
| (零售早期) VOID_STORAGE_OPEN | VOID | 虚空仓库（10.0 后已移除；经典等分支仍可能存在） | 同上 |
| MERCHANT_SHOW | MERCH | 商人（购买/出售） | 同上 |
| MERCHANT_UPDATE 且 InRepairMode()==true | REPAIRS | 修理花费；`updateLog()` 记账后会把 `AC_LOGTYPE` 重置回 `MERCH` | 同上 + `updateLog()` 末尾 |
| TAXIMAP_OPENED | TAXI | 飞行花费 | 同上 |
| (注释) TAXIMAP_CLOSED | — | 被注释不清空，避免关闭后才入账被错分 | 同上（注释） |
| LOOT_OPENED | LOOT | 拾取 | 同上 |
| (注释) LOOT_CLOSED | — | 被注释不清空，理由同上 | 同上（注释） |
| TRADE_SHOW | TRADE | 交易 | 同上 |
| QUEST_COMPLETE | QUEST | 任务奖励/花费 | 同上 |
| QUEST_TURNED_IN | QUEST | 任务奖励/花费 | 同上 |
| (注释) QUEST_FINISHED | — | 被注释不清空，避免窗口关闭早于入账 | 同上（注释） |
| MAIL_INBOX_UPDATE（发票类型为 seller） | AH | 通过 `GetInboxInvoiceInfo()` 识别拍卖行邮件 | `AccountantClassic_DetectAhMail()` 调用处 |
| MAIL_INBOX_UPDATE（否则） | MAIL | 普通邮件 | 同上 |
| GUILDBANKFRAME_OPENED | GUILD | 公会银行 | `AccountantClassic_OnEvent` |
| GUILDBANK_UPDATE_MONEY | GUILD | 公会银行 | 同上 |
| GUILDBANK_UPDATE_WITHDRAWMONEY | GUILD | 公会银行 | 同上 |
| GUILDBANKFRAME_CLOSED | 清空为 "" | 关闭后清空 | 同上 |
| CONFIRM_TALENT_WIPE | TRAIN | 重置天赋 | 同上 |
| TRAINER_SHOW | TRAIN | 训练费用 | 同上 |
| TRAINER_CLOSED | 清空为 "" | 关闭后清空 | 同上 |
| AUCTION_HOUSE_SHOW | AH | 拍卖行 | 同上 |
| AUCTION_HOUSE_CLOSED | 清空为 "" | 关闭后清空 | 同上 |

## 特殊/辅助路径

| 情形 | 处理 | 说明 | 代码位置 |
| --- | --- | --- | --- |
| CHAT_MSG_MONEY | 将 `AC_LOGTYPE` 临时设为 `LOOT`，并通过调节 `AC_LASTMONEY` 强制调用一次 `updateLog()` | 避免等待 `PLAYER_MONEY` 时被其它分类覆盖；并带有首会话基线保护 | `AccountantClassic_OnShareMoney()` |
| PLAYER_MONEY | 真正差额记账触发点 | 计算 `diff = GetMoney() - AC_LASTMONEY`；若 `AC_LOGTYPE==""` 则在 `updateLog()` 中回落到 `OTHER` | `AccountantClassic_OnEvent` → `updateLog()` |
| PLAYER_LOGIN | 一次性基线初始化（仅当 `.primed == false`） | 我们已将基线从 `PLAYER_MONEY` 前移到此，避免吞掉会话首笔真实变动；不做分类设置 | `AccountantClassic_OnEvent` |

## 说明
- `AC_LOGTYPE` 具有“黏性”：被设置后，直到被其它事件改写或被显式清空前都会沿用；因此某些关闭事件被注释不清空，以避免“界面关闭早于金钱入账”导致错分或漏分。
- 若入账瞬间 `AC_LOGTYPE == ""`，`updateLog()` 会自动将其视作 `OTHER`。
- 邮件与拍卖的区分依赖 `GetInboxInvoiceInfo()` 返回的 `invoiceType` 是否为 `"seller"`。
