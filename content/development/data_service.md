# DataService

市场数据采集服务，订阅 Binance 全量交易对并写入 Redis Stream。支持行情（trades/tickers/ohlcv）与订单簿（orderbook）。

相关文件:

1./src/quant_trading/app/commands/data_service

2./src/quant_trading/services/data_service


## 初始化

```python
DataService(
    redis_client: RedisClient,
    config: DataConfig = DataConfig(),
)
```

| 参数 | 说明 |
|------|------|
| `redis_client` | 已连接的 Redis 客户端（必填） |
| `config` | `DataConfig` 实例，支持环境变量覆盖（`DATA_*` 前缀） |

## DataConfig

| 字段 | 类型 | 默认值 | 环境变量 | 说明 |
|------|------|--------|----------|------|
| `exchange_id` | `str` | `binanceusdm` | `DATA_EXCHANGE_ID` | ccxt.pro 交易所 ID |
| `market_type` | `str` | `future` | `DATA_MARKET_TYPE` | `future`（永续合约）或 `spot`（现货） |
| `watch_mode` | `WatchMode` | `trades` | `DATA_WATCH_MODE` | 订阅模式，见下表 |
| `timeframe` | `str` | `1m` | `DATA_TIMEFRAME` | K 线周期，仅 `ohlcv` 模式生效 |
| `orderbook_depth_limit` | `int` | `20` | `DATA_ORDERBOOK_DEPTH_LIMIT` | 订单簿档位数，仅 `orderbook` 模式生效 |
| `stream_maxlen` | `int` | `500000` | `DATA_STREAM_MAXLEN` | Redis Stream 最大条目数 |

### WatchMode

| 值 | Stream Key | 说明 |
|----|------------|------|
| `trades` | `market:trades` | aggTrade（同价同毫秒成交合并） |
| `trades_raw` | `market:trades` | 原始逐笔成交 |
| `tickers` | `market:tickers` | 聚合行情快照（最新价/24h 高低/成交量） |
| `ohlcv` | `market:ohlcv` | K 线，需配合 `timeframe` 使用 |
| `orderbook` | `market:orderbook` | 订单簿快照（WebSocket 增量同步），深度由 `orderbook_depth_limit` 控制 |

## Redis Stream 输出格式

### market:trades（trades / trades_raw）

| 字段 | 说明 |
|------|------|
| `symbol` | 交易对，如 `BTC/USDT:USDT` |
| `price` | 成交价格 |
| `amount` | 成交数量 |
| `side` | `buy` / `sell` |
| `trade_id` | 交易所侧成交 ID |
| `ts` | 成交时间戳（毫秒） |

### market:tickers（tickers）

| 字段 | 说明 |
|------|------|
| `symbol` | 交易对 |
| `last` | 最新成交价 |
| `bid` / `ask` | 买一 / 卖一价 |
| `high` / `low` | 24h 最高 / 最低价 |
| `volume` | 24h 成交量（base） |
| `ts` | 时间戳（毫秒） |

### market:ohlcv（ohlcv）

| 字段 | 说明 |
|------|------|
| `symbol` | 交易对 |
| `timeframe` | K 线周期，如 `1m` |
| `ts` | K 线开始时间戳（毫秒） |
| `open` / `high` / `low` / `close` | OHLC 价格 |
| `volume` | 成交量 |

### market:orderbook（orderbook）

| 字段 | 说明 |
|------|------|
| `symbol` | 交易对 |
| `bids` | JSON 字符串，形如 `[[price, amount], ...]`，按价格降序，至多 `orderbook_depth_limit` 档 |
| `asks` | JSON 字符串，形如 `[[price, amount], ...]`，按价格升序，至多 `orderbook_depth_limit` 档 |
| `ts` | 快照时间戳（毫秒） |

消费端需对 `bids` / `asks` 做 `json.loads`。

## 使用示例

```python
# 默认：aggTrade 模式，从环境变量读取配置
service = DataService(redis_client)

# 显式配置：OHLCV
service = DataService(
    redis_client,
    config=DataConfig(
        watch_mode=WatchMode.OHLCV,
        timeframe="5m",
    ),
)

# 显式配置：订单簿
service = DataService(
    redis_client,
    config=DataConfig(
        watch_mode=WatchMode.ORDERBOOK,
        orderbook_depth_limit=20,
    ),
)

await service.start()
```

或通过环境变量控制：

```bash
DATA_WATCH_MODE=ohlcv
DATA_TIMEFRAME=5m
DATA_EXCHANGE_ID=binance
DATA_MARKET_TYPE=spot
```

## CLI

```bash
# 默认（期货 + aggTrade）
main data-service run

# 指定 watch 模式与市场类型
main data-service run --watch-mode ohlcv --timeframe 5m
main data-service run --exchange binance --market-type spot --watch-mode tickers
main data-service run --watch-mode trades_raw

# 订单簿（20 档）
main data-service run --watch-mode orderbook --orderbook-limit 20

# 通过环境变量覆盖（DATA_* 前缀）
DATA_WATCH_MODE=ohlcv DATA_TIMEFRAME=5m main data-service run
```

### CLI 参数

| 参数 | 简写 | 默认值 | 说明 |
|------|------|--------|------|
| `--exchange` | `-e` | `binanceusdm` | ccxt.pro 交易所 ID |
| `--market-type` | `-m` | `future` | `future` 或 `spot` |
| `--watch-mode` | `-w` | `trades` | `trades` / `trades_raw` / `tickers` / `ohlcv` / `orderbook` |
| `--timeframe` | `-t` | `1m` | K 线周期（仅 `ohlcv` 模式） |
| `--orderbook-limit` | | `20` | 订单簿档位数（仅 `orderbook` 模式） |
| `--maxlen` | | `500000` | Redis Stream 最大条目数 |
| `--env` | `-E` | `dev` | 环境（dev / prod），影响 `.env` 加载 |

## 注意事项

- 需要 Redis 5.0+（XADD），版本不足时启动直接报错
- 每批最多 200 个 symbol（`WS_MAX_SYMBOLS_PER_BATCH`），超出时自动分批，每批独立 Task
- 期货模式（`market_type=future`）只订阅永续合约（`type=swap`），不混入交割合约（`type=future`），因 `watch_*_for_symbols` 要求同批 symbol 类型一致
- 每 3600 秒自动重新拉取 symbol 列表（`SYMBOL_RELOAD_INTERVAL_SECONDS`），感知新上线交易对
- 异常时指数退避重试（5s → 10s → ... 上限 60s），无最大重试次数限制
- `ohlcv` 模式下 `watch_ohlcv_for_symbols` 返回 `{symbol: {timeframe: [[ts,o,h,l,c,v], ...]}}` 嵌套 dict，服务内自动展开
- `orderbook` 模式下 `watch_order_book_for_symbols` 每次返回单个 symbol 的完整 order book dict；Stream 写入频率 ≈ WebSocket 事件频率，`orderbook_depth_limit` 决定每条记录体积，生产建议 ≤ 20
