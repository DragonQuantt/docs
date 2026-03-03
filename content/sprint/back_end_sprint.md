# 工业级量化后端 - 业务核心逻辑迭代计划（模拟盘优先）

> 编写时间: 2026-03-03
> 当前阶段: **模拟盘（dry-run / paper trading）**
> 契约基线: `../api/http.md`, `../api/websocket.md`, `../api/redis.md`
> 策略基线: V1 = `../overview/strategy_2.md`（baseline_rev）; V2 = `../overview/strategy_1.md` + `../overview/strategy_3.md`

---

## 1. 目标与优先级

当前版本以"先能跑通业务闭环"为第一目标，不以"功能最全"或"架构最优雅"为第一目标。

优先级:

1. **P0 跑通优先**: 先打通最短交易闭环
2. **P1 稳定性其次**: 保障恢复能力、幂等性和可观测
3. **P2 回测兼容再次之**: 逐步统一回测与模拟盘口径

非目标（本轮不强求）:

- 实盘真资金上线
- 全量多交易所并行
- 一步到位的回测/模拟盘完全一致

---

## 2. 核心业务闭环

MVP 必须跑通的事件链:

```
Data -> Feature -> Strategy -> Risk -> Order(dry-run) -> Account

feature_calculated
  -> signal_generated
  -> order_approved / risk_rejected
  -> order_executed
  -> account_updated
```

核心不变量:

1. 任何订单请求必须先过 RiskService
2. 风控拒绝后不得继续下单
3. 同一业务请求不得重复下单（幂等）

---

## 3. Sprint 总览

| 版本 | Sprint | 优先级 | 建议工期 | 目标摘要 | 对齐 API |
|------|--------|--------|---------|---------|---------|
| **V1** | V1 Sprint 1 | P0 | 1 周 | 最短闭环跑通（单策略、dry-run） | Redis 事件链 |
| **V1** | V1 Sprint 2 | P0 | 1 周 | 一键启动 + 最小可观测 | HTTP V1 P0 + WS 4 端点 |
| **V1** | V1 Sprint 3 | P1 | 1.5 周 | 恢复机制与幂等保障 | 内部质量，无新 API |
| **V1** | V1 Sprint 4 | P1 | 1.5 周 | 风控前置与执行可靠性 | HTTP V1 P1 + WS alerts |
| **V1** | V1 Sprint 5 | P2 | 1 周 | 策略接口统一 + 信号分析 | HTTP V1 P2 (strategy, signals) |
| **V1** | V1 Sprint 6 | P2 | 1.5 周 | 回测与模拟盘口径对齐 | HTTP V1 P2 (backtest) |
| **V2** | V2 Sprint 1 | V2 | 2 周 | 双源 Bar + ofi_14d + 资产池 | HTTP V2 (asset-pool) + Redis V2 事件 |
| **V2** | V2 Sprint 2 | V2 | 2 周 | Tick 特征 + 多策略并行 | HTTP V2 (strategy 管理, signals/compare) + WS signals |

---

## 4. V1 Sprint 详细计划

> V1 覆盖 strategy_2 (baseline_rev): 30L30S, R1, same_day, 2h+4h 反转基准。
> 数据链路: AssetPool → DirectKline → Feature → Strategy → Risk → Order → Account。

---

### V1 Sprint 1: 最短闭环跑通 (P0)

> 目标: 在模拟盘环境下跑通完整交易链路，不追求完美稳定性。

#### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| 策略最小实现 | `BaselineRevStrategy` | 单策略模式，`generate_signal()` 实现 |
| 事件链打通 | Strategy/Risk/Order/Account 服务联通 | 覆盖主路径和拒绝路径 |
| dry-run 执行 | OrderService dry-run 输出 | 不触发真实下单 |
| 基础配置 | `strategy_config.yaml`, `risk_config.yaml`, `order_config.yaml` | 最小参数集 |

#### 对齐 API 端点

**Redis 事件 (redis.md V1)**:

