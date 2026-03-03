# 工业级量化后端 - 业务核心逻辑迭代计划（模拟盘优先）

> 编写时间: 2026-03-02  
> 当前阶段: **模拟盘（dry-run / paper trading）**  
> 契约基线: `../frontend/01_api_design.md`, `../api/http.md`, `../api/websocket.md`  
> 策略基线: `../overview/strategy_2.md`（MVP）, `../overview/strategy_1.md`（V2）

---

## 1. 目标与优先级

当前版本以“先能跑通业务闭环”为第一目标，不以“功能最全”或“架构最优雅”为第一目标。

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

## 3. Sprint 总览（建议工期）

| Sprint | 优先级阶段 | 建议工期 | 目标摘要 |
|------|------|------|------|
| Sprint 1 | P0 | 1 周 | 最短闭环跑通（单策略、dry-run） |
| Sprint 2 | P0 | 1 周 | 一键启动 + 最小可观测 |
| Sprint 3 | P1 | 1.5 周 | 恢复机制与幂等保障 |
| Sprint 4 | P1 | 1.5 周 | 风控前置与执行可靠性 |
| Sprint 5 | P2 | 1 周 | 策略接口统一（回测兼容基础） |
| Sprint 6 | P2 | 1.5 周 | 回测与模拟盘口径对齐 |
| Sprint 7 | V2 后置 | 2 周 | 双源 Bar 接入 + 统一契约 |
| Sprint 8 | V2 后置 | 2 周 | Tick 特征 + 多策略并行 |

---

## 4. Sprint 详细计划

## Sprint 1: 最短闭环跑通（P0）

> 目标: 在模拟盘环境下跑通完整交易链路，不追求完美稳定性。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| 策略最小实现 | `BaselineRevStrategy` | 单策略模式 |
| 事件链打通 | Strategy/Risk/Order/Account 服务联通 | 覆盖主路径和拒绝路径 |
| dry-run 执行 | OrderService dry-run 输出 | 不触发真实下单 |
| 基础配置 | `strategy_config.yaml`, `risk_config.yaml`, `order_config.yaml` | 最小参数集 |

### 验收标准

- [ ] 100 次回放中，核心事件链完整率 >= 95%
- [ ] `risk_rejected` 分支可触发且不下单
- [ ] `order_executed`、`account_updated` 事件可见

---

## Sprint 2: 一键启动与最小可观测（P0）

> 目标: 新环境可快速启动，故障可快速定位。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| 一键启动 | `run-all-services` 稳定可用 | 本地联调入口 |
| 健康接口 | `GET /api/v1/health` | 服务状态总览 |
| 管线接口 | `GET /api/v1/health/pipeline` | 阶段状态与指标 |
| 心跳推送 | `WS /ws/services` | `type/data/timestamp` 协议 |

### 验收标准

- [ ] 新环境 30 分钟内可拉起最小闭环
- [ ] `health` / `pipeline` 能反映各服务状态
- [ ] `WS /ws/services` 可持续推送且断线可重连

---

## Sprint 3: 恢复机制与幂等保障（P1）

> 目标: 先解决“跑着跑着挂掉后恢复”问题。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| Redis 断连恢复 | BaseEventService 重连增强 | 指数退避 |
| 状态持久化 | `quant:state:{service}:last_run/status` | 重启补执行 |
| 订单幂等 | idempotency key 机制 | 防重复执行 |
| 执行重试 | OrderService 重试策略 | 有上限 + 退避 |

### 验收标准

- [ ] 任一核心服务重启后 5 分钟内恢复
- [ ] 重复事件不产生重复下单
- [ ] Redis 短暂不可用后链路可自动恢复

---

## Sprint 4: 风控前置与执行可靠（P1）

> 目标: 确保“可控地跑”，而不是“盲目地跑”。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| 风控规则落地 | max_position/exposure/drawdown/reserve | 模拟盘默认启用 |
| 风控接口 | `GET /api/v1/risk/status`, `GET /api/v1/risk/alerts`, `GET /api/v1/risk/config` | 状态与审计 |
| 订单审计接口 | `GET /api/v1/orders/history`, `GET /api/v1/orders/trades`, `GET /api/v1/orders/rebalance-history` | 全链路追踪 |
| 告警推送 | `WS /ws/alerts` | 实时风险/执行告警 |

### 验收标准

- [ ] 未过风控的请求下单数 = 0
- [ ] 订单失败均可审计并告警
- [ ] 连续运行 24h 无阻塞故障

