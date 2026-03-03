# CLI 命令参考

Quant Trading 系统命令行接口完整参考。

> **版本说明**: 以下为 V0 已实现的命令。V1/V2 迭代中计划新增 `trading strategy`、`trading risk`、`trading monitor`、`trading account` 等子命令，详见 `docs/architecture.md` 第 3.5 节 CI/CD 流程和 `docs/sprint/back_end_sprint.md`。

## 全局选项

```bash
# 查看所有命令
main --help

# 查看特定命令帮助
main <command> --help
```

## 配置管理

### create-templates

生成所有服务的 YAML 配置文件模板。

**用法**

```bash
main create-templates
```

**说明**

在 `configs/` 目录下创建以下配置文件：
- `server_config.yaml` - 服务器配置
- `db_config.yaml` - 数据库配置
- `asset_pool_config.yaml` - 资产池配置
- `aggregator_config.yaml` - 聚合器配置
- `feature_config.yaml` - 特征计算配置
- `order_config.yaml` - 订单服务配置

**示例**

```bash
# 生成配置模板
main create-templates
```

## 业务服务器

### run-server

启动 WebSocket 市场数据服务器（API 网关）。

**用法**

```bash
main run-server [OPTIONS]
```

**选项**

- `-H, --host TEXT` - 绑定主机地址（默认：0.0.0.0）
- `-p, --port INTEGER` - 绑定端口号（默认：8000）
- `--log-level TEXT` - 日志级别（默认：info）

**示例**

```bash
# 启动服务器
main run-server

# 指定端口
main run-server --port 9000

# 调试模式
main run-server --log-level debug
```

**访问地址**

- WebSocket:
  - `ws://<host>:<port>/ws/positions`
  - `ws://<host>:<port>/ws/portfolio`
  - `ws://<host>:<port>/ws/alerts`
  - `ws://<host>:<port>/ws/services`
- 健康检查: `http://<host>:<port>/api/v1/health`

## 测试客户端

### ticker-test-client

WebSocket 测试客户端，用于测试市场数据流。

**用法**

```bash
main ticker-test-client [OPTIONS]
```

**选项**

- `-u, --uri TEXT` - WebSocket 服务器 URI（默认：ws://localhost:8000/ws/services）
- `-s, --symbol TEXT` - 交易对符号（默认：BTC/USDT）
- `-e, --exchange TEXT` - 交易所 ID（默认：binance）
- `-d, --duration INTEGER` - 运行持续时间（秒，默认：30）
- `-t, --test` - 使用模拟数据模式

**示例**

```bash
# 默认测试
main ticker-test-client

# 模拟数据模式
main ticker-test-client --test

# 连接真实交易所
main ticker-test-client --symbol ETH/USDT --duration 60

# 测试多个交易对
main ticker-test-client --symbol BNB/USDT --exchange binance
```

## 业务客户端

### ticker-db-client

数据持久化服务，将 Ticker 数据存储到 TimescaleDB。

**用法**

```bash
main ticker-db-client [OPTIONS]
```

**选项**

- `-s, --symbol TEXT` - 交易对符号（默认：BTC/USDT）
- `-e, --exchange TEXT` - 交易所 ID（默认：binance）
- `-w, --ws-uri TEXT` - WebSocket 服务器 URI（默认：ws://localhost:8000/ws/positions）
- `-d, --duration INTEGER` - 运行持续时间（秒，默认：无限）
- `-t, --test` - 从服务器接收模拟数据
- `-r, --retention INTEGER` - 数据保留策略（天数，默认：30）
- `-E, --env TEXT` - 环境配置前缀（默认：dev）

**说明**

数据库连接信息从 `configs/.env.{env}` 文件读取（例如 `.env.dev` 或 `.env.prod`）。

**示例**

```bash
# 持久化到开发环境数据库
main ticker-db-client --symbol BTC/USDT --exchange binance

# 生产环境，保留 90 天数据
main ticker-db-client --env prod --retention 90

# 测试模式（有限时间）
main ticker-db-client --test --duration 60
```

### asset-pool-service

资产池服务，根据成交量计算并发布 Top K 资产列表。

**用法**

```bash
main asset-pool-service [OPTIONS]
```

**选项**

