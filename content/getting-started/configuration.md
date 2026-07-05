# 配置说明

Quant Trading 系统使用 YAML 配置文件和环境变量来管理各个服务的配置。本文档详细说明所有配置选项。

- **YAML 配置**（仓库根目录 `yamls/`）：服务运行参数，非敏感，字段名以 `common/configs/yamls/services/` 下的 Pydantic 配置类为准。
- **环境变量 / `.env` 文件**（仓库根目录）：数据库、Redis、交易所密钥等连接与敏感信息。

## 配置文件结构

系统 YAML 配置位于仓库根目录 `yamls/`（可通过环境变量 `YAML_ROOT` 重定向）：

```
yamls/
├── strategy.yaml               # 策略服务配置
├── risk.yaml                   # 风控服务配置
├── execution.yaml              # 执行网关配置
├── data.yaml                   # 行情数据服务配置
├── bar_source.yaml             # Bar 聚合适配服务配置
├── feature.yaml                # 特征服务配置
├── asset_pool.yaml             # 资产池配置
├── account.yaml                # 账户服务配置
└── exchange.yaml               # 交易所通用配置（非敏感）
```

（Phase A 计划）新增 `yamls/order.yaml`（订单服务配置，见下文）。

环境变量文件位于仓库根目录：`.env`（兜底）、`.env.dev`、`.env.prod`，由 CLI 的 `--env` 选项选择加载。

## 生成配置模板

首次使用时，可以使用 CLI 命令生成 / 检查配置文件：

```bash
# 查看所有已注册配置类及对应 YAML 的存在状态
main configs list

# 按默认值生成全部 YAML 模板（已存在则跳过；-f 覆盖）
main configs generate
main configs generate --overwrite

# 验证所有 YAML 可正确解析
main configs load
```

## 环境变量配置

### .env / .env.dev / .env.prod

环境变量按前缀分组，由 `common/configs/envs/services/` 下的 Pydantic 配置类加载。**以下只列键名与含义，请勿将真实密钥提交到仓库。**

**Redis 配置（前缀 `REDIS_`）**

| 键 | 说明 | 默认值 |
|----|------|--------|
| `REDIS_HOST` | Redis 主机地址 | localhost |
| `REDIS_PORT` | Redis 端口 | 6379 |
| `REDIS_DB` | Redis 数据库编号 | 0 |
| `REDIS_PASSWORD` | Redis 密码 | （空） |
| `REDIS_SSL` | 是否启用 SSL（`rediss://`） | false |

**数据库配置（前缀 `DB_`，全部必填，无默认值）**

| 键 | 说明 |
|----|------|
| `DB_HOST` | 数据库主机地址 |
| `DB_PORT` | 数据库端口 |
| `DB_USER` | 数据库用户名 |
| `DB_PASSWORD` | 数据库密码 |
| `DB_NAME` | 数据库名称 |
| `DB_POOL_SIZE` | 连接池大小 |
| `DB_MAX_OVERFLOW` | 连接池溢出上限 |
| `DB_ECHO` | 是否打印 SQL 语句 |

**交易所认证（前缀 `EXCHANGE_`，敏感）**

| 键 | 说明 |
|----|------|
| `EXCHANGE_API_KEY` | API key |
| `EXCHANGE_SECRET` | RSA 私钥内容（CI / 生产） |
| `EXCHANGE_PRIVATE_KEY_PATH` | RSA 私钥文件路径（本地开发） |

加载优先级：`EXCHANGE_SECRET` > `EXCHANGE_PRIVATE_KEY_PATH`。非敏感参数（exchange_id、default_type、sandbox）放在 `yamls/exchange.yaml`，不在环境变量中。

**Account 服务部署参数（前缀 `ACCOUNT_SERVICE_`）**

| 键 | 说明 | 默认值 |
|----|------|--------|
| `ACCOUNT_SERVICE_ENV` | dev / prod | dev |
| `ACCOUNT_SERVICE_HOST` | API 监听地址 | 0.0.0.0 |
| `ACCOUNT_SERVICE_PORT` | API 监听端口 | 8000 |
| `ACCOUNT_SERVICE_LOG_LEVEL` | uvicorn 日志级别 | info |

**路径覆盖（部署 / Docker 用）**

| 键 | 说明 |
|----|------|
| `QUANT_PROJECT_ROOT` | 项目根目录（Docker 中默认 `/app`） |
| `YAML_ROOT` | YAML 配置目录（默认 `{PROJECT_ROOT}/yamls`） |
| `QUANT_CONFIG_ROOT` | configs 目录（默认 `{PROJECT_ROOT}/configs`） |

### 使用环境配置

各 `run` 子命令支持 `--env` / `-E` 选项指定环境：

```bash
# 使用开发环境（加载仓库根目录 .env.dev）
main data-service run --env dev

# 使用生产环境（加载仓库根目录 .env.prod）
main account-service run --env prod
```

