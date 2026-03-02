# 工业级量化后端 - 分阶段开发计划

> 编写时间: 2026-03-02  
> 开发方法论: Agile / Scrum, 8 个 Sprint, 每 Sprint 约 1 周  
> 前置阅读: `docs/architecture.md`, `docs/frontend/01_api_design.md`, `docs/sprint/front_end_sprint.md`

---

## 总览

```
MVP 阶段 (Sprint 1-4, 共 4 周)

Sprint 1               Sprint 2               Sprint 3               Sprint 4
~1 周                  ~1 周                  ~1 周                  ~1 周
------------           ------------           ------------           ------------
解耦与策略骨架          风控与执行闭环          账户与恢复             监控与发布

- BaseStrategy         - RiskService          - AccountService       - MonitorService
- StrategyService      - Order 重构            - account API          - alerts API/WS
- Feature V1 扩展      - 订单/组合 API          - ws/positions         - 健康检查完善

扩展阶段 (Sprint 5-8, 共 4 周)

Sprint 5               Sprint 6               Sprint 7               Sprint 8
~1 周                  ~1 周                  ~1 周                  ~1 周
------------           ------------           ------------           ------------
高频采集               Dollar Bar             Tick 特征与多策略       IaC 与高级接口

- ingestion            - bar 生成             - TickFeatureService    - Terraform
- Redis Streams        - 阈值自适应            - signals/compare API   - backtest API
- DB 批写              - 波动率接口            - strategy_1 接入       - 策略管理 API
```

---

## Sprint 1: 解耦与策略骨架 (MVP)

> **目标**: 策略与执行解耦，先冻结一版前后端接口契约。  
> **交付标准**: `baseline_rev` 可发布信号，前端可联调 `health/strategy` 基础接口。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| Strategy 基础抽象 | `BaseStrategy`, `TargetPortfolio` | 统一策略接口 |
| 策略注册器 | `StrategyRegistry` | 配置驱动创建策略 |
| baseline_rev 接入 | `BaselineRevStrategy` | 对齐 `strategy_2.md` |
| Feature V1 扩展 | ret + zscore | `ret_1h/2h/4h/8h` + `shift(1)` |
| 策略配置 | `configs/strategy_config.yaml` | 先支持 single |
| API 骨架 | `health.py`, `strategy.py` | 统一响应结构 |
| WS 骨架 | `ws_services.py` | 推送服务状态 |

### 前后端联调工作

| 联调项 | 后端准备 | 前端配合 | 完成定义 |
|------|------|------|------|
| `GET /api/v1/health` | 返回服务状态和 heartbeat | 系统状态卡片渲染 | 可展示 healthy/degraded/down |
| `GET /api/v1/strategy/list` | 返回策略列表 | 策略列表页渲染 | 列表字段解析无报错 |
| `GET /api/v1/strategy/{name}/config` | 返回策略参数 | 参数详情只读展示 | 参数字段完整可见 |
| `WS /ws/services` | 推送状态更新 | 前端 store 实时刷新 | 断线自动重连可用 |

### Sprint 1 交付验收标准

- [ ] `signal_generated` 事件可稳定产出
- [ ] `health` 和 `strategy` 接口可用于页面联调
- [ ] 接口结构符合 `docs/frontend/01_api_design.md`

---

## Sprint 2: 风控与执行闭环 (MVP)

> **目标**: 打通 `signal -> risk -> order` 闭环，并开放核心订单与组合接口。  
> **交付标准**: dry-run 下可完整跑通一轮换仓事件链。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| RiskService | `services/risk_service/` | 规则检查 + 拒绝原因 |
| 风控配置 | `configs/risk_config.yaml` | 仓位/敞口/回撤/换手规则 |
| Order 重构 | `position_differ.py`, `order_executor.py` | 只做执行不做选股 |
| 组合接口 | `routes/portfolio.py` | `nav/pnl/target-positions` |
| 订单接口 | `routes/orders.py` | `history/trades/rebalance-history` |
| 风控接口 | `routes/risk.py` | `status/alerts/config` |
| WS 扩展 | `ws_portfolio.py` | 实时 KPI 推送 |

