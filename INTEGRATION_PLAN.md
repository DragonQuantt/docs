# 模拟盘数据持久化 & 回测一致性验证 实施计划

> **核心目标**: quant_trading_backend 持久化模拟盘运行数据，使 CDE 回测框架能消费同一份数据，验证模拟盘与回测结果的一致性。
>
> **架构原则**: CDE 独立 repo、独立部署，不嵌入 backend。两者通过**数据格式契约**（Parquet 文件 + DB 表）衔接。

---

## 现状分析

### quant_trading_backend 已有的持久化

| 数据类型 | 存储 | 状态 |
|---------|------|------|
| 账户元数据 | PostgreSQL `accounts` | ✅ 正常 |
| Portfolio NAV 快照 | PostgreSQL `portfolio_snapshots` + Redis | ✅ 正常 |
| 市场 Trades | Redis Stream `market:trades` (循环 50 万条) | ⚠️ 无长期存储，滚动丢弃 |
| 市场 Tickers | Redis Stream `market:tickers` | ⚠️ 同上 |
| 市场 OHLCV | Redis Stream `market:ohlcv` | ⚠️ 同上 |
| 持仓变动事件 | PostgreSQL `position_events` 表 **已建但从未写入** | ❌ 空表 |
| 特征计算 | FeatureService **TODO 未实现** | ❌ 缺失 |
| 交易信号 | Channel 已定义，从未发布 | ❌ 缺失 |
| Bar 聚合 | 无 | ❌ 缺失 |

### CDE 回测框架需要的数据格式

| 数据 | 格式 | 说明 |
|------|------|------|
| Tick/Trade 数据 | Parquet, 列: `timestamp, price, quantity, isBuyerMaker` | 按 `{symbol}/{YYYY-MM}.parquet` 组织 |
| Bar 数据 | Parquet, 列: `start_time, end_time, open, high, low, close, volume, buy_volume, sell_volume, vwap, dollar_volume, tick_count` + 可选高级特征 | 按 `{bar_type}/bars/{symbol}/{YYYY-MM}.parquet` |
| 特征数据 | 追加到 Bar DataFrame, 或独立 Parquet | 26+ 因子列 (return_N, momentum_N, VPIN 等) |
| 资产池 Profile | Parquet, 列: `date, dollar_volume` | `{symbol}_daily_profile.parquet` |
| 回测输入 | MultiIndex DataFrame `(timestamp, asset)` with features | 1h 对齐 (dollar bar) 或 5min 对齐 (time bar) |

### 核心差距

**backend 的 market trades 数据在 Redis Stream 中滚动丢弃，从未落盘为 Parquet。这意味着模拟盘运行期间的原始市场数据无法被 CDE 回测框架重放。**

---

## Phase 1: 数据持久化补全 (预计 2-3 周)

> 补齐 backend 缺失的持久化能力，使运行数据以 CDE 兼容格式落盘。

### Step 1.1 — 市场数据落盘 (Redis Stream → Parquet)

> **状态**: 并入 backend Phase A 实施。

**目标**: 将 Redis Stream 中滚动丢弃的市场数据定期落盘为 Parquet，优先保住**不可回填**的数据。

**Phase A 修订**:

1. **服务更名**: ~~TradeArchiveService~~ → **ArchiveService**。
2. **落盘对象改为 orderbook 快照 + BarNormalized**:
   - `market:orderbook` 快照 — 历史 L2 订单簿无公开数据源，**不可回填，最高优先**；
   - `BarNormalized` 标准化 bar — 用于与回测的一致性对账、服务重启后的预热。
3. **aggTrades 不再实时落盘**: 可随时从 data.binance.vision 免费回填（backtest 项目的 crypto_data_engine 已具备下载能力），实时归档冗余。

**具体任务**【Phase A 计划】:

1. **创建 `ArchiveService(BaseEventService)`**:
   - `service_name = "archive_service"`
   - 不订阅 Pub/Sub channel，自驱动定时循环
   - 核心逻辑:
     - 每隔 `flush_interval`（默认 60s）从 Redis Stream（`market:orderbook` 及 bar 数据源）读取新条目（用 consumer group, 保证不重复消费不丢数据）
     - 按 symbol 分组，追加写入 Parquet 文件
     - 文件组织: `{data_dir}/orderbook/{symbol}/{YYYY-MM}.parquet`、`{data_dir}/bars/{symbol}/{YYYY-MM}.parquet`
     - 使用 PyArrow 增量写入（append to existing parquet 或 按天/小时分片后月末合并）

