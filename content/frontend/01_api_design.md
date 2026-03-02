# 前端监控仪表盘 — API 接口设计

> 编写时间: 2026-03-02
> 对接后端: `docs/architecture.md` 第 2.2 节各服务 + 第 2.3 节 Redis 键
> 前置阅读: `docs/frontend/00_overview.md`

---

## 目录

- [一、通用约定](#general-conventions)
- [二、REST API](#rest-api)
  - [2.1 Health (系统状态)](#health-api)
  - [2.2 Account (账户)](#account-api)
  - [2.3 Strategy (策略)](#strategy-api)
  - [2.4 Risk (风控)](#risk-api)
  - [2.5 Portfolio (组合收益)](#portfolio-api)
  - [2.6 Orders (订单)](#orders-api)
  - [2.7 Signals (信号)](#signals-api)
  - [2.8 Backtest (回测)](#backtest-api)
- [三、WebSocket API](#websocket-api)
- [四、接口与后端服务映射表](#service-mapping)

---

## 一、通用约定 {#general-conventions}

### Base URL

```
开发环境: http://localhost:8000
生产环境: https://api.quant-trading.example.com
```

### 请求/响应格式

- Content-Type: `application/json`
- 所有时间戳使用 **ISO 8601 UTC** 格式: `"2026-03-02T23:59:00Z"`
- 数值精度: 价格保留 8 位小数, 权重/比例保留 6 位小数, 金额保留 2 位小数

### 统一响应包装

```json
{
  "code": 0,
  "message": "success",
  "data": { ... },
  "timestamp": "2026-03-02T12:00:00Z"
}
```

错误响应:

```json
{
  "code": 40001,
  "message": "Strategy not found: invalid_name",
  "data": null,
  "timestamp": "2026-03-02T12:00:00Z"
}
```

### 错误码定义

| 范围 | 含义 |
|------|------|
| 0 | 成功 |
| 40000-40099 | 请求参数错误 |
| 40100-40199 | 认证/授权错误 |
| 40400-40499 | 资源不存在 |
| 50000-50099 | 服务内部错误 |
| 50300-50399 | 上游服务不可用 (Redis / 交易所) |

### 分页约定

```
GET /api/v1/orders/history?page=1&page_size=50&sort_by=timestamp&sort_order=desc
```

响应中包含分页信息:

```json
{
  "data": {
    "items": [...],
    "pagination": {
      "page": 1,
      "page_size": 50,
      "total_items": 1234,
      "total_pages": 25
    }
  }
}
```

---

## 二、REST API {#rest-api}

### 2.1 Health (系统状态) {#health-api}

#### GET /api/v1/health

全局健康状态总览。

**数据来源**: Redis `quant:heartbeat:{service}` (TTL 键)

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
        "name": "aggregator_service",
        "status": "healthy",
        "last_heartbeat": "2026-03-02T23:58:28Z",
        "uptime_seconds": 86400,
        "metrics": {
          "ticks_processed": 1520340,
          "klines_stored": 4200
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

#### GET /api/v1/health/pipeline

数据管线任务状态 (从数据采集到信号生成的完整链路)。

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
          "ticks_per_second": 245,
          "symbols_streaming": 100
        }
      },
      {
        "name": "dollar_bar_aggregation",
        "display_name": "Dollar Bar 聚合",
        "status": "running",
        "last_run": "2026-03-02T23:58:45Z",
        "next_run": null,
        "metrics": {
          "bars_generated_today": 2100
        }
      },
      {
        "name": "feature_calculation",
        "display_name": "特征计算",
        "status": "idle",
        "last_run": "2026-03-02T23:54:50Z",
        "next_run": "2026-03-02T23:59:00Z",
        "metrics": {
          "features_per_symbol": 32
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

---

### 2.2 Account (账户) {#account-api}

#### GET /api/v1/account/balance

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

#### GET /api/v1/account/positions

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

#### GET /api/v1/account/open-orders

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

### 2.3 Strategy (策略) {#strategy-api}

#### GET /api/v1/strategy/list

已注册策略列表。

**数据来源**: StrategyRegistry (代码内注册表) + Redis `quant:strategy:status:{name}`

**响应**:

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
        "last_signal_time": "2026-03-02T23:55:00Z",
        "performance": {
          "sharpe_live": 1.85,
          "mdd_live": -0.082,
          "total_return": 0.156
        }
      },
      {
        "name": "rev_x_inv_vpin",
        "display_name": "Reversal x Inverse VPIN",
        "description": "zscore_neg_ret_2h * (1 / zscore_vpin), Top 1 策略",
        "status": "inactive",
        "parameters": {
          "long_n": 10,
          "short_n": 10,
          "rebalance_days": 1
        },
        "last_signal_time": null,
        "performance": null
      }
    ],
    "active_mode": "single",
    "active_strategy": "baseline_rev"
  }
}
```

#### GET /api/v1/strategy/{name}/config

获取策略详细配置。

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
    "cron_schedule": "59 23 * * *",
    "required_features": ["zscore_neg_ret_2h", "zscore_neg_ret_4h"],
    "compatible_bar_type": ["time_bar", "dollar_bar"],
    "config_version": "v3",
    "last_modified": "2026-03-01T10:00:00Z",
    "modified_by": "admin"
  }
}
```

#### POST /api/v1/strategy/{name}/toggle

启停策略 (需确认)。

**请求体**:

```json
{
  "action": "activate",
  "confirm": true,
  "reason": "启用 baseline_rev 进行 paper trading"
}
```

**响应**:

```json
{
  "data": {
    "name": "baseline_rev",
    "previous_status": "inactive",
    "new_status": "active",
    "toggled_at": "2026-03-02T23:59:00Z"
  }
}
```

#### POST /api/v1/strategy/{name}/config

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

#### GET /api/v1/strategy/{name}/config/history

策略参数变更历史 (版本管理)。

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
      },
      {
        "change_id": "cr-20260228-001",
        "version": "v2",
        "timestamp": "2026-02-28T10:00:00Z",
        "changes": {
          "rebalance_days": { "old": 7, "new": 1 }
        },
        "reason": "从周度换仓改为日度换仓",
        "status": "applied",
        "applied_at": "2026-02-28T23:59:00Z"
      }
    ]
  }
}
```

---

### 2.4 Risk (风控) {#risk-api}

#### GET /api/v1/risk/status

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

#### POST /api/v1/system/emergency-stop

全局紧急停机。触发后立即阻断所有新下单与换仓请求。

**请求体**:

```json
{
  "confirm": true,
  "reason": "交易所波动异常, 触发全局停机",
  "operator": "admin"
}
```

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

#### POST /api/v1/risk/symbols/disable

关闭单个交易对交易。被停用的交易对将拒绝新订单, 但不自动平仓。

**请求体**:

```json
{
  "symbol": "BTC/USDT:USDT",
  "reason": "该交易对盘口异常, 暂停交易",
  "operator": "admin"
}
```

**状态写入**: Redis `quant:risk:disabled_symbols`

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

#### GET /api/v1/risk/alerts

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
      },
      {
        "alert_id": "alt-20260301-003",
        "severity": "critical",
        "rule_name": "order_execution_failed",
        "message": "下单失败: ETH/USDT:USDT, InsufficientBalance",
        "timestamp": "2026-03-01T23:55:10Z",
        "resolved": false,
        "resolved_at": null
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

#### GET /api/v1/risk/config

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

### 2.5 Portfolio (组合收益) {#portfolio-api}

#### GET /api/v1/portfolio/nav

NAV 净值曲线。

**请求参数**: `?start_date=2026-01-01&end_date=2026-03-02&interval=daily`

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

#### GET /api/v1/portfolio/pnl

每日 PnL。

**请求参数**: `?start_date=2026-02-01&end_date=2026-03-02`

**数据来源**: TimescaleDB `portfolio_snapshots` 表 (计算相邻日 nav 差值)

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

#### GET /api/v1/portfolio/drawdown

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

#### GET /api/v1/portfolio/target-positions

策略输出的目标仓位 (用于与实际仓位对比)。

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

#### GET /api/v1/portfolio/position-deviation

目标仓位 vs 实际仓位偏差。

**数据来源**: 组合 Redis `quant:signal:latest` + `quant:account:positions`

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

#### GET /api/v1/portfolio/volatility

波动率监控 (多层级)。

**请求参数**: `?level=portfolio|strategy|symbol&period=30d`

**数据来源**: TimescaleDB (基于日收益率计算)

**响应** (level=portfolio):

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

**响应** (level=symbol):

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

### 2.6 Orders (订单) {#orders-api}

#### GET /api/v1/orders/history

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

#### GET /api/v1/orders/rebalance-history

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
        "execution_time_ms": 4500
      }
    ]
  }
}
```

#### GET /api/v1/orders/trades

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

### 2.7 Signals (信号) {#signals-api}

#### GET /api/v1/signals/latest

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

#### GET /api/v1/signals/coverage

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
        "missing_symbols": ["NEWTOKEN/USDT:USDT", "DELIST/USDT:USDT"]
      },
      {
        "date": "2026-03-02",
        "pool_size": 488,
        "signals_generated": 475,
        "coverage_rate": 0.9734,
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

#### GET /api/v1/signals/compare

策略对比 (用于策略对比视图)。

**请求参数**: `?strategies=baseline_rev,rev_x_inv_vpin&start_date=2026-01-01`

**数据来源**: TimescaleDB `strategy_performance` 表

**响应**:

```json
{
  "data": {
    "strategies": [
      {
        "name": "baseline_rev",
        "display_name": "Baseline Reversal (2h+4h)",
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
      }
    ]
  }
}
```

---

### 2.8 Backtest (回测) {#backtest-api}

#### POST /api/v1/backtest/run

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

#### GET /api/v1/backtest/{backtest_id}/result

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
    "is_oos_split": "2024-01-01",
    "completed_at": "2026-03-02T12:05:00Z",
    "execution_time_seconds": 300
  }
}
```

#### GET /api/v1/backtest/history

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

## 三、WebSocket API {#websocket-api}

### 3.1 通用协议

所有 WebSocket 端点使用 JSON 消息:

```json
{
  "type": "positions_update",
  "data": { ... },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

客户端心跳:

```json
// 客户端每 30s 发送
{ "type": "ping" }

// 服务端响应
{ "type": "pong", "timestamp": "2026-03-02T23:59:00Z" }
```

### 3.2 /ws/positions

**推送频率**: 每 30 秒

**推送内容**: 当前仓位快照 + 目标仓位偏差

```json
{
  "type": "positions_update",
  "data": {
    "positions": [
      {
        "symbol": "BTC/USDT:USDT",
        "side": "long",
        "actual_weight": 0.0155,
        "target_weight": 0.0167,
        "deviation_pct": -7.2,
        "unrealized_pnl": 1.75
      }
    ],
    "summary": {
      "total_positions": 45,
      "long_count": 30,
      "short_count": 15,
      "total_deviation": 0.045
    }
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

**数据来源**: Redis `quant:account:positions` + `quant:signal:latest`

**后端实现路径**: `controller/routes/ws_positions.py`

### 3.3 /ws/portfolio

**推送频率**: 每 60 秒

**推送内容**: NAV / PnL / 回撤 / 波动率核心指标

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
    "volatility_30d": 0.185
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

**数据来源**: Redis `quant:portfolio:snapshot` (AccountService 定时写入)

**后端实现路径**: `controller/routes/ws_portfolio.py`

### 3.4 /ws/alerts

**推送频率**: 实时 (事件触发)

**推送内容**: 风控告警 / 系统告警

```json
{
  "type": "alert",
  "data": {
    "alert_id": "alt-20260302-002",
    "severity": "critical",
    "rule_name": "order_execution_failed",
    "message": "下单失败: ETH/USDT:USDT, InsufficientBalance",
    "timestamp": "2026-03-02T23:55:10Z"
  },
  "timestamp": "2026-03-02T23:55:10Z"
}
```

**数据来源**: 订阅 Redis Pub/Sub `quant:risk_rejected`, `quant:order_failed` 通道

**后端实现路径**: `controller/routes/ws_alerts.py`

### 3.5 /ws/services

**推送频率**: 每 30 秒

**推送内容**: 所有服务心跳状态

```json
{
  "type": "services_update",
  "data": {
    "services": [
      {
        "name": "asset_pool_service",
        "status": "healthy",
        "last_heartbeat": "2026-03-02T23:58:30Z",
        "latency_ms": 2
      },
      {
        "name": "feature_service",
        "status": "degraded",
        "last_heartbeat": "2026-03-02T23:55:00Z",
        "latency_ms": 240000
      }
    ],
    "pipeline_status": "healthy"
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

**数据来源**: Redis `quant:heartbeat:{service}` (扫描所有 heartbeat 键)

**后端实现路径**: `controller/routes/ws_services.py`

---

## 四、接口与后端服务映射表 {#service-mapping}

| 接口组 | 后端服务 | Redis 键 / DB 表 | 后端路由文件 |
|--------|---------|------------------|-------------|
| `/api/v1/health/*` | MonitorService | `quant:heartbeat:*`, `quant:state:*` | `routes/health.py` |
| `/api/v1/account/*` | AccountService | `quant:account:balance`, `quant:account:positions`, `quant:account:orders` | `routes/account.py` |
| `/api/v1/strategy/*` | StrategyService | `quant:strategy:status:*`, `configs/strategy_config.yaml` | `routes/strategy.py` |
| `/api/v1/system/*` | RiskService | `quant:system:emergency_stop`, `quant:risk:status` | `routes/system.py` |
| `/api/v1/risk/*` | RiskService | `quant:risk:status`, `quant:risk:disabled_symbols`, DB `alerts` | `routes/risk.py` |
| `/api/v1/portfolio/*` | AccountService + FeatureService | `quant:signal:latest`, DB `portfolio_snapshots` | `routes/portfolio.py` |
| `/api/v1/orders/*` | OrderService | DB `orders`, `trades`, `rebalance_events` | `routes/orders.py` |
| `/api/v1/signals/*` | StrategyService + FeatureService | `quant:signal:latest`, DB `signal_snapshots` | `routes/signals.py` |
| `/api/v1/backtest/*` | BacktestEngine | DB `backtests` | `routes/backtest.py` |
| `/ws/positions` | AccountService | `quant:account:positions` + `quant:signal:latest` | `routes/ws_positions.py` |
| `/ws/portfolio` | AccountService | `quant:portfolio:snapshot` | `routes/ws_portfolio.py` |
| `/ws/alerts` | RiskService | Pub/Sub `quant:risk_rejected`, `quant:order_failed` | `routes/ws_alerts.py` |
| `/ws/services` | MonitorService | `quant:heartbeat:*` | `routes/ws_services.py` |

### 后端新增数据库表

| 表名 | 用途 | 关键字段 |
|------|------|---------|
| `portfolio_snapshots` | NAV / PnL 历史 | time, nav, daily_pnl, drawdown, total_equity |
| `orders` | 订单记录 | order_id, symbol, side, amount, price, status, strategy_name, time |
| `trades` | 成交明细 | trade_id, order_id, symbol, price, amount, fee, time |
| `rebalance_events` | 换仓事件 | rebalance_id, strategy_name, trigger, orders_count, turnover, time |
| `alerts` | 告警历史 | alert_id, severity, rule_name, message, resolved, time |
| `signal_snapshots` | 每日信号快照 | date, strategy_name, pool_size, signals_count, missing_symbols |
| `strategy_config_changes` | 参数变更记录 | change_id, strategy_name, old_params, new_params, reason, time |
| `backtests` | 回测记录 | backtest_id, strategy_name, params, status, metrics, nav_curve |

所有表使用 TimescaleDB hypertable (按 time 分区), 保留策略: 90 天热数据, 365 天压缩数据。