### 前后端联调工作

| 联调项 | 后端准备 | 前端配合 | 完成定义 |
|------|------|------|------|
| `GET /api/v1/risk/status` | 规则阈值和当前值 | 风控规则面板 | 阈值颜色正确 |
| `GET /api/v1/orders/history` | 支持分页/筛选 | 订单审计页 | 分页和筛选正常 |
| `GET /api/v1/portfolio/nav` | NAV 序列数据 | 收益曲线 | 图表可缩放查看 |
| `WS /ws/portfolio` | 实时 KPI 事件 | 顶栏实时刷新 | 延迟小于 3 秒 |

### Sprint 2 交付验收标准

- [ ] 风控可产生 `order_approved` 和 `risk_rejected`
- [ ] Order 可执行差量订单（dry-run）
- [ ] 订单、风控、组合接口可用于前端页面

---

## Sprint 3: 账户与恢复机制 (MVP)

> **目标**: 完成账户状态单入口，并补齐仓位联调所需接口。  
> **交付标准**: 前端仓位页可稳定展示目标/实际偏差。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| AccountService | `services/account_service/` | 30 秒轮询交易所 |
| Redis 账户键 | `quant:account:*` | balance/positions/orders |
| 账户接口 | `routes/account.py` | `balance/positions/open-orders` |
| 仓位 WS | `routes/ws_positions.py` | 仓位增量推送 |
| 恢复兜底 | `quant:state:{service}:last_run` | 重启补执行 |
| 重连机制 | BaseEventService 增强 | Redis 断连恢复 |

### 前后端联调工作

| 联调项 | 后端准备 | 前端配合 | 完成定义 |
|------|------|------|------|
| `GET /api/v1/account/positions` | 标准化仓位结构 | 仓位表格渲染 | 字段映射无缺失 |
| `GET /api/v1/portfolio/target-positions` | 目标仓位快照 | 目标/实际对比 | 偏差高亮可触发 |
| `WS /ws/positions` | 仓位实时推送 | 仓位页实时更新 | 连续运行稳定 |
| `GET /api/v1/account/balance` | 余额与可用资金 | 顶栏资金展示 | 精度和单位一致 |

### Sprint 3 交付验收标准

- [ ] AccountService 成为账户状态唯一入口
- [ ] 仓位页面可联调通过
- [ ] 任一服务重启后 5 分钟内恢复

---

## Sprint 4: 监控与 MVP 发布

> **目标**: 形成可运维 MVP，告警和健康检查可用。  
> **交付标准**: MVP 可连续运行并支持前端 Sprint 1-2 全量联调。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| MonitorService | `services/monitor_service/` | 心跳/延迟/失败告警 |
| 监控配置 | `configs/monitor_config.yaml` | 阈值与 webhook |
| 告警接口 | `risk.py`, `health.py` 扩展 | 告警历史与管线状态 |
| 告警 WS | `routes/ws_alerts.py` | 实时告警推送 |
| 联调脚本 | `all_services.py` | 一键本地联调 |
| MVP 报告 | `docs/sprint/*` | 验收结果与遗留问题 |

### 前后端联调工作

| 联调项 | 后端准备 | 前端配合 | 完成定义 |
|------|------|------|------|
| `GET /api/v1/health/pipeline` | 管线阶段状态 | 系统管线页面 | 阶段状态准确展示 |
| `GET /api/v1/risk/alerts` | 告警历史分页 | 告警列表页面 | 筛选和分页正常 |
| `WS /ws/alerts` | 实时告警事件 | 通知中心显示 | 告警 5 秒内到达 |
| 全链路冒烟 | 全服务启动和回放 | 前端核心页面检查 | 页面无阻塞错误 |