---

## Sprint 5: 策略接口统一（P2）

> 目标: 为回测兼容打基础，先统一策略接口，再谈指标一致。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| 统一策略抽象 | `BaseStrategy` 输入/输出固化 | 实盘与回测复用 |
| 数据接口适配 | Feature -> Strategy 输入结构统一 | 降低分叉代码 |
| 兼容验证 | baseline 策略双路径运行 | 模拟盘 + 回测 |

### 验收标准

- [ ] 同一策略类可在模拟盘与回测两条链路运行
- [ ] 策略输入/输出字段一致（结构层面）

---

## Sprint 6: 回测与模拟盘口径对齐（P2）

> 目标: 指标可比，不追求零偏差。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| 口径统一 | 时间边界、费用、换手、信号时间点 | same_day/prev_day 明确 |
| 回测接口 | `POST /api/v1/backtest/run`, `GET /api/v1/backtest/{backtest_id}/result`, `GET /api/v1/backtest/history` | 异步查询 |
| 偏差评估 | 基准策略偏差报告 | 固定窗口对比 |

### 验收标准

- [ ] 关键指标（收益/回撤/换手）偏差落入团队阈值
- [ ] 回测任务提交、轮询、结果查询全链路可用

---

## Sprint 7: 双源 Bar 接入与统一契约（V2 后置）

> 目标: 同时支持 `tick_agg` 与 `direct_kline`，并对下游输出统一 Bar 契约。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| Tick 输入链路 | aggTrade 采集 + Dollar Bar | Redis Streams + 自适应阈值 |
| 直拉 Kline 链路 | DirectKlineService | REST/WS 直拉 kline |
| Bar 统一适配层 | `bar_normalized` 输出 | 屏蔽上游来源差异 |
| Shadow 对账 | `bar_source_mismatch` 告警 | 比较 OHLCV 偏差，超阈值告警/降级 |

### 验收标准

- [ ] `tick_agg` 与 `direct_kline` 均可独立运行
- [ ] 下游 Feature/Strategy 在两种模式下无需改代码
- [ ] `hybrid` 模式可输出偏差告警并支持自动降级
- [ ] 不回归 MVP 主链路稳定性

---

## Sprint 8: Tick 特征与多策略并行（V2 后置）

> 目标: 扩展策略能力，不破坏已验证闭环。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| Tick 特征服务 | 9 个微观特征 | 与 FeatureService 融合 |
| 多策略编排 | single -> ensemble | 策略注册与权重管理 |
| 对比接口 | `GET /api/v1/signals/compare` | 支撑策略对比 |
| 策略管理接口 | `POST /api/v1/strategy/{name}/toggle`, `POST /api/v1/strategy/{name}/config`, `GET /api/v1/strategy/{name}/config/history` | 受控变更 |

### 验收标准

- [ ] 2-4 策略可并行产出信号
- [ ] 策略配置变更可审计、可回溯

---

## 5. 里程碑与闸门（Go/No-Go）

| 里程碑 | 覆盖 Sprint | 闸门条件 |
|------|------|------|
| M1 | Sprint 1-2 | 最短闭环稳定跑通 + 健康接口可用 |
| M2 | Sprint 3-4 | 恢复/幂等/风控前置全部通过 |
| M3 | Sprint 5-6 | 回测兼容基础完成 + 口径偏差可控 |
| M4 | Sprint 7-8 | V2 数据与策略能力上线且不回归 |

---

## 6. 契约冻结规则（本计划适用）

1. REST 路径统一 ` /api/v1/* `
2. WS 端点统一:
   - `/ws/positions`
   - `/ws/portfolio`
   - `/ws/alerts`
   - `/ws/services`
3. WS 消息结构统一: `type`, `data`, `timestamp`
4. REST 响应包装统一: `code`, `message`, `data`, `timestamp`

---

## 7. 风险与回滚策略

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 接口频繁变更 | 前后端返工 | 每周冻结契约版本 |
| Redis 抖动 | 链路中断 | 重连 + 状态恢复 + 兜底任务 |
| 告警风暴 | 运维噪声 | 聚合、降噪、冷却窗口 |
| 回测对齐困难 | 兼容延期 | 先结构一致，再指标收敛 |

回滚原则:

1. 任何不满足不变量的版本禁止进入下一里程碑  
2. V2 变更不得破坏 M1/M2 已通过能力  
3. 风险规则异常时默认降级为“停止下单，继续观测”
