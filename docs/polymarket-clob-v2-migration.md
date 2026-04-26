# Polymarket CLOB V2 Migration Notes

更新时间：2026-04-26

本文基于当前仓库已有文档：

- `docs/polymarket-knowledge-base.md`
- `docs/polymarket-cookbook.md`
- `notes/polymarket-implementation-plan.md`

并对照 Polymarket 官方 CLOB V2 迁移文档整理。目标是回答两个问题：

1. 新旧 API / SDK 到底变了什么。
2. 旧文档里的工作在新 API 下应该如何继续实现。

官方来源：

- https://docs.polymarket.com/v2-migration
- https://docs.polymarket.com/changelog
- https://docs.polymarket.com/api-reference
- https://docs.polymarket.com/api-reference/authentication
- https://docs.polymarket.com/concepts/pusd
- https://docs.polymarket.com/market-data/websocket/overview

## 1. 时间线与结论

官方迁移窗口：

- Go-live：2026-04-28 约 11:00 UTC
- 预计停机：约 1 小时
- 切换期间：全部 open orders 会被清空
- V2 测试地址：`https://clob-v2.polymarket.com`
- 切换后生产地址：仍然是 `https://clob.polymarket.com`

直接结论：

- Gamma API 和 Data API 的职责不变，旧文档里的市场发现、用户持仓、历史活动逻辑继续保留。
- CLOB REST 的公共读接口仍保留原有工作方式，`/price`、`/book`、订单簿快照等逻辑继续可用。
- CLOB 私有交易必须迁移到 V2 SDK：`@polymarket/clob-client-v2` 或 `py-clob-client-v2`。
- 旧的 `@polymarket/clob-client` / `py-clob-client` 在切换后没有向后兼容保证。
- API auth 不变，已有 L1/L2 API key、secret、passphrase 可继续使用。
- 订单签名结构、构造器、费用模型、builder attribution、抵押资产都变了。
- WebSocket URL 基本不变，但实现里要继续保持 schema 兼容解析。

## 2. 新旧 API 总表

| 模块 | V1 / 旧文档 | V2 / 新文档 | 迁移动作 |
| --- | --- | --- | --- |
| Gamma API | `https://gamma-api.polymarket.com` | 不变 | 继续用于 markets/events/series/search |
| Data API | `https://data-api.polymarket.com` | 不变 | 继续用于 positions/activity/closed-positions/value |
| CLOB base | `https://clob.polymarket.com` | 测试期 `https://clob-v2.polymarket.com`，切换后仍是 `https://clob.polymarket.com` | 配置化 `POLYMARKET_CLOB_BASE` |
| CLOB SDK | `@polymarket/clob-client` | `@polymarket/clob-client-v2` | 替换依赖和 import |
| SDK constructor | positional args | options object | `new ClobClient({ host, chain, signer, creds, ... })` |
| chain 参数 | `chainId` | `chain` | 重命名 |
| API auth | L1/L2 auth | 不变 | 继续复用已有 creds |
| Order fields | `nonce`, `feeRateBps`, `taker` 可出现在订单结构里 | 移除这些字段，新增 `timestamp`, `metadata`, `builder` | 订单构建层移除旧字段 |
| Fee model | 费用嵌入 signed order | match time 动态费用 | 不再手动设置 `feeRateBps` |
| Fee query | 分散参数 / 手工逻辑 | `getClobMarketInfo(conditionID)` | 下单前拉 CLOB market info |
| Builder | `POLY_BUILDER_*` HMAC headers + signing SDK | `builderCode` 字段 | 移除 builder signing SDK，仅保留 builder code |
| Collateral | USDC.e | pUSD | API-only trader 需要 USDC.e -> pUSD wrap |
| WebSocket | market/user URL | 官方 FAQ 称 URL 不变 | 保留现有 WS 架构 |
| Open orders | `getOpenOrders()` / direct fallback | 仍需 L2 auth | 保留 direct fallback，但用 V2 client 优先 |

## 3. 旧文档中的工作如何迁移

### 3.1 市场发现：继续走 Gamma