### Sprint 4 交付验收标准

- [ ] 告警路径可用（生成、推送、展示）
- [ ] 健康检查可用于运维判断
- [ ] MVP 连续运行 72 小时无阻塞故障

---

## Sprint 5: 高频采集 (V2)

> **目标**: 建立 aggTrade 高频数据采集链路。  
> **交付标准**: 采集服务可稳定写入 Redis Streams 和 TimescaleDB。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| DataIngestionService | `services/data_ingestion_service/` | WebSocket 采集 |
| Streams 写入 | `quant:stream:aggTrades:{symbol}` | `MAXLEN` 限制 |
| DB 批写 | 批量写入逻辑 | 每秒或每 1000 条 flush |
| 重试策略 | 断线重连与限频退避 | 提升稳定性 |

### 前后端联调工作

| 联调项 | 后端准备 | 前端配合 | 完成定义 |
|------|------|------|------|
| 吞吐指标 | 在健康接口暴露 tps | 系统页展示速率 | 吞吐变化可视化 |
| 延迟指标 | 暴露最新数据延迟 | 延迟告警展示 | 延迟阈值触发正确 |

---

## Sprint 6: Dollar Bar 与分析接口 (V2)

> **目标**: 完成 Dollar Bar 层，并交付前端 Sprint 3 所需分析接口。  
> **交付标准**: 波动率和信号覆盖率接口可联调。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| DollarBarService | `services/dollar_bar_service/` | 阈值自适应和冷启动 |
| Bar 缓存 | `quant:dollar_bar:{symbol}` | 最近 200 bars |
| 波动率接口 | `GET /api/v1/portfolio/volatility` | rolling vol |
| 信号接口初版 | `GET /api/v1/signals/latest`, `GET /api/v1/signals/coverage` | 覆盖率分析 |

### 前后端联调工作

| 联调项 | 后端准备 | 前端配合 | 完成定义 |
|------|------|------|------|
| `GET /api/v1/portfolio/volatility` | 输出组合波动率序列 | 波动率页面 | 曲线和 KPI 一致 |
| `GET /api/v1/signals/coverage` | 输出覆盖率时间序列 | 覆盖率页面 | 低覆盖率可高亮 |

---

## Sprint 7: Tick 特征与多策略 (V2)

> **目标**: 接入 tick 微观特征和 strategy_1，支撑策略对比。  
> **交付标准**: `signals/compare` 可稳定返回多策略对比指标。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| TickFeatureService | `services/tick_feature_service/` | 9 个微观特征 |
| Feature V2 融合 | `feature_service` 扩展 | tick + 日内特征融合 |
| strategy_1 接入 | Top 系列策略实现 | 统一 registry 管理 |
| 对比接口 | `GET /api/v1/signals/compare` | 前端对比页使用 |

### 前后端联调工作

| 联调项 | 后端准备 | 前端配合 | 完成定义 |
|------|------|------|------|
| `GET /api/v1/signals/compare` | 多策略指标与曲线 | 策略对比页面 | 2-4 策略可同屏展示 |
| 策略状态字段 | active/inactive/data_status | 策略卡片增强 | 状态展示一致 |

---

## Sprint 8: IaC 与高级接口 (V2)

> **目标**: 完成基础设施编码化，以及策略管理和回测接口。  
> **交付标准**: 支撑前端 Sprint 4 的参数管理和回测可视化。

### 后端任务清单

| 任务 | 产出 | 说明 |
|------|------|------|
| Terraform | `infra/` | VPC、ECS、Redis、RDS、Secrets |
| 多服务 CI/CD | GitHub Actions 扩展 | 按服务命令部署 |
| 策略管理接口 | `POST /api/v1/strategy/{name}/toggle`, `POST /api/v1/strategy/{name}/config` | 变更审计 |
| 回测接口 | `POST /api/v1/backtest/run`, `GET /api/v1/backtest/{backtest_id}/result`, `GET /api/v1/backtest/history` | 异步任务 |

