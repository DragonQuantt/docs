# 数据库设计

本文档描述量化交易系统使用的关系型数据库表结构（PostgreSQL / TimescaleDB）。仅涉及当前已落地的核心表；更多规划表见 [前端 API 设计](../frontend/01_api_design.md) 中的「后端新增数据库表」。

---

## 表清单

| 表名 | 用途 |
|------|------|
| `account_positions` | 账户持仓快照（按时间 + 标的） |
| `symbol_ohlcv` | 标的 OHLCV K 线快照（按时间 + 标的） |

---

## account_positions

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

ORM 模型定义见后端仓库：

- `quant_trading_backend/src/quant_trading/data/db/models.py`
  - `AccountPosition` → 表 `account_positions`
  - `SymbolOhlcv` → 表 `symbol_ohlcv`

建表可依赖项目迁移或应用启动时执行上述 DDL（或由 ORM 根据模型生成）。
