# DataService

市场数据采集服务（原 TickerService 重构而来），订阅 Binance 全量交易对并写入 Redis Stream。支持行情（trades/tickers/ohlcv）与订单簿（orderbook）。

相关文件:

1./src/quant_trading/app/commands/data_service

2./src/quant_trading/services/data_service

3./yamls/data.yaml

## 职责

- 通过 ccxt.pro WebSocket 订阅交易所行情，按 `watch_mode` 选择订阅类型（trades / trades_raw / tickers / ohlcv / orderbook）
- 将行情写入对应 Redis Stream：`market:trades` / `market:tickers` / `market:ohlcv` / `market:orderbook`
- `orderbook` 模式额外把最新快照覆盖写入 String 键 `quant:orderbook:{symbol}`，供 ExecutionService 做滑点 / 深度 / 新鲜度检查
- 全量 symbol 自动分批订阅（每批 ≤200）、断线指数退避自动重连、定期刷新交易对列表感知上下架

## 初始化

```python
DataService(
    redis_client: RedisClient,
    config: DataConfig = yaml_loader.get(DataConfig),
)
```

| 参数 | 说明 |
|------|------|
| `redis_client` | 已连接的 Redis 客户端（必填） |
| `config` | `DataConfig` 实例，默认经 `yaml_loader` 从 `yamls/data.yaml` 加载 |

## DataConfig（yamls/data.yaml）

配置不再走环境变量前缀，统一从 YAML 加载。优先级：CLI 参数 > `yamls/data.yaml` > `DataConfig` 字段默认值（yaml 文件缺失或字段缺省时回退默认值）。

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `exchange_id` | `str` | `binanceusdm` | ccxt.pro 交易所 ID |
| `market_type` | `str` | `future` | `future`（永续合约）或 `spot`（现货） |
| `watch_mode` | `WatchMode` | `trades` | 订阅模式，见下表 |
| `timeframe` | `str` | `1m` | K 线周期，仅 `ohlcv` 模式生效 |
| `orderbook_depth_limit` | `int` | `20` | 订单簿档位数，仅 `orderbook` 模式生效 |
| `stream_maxlen` | `int` | `500000` | Redis Stream 最大条目数（approximate trim） |
| `WS_RECONNECT_DELAY_SECONDS` | `int` | `5` | 重连基础延迟（指数退避基数） |
| `WS_MAX_SYMBOLS_PER_BATCH` | `int` | `200` | 每批最大订阅 symbol 数（binanceusdm 上限 200） |
| `SYMBOL_RELOAD_INTERVAL_SECONDS` | `int` | `3600` | 交易对列表自动刷新间隔（秒） |

### WatchMode

| 值 | Stream Key | 说明 |
|----|------------|------|
| `trades` | `market:trades` | aggTrade（同价同毫秒成交合并） |
| `trades_raw` | `market:trades` | 原始逐笔成交（individual trade） |
| `tickers` | `market:tickers` | 聚合行情快照（最新价/24h 高低/成交量） |
| `ohlcv` | `market:ohlcv` | K 线，需配合 `timeframe` 使用 |
| `orderbook` | `market:orderbook` | L2 订单簿快照（WebSocket 增量同步），档位数由 `orderbook_depth_limit` 控制 |

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

`orderbook` 模式在写 Stream 的同时，把最新快照覆盖写入 String 键 `quant:orderbook:{symbol}`（`set_json`，载荷含 `symbol / bids / asks / ts_ms / mid_price`，保留原生数值类型）。Stream 负责历史归档，String 负责"最新快照"读取路径，ExecutionService 据此做滑点 / 深度 / 新鲜度检查。

## 使用示例

```python
# 默认：从 yamls/data.yaml 加载（aggTrade 模式）
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

或直接修改 `yamls/data.yaml`：

```yaml
exchange_id: binanceusdm
market_type: future
watch_mode: trades
timeframe: 1m
orderbook_depth_limit: 20
stream_maxlen: 500000
```

## CLI

```bash
# 默认（期货 + aggTrade，配置取 yamls/data.yaml）
main data-service run

# 指定 watch 模式与市场类型
main data-service run --watch-mode ohlcv --timeframe 5m
main data-service run --exchange binance --market-type spot --watch-mode tickers
main data-service run --watch-mode trades_raw

# 订单簿模式（指定档位数）
main data-service run --watch-mode orderbook --orderbook-limit 20
```

### CLI 参数

所有参数默认值为「未指定」，未传时取 `yamls/data.yaml` 中的值。

| 参数 | 简写 | 说明 |
|------|------|------|
| `--exchange` | `-e` | ccxt.pro 交易所 ID（yaml 默认 `binanceusdm`） |
| `--market-type` | `-m` | `future` 或 `spot`，非法值回退 `future`（yaml 默认 `future`） |
| `--watch-mode` | `-w` | `trades` / `trades_raw` / `tickers` / `ohlcv` / `orderbook`，非法值回退 yaml 配置（yaml 默认 `trades`） |
| `--timeframe` | `-t` | K 线周期，仅 `ohlcv` 模式（yaml 默认 `1m`） |
| `--orderbook-limit` | | 订单簿档位数，仅 `orderbook` 模式（yaml 默认 `20`） |
| `--maxlen` | | Redis Stream 最大条目数，approximate trim（yaml 默认 `500000`） |
| `--env` | `-E` | 环境（dev / prod），影响 `.env` 加载（Redis 连接等），默认 `dev` |

## 注意事项

- 需要 Redis 5.0+（XADD），启动时检查 `INFO server` 版本，不足时直接报错
- 每批最多 200 个 symbol（`WS_MAX_SYMBOLS_PER_BATCH`），超出时自动分批，每批独立 asyncio Task
- 期货模式（`market_type=future`）只订阅永续合约（`type=swap`），不混入交割合约（`type=future`），因 `watch_*_for_symbols` 系列方法要求同批 symbol 类型一致；现货模式只订阅 `type=spot`
- 每 3600 秒自动重新拉取 symbol 列表（`SYMBOL_RELOAD_INTERVAL_SECONDS`），列表有变化时取消并重建全部 watch task，感知新上线/下架交易对
- 异常时指数退避重试（5s → 10s → 20s → 40s → 上限 60s），无最大重试次数限制，成功一次后计数归零
- `trades_raw` 模式通过 ccxt 选项 `streamBySubscription=True` 切换到 individual trade stream
- `ohlcv` 模式下 `watch_ohlcv_for_symbols` 返回 `{symbol: {timeframe: [[ts,o,h,l,c,v], ...]}}` 嵌套 dict，服务内自动展开
- `orderbook` 模式下 `watch_order_book_for_symbols` 每次返回单个 symbol 的完整订单簿 dict，逐条写 Stream 并同步刷新 `quant:orderbook:{symbol}` 快照；Stream 写入频率 ≈ WebSocket 事件频率，`orderbook_depth_limit` 决定每条记录体积，生产建议 ≤ 20