2. **Consumer Group 管理**:
   - 启动时在源 Stream（`market:orderbook` 等）上创建 consumer group `archive`
   - 记录消费位点 (Redis Stream ID)，服务重启后从断点继续
   - ACK 已成功写入 Parquet 的消息

3. **创建配置**:
   - `ArchiveConfig`（按 yaml_loader 约定对应 `yamls/archive.yaml`）:
     - `data_dir`: Parquet 输出根目录（默认 `./data`）
     - `flush_interval`: 刷盘间隔秒数（默认 60）
     - `buffer_max_size`: 内存缓冲最大条数（默认 100_000）

4. **创建 ServiceSpec + CLI 命令**:
   - `main archive-service run`

5. **数据完整性**:
   - 启动时检查 Parquet 文件最后时间戳，与 Redis Stream 最早可用数据对比，记录 gap 警告
   - 定期输出统计日志: 已归档条数、文件大小、symbol 数量

**涉及文件 (新建)**（具体以 Phase A 实现为准）:
- `services/archive_service/__init__.py`
- `services/archive_service/service.py`
- `services/archive_service/parquet_writer.py` — 增量 Parquet 写入封装
- `common/configs/yamls/services/archive_yaml.py`
- `controller/gateway/specs.py` — 添加 archive_spec (无 HTTP 路由，纯后台)
- `app/commands/archive_service/__init__.py`

**涉及文件 (修改)**:
- `app/cli.py` — 注册命令
- `docker/docker-compose.yml` — 添加容器

**验收标准**:
- DataService（orderbook 模式）运行 10 分钟后，`data/orderbook/{symbol}/` 下有 Parquet 文件
- bar 归档与 Redis `quant:bar:normalized:*` 快照字段一致，可与回测 bar 逐行对账
- 服务重启后从断点续写，不丢不重
- Tick 级数据不在本服务范围，需要时从 data.binance.vision 回填

---

### Step 1.2 — Position Events 写入

**目标**: 激活 `position_events` 表，记录每一次持仓变动（成交、调仓、资金费率），作为模拟盘交易流水。

**具体任务**:

1. **在 AccountService 中添加 position event 检测与写入**:
   - 对比前后两次 `positions` 快照，检测变化:
     - 新增持仓 → `event_type = "open"`
     - 数量变化 → `event_type = "fill"` (含 `qty_delta`, `fill_price`)
     - 持仓消失 → `event_type = "close"`
   - 每次检测到变化时，调用 `PositionEventRepository.insert()` 写入 DB
   - 从 ccxt position 数据中提取: symbol, side, contracts_delta, entry_price, fee (如有)

2. **资金费率记录**:
   - 如果使用永续合约，定时查询 funding fee
   - 写入 `event_type = "funding"` 的 position_event

3. **添加查询 API**:
   - `GET /api/v1/account/position/events` — 查询持仓变动历史
     - Query: symbol, event_type, start_time, end_time, page, page_size
   - `GET /api/v1/account/position/events/export` — 导出为 CSV

**涉及文件 (修改)**:
- `services/account_service/service.py` — 添加 position diff 检测 + DB 写入逻辑
- `controller/routes/account/position.py` — 添加 events 查询路由

**涉及文件 (新建)**:
- `services/account_service/position_differ.py` — 持仓变动检测逻辑封装

**验收标准**:
- 持仓发生变化时 `position_events` 表有对应记录
- API 查询返回正确的事件历史

---

### Step 1.3 — 信号 & 调仓记录持久化

> **状态**: 并入 backend Phase A 实施 —— `signal_records` / `rebalance_records` / `orders` 三表随 Phase A 落地；表结构权威定义合并至 docs 的 `architecture/database.md`，下文字段列表仅作背景参考。

**目标**: 记录模拟盘的每一次信号生成和调仓决策，以便与回测对比。

**具体任务**:

1. **创建 DB 模型 `SignalRecordModel`** (`signal_records` 表):
   - `id` (PK)
   - `account_id` (FK)
   - `timestamp` (DateTime, 信号生成时间)
   - `strategy_name` (String)
   - `signal_data` (JSONB) — 完整信号: `{symbol: weight/strength}` 映射
   - `metadata` (JSONB, nullable) — 策略参数快照、排名数据等
   - `created_at`

