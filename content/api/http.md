# HTTP API 参考

本页定义对外 RESTful HTTP 接口。服务间内部事件通信请参考 [Redis 事件总线](redis.md)。

版本说明：

| 版本 | 端点数 | 覆盖 Sprint | 策略支持 |
|------|--------|------------|---------|
| **V1 P0** | 6 | Sprint 1-2 | baseline_rev 单策略，最小闭环 |
| **V1 P1** | +14 | Sprint 3-4 | + 风控、订单审计、组合指标 |
| **V1 P2** | +6 | Sprint 5-6 | + 策略查询、回测 |
| **V2** | +6 | Sprint 7-8 | + 多策略管理、信号对比、资产池 |

---

## 一、通用约定

### Base URL

```
开发环境: http://localhost:8000
生产环境: https://api.quant-trading.example.com
```

### 请求/响应格式

- Content-Type: `application/json`
- 所有时间戳使用 **ISO 8601 UTC**: `"2026-03-02T23:59:00Z"`
- 数值精度：价格保留 8 位小数，权重/比例保留 6 位小数，金额保留 2 位小数

### 统一响应包装

成功响应：

```json
{
  "code": 0,
  "message": "success",
  "data": { },
  "timestamp": "2026-03-02T12:00:00Z"
}
```

失败响应：

```json
{
  "code": 40001,
  "message": "Strategy not found: invalid_name",
  "data": null,
  "timestamp": "2026-03-02T12:00:00Z"
}
```

### 错误码

| 范围 | 含义 |
|------|------|
| `0` | 成功 |
| `40000-40099` | 请求参数错误 |
| `40100-40199` | 认证/授权错误 |
| `40400-40499` | 资源不存在 |
| `50000-50099` | 服务内部错误 |
| `50300-50399` | 上游服务不可用 (Redis / 交易所) |

### 分页约定

```
GET /api/v1/orders/history?page=1&page_size=50&sort_by=timestamp&sort_order=desc
```

```json
{
  "data": {
    "items": [ ],
    "pagination": {
      "page": 1,
      "page_size": 50,
      "total_items": 1234,
      "total_pages": 25
    }
  }
}
```

### OpenAPI

- `GET /docs` — Swagger UI
- `GET /openapi.json` — OpenAPI Schema

---

## 二、V1 P0 端点 (Sprint 1-2)

最短闭环 + 最小可观测。

### 2.1 Health

#### `GET /api/v1/health`

全局健康状态总览。

**数据来源**: Redis `quant:heartbeat:{service}`

**响应**:

