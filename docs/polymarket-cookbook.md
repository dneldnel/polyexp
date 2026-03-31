# Polymarket Cookbook

这份文档提供最小可执行的交互范式，目标是只靠这里就能知道如何与 Polymarket API 互动。

## 1. 先理解三个关键 ID

你后续所有代码都会围绕这三个标识转：

- `market slug`
  例如 `btc-updown-5m-1771518900`
- `conditionId`
  一个 market 通常对应一个条件市场
- `tokenId`
  具体某个 outcome 的交易 token，真正下单和订阅盘口时用它

对于二元市场，通常会有两个 token：

- `Up` 对应一个 `tokenId`
- `Down` 对应一个 `tokenId`

## 2. 找到当前市场

### 2.1 按 slug 直接查

```bash
curl "https://gamma-api.polymarket.com/markets?slug=btc-updown-5m-1771518900"
```

### 2.2 按 series 找 live 市场

```bash
curl "https://gamma-api.polymarket.com/events?series_id=10684&active=true&closed=false&limit=25"
```

推荐做法：

1. 取返回中的 `events`
2. 把所有 `event.markets` 展平
3. 过滤出当前需要的市场
4. 从中选出当前 live 或最近将生效的那一个

### 2.3 一个通用的 market 选择函数

```js
function toMs(value) {
  const t = new Date(value).getTime();
  return Number.isFinite(t) ? t : null;
}

function pickLatestLiveMarket(markets, nowMs = Date.now()) {
  const enriched = (Array.isArray(markets) ? markets : [])
    .map((market) => ({
      market,
      startMs: toMs(market.eventStartTime ?? market.startTime ?? market.startDate),
      endMs: toMs(market.endDate ?? market.marketEndTime),
    }))
    .filter((item) => item.endMs != null);

  const live = enriched
    .filter((item) => (item.startMs == null || item.startMs <= nowMs) && nowMs < item.endMs)
    .sort((a, b) => a.endMs - b.endMs);

  if (live.length > 0) {
    return live[0].market;
  }

  const upcoming = enriched
    .filter((item) => nowMs < item.endMs)
    .sort((a, b) => a.endMs - b.endMs);

  return upcoming.length > 0 ? upcoming[0].market : null;
}
```

## 3. 从 market 提取 Up/Down token

`outcomes` 和 `clobTokenIds` 可能是数组，也可能是 JSON 字符串。不要写死只支持一种。

```js
function parseMaybeJsonArray(value) {
  if (Array.isArray(value)) return value;
  if (typeof value === "string" && value.trim()) {
    try {
      const parsed = JSON.parse(value);
      return Array.isArray(parsed) ? parsed : [];
    } catch {
      return [];
    }
  }
  return [];
}

function extractBinaryTokens(market, upLabel = "Up", downLabel = "Down") {
  const outcomes = parseMaybeJsonArray(market.outcomes);
  const tokenIds = parseMaybeJsonArray(market.clobTokenIds);

  let upTokenId = null;
  let downTokenId = null;

  for (let i = 0; i < outcomes.length; i += 1) {
    const label = String(outcomes[i] ?? "").toLowerCase();
    const tokenId = tokenIds[i] ? String(tokenIds[i]) : null;
    if (!tokenId) continue;
    if (label === upLabel.toLowerCase()) upTokenId = tokenId;
    if (label === downLabel.toLowerCase()) downTokenId = tokenId;
  }

  return { upTokenId, downTokenId };
}
```

## 4. 获取当前报价和盘口

### 4.1 获取单边当前报价

```bash
curl "https://clob.polymarket.com/price?token_id=<TOKEN_ID>&side=buy"
```

返回里最重要的是：

- `price`

### 4.2 获取订单簿

```bash
curl "https://clob.polymarket.com/book?token_id=<TOKEN_ID>"
```

常见字段：

- `bids`
- `asks`
- `last_trade_price`

### 4.3 提炼盘口摘要