## YAML 配置文件

字段名以 Pydantic 配置类为准；YAML 中缺失的字段使用代码默认值。

### strategy.yaml（StrategyConfig）

策略服务配置。调度器**每日**在 `rebalance_hour_utc:rebalance_minute_utc`（UTC）触发一次激活策略；当前唯一策略 `monthly_majors_vs_alts` 为 self_feeding（策略内部经 REST 自拉日线，不依赖流式特征链），并基于 `anchor_date` + `rebal_days` 网格自行判断当日是"30 天网格全调仓日"还是"仅止损检查日"。

```yaml
mode: single                        # 策略模式：single | ensemble
active_strategy: monthly_majors_vs_alts  # 激活的策略名（当前唯一策略）
rebalance_hour_utc: 0               # 每日调度时点（UTC 小时）
rebalance_minute_utc: 10            # 每日调度时点（UTC 分钟），00:10 = 日线收盘后
feature_collect_window_seconds: 300 # 特征新鲜度窗口（秒），仅非 self_feeding 策略使用
parameters:                         # monthly_majors_vs_alts 策略参数
  anchor_date: "2026-01-01"         # 调仓网格锚点（UTC）
  rebal_days: 30                    # 固定调仓网格间隔（天）
  top_n: 100                        # 做空 alt 篮子数量
  btc_weight: 0.7                   # BTC 多头权重
  eth_weight: 0.3                   # ETH 多头权重
  dvol_lookback_days: 30            # 美元成交量排名回看窗口（天）
  stop_pct: 0.30                    # 空头单币硬止损（较入场价上涨 30% 即平掉该腿）
  target_vol: 0.20                  # 年化 NAV 目标波动率
  vol_window_days: 30               # 已实现波动率滚动窗口（天，日对数收益）
  lev_min: 0.30                     # 杠杆下限
  lev_max: 2.50                     # 杠杆上限
  min_candidates: 10                # 候选数低于此值策略空仓
  universe_prefilter_top: 200       # 拉取 K 线前按 24h 成交量预筛数量
  vol_recon_days: 150               # 波动率估计重建的历史天数
```

**（Phase A 计划）新增字段：**

```yaml
cron_schedule: "59 23 1 * *"   # croniter 表达式，支持月频等任意换仓频率，替代固定 hour/minute
catch_up: true                 # 服务重启后对错过的换仓时点做补偿执行
```

### crypto_pairs_strategy.yaml（已移除）

> 已移除（2026-06-11，git 历史可查）：crypto pairs 均值回归策略（`crypto_pairs_mean_reversion`）、配置类 `CryptoPairsStrategyConfig`、本 YAML 文件及生成命令 `main strategy-service import-crypto-pairs` 已随"策略清零"改造一并下线。

### risk.yaml（RiskConfig）

风控服务配置。【已实现】equity sizing，方案 `target_weight_equity`：`notional = |target_weight| × equity`。

```yaml
enabled: true                   # 是否启用风控（false 时整批拒绝，发布 quant:risk_rejected）
always_approve: false           # 是否无条件通过（预留开关，当前 sizing 逻辑未读取）
default_equity_usdt: 1000.0     # 权益回退值（USDT）：账户余额快照缺失或过期时使用
balance_max_age_seconds: 600    # quant:account:balance 快照最大可用时长（秒），超龄回退 default_equity_usdt
min_notional_usdt: 5.0          # 单腿最小名义金额（USDT），低于此值剔除（交易所最小下单量）
max_gross_exposure: 5.0         # 组合总名义敞口上限（相对账户权益的倍数），超出时全体等比缩减
```

> 旧的 `pair_base_notional_usdt`（pair spread sizing）已随 crypto pairs 策略移除（2026-06-11，git 历史可查）。
>
> **（Phase A 计划）** 组合级检查扩展：单标的最大权益占比（max_position_pct）、回撤、保证金预留、换手等规则化拒绝。

### execution.yaml（ExecutionConfig）

执行网关（ExecutionService）配置。

```yaml
orderbook_max_age_ms: 5000   # 订单簿快照最大可用时长（毫秒），超龄按 stale 整批拒绝
```

### data.yaml（DataConfig）

行情数据服务配置。

```yaml
exchange_id: binanceusdm     # ccxt.pro 交易所 ID
market_type: future          # future | spot
watch_mode: trades           # trades | trades_raw | tickers | ohlcv | orderbook
stream_maxlen: 500000        # Redis Stream 最大条数（近似裁剪）
timeframe: 1m                # K 线周期（仅 ohlcv 模式）
orderbook_depth_limit: 20    # 订单簿档位数（仅 orderbook 模式）
WS_RECONNECT_DELAY_SECONDS: 5         # WebSocket 重连延迟（秒）
WS_MAX_SYMBOLS_PER_BATCH: 200         # 每批最大订阅数（binanceusdm 上限 200，分批订阅）
SYMBOL_RELOAD_INTERVAL_SECONDS: 3600  # 交易对列表重载间隔（秒）
```