| 通道 | 说明 |
|------|------|
| `quant:asset_pool_updated` | 资产池更新 → 触发数据拉取 |
| `quant:kline_aggregated` | K 线聚合 → 触发特征计算 |
| `quant:feature_calculated` | 特征就绪 → 触发策略信号 |
| `quant:signal_generated` | 信号生成 → 风控审核 |
| `quant:order_approved` | 风控通过 → 下单执行 |
| `quant:risk_rejected` | 风控拒绝 → 告警 |
| `quant:order_executed` | 订单执行 → 账户更新 |
| `quant:order_failed` | 订单失败 → 告警 |
| `quant:account_updated` | 账户同步完成 |

**Redis 键**: `quant:asset_pool:{exchange}`, `quant:features:latest:{symbol}`, `quant:signal:latest`, `quant:account:*`

#### 验收标准

- [ ] 100 次回放中，核心事件链完整率 >= 95%
- [ ] `risk_rejected` 分支可触发且不下单
- [ ] `order_executed`、`account_updated` 事件可见

---

### V1 Sprint 2: 一键启动与最小可观测 (P0)

> 目标: 新环境可快速启动，故障可快速定位。

#### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| 一键启动 | `run-all-services` 稳定可用 | 本地联调入口 |
| 健康接口 | `GET /api/v1/health` | 服务状态总览 |
| 管线接口 | `GET /api/v1/health/pipeline` | 阶段状态与指标 |
| 心跳推送 | `WS /ws/services` | `type/data/timestamp` 协议 |
| 账户接口 | `GET /api/v1/account/balance`, `GET /api/v1/account/positions` | 余额与持仓 |
| 目标仓位 | `GET /api/v1/portfolio/target-positions` | 策略输出的目标仓位 |
| 信号快照 | `GET /api/v1/signals/latest` | 最新策略信号 |
| 仓位推送 | `WS /ws/positions` | 30s 仓位快照 |
| 组合推送 | `WS /ws/portfolio` | 60s 组合指标 |

#### 对齐 API 端点

**HTTP (http.md V1 P0 — 6 端点)**:

- `GET /api/v1/health`
- `GET /api/v1/health/pipeline`
- `GET /api/v1/account/balance`
- `GET /api/v1/account/positions`
- `GET /api/v1/portfolio/target-positions`
- `GET /api/v1/signals/latest`

**WebSocket (websocket.md V1 — 3 端点)**:

- `WS /ws/positions` (30s)
- `WS /ws/portfolio` (60s)
- `WS /ws/services` (30s)

**Redis 键**: `quant:heartbeat:{service}`, `quant:state:{service}:last_run`, `quant:state:{service}:status`, `quant:portfolio:snapshot`

#### 验收标准

- [ ] 新环境 30 分钟内可拉起最小闭环
- [ ] `health` / `pipeline` 能反映各服务状态
- [ ] `WS /ws/services` 可持续推送且断线可重连
- [ ] 6 个 HTTP 端点均可返回正确数据

---

### V1 Sprint 3: 恢复机制与幂等保障 (P1)

> 目标: 先解决"跑着跑着挂掉后恢复"问题。

#### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| Redis 断连恢复 | BaseEventService 重连增强 | 指数退避 |
| 状态持久化 | `quant:state:{service}:last_run/status` | 重启补执行 |
| 订单幂等 | idempotency key 机制 | 防重复执行 |
| 执行重试 | OrderService 重试策略 | 有上限 + 退避 |

#### 对齐 API 端点

本 Sprint 不新增外部 API 端点。聚焦内部质量:

- Redis 事件载荷新增 `idempotency_key` 字段 (见 redis.md `order_executed` 事件)
- Redis 事件载荷新增 `trace_id` 字段 (见 redis.md 统一消息外壳)

#### 验收标准

- [ ] 任一核心服务重启后 5 分钟内恢复
- [ ] 重复事件不产生重复下单
- [ ] Redis 短暂不可用后链路可自动恢复

---

### V1 Sprint 4: 风控前置与执行可靠 (P1)

> 目标: 确保"可控地跑"，而不是"盲目地跑"。

#### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| 风控规则落地 | max_position/exposure/drawdown/reserve/turnover | 模拟盘默认启用 |
| 风控接口 | `GET /api/v1/risk/status`, `GET /api/v1/risk/alerts`, `GET /api/v1/risk/config` | 状态与审计 |
| 系统控制 | `POST /api/v1/system/emergency-stop`, `POST /api/v1/risk/symbols/disable` | 紧急停机 + 停用交易对 |
| 订单审计接口 | `GET /api/v1/orders/history`, `GET /api/v1/orders/trades`, `GET /api/v1/orders/rebalance-history` | 全链路追踪 |
| 活跃挂单 | `GET /api/v1/account/open-orders` | 挂单查询 |
| 组合完整指标 | `GET /api/v1/portfolio/nav`, `pnl`, `drawdown`, `position-deviation`, `volatility` | NAV 曲线、PnL、回撤、偏差、波动率 |
| 告警推送 | `WS /ws/alerts` | 实时风险/执行告警 |

#### 对齐 API 端点

**HTTP (http.md V1 P1 — 14 端点)**:

- `GET /api/v1/risk/status`
- `GET /api/v1/risk/alerts`
- `GET /api/v1/risk/config`
- `POST /api/v1/system/emergency-stop`
- `POST /api/v1/risk/symbols/disable`
- `GET /api/v1/orders/history`
- `GET /api/v1/orders/rebalance-history`
- `GET /api/v1/orders/trades`
- `GET /api/v1/account/open-orders`
- `GET /api/v1/portfolio/nav`
- `GET /api/v1/portfolio/pnl`
- `GET /api/v1/portfolio/drawdown`
- `GET /api/v1/portfolio/position-deviation`
- `GET /api/v1/portfolio/volatility`

**WebSocket (websocket.md V1 — 1 端点)**:

- `WS /ws/alerts` (实时事件触发)

**Redis 键**: `quant:risk:status`, `quant:risk:disabled_symbols`, `quant:system:emergency_stop`

#### 验收标准

- [ ] 未过风控的请求下单数 = 0
- [ ] 订单失败均可审计并告警
- [ ] 连续运行 24h 无阻塞故障
- [ ] 14 个 HTTP 端点均可返回正确数据

---

### V1 Sprint 5: 策略接口统一 + 信号分析 (P2)

> 目标: 为回测兼容打基础，先统一策略接口，再谈指标一致。

#### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| 统一策略抽象 | `BaseStrategy` 输入/输出固化 | 实盘与回测复用 |
| 数据接口适配 | Feature → Strategy 输入结构统一 | 降低分叉代码 |
| 策略列表接口 | `GET /api/v1/strategy/list` | 已注册策略列表 (单策略模式) |
| 策略配置接口 | `GET /api/v1/strategy/{name}/config` | 策略详细配置 |
| 信号覆盖率 | `GET /api/v1/signals/coverage` | 信号覆盖率/缺失率 |
| 兼容验证 | baseline 策略双路径运行 | 模拟盘 + 回测 |

#### 对齐 API 端点

**HTTP (http.md V1 P2 — 3 端点)**:

- `GET /api/v1/strategy/list`
- `GET /api/v1/strategy/{name}/config`
- `GET /api/v1/signals/coverage`

#### 验收标准

- [ ] 同一策略类可在模拟盘与回测两条链路运行
- [ ] 策略输入/输出字段一致（结构层面）
- [ ] 策略列表和信号覆盖率接口返回正确数据

---

### V1 Sprint 6: 回测与模拟盘口径对齐 (P2)

> 目标: 指标可比，不追求零偏差。

#### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| 口径统一 | 时间边界、费用、换手、信号时间点 | same_day/prev_day 明确 |
| 回测提交 | `POST /api/v1/backtest/run` | 异步任务 |
| 回测结果 | `GET /api/v1/backtest/{backtest_id}/result` | 轮询查询 |
| 回测历史 | `GET /api/v1/backtest/history` | 历史任务列表 |
| 偏差评估 | 基准策略偏差报告 | 固定窗口对比 |

#### 对齐 API 端点

**HTTP (http.md V1 P2 — 3 端点)**:

- `POST /api/v1/backtest/run`
- `GET /api/v1/backtest/{backtest_id}/result`
- `GET /api/v1/backtest/history`

