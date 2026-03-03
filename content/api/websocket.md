# WebSocket API 参考

本页定义对外 WebSocket 实时推送接口。服务间内部事件通信请参考 [Redis 事件总线](redis.md)。

版本说明：

| 版本 | 端点数 | 差异 |
|------|--------|------|
| **V1 (MVP)** | 4 个 | 单策略 (baseline_rev)，基础仓位/组合/告警/服务状态推送 |
| **V2** | 4 + 1 个 | 多策略支持，增强载荷字段，新增 `/ws/signals` |

---

## 一、连接信息

### 开发环境

```
ws://localhost:8000/ws/positions
ws://localhost:8000/ws/portfolio
ws://localhost:8000/ws/alerts
ws://localhost:8000/ws/services
ws://localhost:8000/ws/signals        [V2]
```

### 生产环境

```
wss://api.quant-trading.example.com/ws/positions
```

---

## 二、通用协议

### 2.1 消息结构

所有服务端推送消息统一为：

```json
{
  "type": "positions_update",
  "data": { },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string | 消息类型标识 |
| `data` | object | 业务载荷 |
| `timestamp` | string (ISO 8601) | 消息时间 (UTC) |

### 2.2 心跳协议

客户端每 30s 发送心跳，服务端响应：

```json
// 客户端 → 服务端
{ "type": "ping" }