2. **创建 DB 模型 `RebalanceRecordModel`** (`rebalance_records` 表):
   - `id` (PK)
   - `account_id` (FK)
   - `signal_record_id` (FK → signal_records, nullable)
   - `timestamp` (DateTime)
   - `before_weights` (JSONB) — 调仓前持仓权重
   - `target_weights` (JSONB) — 目标权重
   - `after_weights` (JSONB, nullable) — 调仓后实际权重
   - `orders` (JSONB) — 提交的订单列表
   - `execution_summary` (JSONB, nullable) — 成交汇总: 滑点、手续费、偏差
   - `created_at`

3. **在信号生成/调仓流程中写入**:
   - 信号生成后 → 写 `signal_records`
   - 调仓执行后 → 写 `rebalance_records`
   - （具体插入点取决于 backend 的策略执行流程，目前可能尚未实现，此处先建表和 Repository）

4. **创建 Repository + API**:
   - `SignalRecordRepository`
   - `RebalanceRecordRepository`
   - `GET /api/v1/signals/history` — 历史信号
   - `GET /api/v1/rebalances/history` — 历史调仓

**涉及文件 (新建)**:
- `data/db/models/signal_record.py`
- `data/db/models/rebalance_record.py`
- `data/db/repositories/signal_record_repository.py`
- `data/db/repositories/rebalance_record_repository.py`
- `controller/routes/signals/__init__.py`
- `controller/routes/signals/routes.py`
- `controller/routes/rebalances/__init__.py`
- `controller/routes/rebalances/routes.py`

**涉及文件 (修改)**:
- `data/db/models/__init__.py` — 注册新模型

**验收标准**:
- 表创建成功，Repository CRUD 单元测试通过
- API 可查询（初始为空，待策略执行流程接入）

---

### Step 1.4 — Daily Profile 生成

> 适配说明: Step 1.1 修订后 aggTrades 不再实时落盘，Tick 数据按需回填（data.binance.vision）后再生成 profile，或直接采用下方第 2 点方案从 tickers 聚合。

**目标**: 从已持久化的 trade 数据生成每日 dollar volume profile，供 CDE 动态资产池选择使用。

**具体任务**:

1. **创建定时任务** (在 TradeArchiveService 中或独立 cron):
   - 每日 UTC 00:05 触发
   - 读取前一天所有 symbol 的 Parquet trade 数据
   - 计算: `daily_dollar_volume = sum(price × quantity)` per symbol
   - 追加到 `{data_dir}/profiles/{symbol}_daily_profile.parquet`
   - 格式与 CDE `load_asset_pool_from_profiles()` 期望一致:
     - 列: `date (datetime), dollar_volume (float64)`

2. **或者**: 从 `market:tickers` Redis Stream 直接聚合（ticker 有 volume 信息），避免依赖 Parquet 回读

**涉及文件 (新建)**:
- `services/trade_archive_service/profile_generator.py`

**涉及文件 (修改)**:
- `services/trade_archive_service/service.py` — 添加定时 profile 生成

**验收标准**:
- `profiles/` 目录下有 daily profile Parquet 文件
- CDE 的 `load_asset_pool_from_profiles()` 能直接读取

---

### Step 1.5 — 前端: 数据归档状态监控

> 适配说明: 归档服务已更名 ArchiveService（见 Step 1.1 修订），监控对象相应为 orderbook / bar 归档状态。

**目标**: 在前端 System 页面添加数据归档服务的状态展示。

**具体任务**:

1. **后端 API**:
   - `GET /api/v1/trade-archive/status` — 返回:
     - 归档服务运行状态
     - 已归档 symbol 数量
     - 今日归档条数
     - 最后归档时间
     - 各 symbol 数据覆盖范围 (最早日期 ~ 最新日期)
     - 磁盘占用

2. **前端**: 在 `/system` 页面添加 "数据归档" card:
   - 服务状态指示灯
   - 统计信息展示
   - 数据覆盖范围时间线

**涉及文件 (新建, 后端)**:
- `controller/routes/trade_archive/__init__.py`
- `controller/routes/trade_archive/routes.py`

**涉及文件 (修改, 前端)**:
- `src/features/system/api.ts` — 添加 fetchArchiveStatus
- `src/features/system/types.ts` — 添加 ArchiveStatus 类型
- `src/features/system/views/SystemView.vue` — 添加归档状态 card

