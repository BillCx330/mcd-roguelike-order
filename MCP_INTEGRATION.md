# 麦当劳 MCP 集成说明

## Server

- Server：麦当劳中国 MCP Server
- Endpoint：`https://mcp.mcd.cn`
- Transport：Streamable HTTP
- 认证：`Authorization: Bearer <个人 MCP Token>`
- 限流：每个 Token 每分钟最多 600 次请求，超过返回 429；Skill 侧控制调用频率
- 开放平台与 Token 申请：[open.mcd.cn/mcp](https://open.mcd.cn/mcp)

Token 由用户在自己的 MCP 客户端中配置。本项目不收集、保存或代理 Token。

## 调用流程

一局游戏的端到端调用顺序：

1. **开局**：`now-time-info` 读取现实时间做副本背景；`query-my-account` 查积分余额与临期积分；需要活动事件时调用 `campaign-calendar` 查询当月营销日历。
2. **探索**：按随机路线进入房间。补给格用 `available-coupons` 查可领券，开局已告知自动领取规则后调用 `auto-bind-coupons`，再用 `query-my-coupons` 核对；食材密室用 `query-meals`、`query-meal-detail`、`list-nutrition-foods` 取餐品与营养；积分抽奖站先用 `query-lottery-info` 展示消耗，玩家选择真实抽奖才调用 `draw-lottery`，之后 `query-my-prizes` 查奖品；积分商城支线按玩家主动选择调用 `mall-points-products`、`mall-product-detail`、`mall-create-order`。
3. **终局锻造**：Skill 本地筛选候选组合，用 `query-store-coupons` 查门店可用券，再用 `calculate-price` 对候选方案逐个核价（该工具负责算价，不负责全局最优组合搜索）。
4. **传送现世（可选）**：`delivery-query-addresses` 读取地址供玩家选择，`delivery-query-stores` 查询可配送门店，展示最终账单并经玩家确认后 `create-order` 创建待支付订单。

## Tool 与游戏流程

| 游戏阶段 | Tool | 用途 |
| --- | --- | --- |
| 开局与时空副本 | `now-time-info`、`campaign-calendar` | 读取当前时间和真实活动日历作为副本背景；不把营销文案推断成折扣。 |
| 优惠券补给格 | `available-coupons`、`auto-bind-coupons`、`query-my-coupons` | 查询可领券；玩家进入并已获知自动领券规则后领取，再核对账户券状态。 |
| 食材密室与 HP | `query-meals`、`list-nutrition-foods`、`query-meal-detail` | 获取当前门店菜单、套餐组成和营养信息；同步展示真实营养值与游戏 HP。 |
| 积分信息 | `query-my-account` | 按需查看积分与临期积分；不把真实积分自动当作游戏行动点消耗。 |
| 积分抽奖站 | `query-lottery-info`、`draw-lottery`、`query-my-prizes` | 查询活动状态、奖品、消耗规则和用户资源；真实抽奖须先展示消耗并等玩家明确选择。 |
| 套餐锻造 | `query-store-coupons`、`calculate-price` | Skill 在本地筛选套餐候选，再由 MCP 对候选组合进行实际价格计算。 |
| 传送现世 | `delivery-query-addresses`、`delivery-query-stores`、`create-order` | 按用户选择的配送地址查询门店；展示最终账单并获确认后创建待支付订单。 |
| 积分商城支线 | `mall-points-products`、`mall-product-detail`、`mall-create-order` | 玩家主动进入兑换流程后查询；展示积分/金额与兑换结果，确认后才兑换。 |

## 游戏随机性

路线、骰点、普通剧情事件和虚拟战利品由 Skill 在对话中生成，不会调用 MCP 伪造账户数据。每局随机洗牌 5–6 个房间，事件文案、掷骰播报和终局 Boss 从素材池轮换，同一局内尽量不重复。活动、餐品、价格、优惠券、积分、抽奖和订单状态均以真实 MCP 返回为准。

官方工具说明列出抽奖活动状态、奖品、消耗规则及用户可用资源，没有承诺提供中奖概率。若实时响应没有概率字段，Skill 不声称虚拟抽奖复刻官方概率；会明确标注游戏内模拟概率。虚拟结果不会进入麦当劳账户，也不会生成真实奖券。

## 业务价值

随机路线和程序员叙事降低“吃什么”的决策负担；MCP 的实时菜单、优惠、营养和价格数据让游戏结果能落回真实点餐。免费券在补给格集中领取，玩家可选的积分抽奖与最终订单保留清晰确认，让趣味流程同时具备实际可用性。

## 有副作用的动作

- 进入优惠券房间后，若玩家已在本局开始时获知会自动领取可领免费券，Skill 可调用 `auto-bind-coupons` 并汇报实际结果。
- `draw-lottery`、`mall-create-order` 会消耗积分或次数。必须展示本次实际消耗并等待用户选择真实操作。
- `create-order` 会创建真实订单。必须在调用前展示餐品、门店/地址和最终应付金额，并等待用户确认。创建订单不等于支付，支付由用户在官方页面完成。

## 原型与真实调用边界

`demo.html` 是本地模拟试玩，不连接 MCP Server。真实工具调用由已配置 MCP Server 的兼容 AI 客户端按照 `SKILL.md` 执行。网页原型中的结果均为虚构演示数据。
