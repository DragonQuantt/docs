# Top 10 策略全链路逻辑文档

> 生成时间: 2025-02-14
> IS/OOS 分割: 2024-01-01 | Fee: 10bps | 数据源: Binance Futures Dollar Bars

## 全局排名总览

| 排名 | 策略 | NLS | Rebal | OOS Sharpe | OOS MDD | OOS Calmar | 实验 |
|:---:|------|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | rev_x_inv_vpin | 10L10S | R1 | +3.33 | -21.1% | 5.35 | T9 |
| 2 | rev_vpin_filter_t1.0 | 30L30S | R1 | +3.06 | -8.9% | 6.15 | T9 |
| 3 | rev_vpin_filter_t1.5 | 30L30S | R1 | +3.02 | -11.9% | 4.68 | T9 |
| 4 | rev_jump_filter_t1.0 | 10L10S | R1 | +2.71 | -15.9% | 5.67 | T9 |
| 5 | rev_jump_filter_t1.5 | 30L30S | R1 | +2.38 | -12.4% | 3.34 | T9 |
| 6 | rev_noise_boost | 30L30S | R1 | +2.26 | -12.0% | 3.36 | T9 |
| 7 | regime_switch | 30L30S | R1 | +2.21 | -16.2% | 2.59 | T9 |
| 8 | vol_weighted_blend | 10L10S | R1 | +2.20 | -25.8% | 3.26 | T8 |
| 9 | quad_blend | 30L30S | R1 | +1.99 | -16.8% | 2.42 | T8 |
| 10 | baseline_rev (2h+4h) | 30L30S | R1 | +1.95 | -15.7% | 2.47 | T8 |

---

## 共用数据管线 (所有策略通用)

### Stage 0: 原始数据采集

```
Binance Futures REST API
    → aggTrades (逐笔成交)
    → 字段: agg_trade_id, price, quantity, first_trade_id, last_trade_id, timestamp_ms, is_buyer_maker
    → 存储: E:\data\binance_futures\{SYMBOL}\{SYMBOL}-aggTrades-{YYYY-MM}.parquet
    → 频率: 每月一个文件, BTCUSDT ~7.4M ticks/月
```

### Stage 1: Dollar Bar 聚合

```
原始 tick → aggregate_bars() [src/crypto_data_engine/services/bar_aggregator/unified.py]
    → 按累计成交金额 (dollar_volume) 切分, 阈值由 auto_K50_ema 自适应
    → 每根 bar 产出 23 列:
      start_time, end_time, open, high, low, close,
      volume, buy_volume, sell_volume, vwap,
      tick_count, dollar_volume, price_std, volume_std,
      up_move_ratio, down_move_ratio, reversals,
      buy_sell_imbalance (OFI), max_trade_volume, max_trade_ratio,
      tick_interval_mean, path_efficiency, impact_density
    → 存储: E:\data\dollar_bar\bars\{SYMBOL}\{SYMBOL}_dollar_bar_auto_K50_ema_{YYYY-MM}.parquet
    → BTCUSDT ~2000 bars/月
```

### Stage 2: Tick 微观特征 Enrich

```
已有 bars + 原始 ticks → TickFeatureEnricher [src/.../tick_feature_enricher.py]
    → 对每根 bar, 取 rolling 50 bars 的 tick window
    → 计算 9 个 tick 特征 (tick_ 前缀):
      tick_vpin              — Volume-Synchronized Probability of Informed Trading
      tick_toxicity_run_mean — 毒性连续值均值
      tick_toxicity_run_max  — 毒性连续值最大
      tick_toxicity_ratio    — 毒性 bucket 比例
      tick_kyle_lambda       — Kyle's Lambda (价格冲击系数)
      tick_burstiness        — 交易时间聚集度
      tick_jump_ratio        — 价格跳跃比例
      tick_whale_imbalance   — 大单买卖不平衡
      tick_whale_impact      — 大单价格冲击
    → 输出 32 列 enriched bars
    → 存储: E:\data\dollar_bar\bars_enriched\{SYMBOL}\*.parquet
    → 488 symbols 全量 enriched
```

### Stage 3: 日频聚合

```
enriched bars → 按日历天 groupby("date") 聚合:
    open  = first(open)     high = max(high)      low = min(low)
    close = last(close)     volume = sum(volume)   dollar_volume = sum(dollar_volume)
    ofi   = mean(buy_sell_imbalance)               n_bars = count(close)
    tick_vpin = mean(tick_vpin)                     ... 其余 tick 特征同理取 mean
    ret   = close.pct_change()                     (日收益率)
```

### Stage 4: 日内特征提取 (intraday, 用于反转信号)

```
对每个 symbol, 将 bar 级时间戳转为 nanoseconds:
    ts = start_time.values.astype("datetime64[ns]").astype("int64")

对每个交易日 date:
    g_eod = date.value + 23h59m (ns)  — 当日 EOD 锚点
    i1 = searchsorted(ts, g_eod) - 1  — EOD 前最后一根 bar

    对每个 horizon H ∈ {1h, 2h, 4h, 8h}:
        i2 = searchsorted(ts, ts[i1] - H) - 1  — 往前推 H 时间
        ret_H = close[i1] / close[i2] - 1       — H 小时收益率

    对 tick 特征:
        取 [ts[i1] - 24h, ts[i1]] 范围内所有 bars
        tick_feature_24h = nanmean(tick_feature[window])
```

### Stage 5: Z-Score 标准化

```
对每个 symbol 单独做时序 z-score (window=30天):
    rolling_mean = mean(last 30 days of raw_value)  [shift(1) 避免 look-ahead]
    rolling_std  = std(last 30 days of raw_value)   [shift(1)]
    zscore = (raw_value - rolling_mean) / rolling_std

反转信号取负号 (negate=True): zscore = -zscore
    → 价格跌 (ret<0) → zscore 为正 → 做多信号
```

### Stage 6: 回测引擎

```
遍历所有交易日 all_dates:

    1. 计算当日 PnL (用旧权重):
       pnl = Σ(weight[sym] × ret[sym][date])  对所有持仓

    2. 换仓判断 (每 rebal_days 天):
       same_day 模式: sig_date = date       — 当日信号, 0h 延迟
       prev_day 模式: sig_date = date - 1   — 前日信号, 24h 延迟

       sig_day = signal_dict[sig_date]
       ranked = sort(sig_day, by=value)
       long  = ranked[-N_long:]   权重 = +1/(N_long+N_short) 每只
       short = ranked[:N_short]   权重 = -1/(N_long+N_short) 每只

    3. 扣除换手费:
       turnover = Σ|target_weight - old_weight| 对所有涉及的 symbol
       pnl -= turnover × FEE(0.001)

    4. 更新 NAV:
       nav[t] = nav[t-1] × (1 + pnl)
```

---

以下为每个策略的具体信号构造逻辑。

