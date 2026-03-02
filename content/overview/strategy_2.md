# 策略 #10: baseline_rev — 2h+4h 反转基准

> OOS Sharpe: **+1.95** | OOS MDD: -15.7% | Calmar: 2.47
> 配置: 30L30S, R1, same_day | 实验: T8

## 策略思想

最简单的反转策略 — 只用 2h 和 4h 两个频率的价格收益 z-score 等权混合。
这是所有 T8/T9 策略的**基准线**，其他策略都是在此基础上叠加 tick 特征或多频率混合。

**这个策略证明了一个核心结论**: 2h-4h 短期价格反转是加密货币 dollar bar 上
唯一稳健的横截面 alpha 源。所有其他因子（OFI、tick微观、量价背离等）
作为独立因子都不如它。

## 信号构造

```
1. 对每个 symbol, 每个交易日:
   ret_2h = close(EOD) / close(EOD - 2h) - 1
   ret_4h = close(EOD) / close(EOD - 4h) - 1

2. 时序 z-score 标准化 + 取负号:
   z2h = -zscore(ret_2h, window=30)   // 跌了的 → 正信号 → 做多
   z4h = -zscore(ret_4h, window=30)

3. 混合:
   signal = mean(z2h, z4h)   // 两个都有效时取均值, 只有一个时直接用
```

### 下单逻辑

```
每日 23:59 UTC:
    1. 对所有 488 symbols 计算 signal
    2. 排序: signal 最大的 30 只做多, 最小的 30 只做空
    3. 每只等权 ±1.67% (总多头 50%, 总空头 50%)
    4. 市价单执行, 扣除 10bps 换手费
```

## 完整时间线 (单次交易日)

```
T = 某个交易日 (例如 2024-06-15)

00:00 UTC  ← 日历天开始
   ...     ← dollar bars 持续产生 (volume-clock)
21:59 UTC  ← EOD - 2h: 取此刻的 close 价格作为 close(EOD-2h)
19:59 UTC  ← EOD - 4h: 取此刻的 close 价格作为 close(EOD-4h)
23:59 UTC  ← EOD: 取此刻的 close 价格作为 close(EOD)

           ret_2h = close(23:59) / close(21:59) - 1
           ret_4h = close(23:59) / close(19:59) - 1

           计算 zscore, 生成 signal
           排序, 确定目标仓位
           执行换仓 (市价单, ~1分钟内完成)

T+1 00:00  ← 新仓位开始持有, 赚取 T+1 的日收益
T+1 23:59  ← 再次计算信号, 再次换仓
```

## 为什么是 same_day 而非 prev_day

```
prev_day (延迟 1 天):
    T-1 23:59 计算信号 → T 00:00 开始持有 → T+1 00:00 PnL 结算
    信号延迟 ~24h → 2h 信号已过期 → Sharpe 从 +1.95 暴跌到 -1.72

same_day (无延迟):
    T 23:59 计算信号 → T+1 00:00 开始持有 → T+1 23:59 PnL 结算
    信号延迟 ~1min → 2h 信号仍然新鲜 → Sharpe +1.95 ✅

same_day 在加密市场可行: 7×24 交易, 无收盘/开盘间隔
```

## 源代码

- T8 版: `scripts/run_expT8_reversal_signals.py` — 信号 "baseline_2h4h"
- T9 版: `scripts/run_expT9_tick_regime_factors.py` — 信号 "baseline_rev"
