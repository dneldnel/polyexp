# Polymarket Knowledge Base

更新时间：2026-03-29

本文目标是提供一份自洽、可直接落地的 Polymarket 集成知识库，而不是只做概念摘要。

## 1. 接口分层

工程上建议把 Polymarket 交互拆成四套接口面：

1. `Gamma API`
   用于找市场、按 series/event 搜市场、拿 market 元信息。
2. `Data API`
   用于查用户活动、持仓、已平仓记录、组合价值。
3. `CLOB REST`
   用于报价、订单簿、余额/allowance、下单、撤单、查订单。
4. `WebSocket`
   用于市场盘口实时流、用户订单/成交事件、公开活动事件、UI 使用的 live 价格流。

默认地址：

- `https://gamma-api.polymarket.com`
- `https://data-api.polymarket.com`
- `https://clob.polymarket.com`
- `wss://ws-subscriptions-clob.polymarket.com/ws/market`
- `wss://ws-subscriptions-clob.polymarket.com/ws/user`
- `wss://ws-live-data.polymarket.com`

## 2. 市场搜索与市场信息获取

### 2.1 最常用的市场搜索入口

市场发现通常走以下路径：

- `GET /markets?slug=<market_slug>`
  通过固定 slug 精确查市场。
- `GET /events?series_id=<id>&active=true&closed=false`
  通过 series 找 live events，再从 event 里展平 markets。
- `GET /series?slug=<series_slug>`
  先把 series slug 转成 series id，再继续走 events。
- `GET /markets?seriesSlug=<series_slug>&active=true&closed=false&enableOrderBook=true`
  直接按 series slug 拉一批市场。
- `GET /markets?active=true&closed=false&enableOrderBook=true&limit=<n>&offset=<n>`
  最后兜底，全量分页扫描活跃市场。

### 2.2 推荐的市场解析流程

推荐流程是：

1. 如果用户指定了固定 `market slug`，直接查该市场。
2. 否则优先用 `series_id -> /events -> flattenEventMarkets()`。
3. 如果只有 `series_slug`，先 `/series` 找到 `series_id`，再查 `/events`。
4. 失败后回退到 `/markets?seriesSlug=...`。
5. 再失败才做活跃市场分页扫描。
6. 对候选市场使用 `pickLatestLiveMarket()`，取当前 live 或最近即将生效的市场。

### 2.3 market / condition / token 的关系

核心关系是：

- 一个 market 通常对应一个 `conditionId`
- 一个二元市场通常会有两个 outcome，例如 `Up` / `Down`
- `market.outcomes` 和 `market.clobTokenIds` 一一对应
- 真正下单、订阅盘口、查报价时使用的是 `tokenId`

也就是说：

- 找市场：通常从 `slug / series / event / market`
- 交易和盘口：最终落到 `tokenId`

### 2.4 解析 market 时的坑

高频坑之一是：

- `outcomes` 可能是数组
- 也可能是 JSON 字符串
- `clobTokenIds` 也是一样

所以不能假设：

- `market.outcomes` 一定是数组
- `market.clobTokenIds` 一定是数组

正确做法是先兼容解析，再按 outcome label 映射 token：

- `Up` -> `upTokenId`
- `Down` -> `downTokenId`

如果 label 配错，后续所有报价和下单都会错 token。

### 2.5 market slug 的时间语义

很多运行时逻辑都会依赖 slug 解析，例如：

- `btc-updown-5m-1771518900`

这个 slug 通常编码了：

- 标的
- 市场类型
- 时间窗口长度
- 窗口起始 epoch 秒

这套语义被用来做：

- 当前桶窗口计算
- 下一市场 slug 推导
- 市场 rollover 切换

如果后面做周期类市场套利，这块逻辑值得直接复用。

## 3. 选项信息、当前报价与盘口

### 3.1 选项信息

对于二元市场，选项信息主要来自 market 元数据：

- `outcomes`
- `outcomePrices`
- `clobTokenIds`

常见用法：

- 从 `outcomes` 找到 `Up/Down`
- 从同位置的 `clobTokenIds` 找到对应 token
- 从 `outcomePrices` 取 UI 侧概率/价格参考

