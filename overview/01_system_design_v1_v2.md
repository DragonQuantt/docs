# 系统设计方案 — V1 / V2 迭代规划

> 编写时间: 2026-02-21
> 基于: 00_overview.md 策略研究成果 + 现有代码架构 Review

---

## 现状分析 (V0 — 当前已实现)

### 已完成

| 模块 | 实现状态 | 说明 |
|------|---------|------|
| Asset Pool Service | ✅ 完成 | 按 30 日 USDT 成交额筛选 Top-100 合约, 周期性更新 |
| Aggregator Service | ✅ 完成 | CCXT Pro WebSocket 接入, time-bar 聚合 (1m OHLCV), cross-section 触发 |
| Feature Service | ✅ 完成 | SMA / EMA / Volatility / Momentum / Volume Ratio, lookback = 7/14/30 |
| Futures Order Service | ✅ 完成 | 简单动量反转策略, 5L5S, 周度换仓, market order |
| Redis 事件总线 | ✅ 完成 | BaseEventService 抽象, Pub/Sub 四阶段链路 |
| 基础设施 | ✅ 完成 | Docker Compose / AWS ECS Fargate / GitHub Actions CI/CD |

### 核心差距 (V0 vs 目标系统)

| 维度 | V0 现状 | 目标 (00_overview) |
|------|---------|-------------------|
| Bar 类型 | Time Bar (1m) | **Dollar Bar** (自适应阈值 auto_K50_ema) |
| 特征体系 | 基础技术指标 (6 个) | **Tick 微观结构** (VPIN, Kyle's Lambda 等 9 个) + 日内特征 (1h/2h/4h/8h ret) |
| 信号构造 | 单因子 momentum | **多因子复合** (reversal × VPIN, regime switch 等 10 套) |
| 标准化 | 无 | **Z-Score** (30 日 rolling, shift(1) 防未来泄漏) |
| 回测 | 无 | **完整回测引擎** (IS/OOS, turnover fee, NAV 曲线) |
| 风控 | 无 | 需要仓位控制 / 止损 / 最大回撤限制 |
| 账户管理 | 无 (CLI stub) | 实时余额 / 仓位 / 挂单监控 |
| 监控告警 | 无 | 需要服务健康检查 / 异常告警 |

---

## V1 — 基础完善 + 回测闭环

### 目标

> **让系统从"能跑"升级为"可验证、可监控、可安全运行"的量化交易平台。**
> 核心交付: 回测引擎 + 风控层 + 账户服务 + 监控, 策略仍使用简化版本。

### V1 微服务拆分

```
┌──────────────────────────────────────────────────────────────────┐
│                        V1 服务全景                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Asset Pool Service]     — 保持现状, 小幅优化                    │
│         │                                                        │
│         ▼ asset_pool_updated                                     │
│  [Aggregator Service]     — 保持现状 (time-bar)                   │
│         │                                                        │
│         ▼ kline_aggregated                                       │
│  [Feature Service]        — 扩展: 加入 Z-Score + 日内 ret 特征    │
│         │                                                        │
│         ▼ feature_calculated                                     │
│  [Strategy Service] ★新   — 策略与下单解耦, 输出目标仓位           │
│         │                                                        │
│         ▼ signal_generated                                       │
│  [Risk Service] ★新       — 风控检查, 仓位约束, 通过后 emit        │
│         │                                                        │
│         ▼ order_approved                                         │
│  [Order Service] ★重构    — 纯执行层: 拆单/下单/重试               │
│         │                                                        │
│         ▼ order_executed                                         │
│  [Account Service] ★新    — 实时同步余额/仓位/挂单                 │
│                                                                  │
│  [Backtest Engine] ★新    — 离线: 信号回放 + NAV 计算              │
│  [Monitor Service] ★新    — 健康检查 + 异常告警                    │
│  [API Gateway]            — 重构: 暴露 REST API 供前端/告警使用    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### V1 各模块详细设计

#### 1. Strategy Service (新增)

**职责:** 将"策略逻辑"从 `FuturesOrderService` 中抽离, 使策略可插拔。

```python
# 核心抽象
class BaseStrategy(ABC):
    @abstractmethod
    def generate_signal(self, features: Dict[str, Dict]) -> TargetPortfolio:
        """输入特征, 输出目标仓位权重"""

class TargetPortfolio:
    positions: Dict[str, TargetPosition]  # symbol -> {side, weight, reason}
    signal_timestamp: datetime
    strategy_name: str
```

**V1 内置策略:**
- `MomentumReversalStrategy` — 从现有 FuturesOrderService 迁移
- `BaselineRevStrategy` — 00_overview 中排名第 10 的 baseline_rev (2h+4h), 作为对照组

**关键设计:**
- 策略只输出目标权重 (e.g. `{"BTC/USDT:USDT": +0.1, "ETH/USDT:USDT": -0.1}`), 不涉及下单
- 通过 YAML 配置切换策略: `strategy: momentum_reversal`
- 策略通过 `signal_generated` 事件将目标仓位传给 Risk Service

#### 2. Risk Service (新增)

**职责:** 在策略信号到达 Order Service 前进行风控检查。

**V1 风控规则:**

| 规则 | 说明 | 默认值 |
|------|------|--------|
| max_position_pct | 单币种最大仓位占比 | 15% |
| max_total_exposure | 最大总敞口 (多头+空头绝对值之和) | 100% |
| max_drawdown_halt | 触发暂停交易的最大回撤 | 30% |
| min_balance_reserve | 最低保留余额 (USDT) | 100 |
| max_order_value | 单笔订单最大金额 | 5000 USDT |
| blacklist | 禁止交易的币种 | [] |

**事件流:**
```
signal_generated → Risk Service 检查 →
  通过 → emit order_approved
  拒绝 → emit risk_rejected + 告警
```

#### 3. Order Service (重构)

**职责:** 从现有的"策略+下单一体"重构为纯执行层。

**V1 改进:**
- 接收 `order_approved` 事件中的目标仓位, 计算差量订单 (当前仓位 vs 目标仓位)
- 增加下单重试机制 (指数退避, 最多 3 次)
- 增加 amount 精度处理 (按交易所 precision 规则 round)
- 支持 dry-run 模式 (记录但不实际下单)
- 下单结果通过 `order_executed` 事件广播

#### 4. Account Service (新增)

**职责:** 统一管理账户状态, 供其他服务查询。

**实现方式:**
- 定时轮询 (30s 间隔) Binance REST API 获取:
  - 账户余额 (`fetch_balance`)
  - 当前持仓 (`fetch_positions`)
  - 活跃挂单 (`fetch_open_orders`)
- 数据缓存到 Redis:
  - `quant:account:balance` — JSON
  - `quant:account:positions` — JSON
  - `quant:account:orders` — JSON
- 其他服务读 Redis 即可, 无需各自调交易所 API
- 余额/仓位变化时 emit `account_updated` 事件

#### 5. Feature Service (扩展)

**V1 新增特征:**

| 特征 | 计算方式 | 来源 |
|------|---------|------|
| ret_1h, ret_2h, ret_4h, ret_8h | 日内不同窗口的收益率 | 00_overview Stage 4 |
| zscore_momentum_N | momentum 的 30 日 rolling z-score | 00_overview Stage 5 |
| zscore_volatility_N | volatility 的 30 日 rolling z-score | 00_overview Stage 5 |

**关键:** Z-Score 计算时 `shift(1)` 避免 look-ahead bias。

#### 6. Backtest Engine (新增)

**职责:** 离线回测, 验证策略有效性。

**设计:**
```
┌───────────────┐    ┌──────────────┐    ┌──────────────┐
│  Historical   │───▶│   Strategy   │───▶│   Backtest   │
│  Data Loader  │    │  (same code) │    │   Engine     │
└───────────────┘    └──────────────┘    └──────────────┘
                                               │
                                    ┌──────────┴──────────┐
                                    │   Report Generator  │
                                    │  Sharpe / MDD /     │
                                    │  Calmar / Turnover  │
                                    └─────────────────────┘
```

**核心原则:** 策略代码在回测和实盘中完全复用 (BaseStrategy 接口统一)。

**V1 回测功能:**
- 数据源: 从 Redis 或 CSV/Parquet 加载历史 kline
- IS/OOS 分割: 支持按日期切分
- 费用模型: 固定手续费率 (默认 10bps)
- 换手计算: `turnover = Σ|target_weight - old_weight|`
- 输出指标: Sharpe, MDD, Calmar, 年化收益率, 换手率, NAV 曲线
- CLI 入口: `python -m quant_trading.app.cli backtest --strategy momentum_reversal --start 2024-01-01`

#### 7. Monitor Service (新增)

**V1 监控能力:**

| 监控项 | 方式 | 告警渠道 |
|--------|------|---------|
| 服务存活 | 各服务定期向 Redis 写 heartbeat | Redis TTL 过期 → 告警 |
| 数据延迟 | 检查最新 kline 时间戳 vs 当前时间 | 超过阈值 → 告警 |
| 异常仓位 | 对比 target vs actual positions | 偏差过大 → 告警 |
| 资金变动 | 余额大幅下降检测 | 下降超 5% → 告警 |
| 策略指标 | 实时 PnL / 回撤 / 仓位比 | 定期报告 |

**告警渠道 (V1):** 日志 + 简易 Webhook (Telegram / DingTalk)。

#### 8. API Gateway (重构)

**V1 暴露的 REST API:**

```
GET  /api/v1/health                   — 全局健康状态
GET  /api/v1/account/balance          — 账户余额
GET  /api/v1/account/positions        — 当前持仓
GET  /api/v1/asset-pool               — 资产池列表
GET  /api/v1/features/{symbol}        — 最新特征
GET  /api/v1/strategy/status          — 策略运行状态
POST /api/v1/strategy/rebalance       — 手动触发换仓
GET  /api/v1/monitor/heartbeats       — 各服务心跳
```

### V1 Redis 事件流 (完整)

```
asset_pool_updated
    → kline_aggregated
        → feature_calculated
            → signal_generated        ★ 新
                → order_approved      ★ 新 (Risk 通过)
                   or risk_rejected   ★ 新 (Risk 拒绝)
                    → order_executed  ★ 新
                        → account_updated  ★ 新
```

### V1 配置文件新增

```
configs/
├── strategy_config.yaml       ★ 策略选择 + 参数
├── risk_config.yaml           ★ 风控规则
├── account_config.yaml        ★ 账户轮询间隔
├── monitor_config.yaml        ★ 告警阈值 + 渠道
├── backtest_config.yaml       ★ 回测参数
├── order_config.yaml          (重构: 移除策略参数, 保留执行参数)
├── asset_pool_config.yaml     (保持)
├── aggregator_config.yaml     (保持)
├── feature_config.yaml        (扩展: 新增 z-score 配置)
├── exchange_config.yaml       (保持)
└── db_config.yaml             (保持)
```

### V1 注意要点

1. **策略与执行解耦是最高优先级** — 现有 `FuturesOrderService` 将策略逻辑和下单逻辑混在一起, 无法独立测试策略, 也无法复用到回测。拆分后策略代码可同时用于实盘和回测。

2. **回测与实盘共用策略代码** — `BaseStrategy.generate_signal()` 的输入输出必须完全一致。回测引擎模拟 Feature Service 的输出格式, 策略无需感知自己在回测还是实盘。

3. **Risk Service 必须在 Order Service 之前** — 任何下单请求必须经过风控审批。这是资金安全的底线。

4. **Account Service 避免 API 调用竞争** — 统一由 Account Service 调用交易所 API, 其他服务从 Redis 读取。避免多个服务同时调用交易所 API 导致限频。

5. **Z-Score shift(1) 防泄漏** — Feature Service 中计算 Z-Score 时, rolling mean/std 必须 `shift(1)`, 即只用截至前一根 bar 的数据。这在回测中尤为关键。

6. **渐进迁移** — V1 不替换聚合方式 (仍用 time-bar), 不引入 tick 微观特征。先用现有数据管线验证回测闭环和风控的正确性。

7. **测试覆盖** — 策略 `generate_signal` 必须有单测; 风控规则必须有单测; 回测引擎必须与手动计算结果比对。

---

## V2 — Dollar Bar + Tick 微观特征 + 高级策略

### 目标

> **实现 00_overview.md 中描述的完整数据管线和策略体系, 达到研究阶段验证过的回测效果。**
> 核心交付: Dollar Bar 聚合 + Tick 微观特征 + Top 10 策略上线 + 多策略并行。

### V2 微服务拆分

```
┌──────────────────────────────────────────────────────────────────┐
│                        V2 服务全景                               │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [Data Ingestion Service] ★新                                    │
│       │  — 采集 aggTrades 逐笔成交                               │
│       │  — 持久化到 TimescaleDB / Parquet                        │
│       ▼                                                          │
│  [Dollar Bar Service] ★新                                        │
│       │  — 自适应阈值 (auto_K50_ema) dollar bar 聚合              │
│       │  — 输出 23 列 enriched bar                               │
│       ▼ dollar_bar_generated                                     │
│  [Tick Feature Service] ★新                                      │
│       │  — VPIN / Kyle's Lambda / Burstiness 等 9 个 tick 特征   │
│       │  — rolling 50 bars window                                │
│       ▼ tick_features_enriched                                   │
│  [Feature Service] ★扩展                                         │
│       │  — 日内特征 (1h/2h/4h/8h ret)                           │
│       │  — Z-Score 标准化                                        │
│       │  — 聚合 tick 特征 + 传统特征                              │
│       ▼ feature_calculated                                       │
│  [Strategy Service] ★扩展                                        │
│       │  — 多策略并行 (rev_x_inv_vpin, regime_switch 等)          │
│       │  — 策略组合/集成信号                                      │
│       ▼ signal_generated                                         │
│  [Risk Service] ★增强                                            │
│       │  — 动态风控 (波动率自适应仓位)                             │
│       │  — 策略间相关性检查                                       │
│       ▼ order_approved                                           │
│  [Order Service] ★增强                                           │
│       │  — Limit Order 支持                                      │
│       │  — TWAP/VWAP 拆单算法                                    │
│       ▼ order_executed                                           │
│  [Account Service]        — V1 基础上增加 PnL 归因                │
│  [Backtest Engine] ★增强  — Dollar Bar 回测 + 策略组合回测         │
│  [Monitor Service] ★增强  — 策略级监控 + 风控仪表盘                │
│  [API Gateway] ★增强      — WebSocket 推送 + 策略管理 API         │
│                                                                  │
│  [Asset Pool Service]     — 增加多交易所支持                      │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### V2 各模块详细设计

#### 1. Data Ingestion Service (新增)

**职责:** 采集交易所逐笔成交数据 (aggTrades), 这是 Dollar Bar 和 Tick 特征的原始输入。

**设计:**
```
Binance Futures WebSocket (aggTrade stream)
    → 内存缓冲 (按 symbol 分桶)
    → 批量写入 TimescaleDB hypertable: aggTrades_{exchange}
    → 同时推送到 Redis Stream (供 Dollar Bar Service 实时消费)
```

**存储 schema:**
```sql
CREATE TABLE aggTrades (
    time         TIMESTAMPTZ NOT NULL,
    symbol       TEXT NOT NULL,
    agg_trade_id BIGINT,
    price        DOUBLE PRECISION,
    quantity     DOUBLE PRECISION,
    is_buyer_maker BOOLEAN
);
SELECT create_hypertable('aggTrades', 'time');
```

**关键:** 这是数据量最大的服务。488 symbols × ~7M ticks/month (以 BTC 为例)。需要:
- 批量写入 (每 1000 条或每 1 秒 flush)
- TimescaleDB 压缩策略 (7 天后自动压缩)
- Redis Stream 限制长度 (MAXLEN ~100000), 仅保留最近数据供实时消费

#### 2. Dollar Bar Service (新增)

**职责:** 将逐笔成交聚合为 Dollar Bar, 对应 00_overview Stage 1。

**核心算法:**
```
累计 dollar_volume += price × quantity
当 dollar_volume >= threshold:
    关闭当前 bar, 输出 23 列
    重置累计器
    threshold = auto_K50_ema(近期 bar 的 dollar_volume 中位数)
```

**输出 23 列 (对应 00_overview):**
```
start_time, end_time, open, high, low, close,
volume, buy_volume, sell_volume, vwap,
tick_count, dollar_volume, price_std, volume_std,
up_move_ratio, down_move_ratio, reversals,
buy_sell_imbalance (OFI), max_trade_volume, max_trade_ratio,
tick_interval_mean, path_efficiency, impact_density
```

**事件:** bar 关闭时 emit `dollar_bar_generated`

#### 3. Tick Feature Service (新增)

**职责:** 对应 00_overview Stage 2 — Tick 微观特征 Enrich。

**计算的 9 个特征:**

| 特征名 | 含义 | 计算方式 |
|--------|------|---------|
| tick_vpin | 知情交易概率 | Volume-Synchronized PIN |
| tick_toxicity_run_mean | 毒性连续值均值 | 连续同方向 trade 的 run length 均值 |
| tick_toxicity_run_max | 毒性连续值最大 | 同上取 max |
| tick_toxicity_ratio | 毒性 bucket 比例 | 毒性 bucket 占总 bucket 比例 |
| tick_kyle_lambda | 价格冲击系数 | 回归: ΔP = λ × signed_volume |
| tick_burstiness | 交易时间聚集度 | 到达间隔的 CV (变异系数) |
| tick_jump_ratio | 价格跳跃比例 | |ΔP| > threshold 的 tick 占比 |
| tick_whale_imbalance | 大单买卖不平衡 | 大单 buy_vol - 大单 sell_vol |
| tick_whale_impact | 大单价格冲击 | 大单前后价格变化均值 |

**窗口:** rolling 50 bars 的 tick window。

**实现方式:** 订阅 `dollar_bar_generated`, 从 Redis Stream/TimescaleDB 读取对应窗口的 raw ticks, 计算特征后附加到 bar 上。

#### 4. Feature Service (V2 扩展)

**在 V1 基础上新增:**
- 接收 `tick_features_enriched` 事件 (而非仅 `kline_aggregated`)
- 日频聚合 (00_overview Stage 3): 将 bar 级数据按日聚合
- 日内多 horizon 收益率 (00_overview Stage 4): ret_1h, ret_2h, ret_4h, ret_8h
- Z-Score 支持 negate 模式: 反转信号取负
- 输出融合 tick 特征 + 传统特征的完整特征向量

#### 5. Strategy Service (V2 扩展 — 多策略并行)

**V2 新增策略 (对应 00_overview Top 10):**

| 策略 | 信号构造 | 推荐 NLS |
|------|---------|----------|
| rev_x_inv_vpin | zscore_rev_2h × (1 / zscore_vpin) | 10L10S |
| rev_vpin_filter_t1.0 | zscore_rev_2h, 仅保留 zscore_vpin > 1.0 | 30L30S |
| rev_vpin_filter_t1.5 | zscore_rev_2h, 仅保留 zscore_vpin > 1.5 | 30L30S |
| rev_jump_filter_t1.0 | zscore_rev_2h, 仅保留 zscore_jump > 1.0 | 10L10S |
| rev_jump_filter_t1.5 | zscore_rev_2h, 仅保留 zscore_jump > 1.5 | 30L30S |
| rev_noise_boost | zscore_rev_2h × (1 + zscore_noise) | 30L30S |
| regime_switch | 根据 vol 分位切换: 高波做趋势, 低波做反转 | 30L30S |
| vol_weighted_blend | 多信号按波动率加权融合 | 10L10S |
| quad_blend | 四因子等权混合 | 30L30S |
| baseline_rev | zscore_rev_2h + zscore_rev_4h 简单叠加 | 30L30S |

**多策略编排:**
```yaml
# strategy_config.yaml (V2)
strategies:
  - name: rev_x_inv_vpin
    weight: 0.3
    nls: 10L10S
  - name: rev_vpin_filter_t1.0
    weight: 0.3
    nls: 30L30S
  - name: regime_switch
    weight: 0.4
    nls: 30L30S

ensemble_method: weighted_average  # 加权平均目标仓位
```

#### 6. Risk Service (V2 增强)

**新增规则:**

| 规则 | 说明 |
|------|------|
| 波动率自适应仓位 | 高波动环境自动缩减仓位 |
| 策略间相关性检查 | 当多策略信号高度一致时发出预警 |
| 动态止损 | 单仓位浮亏超 N 倍 ATR 自动平仓 |
| 杠杆自适应 | 根据组合波动率调整杠杆 (目标 vol = 15%) |

#### 7. Order Service (V2 增强)

- **Limit Order 支持:** 在流动性好的品种上使用 limit order 降低滑点
- **TWAP 拆单:** 大单拆分为多笔小单, 按时间均匀执行
- **智能路由:** 根据 order book depth 选择 market/limit

### V2 数据流 (完整)

```
[Data Ingestion] ──aggTrade──▶ TimescaleDB + Redis Stream
                                      │
                                      ▼
[Dollar Bar Service] ──dollar_bar_generated──▶ Redis
                                                │
                                                ▼
[Tick Feature Service] ──tick_features_enriched──▶ Redis
                                                     │
[Asset Pool Service] ──asset_pool_updated──┐         │
                                           ▼         ▼
                    [Feature Service] ◀─── 融合 ───────┘
                           │
                           ▼ feature_calculated
                    [Strategy Service] (多策略并行)
                           │
                           ▼ signal_generated
                    [Risk Service]
                           │
                           ▼ order_approved
                    [Order Service]
                           │
                           ▼ order_executed
                    [Account Service]
```

### V2 注意要点

1. **数据量级飞跃** — aggTrade 数据量比 kline 大 2-3 个数量级。TimescaleDB 必须启用压缩和分区策略, Redis Stream 必须限制长度。考虑引入 Kafka 替代 Redis Stream 处理高吞吐场景。

2. **Dollar Bar 自适应阈值的冷启动** — 系统启动时没有历史 bar, 无法计算 EMA 阈值。解决方案: 预加载最近 N 天的 aggTrade 数据, 先离线生成种子 bar 确定初始阈值。

3. **Tick Feature 计算开销** — VPIN、Kyle's Lambda 等需要逐笔 tick 数据, rolling 50 bars window 可能涉及数万条 tick。需要:
   - 增量计算 (滑动窗口, 不重复扫描)
   - 考虑用 NumPy/Polars 向量化, 避免 Python 循环

4. **多策略权重调优** — V2 支持多策略并行, 但策略权重 (ensemble weight) 需要定期根据近期表现调整。初期可用固定权重, 后续考虑引入在线学习 (如 EWA)。

5. **回测与实盘的 Dollar Bar 一致性** — 离线回测的 Dollar Bar (从 Parquet 批量计算) 和实盘 (从 WebSocket 逐笔喂入) 必须产出完全一致的结果。需要端到端对比测试。

6. **Redis → Kafka 迁移的时机** — 如果 488 symbols 全量接入 aggTrade, Redis Pub/Sub 可能成为瓶颈。V2 可考虑:
   - 高频数据链路 (aggTrade → Dollar Bar): 用 Kafka / Redis Stream
   - 低频事件链路 (feature_calculated → signal 等): 保持 Redis Pub/Sub

7. **渐进上线** — V2 策略应先在回测中验证 OOS 表现, 再切入 paper trading (dry-run), 最后才投入实盘。建议同时运行 V1 策略 (momentum_reversal) 和 V2 策略, 对比实盘效果。

---

## 迭代路线图

```
V0 (当前)                    V1                           V2
──────────                ──────────                  ──────────
✅ 基础管线               策略/执行解耦                 aggTrade 采集
✅ 简单策略               风控服务                     Dollar Bar 聚合
✅ Redis Pub/Sub          账户服务                     Tick 微观特征
✅ Docker/ECS             回测引擎                     10 套策略上线
                          监控告警                     多策略集成
                          Z-Score 特征                 动态风控
                          API Gateway                  Limit/TWAP 下单
                          测试覆盖                     Kafka (可选)
                          ─────────                    ─────────
                          ~6-8 周                      ~10-14 周
```

### 依赖关系

```
V1-1: Strategy Service 重构 (无外部依赖, 最先开始)
V1-2: Risk Service (依赖 V1-1 的 TargetPortfolio 接口)
V1-3: Order Service 重构 (依赖 V1-2 的 order_approved 事件)
V1-4: Account Service (独立, 可与 V1-1 并行)
V1-5: Feature Service 扩展 (独立, 可与 V1-1 并行)
V1-6: Backtest Engine (依赖 V1-1 和 V1-5)
V1-7: Monitor Service (依赖 V1-4, 可在最后集成)
V1-8: API Gateway 重构 (依赖所有服务就绪)

V2-1: Data Ingestion Service (独立, 可提前启动)
V2-2: Dollar Bar Service (依赖 V2-1)
V2-3: Tick Feature Service (依赖 V2-2)
V2-4: Feature Service V2 扩展 (依赖 V2-3)
V2-5: 策略实现 (依赖 V2-4)
V2-6: 多策略集成 + 风控增强 (依赖 V2-5)
```

---

## 技术选型建议

| 决策点 | V1 推荐 | V2 推荐 | 理由 |
|--------|---------|---------|------|
| 消息总线 | Redis Pub/Sub | Redis Pub/Sub + Kafka (高频链路) | V1 数据量可控; V2 aggTrade 量级需要 Kafka |
| 时序数据库 | TimescaleDB | TimescaleDB | 已部署, 够用 |
| 回测框架 | 自研 (轻量) | 自研 (扩展) | 与实盘策略接口一致更重要, 不需要第三方框架 |
| 告警渠道 | Webhook (Telegram) | Webhook + Grafana 仪表盘 | V1 轻量; V2 需要可视化 |
| 配置管理 | YAML + Pydantic | 同上 | 现有方案够用 |
| 计算加速 | — | NumPy / Polars | Tick 特征计算需要向量化 |
