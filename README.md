# polexp

目前先作为新的 Polymarket 项目的知识基座。

我已经把与 Polymarket 交互最核心的知识整理到这里，重点覆盖：

- 市场搜索与市场选择
- market / condition / token 的关系
- 当前报价、盘口、流动性获取
- 账户、持仓、成交、订单查询
- CLOB 认证、余额/allowance、下单、撤单
- 市场 WS、用户 WS、活动 WS、重连与补偿
- 适合后续项目直接复用的模块边界

文档索引：

- [docs/polymarket-knowledge-base.md](./docs/polymarket-knowledge-base.md)
- [docs/polymarket-cookbook.md](./docs/polymarket-cookbook.md)
- [notes/polymarket-implementation-plan.md](./notes/polymarket-implementation-plan.md)

当前结论：

- Polymarket 相关能力至少分成四层：`Gamma API`、`Data API`、`CLOB REST`、`WebSocket`
- 搜市场主要依赖 `Gamma API`，下单和余额主要依赖 `CLOB`
- 实时状态要靠 `WS`，但成交与资金核对应始终由 `REST` 做补偿和纠偏
- 文档现在是自洽的，不要求再回头查看其他目录才能理解 Polymarket 交互方式
- 后续如果开始写代码，建议按 `市场发现 -> 报价/盘口 -> 账户校验 -> 下单 -> 用户回报 -> 对账` 的顺序推进
