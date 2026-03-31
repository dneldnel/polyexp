# Polymarket Implementation Plan

这是基于当前知识库整理的第一版实现路线，不是最终架构，但足够作为起步蓝图。

## Phase 1: 先打通读数据

建议先实现：

- `src/adapters/gamma.ts`
- `src/services/marketResolver.ts`
- `src/adapters/clobPublic.ts`
- `src/services/quoteService.ts`

目标：

- 输入 `seriesSlug` 或 `marketSlug`
- 输出：
  - `marketId`
  - `conditionId`
  - `tokenIds`
  - `bestBid/bestAsk`
  - `spread`
  - `side liquidity`

## Phase 2: 打通私有账户与下单

建议实现：

- `src/adapters/clobPrivate.ts`
- `src/services/accountService.ts`
- `src/services/orderService.ts`

目标：

- 校验 auth
- 获取 balance / allowance
- place limit order
- place market order
- cancel order
- get order / get open orders

## Phase 3: 打通实时事件

建议实现：

- `src/ws/marketWs.ts`
- `src/ws/userWs.ts`
- `src/state/orderBook.ts`
- `src/state/orderState.ts`

目标：

- 实时接市场盘口
- 实时接用户订单/成交
- 建立本地 order state machine
- 让 REST 与 WS 可以互相纠偏

## Phase 4: 做可持续运行能力

建议实现：

- `src/services/backfill.ts`
- `src/services/reconcile.ts`
- `src/health/check.ts`
- `src/monitor/`

目标：

- 重连后回补历史
- 定期做仓位/订单对账
- 启动前做健康检查
- 让系统可以长期跑，而不是只会“演示一次”

## 最初不要急着做的事

先不要做：

- 复杂策略层
- 多交易源对冲逻辑
- 过早的前端面板
- 太早抽象成大型框架

先把这条链路跑顺：

市场发现 -> token 解析 -> 报价/盘口 -> 账户校验 -> 下单 -> 用户回报 -> 对账
