# HTTP API

> 对外 RESTful 接口。服务内部事件见 [Redis 事件总线](redis.md)。
>
> **账户维度**：每个策略一条竖直栈、绑一个账户。**策略作用域**资源走 `/strategies/{name}/…`；**账户作用域**资源（组合/风控/订单/账户）走 `/accounts/{account}/…`。二者一一对应：`/strategies/{name}` 携带 `account_id` 并 link 到其账户资源。

## 一、通用约定

- Base URL：`http://localhost:8000`（开发）
- `Content-Type: application/json`；时间戳 ISO 8601 UTC；价格 8 位小数、权重 6 位、金额 2 位。
- 统一包装：`{ "code": 0, "message": "success", "data": {…}, "timestamp": "…" }`，失败 `code` 非 0、`data: null`。
- 错误码段：`0` 成功 / `40000-40099` 参数 / `40100-40199` 认证 / `40400-40499` 不存在 / `50000-50099` 内部 / `50300-50399` 上游不可用。
- 分页：`?page=1&page_size=50&sort_by=…&sort_order=desc` → `data.items[]` + `data.pagination{page,page_size,total_items,total_pages}`。
- OpenAPI：`GET /docs`（Swagger UI）/ `GET /openapi.json`。

## 二、Health

### `GET /api/v1/health`

全局健康总览（聚合各栈心跳）。数据源 `quant:heartbeat:{service}@{account}`。

```json
{ "data": {
  "overall_status": "healthy",
  "stacks": [
    { "account": "acctA", "strategy": "ml_dollar_bar", "status": "healthy",
      "services": [ { "name": "risk_service", "status": "healthy", "last_heartbeat": "…", "uptime_seconds": 86400 } ] },
    { "account": "acctB", "strategy": "monthly_majors_vs_alts", "status": "healthy", "services": [ … ] }
  ],
  "shared": [ { "name": "data_service", "status": "healthy", "last_heartbeat": "…" } ],
  "checked_at": "2026-06-13T00:10:00Z"
} }
```

`status`：`healthy` | `degraded` | `down`（心跳 >120s → down）。

### `GET /api/v1/health/pipeline?account=acctA`

某栈管线各阶段状态。数据源 `quant:{account}:state:{service}:*`。`stages[]` = `{ name, display_name, status(running|idle|error), last_run, next_run, metrics }`。

## 三、策略 API（分层）

L1 契约核心对所有策略一致；L2 状态视图按策略类型多态；L3 诊断按能力声明。

### 3.1 L1 契约核心

#### `GET /api/v1/strategies`

注册表全部策略及运行态（N 元素数组）。数据源 StrategyRegistry + Redis。

#### `GET /api/v1/strategies/{name}`

```json
{ "data": {
  "name": "ml_dollar_bar",
  "account_id": "acctA",
  "enabled": true,
  "execution_mode": "event_driven",
  "self_feeding": true,
  "capabilities": ["state", "evaluate"],
  "parameters": { "...": "yaml 原样，只读" },
  "runtime": { "last_signal_at": "…", "last_rebalance_id": "rb-ml_dollar_bar-20260613T0007", "signals_today": 184 },
  "links": { "portfolio": "/api/v1/accounts/acctA/portfolio", "orders": "/api/v1/accounts/acctA/orders?strategy=ml_dollar_bar" }
} }
```

| 字段 | 说明 |
|------|------|
| `account_id` | 绑定的（子）账户——决定去哪查 NAV/持仓/订单 |
| `execution_mode` | `scheduler`（策略 5）\| `event_driven`（策略 6） |
| `capabilities` | 该策略支持的可选端点（L2/L3） |
| `parameters` | 来自 yaml，**只读**（改参走 yaml + git + 重启） |

#### `POST /api/v1/strategies/{name}/enable` / `disable`

请求体 `{ confirm, reason, operator }`。**语义**：disable 写 `quant:{account}:strategy:enabled:{name}`，在该栈 Risk 闸门拒批——策略**照常评估、信号照常记录为影子信号**，只是不流向执行。与全局急停（所有栈 + Order 二次拦截）是两个层级。

#### `GET /api/v1/strategies/{name}/target-portfolio`

当前目标组合（= `quant:{account}:signal:latest` 投影）。