**验收标准**: System 页面能看到归档服务状态和数据覆盖情况

---

### Step 1.6 — 集成测试

> 适配说明: aggTrades 不再实时落盘（见 Step 1.1 修订），下方 Trade Parquet 相关测试改用按需回填的 Tick 数据。

**目标**: 验证 Phase 1 所有持久化数据可被 CDE 直接消费。

**具体任务**:

1. **Trade Parquet 兼容性测试**:
   - 用 backend 归档的 Parquet 文件，调用 CDE 的 `aggregate_bars()` 生成 dollar bar
   - 验证输出 bar 的列和格式正确

2. **Daily Profile 兼容性测试**:
   - 用 backend 生成的 profile 文件，调用 CDE 的 `load_asset_pool_from_profiles()`
   - 验证返回的 symbol 列表合理

3. **Position Events 对比**:
   - 模拟盘 position_events 的 fill 记录与 CDE 回测的 TradeRecord 格式映射验证

4. **Signal Records 对比**:
   - signal_records 的 signal_data 格式与 CDE SignalOutput 的映射验证

**涉及文件 (新建)**:
- `tests/integration/test_parquet_compat.py`
- `tests/integration/test_profile_compat.py`
- `tests/integration/test_data_format_mapping.py`

**验收标准**: CDE 能直接读取 backend 产出的所有数据文件，无需转换

---

## Phase 2: 一致性验证工具 (预计 2-3 周)

> 在 CDE 和 backend 之间建立一致性验证流程:
> 用模拟盘同期的市场数据跑回测，对比回测结果与模拟盘实际结果。

### Step 2.1 — CDE 侧: 模拟盘数据适配器

> 适配说明: Phase A 后 backend 直接归档 BarNormalized，tick Parquet 不再实时产出（按需从 data.binance.vision 回填）。

**目标**: 在 CDE 中创建 DataLoader，能直接读取 backend 持久化的数据。

**具体任务**:

1. **创建 `LiveDataLoader(IDataLoader)`** 在 CDE repo 中:
   - `load_bars()`:
     - 读取 backend 归档的 tick Parquet (`{data_dir}/trades/{symbol}/{YYYY-MM}.parquet`)
     - 调用 CDE 自身的 `aggregate_bars()` 生成 bar
     - 或直接读取已聚合的 bar 文件（如果 backend 也做了聚合）
   - `load_features()`:
     - 调用 CDE 的 `UnifiedFeatureCalculator.calculate()`
   - 配置: `data_dir` 指向 backend 的归档目录（可以是网络挂载或共享目录）

2. **创建 `LiveTradeLoader`**:
   - 从 backend PostgreSQL 读取 `position_events` 表
   - 转换为 CDE 的 `TradeRecord` 格式
   - 从 backend PostgreSQL 读取 `signal_records` 表
   - 转换为 CDE 的 `SignalOutput` 格式

3. **配置文件**:
   - 新增 `data/config/live_data_source.yaml`:
     ```yaml
     backend_data_dir: "/path/to/backend/data/trades"
     backend_db_url: "postgresql://..."
     ```

**涉及文件 (在 CDE repo 新建)**:
- `src/crypto_data_engine/services/data_loader/live_data_loader.py`
- `src/crypto_data_engine/services/data_loader/live_trade_loader.py`
- `data/config/config_templates/live_data_source.yaml`

**验收标准**: CDE 能通过 LiveDataLoader 读取 backend 数据并运行回测

---

### Step 2.2 — CDE 侧: 一致性对比引擎

**目标**: 创建工具自动对比模拟盘实际结果与回测结果。

**具体任务**:

1. **创建 `ConsistencyAnalyzer`** 类:
   ```python
   class ConsistencyAnalyzer:
       def compare(
           self,
           live_nav: pd.Series,           # 模拟盘 NAV 时间序列 (from portfolio_snapshots)
           backtest_nav: pd.Series,        # 回测 NAV 时间序列 (from BacktestResult)
           live_trades: List[TradeRecord],  # 模拟盘成交 (from position_events)
           backtest_trades: List[TradeRecord],  # 回测成交
           live_signals: List[SignalOutput],    # 模拟盘信号 (from signal_records)
           backtest_signals: List[SignalOutput], # 回测信号
       ) -> ConsistencyReport
   ```