#### 验收标准

- [ ] 关键指标（收益/回撤/换手）偏差落入团队阈值
- [ ] 回测任务提交、轮询、结果查询全链路可用

---

## 5. V2 Sprint 详细计划

> V2 新增 strategy_1 (Top 10 多因子) + strategy_3 (ofi_14d 单因子动量)。
> 数据链路扩展: DataIngestion → DollarBar → TickFeature → BarSourceAdapter → FeatureService。
> 支持多策略并行和 ensemble 模式。

---

### V2 Sprint 1: 双源 Bar 接入 + ofi_14d 基础 + 资产池

> 目标: 同时支持 `tick_agg` 与 `direct_kline`，并对下游输出统一 Bar 契约。建立 strategy_3 (ofi_14d) 所需的数据基础。

#### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| Tick 输入链路 | aggTrade 采集 + Dollar Bar | Redis Streams + 自适应阈值 |
| 直拉 Kline 链路 | DirectKlineService | REST/WS 直拉 kline |
| Bar 统一适配层 | `bar_normalized` 输出 | 屏蔽上游来源差异 |
| Shadow 对账 | `bar_source_mismatch` 告警 | 比较 OHLCV 偏差，超阈值告警/降级 |
| T50 月度池 | AssetPoolService 多池扩展 | `t50_monthly` 池，按月更新，历史 >=60 天过滤 |
| 日级聚合 | `daily_aggregator.py` | ofi_d, dollar_volume_d, close_d, ret_d |
| 多日滚动特征 | `rolling_features.py` | ofi_14d = RM_14(ofi_d) |
| 资产池接口 | `GET /api/v1/asset-pool/list`, `GET /api/v1/asset-pool/{pool_name}/symbols` | 资产池查询 |

#### 对齐 API 端点

**HTTP (http.md V2 — 2 端点)**:

- `GET /api/v1/asset-pool/list`
- `GET /api/v1/asset-pool/{pool_name}/symbols`

**Redis 新增事件 (redis.md V2)**:

| 通道 | 说明 |
|------|------|
| `quant:dollar_bar_generated` | Dollar Bar 23 列 (含 buy_sell_imbalance, dollar_volume) |
| `quant:tick_features_enriched` | 9 个 tick 微观特征 (本 Sprint 预留结构，Sprint 2 填充) |
| `quant:kline_raw` | 直拉 kline (双源模式) |
| `quant:bar_normalized` | 统一 Bar 契约 |
| `quant:bar_source_mismatch` | 双源对账偏差告警 |

**Redis Streams**: `quant:stream:aggTrades:{symbol}` (消费者组 `dollar_bar_consumers`)

**Redis 新增键**: `quant:asset_pool:{exchange}:t50_monthly`, `quant:features:daily:{symbol}`, `quant:features:rolling:{symbol}`, `quant:dollar_bar:{symbol}`

#### 验收标准

- [ ] `tick_agg` 与 `direct_kline` 均可独立运行
- [ ] 下游 Feature/Strategy 在两种模式下无需改代码
- [ ] `hybrid` 模式可输出偏差告警并支持自动降级
- [ ] T50 月度池按月自动更新，历史不足 60 天的标的被过滤
- [ ] `ofi_14d` 因子可正确计算并写入 `quant:features:rolling:{symbol}`
- [ ] 不回归 V1 主链路稳定性

---

### V2 Sprint 2: Tick 特征 + 多策略并行

> 目标: 扩展策略能力，不破坏已验证闭环。实现 strategy_1 全部 Top 10 策略和 strategy_3 完整上线。

#### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| Tick 特征服务 | 9 个微观特征 (VPIN, Kyle's Lambda, ...) | TickFeatureService 与 FeatureService 融合 |
| strategy_1 策略实现 | rev_x_inv_vpin, rev_vpin_filter, rev_jump_filter, regime_switch 等 | 对应 Top 1-10 |
| strategy_3 完整上线 | Ofi14dStrategy | R14 换仓 + T50 池 + prev_day 信号 |
| 多策略编排 | single → ensemble | StrategyRegistry + 权重管理 |
| 灵活换仓调度 | R1 / R14 并行 | StrategyService 按策略独立调度 |
| 策略启停 | `POST /api/v1/strategy/{name}/toggle` | 多策略模式下管理各策略状态 |
| 参数变更 | `POST /api/v1/strategy/{name}/config` | 受控发布 (next_rebalance) |
| 变更历史 | `GET /api/v1/strategy/{name}/config/history` | 审计记录 |
| 信号对比 | `GET /api/v1/signals/compare` | 多策略绩效对比 |
| 信号推送 | `WS /ws/signals` | 实时策略信号变更 |

#### 对齐 API 端点

**HTTP (http.md V2 — 4 端点)**:

- `POST /api/v1/strategy/{name}/toggle`
- `POST /api/v1/strategy/{name}/config`
- `GET /api/v1/strategy/{name}/config/history`
- `GET /api/v1/signals/compare`

**WebSocket (websocket.md V2 — 1 新增端点 + 增强)**:

- `WS /ws/signals` (新增，事件触发)
- `WS /ws/positions` 增强: 新增 `strategy_source`, `mode`, `active_strategies` 字段
- `WS /ws/portfolio` 增强: 新增 `per_strategy_metrics` 字段
- `WS /ws/alerts` 增强: 新增 `bar_source_mismatch` 告警类别

**HTTP V1 端点增强**:

- `GET /api/v1/strategy/list`: 返回 `active_mode: "ensemble"`, `ensemble_weight` 字段
- `GET /api/v1/signals/latest`: 新增 `per_strategy_signals` 字段
- `GET /api/v1/portfolio/target-positions`: 支持 ensemble 加权后的目标仓位

**Redis 新增键**: `quant:strategy:status:{name}`

#### 验收标准

- [ ] 2-4 策略可并行产出信号
- [ ] strategy_3 (ofi_14d) R14 换仓周期正确执行
- [ ] strategy_1 (rev_x_inv_vpin) 连续调制信号正确
- [ ] 策略配置变更可审计、可回溯
- [ ] ensemble 模式加权聚合输出正确
- [ ] 不回归 V1 主链路稳定性

---

## 6. 里程碑与闸门（Go/No-Go）

| 里程碑 | 版本 | 覆盖 Sprint | 闸门条件 |
|--------|------|------------|---------|
| M1 | **V1** | V1 Sprint 1-2 | 最短闭环稳定跑通 + 健康接口可用 |
| M2 | **V1** | V1 Sprint 3-4 | 恢复/幂等/风控前置全部通过 |
| M3 | **V1** | V1 Sprint 5-6 | 回测兼容基础完成 + 口径偏差可控 |
| M4 | **V2** | V2 Sprint 1-2 | V2 数据与策略能力上线且不回归 V1 |

---

## 7. 契约冻结规则（本计划适用）

1. REST 路径统一 `/api/v1/*`
2. WS 端点统一:
   - V1: `/ws/positions`, `/ws/portfolio`, `/ws/alerts`, `/ws/services`
   - V2 新增: `/ws/signals`
3. WS 消息结构统一: `type`, `data`, `timestamp`
4. REST 响应包装统一: `code`, `message`, `data`, `timestamp`
5. V2 端点对 V1 客户端向后兼容 (新增字段不影响旧客户端)

---

## 8. 风险与回滚策略

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 接口频繁变更 | 前后端返工 | 每周冻结契约版本 |
| Redis 抖动 | 链路中断 | 重连 + 状态恢复 + 兜底任务 |
| 告警风暴 | 运维噪声 | 聚合、降噪、冷却窗口 |
| 回测对齐困难 | 兼容延期 | 先结构一致，再指标收敛 |
| V2 回归 V1 | MVP 链路失效 | V2 变更不得破坏 M1/M2 已通过能力 |
| 多策略冲突 | 信号矛盾 | ensemble 加权聚合 + 独立仓位限制 |

回滚原则:

1. 任何不满足不变量的版本禁止进入下一里程碑
2. V2 变更不得破坏 M1/M2 已通过能力
3. 风险规则异常时默认降级为"停止下单，继续观测"