```js
function toNum(value) {
  const n = Number(value);
  return Number.isFinite(n) ? n : null;
}

function summarizeOrderBook(book, depthLevels = 5) {
  const bids = Array.isArray(book?.bids) ? book.bids : [];
  const asks = Array.isArray(book?.asks) ? book.asks : [];

  const bestBid = bids.reduce((best, row) => {
    const price = toNum(row.price);
    if (price == null) return best;
    return best == null ? price : Math.max(best, price);
  }, null);

  const bestAsk = asks.reduce((best, row) => {
    const price = toNum(row.price);
    if (price == null) return best;
    return best == null ? price : Math.min(best, price);
  }, null);

  const bidLiquidity = bids.slice(0, depthLevels).reduce((sum, row) => {
    return sum + (toNum(row.size) ?? 0);
  }, 0);

  const askLiquidity = asks.slice(0, depthLevels).reduce((sum, row) => {
    return sum + (toNum(row.size) ?? 0);
  }, 0);

  return {
    bestBid,
    bestAsk,
    spread: bestBid != null && bestAsk != null ? bestAsk - bestBid : null,
    bidLiquidity,
    askLiquidity,
  };
}
```

## 5. 获取账户、持仓和历史

### 5.1 账户价值

```bash
curl "https://data-api.polymarket.com/value?user=<ADDRESS>"
```

### 5.2 当前持仓

```bash
curl "https://data-api.polymarket.com/positions?user=<ADDRESS>&sizeThreshold=0"
```

### 5.3 已平仓记录

```bash
curl "https://data-api.polymarket.com/closed-positions?user=<ADDRESS>&limit=20&offset=0"
```

### 5.4 成交历史

优先试：

```bash
curl "https://data-api.polymarket.com/activity?user=<ADDRESS>&type=TRADE&limit=20"
```

如果接口兼容层不同，再试：

```bash
curl "https://data-api.polymarket.com/activity?user=<ADDRESS>&activityType=TRADE&limit=20"
curl "https://data-api.polymarket.com/trades?user=<ADDRESS>&limit=20"
```

## 6. CLOB 私有认证

推荐环境变量：

```bash
export POLYMARKET_CLOB_BASE="https://clob.polymarket.com"
export POLYMARKET_CHAIN_ID="137"
export POLYMARKET_PRIVATE_KEY="0x..."
export POLYMARKET_FUNDER="0x..."
export POLYMARKET_SIGNATURE_TYPE="2"
```

其中：

- `0` = `EOA`
- `1` = `POLY_PROXY`
- `2` = `POLY_GNOSIS_SAFE`

如果 `signer` 和 `funder` 不是同一个地址，优先检查是不是应该用 `2`。

## 7. 生成或复用 API key

使用 `@polymarket/clob-client` 时，建议优先尝试：

1. `deriveApiKey()`
2. `createApiKey()`
3. `createOrDeriveApiKey()`

最小示例：

```js
import { ClobClient } from "@polymarket/clob-client";
import { Wallet } from "ethers";

const signer = new Wallet(process.env.POLYMARKET_PRIVATE_KEY);
const host = process.env.POLYMARKET_CLOB_BASE ?? "https://clob.polymarket.com";
const chainId = Number(process.env.POLYMARKET_CHAIN_ID ?? 137);
const signatureType = Number(process.env.POLYMARKET_SIGNATURE_TYPE ?? 0);
const funder = process.env.POLYMARKET_FUNDER || undefined;

const client = new ClobClient(host, chainId, signer, undefined, signatureType, funder);

const creds =
  await client.deriveApiKey()
  ?? await client.createApiKey()
  ?? await client.createOrDeriveApiKey();
```

实际工程里要对每一步做返回值校验，不能假设返回一定是合法 creds。

## 8. 查询余额和 allowance

建议顺序：

1. `updateBalanceAllowance({ asset_type: "COLLATERAL" })`
2. `getBalanceAllowance({ asset_type: "COLLATERAL" })`

原因：

- allowance 可能有服务端缓存
- 直接读可能得到旧值，甚至像 `0`

还要注意：

- balance/allowance 常常是 USDC base units
- 也就是 6 位小数整数

```js
function normalizeCollateralUsdcAmount(value) {
  if (typeof value === "string" && value.includes(".")) {
    const n = Number(value);
    return Number.isFinite(n) ? n : null;
  }
  const n = Number(value);
  return Number.isFinite(n) ? n / 1e6 : null;
}
```

## 9. 下限价单

```js
const order = {
  tokenID: "<TOKEN_ID>",
  price: 0.54,
  size: 10,
  side: "BUY",
};

const response = await client.createAndPostOrder(
  order,
  { tickSize: "0.01", negRisk: false },
  "GTC",
  false,
  false
);
```