- `-e, --exchange TEXT` - 交易所 ID（从配置读取）
- `-k, --top-k INTEGER` - Top K 资产数量（从配置读取）
- `-d, --days INTEGER` - 回溯天数（从配置读取）
- `-q, --quote TEXT` - 计价货币过滤器（从配置读取）
- `-i, --interval FLOAT` - 更新间隔（小时，从配置读取）
- `--once` - 仅运行一次后退出
- `-E, --env TEXT` - 环境配置（默认：dev）

**说明**

配置从 `configs/asset_pool_config.yaml` 加载，命令行选项会覆盖配置文件。

**示例**

```bash
# 使用配置文件启动
main asset-pool-service

# 覆盖 Top K 参数
main asset-pool-service --top-k 50

# 单次运行
main asset-pool-service --once

# 自定义更新间隔
main asset-pool-service --interval 12
```

### aggregator-service

Tick 聚合服务，将 Tick 数据聚合为 K 线（OHLCV）。

**用法**

```bash
main aggregator-service [OPTIONS]
```

**选项**

- `-e, --exchange TEXT` - 交易所 ID（从配置读取）
- `-t, --timeframe TEXT` - 聚合时间框架（从配置读取）
- `-r, --rebalance-klines INTEGER` - N 根 K 线后触发再平衡（从配置读取）
- `-n, --retention INTEGER` - 每个品种最多保留 K 线数量（从配置读取）
- `--test` - 使用模拟 Tick 数据
- `-E, --env TEXT` - 环境配置（默认：dev）

**说明**

- 配置从 `configs/aggregator_config.yaml` 加载
- 支持的时间框架：1m, 5m, 15m, 30m, 1h, 4h, 1d
- K 线数据实时存储到 Redis
- 累积 N 根 K 线后发布 `kline_aggregated` 事件触发特征计算

**示例**

```bash
# 使用配置启动
main aggregator-service

# 模拟数据模式
main aggregator-service --test

# 自定义再平衡间隔
main aggregator-service --rebalance-klines 10

# 更改时间框架
main aggregator-service --timeframe 5m
```

### feature-service

特征计算服务，基于 K 线数据计算技术指标和特征。

**用法**

```bash
main feature-service [OPTIONS]
```

**选项**

- `-p, --periods TEXT` - 逗号分隔的回溯周期列表（从配置读取）
- `-E, --env TEXT` - 环境配置（默认：dev）

**说明**

- 配置从 `configs/feature_config.yaml` 加载
- 订阅 Redis `kline_aggregated` 事件
- 计算完成后发布 `feature_calculated` 事件

**示例**

```bash
# 使用配置启动
main feature-service

# 自定义回溯周期
main feature-service --periods 5,10,20,50
```

### order-service

订单执行服务，根据特征信号执行交易策略。

**用法**

```bash
main order-service [OPTIONS]
```

**选项**

- `-b, --balance FLOAT` - 初始现金余额（从配置读取）
- `-m, --max-positions INTEGER` - 最大持仓数量（从配置读取）
- `-t, --threshold FLOAT` - 动量阈值（从配置读取）
- `-E, --env TEXT` - 环境配置（默认：dev）

**说明**

- 配置从 `configs/order_config.yaml` 加载
- 订阅 Redis `feature_calculated` 事件
- 执行交易后发布 `order_update` 和 `position_update` 事件

**示例**

```bash
# 使用配置启动
main order-service

# 自定义参数
main order-service --balance 50000 --max-positions 20 --threshold 0.02
```

### run-all-services

一次性启动所有业务服务。

**用法**

```bash
main run-all-services [OPTIONS]
```

**选项**

- `--test` - 使用模拟 Tick 数据
- `-E, --env TEXT` - 环境配置（默认：dev）

**说明**

启动以下所有服务：
- Asset Pool Service（资产池服务）
- Aggregator Service（聚合器服务）
- Feature Service（特征服务）
- Order Service（订单服务）

所有服务从各自的 YAML 配置文件加载参数。

**示例**

```bash
# 启动所有服务
main run-all-services

# 测试模式
main run-all-services --test

# 生产环境
main run-all-services --env prod
```

## 环境指定

所有服务都支持 `--env` / `-E` 选项来指定环境：

- `dev`（默认）：使用 `configs/.env.dev`
- `prod`：使用 `configs/.env.production`

环境文件应包含：
- `REDIS_HOST` - Redis 主机地址
- `REDIS_PORT` - Redis 端口
- `REDIS_DB` - Redis 数据库编号
- `DB_*` - TimescaleDB 连接参数（仅 ticker-db-client 需要）
