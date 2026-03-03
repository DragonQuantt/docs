# 策略 1: rev_x_inv_vpin — 反转 × 逆 VPIN 连续调制

> 文档更新时间: 2026-03-03
> 策略定位: Top 1（T9）
> OOS 指标: Sharpe +3.33 | MDD -21.1% | Calmar 5.35（**回测结果**，实盘可能偏离）
> 组合配置: 10L10S, R1, same_day, fee=10bps
> **适用范围**: 本文档以模拟盘/回测为主；实盘上线需单独做风控、滑点与资金管理设计，并先经模拟盘验证。

---

## 1. 策略定位

`rev_x_inv_vpin` 的核心思想是:

1. 先用短周期反转构建基础信号（2h + 4h）。
2. 再用 `tick_vpin_24h` 做连续调制。
3. 当市场更“噪音主导”（VPIN 低）时放大反转信号。
4. 当市场更“知情交易主导”（VPIN 高）时压缩反转信号。

它和阈值过滤版（例如 `rev_vpin_filter_t1.0`）不同点在于: 调制函数是连续的，不是硬切断。

---

## 2. 输入数据与特征定义

策略在每个交易日（EOD=23:59 UTC）对每个 symbol 需要以下输入:

- `ret_2h[date][sym]`: `close(EOD) / close(EOD-2h) - 1`
- `ret_4h[date][sym]`: `close(EOD) / close(EOD-4h) - 1`
- `tick_vpin_24h[date][sym]`: EOD 前 24 小时内所有 bar 的 `tick_vpin` 均值

标准化约束（必须一致）:

- `window = 30`（时序 z-score）
- `ret` 因子 `negate=True`，`vpin` 因子 `negate=False`

---

## 3. 信号构造公式

### 3.1 反转基准信号

```text
z2h = zscore(ret_2h, window=30, negate=True)
z4h = zscore(ret_4h, window=30, negate=True)
baseline_rev = mean(valid(z2h, z4h))
```

### 3.2 VPIN 构建以及标准化
 1. 先算每根 bar 的 tick_vpin
     在 compute_vpin 里把 ticks 按“等成交量桶”切分，计算每个桶的
     |buy_vol - sell_vol| / (buy_vol + sell_vol)，最后取均值。
     参考代码:
```text
def compute_vpin(
      quantities: np.ndarray,
      is_buyer_maker: np.ndarray,
      n_buckets: int = 50,
  ) -> float:
      if len(quantities) < n_buckets * 2:
          return np.nan

      buy_vol = quantities.copy().astype(float)
      sell_vol = quantities.copy().astype(float)
      buy_vol[is_buyer_maker] = 0.0
      sell_vol[~is_buyer_maker] = 0.0

      total_vol = quantities.sum()
      if total_vol <= 0:
          return np.nan
      bucket_size = total_vol / n_buckets

      cum_vol = np.cumsum(quantities.astype(float))
      cum_buy = np.cumsum(buy_vol)
      cum_sell = np.cumsum(sell_vol)

      boundaries = np.arange(1, n_buckets + 1) * bucket_size
      bucket_idx = np.searchsorted(cum_vol, boundaries, side="right")
      bucket_idx = np.minimum(bucket_idx, len(quantities) - 1)

      vpin_vals = []
      prev_cum_buy = 0.0
      prev_cum_sell = 0.0
      prev_idx = 0
      for bi in bucket_idx:
          if bi <= prev_idx and len(vpin_vals) > 0:
              continue
          b_buy = cum_buy[bi] - prev_cum_buy
          b_sell = cum_sell[bi] - prev_cum_sell
          b_total = b_buy + b_sell
          if b_total > 0:
              vpin_vals.append(abs(b_buy - b_sell) / b_total)
          prev_cum_buy = cum_buy[bi]
          prev_cum_sell = cum_sell[bi]
          prev_idx = bi

      if not vpin_vals:
          return np.nan
      return float(np.mean(vpin_vals))
```
2. 再算 tick_vpin_24h[date][sym]
    在每天 EOD（23:59）先找最后一根 bar i1，再找 i1 往前 24h 的起点 i24，对区间 i24:i1+1 的 bar 级 tick_vpin 取
    nanmean。
    参考代码:
```text
  i1 = np.searchsorted(ts, g_eod, side="right") - 1
  if i1 < 0 or i1 >= len(cl) or ts[i1] < g_start:
      continue

  # Tick features: mean over trailing 24h window of bars
  i24 = np.searchsorted(ts, ts[i1] - H24, side="right") - 1
  if i24 < 0:
      i24 = 0
  win24 = slice(i24, i1 + 1)
  for tc in tick_cols:
      if sym in sym_tick_features[tc]:
          arr = sym_tick_features[tc][sym]
          vals = arr[win24]
          v = np.nanmean(vals)
          if np.isfinite(v):
              raw_features[f"{tc}_24h"][gd][sym] = v
```
3. 最后做 zscore
    vpin_z = zscore(tick_vpin_24h, window=30, negate=False)。


### 3.3 连续调制(信号层产出)

```text
multiplier = clip(1.0 - 0.3 * vpin_z, 0.3, 1.7)
signal = baseline_rev * multiplier
```

解释:

- `vpin_z = +2` -> `multiplier = 0.4` -> 压缩反转 60%
- `vpin_z = 0` -> `multiplier = 1.0` -> 不调制
- `vpin_z = -2` -> `multiplier = 1.6` -> 放大反转 60%

---

## 4. 调仓与持仓规则

调仓时点: 每日 `23:59 UTC`（same_day 模式）

组合构建:

1. 计算全部候选 symbol 的 `signal`。
2. 从大到小排序。
3. 做多 Top 10（每只 `+5%`）。
4. 做空 Bottom 10（每只 `-5%`）。
5. 组合总敞口: 多头 50% + 空头 50% = 100% dollar-neutral。

执行与费用:

- 使用差量下单（target - current）
- 扣除换手费 `10bps`(回测),也就是说，模拟盘/实盘里需要对手续费和滑点进行监控，合计单次交易不可以大于10bps
- 对最小下单量/最小名义金额做截断
