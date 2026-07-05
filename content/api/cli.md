# CLI 命令

入口 `main`（`quant_trading.app.cli:main`）。每条竖直栈的服务用 `--account` 绑定账户与键空间；共享层服务不带 `--account`。

!!! note "本页是目标形态"
    下文按两账户、`--account` 的目标形态书写。当前实测为单账户（仅 `--env`）、已注册 8 个命令——逐命令差异见末尾[实现状态](#status)。事件契约见 [Redis](redis.md)，不在此重复。

## 通用选项

所有 `run` 子命令共享：

| 选项 | 说明 |
|------|------|
| `-E, --env` | 环境 `dev` / `prod`，加载 `.env.{env}`。默认 `dev` |
| `-A, --account` | 栈/账户标识，每栈服务必填。决定键空间 `quant:{account}:*` 与 `.env.acct{account}` |

```bash
main --help            # 全部命令
main <command> --help  # 单命令帮助
```

## 数据库与配置

### `main db init`

按 ORM 创建缺失的数据库表，幂等。首次部署执行一次。

**用法**

```bash
main db init [-E env]
```

### `main configs`

管理 YAML 配置。

| 子命令 | 作用 |
|--------|------|
| `configs list` | 列出所有配置类对应的 YAML 路径与状态 |
| `configs generate [-f]` | 按默认值生成模板，`-f` 覆盖已存在 |
| `configs load` | 加载并校验所有 YAML |

## 共享层服务

单实例，不带 `--account`。

### `main data-service run`

采集公共行情写入 Redis。

**用法**

```bash
main data-service run [选项]
```

**选项**

| 选项 | 说明 |
|------|------|
| `-w, --watch-mode` | `trades_raw` / `orderbook` / `ohlcv` / `tickers`。默认 `trades` |
| `-e, --exchange` | 交易所 ID。默认 `binanceusdm` |
| `--orderbook-limit` | 订单簿档位，`orderbook` 模式。默认 20 |
| `--maxlen` | Stream 最大条数。默认 500000 |

**示例**

```bash
main data-service run --watch-mode trades_raw   # 逐笔流，供策略 6
main data-service run --watch-mode orderbook    # 盘口快照，供两栈执行校验
```

### `main api-gateway run`

对外 HTTP / WebSocket 网关。

**用法**

```bash
main api-gateway run [-H host] [-P port]
```

**选项**

| 选项 | 说明 |
|------|------|
| `-H, --host` | 监听地址。默认 `0.0.0.0` |
| `-P, --port` | 监听端口。默认 8000 |

### `main alert-service run`

聚合各栈拒绝 / 失败事件与心跳超时，推送 Telegram。

**用法**

```bash
main alert-service run
```

### `main asset-pool-service run`

写资产池 SET。两策略自发现宇宙，通常不需要。

**用法**

```bash
main asset-pool-service run
```

## 每栈服务

每账户一份，`--account` 必填。一条栈含五个服务：

| 命令 | 角色 |
|------|------|
| `strategy-service run` | 产出目标组合 |
| `risk-service run` | USDT sizing |
| `execution-service run` | 盘口校验 |
| `order-service run` | 差量下单 |
| `account-service run` | 账户同步 |

### `main strategy-service run`

运行该栈的策略，产出目标组合。按策略 `execution_mode` 自动选 scheduler 或 event_driven。

**用法**

```bash
main strategy-service run -A <account>
```

**示例**

```bash
main strategy-service run -A acctA   # 策略 6，event_driven
main strategy-service run -A acctB   # 策略 5，scheduler
```

### `main risk-service run`

按账户权益给目标组合做 USDT sizing。参数见 `yamls/risk.<account>.yaml`。

**用法**

```bash
main risk-service run -A <account>
```

### `main execution-service run`

读盘口快照做逐腿深度 / 新鲜度校验，名义金额折算合约数。

**用法**

```bash
main execution-service run -A <account>
```

### `main order-service run`

对本账户净持仓做差量下单（先平后开），幂等，默认 `dry_run`。

**用法**

```bash
main order-service run -A <account>
```

### `main account-service run`

轮询本子账户私有数据，写组合快照。

**用法**

```bash
main account-service run -A <account> [-p poll]
```

**选项**

| 选项 | 说明 |
|------|------|
| `-p, --poll` | 轮询间隔，秒。默认 30 |

## 训练

### `main strategy6-train`

策略 6 的季度离线重训，产出模型工件供插件热加载。非常驻进程。

**用法**

```bash
main strategy6-train
```

## 起栈示例

```bash
main db init

# 共享层
main data-service run --watch-mode trades_raw &
main data-service run --watch-mode orderbook &
main api-gateway run &

# 栈 A（策略 6）+ 栈 B（策略 5）
for s in strategy risk execution order account; do
  main $s-service run -A acctA &
  main $s-service run -A acctB &
done
```

## 环境与密钥

每栈一份 `.env.acct{X}`，含 `ACCOUNT_ID`（= `{account}`）与该子账户的 `EXCHANGE_API_KEY` / `EXCHANGE_SECRET` / `EXCHANGE_PRIVATE_KEY_PATH`；共享 `REDIS_*` / `DB_*`。密钥勿提交仓库，生产走 Secrets Manager。详见[配置说明](../getting-started/configuration.md)。

## 实现状态 {#status}

| 命令 | 状态 |
|------|------|
| `db init`、`configs *` | 已实现 |
| `data-service run`、`asset-pool-service run` | 已实现 |
| `strategy-service run`（scheduler）、`risk-service run`、`account-service run` | 已实现 |
| `execution-service run` | 服务已实现，CLI 命令未注册（`main` 暂不可调） |
| `--account` 多账户与键空间 | 规划 |
| `strategy-service run`（event_driven）、`order-service run` | 规划 |
| `api-gateway run`、`alert-service run`、`strategy6-train` | 规划 |
