# 数据库设计

本文档描述量化交易系统使用的关系型数据库表结构（PostgreSQL / TimescaleDB），是表结构的**唯一权威定义**（signal_records / rebalance_records / orders 原定义散见于 `INTEGRATION_PLAN.md`，已合并于此）。覆盖「已实现」与「Phase A 实施范围」两类表；其余远期规划表（`users`、`trades`、`alerts`、`backtests` 等）见 [前端 API 设计](../frontend/01_api_design.md) 中的「后端新增数据库表」，保持规划状态。

---

## 表清单

| 表名 | 用途 | 状态 |
|------|------|------|
| `accounts` | 账户元数据（基准币种、初始资金、状态） | 已实现 |
| `portfolio_snapshots` | Portfolio 净值快照（NAV / PnL / 回撤） | 已实现 |
| `position_events` | 持仓变动事件流水（成交、调仓、资金费） | 已实现 |
| `signal_records` | 信号生成记录 | Phase A 实施范围 |
| `rebalance_records` | 调仓决策与执行记录 | Phase A 实施范围 |
| `orders` | 订单流水（OrderService 下单记录） | Phase A 实施范围 |
| `account_positions` | 账户持仓快照（按时间 + 标的） | 历史设计（ORM 模型已移除，仅存档） |
| `symbol_ohlcv` | 标的 OHLCV K 线快照（按时间 + 标的） | 历史设计（ORM 模型已移除，仅存档） |

---

## accounts（已实现）

账户元数据表，每个 `account_id` 唯一标识一个交易账户。对应 ORM 模型 `AccountModel`。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `account_id` | `VARCHAR(64)` | NOT NULL, PK | 账户唯一标识 |
| `account_name` | `VARCHAR(128)` | NOT NULL | 账户名称 |
| `base_ccy` | `VARCHAR(16)` | NOT NULL, DEFAULT 'USDT' | 基准币种 |
| `initial_cash` | `NUMERIC(36,18)` | NOT NULL | 初始资金 |
| `status` | `VARCHAR(32)` | NOT NULL, DEFAULT 'active' | 账户状态 |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | 创建时间 |
| `updated_at` | `TIMESTAMPTZ` | NULL, DEFAULT now() | 更新时间（on update 自动刷新） |

**主键**: `account_id`

```sql
CREATE TABLE accounts (
    account_id    VARCHAR(64)     NOT NULL,
    account_name  VARCHAR(128)    NOT NULL,
    base_ccy      VARCHAR(16)     NOT NULL DEFAULT 'USDT',
    initial_cash  NUMERIC(36,18)  NOT NULL,
    status        VARCHAR(32)     NOT NULL DEFAULT 'active',
    created_at    TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ              DEFAULT now(),
    PRIMARY KEY (account_id)
);
```

---

## portfolio_snapshots（已实现）

Portfolio 净值快照表，定时从 Redis 实时快照持久化到 PostgreSQL，用于历史净值查询、盈亏统计和回撤分析。对应 ORM 模型 `PortfolioSnapshotModel`。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | `SERIAL` | NOT NULL, PK | 自增主键 |
| `account_id` | `VARCHAR(64)` | NOT NULL, FK → accounts | 关联账户 |
| `nav` | `NUMERIC(36,18)` | NOT NULL | 净值 |
| `total_equity` | `NUMERIC(36,18)` | NULL | 总权益 |
| `available_balance` | `NUMERIC(36,18)` | NULL | 可用余额 |
| `used_margin` | `NUMERIC(36,18)` | NULL | 已用保证金 |
| `unrealized_pnl` | `NUMERIC(36,18)` | NULL | 未实现盈亏 |
| `daily_pnl` | `NUMERIC(36,18)` | NULL | 当日盈亏 |
| `daily_pnl_pct` | `NUMERIC(36,18)` | NULL | 当日盈亏百分比 |
| `drawdown` | `NUMERIC(36,18)` | NULL | 当前回撤 |
| `max_drawdown` | `NUMERIC(36,18)` | NULL | 最大回撤 |
| `total_return` | `NUMERIC(36,18)` | NULL | 累计收益率 |
| `recorded_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | 快照时间 |

**主键**: `id`；**索引**: `ix_portfolio_snapshots_account_time (account_id, recorded_at)`

```sql
CREATE TABLE portfolio_snapshots (
    id                 SERIAL          PRIMARY KEY,
    account_id         VARCHAR(64)     NOT NULL REFERENCES accounts(account_id),
    nav                NUMERIC(36,18)  NOT NULL,
    total_equity       NUMERIC(36,18),
    available_balance  NUMERIC(36,18),
    used_margin        NUMERIC(36,18),
    unrealized_pnl     NUMERIC(36,18),
    daily_pnl          NUMERIC(36,18),
    daily_pnl_pct      NUMERIC(36,18),
    drawdown           NUMERIC(36,18),
    max_drawdown       NUMERIC(36,18),
    total_return       NUMERIC(36,18),
    recorded_at        TIMESTAMPTZ     NOT NULL DEFAULT now()
);
CREATE INDEX ix_portfolio_snapshots_account_time
    ON portfolio_snapshots (account_id, recorded_at);