### 3.2 当前报价

常见报价接口是：

- `GET /price?token_id=<tokenId>&side=<buy|sell>`

实践上最常见的是：

- `side=buy`

因为它直接给出“现在买这个 outcome 的价格”。

### 3.3 当前盘口

盘口使用：

- `GET /book?token_id=<tokenId>`

订单簿摘要通常至少要抽出：

- `bestBid`
- `bestAsk`
- `spread`
- 某个深度内的 `bidLiquidity`
- 某个深度内的 `askLiquidity`

实战上，order book 应该被当成 live validation 的核心输入，而不是只看单个 price。

### 3.4 推荐的报价流程

推荐的组合式快照流程：

1. 先解析当前 market。
2. 从 market 里拿到 `upTokenId` / `downTokenId`。
3. 并行调用：
   - `/price` 获取买价
   - `/book` 获取盘口
4. 如果 CLOB 报价失败，再退回到 market 自带的 `outcomePrices`。

这是一条很适合直接复用的基线流程。

## 4. 账户、持仓、历史与用户数据

Data API 通常负责账户、持仓、成交与历史数据。

### 4.1 账户组合价值

- `GET /value?user=<address>`

用于获取：

- 账户总价值
- 组合价值快照

### 4.2 当前持仓

- `GET /positions?user=<address>&sizeThreshold=<n>`

用于获取：

- 当前未平仓 positions
- token / condition / size / avgPrice / currentValue 等字段

### 4.3 已平仓记录

- `GET /closed-positions?user=<address>`

支持常见过滤：

- `market`
- `eventId`
- `title`
- `limit`
- `offset`
- `sortBy`
- `sortDirection`

适合做：

- 历史结算统计
- PnL 统计
- 胜率和 ROI 报表

### 4.4 成交历史 / 活动历史

实际接入时建议兼容以下几种形态：

- `GET /activity?user=<address>&type=TRADE`
- `GET /activity?user=<address>&activityType=TRADE`
- `GET /trades?user=<address>`

这是因为不同接口形态或兼容层返回字段会有差异。

### 4.5 公开活动流与 backfill

对于公开活动流，推荐使用下面这套模式：

- WebSocket 订阅 leader 的公开 trade activity
- 断线或重启后再用 `Data API /activity` 补历史缺口

这套模式对后续项目非常有用，尤其当后面你想做：

- 跟单
- 对手盘观察
- 某些地址驱动的策略

## 5. CLOB 认证、余额与下单

## 5.1 认证参数

CLOB 私有接口通常需要以下信息：

- `privateKey`
- `chainId`
- `signatureType`
- 可选 `funder/proxy wallet`
- L2 API creds:
  - `apiKey`
  - `secret`
  - `passphrase`

### 5.2 signature type

项目内已经使用并固化了三种签名类型：

- `EOA` = `0`
- `POLY_PROXY` = `1`
- `POLY_GNOSIS_SAFE` = `2`

关键经验：

- 如果 `signer address != funder address`
- 且没有显式设置 signature type
- 建议默认按 `POLY_GNOSIS_SAFE = 2` 的方向排查

原因是很多 Polymarket web 场景本质上是 safe / funder 模式。

### 5.3 API key 获取策略

推荐按以下顺序尝试：

1. 先尝试 `deriveApiKey()`
2. 如果不行，再尝试 `createApiKey()` 或 `createOrDeriveApiKey()`

实战经验说明：

- 首次接入时，经常需要 create 或 createOrDerive
- 后续可以直接复用现成的 `apiKey/secret/passphrase`

### 5.4 余额与 allowance

查询资金侧常用的是：

- `getBalanceAllowance({ asset_type: "COLLATERAL" })`
- `updateBalanceAllowance({ asset_type: "COLLATERAL" })`

非常关键的经验：

- 在 `getBalanceAllowance()` 之前建议先主动 `updateBalanceAllowance()`
- 否则服务端缓存可能导致 allowance 一直是旧值，甚至看起来像 `0`

另一个坑：