```json
{ "data": {
  "strategy_name": "ml_dollar_bar", "account_id": "acctA",
  "rebalance_id": "rb-ml_dollar_bar-20260613T0007", "signal_timestamp": "…", "signal_type": "target_weight",
  "legs": [ { "symbol": "BTC/USDT:USDT", "side": "long", "target_weight": 0.05 } ],
  "gross_weight": 0.85, "net_weight": 0.10,
  "metadata": { "...": "策略私有，原样透传" },
  "links": { "risk": "/api/v1/accounts/acctA/risk/decisions/rb-ml_dollar_bar-20260613T0007",
             "orders": "/api/v1/accounts/acctA/orders?rebalance_id=rb-ml_dollar_bar-20260613T0007" }
} }
```

信封字段 + `legs[]` 对所有策略字节级一致；`gross/net_weight` 是便利聚合；`metadata` 原样透传不重组。

#### `GET /api/v1/strategies/{name}/signals?from=&to=&page=` · `GET …/signals/{rebalance_id}`

历史信号批次（每条 = 完整批次，含被 Risk 拒绝的影子批次）。数据源 `signal_records` 表。streaming 策略**仅在目标有效变更时落库**（非每 bar）。

### 3.2 L2 状态视图

#### `GET /api/v1/strategies/{name}/state`

信封统一，body 按 `view.type` 多态（每策略族一份视图，归策略插件 `state_view()`）。

**`grid_period`**（策略 5）——回答「现在暴露在什么风险里」：

```json
{ "data": { "strategy_name": "monthly_majors_vs_alts", "as_of": "…", "view": {
  "type": "grid_period", "period_start": "2026-05-31", "next_rebalance": "2026-06-30", "day_in_period": 13,
  "leverage": 1.15, "realised_vol_at_entry": 0.174,
  "legs": [ { "symbol": "SOL/USDT:USDT", "side": "short", "entry_price": 152.3, "current_price": 141.2, "distance_to_stop_pct": 0.402 } ],
  "stopped": [ { "symbol": "LABUSDT", "date": "2026-06-01", "entry_price": 1.02, "exit_price": 1.34 } ]
} } }
```

`distance_to_stop_pct` = 空头腿距 +30% 硬止损线的剩余涨幅。

**`ml_composite`**（策略 6）——回答「模型/数据流是否健康」：

```json
{ "data": { "strategy_name": "ml_dollar_bar", "as_of": "…", "view": {
  "type": "ml_composite",
  "models": { "shifts": [5,10,15], "cutoff": "2026-04-01", "loaded_at": "…" },
  "bar_flow": [ { "symbol": "BTC/USDT:USDT", "last_dollar_bar": "2026-06-13T00:07:41Z", "expected_interval_s": 240, "actual_lag_s": 95 } ],
  "circuit_breaker": { "active": false, "cooldown_left": 0, "equity_vs_peak": -0.03 },
  "last_inference": { "symbol": "BTC/USDT:USDT", "p_pos": 0.62, "held": 1.0 }
} } }
```

### 3.3 L3 诊断

#### `POST /api/v1/strategies/{name}/evaluate` 〔按 capabilities〕

干跑一次完整评估，**不发布信号、不写状态**。响应 = 将发布的批次（target-portfolio 形状）+ `diagnostics`（复用该策略 view 类型 + 中间量）。策略 5：dvol 排名表 + 排除原因 + 杠杆输入输出，约 15s 同步返回，**仅手动触发**（每次打一轮 REST）。策略 6：单次推理。

## 四、账户作用域资源（`/accounts/{account}/…`）

### 4.1 Account

- `GET /api/v1/accounts/{account}/balance` — 数据源 `quant:{account}:account:balance`。
- `GET /api/v1/accounts/{account}/positions` — `quant:{account}:account:positions`，含 `positions[]` + `long/short_count`。
- `GET /api/v1/accounts/{account}/open-orders` — `quant:{account}:account:orders`。

### 4.2 Portfolio（每账户一条 NAV）

| 端点 | 说明 | 数据源 |
|------|------|--------|
| `GET …/portfolio` | NAV/PnL/回撤/Sharpe 快照 | `quant:{account}:portfolio:snapshot` |
| `GET …/portfolio/nav?start&end&interval` | NAV 曲线 | `portfolio_snapshots` 表 |
| `GET …/portfolio/pnl?start&end` | 每日 PnL + 汇总（胜率/最佳最差日） | `portfolio_snapshots` |
| `GET …/portfolio/drawdown?start` | 回撤曲线 + 当前/最大回撤 | NAV 计算 |
| `GET …/portfolio/position-deviation` | 目标 vs 实际仓位偏差 | `quant:{account}:signal:latest` + `…:account:positions` |
| `GET …/portfolio/volatility?period=30d&level=portfolio\|symbol` | 波动率/Sharpe/Calmar/Sortino | `portfolio_snapshots` |

