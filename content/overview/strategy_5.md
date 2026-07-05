# 策略 5: monthly_majors_vs_alts — 月度大币-山寨轮动（vol targeting）

> 编写时间: 2026-06-11（同日由框架稿更新为实现稿）
> 状态: **已实现**（分支 `feat/majors_vs_alts_vol_target`，163 测试通过 + 真实行情冒烟验证）
> 来源: 研究脚本 `新建文件夹/scripts/eda_vol_target.py`
> 代码: `src/quant_trading/services/strategy_service/strategies/monthly_majors_vs_alts.py`

---

## 一、策略定义

做多大币、做空山寨篮子，叠加逐币硬止损与 NAV 波动率目标：

| 维度 | 规则 |
|------|------|
| 多头 | BTC **0.7** + ETH **0.3**（× leverage） |
| 空头 | 按**过去 30 日均美元成交额**排名的 top-N 山寨（默认 100，剔除 BTC/ETH 与稳定币本位），等权 **−1/N**（× leverage） |
| 敞口 | 毛 2.0 × leverage，净 0 |
| 调仓 | 自 `anchor_date` 起 **30 天固定网格**（非自然月） |
| 止损 | **每日收盘检查**：任一空头币自入场上涨超 **30%** → 当日平掉该腿，本期不再进入（权重不重分配，分母仍为原篮子大小） |
| 杠杆 | 每期期初 `leverage = target_vol / realised_vol`，截断 `[0.30, 2.50]`，期内固定；realised_vol = 策略自身日对数收益的 **SMA 30 日** std × √365 |

## 二、运行机制（与服务的衔接）

1. StrategyService 调度器**每日 00:10 UTC** 触发（日线收盘后）；策略内部判断当天是网格调仓日（输出全新组合）还是仅止损检查日（输出去掉已止损腿的组合）。
2. 策略为 **self_feeding**：触发时通过 ccxt REST 拉取主网日线（24h 成交额预筛 top `universe_prefilter_top` 后逐一取 1d K 线），**不依赖流式特征链**——消除冷启动，且与研究侧日线数据同源。
3. **无状态设计**：每次运行从数据全量重推网格、入场价、止损状态与杠杆（杠杆所需的策略历史日收益用 `vol_recon_days` 窗口按研究逻辑重建）。服务重启永不失同步，实盘口径 ≡ 回测口径。
4. 输出 `StrategySignalBatch`：`signal_type=target_weight`，每腿带 `target_weight`（带符号 NAV 占比），确定性 `rebalance_id = rb-monthly_majors_vs_alts-{触发时刻}`；metadata 含 `leverage / realised_vol / period_start / next_rebalance / is_rebalance_day / basket_size / stopped`。
5. 下游 RiskService（已实现）按 `|weight| × equity` 定名义，minNotional 剔腿、总敞口上限缩减。

## 三、配置（`yamls/strategy.yaml` 实际内容）

```yaml
mode: single
active_strategy: monthly_majors_vs_alts
rebalance_hour_utc: 0
rebalance_minute_utc: 10
parameters:
  anchor_date: "2026-01-01"     # 调仓网格锚点（上线前须定为真实首仓日）
  rebal_days: 30
  top_n: 100                    # 山寨篮子大小（研究默认）
  btc_weight: 0.7
  eth_weight: 0.3
  dvol_lookback_days: 30
  stop_pct: 0.30
  target_vol: 0.20              # 研究扫过 0.15 / 0.20 / 0.25 三档
  vol_window_days: 30
  lev_min: 0.30
  lev_max: 2.50
  min_candidates: 10            # 候选不足则整期空仓
  universe_prefilter_top: 200
  vol_recon_days: 150
```

## 四、与研究脚本的有意偏差

1. 实盘选篮**不要求期末收盘价存在**（研究中该过滤是回测记账便利，非信号信息）；杠杆所用的历史收益重建部分仍保留该过滤以对齐回测。
2. 日度美元成交额用 `close × volume` 近似（研究用逐笔成交额聚合）——仅影响排名边缘，可忽略。
3. 实盘波动率重建不含资金费率（研究脚本中 funding 为可选项）。

## 五、上线前待确认事项

- **`anchor_date`**：决定整个调仓网格，须定为真实首次建仓日。
- **`target_vol` 档位**：当前取中档 0.20，按回测结论调整。
- **`top_n=100` 与小资金冲突**：1000 USDT 本金下每条山寨腿约 10 USDT，贴近交易所 minNotional，Risk 剔腿会使篮子失真；小资金建议 top_n 降至 20–30。
- **XAU/XAG 等代币化贵金属**会按成交额进入做空篮子（研究宇宙定义如此）；如不想做空贵金属需加排除清单。

## 六、真实行情冒烟结果（2026-06-11）

```
leverage = 1.148（realised_vol 17.4%，target 20%）
BTC +0.8034 / ETH +0.3443 / 9 条山寨空腿各 −0.1148
stopped: LABUSDT @ 2026-06-01（入场后逆势 +30%+，止损正确触发）
period_start 2026-05-31, next_rebalance 2026-06-30
```
