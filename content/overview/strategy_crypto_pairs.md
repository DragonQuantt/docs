# 策略: crypto_pairs_mean_reversion

> 来源: `crypto_1min_research` pairs mean-reversion 研究流程  
> 部署状态: dry-run ready，接入 Strategy → Risk → Execution 检查链路

## 策略定义

`crypto_pairs_mean_reversion` 是事件驱动的 pair spread 均值回归策略。Pair 选择与参数估计仍由离线研究流程完成，后端只加载 frozen picks 与 formation segments，在实时 1m bar 到达后维护每个 pair 的状态机。

每个 pair 使用以下冻结参数:

- `freq`: pair 的交易频率，如 `15min`、`1h`、`4h`
- `hedge_ratio`: OLS 对冲比例 beta
- `intercept`、`spread_mean`、`spread_std`: spread 标准化参数
- `is_tradable`: rolling cointegration gate 结果
- `z_entry`、`z_exit`、`z_stop`: entry、止盈、硬止损阈值

## 运行机制

1. `FeatureService` 在每根标准化 bar 后发布 `feature_calculated`，payload 内包含 `close`。
2. `StrategyService` 将 1m close 按 pair 的 `freq` 收口。
3. 当 Y/X 两腿同一 freq bar 都收口后，计算:

```text
spread = log(Y) - beta * log(X) - intercept
z = (spread - spread_mean) / spread_std
```

4. 状态机与研究包保持一致:

```text
segment 变化且 flatten_on_segment_change=true -> flat
gate 关闭 -> flat
z 无效 -> flat
持仓且 |z| >= z_stop -> flat
持仓且 bars_in_trade >= max_bars_in_trade -> flat
long spread 且 z >= -z_exit -> flat
short spread 且 z <= z_exit -> flat
空仓且 z <= -z_entry -> long spread
空仓且 z >= z_entry -> short spread
```

## 信号契约

策略输出仍是 `StrategySignalBatch`，但每条腿带 `metadata`:

```json
{
  "symbol": "BTC/USDT",
  "side": "long",
  "signal_value": 1.0,
  "reason": "crypto_pairs_1h:BTCUSDT-ETHUSDT_y",
  "metadata": {
    "strategy_type": "pair_spread",
    "pair_id": "1h:BTCUSDT-ETHUSDT",
    "leg": "y",
    "target_state": 1,
    "hedge_ratio": 1.25,
    "leg_notional_multiplier": 1.0,
    "freq": "1h"
  }
}
```

`RiskService` 识别 `strategy_type=pair_spread` 后做 pair sizing: Y 腿使用 `pair_base_notional_usdt`，X 腿使用 `pair_base_notional_usdt * abs(beta)`。真实下单不在本策略首版部署范围内。

## 导入流程

从研究产物生成后端配置:

```bash
main strategy-service import-crypto-pairs \
  --scan-dir ../crypto_1min_research/artifacts/pairs_scan \
  --top-n 15
```

该命令会写入:

- `yamls/crypto_pairs_strategy.yaml`
- `yamls/strategy.yaml`
- `yamls/asset_pool.yaml`