- 余额和 allowance 很多时候是 USDC base units
- 也就是 6 位小数的整数值
- 但有些包装层也可能直接返回十进制字符串

所以解析时要同时兼容：

- `1234567` -> `1.234567`
- `1.234567` -> `1.234567`

### 5.5 限价单

常见调用形态：

- `createAndPostOrder(userOrder, options, orderType, deferExec, postOnly)`

基础字段：

- `tokenID`
- `price`
- `size`
- `side`

可选项：

- `tickSize`
- `negRisk`
- `feeRateBps`
- `nonce`
- `expiration`
- `taker`
- `orderType = GTC | GTD`

### 5.6 市价型订单

很多运行时会更偏向用 market order 风格下单：

- `createAndPostMarketOrder(userMarketOrder, options, orderType, deferExec)`

核心字段：

- `tokenID`
- `amount`
- `side`

常见订单类型：

- `FOK`
- `FAK`

比较常见的 live 执行选择是：

- `OrderType.FOK`

发 market order 前建议先查：

- `getTickSize(tokenID)`
- `getNegRisk(tokenID)`

再把这些参数一起传给下单函数。

这说明在实现里不要把 market order 写成“只塞 tokenID + amount + side”这么简单。

### 5.7 下单前的实时校验

推荐的 live 校验范式：

下单前至少查：

- 当前 `balance`
- 当前 `allowance`
- 当前 `orderBook`
- 当前 `marketReferencePrice`
- 当前可交易方向上的 `liquidity`

这样可以在真正发单前给出机器可读的 skip / reject 原因。

### 5.8 下单结果应该怎么理解

必须区分：

1. 下单请求被撮合系统接受
2. 订单真正成交

工程上不能把“提交成功”误判为“成交成功”。

推荐定义：

- `HTTP success + response.success/orderID`
  只代表订单被接受
- 成交与否仍要看：
  - User WS 事件
  - `getOrder(orderId)`
  - `getTrades` 或 Data API 成交记录

### 5.9 撤单

已验证的撤单能力包括：

- `cancelOrder(orderId)`
- `cancelOrders(orderIds)`
- `cancelAllOrders()`
- `cancelMarketOrders({ market, assetId })`

### 5.10 open orders 查询兼容性

`open orders` 查询有一个常见兼容性问题：

- 某些版本或某些响应形态下，`getOpenOrders()` 可能报
  `response.data is not iterable`

它的处理方式不是直接放弃，而是：

1. 自己生成 L2 headers
2. 直接请求 `GET /data/orders`
3. 用 cursor 分页把多页结果合并

这说明未来如果封装 open orders，最好保留一个 direct fetch fallback。

## 6. WebSocket

## 6.1 CLOB Market WS

市场 WS 建议这样组织：

地址：

- `wss://ws-subscriptions-clob.polymarket.com/ws/market`

常见订阅消息形态：

```json
{
  "type": "market",
  "assets_ids": ["token_id_1", "token_id_2"]
}
```

增量切换时还会发：

```json
{
  "assets_ids": ["token_id"],
  "operation": "subscribe"
}
```

或：

```json
{
  "assets_ids": ["token_id"],
  "operation": "unsubscribe"
}
```

运行时经验：

- 保持 5 秒一个 `"PING"`
- 切换订阅时做“增量订阅 + 全量重发”双保险
- reconnect 要带 backoff
- 解析器不要强绑定单一 schema

当前解析器兼容的事件形态包括：

- `book`
- `price_changes`
- `changes`
- `price_change`
- 顶层直接是 `event_type=book/price_change`

### 6.2 User WS

地址：

- `wss://ws-subscriptions-clob.polymarket.com/ws/user`

推荐的订阅消息：

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

这条流在项目里被用来实时接收：

- order 事件
- trade/fill 事件
- money/balance 事件

推荐在本地统一归一化为：

- `order`
- `trade`
- `money`

并为订单建立状态机：

- `PENDING_SUBMIT`
- `SUBMITTED`
- `LIVE`
- `PARTIALLY_FILLED`
- `FILLED`
- `CANCELED`
- `REJECTED`
- `EXPIRED`

### 6.3 User WS 的运行时策略