`position-deviation` 的 `deviations[].status`：`acceptable`(<10%) \| `warning`(10-50%) \| `missing` \| `extra`。

> 多账户对比/合并 NAV 是各账户 portfolio 之上的视图运算，由前端或后续聚合端点按需做，不在契约层分叉。

### 4.3 Risk

- `GET /api/v1/accounts/{account}/risk/status` — 该栈风控状态：`trading_enabled` / `dry_run` / `rules[]`（`{rule_name, threshold, current_value, status}`）。
- `GET …/risk/config` — 只读当前风控配置（`default_equity_usdt` / `balance_max_age_seconds` / `min_notional_usdt` / `max_gross_exposure`；后续 `max_drawdown_halt` 等）。⚠️ vol-target 策略合法毛敞口 = `2 × leverage`，`max_gross_exposure` 须按此校准。
- `GET …/risk/alerts?page=&severity=&start_date=` — 告警历史。数据源 `alerts` 表。

### 4.4 Orders

- `GET …/orders?page=&symbol=&side=&rebalance_id=&strategy=&start_date=` — 订单历史。数据源 `orders` 表。
- `GET …/orders/rebalance-history` — 换仓事件（每次完整调仓汇总：`orders_total/filled/failed`、`turnover`、`fee_total`、`dry_run`）。数据源 `rebalance_records` 表。
- `GET …/orders/trades?page=&symbol=&start_date=` — 成交明细。
- `GET …/orders/{rebalance_id}` — 单批审计（before/after 权重、逐单成交、滑点/手续费）。

## 五、System（全局）

- `POST /api/v1/system/emergency-stop` — 全局急停。请求体 `{ confirm:true, reason, operator }`，可选 `account`（只停某栈）。写 `quant:system:emergency_stop` → 各栈 Risk 拒批 + Order 二次拦截（**只停新单，不自动平仓**）。
- `POST /api/v1/accounts/{account}/risk/symbols/disable` — 停用某交易对：拒绝新单、不自动平仓。

## 六、Backtest

- `POST /api/v1/backtest/run` — 提交回测。请求体 `{ strategy_name, parameters?, start_date, end_date, fee_bps?, data_source }`，`data_source`：`kline`（策略 5）\| `dollar_bar`（策略 6）。返回 `backtest_id`。
- `GET /api/v1/backtest/{backtest_id}/result` — 结果（`metrics{total_return, sharpe, max_drawdown, calmar, avg_turnover, …}` + `nav_curve[]`）。
- `GET /api/v1/backtest/history` — 历史任务。

回测与实盘共用同一份策略代码（同一 `ScheduledStrategy`/`StreamingStrategy` 实现）。

## 七、后端服务映射

| 接口组 | 服务 | 数据源 | 路由 |
|--------|------|--------|------|
| `/health/*` | Monitor + API 网关 | `quant:heartbeat:*@*`、`quant:{account}:state:*` | `routes/health.py` |
| `/strategies/*` | Strategy + 策略插件 `state_view()` | Registry、`quant:{account}:signal:latest`、`…:strategy:enabled:*`、`signal_records` | `routes/strategies.py` |
| `/accounts/{account}/balance\|positions\|portfolio/*` | Account | `quant:{account}:account:*`、`portfolio_snapshots` | `routes/account.py` |
| `/accounts/{account}/risk/*` | Risk | `quant:{account}:risk:*`、`alerts` | `routes/risk.py` |
| `/accounts/{account}/orders/*` | Order | `orders`、`trades`、`rebalance_records` | `routes/orders.py` |
| `/system/*` | API 网关 | `quant:system:*` | `routes/system.py` |
| `/backtest/*` | Backtest Engine | `backtests` | `routes/backtest.py` |

## 八、刻意不做

| 不做 | 理由 |
|------|------|
| `POST .../config`（API 改参数） | 参数是研究产物，走 yaml + git + 重启：有 diff、有审计、可回滚 |
| ensemble 预埋字段 | N 策略时列表多元素、各自 view 天然成立；届时只加顶层聚合视图，不预埋空壳 |
| 信号 WebSocket | scheduler 事件密度极低（轮询足够）；streaming 策略状态用 `GET …/state` 轮询 |
| 跨账户合并 NAV 端点 | 各账户独立 NAV 已足；合并是前端视图运算，需要时再做 |
