# 策略 3: ofi_14d — 单因子动量（模拟盘实现规格）

> 文档更新时间: 2026-03-03  
> 策略定位: 单因子动量主策略  
> 目标口径: 与 `run_mom_step3_sweep.py` / `run_mom_step6_combo.py` 中 `ofi_14d` 腿一致  
> 研究指标（OOS）: Sharpe +2.40 | MDD -12.1% | AnnRet 58.8% | Calmar 4.84

---

## 1. 策略定义

`ofi_14d` 是截面动量多空策略:

1. 用 Dollar Bar 先聚合到日级 `ofi_daily`
2. 再做 14 日滚动均值得到 `ofi_14d`
3. 每 14 个交易日调仓一次（`R14`）
4. 在上月流动性 Top50 池中做 `15L15S`(流动性较好的币才有中期动量效应,历史不足 60 天标的不交易(资产池过滤))
5. 多空各 `50%`，组合美元中性

参数固定:

- `rebal = 14`
- `pool = T50`（上月池）
- `n_ls = 15`
- `fee = 10 bps`（次数据为回测数据,模拟盘中，费用+滑点单次费用不超10bps即可）

---

## 2. 字段契约

### 2.1 Tick 输入字段

| 字段 | 类型 | 含义 |
|---|---|---|
| `timestamp` | int(ms) | 成交时间 |
| `price` | float | 成交价 |
| `quantity` | float | 成交量 |
| `isBuyerMaker` / `is_buyer_maker` | bool | 主动方方向标记 |

### 2.2 Dollar Bar 字段（策略使用子集）

| 字段 | 聚合公式 |
|---|---|
| `open` | 首笔价格 |
| `high` | bar 内最高价 |
| `low` | bar 内最低价 |
| `close` | 末笔价格 |
| `volume` | \(\sum q_i\) |
| `buy_volume` | \(\sum q_i, isBuyerMaker=False\) |
| `sell_volume` | \(\sum q_i, isBuyerMaker=True\) |
| `dollar_volume` | \(\sum p_i q_i\) |
| `buy_sell_imbalance` | \(\frac{buy\_volume-sell\_volume}{volume}\) |

### 2.3 日级字段

| 字段 | 聚合公式 |
|---|---|
| `close_d` | 当日最后一根 bar 的 `close` |
| `ret_d` | \(\frac{close_d}{close_{d-1}} - 1\) |
| `ofi_d` | 当日 `buy_sell_imbalance` 的均值 |
| `dollar_volume_d` | 当日 `dollar_volume` 之和 |

仅保留历史长度 `>= 60` 天的 symbol。

---

## 3. 聚合与因子公式

### 3.1 Tick -> Dollar Bar 切分

设阈值 \(\theta\)（美元）:

\[
S_t=\sum_{i\in bar}p_iq_i
\]

当 \(S_t \ge \theta\) 时收口该 bar，下一笔 tick 开新 bar。

### 3.2 日级 OFI

\[
ofi_d = mean\_{bars\in day}(buy\_sell\_imbalance)
\]

### 3.3 因子定义

滚动均值:

\[
RM_w(x_t)=mean(x_{t-w+1:t})
\]

有效值条件:

\[
count_{valid}\ge \max(\lfloor w/2\rfloor,3)
\]

因子:

\[
ofi\_{14d}(t)=RM_{14}(ofi_d(t))
\]

---

## 4. 资产池与信号

### 4.1 月度流动性池（T50）

按月统计每个标的:

\[
DV_{m,sym}=\sum_{d\in m}dollar\_volume_{d,sym}
\]

降序取 Top50 形成 `T50`。  
交易日 \(t\) 使用 \(t\) 的上个月池（1 个月滞后）。

### 4.2 截面打分与选股

在调仓日 \(t\) 用 \(t-1\) 因子值（`prev_day`）:

\[
z_s=\frac{f_s-\mu_f}{\sigma_f}
\]

- Long: Top15
- Short: Bottom15

调仓跳过条件:

1. `len(signals) < 30`
2. \(\sigma_f < 1e^{-10}\)

---

## 5. 下单策略（模拟盘）

### 5.1 调仓触发

\[
di>0 \land di \bmod 14 = 0
\]

其中 `di` 为交易日索引（从 0 开始）。

### 5.2 目标权重

每个调仓日:

- 多头每只: \(+0.5/15\)
- 空头每只: \(-0.5/15\)

### 5.3 差量下单

对每个标的 \(s\):

\[
\Delta w_s = w_s^{new}-w_s^{old}
\]

\[
target\_notional_s = NAV_t \cdot w_s^{new}
\]

\[
order\_qty_s=\frac{target\_notional_s-current\_notional_s}{price_s}
\]

- \(order\_qty_s>0\): 买入  
- \(order\_qty_s<0\): 卖出  

### 5.4 费用与 NAV 更新

先按旧仓位计当日收益:

\[
pnl_t=\sum_s w_s^{old}\cdot ret_{s,t}
\]

\[
NAV_t^{pre}=NAV_{t-1}\cdot(1+pnl_t)
\]

调仓时扣费:

\[
turnover_t=\sum_s |w_s^{new}-w_s^{old}|
\]

\[
NAV_t=NAV_t^{pre}\cdot(1-0.001\cdot turnover_t)
\]

非调仓日:

\[
NAV_t=NAV_t^{pre}
\]

---

## 6. 风控逻辑

### 6.1 必须保留的硬约束

2. 上月流动性池滞后
3. 信号不足 30 只时不调仓
4. 截面标准差过小不调仓
5. 多空固定 `+50%/-50%`（美元中性）
6. 历史不足 60 天标的不交易(资产池过滤)

### 6.2 执行层建议约束

1. 小单过滤: \(|\Delta w|<0.001\) 可忽略
2. 反向换仓先平后开
3. 现金不足按可成交最大数量缩单
4. 手续费与滑点分开记账

---

