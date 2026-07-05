# WebSocket API

> 对外实时推送。内部事件见 [Redis 事件总线](redis.md)。
>
> **账户维度**：仓位/组合推送按账户作用域（每栈一条 NAV）；服务状态/告警跨栈聚合。

## 一、连接

```
ws://localhost:8000/ws/positions?account=acctA
ws://localhost:8000/ws/portfolio?account=acctA
ws://localhost:8000/ws/alerts
ws://localhost:8000/ws/services
```

`positions` / `portfolio` 必带 `?account=`；`alerts` / `services` 跨栈聚合，无需 account。

## 二、通用协议

消息统一：`{ "type": "...", "data": {…}, "timestamp": "ISO8601 UTC" }`。

- 心跳：客户端每 30s 发 `{"type":"ping"}`，服务端回 `{"type":"pong","timestamp":"…"}`；>90s 无 ping 服务端可断开。
- 错误：`{"type":"error","data":{"code":50001,"message":"…"}}`。
- 重连：收到 `close`/`error` 后指数退避（`3s→6s→12s→24s`，最多 10 次）；重连后服务端推全量快照。

## 三、端点

### 3.1 `/ws/positions?account={account}` — 仓位快照（每 30s）

数据源 `quant:{account}:account:positions` + `…:signal:latest`。

首条 `positions_snapshot` / 增量 `positions_update`：

```json
{ "type": "positions_snapshot", "data": {
  "account": "acctA", "strategy_name": "ml_dollar_bar", "signal_timestamp": "…",
  "positions": [
    { "symbol": "BTC/USDT:USDT", "side": "long", "contracts": 0.0135, "entry_price": 62450.0, "mark_price": 62800.0,
      "actual_weight": 0.0498, "target_weight": 0.05, "deviation_pct": -0.4, "unrealized_pnl": 4.7, "leverage": 1 }
  ],
  "summary": { "total_positions": 20, "long_count": 1, "short_count": 19, "total_deviation": 0.012, "missing_positions": 0, "extra_positions": 0 }
}, "timestamp": "…" }
```

`deviation_pct = (actual − target)/target × 100`。

### 3.2 `/ws/portfolio?account={account}` — 组合指标（每 60s）

数据源 `quant:{account}:portfolio:snapshot`。

```json
{ "type": "portfolio_snapshot", "data": {
  "account": "acctA", "strategy_name": "ml_dollar_bar", "dry_run": true, "trading_days": 61,
  "nav": 10234.56, "initial_nav": 10000.0, "total_return": 0.0235,
  "daily_pnl": -12.5, "daily_pnl_pct": -0.0012, "drawdown": -0.053, "max_drawdown": -0.082, "max_drawdown_date": "2026-02-20",
  "sharpe_30d": 1.85, "volatility_30d": 0.185, "calmar_30d": 2.47,
  "total_equity": 10234.56, "available_balance": 5120.3, "used_margin": 5114.26, "unrealized_pnl": 123.45
}, "timestamp": "…" }
```

增量 `portfolio_update` 推核心数值字段（`nav/total_return/daily_pnl/drawdown/sharpe_30d/volatility_30d/total_equity/unrealized_pnl`）。

### 3.3 `/ws/alerts` — 告警（实时，跨栈）

订阅各栈 `quant:{account}:risk_rejected` / `execution_rejected` / `order_failed`。

```json
{ "type": "alert", "data": {
  "alert_id": "alt-20260613-001", "severity": "critical", "account": "acctA",
  "category": "risk_rejected", "rule_name": "emergency_stop", "display_name": "全局急停",
  "message": "急停置位，拒批 rb-ml_dollar_bar-20260613T0007",
  "details": { "strategy_name": "ml_dollar_bar", "rebalance_id": "rb-…" },
  "timestamp": "…"
}, "timestamp": "…" }
```

`severity`：`info` \| `warning` \| `critical`；`category`：`risk_rejected` \| `execution_rejected` \| `order_failed`。

### 3.4 `/ws/services` — 服务状态（每 30s，跨栈）

数据源 `quant:heartbeat:{service}@{account}`。

```json
{ "type": "services_snapshot", "data": {
  "stacks": [
    { "account": "acctA", "strategy": "ml_dollar_bar", "status": "healthy",
      "services": [ { "name": "risk_service", "status": "healthy", "last_heartbeat": "…", "uptime_seconds": 86400, "latency_ms": 2 } ] },
    { "account": "acctB", "strategy": "monthly_majors_vs_alts", "status": "healthy", "services": [ … ] }
  ],
  "shared": [ { "name": "data_service", "status": "healthy", "last_heartbeat": "…", "latency_ms": 5 } ],
  "overall_status": "healthy"
}, "timestamp": "…" }
```

每栈服务集：`strategy_service` / `risk_service` / `execution_service` / `order_service` / `account_service`；共享层：`data_service` / `api_gateway` / `alert_service`。`status`：`healthy` \| `degraded`（latency >120000ms）\| `down`（心跳过期）。

## 四、说明

- ensemble / 多策略合并载荷不预埋——N 策略时各账户已有独立 `positions`/`portfolio` 推送，跨账户聚合是前端视图运算，需要时随实现定义。
- 信号变更不走 WS：scheduler 事件密度低（轮询足够）；streaming 策略状态用 `GET /api/v1/strategies/{name}/state` 轮询。
