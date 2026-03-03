# Redis 事件总线（内部通信）参考

本页定义服务间内部通信契约，作为 HTTP / WebSocket 外部 API 的补充。

## 适用范围

- 面向后端服务之间通信（Strategy / Risk / Order / Account / Monitor / Feature）。
- 不面向浏览器前端直连。
- 前端对接仍以 HTTP / WebSocket 为准。

## 通信分层

### 1) Redis Pub/Sub（低频业务事件）

- 典型场景：`signal_generated`、`order_approved`、`risk_rejected`。
- 语义：`at-most-once`（订阅者不在线时可能丢消息）。
- 用法：实时触发主流程。

### 2) Redis Streams（高频数据流）

- 典型场景：`aggTrades` 高频数据。
- 语义：`at-least-once`（消费者组 + ACK）。
- 用法：高吞吐数据管线与短期回放。

## Pub/Sub 通道契约

| 通道 | 发布者 | 订阅者 | 用途 |
|------|--------|--------|------|
| `quant:dollar_bar_generated` | DollarBarService | TickFeatureService | Dollar Bar 产出事件 |
| `quant:tick_features_enriched` | TickFeatureService | BarSourceAdapter | Tick 特征增强完成 |
| `quant:kline_raw` | DirectKlineService | BarSourceAdapter | 直拉 Kline 原始事件 |
| `quant:bar_normalized` | BarSourceAdapter | FeatureService | 统一 Bar 契约事件 |
| `quant:bar_source_mismatch` | BarSourceAdapter | MonitorService | 双源对账偏差告警 |
| `quant:signal_generated` | StrategyService | RiskService | 策略信号产出 |
| `quant:order_approved` | RiskService | OrderService | 风控通过，允许执行 |
| `quant:risk_rejected` | RiskService | MonitorService | 风控拒绝告警 |
| `quant:order_executed` | OrderService | AccountService, MonitorService | 订单执行成功 |
| `quant:order_failed` | OrderService | MonitorService | 订单执行失败 |
| `quant:account_updated` | AccountService | RiskService | 账户状态更新 |

## Streams / Redis 键约定

| 键模式 | 类型 | 用途 |
|--------|------|------|
| `quant:stream:aggTrades:{symbol}` | Stream | 高频逐笔成交数据（MAXLEN ~100000） |
| `quant:dollar_bar:{symbol}` | List | Dollar Bar 缓存（最近 200 条） |
| `quant:kline:raw:{symbol}:{timeframe}` | String(JSON) | 直拉 Kline 最新快照 |
| `quant:bar:normalized:{symbol}:{timeframe}` | String(JSON) | 统一 Bar 快照 |
| `quant:account:balance` | String(JSON) | 账户余额快照 |
| `quant:account:positions` | String(JSON) | 持仓快照 |
| `quant:account:orders` | String(JSON) | 活跃挂单快照 |
| `quant:portfolio:snapshot` | String(JSON) | NAV / PnL / 回撤等组合指标快照（AccountService 定时写入） |
| `quant:signal:latest` | String(JSON) | 最新策略目标仓位 |
| `quant:heartbeat:{service}` | String(JSON+TTL) | 服务心跳 |
| `quant:state:{service}:last_run` | String | 上次执行时间 |
| `quant:state:{service}:status` | String | 运行状态（`running|idle|error`） |

## 消息结构建议（内部统一）

为减少跨服务歧义，推荐所有 Pub/Sub 消息采用统一 JSON 外壳：

```json
{
  "event_type": "signal_generated",
  "source_service": "strategy_service",
  "timestamp": "2026-03-02T23:59:00Z",
  "trace_id": "trc-20260302-0001",
  "data": {}
}
```

字段说明：

- `event_type`: 事件类型（与通道语义一致）。
- `source_service`: 事件生产者服务名。
- `timestamp`: ISO 8601 UTC。
- `trace_id`: 跨服务追踪 ID（可选但强烈推荐）。
- `data`: 事件业务载荷。

## 可靠性与恢复约定

1. 主路径采用 Pub/Sub 实时触发。
2. 服务必须实现定时兜底任务（防止 Pub/Sub 丢消息导致流程中断）。
3. 服务重启后需读取 `quant:state:{service}:*` 做补执行判断。
4. Order 相关消费侧必须实现幂等（防止重复执行下单）。
5. `hybrid` 模式建议并行消费 `tick_agg` 与 `direct_kline`，并基于 `bar_source_mismatch` 做自动降级。

## 与外部 API 的边界

- 对外查询与控制：使用 [HTTP API](http.md)。
- 对外实时推送：使用 [WebSocket API](websocket.md)。
- 服务内部事件流：使用本文档定义的 Redis 契约。