旧文档中的这些流程保留：

- `GET /markets?slug=<market_slug>`
- `GET /events?series_id=<id>&active=true&closed=false`
- `GET /series?slug=<series_slug>`
- `GET /markets?seriesSlug=<series_slug>&active=true&closed=false&enableOrderBook=true`
- 活跃市场分页扫描
- `pickLatestLiveMarket()`
- `extractBinaryTokens()`

实现落点：

- `src/adapters/gamma.ts`
- `src/services/marketResolver.ts`

V2 迁移不要求重写 Gamma 逻辑。仍然需要从 market 元数据里解析：

- `conditionId`
- `outcomes`
- `clobTokenIds`
- `tokenId`

注意：V2 新增的 `getClobMarketInfo(conditionID)` 不是替代 Gamma，而是补充 CLOB 交易参数，例如 minimum tick size、minimum order size、fee details、token info、RFQ enabled。

推荐新增接口：

```ts
export type ClobMarketInfo = {
  conditionId: string;
  minimumTickSize: string | null;
  minimumOrderSize: string | null;
  feeDetails: unknown;
  tokens: Array<{ tokenId: string; outcome: string }>;
  rfqEnabled: boolean | null;
};
```

### 3.2 当前报价与盘口：公共 CLOB 读逻辑保留

旧文档中的这些流程保留：

- `GET /price?token_id=<TOKEN_ID>&side=buy`
- `GET /book?token_id=<TOKEN_ID>`
- `summarizeOrderBook()`
- CLOB 报价失败时 fallback 到 Gamma `outcomePrices`

实现落点：

- `src/adapters/clobPublic.ts`
- `src/services/quoteService.ts`

需要新增的 V2 能力：

- `getClobMarketInfo(conditionID)`
- 下单前同时校验 `minimumTickSize`、`minimumOrderSize`、fee details

推荐 quote snapshot 结构：

```ts
export type QuoteSnapshot = {
  conditionId: string;
  tokenId: string;
  side: "buy" | "sell";
  price: number | null;
  bestBid: number | null;
  bestAsk: number | null;
  spread: number | null;
  bidLiquidity: number;
  askLiquidity: number;
  clobMarketInfo?: ClobMarketInfo;
};
```

### 3.3 CLOB client 初始化：必须改

旧写法：

```ts
import { ClobClient } from "@polymarket/clob-client";

const client = new ClobClient(
  host,
  chainId,
  signer,
  creds,
  signatureType,
  funderAddress
);
```

V2 写法：

```ts
import { ClobClient } from "@polymarket/clob-client-v2";

const client = new ClobClient({
  host,
  chain: chainId,
  signer,
  creds,
  signatureType,
  funderAddress,
});
```

环境变量建议：

```bash
export POLYMARKET_CLOB_BASE="https://clob-v2.polymarket.com"
export POLYMARKET_CHAIN_ID="137"
export POLYMARKET_PRIVATE_KEY="0x..."
export POLYMARKET_FUNDER="0x..."
export POLYMARKET_SIGNATURE_TYPE="2"
export POLYMARKET_API_KEY="..."
export POLYMARKET_API_SECRET="..."
export POLYMARKET_API_PASSPHRASE="..."
export POLYMARKET_BUILDER_CODE="0x..."
```

切换后只需要把 `POLYMARKET_CLOB_BASE` 改回或默认到：

```bash
export POLYMARKET_CLOB_BASE="https://clob.polymarket.com"
```

### 3.4 API key 获取：保留策略，但用 V2 client

旧文档推荐顺序仍然成立：

1. `deriveApiKey()`
2. `createApiKey()`
3. `createOrDeriveApiKey()`

V2 中 L1/L2 auth 不变，因此已有 `apiKey / secret / passphrase` 不需要重建。

实现落点：

- `src/adapters/clobPrivate.ts`
- `src/services/authService.ts`

推荐不要在每次启动都创建 key。优先从环境变量或 secret manager 读已有 creds，仅在缺失时 derive/create。

### 3.5 余额和 allowance：概念保留，资产换成 pUSD

旧文档里：