```

---

## position_events（已实现）

持仓变动事件流水表，记录每一次仓位变化（成交、手动调整、资金费等）。对应 ORM 模型 `PositionEventModel`。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | `SERIAL` | NOT NULL, PK | 自增主键 |
| `account_id` | `VARCHAR(64)` | NOT NULL, FK → accounts, 索引 | 关联账户 |
| `symbol` | `VARCHAR(64)` | NOT NULL, 索引 | 交易对符号 |
| `event_time` | `TIMESTAMPTZ` | NOT NULL | 事件发生时间 |
| `side` | `VARCHAR(16)` | NOT NULL | 方向（long/short 或 buy/sell） |
| `qty_delta` | `NUMERIC(36,18)` | NOT NULL | 数量变化（带符号） |
| `fill_price` | `NUMERIC(36,18)` | NOT NULL | 成交价 |
| `fee` | `NUMERIC(36,18)` | NOT NULL, DEFAULT 0 | 手续费 |
| `fee_asset` | `VARCHAR(16)` | NOT NULL, DEFAULT 'USDT' | 手续费币种 |
| `event_type` | `VARCHAR(32)` | NOT NULL, DEFAULT 'fill' | 事件类型（open/fill/close/funding 等） |
| `note` | `TEXT` | NULL | 备注 |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | 写入时间 |

**主键**: `id`

```sql
CREATE TABLE position_events (
    id          SERIAL          PRIMARY KEY,
    account_id  VARCHAR(64)     NOT NULL REFERENCES accounts(account_id),
    symbol      VARCHAR(64)     NOT NULL,
    event_time  TIMESTAMPTZ     NOT NULL,
    side        VARCHAR(16)     NOT NULL,
    qty_delta   NUMERIC(36,18)  NOT NULL,
    fill_price  NUMERIC(36,18)  NOT NULL,
    fee         NUMERIC(36,18)  NOT NULL DEFAULT 0,
    fee_asset   VARCHAR(16)     NOT NULL DEFAULT 'USDT',
    event_type  VARCHAR(32)     NOT NULL DEFAULT 'fill',
    note        TEXT,
    created_at  TIMESTAMPTZ     NOT NULL DEFAULT now()
);
CREATE INDEX ix_position_events_account_id ON position_events (account_id);
CREATE INDEX ix_position_events_symbol ON position_events (symbol);
```

---

## signal_records（Phase A 实施范围）

信号生成记录表，记录每一次信号生成，用于审计与回测对比。`account_id` 即栈/账户维度（每策略一栈一账户）。**streaming 策略（策略 6）逐美元 bar 产信号，仅在目标有效变更时写一行**（非每 bar），避免爆量；scheduler 策略每次调度写一行。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | `SERIAL` | NOT NULL, PK | 自增主键 |
| `account_id` | `VARCHAR(64)` | NOT NULL, FK → accounts | 关联账户 |
| `timestamp` | `TIMESTAMPTZ` | NOT NULL | 信号生成时间 |
| `strategy_name` | `VARCHAR(64)` | NOT NULL | 策略名 |
| `signal_type` | `VARCHAR(32)` | NULL | 信号类型，当前统一 `target_weight` |
| `rebalance_id` | `VARCHAR(128)` | NULL, 索引 | 调仓批次 ID，确定性生成：scheduler = 策略名 + 计划触发时刻；streaming = 策略名 + bar 收盘时间戳 |
| `signal_data` | `JSONB` | NOT NULL | 完整信号：`{symbol: weight/strength}` 映射 |
| `metadata` | `JSONB` | NULL | 策略参数快照、排名数据等 |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | 写入时间 |

```sql
-- Phase A 计划稿
CREATE TABLE signal_records (
    id             SERIAL        PRIMARY KEY,
    account_id     VARCHAR(64)   NOT NULL REFERENCES accounts(account_id),
    timestamp      TIMESTAMPTZ   NOT NULL,
    strategy_name  VARCHAR(64)   NOT NULL,
    signal_type    VARCHAR(32),
    rebalance_id   VARCHAR(128),
    signal_data    JSONB         NOT NULL,
    metadata       JSONB,
    created_at     TIMESTAMPTZ   NOT NULL DEFAULT now()
);
CREATE INDEX ix_signal_records_rebalance_id ON signal_records (rebalance_id);
```

---

## rebalance_records（Phase A 实施范围）

调仓决策与执行记录表，记录每一次调仓的前后权重、订单与成交汇总。原定义见 `INTEGRATION_PLAN.md` Step 1.3，已合并于此。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | `SERIAL` | NOT NULL, PK | 自增主键 |
| `account_id` | `VARCHAR(64)` | NOT NULL, FK → accounts | 关联账户 |
| `signal_record_id` | `INTEGER` | NULL, FK → signal_records | 关联信号记录 |
| `rebalance_id` | `VARCHAR(128)` | NOT NULL, UNIQUE | 调仓批次 ID（幂等键，与 Redis `quant:order:rebalance:{rebalance_id}` 对应；Phase A 契约新增） |
| `timestamp` | `TIMESTAMPTZ` | NOT NULL | 调仓时间 |
| `before_weights` | `JSONB` | NOT NULL | 调仓前持仓权重（策略虚拟账本口径） |
| `target_weights` | `JSONB` | NOT NULL | 目标权重 |
| `after_weights` | `JSONB` | NULL | 调仓后实际权重 |
| `orders` | `JSONB` | NOT NULL | 提交的订单列表 |
| `execution_summary` | `JSONB` | NULL | 成交汇总：滑点、手续费、偏差 |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | 写入时间 |

```sql
-- Phase A 计划稿
CREATE TABLE rebalance_records (
    id                 SERIAL        PRIMARY KEY,
    account_id         VARCHAR(64)   NOT NULL REFERENCES accounts(account_id),
    signal_record_id   INTEGER       REFERENCES signal_records(id),
    rebalance_id       VARCHAR(128)  NOT NULL UNIQUE,
    timestamp          TIMESTAMPTZ   NOT NULL,
    before_weights     JSONB         NOT NULL,
    target_weights     JSONB         NOT NULL,
    after_weights      JSONB,
    orders             JSONB         NOT NULL,
    execution_summary  JSONB,
    created_at         TIMESTAMPTZ   NOT NULL DEFAULT now()
);
```

---

## orders（Phase A 实施范围）

订单流水表，由 Phase A 的 OrderService 写入，记录每一笔（含 dry-run）订单的提交与终态。`INTEGRATION_PLAN.md` 未含独立 orders 表定义，本表结合前端 API 设计的关键字段（order_id, symbol, side, amount, price, status, strategy_name, time）与 Phase A 幂等设计（rebalance_id + clientOrderId）整理，以实施时迁移为准。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `id` | `BIGSERIAL` | NOT NULL, PK | 自增主键 |
| `account_id` | `VARCHAR(64)` | NOT NULL, FK → accounts | 关联账户 |
| `rebalance_id` | `VARCHAR(128)` | NOT NULL, 索引 | 所属调仓批次（幂等链路一环） |
| `client_order_id` | `VARCHAR(64)` | NOT NULL, UNIQUE | 客户端订单 ID（交易所幂等去重） |
| `exchange_order_id` | `VARCHAR(64)` | NULL | 交易所返回订单 ID |
| `strategy_name` | `VARCHAR(64)` | NOT NULL | 来源策略（虚拟账本归属） |
| `symbol` | `VARCHAR(64)` | NOT NULL | 交易对符号 |
| `side` | `VARCHAR(8)` | NOT NULL | buy / sell |
| `order_type` | `VARCHAR(16)` | NOT NULL, DEFAULT 'market' | 订单类型 |
| `amount` | `NUMERIC(36,18)` | NOT NULL | 委托数量 |
| `price` | `NUMERIC(36,18)` | NULL | 委托价（市价单为 NULL） |
| `filled_amount` | `NUMERIC(36,18)` | NULL | 成交数量 |
| `avg_fill_price` | `NUMERIC(36,18)` | NULL | 成交均价 |
| `fee` | `NUMERIC(36,18)` | NULL | 手续费 |
| `status` | `VARCHAR(16)` | NOT NULL | pending / submitted / filled / failed / canceled |
| `dry_run` | `BOOLEAN` | NOT NULL, DEFAULT true | 是否模拟下单（Phase A 默认 dry_run） |
| `error` | `TEXT` | NULL | 失败原因 |
| `created_at` | `TIMESTAMPTZ` | NOT NULL, DEFAULT now() | 创建时间 |
| `updated_at` | `TIMESTAMPTZ` | NULL | 状态更新时间 |

```sql
-- Phase A 计划稿
CREATE TABLE orders (
    id                 BIGSERIAL       PRIMARY KEY,
    account_id         VARCHAR(64)     NOT NULL REFERENCES accounts(account_id),
    rebalance_id       VARCHAR(128)    NOT NULL,
    client_order_id    VARCHAR(64)     NOT NULL UNIQUE,
    exchange_order_id  VARCHAR(64),
    strategy_name      VARCHAR(64)     NOT NULL,
    symbol             VARCHAR(64)     NOT NULL,
    side               VARCHAR(8)      NOT NULL,
    order_type         VARCHAR(16)     NOT NULL DEFAULT 'market',
    amount             NUMERIC(36,18)  NOT NULL,
    price              NUMERIC(36,18),
    filled_amount      NUMERIC(36,18),
    avg_fill_price     NUMERIC(36,18),
    fee                NUMERIC(36,18),
    status             VARCHAR(16)     NOT NULL,
    dry_run            BOOLEAN         NOT NULL DEFAULT true,
    error              TEXT,
    created_at         TIMESTAMPTZ     NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ
);
CREATE INDEX ix_orders_rebalance_id ON orders (rebalance_id);
```

---

## account_positions

> **状态: 历史设计（已废弃）**。对应 ORM 模型已从当前代码库移除：实时持仓现由 Redis `quant:account:positions` 快照承载，持仓变动流水由 `position_events` 表记录。以下定义仅供存档。

账户持仓快照表，按 `snapshot_time` 与 `symbol` 唯一确定一条记录。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `snapshot_time` | `TIMESTAMPTZ` | NOT NULL, PK | 快照时间（含时区） |
| `symbol` | `VARCHAR` | NOT NULL, PK | 交易对符号 |
| `side` | `VARCHAR` | NOT NULL | 方向（如 long/short） |
| `size` | `NUMERIC` | NOT NULL | 持仓数量 |

**主键**: `(snapshot_time, symbol)`

```sql
CREATE TABLE account_positions (
    snapshot_time  TIMESTAMPTZ  NOT NULL,
    symbol         VARCHAR      NOT NULL,
    side           VARCHAR      NOT NULL,
    size           NUMERIC      NOT NULL,
    PRIMARY KEY (snapshot_time, symbol)
);
```

---

## symbol_ohlcv

> **状态: 历史设计（已废弃）**。对应 ORM 模型已从当前代码库移除：K 线数据现由 Redis Stream `market:ohlcv` 与缓存键 `quant:kline:{symbol}:{timeframe}` 承载。以下定义仅供存档。

标的 OHLCV K 线快照表，按 `snapshot_time` 与 `symbol` 唯一确定一条记录。

| 字段 | 类型 | 约束 | 说明 |
|------|------|------|------|
| `snapshot_time` | `TIMESTAMPTZ` | NOT NULL, PK | 快照时间（K 线时间戳，含时区） |
| `symbol` | `VARCHAR` | NOT NULL, PK | 交易对符号 |
| `open` | `NUMERIC` | NOT NULL | 开盘价 |
| `high` | `NUMERIC` | NOT NULL | 最高价 |
| `low` | `NUMERIC` | NOT NULL | 最低价 |
| `close` | `NUMERIC` | NOT NULL | 收盘价 |
| `volume` | `NUMERIC` | NOT NULL | 成交量 |

**主键**: `(snapshot_time, symbol)`

```sql
CREATE TABLE symbol_ohlcv (
    snapshot_time  TIMESTAMPTZ  NOT NULL,
    symbol         VARCHAR      NOT NULL,
    open           NUMERIC      NOT NULL,
    high           NUMERIC      NOT NULL,
    low            NUMERIC      NOT NULL,
    close          NUMERIC      NOT NULL,
    volume         NUMERIC      NOT NULL,
    PRIMARY KEY (snapshot_time, symbol)
);
```

---

## 与代码的对应关系

ORM 模型定义见后端仓库 `quant_trading_backend/src/quant_trading/data/db/models/`：

- `account_model.py` — `AccountModel` → 表 `accounts`（已实现）
- `portfolio_snapshot_model.py` — `PortfolioSnapshotModel` → 表 `portfolio_snapshots`（已实现）
- `position_event_model.py` — `PositionEventModel` → 表 `position_events`（已实现）
- `signal_records` / `rebalance_records` / `orders` 的模型将在 Phase A 实施时新增（Phase A 实施范围）
- 旧版 `AccountPosition` / `SymbolOhlcv` 模型已移除，对应表为历史设计

建表方式：`main db init`（按 ORM 创建缺失表），或执行上述 DDL。
