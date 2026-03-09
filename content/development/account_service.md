# AccountService

账户数据同步服务，订阅 Binance User Data Stream，将账户余额、持仓、挂单实时写入 Redis，并暴露 REST + WebSocket API。

相关文件：

1. `src/quant_trading/services/account_service/service.py`
2. `src/quant_trading/app/commands/account_service/__init__.py`
3. `src/quant_trading/controller/routes/account/`

---

## 架构概览

```
Binance User Data Stream (ccxt.pro)
    ├── watch_balance()    ─┐
    ├── watch_positions()  ─┼─→ _on_state_updated() → Redis SET + Pub quant:account_updated
    └── watch_orders()     ─┘

_periodic_refresh_loop()          每 poll_interval 秒 REST 全量同步（刷新 mark_price）
_rest_fallback_loop()             WS 连续故障时启动，恢复后自动退出

PostgreSQL (_db_persist_loop())   每 db_persist_interval 秒更新 accounts.updated_at
```

### 数据同步优先级

| 优先级 | 路径 | 触发时机 |
|--------|------|----------|
| 1 | **WS 主路径** — `watch_balance/positions/orders` | Binance 推送 `ACCOUNT_UPDATE` / `ORDER_TRADE_UPDATE` 事件时立即触发 |
| 2 | **周期性 REST 刷新** — `_periodic_refresh_loop` | 每 `poll_interval` 秒，强制刷新 mark_price 与 actual_weight |
| 3 | **REST 降级兜底** — `_rest_fallback_loop` | WS 连续失败 ≥ `ws_max_consecutive_errors` 次时自动切换 |

> WS 事件仅在实际成交时由 Binance 推送，周期性刷新确保 mark_price 和 actual_weight 始终为当前行情值。

---

## Redis 写入

### 数据 Key（GET/SET）

| Key | 内容 |
|-----|------|
| `quant:account:balance` | 余额快照（total_equity、available_balance、used_margin、unrealized_pnl） |
| `quant:account:positions` | 持仓列表（symbol、side、contracts、mark_price、actual_weight 等） |
| `quant:account:orders` | 活跃挂单列表 |
| `quant:portfolio:snapshot` | Portfolio 综合快照（NAV、收益率、回撤、日盈亏等） |
| `quant:portfolio:initial_nav` | 初始净值（首次同步时写入，后续只读） |
| `quant:portfolio:peak_nav` | 历史最高净值（用于回撤计算） |
| `quant:portfolio:daily_open_nav:{YYYYMMDD}` | 当日开盘净值，TTL 48h |
| `quant:heartbeat:account_service` | 服务心跳，TTL `heartbeat_ttl` 秒 |

### Pub/Sub

| 动作 | Channel | 说明 |
|------|---------|------|
| **Publish** | `quant:account_updated` | 每次账户状态变化后发布，携带余额摘要与持仓计数 |
| **Subscribe** | `quant:order_rebalanced` | 收到再平衡完成事件后立即触发一次 REST 全量同步 |

---

## REST API

所有端点前缀：`/api/v1/account`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/balance` | 当前余额（从 Redis 缓存读取） |
| GET | `/positions` | 当前持仓列表，含 `mark_price`、`actual_weight` |
| GET | `/open-orders` | 当前活跃挂单 |
| GET | `/health` | 服务健康状态（Redis 连通性 + 最新心跳数据） |
| GET | `/portfolio/snapshot` | Portfolio 综合快照（NAV、收益率、回撤等） |
| GET | `/portfolio/target-positions` | 策略输出的目标持仓（读 `quant:signal:latest`） |
| GET | `/portfolio/position-deviation` | 目标仓位与实际仓位的偏差分析 |
| GET | `/signals/latest` | 最新信号快照（读 `quant:signal:latest`） |

所有响应格式：

```json
{
  "code": 200,
  "message": "success",
  "data": { ... },
  "timestamp": "2026-03-09T06:22:50Z"
}
```

数据不可用时返回 `code: 50300`。

---

## WebSocket API

| 端点 | 说明 |
|------|------|
| `ws://<host>/api/v1/account/ws/positions` | 实时持仓推流 |
| `ws://<host>/api/v1/account/ws/portfolio` | 实时 Portfolio 快照推流 |

### 推送机制（三路并发）

每个连接内部运行三个并发 Task：