- `getBalanceAllowance({ asset_type: "COLLATERAL" })`
- `updateBalanceAllowance({ asset_type: "COLLATERAL" })`
- USDC 6 decimals 归一化

V2 中 collateral 从 USDC.e 换成 pUSD。pUSD 仍是 Polygon ERC-20，6 decimals。API-only trader 需要通过 Collateral Onramp 把 USDC.e wrap 成 pUSD。

实现落点：

- `src/services/accountService.ts`
- `src/services/collateralService.ts`

需要新增检查：

- pUSD balance 是否足够
- pUSD allowance 是否足够
- API-only 账户是否完成 USDC.e -> pUSD wrap
- 是否还存在硬编码 USDC.e 合约地址

推荐保留旧的金额归一化函数，但命名不要继续写死 `Usdc`：

```ts
function normalizeCollateralAmount(value: unknown): number | null {
  if (typeof value === "string" && value.includes(".")) {
    const n = Number(value);
    return Number.isFinite(n) ? n : null;
  }
  const n = Number(value);
  return Number.isFinite(n) ? n / 1e6 : null;
}
```

### 3.6 限价单：移除旧字段

旧文档中的 V1 下单字段：

- `tokenID`
- `price`
- `size`
- `side`
- `tickSize`
- `negRisk`
- `feeRateBps`
- `nonce`
- `expiration`
- `taker`

V2 下单规则：

- 保留：`tokenID`, `price`, `size`, `side`, `expiration`
- 移除：`feeRateBps`, `nonce`, `taker`
- 新增可选：`builderCode`
- SDK/后端处理：`timestamp`, `metadata`, `builder`

V2 示例：

```ts
import { OrderType, Side } from "@polymarket/clob-client-v2";

const response = await client.createAndPostOrder(
  {
    tokenID,
    price: 0.54,
    size: 10,
    side: Side.BUY,
    builderCode: process.env.POLYMARKET_BUILDER_CODE,
  },
  {
    tickSize,
    negRisk,
  },
  OrderType.GTC
);
```

实现落点：

- `src/services/orderService.ts`
- `src/domain/orders.ts`

订单 DTO 应该拆成两层：

- `OrderIntent`：业务意图，允许表达 target token、side、size、max price 等。
- `ClobOrderRequestV2`：真正传给 V2 SDK 的字段，不允许包含 `feeRateBps`、`nonce`、`taker`。

### 3.7 市价型订单：保留 FOK/FAK，但加 fee-aware 参数

旧文档流程保留：

- 先查 `getTickSize(tokenID)`
- 再查 `getNegRisk(tokenID)`
- 再 `createAndPostMarketOrder(...)`
- 常见 `OrderType.FOK`

V2 变化：

- Market order 仍然是 SDK 封装出来的 marketable order。
- 可传 `userUSDCBalance`，让 SDK 做 fee-adjusted fill amount。
- 可传 `builderCode`。
- 不再手动设置 `feeRateBps`。

V2 示例：

```ts
import { OrderType, Side } from "@polymarket/clob-client-v2";

const response = await client.createAndPostMarketOrder(
  {
    tokenID,
    amount: 5,
    side: Side.BUY,
    orderType: OrderType.FOK,
    userUSDCBalance: collateralBalance,
    builderCode: process.env.POLYMARKET_BUILDER_CODE,
  },
  {
    tickSize,
    negRisk,
  },
  OrderType.FOK
);
```

注意：官方 V2 文档字段名仍写 `userUSDCBalance`，但底层 collateral 已切到 pUSD。代码里可以在封装层命名为 `collateralBalance`，在调用 SDK 时映射到 `userUSDCBalance`。

### 3.8 Fee model：不要再手工算 feeRateBps

旧逻辑：

- 订单里可能带 `feeRateBps`
- 手工估算费用

V2 逻辑：

- 费用在 match time 由协议决定。
- maker 不收费，taker 收费。
- 使用 `getClobMarketInfo(conditionID)` 获取 fee details。
- SDK 自动处理 fee-adjusted amount。

实现落点：