2. **ConsistencyReport 内容**:

   a. **NAV 对比**:
      - NAV 相关系数 (Pearson, Spearman)
      - NAV 最大偏差 (绝对值, 百分比)
      - NAV 偏差时间序列
      - 累计收益差异

   b. **信号对比**:
      - 信号一致率 (同一时间点，多少 symbol 的方向一致)
      - 权重偏差统计 (MAE, RMSE)
      - 不一致信号的 case 列表

   c. **交易对比**:
      - 交易笔数差异
      - 对应交易的价格偏差 (实际成交 vs 回测模拟)
      - 滑点分析: 实际滑点 vs 回测假设滑点
      - 手续费对比

   d. **归因分析**:
      - NAV 偏差分解: 信号差异贡献 vs 执行差异贡献 vs 费用差异贡献
      - 识别最大偏差来源

3. **报告输出**:
   - JSON 格式 (供前端消费)
   - 可选 HTML/PDF 报告
   - Matplotlib 图表 (NAV 双线对比, 偏差分布等)

**涉及文件 (在 CDE repo 新建)**:
- `src/crypto_data_engine/services/consistency/__init__.py`
- `src/crypto_data_engine/services/consistency/analyzer.py`
- `src/crypto_data_engine/services/consistency/report.py`
- `src/crypto_data_engine/services/consistency/metrics.py`

**验收标准**: 给定一段模拟盘数据 + 同期回测结果，能输出完整一致性报告

---

### Step 2.3 — CDE 侧: 一致性验证 CLI & API

**目标**: 提供便捷的命令行和 API 接口运行一致性验证。

**具体任务**:

1. **CLI 命令** (Typer):
   ```bash
   # 一键验证: 读取 backend 数据 → 跑回测 → 对比
   cde consistency run \
     --backend-data-dir /path/to/backend/data \
     --backend-db-url postgresql://... \
     --strategy momentum_20 \
     --start-date 2025-01-01 \
     --end-date 2025-03-01 \
     --output-dir ./consistency_reports/

   # 仅对比 (已有回测结果)
   cde consistency compare \
     --live-nav-csv ./live_nav.csv \
     --backtest-result ./backtest_result.json \
     --output-dir ./consistency_reports/
   ```

2. **API 端点** (CDE 的 FastAPI):
   - `POST /api/v1/consistency/run` — 提交一致性验证任务
   - `GET  /api/v1/consistency/{task_id}/status` — 查询进度
   - `GET  /api/v1/consistency/{task_id}/report` — 获取报告

**涉及文件 (在 CDE repo 新建)**:
- `src/crypto_data_engine/app/commands/consistency.py` — CLI
- `src/crypto_data_engine/api/routers/consistency.py` — API

**验收标准**: 一条命令即可完成"读取 backend 数据 → 回测 → 对比 → 输出报告"

---

### Step 2.4 — Backend 侧: 一致性验证 Proxy API

**目标**: 在 quant_trading_backend 中添加 proxy API，前端无需直接对接 CDE。

**具体任务**:

1. **创建 API 路由** `controller/routes/consistency/`:
   - `POST /api/v1/consistency/run` — 转发到 CDE API，或直接调用 CDE CLI
   - `GET  /api/v1/consistency/{task_id}/status`
   - `GET  /api/v1/consistency/{task_id}/report`
   - `GET  /api/v1/consistency/history` — 历史验证记录

2. **方案选择**:
   - **方案 A (推荐)**: Backend 通过 httpx 调用 CDE 的 REST API（CDE 独立部署）
   - **方案 B**: Backend 通过 subprocess 调用 CDE CLI（如果 CDE 和 backend 在同一机器）
   - **方案 C**: Backend 直接 import CDE 包调用（如果不想独立部署 CDE API）

3. **创建配置**:
   - `ConsistencyYamlConfig`:
     - `cde_api_url`: CDE 服务地址（方案 A）
     - `cde_cli_path`: CDE CLI 路径（方案 B）

**涉及文件 (新建)**:
- `controller/routes/consistency/__init__.py`
- `controller/routes/consistency/routes.py`
- `common/configs/yamls/services/consistency_yaml.py`

**验收标准**: 前端通过 backend API 即可触发一致性验证，无需知道 CDE 的存在

---

### Step 2.5 — 前端: 一致性验证页面

**目标**: 在前端添加一致性验证的配置、触发和结果展示。

**具体任务**:

1. **创建 `consistency` feature 模块**:
   ```
   src/features/consistency/
   ├── api.ts
   ├── types.ts
   ├── store.ts
   └── views/
       ├── ConsistencyView.vue    — 主视图
       ├── RunConfigPanel.vue     — 验证配置: 策略、日期范围、参数
       └── ReportView.vue         — 报告展示
   ```

2. **ReportView 可视化**:

   a. **NAV 双线对比图** (ECharts):
      - 模拟盘 NAV (蓝线) vs 回测 NAV (橙线)
      - 偏差区域填充 (灰色)
      - DataZoom 交互

   b. **偏差分析图**:
      - NAV 偏差百分比时间序列
      - 偏差分布直方图

   c. **信号一致性表**:
      - 按时间点列出: 一致 symbol 数 / 总数, 不一致 symbol 详情

   d. **交易对比表**:
      - 对应交易: 模拟盘价格 vs 回测价格, 滑点差异

   e. **KPI 汇总**:
      - NAV 相关系数, 最大偏差, 信号一致率, 平均滑点差

3. **路由**: `/consistency`

**涉及文件 (新建)**:
- `src/features/consistency/api.ts`
- `src/features/consistency/types.ts`
- `src/features/consistency/store.ts`
- `src/features/consistency/views/ConsistencyView.vue`
- `src/features/consistency/views/RunConfigPanel.vue`
- `src/features/consistency/views/ReportView.vue`
- `src/features/consistency/components/NavCompareChart.vue`
- `src/features/consistency/components/DeviationChart.vue`
- `src/features/consistency/components/SignalMatchTable.vue`
- `src/features/consistency/components/TradeCompareTable.vue`

**涉及文件 (修改)**:
- `src/app/router/index.ts` — 添加路由
- 侧边栏 — 添加"一致性验证"入口

**验收标准**: 能从前端触发验证、查看 NAV 对比图表和完整报告

---

### Step 2.6 — 集成测试

> 适配说明: 端到端链路起点为按需回填的 Tick 数据或 backend 归档的 bar（见 Step 1.1 修订）。

**具体任务**:

1. **模拟场景测试**:
   - 用已有的模拟盘历史数据（portfolio_snapshots + mock trades）
   - 用同期 tick 数据跑 CDE 回测
   - 验证 ConsistencyAnalyzer 输出合理

2. **格式兼容性端到端**:
   - Backend 归档 trades → CDE 读取 → 聚合 bar → 计算特征 → 跑回测 → 对比 → 前端展示
   - 完整链路无报错

3. **边界条件**:
   - 模拟盘数据不足一个月
   - 部分 symbol 缺失数据
   - 策略参数不完全匹配

**验收标准**: 端到端流程跑通，一致性报告结果合理

---

## 依赖关系

```
Phase 1:
  Step 1.1 (Trade落盘) ──→ Step 1.4 (Daily Profile)
       │                          │
       └──→ Step 1.6 (集成测试) ←─┘
  Step 1.2 (Position Events) ──→ Step 1.6
  Step 1.3 (Signal/Rebalance Records) ──→ Step 1.6
  Step 1.5 (前端监控) — 独立，可并行

Phase 2:
  Step 2.1 (CDE DataLoader) ──→ Step 2.2 (对比引擎) ──→ Step 2.3 (CLI/API)
                                                              │
                                                     Step 2.4 (Backend Proxy)
                                                              │
                                                     Step 2.5 (前端页面)
                                                              │
                                                     Step 2.6 (集成测试)
```

## Sub-Agent 分配建议

| Agent | 负责 Steps | 说明 |
|-------|-----------|------|
| Agent A (Backend-Persistence) | 1.1, 1.2, 1.4 | Trade 落盘 + Position Events + Profile |
| Agent B (Backend-Records) | 1.3 | Signal/Rebalance 表 + Repository + API |
| Agent C (Frontend-Monitor) | 1.5 | System 页面归档状态 |
| Agent D (CDE-Consistency) | 2.1, 2.2, 2.3 | CDE 侧全部工作 |
| Agent E (Backend-Proxy) | 2.4 | Backend proxy API |
| Agent F (Frontend-Consistency) | 2.5 | 一致性验证前端 |
| Agent G (Testing) | 1.6, 2.6 | 两阶段集成测试 |

Agent A + B + C 可并行启动；Agent D 依赖 Phase 1 完成；Agent E/F 依赖 Agent D。