// 服务端 → 客户端
{ "type": "pong", "timestamp": "2026-03-02T23:59:00Z" }
```

超过 90s 未收到客户端 ping，服务端可主动断开连接。

### 2.3 错误消息

当发生异常时推送：

```json
{
  "type": "error",
  "data": {
    "code": 50001,
    "message": "Redis connection lost, retrying..."
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

### 2.4 重连建议

- 客户端收到 `close` / `error` 后做**指数退避重连**
- 退避窗口：`3s → 6s → 12s → 24s`，最大重试 10 次
- 重连成功后服务端自动推送最新全量快照

---

## 三、V1 端点

### 3.1 `/ws/positions` — 仓位快照

**推送频率**: 每 30 秒

**推送内容**: 当前仓位快照 + 目标仓位偏差

**数据来源**: Redis `quant:account:positions` + `quant:signal:latest`

**后端实现路径**: `controller/routes/ws_positions.py`

#### 首条全量快照 (连接/重连后)

```json
{
  "type": "positions_snapshot",
  "data": {
    "positions": [
      {
        "symbol": "BTC/USDT:USDT",
        "side": "long",
        "contracts": 0.005,
        "entry_price": 62450.00,
        "mark_price": 62800.00,
        "actual_weight": 0.0155,
        "target_weight": 0.0167,
        "deviation_pct": -7.2,
        "unrealized_pnl": 1.75,
        "leverage": 1
      },
      {
        "symbol": "DOGE/USDT:USDT",
        "side": "short",
        "contracts": 500.0,
        "entry_price": 0.1520,
        "mark_price": 0.1505,
        "actual_weight": -0.0162,
        "target_weight": -0.0167,
        "deviation_pct": 3.0,
        "unrealized_pnl": 0.75,
        "leverage": 1
      }
    ],
    "summary": {
      "total_positions": 45,
      "long_count": 30,
      "short_count": 15,
      "total_deviation": 0.045,
      "missing_positions": 5,
      "extra_positions": 2
    },
    "strategy_name": "baseline_rev",
    "signal_timestamp": "2026-03-02T23:55:00Z"
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

#### 增量更新 (每 30s)

```json
{
  "type": "positions_update",
  "data": {
    "positions": [ ],
    "summary": {
      "total_positions": 45,
      "long_count": 30,
      "short_count": 15,
      "total_deviation": 0.042,
      "missing_positions": 4,
      "extra_positions": 2
    },
    "strategy_name": "baseline_rev",
    "signal_timestamp": "2026-03-02T23:55:00Z"
  },
  "timestamp": "2026-03-02T23:59:30Z"
}
```

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `positions[].symbol` | string | 交易对 (CCXT 统一格式) |
| `positions[].side` | string | `"long"` \| `"short"` |
| `positions[].contracts` | float | 持仓合约数 |
| `positions[].entry_price` | float | 开仓均价 |
| `positions[].mark_price` | float | 标记价格 |
| `positions[].actual_weight` | float | 实际仓位权重 (占总权益比) |
| `positions[].target_weight` | float | 目标仓位权重 (来自策略信号) |
| `positions[].deviation_pct` | float | 偏差百分比 `(actual - target) / target × 100` |
| `positions[].unrealized_pnl` | float | 未实现盈亏 (USDT) |
| `positions[].leverage` | int | 杠杆倍数 |
| `summary.total_positions` | int | 持仓总数 |
| `summary.long_count` | int | 多头数量 |
| `summary.short_count` | int | 空头数量 |
| `summary.total_deviation` | float | 平均偏差绝对值 |
| `summary.missing_positions` | int | 目标有但实际无的仓位数 |
| `summary.extra_positions` | int | 实际有但目标无的仓位数 |
| `strategy_name` | string | V1 固定 `"baseline_rev"` |
| `signal_timestamp` | string (ISO 8601) | 信号生成时间 |

---

### 3.2 `/ws/portfolio` — 组合指标

**推送频率**: 每 60 秒

**推送内容**: NAV / PnL / 回撤 / 波动率核心指标

**数据来源**: Redis `quant:portfolio:snapshot`

**后端实现路径**: `controller/routes/ws_portfolio.py`

#### 首条全量快照

```json
{
  "type": "portfolio_snapshot",
  "data": {
    "nav": 10234.56,
    "initial_nav": 10000.00,
    "total_return": 0.023456,
    "daily_pnl": -12.50,
    "daily_pnl_pct": -0.0012,
    "drawdown": -0.053,
    "max_drawdown": -0.082,
    "max_drawdown_date": "2026-02-20",
    "sharpe_30d": 1.85,
    "volatility_30d": 0.185,
    "calmar_30d": 2.47,
    "win_rate_30d": 0.62,
    "total_equity": 10234.56,
    "available_balance": 5120.30,
    "used_margin": 5114.26,
    "unrealized_pnl": 123.45,
    "strategy_name": "baseline_rev",
    "dry_run": true,
    "trading_days": 61
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

#### 增量更新 (每 60s)

```json
{
  "type": "portfolio_update",
  "data": {
    "nav": 10234.56,
    "total_return": 0.023456,
    "daily_pnl": -12.50,
    "daily_pnl_pct": -0.0012,
    "drawdown": -0.053,
    "max_drawdown": -0.082,
    "sharpe_30d": 1.85,
    "volatility_30d": 0.185,
    "total_equity": 10234.56,
    "unrealized_pnl": 123.45
  },
  "timestamp": "2026-03-02T24:00:00Z"
}
```

#### 字段说明

| 字段 | 类型 | 全量 | 增量 | 说明 |
|------|------|:----:|:----:|------|
| `nav` | float | ✓ | ✓ | 当前 NAV 净值 |
| `initial_nav` | float | ✓ | | 初始 NAV |
| `total_return` | float | ✓ | ✓ | 累计收益率 |
| `daily_pnl` | float | ✓ | ✓ | 当日 PnL (USDT) |
| `daily_pnl_pct` | float | ✓ | ✓ | 当日 PnL 百分比 |
| `drawdown` | float | ✓ | ✓ | 当前回撤 (负数) |
| `max_drawdown` | float | ✓ | ✓ | 最大历史回撤 |
| `max_drawdown_date` | string | ✓ | | 最大回撤发生日期 |
| `sharpe_30d` | float | ✓ | ✓ | 30 日滚动 Sharpe |
| `volatility_30d` | float | ✓ | ✓ | 30 日滚动年化波动率 |
| `calmar_30d` | float | ✓ | | 30 日 Calmar 比率 |
| `win_rate_30d` | float | ✓ | | 30 日胜率 |
| `total_equity` | float | ✓ | ✓ | 总权益 (USDT) |
| `available_balance` | float | ✓ | | 可用余额 |
| `used_margin` | float | ✓ | | 已用保证金 |
| `unrealized_pnl` | float | ✓ | ✓ | 未实现盈亏 |
| `strategy_name` | string | ✓ | | V1: `"baseline_rev"` |
| `dry_run` | bool | ✓ | | 是否为模拟盘 |
| `trading_days` | int | ✓ | | 运行交易日数 |

---

### 3.3 `/ws/alerts` — 告警推送

**推送频率**: 实时 (事件触发)

**推送内容**: 风控告警 + 执行告警

**数据来源**: 订阅 Redis Pub/Sub `quant:risk_rejected`, `quant:order_failed`

**后端实现路径**: `controller/routes/ws_alerts.py`

#### 风控拒绝告警

```json
{
  "type": "alert",
  "data": {
    "alert_id": "alt-20260302-001",
    "severity": "critical",
    "category": "risk_rejected",
    "rule_name": "max_drawdown_halt",
    "display_name": "最大回撤止损",
    "message": "当前回撤 32% 超过阈值 30%, 触发暂停交易",
    "details": {
      "threshold": 0.30,
      "actual": 0.32,
      "strategy_name": "baseline_rev",
      "rebalance_id": "rb-20260302-001"
    },
    "timestamp": "2026-03-02T23:59:01Z"
  },
  "timestamp": "2026-03-02T23:59:01Z"
}
```

#### 订单失败告警

```json
{
  "type": "alert",
  "data": {
    "alert_id": "alt-20260302-002",
    "severity": "warning",
    "category": "order_failed",
    "rule_name": "order_execution_failed",
    "display_name": "下单执行失败",
    "message": "下单失败: ETH/USDT:USDT, InsufficientBalance",
    "details": {
      "symbol": "ETH/USDT:USDT",
      "side": "buy",
      "error_code": "InsufficientBalance",
      "retry_count": 3,
      "order_id": "ord-20260302-005"
    },
    "timestamp": "2026-03-02T23:59:08Z"
  },
  "timestamp": "2026-03-02T23:59:08Z"
}
```

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `alert_id` | string | 告警唯一 ID，格式 `alt-{YYYYMMDD}-{seq}` |
| `severity` | string | `"info"` \| `"warning"` \| `"critical"` |
| `category` | string | 告警来源类别: `"risk_rejected"` \| `"order_failed"` |
| `rule_name` | string | 触发的规则标识 |
| `display_name` | string | 规则显示名 (中文) |
| `message` | string | 可读告警描述 |
| `details` | object | 告警详细信息 (结构因 category 不同而异) |
| `timestamp` | string (ISO 8601) | 告警发生时间 |

---

### 3.4 `/ws/services` — 服务状态

**推送频率**: 每 30 秒

**推送内容**: 所有服务的心跳状态

**数据来源**: Redis `quant:heartbeat:{service}`

**后端实现路径**: `controller/routes/ws_services.py`

#### 首条全量快照

```json
{
  "type": "services_snapshot",
  "data": {
    "services": [
      {
        "name": "asset_pool_service",
        "display_name": "资产池",
        "status": "healthy",
        "last_heartbeat": "2026-03-02T23:58:30Z",
        "uptime_seconds": 86400,
        "latency_ms": 2,
        "metrics": {
          "symbols_count": 100,
          "last_update": "2026-03-02T00:00:00Z"
        }
      },
      {
        "name": "aggregator_service",
        "display_name": "K 线聚合",
        "status": "healthy",
        "last_heartbeat": "2026-03-02T23:58:28Z",
        "uptime_seconds": 86400,
        "latency_ms": 5,
        "metrics": {
          "klines_stored": 4200,
          "symbols_streaming": 100
        }
      },
      {
        "name": "feature_service",
        "display_name": "特征计算",
        "status": "degraded",
        "last_heartbeat": "2026-03-02T23:55:00Z",
        "uptime_seconds": 86400,
        "latency_ms": 240000,
        "metrics": {
          "last_calculation": "2026-03-02T23:54:50Z",
          "symbols_calculated": 98
        }
      },
      {
        "name": "strategy_service",
        "display_name": "策略引擎",
        "status": "healthy",
        "last_heartbeat": "2026-03-02T23:58:30Z",
        "uptime_seconds": 86400,
        "latency_ms": 3,
        "metrics": {
          "active_strategy": "baseline_rev",
          "last_signal_time": "2026-03-02T23:55:00Z"
        }
      },
      {
        "name": "risk_service",
        "display_name": "风控",
        "status": "healthy",
        "last_heartbeat": "2026-03-02T23:58:30Z",
        "uptime_seconds": 86400,
        "latency_ms": 2,
        "metrics": {
          "rules_active": 5,
          "last_check": "2026-03-02T23:55:01Z"
        }
      },
      {
        "name": "order_service",
        "display_name": "订单执行",
        "status": "healthy",
        "last_heartbeat": "2026-03-02T23:58:30Z",
        "uptime_seconds": 86400,
        "latency_ms": 2,
        "metrics": {
          "dry_run": true,
          "last_orders_count": 12
        }
      },
      {
        "name": "account_service",
        "display_name": "账户同步",
        "status": "healthy",
        "last_heartbeat": "2026-03-02T23:58:30Z",
        "uptime_seconds": 86400,
        "latency_ms": 2,
        "metrics": {
          "sync_interval": 30,
          "last_sync": "2026-03-02T23:58:30Z"
        }
      }
    ],
    "pipeline_status": "healthy",
    "total_services": 7,
    "healthy_count": 6,
    "degraded_count": 1,
    "down_count": 0
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

#### 增量更新 (每 30s)

```json
{
  "type": "services_update",
  "data": {
    "services": [ ],
    "pipeline_status": "healthy",
    "total_services": 7,
    "healthy_count": 6,
    "degraded_count": 1,
    "down_count": 0
  },
  "timestamp": "2026-03-02T23:59:30Z"
}
```

#### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `services[].name` | string | 服务标识名 |
| `services[].display_name` | string | 显示名 (中文) |
| `services[].status` | string | `"healthy"` \| `"degraded"` \| `"down"` |
| `services[].last_heartbeat` | string (ISO 8601) | 最后一次心跳时间 |
| `services[].uptime_seconds` | int | 连续运行秒数 |
| `services[].latency_ms` | int | 距上次心跳的延迟 (毫秒)。>120000 → `degraded`，heartbeat 过期 → `down` |
| `services[].metrics` | object | 服务特有指标 (结构因服务不同而异) |
| `pipeline_status` | string | 管线整体状态。全部 healthy → `"healthy"`；任一 degraded → `"degraded"`；任一 down → `"down"` |
| `total_services` | int | 总服务数 |
| `healthy_count` | int | 健康服务数 |
| `degraded_count` | int | 降级服务数 |
| `down_count` | int | 宕机服务数 |

#### V1 服务列表

| 服务名 | display_name | 说明 |
|--------|-------------|------|
| `asset_pool_service` | 资产池 | 24h 更新 |
| `aggregator_service` | K 线聚合 | 拉取 kline |
| `feature_service` | 特征计算 | ret/zscore |
| `strategy_service` | 策略引擎 | baseline_rev |
| `risk_service` | 风控 | 规则检查 |
| `order_service` | 订单执行 | 差量下单 |
| `account_service` | 账户同步 | 30s 轮询 |

---

## 四、V2 增强 [V2]

### 4.1 `/ws/positions` 增强 [V2]

V2 多策略模式下，仓位推送增加策略来源信息：

**新增字段**:

```json
{
  "type": "positions_update",
  "data": {
    "positions": [
      {
        "symbol": "BTC/USDT:USDT",
        "side": "long",
        "contracts": 0.005,
        "entry_price": 62450.00,
        "mark_price": 62800.00,
        "actual_weight": 0.0155,
        "target_weight": 0.0167,
        "deviation_pct": -7.2,
        "unrealized_pnl": 1.75,
        "leverage": 1,
        "strategy_source": "baseline_rev",
        "ensemble_contribution": 0.3
      }
    ],
    "summary": {
      "total_positions": 45,
      "long_count": 30,
      "short_count": 15,
      "total_deviation": 0.042,
      "missing_positions": 4,
      "extra_positions": 2
    },
    "mode": "ensemble",
    "active_strategies": ["baseline_rev", "rev_x_inv_vpin", "ofi_14d"],
    "signal_timestamp": "2026-03-02T23:55:00Z"
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

| 新增字段 | 类型 | 说明 |
|---------|------|------|
| `positions[].strategy_source` | string | 该仓位来自哪个策略 |
| `positions[].ensemble_contribution` | float | ensemble 模式下该策略的权重贡献 |
| `mode` | string | `"single"` \| `"ensemble"` |
| `active_strategies` | string[] | 当前激活的策略列表 |

V2 中 `strategy_name` 字段替换为 `mode` + `active_strategies`。V1 客户端兼容: `mode` 不存在时按 `strategy_name` 处理。

---

### 4.2 `/ws/portfolio` 增强 [V2]

V2 新增按策略拆分的指标：

```json
{
  "type": "portfolio_update",
  "data": {
    "nav": 10234.56,
    "total_return": 0.023456,
    "daily_pnl": -12.50,
    "daily_pnl_pct": -0.0012,
    "drawdown": -0.053,
    "max_drawdown": -0.082,
    "sharpe_30d": 1.85,
    "volatility_30d": 0.185,
    "total_equity": 10234.56,
    "unrealized_pnl": 123.45,
    "mode": "ensemble",
    "per_strategy_metrics": [
      {
        "strategy_name": "baseline_rev",
        "weight": 0.3,
        "pool_name": "default",
        "contribution_pnl": -4.50,
        "long_count": 30,
        "short_count": 30
      },
      {
        "strategy_name": "rev_x_inv_vpin",
        "weight": 0.3,
        "pool_name": "default",
        "contribution_pnl": 2.80,
        "long_count": 10,
        "short_count": 10
      },
      {
        "strategy_name": "ofi_14d",
        "weight": 0.4,
        "pool_name": "t50_monthly",
        "contribution_pnl": -10.80,
        "long_count": 15,
        "short_count": 15
      }
    ]
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

| 新增字段 | 类型 | 说明 |
|---------|------|------|
| `mode` | string | `"single"` \| `"ensemble"` |
| `per_strategy_metrics` | object[] | 各策略的贡献指标 (仅 ensemble 模式) |
| `per_strategy_metrics[].strategy_name` | string | 策略名 |
| `per_strategy_metrics[].weight` | float | ensemble 权重 |
| `per_strategy_metrics[].pool_name` | string | 使用的资产池 |
| `per_strategy_metrics[].contribution_pnl` | float | 该策略对当日 PnL 的贡献 |
| `per_strategy_metrics[].long_count` | int | 多头数量 |
| `per_strategy_metrics[].short_count` | int | 空头数量 |

---

### 4.3 `/ws/alerts` 增强 [V2]

V2 新增 `bar_source_mismatch` 告警类别：

```json
{
  "type": "alert",
  "data": {
    "alert_id": "alt-20260302-003",
    "severity": "warning",
    "category": "bar_source_mismatch",
    "rule_name": "bar_source_deviation",
    "display_name": "双源数据偏差",
    "message": "BTC/USDT:USDT 双源 close 偏差 4.8bps, 连续 3 次",
    "details": {
      "symbol": "BTC/USDT:USDT",
      "mismatch_type": "close_price",
      "deviation_bps": 4.8,
      "threshold_bps": 5,
      "consecutive_mismatches": 3,
      "auto_fallback_triggered": false
    },
    "timestamp": "2026-03-02T14:53:00Z"
  },
  "timestamp": "2026-03-02T14:53:00Z"
}
```

V2 告警 `category` 取值扩展为: `"risk_rejected"` | `"order_failed"` | `"bar_source_mismatch"`

---

### 4.4 `/ws/signals` — 策略信号推送 [V2 新增]

**推送频率**: 事件触发 (每次 signal_generated 后)

**推送内容**: 最新策略信号快照

**数据来源**: 订阅 Redis Pub/Sub `quant:signal_generated`

**后端实现路径**: `controller/routes/ws_signals.py`

#### 消息格式

```json
{
  "type": "signal_update",
  "data": {
    "mode": "ensemble",
    "signal_timestamp": "2026-03-02T23:59:00Z",
    "rebalance_id": "rb-20260302-001",
    "ensemble_positions": [
      {
        "symbol": "BTC/USDT:USDT",
        "side": "long",
        "weight": 0.0125,
        "rank": 1
      }
    ],
    "per_strategy_signals": [
      {
        "strategy_name": "baseline_rev",
        "pool_name": "default",
        "long_count": 30,
        "short_count": 30,
        "top_signals": [
          { "symbol": "BTC/USDT:USDT", "signal_value": 1.85, "side": "long", "rank": 1 },
          { "symbol": "DOGE/USDT:USDT", "signal_value": -2.10, "side": "short", "rank": 480 }
        ]
      },
      {
        "strategy_name": "ofi_14d",
        "pool_name": "t50_monthly",
        "long_count": 15,
        "short_count": 15,
        "top_signals": [
          { "symbol": "SOL/USDT:USDT", "signal_value": 2.45, "side": "long", "rank": 1 },
          { "symbol": "AVAX/USDT:USDT", "signal_value": -1.80, "side": "short", "rank": 50 }
        ]
      }
    ],
    "total_symbols_in_pool": 488
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `mode` | string | `"single"` \| `"ensemble"` |
| `signal_timestamp` | string (ISO 8601) | 信号计算时间 |
| `rebalance_id` | string | 换仓 ID |
| `ensemble_positions` | object[] | ensemble 加权后的最终目标仓位 (仅 ensemble 模式) |
| `per_strategy_signals` | object[] | 各策略独立信号摘要 |
| `per_strategy_signals[].strategy_name` | string | 策略名 |
| `per_strategy_signals[].pool_name` | string | 资产池 |
| `per_strategy_signals[].long_count` | int | 多头数量 |
| `per_strategy_signals[].short_count` | int | 空头数量 |
| `per_strategy_signals[].top_signals` | object[] | Top/Bottom 信号摘要 (各取前 5) |
| `total_symbols_in_pool` | int | 资产池总 symbol 数 |

---

### 4.5 V2 服务列表扩展 [V2]

V2 的 `/ws/services` 新增以下服务：

| 新增服务名 | display_name | 说明 |
|-----------|-------------|------|
| `data_ingestion_service` | 数据采集 | aggTrade WebSocket |
| `dollar_bar_service` | Dollar Bar 聚合 | 自适应阈值 |
| `tick_feature_service` | Tick 特征 | 9 个微观特征 |
| `bar_source_adapter` | Bar 适配层 | 双源归一化 |
| `monitor_service` | 监控告警 | 心跳检查 + 告警 |

---

## 五、版本兼容性

| 场景 | V1 客户端连接 V2 服务端 | V2 客户端连接 V1 服务端 |
|------|:---:|:---:|
| `/ws/positions` | 兼容。忽略新增字段 (`strategy_source`, `mode` 等) | 兼容。新增字段不存在时使用默认值 |
| `/ws/portfolio` | 兼容。忽略 `per_strategy_metrics` | 兼容。`per_strategy_metrics` 为 null |
| `/ws/alerts` | 兼容。新 category 按 `"warning"` 显示 | 兼容。不会收到 V2 专有告警 |
| `/ws/services` | 兼容。新增服务正常显示 | 兼容。缺少的服务不显示 |
| `/ws/signals` | 不适用 (V1 无此端点) | 连接失败，降级为轮询 HTTP |