### 前后端联调工作

| 联调项 | 后端准备 | 前端配合 | 完成定义 |
|------|------|------|------|
| 策略参数变更 | 提交、应用、历史查询 | 参数管理页面 | 可追踪变更记录 |
| 回测任务流程 | 提交、轮询、结果查询 | 回测页面 | 状态流转完整 |

---

## 前后端联调专项分析

### 1. 联调节奏对齐矩阵

| 前端 Sprint (2 周) | 时间窗口 | 对应后端 Sprint (1 周) | 关键联调范围 | 主要风险 |
|------|------|------|------|------|
| FE Sprint 1 | Week 1-2 | BE Sprint 1-2 | `health/strategy` + 基础组合接口 | 账户接口若延期会影响仓位页 |
| FE Sprint 2 | Week 3-4 | BE Sprint 3-4 | `risk/orders/account/alerts` + WS | 告警口径不一致 |
| FE Sprint 3 | Week 5-6 | BE Sprint 5-6 | `volatility/signals` | 高频链路波动导致图表抖动 |
| FE Sprint 4 | Week 7-8 | BE Sprint 7-8 | `signals/compare` + `strategy config` + `backtest` | 异步任务处理复杂 |

### 2. 必须统一的联调契约

| 契约项 | 统一约定 |
|------|------|
| REST 响应包装 | `code`, `message`, `data`, `timestamp` |
| 时间格式 | ISO 8601 UTC |
| 数值精度 | 价格 8 位, 权重 6 位, 金额 2 位 |
| 分页字段 | `page`, `page_size`, `total_items`, `total_pages` |
| WS 消息结构 | `event`, `payload`, `timestamp` |
| 状态枚举 | health/risk/order 的枚举固定 |

### 3. 高风险联调点与缓解方案

| 风险 | 影响 | 缓解方案 |
|------|------|------|
| 接口定义变更频繁 | 前端返工 | 每周五冻结一次接口版本 |
| REST 与 WS 时序不一致 | 页面跳变 | 增加 `as_of` 字段并按时间戳去重 |
| 账户数据非原子更新 | 偏差误判 | 账户快照增加批次版本号 |
| 告警风暴 | 页面通知过载 | 服务端做告警聚合与冷却窗口 |
| 回测任务超时 | 交互卡顿 | 异步任务 + 轮询 + 超时提示 |

### 4. 必做联调用例

1. 停止一个服务，前端 2 分钟内显示 degraded/down。  
2. 注入超限信号，前端可看到 `risk_rejected` 告警。  
3. 制造目标与实际仓位偏差，前端正确高亮 missing/extra。  
4. 走完一次换仓流程，前端可追踪信号、风控、执行、结果。  
5. 提交回测任务，状态从 pending 到 completed 且结果可查看。  

---

## 里程碑

| 里程碑 | 日期 | 完成标准 |
|------|------|------|
| M1 | 2026-03-15 | 策略、风控、执行闭环完成（dry-run） |
| M2 | 2026-03-29 | MVP 可运维并完成前端 Sprint 1-2 联调 |
| M3 | 2026-04-12 | 波动率与信号覆盖率接口完成联调 |
| M4 | 2026-04-26 | V2 + IaC + 回测管理闭环 |

---

## 风险与注意事项

| 风险 | 影响 | 缓解措施 |
|------|------|------|
| 交易所限频 | 账户延迟和订单失败 | AccountService 单入口 + 退避重试 |
| 高频数据堆积 | Redis 内存压力 | Streams `MAXLEN` + 批量消费 |
| 多服务联调复杂 | 回归成本高 | 每 Sprint 预留 15%-20% 缓冲 |
| 测试覆盖不足 | 发布风险上升 | 每 Sprint 至少 1 条核心集成用例 |
| 契约未冻结 | 前后端反复返工 | 采用契约先行和版本冻结 |