**Redis 写入：** 按 `watch_mode` 写入 `market:trades` / `market:tickers` / `market:ohlcv` / `market:orderbook` Stream；orderbook 模式同时覆盖写最新快照键 `quant:orderbook:{symbol}`。

### bar_source.yaml（BarSourceConfig）

Bar 聚合适配服务（BarSourceAdapterService）配置：消费 trades Stream，聚合为统一 bar。

```yaml
bar_type: time_bar           # time_bar | dollar_bar
exchange_id: binanceusdm
market_type: future
time_bar_interval: 1m        # time_bar 切分周期（dollar_bar 时下游 timeframe 为 "dollar"）
tick_features_enabled: false # 是否计算 tick 微观特征
tick_features_rolling_window: 50
consumer_group: bar_source_consumers  # Stream 消费者组
stream_key: market:trades    # 消费的 Stream 键
batch_size: 100              # 每次批量读取条数
block_ms: 1000               # XREADGROUP 阻塞时长（毫秒）
symbol_reload_interval_seconds: 3600   # 资产池重载间隔（秒）
stream_read_retry_delay_seconds: 5     # 消费异常重试基础延迟（秒，指数退避）
```

**Redis 事件：** 发布 `quant:bar_normalized`；覆盖写 `quant:bar:normalized:{symbol}:{timeframe}`。

### feature.yaml（FeatureConfig）

特征计算服务配置。

```yaml
lookback_periods: [7, 14, 30]   # MA / rolling 计算周期列表
return_periods_hours: [2, 4]    # 收益率回溯周期（小时）
zscore_window: 30               # z-score 滑窗大小
zscore_negate: true             # 是否取负（负收益 z-score 越高排名越靠前）
```

**Redis 事件：** 订阅 `quant:bar_normalized`；发布 `quant:feature_calculated`；写 `quant:features:latest:{symbol}`。

### asset_pool.yaml（AssetPoolConfig）

资产池配置（当前为硬编码交易对列表）。

```yaml
exchange_id: binanceusdm
symbols:                     # 交易对列表（写入 Redis SET）
  - BTC/USDT
  - ETH/USDT
  # ...
```

**Redis 事件：** 写 `quant:asset_pool:{exchange}` SET；发布 `quant:asset_pool_updated`。

### account.yaml（AccountConfig）

账户服务配置。

```yaml
account_id: default          # 账户唯一标识（DB 持久化用）
account_name: 默认账户        # 账户显示名称
poll_interval: 30            # REST 轮询间隔（秒）
db_persist_interval: 30      # DB 持久化间隔（秒）
ws_max_consecutive_errors: 5 # WS 连续失败 N 次后切 REST
ws_reconnect_delay: 10       # WS 重连延迟（秒）
heartbeat_ttl: 120           # Redis 心跳 TTL（秒）
nav_history_hours: 24        # NAV 历史保留窗口（小时，Sorted Set）
```

### exchange.yaml（ExchangeConfig）

交易所通用配置（非敏感，被 AccountService、DataService 等复用）。

```yaml
exchange_id: binance         # ccxt 交易所 ID
default_type: future         # spot | future
sandbox: true                # 是否测试网
```

认证信息（api_key、secret）通过环境变量 `EXCHANGE_*` 提供，不放入 YAML。

### order.yaml（Phase A 计划）

订单服务（OrderService）配置，Phase A 新增文件，随 OrderService 落地。

```yaml
dry_run: true                # 默认 dry-run，不真实下单
min_delta_pct: 0.01          # 差量下单最小变动比例，低于此值不动仓
```

具体字段以 Phase A 实施时的 OrderConfig 配置类为准，预计还包含重试次数、下单限速等键。

## 配置优先级

**YAML 域**（服务运行参数），从高到低：

1. **命令行参数**：仅部分命令暴露（如 data-service、account-service）
2. **YAML 配置文件**：`yamls/*.yaml`
3. **代码默认值**：Pydantic 配置类中的默认值

**环境变量域**（连接与敏感信息），从高到低：

1. **`.env.{env}` 文件**：CLI `--env` 选项加载（覆盖写入进程环境变量）
2. **系统环境变量**
3. **仓库根目录 `.env`**：pydantic-settings 兜底加载
4. **代码默认值**

**示例：**

```bash
# 使用 yamls/data.yaml 中的 watch_mode
main data-service run

# 命令行参数覆盖 YAML 配置
main data-service run --watch-mode orderbook

# 指定环境文件
main data-service run --env prod --watch-mode orderbook
```

## 下一步

- [架构设计](../architecture/index.md) - 了解系统架构和数据流
- [CLI 命令](../api/cli.md) - 查看所有 CLI 命令和选项
- [用户指南](../guides/index.md) - 深入使用各个组件
- [部署运维](../deployment/index.md) - 生产环境部署指南
