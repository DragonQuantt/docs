# 策略 6: ML dollar-bar 方向预测 — dollar bar + LightGBM（SOTA Config A+++）

> 编写时间: 2026-06-13
> 状态: **设计稿（未实现）** —— 接入方式已定（self-feeding 流式插件 / 独立子账户 / `event_driven`），代码待建
> 研究来源: `ML_BT/scripts/final/`（入口 `run_pipeline.py`：soft-DP oracle 标签 + 289 因果特征 + 池化 LGBM-RF + walk-forward）
> 详细研究稿: backend 仓库 `docs/strategies/strategy1_ml_dollar_bar_lgbm.md`（逐阶段拆解）
> 部署: **账户 A · 栈 A · `execution_mode = event_driven`**（拓扑与命名空间见 [架构 §1.2.2](../architecture/index.md#account-stacks)）

---

## 一、策略定义

在**美元 bar**（按累计美元成交额切分，bar 时长可变）上，用 ML 模型预测每根 bar 的多空方向，逐币 bang-bang 持仓，1/N 等权组合 + 组合级回撤熔断。

| 维度 | 规则 |
|------|------|
| 宇宙 | 20 个大市值 USDT 永续（BTC ETH SOL ADA XRP BNB AVAX LINK DOGE DOT TRX LTC ATOM COMP UNI RUNE NEAR FIL AAVE GRT），顺序固定（绑定 `sym_id`） |
| bar | **美元 bar**，每币固定美元阈值（**= 训练数据 `dollar_bar_final` 同口径**），含 `large_buy_count` / `rsj_pos_preavg` / `rsj_neg_preavg` / buy-sell volume |
| 标签（仅训练） | soft-DP oracle：全序列前向-后向求每 bar 自由能差 `D`，`sign(D)` 为方向、`\|D\|/腿长` 为样本权重（**刻意 look-ahead，靠 walk-forward 控泄漏**） |
| 特征 | 289 列**因果**特征：chip logspaced 260 + time 7 + xfactors_rsj 13 + cand9 9 |
| 模型 | 池化 LightGBM-RF 二分类，shift ∈ {5,10,15} 三模型 ensemble，`sym_id` categorical；walk-forward 季度重训 |
| 仓位 | `desired = clip(20·(p_pos−0.5), ±1)`，hysteresis（\|desired−held\|>0.99 才换）→ 仓位钉在 ±1、换手极低 |
| 组合 | 1/N 等权，逐 bar log 复利；成本 5 bps/边（按 \|Δpos\|） |
| 熔断（CB） | 1/N 复利净值跌破峰值 90% → 全平 + 200 bar 冷却，冷却结束 peak 重置（用 t−1 收盘状态判，因果） |

## 二、运行机制（event_driven 流式插件，与服务衔接）

策略是一个**有状态、长驻、流式**的插件，封在 StrategyService[`event_driven`] 内（见 [架构 §1.3 / §2.2.7](../architecture/index.md)）：

```
启动: backfill 暖机(~2000 bar/币) → restore_state → attach market:trades 消费者组
每条 tick:  per-symbol 美元bar累加器.add(trade)
bar 完成(某币累计成交额过阈值):
  1. 更新该币 289 特征滑窗(尾部最长 ~2000 bar)
  2. 3×shift LGBM 推理 → 平均 → p_pos
  3. desired=clip(20·(p_pos−0.5),±1); hysteresis 更新该币 held
  4. 更新组合 CB(1/N 复利净值判回撤; 触发→全平+冷却)
  5. 组装全 20 币目标组合 → emit(StrategySignalBatch)
  6. snapshot_state 落 Redis
```

1. **数据来源**：以独立消费者组消费 DataService 写入的 `market:trades`（`trades_raw` 模式，带 `is_buyer_maker`，供算微观列）——**不自开 WebSocket**，保持 DataService「公共行情唯一所有者」不变量。
2. **高信号频率、低下单频率**：每根 bar 都 emit 全 20 币目标（满足契约「每批次 = 完整目标组合」），但 hysteresis(0.99) 让绝大多数 bar 不翻仓，下游 OrderService 对账户 A 净持仓差量后**大多数 bar 产生 0 单**。
3. **幂等**：`rebalance_id = rb-strat1-{bar 收盘时间戳}`，重处理同一根 bar → 同 id → 不重复下单（契约规则 #3 对 event_driven 的定义）。
4. **信号契约**：`signal_type = target_weight`，每腿 `target_weight = held / N × book_leverage`（CB 触发时全置 0）；下游 RiskService(A) 照常 `\|weight\| × equity_A` sizing——**契约一字不改**。

## 三、美元 bar 必须钉死研究口径（正确性，非优化）

模型的边际完全建立在「实盘 bar 与训练 bar 同口径」上，否则是喂 OOD 输入：

- 美元 bar **阈值 = 训练数据 `dollar_bar_final` 的每币固定阈值**，**不是** BarSourceAdapter 规划的 `auto_K50_ema` 自适应——阈值表随模型工件下发，不是独立运行时 config。
- bar 内产 `large_buy_count`（大单买笔数）/ `rsj_pos_preavg` / `rsj_neg_preavg` / buy-sell volume / dollar_volume，从 raw aggTrades 算（所以必须消费 `trades_raw`，不能是已聚合 OHLCV）。
- 因口径与模型绑定，**美元 bar 聚合放在策略插件内部，而非通用 BarSourceAdapter**——这也意味着设计文档里那套「通用 dollar_bar + tick_feature 流式链（V2）」**在本方案下不需要建**。

## 四、状态、冷启动与模型服务

| 状态 | 能否从数据重推 | 处理 |
|------|----------------|------|
| 美元 bar 进度 / 特征滑窗 | 能（replay 近期 trades） | 重启回放 `market:trades`（MAXLEN ~50 万足够近期）重建 |
| hysteresis `held` | 理论能（从 flat 点 replay p_pos），但窗口无界 | **持久化快照** `quant:acctA:strategy:strat1:state` |
| 组合 CB `peak/cum/cooldown` | 不能（依赖上线以来实时净值曲线） | **必须持久化** |

- **冷启动暖机**：289 特征最长窗口需每币 ~2000 bar；先从 Binance Vision / REST aggTrades **backfill** 建种子 bar + 暖特征，再 attach 实时流。这是策略 1 独有、不可省的成本（策略 2 是零冷启动）。
- **模型服务**：离线季度 walk-forward 重训（≈ 研究侧 `scripts/final` 的批处理），产出工件 = 3×shift LGBM + **每币美元 bar 阈值表** + **symbols 顺序（`sym_id` 映射）** + 特征列 + cutoff。插件加载当前 cutoff 模型，季度 rollover **热切换**（保留 held，不空仓）。⚠️ `sym_id` 是 categorical，实盘 symbols 顺序必须与训练**逐位一致**。

## 五、配置（`yamls/strategy.yaml`，栈 A 实例；草案）

```yaml
mode: single
active_strategy: ml_dollar_bar          # 策略注册名（待定）
execution_mode: event_driven            # 2026-06-13 新增形态
account: acctA                          # 键空间 quant:acctA:* + 子账户 API key
parameters:
  universe: [BTCUSDT, ETHUSDT, ...]     # 20 币，顺序绑定 sym_id（来自模型工件）
  model_artifact: "s3://.../models/active"   # 3×shift LGBM + 阈值表 + symbols 顺序
  shifts: [5, 10, 15]
  position_scale: 20.0                  # k
  position_cap: 1.0
  hysteresis_threshold: 0.99
  cost_bps: 5.0
  book_leverage: 1.0                    # held/N × 此系数 → target_weight（资本利用率）
  cb_stop_loss_pct: 0.10
  cb_cooldown_bars: 200
  warmup_bars: 2000                     # 冷启动每币暖机 bar 数
```

## 六、上线前待确认

- **美元 bar 阈值表**：必须取自训练数据生成口径，不能用独立自适应方案（§三）。
- **cand9 重建差异**：研究稿 cand9 为重建实现（原 `candidate_library.py` 已删），与历史 SOTA 产出非 bit 级一致——上线模型须用与实盘特征**同一份** cand9 实现重训。
- **`book_leverage`**：账户 A 的资本利用率 / gross，需与账户 B（策略 2）的资金分配一并定。
- **backfill 链路**：冷启动 2000 bar/币的 aggTrades 回填（Binance Vision / REST）须先跑通。
- **records 落库策略**：逐 bar emit 会让 `signal_records` 爆量 → 仅在目标**有效变更**时写（见 [database](../architecture/database.md)）。

## 七、与策略 2 / 现有服务的关系

- 与[策略 2](strategy_5.md) **分账户、分栈、互不踩仓**；共享只有 DataService（公共行情）+ Redis/DB/API/Alert（[架构 §1.2.2](../architecture/index.md#account-stacks)）。
- 复用下游 RiskService(A) / ExecutionService(A) / OrderService(A) / AccountService(A) —— 与策略 2 同一套代码、不同实例 + 不同子账户 + `quant:acctA:*` 键空间。
- ExecutionService(A) 逐腿校验（Phase A A8）对本策略是默认形态（单币更新 = 批次大小 1）。
- L2 状态视图为 `ml_composite`（模型时效 / 美元 bar 流健康 / 最近推理 / CB 状态），契约见 [HTTP API §5.3](../api/http.md)。