- 删除订单输入里的 `feeRateBps`
- 删除 raw order signing 里的 `feeRateBps`
- 如果 UI/日志需要展示费用，只展示 estimated fee，不把它作为 signed order 字段

### 3.9 Builder Program：改成 builderCode

旧逻辑：

- `@polymarket/builder-signing-sdk`
- `POLY_BUILDER_API_KEY`
- `POLY_BUILDER_SECRET`
- `POLY_BUILDER_PASSPHRASE`
- `POLY_BUILDER_SIGNATURE`

V2 逻辑：

- 下单时使用 `builderCode`
- 可以每个 order 传 `builderCode`
- 也可以在 client construction 里设置 builder config
- builder code 是公开标识，不是 secret

实现落点：

- 移除 builder signing SDK 依赖。
- 移除 order signing 的 builder HMAC headers。
- 新增 `POLYMARKET_BUILDER_CODE`。
- 订单服务统一在 V2 order request 里注入 builder code。

### 3.10 Raw API / 手动签名：不建议优先做

如果不用 SDK，V2 raw signing 必须更新：

- EIP-712 Exchange domain version：`"1"` -> `"2"`
- Exchange verifying contract：改成 V2 contracts
- Neg Risk exchange verifying contract：改成 V2 contracts
- Order type 移除：`taker`, `expiration`, `nonce`, `feeRateBps`
- Order type 新增：`timestamp`, `metadata`, `builder`
- `side` 在签名 payload 里仍是 `uint8`：`0 = BUY`, `1 = SELL`
- POST `/order` body 中 `side` 仍是 `"BUY"` / `"SELL"`
- L1/L2 auth headers 不变

本仓库建议：

- 第一阶段只走官方 V2 SDK。
- 不在业务代码里维护 raw EIP-712 signing。
- 如果后续必须做 raw REST，下沉到独立 `src/adapters/clobSignerV2.ts`，不要混进订单服务。

### 3.11 Open orders / order state：保留 fallback，但先信 V2 SDK

旧文档提到 `getOpenOrders()` 可能出现 `response.data is not iterable`，建议 direct fetch `GET /data/orders` fallback。

V2 实现策略：

1. 优先使用 V2 SDK `getOpenOrders()`。
2. 如果 SDK 抛结构兼容错误，再 direct fetch。
3. direct fetch 仍要使用 L2 auth headers。
4. 分页和 cursor 合并逻辑保留。

实现落点：

- `src/adapters/clobPrivate.ts`
- `src/services/orderService.ts`
- `src/services/reconcile.ts`

### 3.12 WebSocket：URL 不变，解析器继续宽松

旧文档中的 URL 保留：

- `wss://ws-subscriptions-clob.polymarket.com/ws/market`
- `wss://ws-subscriptions-clob.polymarket.com/ws/user`
- `wss://ws-live-data.polymarket.com`

官方 V2 FAQ 明确 WebSocket URL 不变，大多数 payload 不变。

保留旧文档运行策略：

- market WS 每 5 秒 ping
- reconnect backoff
- 增量订阅 + 全量重发双保险
- user WS 连接后拉 balance snapshot
- 用户事件后 debounce 刷新 balance
- 保留 recent raw messages
- parser 不强绑定单一 schema

建议新增兼容字段：

- `tick_size_change`
- `last_trade_price`
- `best_bid_ask`
- `new_market`
- `market_resolved`

实现落点：

- `src/ws/marketWs.ts`
- `src/ws/userWs.ts`
- `src/state/orderBook.ts`
- `src/state/orderState.ts`

## 4. 推荐代码结构

```text
src/
  config/
    polymarket.ts
  adapters/
    gamma.ts
    dataApi.ts
    clobPublic.ts
    clobPrivate.ts
  services/
    marketResolver.ts
    quoteService.ts
    accountService.ts
    collateralService.ts
    orderService.ts
    backfill.ts
    reconcile.ts
  ws/
    marketWs.ts
    userWs.ts
  state/
    orderBook.ts
    orderState.ts
  domain/
    markets.ts
    orders.ts
    balances.ts
```

## 5. V2 最小落地顺序

