# 配置说明

Quant Trading 系统使用 YAML 配置文件和环境变量来管理各个服务的配置。本文档详细说明所有配置选项。

## 配置文件结构

系统配置文件位于 `configs/` 目录：

```
configs/
├── .env.dev              # 开发环境变量
├── .env.production       # 生产环境变量
├── server_config.yaml    # 服务器配置
├── db_config.yaml        # 数据库配置
├── asset_pool_config.yaml    # 资产池服务配置
├── aggregator_config.yaml    # 聚合器服务配置
├── feature_config.yaml       # 特征服务配置
└── order_config.yaml         # 订单服务配置
```

## 生成配置模板

首次使用时，可以使用 CLI 命令生成所有配置文件模板：

```bash
main create-templates
```

## 环境变量配置

### .env.dev / .env.production

环境变量文件用于配置数据库连接、Redis 连接等敏感信息。

**服务器配置**

```bash
SERVER_HOST=0.0.0.0      # API 服务器监听地址
SERVER_PORT=8000         # API 服务器端口
```

**数据库配置（TimescaleDB）**

```bash
DB_HOST=localhost        # 数据库主机地址
DB_PORT=5432            # 数据库端口
DB_USER=postgres        # 数据库用户名
DB_PASSWORD=postgres    # 数据库密码
DB_NAME=tsdb           # 数据库名称
```

**Redis 配置**

```bash
REDIS_HOST=localhost    # Redis 主机地址
REDIS_PORT=6379        # Redis 端口
REDIS_DB=0             # Redis 数据库编号
```

**调试配置**

```bash
DEBUG=true             # 开启调试模式（仅开发环境）
```

### 使用环境配置

大多数 CLI 命令支持 `--env` / `-E` 选项指定环境：

```bash
# 使用开发环境
main ticker-db-client --env dev

# 使用生产环境
main asset-pool-service --env prod
```

## YAML 配置文件

### server_config.yaml

API 服务器配置。

```yaml
log_level: INFO         # 日志级别（DEBUG, INFO, WARNING, ERROR）
reload: false          # 热重载（仅开发环境）
workers: 4             # 工作进程数量
```

**说明：**
- `log_level`：控制日志输出详细程度
- `reload`：开发环境下启用自动重载代码
- `workers`：生产环境建议设置为 CPU 核心数

### db_config.yaml

数据库连接池配置。

```yaml
db_pool_size: 10           # 连接池大小
db_pool_timeout: 30        # 连接超时（秒）
db_pool_recycle: 3600      # 连接回收时间（秒）
db_echo: false             # 是否打印 SQL 语句
```

**说明：**
- `db_pool_size`：最大并发数据库连接数
- `db_pool_timeout`：获取连接的最大等待时间
- `db_pool_recycle`：连接存活时间，防止数据库断开连接
- `db_echo`：调试时可设为 `true` 查看 SQL 语句

### asset_pool_config.yaml

资产池服务配置，用于筛选交易标的。

```yaml
exchange: binance          # 交易所 ID
top_k: 100                # 选择 Top K 资产
days: 30                  # 回溯天数
quote_asset: USDT         # 计价货币过滤器
max_workers: 10           # 并发工作线程数
interval_hours: 24.0      # 更新间隔（小时）
```

**说明：**
- `exchange`：支持的交易所（通过 ccxt 库）
- `top_k`：根据成交量排名选择前 K 个资产
- `days`：计算成交量的历史天数
- `quote_asset`：只选择特定计价货币的交易对（如 USDT、BTC）
- `max_workers`：并发获取市场数据的线程数
- `interval_hours`：定时任务更新间隔，24.0 表示每天更新一次

**Redis 事件：**
- 发布：`asset_pool_updated` - 资产池更新完成

### aggregator_config.yaml

Tick 数据聚合服务配置，将 Tick 转换为 K 线。

```yaml
exchange: binance                  # 交易所 ID
timeframe: 1m                     # K 线时间周期
kline_retention_count: 100        # 每个品种保留 K 线数量
rebalance_interval_klines: 7      # N 根 K 线后触发再平衡
```

**说明：**
- `exchange`：交易所标识
- `timeframe`：K 线周期，支持 `1m`, `5m`, `15m`, `30m`, `1h`, `4h`, `1d`
- `kline_retention_count`：Redis 中每个交易对最多保留的 K 线数量
- `rebalance_interval_klines`：累积 N 根 K 线后发布聚合完成事件

**Redis 事件：**
- 订阅：`asset_pool_updated` - 更新订阅列表
- 发布：`kline_aggregated` - K 线聚合完成

**存储：**
- K 线数据实时存储到 Redis（key: `kline:{exchange}:{symbol}`）

### feature_config.yaml

技术特征计算服务配置。

```yaml
lookback_periods:              # 回溯周期列表
  - 7
  - 14
  - 30
```

**说明：**
- `lookback_periods`：计算滚动特征的窗口期列表（K 线根数）
  - 7：短期特征（约 1 周）
  - 14：中期特征（约 2 周）
  - 30：长期特征（约 1 月）

**计算指标：**
- 收益率（returns）
- 波动率（volatility）
- 动量指标（momentum）
- 其他技术指标

**Redis 事件：**
- 订阅：`kline_aggregated` - K 线更新触发计算
- 发布：`feature_calculated` - 特征计算完成

**存储：**
- 特征数据存储到 Redis（key: `features:{exchange}:{symbol}`）

### order_config.yaml

订单执行服务配置，包含交易策略参数。

```yaml
initial_balance: 10000.0       # 初始资金（USDT）
max_positions: 10              # 最大持仓数量
momentum_threshold: 0.0        # 动量阈值
momentum_period: 14            # 动量计算周期
```

**说明：**
- `initial_balance`：模拟交易初始资金（单位：USDT）
- `max_positions`：最多同时持有的交易对数量
- `momentum_threshold`：触发交易的动量阈值（正值买入，负值卖出）
- `momentum_period`：计算动量指标的周期（K 线根数）

**Redis 事件：**
- 订阅：`feature_calculated` - 特征更新触发交易决策
- 发布：`order_update` - 订单状态更新
- 发布：`position_update` - 持仓变化更新

**存储：**
- 持仓信息：Redis（key: `positions:{exchange}`）
- 订单历史：Redis（key: `orders:{exchange}:{symbol}`）

## 配置优先级

配置加载遵循以下优先级（从高到低）：

1. **命令行参数**：直接通过 CLI 选项传入
2. **YAML 配置文件**：服务专属配置
3. **环境变量文件**：`.env.dev` 或 `.env.production`
4. **系统环境变量**：操作系统级环境变量
5. **默认值**：代码中定义的默认值

**示例：**

```bash
# 使用配置文件中的 top_k
main asset-pool-service

# 命令行参数覆盖配置文件
main asset-pool-service --top-k 50

# 指定环境文件
main asset-pool-service --env prod --top-k 50
```

## 下一步

- [架构设计](../architecture/index.md) - 了解系统架构和数据流
- [CLI 命令](../api/cli.md) - 查看所有 CLI 命令和选项
- [用户指南](../guides/index.md) - 深入使用各个组件
- [部署运维](../deployment/index.md) - 生产环境部署指南