用户 WS 不应该只做“连上就收消息”，而应该带完整运维逻辑：

- ping/pong
- stale connection 检测
- reconnect backoff
- connect 后拉一次 balance snapshot
- 收到用户事件后 debounce 刷新资金快照
- 可选周期性 balance poll
- 保存 recent raw messages / recent normalized events 便于 debug

这是比较值得直接吸收的实现模式。

### 6.4 Live Price WS

常见 live price WS 地址：

- `wss://ws-live-data.polymarket.com`

订阅：

```json
{
  "action": "subscribe",
  "subscriptions": [
    {
      "topic": "crypto_prices_chainlink",
      "type": "*",
      "filters": ""
    }
  ]
}
```

这条流适合做：

- BTC/ETH 等参考价格
- bucket 计时
- 概率市场与底层标的价格的联动逻辑

### 6.5 公开活动 WS

如果要接公开活动流，常见方式是通过 `@polymarket/real-time-data-client` 订阅：

- `topic = "activity"`
- `type = "trades"`

收到消息后再过滤：

- `payload.proxyWallet === leaderWallet`

适用场景：

- 跟单
- 地址行为分析
- 做事件驱动回放和补偿

## 7. 工程实践与踩坑结论

### 7.1 WS 负责快，REST 负责准

推荐默认原则：

- WS 用来抢实时性
- REST 用来做对账、补偿、重放、权威确认

### 7.2 任何资金/订单逻辑都要保留原始响应

应至少持久化：

- request payload
- raw response
- order id
- 本地 client order id
- 时间戳
- 最后一条 WS 事件
- 最后一条 REST snapshot

### 7.3 下单要幂等

不做幂等的后果：

- 重试时重复下单
- 断线恢复后难以判断哪笔已提交

### 7.4 reconnect 后一定要补历史

推荐保留一套 backfill 策略：

- 记住最后看到的 trade marker
- 重连或重启后从 `/activity` 做时间窗口回补
- 对重复事件去重

### 7.5 运行中要做 reconciliation

仅靠事件流无法保证账户状态长期无漂移。

推荐定时对比：

- leader positions
- follower positions

可发现：

- 漏单
- 未完成卖出
- size mismatch
- 账户侧残留仓位

### 7.6 invalid signature 往往不是“单纯签名坏了”

最常见原因通常是：

- signer/funder 不匹配
- signature type 配错

如果：

- 私钥对应地址不是实际交易账户
- 但你又在用 funder/proxy/safe

那就要重点检查：

- `POLYMARKET_SIGNATURE_TYPE`
- `POLYMARKET_FUNDER`

### 7.7 市场字段要做宽松解析

尤其是：

- `outcomes`
- `clobTokenIds`
- WS payload 里的 `payload/data/events/orders/trades`

不能写成只接受一种 shape 的强假设代码。

## 8. 建议模块拆分

推荐直接按下面拆：

1. `src/adapters/gamma.ts`
   封装 `/markets`、`/series`、`/events`
2. `src/adapters/dataApi.ts`
   封装 `/activity`、`/positions`、`/closed-positions`、`/value`
3. `src/adapters/clobPublic.ts`
   封装 `/price`、`/book`
4. `src/adapters/clobPrivate.ts`
   封装 auth、余额、allowance、下单、撤单、查订单
5. `src/services/marketResolver.ts`
   做 slug/series/event 到 token 的解析
6. `src/services/quoteService.ts`
   做价格、盘口、流动性快照
7. `src/ws/marketWs.ts`
   维护市场盘口订阅与重连
8. `src/ws/userWs.ts`
   维护用户订单/成交/资金事件订阅
9. `src/services/backfill.ts`
   做 reconnect / restart 后数据补偿
10. `src/services/reconcile.ts`
    做仓位或订单状态纠偏

## 9. 建议优先级

如果接下来要开始写代码，建议顺序是：

1. 市场搜索与 token 解析
2. 公开报价与盘口快照
3. CLOB 私有认证与余额检查
4. 下单与撤单
5. User WS 与订单状态机
6. Market WS 与本地 order book
7. backfill / reconcile / monitor