### Phase 0: 配置与依赖

- 安装 `@polymarket/clob-client-v2`。
- 删除或停止使用 `@polymarket/clob-client`。
- 新增 `POLYMARKET_CLOB_BASE`，测试期指向 `https://clob-v2.polymarket.com`。
- 保留 `POLYMARKET_CHAIN_ID=137`，但代码里映射到 `chain`。
- 新增可选 `POLYMARKET_BUILDER_CODE`。

### Phase 1: 只读链路

- 复用 Gamma market resolver。
- 复用 token extraction。
- 复用 `/price` 和 `/book`。
- 新增 `getClobMarketInfo(conditionID)`。
- quote snapshot 加上 minimum tick size、minimum order size、fee details。

### Phase 2: 账户链路

- 用 V2 client 复用已有 API creds。
- 保留 derive/create fallback。
- 余额和 allowance 逻辑改名为 collateral，不再写死 USDC.e。
- 增加 pUSD readiness check。
- API-only trader 增加 wrap pUSD 的操作说明或脚本。

### Phase 3: 下单链路

- 修改 client constructor。
- 删除 `feeRateBps`、`nonce`、`taker`。
- limit order 加 `builderCode` 注入能力。
- market order 支持 `userUSDCBalance`。
- 下单前校验：balance、allowance、order book、market info、liquidity、tick size、minimum size。

### Phase 4: 实时与对账

- 保留 market/user WS URL。
- parser 增加 V2 文档里的事件类型。
- 启动后拉一次 open orders。
- 重连后跑 backfill 和 reconcile。
- 2026-04-28 切换后自动重建 open orders，不假设旧订单仍存在。

## 6. 迁移检查清单

依赖：

- [ ] `@polymarket/clob-client-v2` 已安装
- [ ] 旧 `@polymarket/clob-client` 不再被业务代码 import
- [ ] 代码里没有 `py-clob-client` V1 用法

配置：

- [ ] `POLYMARKET_CLOB_BASE` 可配置
- [ ] 测试期指向 `https://clob-v2.polymarket.com`
- [ ] 切换后默认 `https://clob.polymarket.com`
- [ ] `chainId` 输入统一映射为 V2 constructor 的 `chain`

订单：

- [ ] 没有业务输入字段 `feeRateBps`
- [ ] 没有业务输入字段 `nonce`
- [ ] 没有业务输入字段 `taker`
- [ ] builder attribution 使用 `builderCode`
- [ ] market buy order 可传 collateral balance 给 SDK

资金：

- [ ] 不再硬编码 USDC.e 为交易 collateral
- [ ] pUSD balance / allowance 检查完成
- [ ] API-only trader 有 USDC.e -> pUSD wrap 路径
- [ ] 金额归一化仍按 6 decimals

实时：

- [ ] WS URL 保持不变
- [ ] user WS auth 继续使用 apiKey / secret / passphrase
- [ ] parser 兼容 `book`、`price_change`、`tick_size_change`、`last_trade_price`、`best_bid_ask`
- [ ] reconnect 后会 backfill / reconcile

切换日：

- [ ] 2026-04-28 约 11:00 UTC 停止自动挂新单
- [ ] 迁移窗口后重新拉 market info / balance / allowance / open orders
- [ ] 不尝试恢复旧 open order ID
- [ ] 重新按当前策略挂单

## 7. 对旧文档的修正建议

`docs/polymarket-knowledge-base.md` 可以保留为 V1/V2 通用知识库，但建议后续补充：

- CLOB SDK 改为 V2。
- constructor 改为 options object。
- collateral 章节从 USDC.e 更新为 pUSD。
- 下单字段删除 `feeRateBps`、`nonce`、`taker`。
- fee 章节说明 `getClobMarketInfo()`。
- builder 章节从 HMAC headers 改成 `builderCode`。

`docs/polymarket-cookbook.md` 建议后续拆成：

- `polymarket-cookbook-v1.md`
- `polymarket-cookbook-v2.md`

原因是 cookbook 需要可直接复制执行，V1/V2 混写容易导致生产事故。