```json
{
  "data": {
    "overall_status": "healthy",
    "services": [
      {
        "name": "asset_pool_service",
        "status": "healthy",
        "last_heartbeat": "2026-03-02T23:58:30Z",
        "uptime_seconds": 86400,
        "metrics": {
          "symbols_count": 100
        }
      },
      {
        "name": "feature_service",
        "status": "degraded",
        "last_heartbeat": "2026-03-02T23:55:00Z",
        "uptime_seconds": 86400,
        "metrics": {
          "last_calculation": "2026-03-02T23:54:50Z",
          "symbols_calculated": 98
        }
      }
    ],
    "checked_at": "2026-03-02T23:59:00Z"
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `overall_status` | string | `"healthy"` \| `"degraded"` \| `"down"` |
| `services[].name` | string | 服务标识 |
| `services[].status` | string | `"healthy"` \| `"degraded"` \| `"down"` |
| `services[].last_heartbeat` | string (ISO 8601) | 最后心跳时间 |
| `services[].uptime_seconds` | int | 连续运行秒数 |
| `services[].metrics` | object | 服务特有指标 |

---

#### `GET /api/v1/health/pipeline`

数据管线各阶段状态。

**数据来源**: Redis `quant:state:{service}:last_run` + `quant:state:{service}:status`

**响应**:

```json
{
  "data": {
    "stages": [
      {
        "name": "data_ingestion",
        "display_name": "数据采集",
        "status": "running",
        "last_run": "2026-03-02T23:59:00Z",
        "next_run": null,
        "metrics": {
          "symbols_streaming": 100
        }
      },
      {
        "name": "feature_calculation",
        "display_name": "特征计算",
        "status": "idle",
        "last_run": "2026-03-02T23:54:50Z",
        "next_run": "2026-03-02T23:59:00Z",
        "metrics": {
          "features_per_symbol": 4
        }
      },
      {
        "name": "signal_generation",
        "display_name": "信号生成",
        "status": "idle",
        "last_run": "2026-03-02T23:55:00Z",
        "next_run": "2026-03-02T23:59:00Z",
        "metrics": {
          "active_strategy": "baseline_rev",
          "long_count": 30,
          "short_count": 30
        }
      },
      {
        "name": "order_execution",
        "display_name": "订单执行",
        "status": "idle",
        "last_run": "2026-03-02T23:55:10Z",
        "next_run": null,
        "metrics": {
          "last_orders_count": 12,
          "dry_run": true
        }
      }
    ]
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `stages[].name` | string | 阶段标识 |
| `stages[].display_name` | string | 显示名 |
| `stages[].status` | string | `"running"` \| `"idle"` \| `"error"` |
| `stages[].last_run` | string (ISO 8601) \| null | 上次执行时间 |
| `stages[].next_run` | string (ISO 8601) \| null | 下次预计执行时间 |
| `stages[].metrics` | object | 阶段特有指标 |

---

### 2.2 Account

#### `GET /api/v1/account/balance`

账户余额。

**数据来源**: Redis `quant:account:balance`

**响应**:

```json
{
  "data": {
    "total_equity": 10234.56,
    "available_balance": 5120.30,
    "used_margin": 5114.26,
    "unrealized_pnl": 123.45,
    "currency": "USDT",
    "updated_at": "2026-03-02T23:58:30Z"
  }
}
```

---

#### `GET /api/v1/account/positions`

当前持仓列表。

**数据来源**: Redis `quant:account:positions`

**响应**:

```json
{
  "data": {
    "positions": [
      {
        "symbol": "BTC/USDT:USDT",
        "side": "long",
        "contracts": 0.005,
        "entry_price": 62450.00,
        "mark_price": 62800.00,
        "unrealized_pnl": 1.75,
        "leverage": 1,
        "margin": 312.25,
        "liquidation_price": null,
        "updated_at": "2026-03-02T23:58:30Z"
      }
    ],
    "total_positions": 45,
    "long_count": 30,
    "short_count": 15,
    "updated_at": "2026-03-02T23:58:30Z"
  }
}
```

---

### 2.3 Portfolio (基础)

#### `GET /api/v1/portfolio/target-positions`

策略输出的目标仓位。

**数据来源**: Redis `quant:signal:latest`

**响应**:

```json
{
  "data": {
    "strategy_name": "baseline_rev",
    "signal_timestamp": "2026-03-02T23:55:00Z",
    "positions": [
      {
        "symbol": "BTC/USDT:USDT",
        "side": "long",
        "target_weight": 0.0167,
        "signal_value": 1.85,
        "reason": "baseline_rev_long"
      },
      {
        "symbol": "DOGE/USDT:USDT",
        "side": "short",
        "target_weight": -0.0167,
        "signal_value": -2.10,
        "reason": "baseline_rev_short"
      }
    ],
    "total_long": 30,
    "total_short": 30
  }
}
```

---

### 2.4 Signals (基础)

#### `GET /api/v1/signals/latest`

最新策略信号快照。

**数据来源**: Redis `quant:signal:latest`

**响应**:

```json
{
  "data": {
    "strategy_name": "baseline_rev",
    "signal_timestamp": "2026-03-02T23:55:00Z",
    "signals": [
      {
        "symbol": "BTC/USDT:USDT",
        "signal_value": 1.85,
        "rank": 1,
        "side": "long"
      },
      {
        "symbol": "DOGE/USDT:USDT",
        "signal_value": -2.10,
        "rank": 488,
        "side": "short"
      }
    ],
    "total_symbols_with_signal": 480,
    "total_symbols_in_pool": 488
  }
}
```

---

## 三、V1 P1 端点 (Sprint 3-4)

风控前置 + 订单审计 + 完整组合指标。

### 3.1 Risk

#### `GET /api/v1/risk/status`

当前风控状态概览。

**数据来源**: Redis `quant:risk:status`

**响应**:

```json
{
  "data": {
    "overall_status": "normal",
    "trading_enabled": true,
    "dry_run": true,
    "rules": [
      {
        "rule_name": "max_position_pct",
        "display_name": "单币种最大仓位占比",
        "threshold": 0.15,
        "current_value": 0.08,
        "status": "ok",
        "worst_symbol": "BTC/USDT:USDT"
      },
      {
        "rule_name": "max_total_exposure",
        "display_name": "最大总敞口",
        "threshold": 1.0,
        "current_value": 0.85,
        "status": "ok",
        "detail": "long=0.50, short=0.35"
      },
      {
        "rule_name": "max_drawdown_halt",
        "display_name": "最大回撤止损",
        "threshold": 0.30,
        "current_value": 0.053,
        "status": "ok"
      },
      {
        "rule_name": "min_balance_reserve",
        "display_name": "最低保留余额",
        "threshold": 100.0,
        "current_value": 5120.30,
        "status": "ok"
      },
      {
        "rule_name": "max_daily_turnover",
        "display_name": "每日最大换手率",
        "threshold": 3.0,
        "current_value": 0.45,
        "status": "ok"
      }
    ],
    "last_check": "2026-03-02T23:58:00Z"
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `overall_status` | string | `"normal"` \| `"warning"` \| `"halted"` |
| `trading_enabled` | bool | 交易是否开启 |
| `dry_run` | bool | 是否为模拟盘 |
| `rules[].rule_name` | string | 规则标识 |
| `rules[].display_name` | string | 规则显示名 |
| `rules[].threshold` | float | 阈值 |
| `rules[].current_value` | float | 当前值 |
| `rules[].status` | string | `"ok"` \| `"warning"` \| `"violated"` |

---

#### `GET /api/v1/risk/alerts`

告警历史 (分页)。

**请求参数**: `?page=1&page_size=20&severity=all&start_date=2026-03-01`

**数据来源**: TimescaleDB `alerts` 表

**响应**:

```json
{
  "data": {
    "items": [
      {
        "alert_id": "alt-20260302-001",
        "severity": "warning",
        "rule_name": "max_total_exposure",
        "message": "总敞口接近阈值: 0.92 / 1.0",
        "current_value": 0.92,
        "threshold": 1.0,
        "timestamp": "2026-03-02T20:15:00Z",
        "resolved": true,
        "resolved_at": "2026-03-02T20:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total_items": 47,
      "total_pages": 3
    }
  }
}
```

---

#### `GET /api/v1/risk/config`

当前风控配置 (只读)。

**数据来源**: `configs/risk_config.yaml`

**响应**:

```json
{
  "data": {
    "max_position_pct": 0.15,
    "max_total_exposure": 1.0,
    "max_drawdown_halt": 0.30,
    "min_balance_reserve": 100.0,
    "max_order_value": 5000.0,
    "max_daily_turnover": 3.0,
    "blacklist": [],
    "dry_run": true
  }
}
```

---

### 3.2 System

#### `POST /api/v1/system/emergency-stop`

全局紧急停机。触发后立即阻断所有新下单与换仓请求。

**请求体**:

```json
{
  "confirm": true,
  "reason": "交易所波动异常, 触发全局停机",
  "operator": "admin"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `confirm` | bool | 是 | 必须为 `true` |
| `reason` | string | 是 | 停机原因 |
| `operator` | string | 是 | 操作者标识 |

**状态写入**: Redis `quant:system:emergency_stop`, `quant:risk:status`

**响应**:

```json
{
  "data": {
    "emergency_stop": true,
    "trading_enabled": false,
    "scope": "all_symbols",
    "triggered_at": "2026-03-02T23:59:00Z",
    "triggered_by": "admin"
  }
}
```

---

#### `POST /api/v1/risk/symbols/disable`

停用单个交易对。被停用的交易对将拒绝新订单，但不自动平仓。

**请求体**:

```json
{
  "symbol": "BTC/USDT:USDT",
  "reason": "该交易对盘口异常, 暂停交易",
  "operator": "admin"
}
```

**响应**:

```json
{
  "data": {
    "symbol": "BTC/USDT:USDT",
    "trading_enabled": false,
    "disabled_at": "2026-03-02T23:59:00Z",
    "disabled_by": "admin"
  }
}
```

---

### 3.3 Orders

#### `GET /api/v1/orders/history`

历史订单 (分页)。

**请求参数**: `?page=1&page_size=50&symbol=&side=&start_date=2026-03-01`

**数据来源**: TimescaleDB `orders` 表

**响应**:

```json
{
  "data": {
    "items": [
      {
        "order_id": "ord-20260302-001",
        "symbol": "BTC/USDT:USDT",
        "side": "buy",
        "type": "market",
        "amount": 0.005,
        "filled_amount": 0.005,
        "price": null,
        "avg_fill_price": 62450.00,
        "cost": 312.25,
        "fee": 0.125,
        "status": "filled",
        "strategy_name": "baseline_rev",
        "rebalance_id": "rb-20260302-001",
        "dry_run": true,
        "created_at": "2026-03-02T23:55:05Z",
        "filled_at": "2026-03-02T23:55:06Z"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 50,
      "total_items": 234,
      "total_pages": 5
    }
  }
}
```

---

#### `GET /api/v1/orders/rebalance-history`

换仓历史 (每次完整换仓事件)。

**数据来源**: TimescaleDB `rebalance_events` 表

**响应**:

```json
{
  "data": {
    "rebalances": [
      {
        "rebalance_id": "rb-20260302-001",
        "strategy_name": "baseline_rev",
        "timestamp": "2026-03-02T23:55:00Z",
        "trigger": "cron",
        "orders_total": 24,
        "orders_filled": 22,
        "orders_failed": 2,
        "turnover": 0.45,
        "fee_total": 4.50,
        "long_symbols": 30,
        "short_symbols": 30,
        "execution_time_ms": 4500,
        "dry_run": true
      }
    ]
  }
}
```

---

#### `GET /api/v1/orders/trades`

成交明细 (分页)。

**请求参数**: `?page=1&page_size=100&symbol=&start_date=2026-03-01`

**数据来源**: TimescaleDB `trades` 表

**响应**:

```json
{
  "data": {
    "items": [
      {
        "trade_id": "t-123456",
        "order_id": "ord-20260302-001",
        "symbol": "BTC/USDT:USDT",
        "side": "buy",
        "amount": 0.005,
        "price": 62450.00,
        "cost": 312.25,
        "fee": 0.125,
        "fee_currency": "USDT",
        "timestamp": "2026-03-02T23:55:06Z"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 100,
      "total_items": 890,
      "total_pages": 9
    }
  }
}
```

---

#### `GET /api/v1/account/open-orders`

活跃挂单。

**数据来源**: Redis `quant:account:orders`

**响应**:

```json
{
  "data": {
    "orders": [
      {
        "order_id": "123456789",
        "symbol": "ETH/USDT:USDT",
        "side": "buy",
        "type": "limit",
        "amount": 0.5,
        "price": 3200.00,
        "filled": 0.0,
        "status": "open",
        "created_at": "2026-03-02T23:55:00Z"
      }
    ],
    "total_open_orders": 3
  }
}
```

---

### 3.4 Portfolio (完整)

#### `GET /api/v1/portfolio/nav`

NAV 净值曲线。

**请求参数**: `?start_date=2026-01-01&end_date=2026-03-02&interval=daily`

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `start_date` | string (YYYY-MM-DD) | 否 | 30 天前 | 起始日期 |
| `end_date` | string (YYYY-MM-DD) | 否 | 今天 | 结束日期 |
| `interval` | string | 否 | `"daily"` | `"daily"` \| `"hourly"` |

**数据来源**: TimescaleDB `portfolio_snapshots` 表

**响应**:

```json
{
  "data": {
    "initial_nav": 10000.0,
    "current_nav": 10234.56,
    "total_return": 0.023456,
    "datapoints": [
      { "timestamp": "2026-01-01T00:00:00Z", "nav": 10000.0 },
      { "timestamp": "2026-01-02T00:00:00Z", "nav": 10025.0 },
      { "timestamp": "2026-03-02T00:00:00Z", "nav": 10234.56 }
    ]
  }
}
```

---

#### `GET /api/v1/portfolio/pnl`

每日 PnL。

**请求参数**: `?start_date=2026-02-01&end_date=2026-03-02`

**数据来源**: TimescaleDB `portfolio_snapshots` 表

**响应**:

```json
{
  "data": {
    "daily_pnl": [
      { "date": "2026-03-01", "pnl": 45.30, "pnl_pct": 0.0044 },
      { "date": "2026-03-02", "pnl": -12.50, "pnl_pct": -0.0012 }
    ],
    "summary": {
      "total_pnl": 234.56,
      "avg_daily_pnl": 3.80,
      "best_day": { "date": "2026-02-15", "pnl": 89.20 },
      "worst_day": { "date": "2026-02-20", "pnl": -67.30 },
      "win_rate": 0.62
    }
  }
}
```

---

#### `GET /api/v1/portfolio/drawdown`

回撤曲线。

**请求参数**: `?start_date=2026-01-01`

**数据来源**: 基于 NAV 计算

**响应**:

```json
{
  "data": {
    "current_drawdown": -0.053,
    "max_drawdown": -0.082,
    "max_drawdown_date": "2026-02-20",
    "drawdown_curve": [
      { "timestamp": "2026-01-01T00:00:00Z", "drawdown": 0.0 },
      { "timestamp": "2026-02-20T00:00:00Z", "drawdown": -0.082 },
      { "timestamp": "2026-03-02T00:00:00Z", "drawdown": -0.053 }
    ],
    "recovery_days": null
  }
}
```

---

#### `GET /api/v1/portfolio/position-deviation`

目标仓位 vs 实际仓位偏差。

**数据来源**: Redis `quant:signal:latest` + `quant:account:positions`

**响应**:

```json
{
  "data": {
    "deviations": [
      {
        "symbol": "BTC/USDT:USDT",
        "target_weight": 0.0167,
        "actual_weight": 0.0155,
        "deviation": -0.0012,
        "deviation_pct": -7.2,
        "status": "acceptable"
      },
      {
        "symbol": "XRP/USDT:USDT",
        "target_weight": -0.0167,
        "actual_weight": 0.0,
        "deviation": 0.0167,
        "deviation_pct": 100.0,
        "status": "missing"
      }
    ],
    "summary": {
      "total_symbols": 60,
      "matched": 55,
      "missing": 3,
      "extra": 2,
      "avg_deviation_pct": 4.5,
      "max_deviation_pct": 100.0
    },
    "checked_at": "2026-03-02T23:59:00Z"
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `deviations[].status` | string | `"acceptable"` (偏差 <10%) \| `"warning"` (10%-50%) \| `"missing"` (目标有实际无) \| `"extra"` (实际有目标无) |

---

#### `GET /api/v1/portfolio/volatility`

波动率监控。

**请求参数**: `?level=portfolio&period=30d`

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `level` | string | 否 | `"portfolio"` | `"portfolio"` \| `"symbol"` |
| `period` | string | 否 | `"30d"` | `"7d"` \| `"30d"` \| `"90d"` |

**响应 (level=portfolio)**:

```json
{
  "data": {
    "level": "portfolio",
    "period": "30d",
    "metrics": {
      "annualized_volatility": 0.185,
      "daily_volatility": 0.0117,
      "sharpe_ratio": 1.85,
      "calmar_ratio": 2.47,
      "sortino_ratio": 2.31
    },
    "volatility_timeseries": [
      { "date": "2026-02-01", "rolling_vol_30d": 0.175 },
      { "date": "2026-03-02", "rolling_vol_30d": 0.185 }
    ]
  }
}
```

**响应 (level=symbol)**:

```json
{
  "data": {
    "level": "symbol",
    "period": "30d",
    "symbols": [
      {
        "symbol": "BTC/USDT:USDT",
        "annualized_volatility": 0.45,
        "contribution_to_portfolio_vol": 0.032,
        "weight": 0.0167
      }
    ]
  }
}
```

---

## 四、V1 P2 端点 (Sprint 5-6)

策略接口统一 + 回测兼容。

### 4.1 Strategy (基础查询)

#### `GET /api/v1/strategy/list`

已注册策略列表。

**数据来源**: StrategyRegistry + Redis `quant:strategy:status:{name}`

**响应 (V1 单策略模式)**:

```json
{
  "data": {
    "strategies": [
      {
        "name": "baseline_rev",
        "display_name": "Baseline Reversal (2h+4h)",
        "description": "2h+4h 反转基准策略, zscore_neg_ret_2h + zscore_neg_ret_4h",
        "status": "active",
        "parameters": {
          "long_n": 30,
          "short_n": 30,
          "rebalance_days": 1
        },
        "pool_name": "default",
        "required_features": ["zscore_neg_ret_2h", "zscore_neg_ret_4h"],
        "last_signal_time": "2026-03-02T23:55:00Z",
        "performance": {
          "sharpe_live": 1.85,
          "mdd_live": -0.082,
          "total_return": 0.156
        }
      }
    ],
    "active_mode": "single",
    "active_strategy": "baseline_rev"
  }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `strategies[].name` | string | 策略标识 |
| `strategies[].display_name` | string | 显示名 |
| `strategies[].description` | string | 策略描述 |
| `strategies[].status` | string | `"active"` \| `"inactive"` \| `"error"` |
| `strategies[].parameters` | object | 当前参数 |
| `strategies[].pool_name` | string | 使用的资产池 |
| `strategies[].required_features` | string[] | 所需特征列表 |
| `strategies[].last_signal_time` | string \| null | 最后信号时间 |
| `strategies[].performance` | object \| null | 运行期指标 |
| `active_mode` | string | V1: `"single"` |
| `active_strategy` | string | V1: `"baseline_rev"` |

---

#### `GET /api/v1/strategy/{name}/config`

策略详细配置。

**响应**:

```json
{
  "data": {
    "name": "baseline_rev",
    "parameters": {
      "long_n": 30,
      "short_n": 30,
      "rebalance_days": 1
    },
    "pool_name": "default",
    "cron_schedule": "59 23 * * *",
    "required_features": ["zscore_neg_ret_2h", "zscore_neg_ret_4h"],
    "compatible_bar_type": ["time_bar", "dollar_bar"],
    "config_version": "v1",
    "last_modified": "2026-03-01T10:00:00Z"
  }
}
```

---

### 4.2 Signals (扩展)

#### `GET /api/v1/signals/coverage`

信号覆盖率 / 缺失率。

**请求参数**: `?start_date=2026-02-01&end_date=2026-03-02`

**数据来源**: TimescaleDB `signal_snapshots` 表

**响应**:

```json
{
  "data": {
    "daily_coverage": [
      {
        "date": "2026-03-01",
        "pool_size": 488,
        "signals_generated": 480,
        "coverage_rate": 0.9836,
        "missing_symbols": ["NEWTOKEN/USDT:USDT"]
      }
    ],
    "summary": {
      "avg_coverage_rate": 0.9785,
      "min_coverage_rate": 0.9650,
      "min_coverage_date": "2026-02-15"
    }
  }
}
```

---

### 4.3 Backtest

#### `POST /api/v1/backtest/run`

提交回测任务。

**请求体**:

```json
{
  "strategy_name": "baseline_rev",
  "parameters": {
    "long_n": 30,
    "short_n": 30
  },
  "start_date": "2024-01-01",
  "end_date": "2025-01-01",
  "fee_bps": 10,
  "rebalance_days": 1,
  "data_source": "dollar_bar"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `strategy_name` | string | 是 | 策略标识 |
| `parameters` | object | 否 | 覆盖默认参数 |
| `start_date` | string (YYYY-MM-DD) | 是 | 回测开始日期 |
| `end_date` | string (YYYY-MM-DD) | 是 | 回测结束日期 |
| `fee_bps` | int | 否 | 手续费 (基点)，默认 10 |
| `rebalance_days` | int | 否 | 换仓周期，默认使用策略配置值 |
| `data_source` | string | 否 | `"dollar_bar"` \| `"kline"`，默认 `"dollar_bar"` |

**响应**:

```json
{
  "data": {
    "backtest_id": "bt-20260302-001",
    "status": "running",
    "submitted_at": "2026-03-02T12:00:00Z"
  }
}
```

---

#### `GET /api/v1/backtest/{backtest_id}/result`

查询回测结果。

**响应**:

```json
{
  "data": {
    "backtest_id": "bt-20260302-001",
    "status": "completed",
    "strategy_name": "baseline_rev",
    "parameters": {
      "long_n": 30,
      "short_n": 30,
      "start_date": "2024-01-01",
      "end_date": "2025-01-01",
      "fee_bps": 10
    },
    "metrics": {
      "total_return": 0.195,
      "annualized_return": 0.195,
      "sharpe": 1.95,
      "max_drawdown": -0.157,
      "calmar": 2.47,
      "avg_turnover": 0.42,
      "total_trades": 365,
      "win_rate": 0.55
    },
    "nav_curve": [
      { "date": "2024-01-01", "nav": 10000.0 },
      { "date": "2025-01-01", "nav": 11950.0 }
    ],
    "completed_at": "2026-03-02T12:05:00Z",
    "execution_time_seconds": 300
  }
}
```

---

#### `GET /api/v1/backtest/history`

历史回测任务列表。

**响应**:

```json
{
  "data": {
    "backtests": [
      {
        "backtest_id": "bt-20260302-001",
        "strategy_name": "baseline_rev",
        "status": "completed",
        "sharpe": 1.95,
        "max_drawdown": -0.157,
        "submitted_at": "2026-03-02T12:00:00Z"
      }
    ]
  }
}
```

---

## 五、V2 新增端点 [V2]

Sprint 7-8：多策略管理、信号对比、资产池查询。

### 5.1 Strategy 管理 [V2]

#### `POST /api/v1/strategy/{name}/toggle` [V2]

启停策略 (多策略模式下管理各策略状态)。

**请求体**:

```json
{
  "action": "activate",
  "confirm": true,
  "reason": "启用 ofi_14d 策略进入 ensemble"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `action` | string | 是 | `"activate"` \| `"deactivate"` |
| `confirm` | bool | 是 | 必须为 `true` |
| `reason` | string | 是 | 操作原因 |

**响应**:

```json
{
  "data": {
    "name": "ofi_14d",
    "previous_status": "inactive",
    "new_status": "active",
    "toggled_at": "2026-03-02T23:59:00Z"
  }
}
```

---

#### `POST /api/v1/strategy/{name}/config` [V2]

提交策略参数变更请求 (受控发布)。

**请求体**:

```json
{
  "parameters": {
    "long_n": 20,
    "short_n": 20
  },
  "reason": "缩小持仓数量, 从 30L30S 调整为 20L20S",
  "apply_mode": "next_rebalance"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `parameters` | object | 是 | 需要变更的参数 (仅传变更字段) |
| `reason` | string | 是 | 变更原因 |
| `apply_mode` | string | 否 | `"next_rebalance"` (默认) \| `"immediate"` |

**响应**:

```json
{
  "data": {
    "change_request_id": "cr-20260302-001",
    "status": "pending_review",
    "changes": {
      "long_n": { "old": 30, "new": 20 },
      "short_n": { "old": 30, "new": 20 }
    },
    "apply_mode": "next_rebalance",
    "submitted_at": "2026-03-02T23:59:00Z"
  }
}
```

---

#### `GET /api/v1/strategy/{name}/config/history` [V2]

策略参数变更历史。

**响应**:

```json
{
  "data": {
    "changes": [
      {
        "change_id": "cr-20260302-001",
        "version": "v3",
        "timestamp": "2026-03-02T10:00:00Z",
        "changes": {
          "long_n": { "old": 30, "new": 20 }
        },
        "reason": "缩小持仓数量",
        "status": "applied",
        "applied_at": "2026-03-02T23:59:00Z"
      }
    ]
  }
}
```

---

### 5.2 Signals 对比 [V2]

#### `GET /api/v1/signals/compare` [V2]

多策略信号与绩效对比。

**请求参数**: `?strategies=baseline_rev,rev_x_inv_vpin,ofi_14d&start_date=2026-01-01`

**数据来源**: TimescaleDB `strategy_performance` 表

**响应**:

```json
{
  "data": {
    "strategies": [
      {
        "name": "baseline_rev",
        "display_name": "Baseline Reversal (2h+4h)",
        "pool_name": "default",
        "rebalance_days": 1,
        "metrics": {
          "total_return": 0.0234,
          "annualized_return": 0.142,
          "sharpe": 1.85,
          "max_drawdown": -0.082,
          "calmar": 1.73,
          "daily_turnover": 0.45,
          "win_rate": 0.62
        },
        "nav_curve": [
          { "date": "2026-01-01", "nav": 10000.0 },
          { "date": "2026-03-02", "nav": 10234.0 }
        ]
      },
      {
        "name": "rev_x_inv_vpin",
        "display_name": "Reversal x Inverse VPIN",
        "pool_name": "default",
        "rebalance_days": 1,
        "metrics": {
          "total_return": 0.0456,
          "annualized_return": 0.278,
          "sharpe": 3.33,
          "max_drawdown": -0.211,
          "calmar": 1.32,
          "daily_turnover": 0.78,
          "win_rate": 0.58
        },
        "nav_curve": [
          { "date": "2026-01-01", "nav": 10000.0 },
          { "date": "2026-03-02", "nav": 10456.0 }
        ]
      },
      {
        "name": "ofi_14d",
        "display_name": "OFI 14d Momentum",
        "pool_name": "t50_monthly",
        "rebalance_days": 14,
        "metrics": {
          "total_return": 0.0380,
          "annualized_return": 0.231,
          "sharpe": 2.40,
          "max_drawdown": -0.121,
          "calmar": 1.91,
          "daily_turnover": 0.12,
          "win_rate": 0.55
        },
        "nav_curve": [
          { "date": "2026-01-01", "nav": 10000.0 },
          { "date": "2026-03-02", "nav": 10380.0 }
        ]
      }
    ]
  }
}
```

---

### 5.3 Asset Pool [V2]

#### `GET /api/v1/asset-pool/list` [V2]

资产池列表。

**响应**:

```json
{
  "data": {
    "pools": [
      {
        "pool_name": "default",
        "display_name": "默认池 (Top-100 日成交额)",
        "metric": "usdt_volume_30d",
        "top_k": 100,
        "update_interval": "24h",
        "current_size": 100,
        "last_updated": "2026-03-02T00:00:00Z",
        "used_by_strategies": ["baseline_rev", "rev_x_inv_vpin"]
      },
      {
        "pool_name": "t50_monthly",
        "display_name": "T50 月度流动性池",
        "metric": "dollar_volume_monthly",
        "top_k": 50,
        "update_interval": "monthly",
        "lag": 1,
        "min_history_days": 60,
        "current_size": 50,
        "last_updated": "2026-03-01T00:00:00Z",
        "used_by_strategies": ["ofi_14d"]
      }
    ]
  }
}
```

---

#### `GET /api/v1/asset-pool/{pool_name}/symbols` [V2]

资产池详情。

**响应**:

```json
{
  "data": {
    "pool_name": "t50_monthly",
    "symbols": [
      {
        "symbol": "BTC/USDT:USDT",
        "rank": 1,
        "metric_value": 125000000.00,
        "history_days": 730
      },
      {
        "symbol": "ETH/USDT:USDT",
        "rank": 2,
        "metric_value": 85000000.00,
        "history_days": 730
      }
    ],
    "total_symbols": 50,
    "metric": "dollar_volume_monthly",
    "reference_month": "2026-02",
    "last_updated": "2026-03-01T00:00:00Z"
  }
}
```

---

### 5.4 `GET /api/v1/strategy/list` V2 增强 [V2]

V2 多策略模式下，同一端点返回更多策略和 ensemble 信息：

```json
{
  "data": {
    "strategies": [
      {
        "name": "baseline_rev",
        "display_name": "Baseline Reversal (2h+4h)",
        "description": "2h+4h 反转基准策略",
        "status": "active",
        "parameters": { "long_n": 30, "short_n": 30, "rebalance_days": 1 },
        "pool_name": "default",
        "required_features": ["zscore_neg_ret_2h", "zscore_neg_ret_4h"],
        "ensemble_weight": 0.3,
        "last_signal_time": "2026-03-02T23:55:00Z",
        "performance": { "sharpe_live": 1.85, "mdd_live": -0.082 }
      },
      {
        "name": "rev_x_inv_vpin",
        "display_name": "Reversal x Inverse VPIN",
        "description": "反转 × 逆 VPIN 连续调制 (Top 1)",
        "status": "active",
        "parameters": { "long_n": 10, "short_n": 10, "rebalance_days": 1 },
        "pool_name": "default",
        "required_features": ["zscore_neg_ret_2h", "zscore_neg_ret_4h", "tick_vpin_24h"],
        "ensemble_weight": 0.3,
        "last_signal_time": "2026-03-02T23:55:00Z",
        "performance": { "sharpe_live": 3.10, "mdd_live": -0.195 }
      },
      {
        "name": "ofi_14d",
        "display_name": "OFI 14d Momentum",
        "description": "14日滚动OFI截面动量, T50月度池, R14换仓",
        "status": "active",
        "parameters": { "long_n": 15, "short_n": 15, "rebalance_days": 14, "min_candidates": 30 },
        "pool_name": "t50_monthly",
        "required_features": ["ofi_14d", "zscore_ofi_14d"],
        "ensemble_weight": 0.4,
        "last_signal_time": "2026-02-17T23:59:00Z",
        "performance": { "sharpe_live": 2.25, "mdd_live": -0.098 }
      }
    ],
    "active_mode": "ensemble",
    "ensemble_method": "weighted_average",
    "active_strategies": ["baseline_rev", "rev_x_inv_vpin", "ofi_14d"]
  }
}
```

**V2 新增字段**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `strategies[].ensemble_weight` | float \| null | ensemble 模式下的权重 (single 模式为 null) |
| `active_mode` | string | `"single"` \| `"ensemble"` |
| `ensemble_method` | string \| null | `"weighted_average"` (ensemble 模式) |
| `active_strategies` | string[] | 当前激活的所有策略名 |

---

### 5.5 `GET /api/v1/signals/latest` V2 增强 [V2]

V2 多策略模式下增加 `per_strategy_signals` 字段：

```json
{
  "data": {
    "mode": "ensemble",
    "signal_timestamp": "2026-03-02T23:55:00Z",
    "ensemble_signals": [
      {
        "symbol": "BTC/USDT:USDT",
        "signal_value": 1.52,
        "rank": 1,
        "side": "long"
      }
    ],
    "per_strategy_signals": [
      {
        "strategy_name": "baseline_rev",
        "pool_name": "default",
        "total_symbols_with_signal": 480,
        "total_symbols_in_pool": 488,
        "long_count": 30,
        "short_count": 30
      },
      {
        "strategy_name": "ofi_14d",
        "pool_name": "t50_monthly",
        "total_symbols_with_signal": 48,
        "total_symbols_in_pool": 50,
        "long_count": 15,
        "short_count": 15
      }
    ],
    "total_symbols_with_signal": 480,
    "total_symbols_in_pool": 488
  }
}
```

---

## 六、端点总览与版本标注

| 端点 | 方法 | 版本 | Sprint | 说明 |
|------|------|------|--------|------|
| `/api/v1/health` | GET | V1 P0 | 1-2 | 全局健康 |
| `/api/v1/health/pipeline` | GET | V1 P0 | 1-2 | 管线状态 |
| `/api/v1/account/balance` | GET | V1 P0 | 1-2 | 余额 |
| `/api/v1/account/positions` | GET | V1 P0 | 1-2 | 持仓 |
| `/api/v1/portfolio/target-positions` | GET | V1 P0 | 1-2 | 目标仓位 |
| `/api/v1/signals/latest` | GET | V1 P0 | 1-2 | 最新信号 |
| `/api/v1/risk/status` | GET | V1 P1 | 3-4 | 风控状态 |
| `/api/v1/risk/alerts` | GET | V1 P1 | 3-4 | 告警历史 |
| `/api/v1/risk/config` | GET | V1 P1 | 3-4 | 风控配置 |
| `/api/v1/system/emergency-stop` | POST | V1 P1 | 3-4 | 紧急停机 |
| `/api/v1/risk/symbols/disable` | POST | V1 P1 | 3-4 | 停用交易对 |
| `/api/v1/orders/history` | GET | V1 P1 | 3-4 | 订单历史 |
| `/api/v1/orders/rebalance-history` | GET | V1 P1 | 3-4 | 换仓历史 |
| `/api/v1/orders/trades` | GET | V1 P1 | 3-4 | 成交明细 |
| `/api/v1/account/open-orders` | GET | V1 P1 | 3-4 | 活跃挂单 |
| `/api/v1/portfolio/nav` | GET | V1 P1 | 3-4 | NAV 曲线 |
| `/api/v1/portfolio/pnl` | GET | V1 P1 | 3-4 | 每日 PnL |
| `/api/v1/portfolio/drawdown` | GET | V1 P1 | 3-4 | 回撤曲线 |
| `/api/v1/portfolio/position-deviation` | GET | V1 P1 | 3-4 | 仓位偏差 |
| `/api/v1/portfolio/volatility` | GET | V1 P1 | 3-4 | 波动率 |
| `/api/v1/strategy/list` | GET | V1 P2 | 5-6 | 策略列表 |
| `/api/v1/strategy/{name}/config` | GET | V1 P2 | 5-6 | 策略配置 |
| `/api/v1/signals/coverage` | GET | V1 P2 | 5-6 | 信号覆盖率 |
| `/api/v1/backtest/run` | POST | V1 P2 | 5-6 | 回测提交 |
| `/api/v1/backtest/{id}/result` | GET | V1 P2 | 5-6 | 回测结果 |
| `/api/v1/backtest/history` | GET | V1 P2 | 5-6 | 回测历史 |
| `/api/v1/strategy/{name}/toggle` | POST | **V2** | 7-8 | 策略启停 |
| `/api/v1/strategy/{name}/config` | POST | **V2** | 7-8 | 参数变更 |
| `/api/v1/strategy/{name}/config/history` | GET | **V2** | 7-8 | 变更历史 |
| `/api/v1/signals/compare` | GET | **V2** | 7-8 | 多策略对比 |
| `/api/v1/asset-pool/list` | GET | **V2** | 7-8 | 资产池列表 |
| `/api/v1/asset-pool/{pool_name}/symbols` | GET | **V2** | 7-8 | 资产池详情 |

---

## 七、后端服务映射

| 接口组 | 后端服务 | 数据源 | 路由文件 |
|--------|---------|--------|---------|
| `/api/v1/health/*` | MonitorService | `quant:heartbeat:*`, `quant:state:*` | `routes/health.py` |
| `/api/v1/account/*` | AccountService | `quant:account:*` | `routes/account.py` |
| `/api/v1/strategy/*` | StrategyService | `quant:strategy:status:*`, YAML | `routes/strategy.py` |
| `/api/v1/system/*` | RiskService | `quant:system:*`, `quant:risk:*` | `routes/system.py` |
| `/api/v1/risk/*` | RiskService | `quant:risk:*`, DB `alerts` | `routes/risk.py` |
| `/api/v1/portfolio/*` | AccountService + FeatureService | `quant:signal:latest`, DB `portfolio_snapshots` | `routes/portfolio.py` |
| `/api/v1/orders/*` | OrderService | DB `orders`, `trades`, `rebalance_events` | `routes/orders.py` |
| `/api/v1/signals/*` | StrategyService + FeatureService | `quant:signal:latest`, DB `signal_snapshots` | `routes/signals.py` |
| `/api/v1/backtest/*` | BacktestEngine | DB `backtests` | `routes/backtest.py` |
| `/api/v1/asset-pool/*` [V2] | AssetPoolService | `quant:asset_pool:*` | `routes/asset_pool.py` |

---

## 八、CORS 配置

开发环境使用宽松策略；生产环境应限制 `allow_origins` 到前端实际域名。

---

## 九、废弃端点

以下旧契约已废弃，不再作为主文档：

- `GET /health` → 使用 `GET /api/v1/health`
- `GET /v1/spot/top{top_k}/{days}days` → 使用 `GET /api/v1/asset-pool/{pool_name}/symbols`
- `WS /ws` (deprecated ticker_websocket) → 使用新 WebSocket 端点