| Task | 触发条件 |
|------|----------|
| `push_to_client` | 连接建立立即推一次；之后监听 `quant:account_updated` 事件，每次触发时推送 |
| `interval_push_to_client` | 每 **3 秒**从 Redis 拉取最新快照并推送（即使无新事件） |
| `receive_from_client` | 持续接收客户端帧，感知断连；断连时取消其他两个 Task 并释放资源 |

任一 Task 结束（断连、异常）时，其余 Task 自动取消，`pubsub` 资源随即释放。

---

## 配置

### YAML 配置（`account.yaml`）

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `account_id` | `str` | `default` | 账户唯一标识，用于 DB 持久化 |
| `account_name` | `str` | `默认账户` | 账户显示名称 |
| `poll_interval` | `int` | `30` | REST 周期刷新间隔（秒），同时控制 REST 降级轮询频率 |
| `db_persist_interval` | `int` | `30` | PostgreSQL `accounts.updated_at` 刷新间隔（秒） |
| `ws_max_consecutive_errors` | `int` | `5` | WS 连续失败次数阈值，超过后切 REST 降级 |
| `ws_reconnect_delay` | `int` | `10` | WS 异常后重连等待时间（秒） |
| `heartbeat_ttl` | `int` | `120` | Redis 心跳 Key 的 TTL（秒） |

### 环境变量（`ACCOUNT_SERVICE_` 前缀）

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `ACCOUNT_SERVICE_ENV` | `dev` | 运行环境（dev / prod） |
| `ACCOUNT_SERVICE_HOST` | `0.0.0.0` | API 监听地址 |
| `ACCOUNT_SERVICE_PORT` | `8000` | API 监听端口 |
| `ACCOUNT_SERVICE_LOG_LEVEL` | `info` | uvicorn 日志级别 |

交易所凭据通过 `EXCHANGE_API_KEY` / `EXCHANGE_SECRET` 设置（`ExchangeEnvConfig`）。

---

## CLI

```bash
# 默认启动（读 .env.dev）
main account-service run

# 指定环境
main account-service run --env prod

# 覆盖端口与轮询间隔
main account-service run --port 8001 --poll 60

# 完整参数
main account-service run --env dev --poll 30 --host 0.0.0.0 --port 8000 --log-level info
```

### CLI 参数

| 参数 | 简写 | 默认值来源 | 说明 |
|------|------|-----------|------|
| `--env` | `-E` | `ACCOUNT_SERVICE_ENV` | 环境（dev / prod），决定加载哪个 `.env` 文件 |
| `--poll` | `-p` | `account.yaml poll_interval` | REST 周期刷新间隔（秒） |
| `--host` | `-H` | `ACCOUNT_SERVICE_HOST` | API 监听地址 |
| `--port` | `-P` | `ACCOUNT_SERVICE_PORT` | API 监听端口 |
| `--log-level` | | `ACCOUNT_SERVICE_LOG_LEVEL` | 日志级别 |

启动后控制台打印实际监听地址与 WS 连接地址，例如：

```
============================================================
  Account Service (API + Worker)
============================================================
  Exchange: binanceusdm
  Mode: DEMO (Testnet)
  Poll interval (s): 30
  API: http://0.0.0.0:8001
  Account API: http://0.0.0.0:8001/api/v1/account/balance
  Health: http://0.0.0.0:8001/api/v1/account/health
  WS positions: ws://0.0.0.0:8001/api/v1/account/ws/positions
  WS portfolio: ws://0.0.0.0:8001/api/v1/account/ws/portfolio
  Press Ctrl+C to stop
============================================================
```

---

## 注意事项

- **WS 事件触发频率**：Binance User Data Stream 的 `ACCOUNT_UPDATE` 仅在实际成交时推送；无交易活动时 `watch_positions` 启动后只返回一次缓存快照。周期性 REST 刷新（`poll_interval`）解决了此问题，确保 `mark_price` 和 `actual_weight` 按时更新。
- **DB 可选**：未配置 `DatabaseManager` 时，PostgreSQL 持久化任务直接跳过，不影响 Redis 数据流。
- **首次启动**：启动时先做一次完整 REST 同步写入 Redis，再启动 WS 监听，确保服务就绪后 API 立即可用。
- **Windows 事件循环**：在 Windows 上使用 `main db init` 命令时，`setup_event_loop()` 自动切换为 `SelectorEventLoop`，避免 psycopg 与 `ProactorEventLoop` 的兼容问题。