常见参数：

- `tokenID`
- `price`
- `size`
- `side`
- `tickSize`
- `negRisk`
- `orderType = GTC | GTD`
- `deferExec`
- `postOnly`

## 10. 下 market order

先查：

- `getTickSize(tokenID)`
- `getNegRisk(tokenID)`

再下单：

```js
const tickSize = await client.getTickSize("<TOKEN_ID>");
const negRisk = await client.getNegRisk("<TOKEN_ID>");

const response = await client.createAndPostMarketOrder(
  {
    tokenID: "<TOKEN_ID>",
    amount: 5,
    side: "BUY",
  },
  { tickSize, negRisk },
  "FOK",
  false
);
```

常见选择：

- `FOK`
- `FAK`

重要：

- 提交成功不代表成交成功
- 订单被接受后，还要继续看用户 WS 和订单查询

## 11. 查询订单和 open orders

常见能力：

- `getOpenOrders()`
- `getOrder(orderId)`
- `cancelOrder(orderId)`
- `cancelOrders(orderIds)`
- `cancelAllOrders()`
- `cancelMarketOrders({ market, assetId })`

兼容性注意：

- 某些形态下 `getOpenOrders()` 可能报 `response.data is not iterable`
- 更稳的做法是保留一个 direct fetch fallback，请求 `GET /data/orders` 并自行处理 cursor 分页

## 12. 市场 WebSocket

地址：

- `wss://ws-subscriptions-clob.polymarket.com/ws/market`

订阅示例：

```json
{
  "type": "market",
  "assets_ids": ["token_id_1", "token_id_2"]
}
```

增量调整订阅：

```json
{
  "assets_ids": ["token_id_1"],
  "operation": "subscribe"
}
```

```json
{
  "assets_ids": ["token_id_1"],
  "operation": "unsubscribe"
}
```

运行建议：

- 每 5 秒发一次 `"PING"`
- reconnect 要做 backoff
- 切订阅时同时做增量更新和全量重发
- 消息解析不要写死单一 schema

常见事件形态：

- `book`
- `price_change`
- `price_changes`
- `changes`

## 13. 用户 WebSocket

地址：

- `wss://ws-subscriptions-clob.polymarket.com/ws/user`

订阅示例：

```json
{
  "type": "USER",
  "markets": ["condition_id"],
  "asset_ids": ["token_id"],
  "auth": {
    "apiKey": "...",
    "secret": "...",
    "passphrase": "..."
  }
}
```

建议在本地把用户事件统一归一化为：

- `order`
- `trade`
- `money`

并维护订单状态机：

- `PENDING_SUBMIT`
- `SUBMITTED`
- `LIVE`
- `PARTIALLY_FILLED`
- `FILLED`
- `CANCELED`
- `REJECTED`
- `EXPIRED`

运行建议：

- 连接后立即发订阅
- 周期性 ping/pong
- stale connection 检测
- connect 后主动拉一份余额快照
- 收到用户事件后 debounce 刷新资金信息
- 保留 recent raw messages 便于 debug

## 14. 公开活动 WS

如果要监听公开地址活动，可使用：

- `@polymarket/real-time-data-client`

订阅主题：

- `topic = "activity"`
- `type = "trades"`

然后再按地址过滤：

- `payload.proxyWallet === targetWallet`

这条流适合做：

- 跟单
- 地址监控
- 重连后的增量恢复入口

## 15. Backfill 和对账

推荐模式：

1. WS 负责实时
2. REST 负责补偿和权威确认
3. reconnect / restart 后跑 backfill
4. 定时做 position reconciliation

backfill 常用接口：

- `GET /activity?user=<ADDRESS>&type=TRADE&limit=<n>&offset=<n>`

reconciliation 常用接口：

- `GET /positions?user=<ADDRESS>`

最小原则：

- 对事件流做去重
- 对订单做幂等
- 对账户状态做定期纠偏

## 16. 最小实现顺序

如果接下来要开始写代码，建议顺序是：

1. 市场搜索
2. token 解析
3. 当前 price + order book 快照
4. balance + allowance
5. limit order / market order
6. user ws
7. market ws
8. backfill + reconcile
