# Polymarket CLOB V2 Implementation Plan

更新时间：2026-04-26

这是 `docs/polymarket-clob-v2-migration.md` 的执行版计划，用于把当前旧文档里的能力迁移到 Polymarket CLOB V2。

## 0. 关键决策

- 只把 CLOB 交易层升级到 V2。
- Gamma API 和 Data API 先不重写。
- 公共报价和订单簿接口先保持 REST 形态。
- 私有下单、撤单、订单查询统一走 `@polymarket/clob-client-v2`。
- 不在第一版实现 raw EIP-712 order signing。
- collateral 统一命名为 `collateral`，不要在业务域里继续写死 `USDC.e`。

## 1. 配置层

建议新增：

- `POLYMARKET_CLOB_BASE`
- `POLYMARKET_CHAIN_ID`
- `POLYMARKET_PRIVATE_KEY`
- `POLYMARKET_FUNDER`
- `POLYMARKET_SIGNATURE_TYPE`
- `POLYMARKET_API_KEY`
- `POLYMARKET_API_SECRET`
- `POLYMARKET_API_PASSPHRASE`
- `POLYMARKET_BUILDER_CODE`

测试期默认：

```bash
POLYMARKET_CLOB_BASE=https://clob-v2.polymarket.com
```

切换后默认：

```bash
POLYMARKET_CLOB_BASE=https://clob.polymarket.com
```

## 2. Adapter 层

### `src/adapters/gamma.ts`

继续实现：

- `getMarketBySlug(slug)`
- `getSeriesBySlug(slug)`
- `getEventsBySeriesId(seriesId)`
- `getMarketsBySeriesSlug(seriesSlug)`
- `scanActiveMarkets(params)`

### `src/adapters/dataApi.ts`

继续实现：

- `getValue(address)`
- `getPositions(address)`
- `getClosedPositions(address, params)`
- `getActivity(address, params)`
- `getTrades(address, params)`

### `src/adapters/clobPublic.ts`

继续实现：

- `getPrice(tokenId, side)`
- `getBook(tokenId)`

新增：

- `getClobMarketInfo(conditionId)`

### `src/adapters/clobPrivate.ts`

使用 V2 SDK：

```ts
import { ClobClient } from "@polymarket/clob-client-v2";

export function createClobClientV2(config: PolymarketConfig): ClobClient {
  return new ClobClient({
    host: config.clobBaseUrl,
    chain: config.chainId,
    signer: config.signer,
    creds: config.creds,
    signatureType: config.signatureType,
    funderAddress: config.funderAddress,
    builderConfig: config.builderCode
      ? { builderCode: config.builderCode }
      : undefined,
  });
}
```

需要封装：

- `deriveOrCreateApiKey()`
- `updateBalanceAllowance()`
- `getBalanceAllowance()`
- `createAndPostOrderV2()`
- `createAndPostMarketOrderV2()`
- `getOrder()`
- `getOpenOrders()`
- `cancelOrder()`
- `cancelOrders()`
- `cancelAllOrders()`

## 3. Service 层

### `src/services/marketResolver.ts`

保留旧流程：

1. market slug 精确查找。
2. series id -> events -> flatten markets。
3. series slug -> series id -> events。
4. fallback 到 markets by series slug。
5. fallback 到 active market scan。
6. `pickLatestLiveMarket()`。

输出必须包含：

- `market`
- `conditionId`
- `upTokenId`
- `downTokenId`
- `outcomeTokenMap`

### `src/services/quoteService.ts`

输入：

- `conditionId`
- `tokenId`
- `side`

输出：

- price
- order book summary
- CLOB market info
- fallback source

下单前必须能提供：

- `bestBid`
- `bestAsk`
- `spread`
- `bidLiquidity`
- `askLiquidity`
- `minimumTickSize`
- `minimumOrderSize`
- `feeDetails`

### `src/services/accountService.ts`

实现：

- API creds readiness check
- pUSD/collateral balance check
- allowance check
- `updateBalanceAllowance()` before `getBalanceAllowance()`
- collateral amount normalization

### `src/services/collateralService.ts`

第一版可以只做 readiness report：

- 当前 pUSD balance
- 当前 pUSD allowance
- 是否需要 wrap
- 需要 wrap 的 USDC.e base units

后续再实现链上 `wrap()` 交易。

### `src/services/orderService.ts`

核心规则：

- 入参不能包含 `feeRateBps`
- 入参不能包含 `nonce`
- 入参不能包含 `taker`
- limit order 自动注入 `builderCode`
- market order 自动注入 `builderCode`
- market buy order 可传 collateral balance 给 SDK 的 `userUSDCBalance`

下单前检查：

1. tokenId 存在。
2. conditionId 存在。
3. market 未 closed。
4. balance 足够。
5. allowance 足够。
6. order book 有可成交流动性。
7. price 满足 tick size。
8. size 满足 minimum order size。

下单后不要直接认为成交：

- HTTP success 只代表 accepted。
- 成交状态来自 user WS、`getOrder()`、trades/activity backfill。

## 4. WS 层

### `src/ws/marketWs.ts`

保留 endpoint：

```text
wss://ws-subscriptions-clob.polymarket.com/ws/market
```

兼容事件：

- `book`
- `price_change`
- `price_changes`
- `changes`
- `tick_size_change`
- `last_trade_price`
- `best_bid_ask`
- `new_market`
- `market_resolved`

### `src/ws/userWs.ts`

保留 endpoint：

```text
wss://ws-subscriptions-clob.polymarket.com/ws/user
```

auth 继续使用：

- `apiKey`
- `secret`
- `passphrase`

归一化事件：

- `order`
- `trade`
- `money`

订单状态机保留：

- `PENDING_SUBMIT`
- `SUBMITTED`
- `LIVE`
- `PARTIALLY_FILLED`
- `FILLED`
- `CANCELED`
- `REJECTED`
- `EXPIRED`

## 5. Reconcile / Backfill

重连或重启后执行：

1. 拉 balance / allowance。
2. 拉 open orders。
3. 拉 positions。
4. 拉 recent activity/trades。
5. 对本地订单状态机做幂等更新。

2026-04-28 切换后特别处理：

- 不假设旧 open orders 存在。
- 丢弃旧 open order ID 的恢复逻辑。
- 以新 open orders 快照为准。
- 按策略重新挂单。

## 6. 第一批验收用例

- 能用 V2 base URL 创建 client。
- 能 derive 或复用 API creds。
- 能通过 Gamma 找到 market 并解析 token。
- 能通过 CLOB V2 查 price/book。
- 能通过 `getClobMarketInfo(conditionId)` 获取市场参数。
- 能读取 collateral balance / allowance。
- 能下最小 size 的限价单。
- 能撤销刚下的限价单。
- 能下 FOK market order dry run 或小额实盘。
- 能从 user WS 收到 order/trade 事件。
- 能重启后 reconcile open orders / positions。

## 7. 暂不做

- raw REST order signing
- 自研 fee calculation
- builder relayer / gasless transaction
- 自动 USDC.e -> pUSD wrap 交易
- 大型策略层
- 前端面板
