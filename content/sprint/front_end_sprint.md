# 前端监控仪表盘 — 分阶段开发计划

> 编写时间: 2026-03-02
> 开发方法论: Agile / Scrum, 4 个 Sprint, 每 Sprint 约 2 周
> 前置阅读: `docs/frontend/00_overview.md`, `docs/frontend/01_api_design.md`

---

## 总览

```
Sprint 1 (P1)          Sprint 2 (P2)          Sprint 3 (P3)          Sprint 4 (P4)
~2 周                  ~2 周                  ~2 周                  ~2 周
──────────────         ──────────────         ──────────────         ──────────────
核心可观测性           风控与订单              波动率与信号            参数管理与高级

- 项目脚手架           - 风控状态面板          - 波动率监控            - 策略参数调整
- 服务健康面板         - 告警历史              - 信号覆盖率            - 策略启停控制
- 实时仓位监控         - 订单审计页
- NAV / PnL 曲线       - 回撤监控
- 策略列表 (只读)           
```

---

## Sprint 1: 核心可观测性 (MVP)

> **目标**: 让用户能看到系统是否在正常运行, 仓位是否到位, 策略是否在赚钱。
> **交付标准**: 部署后可直接用于日常盯盘。

### 后端前置工作

> **依赖说明**: 以下接口对应后端 Sprint 1-2 的交付内容。若前后端并行开发，前端应先使用 Mock 数据。

在写前端之前, 后端需要先完成以下接口 (在 `controller/routes/` 下新增):

| 路由文件 | 接口 | 数据来源 |
|---------|------|---------|
| `routes/health.py` | `GET /api/v1/health`, `GET /api/v1/health/pipeline` | Redis heartbeat + state keys |
| `routes/account.py` | `GET /api/v1/account/balance`, `GET /api/v1/account/positions` | Redis account keys |
| `routes/portfolio.py` | `GET /api/v1/portfolio/nav`, `GET /api/v1/portfolio/target-positions` | TimescaleDB + Redis |
| `routes/strategy.py` | `GET /api/v1/strategy/list`, `GET /api/v1/strategy/{name}/config` | StrategyRegistry + Redis |
| `routes/ws_positions.py` | `WS /ws/positions` | Redis 定时读取推送 |
| `routes/ws_portfolio.py` | `WS /ws/portfolio` | Redis 定时读取推送 |
| `routes/ws_services.py` | `WS /ws/services` | Redis heartbeat 扫描推送 |

### 前端任务清单

#### 1.1 项目脚手架 (Day 1-2)

| 任务 | 产出 | 说明 |
|------|------|------|
| 初始化 Vue 3 + Vite + TypeScript 项目 | `frontend/` 目录 | `npm create vite@latest frontend -- --template vue-ts` |
| 安装依赖 | `package.json` | Element Plus, Pinia, Vue Router, Axios, ECharts, lightweight-charts |
| 配置 Vite 代理 | `vite.config.ts` | 开发环境 `/api` 代理到 `http://localhost:8000` |
| 搭建主布局 | `AppLayout.vue`, `TopBar.vue`, `SideNav.vue` | 左侧导航 + 顶部状态栏 + RouterView |
| 配置路由 | `router/index.ts` | 10 个页面路由 |
| 封装 Axios 实例 | `api/client.ts` | baseURL, 错误拦截, 响应统一解包 |
| 封装 WebSocket | `composables/useWebSocket.ts` | 自动重连 + 组件卸载清理 |
| TypeScript 类型定义 | `types/*.ts` | 与 `01_api_design.md` 中 JSON Schema 对齐 |

#### 1.2 系统状态页 (Day 3-4)

| 任务 | 产出 | 对接接口 |
|------|------|---------|
| 服务状态卡片组件 | `ServiceCard.vue` | `GET /api/v1/health` |
| 状态指示灯组件 | `StatusDot.vue` | 绿=healthy, 黄=degraded, 红=down |
| 数据管线阶段组件 | `PipelineStages.vue` | `GET /api/v1/health/pipeline` |
| 系统状态页面 | `SystemView.vue` | 组合上述组件 |
| WebSocket 实时更新 | `stores/system.ts` | `WS /ws/services` |

**页面线框:**

```
┌─────────────────────────────────────────────────────┐
│  系统状态                                            │
│                                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐│
│  │ AssetPool │ │Aggregator│ │ Feature  │ │Strategy││
│  │ ● healthy│ │ ● healthy│ │ ◐ degraded│ │● healthy││
│  │ 86400s   │ │ 86400s   │ │ 240s ago │ │ 86400s ││
│  └──────────┘ └──────────┘ └──────────┘ └────────┘│
│                                                     │
│  数据管线                                            │
│  ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐│
│  │数据采集│──▶│Bar聚合 │──▶│特征计算│──▶│信号生成││
│  │running │   │running │   │idle    │   │idle    ││
│  │245t/s  │   │2100bar │   │上次23:54│  │上次23:55││
│  └────────┘   └────────┘   └────────┘   └────────┘│
└─────────────────────────────────────────────────────┘
```

#### 1.3 仓位监控页 (Day 5-7)

| 任务 | 产出 | 对接接口 |
|------|------|---------|
| 仓位表格组件 | `PositionTable.vue` | `GET /api/v1/account/positions` |
| 目标仓位对比 | 目标 vs 实际列 | `GET /api/v1/portfolio/target-positions` |
| 偏差高亮 | 偏差 > 10% 红色 | 前端计算 |
| 多空分布饼图 | ECharts Pie | 统计 long/short 数量 |
| Pinia Store | `stores/positions.ts` | WS `/ws/positions` 自动更新 |
| 仓位监控页面 | `PositionsView.vue` | 组合上述组件 |

**页面线框:**

```
┌──────────────────────────────────────────────────────────────────┐
│  仓位监控                       多空分布: [==== 30 Long ====]    │
│                                           [=== 15 Short ===]    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ Symbol          │ Side │ Target │ Actual │ Deviation │ PnL │  │
│  │ BTC/USDT:USDT  │ Long │ 1.67% │ 1.55%  │ -7.2% ● │+1.75│  │
│  │ ETH/USDT:USDT  │ Long │ 1.67% │ 1.70%  │ +1.8% ● │+0.50│  │
│  │ XRP/USDT:USDT  │Short │-1.67% │  0.00% │ 100%  ● │ --- │  │
│  │ ...            │      │       │        │         │     │  │
│  └────────────────────────────────────────────────────────────┘  │
│  Total: 60 positions | Matched: 55 | Missing: 3 | Extra: 2      │
└──────────────────────────────────────────────────────────────────┘
```

#### 1.4 收益监控页 (Day 8-10)

| 任务 | 产出 | 对接接口 |
|------|------|---------|
| NAV 曲线组件 | `NavChart.vue` | `GET /api/v1/portfolio/nav` |
| TradingView 封装 | lightweight-charts createChart | 支持缩放/十字线/时间轴 |
| KPI 卡片组件 | `KpiCard.vue` | 通用: 标题 + 数值 + 变化率 + 颜色 |
| 日 PnL 柱状图 | `PnlBarChart.vue` (ECharts) | `GET /api/v1/portfolio/pnl` |
| Pinia Store | `stores/portfolio.ts` | WS `/ws/portfolio` 实时更新顶栏 KPI |
| 收益监控页面 | `PnlView.vue` | 组合上述组件 |

**页面线框:**

```
┌──────────────────────────────────────────────────────────────────┐
│  收益监控                                                        │
│                                                                  │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌─────────┐             │
│  │NAV      │ │总收益    │ │Sharpe(30d)│ │日均PnL  │             │
│  │$10,234  │ │+2.35%   │ │1.85      │ │$3.80    │             │
│  └─────────┘ └─────────┘ └──────────┘ └─────────┘             │
│                                                                  │
│  NAV 曲线 (TradingView Lightweight Charts)                       │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  $10,400 ┤                                          ╱╲     │  │
│  │  $10,200 ┤                              ╱──────────╱  ╲──  │  │
│  │  $10,000 ┤──────────╱──────────────────╱                   │  │
│  │          └─────┬─────┬─────┬─────┬─────┬─────              │  │
│  │             Jan    Jan    Feb    Feb    Mar                 │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  日 PnL (ECharts Bar)                                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  ██  ██     ██  ██ ██  ██  ██     ██  ██              ██  │  │
│  │  ██  ██  ▓▓ ██  ██ ██  ██  ██  ▓▓ ██  ██  ▓▓  ██  ▓▓ ██  │  │
│  │  ──────────────────────────────────────────────────────────│  │
│  │      ▓▓         ▓▓                     ▓▓  ▓▓            │  │
│  └────────────────────────────────────────────────────────────┘  │
│  ██ 盈利    ▓▓ 亏损    胜率: 62%                                 │
└──────────────────────────────────────────────────────────────────┘
```

#### 1.5 策略列表 (只读) + 总览页 (Day 11-14)

| 任务 | 产出 | 对接接口 |
|------|------|---------|
| 策略卡片组件 | `StrategyCard.vue` | `GET /api/v1/strategy/list` |
| 策略列表页面 | `StrategyView.vue` | 卡片网格, 显示名称/状态/参数/上次信号时间 |
| 总览 Dashboard | `DashboardView.vue` | 聚合: 小 NAV 曲线 + KPI 卡片 + 服务灯 + 最近告警 |
| 顶栏实时 KPI | `TopBar.vue` 增强 | WS `/ws/portfolio` → NAV / PnL / 回撤 / 告警数 |

### Sprint 1 交付验收标准

- [ ] 打开首页能看到 NAV、日 PnL、最大回撤、告警数 4 个核心 KPI
- [ ] 系统状态页能看到所有服务心跳, degraded/down 服务红色高亮
- [ ] 仓位页能看到目标 vs 实际仓位, 偏差 >10% 自动标红
- [ ] 收益页 NAV 曲线可缩放, 日 PnL 柱状图区分盈亏颜色
- [ ] 策略列表页能查看所有已注册策略及其参数
- [ ] WebSocket 连接断开后 3 秒内自动重连

---

## Sprint 2: 风控与订单

> **目标**: 让用户能看到风控是否正常、订单是否正确执行、回撤是否可控。
> **交付标准**: 任何风控异常和订单失败都能在仪表盘上立即发现。

### 后端前置工作

> **依赖说明**: 以下接口对应后端 Sprint 3-4 的交付内容。

| 路由文件 | 接口 | 数据来源 |
|---------|------|---------|
| `routes/risk.py` | `GET /api/v1/risk/status`, `GET /api/v1/risk/alerts`, `GET /api/v1/risk/config` | Redis risk keys + DB |
| `routes/orders.py` | `GET /api/v1/orders/history`, `GET /api/v1/orders/trades`, `GET /api/v1/orders/rebalance-history` | TimescaleDB |
| `routes/portfolio.py` (扩展) | `GET /api/v1/portfolio/drawdown`, `GET /api/v1/portfolio/position-deviation` | 计算 + Redis |
| `routes/ws_alerts.py` | `WS /ws/alerts` | 订阅 Redis Pub/Sub 风控通道 |

### 前端任务清单

#### 2.1 风控状态面板 (Day 1-4)

| 任务 | 产出 | 说明 |
|------|------|------|
| 风控规则表格 | `RiskRulesTable.vue` | 规则名 / 阈值 / 当前值 / 状态, 接近阈值黄色 |
| 进度条组件 | Element Plus `el-progress` | 当前值/阈值 可视化 (绿 → 黄 → 红) |
| 告警历史时间线 | `AlertTable.vue` | 分页表格 + 严重度筛选 + 时间范围 |
| 实时告警推送 | `stores/risk.ts` + WS | WS `/ws/alerts` → 新告警弹出 Notification |
| 风控面板页面 | `RiskView.vue` | 组合上述组件 |

**页面线框:**

```
┌──────────────────────────────────────────────────────────────────┐
│  风控面板                                       状态: ● 正常     │
│                                                                  │
│  风控规则                                                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 规则              │ 阈值    │ 当前值  │ 使用率 │ 状态     │  │
│  │ 单币种最大仓位     │ 15%    │ 8%     │ ████░░ │ ● OK    │  │
│  │ 最大总敞口         │ 100%   │ 85%    │ ██████ │ ◐ 注意  │  │
│  │ 最大回撤止损       │ 30%    │ 5.3%   │ █░░░░░ │ ● OK    │  │
│  │ 最低保留余额       │ $100   │ $5,120 │ ░░░░░░ │ ● OK    │  │
│  │ 每日最大换手率     │ 3.0    │ 0.45   │ █░░░░░ │ ● OK    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  告警历史                                                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 03-02 23:55 │ ● CRITICAL │ 下单失败: ETH/USDT           │  │
│  │ 03-02 20:15 │ ◐ WARNING  │ 总敞口接近阈值: 0.92/1.0      │  │
│  │ 03-01 23:55 │ ● CRITICAL │ 下单失败: SOL/USDT            │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

#### 2.2 订单审计页 (Day 5-8)

| 任务 | 产出 | 说明 |
|------|------|------|
| 订单历史表格 | `OrderTable.vue` | 分页 + symbol/side/status 筛选 + 时间范围 |
| 成交明细表格 | 复用 `OrderTable.vue` 变体 | 不同列配置 |
| 换仓事件时间线 | `RebalanceTimeline.vue` | 每次换仓的汇总卡片 (订单数/成功率/换手率/费用) |
| 订单审计页面 | `OrdersView.vue` | Tab 切换: 订单 / 成交 / 换仓 |

**页面线框:**

```
┌──────────────────────────────────────────────────────────────────┐
│  订单审计    [订单历史] [成交明细] [换仓事件]                      │
│                                                                  │
│  筛选: [Symbol ▼] [Side ▼] [Status ▼] [2026-03-01 ~ 2026-03-02]│
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ 时间      │ Symbol     │ Side │ 数量   │ 价格     │ 状态  │  │
│  │ 23:55:06 │ BTC/USDT  │ Buy  │ 0.005  │ $62,450 │ Filled│  │
│  │ 23:55:07 │ ETH/USDT  │ Sell │ 0.100  │ $3,180  │ Filled│  │
│  │ 23:55:08 │ SOL/USDT  │ Buy  │ 2.500  │ $142.30 │Failed │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                              Page 1/5  [< 1 2 >]│
└──────────────────────────────────────────────────────────────────┘
```

#### 2.3 回撤监控 (Day 9-11)

| 任务 | 产出 | 说明 |
|------|------|------|
| 回撤曲线组件 | `DrawdownChart.vue` (ECharts) | 填充面积图, 最大回撤水位线 |
| 回撤 KPI 卡片 | 当前回撤 / 最大回撤 / 恢复天数 | 复用 `KpiCard.vue` |
| 收益页扩展 | `PnlView.vue` 增加回撤 Tab | Tab: NAV / 日PnL / 回撤 |

#### 2.4 仓位偏差增强 (Day 12-14)

| 任务 | 产出 | 说明 |
|------|------|------|
| 偏差详情接口对接 | `GET /api/v1/portfolio/position-deviation` | 新接口 |
| 偏差汇总卡片 | matched/missing/extra 统计 | 顶部 KPI |
| 缺失仓位高亮 | 红色行, "MISSING" 标签 | 表格条件样式 |
| 多余仓位高亮 | 黄色行, "EXTRA" 标签 | 表格条件样式 |

### Sprint 2 交付验收标准

- [ ] 风控面板能看到所有 6 条规则的实时指标, 接近阈值 80% 变黄, 超过变红
- [ ] 风控告警通过 WebSocket 实时弹出 Notification
- [ ] 订单审计页能按 symbol / side / status 筛选, 支持分页
- [ ] 换仓事件页能看到每次换仓的汇总信息 (成功率/失败数/费用)
- [ ] 回撤曲线清晰显示最大回撤水位线和恢复趋势
- [ ] 仓位偏差中, missing/extra 仓位有明确的视觉标记

---

## Sprint 3: 波动率与信号分析

> **目标**: 让用户能深入分析策略信号质量和组合风险特征。
> **交付标准**: 能回答 "哪些 symbol 信号缺失?" "组合波动率在上升吗?" "baseline_rev 比 strategy_1 好吗?"

### 后端前置工作

> **依赖说明**: 以下接口对应后端 Sprint 5-6 的交付内容。

| 路由文件 | 接口 | 数据来源 |
|---------|------|---------|
| `routes/portfolio.py` (扩展) | `GET /api/v1/portfolio/volatility` | TimescaleDB 计算 |
| `routes/signals.py` | `GET /api/v1/signals/latest`, `GET /api/v1/signals/coverage`, `GET /api/v1/signals/compare` | Redis + TimescaleDB |

### 前端任务清单

#### 3.1 波动率监控页 (Day 1-5)

| 任务 | 产出 | 说明 |
|------|------|------|
| 组合级波动率 | 年化 vol / Sharpe / Sortino KPI + rolling vol 曲线 | ECharts Line |
| 策略级波动率 | 多策略波动率对比柱状图 | ECharts Bar |
| Symbol 级热力图 | 按 vol 着色的 symbol 矩阵 | ECharts Heatmap |
| 三层 Tab 切换 | `VolatilityView.vue` | Tab: 组合 / 策略 / Symbol |

**页面线框 (组合级):**

```
┌──────────────────────────────────────────────────────────────────┐
│  波动率监控    [组合级] [策略级] [Symbol级]                        │
│                                                                  │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌─────────┐             │
│  │年化波动率│ │Sharpe    │ │Sortino   │ │Calmar   │             │
│  │18.5%    │ │1.85      │ │2.31      │ │2.47     │             │
│  └─────────┘ └──────────┘ └──────────┘ └─────────┘             │
│                                                                  │
│  30 日 Rolling Volatility                                        │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  22% ┤          ╱╲                                         │  │
│  │  20% ┤     ╱───╱  ╲──╲                                    │  │
│  │  18% ┤────╱            ╲────────────╱──────── 当前: 18.5%  │  │
│  │  16% ┤                               ╲──────              │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

**页面线框 (Symbol 级热力图):**

```
┌──────────────────────────────────────────────────────────────────┐
│  波动率监控    [组合级] [策略级] [Symbol级]                        │
│                                                                  │
│  Symbol 波动率热力图 (按年化波动率着色)                            │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ BTC  ETH  BNB  SOL  XRP  ADA  DOGE DOT  AVAX LINK ...   │  │
│  │ ■    ■    ■    ■    ■    ■    ■    ■    ■    ■          │  │
│  │ 45%  52%  38%  68%  42%  55%  78%  48%  61%  39%        │  │
│  │                                                          │  │
│  │ ■ <30% (低)  ■ 30-50% (中)  ■ 50-70% (高)  ■ >70% (极高)│  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

#### 3.2 信号覆盖率页 (Day 6-8)

| 任务 | 产出 | 说明 |
|------|------|------|
| 覆盖率趋势图 | ECharts Line (coverage_rate vs date) | 低于 95% 标红区域 |
| 缺失 symbol 列表 | 表格: 日期 / 缺失 symbol / 原因 | 点击展开详情 |
| 覆盖率 KPI 卡片 | 平均 / 最低 / 当前 | 复用 `KpiCard.vue` |
| 信号分析页面 | `SignalsView.vue` | Tab: 最新信号 / 覆盖率 / 缺失明细 |

#### 3.3 策略对比页 (Day 9-14)

| 任务 | 产出 | 说明 |
|------|------|------|
| 策略选择器 | Element Plus Select (多选, 最多 4 个) | 从策略列表读取 |
| 双轴 NAV 叠加曲线 | TradingView 多 Series | 不同颜色区分策略 |
| 指标对比卡片网格 | Sharpe / MDD / Calmar / 换手率 / 胜率 | 高亮最优值 |
| 策略对比页面 | `CompareView.vue` | 选择策略 → 加载对比数据 |

**页面线框:**

```
┌──────────────────────────────────────────────────────────────────┐
│  策略对比    选择: [baseline_rev ✓] [rev_x_inv_vpin ✓]           │
│                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │ Sharpe       │ │ Max DD      │ │ Calmar       │              │
│  │ base: 1.85  │ │ base: -8.2% │ │ base: 2.47  │              │
│  │ vpin: 3.33 ★│ │ vpin:-21.1% │ │ vpin: 1.32  │              │
│  └─────────────┘ └─────────────┘ └─────────────┘               │
│                                                                  │
│  NAV 曲线叠加                                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  ── baseline_rev (蓝)                                      │  │
│  │  ── rev_x_inv_vpin (橙)                                    │  │
│  │  $10,400 ┤                   ╱──── 橙                      │  │
│  │  $10,200 ┤          ╱───────╱  ╱── 蓝                      │  │
│  │  $10,000 ┤─────────╱───────────                            │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Sprint 3 交付验收标准

- [ ] 组合级波动率页显示 rolling 30d vol 曲线 + Sharpe/Sortino/Calmar
- [ ] Symbol 级热力图能直观看到高波动 symbol
- [ ] 信号覆盖率趋势图清晰标注低覆盖率日期
- [ ] 策略对比页能同时叠加 2-4 个策略 NAV 曲线, 指标高亮最优值
- [ ] 所有图表支持 tooltip 显示详细数值

---

## Sprint 4: 参数管理与高级功能

> **目标**: 让用户能安全地调整策略参数、管理策略生命周期、查看回测结果。
> **交付标准**: 参数变更有审计记录, 策略启停有确认步骤, 回测结果可视化。

### 后端前置工作

> **依赖说明**: 以下接口对应后端 Sprint 7-8 的交付内容。

| 路由文件 | 接口 | 数据来源 |
|---------|------|---------|
| `routes/strategy.py` (扩展) | `POST /api/v1/strategy/{name}/toggle`, `POST /api/v1/strategy/{name}/config`, `GET /api/v1/strategy/{name}/config/history` | Redis + YAML + TimescaleDB |
| `routes/backtest.py` | `POST /api/v1/backtest/run`, `GET /api/v1/backtest/{id}/result`, `GET /api/v1/backtest/history` | BacktestEngine + TimescaleDB |

### 前端任务清单

#### 4.1 策略参数调整 (Day 1-5)

| 任务 | 产出 | 说明 |
|------|------|------|
| 参数编辑表单 | Element Plus Form | 从当前参数自动生成表单字段 |
| 变更对比 Diff | 旧值 vs 新值 高亮 | 类似 Git diff 展示 |
| 变更原因输入 | 必填 textarea | 审计需要 |
| 二次确认对话框 | `ConfirmDialog.vue` | 显示变更内容, 必须输入 "CONFIRM" 才能提交 |
| 参数变更接口 | `POST /api/v1/strategy/{name}/config` | 提交变更请求 |

**受控发布流程:**

```
用户修改参数
    │
    ▼
前端展示 Diff (旧值 → 新值)
    │
    ▼
用户输入变更原因
    │
    ▼
二次确认对话框 (输入 "CONFIRM")
    │
    ▼
POST /api/v1/strategy/{name}/config
    │
    ▼
后端写入 change_request (status: pending)
    │
    ▼
下一次 rebalance 时 StrategyService 读取并应用
    │
    ▼
status: applied
```

#### 4.2 策略启停控制 (Day 6-7)

| 任务 | 产出 | 说明 |
|------|------|------|
| 启停开关 | Element Plus Switch + 确认弹窗 | 停用策略 = 不再生成信号 |
| 状态历史 | 启停记录时间线 | 复用告警时间线组件 |

#### 4.3 版本管理 (Day 8-10)

| 任务 | 产出 | 说明 |
|------|------|------|
| 参数变更历史表格 | 时间 / 版本号 / 变更内容 / 原因 / 状态 | `GET /api/v1/strategy/{name}/config/history` |
| 版本对比 | 选择两个版本, 展示参数 diff | 前端 JSON diff 计算 |
| 策略管理页增强 | `StrategyView.vue` 增加 Tab | Tab: 概览 / 参数 / 历史 / 启停 |

#### 4.4 回测可视化 (Day 11-13)

| 任务 | 产出 | 说明 |
|------|------|------|
| 回测参数表单 | 策略 / 时间范围 / 费率 / 换仓频率 | Element Plus Form |
| 回测状态轮询 | 提交后轮询 status 直到 completed | Polling composable |
| 结果指标卡片 | Sharpe / MDD / Calmar / 总收益 / 胜率 | 复用 `KpiCard.vue` |
| 回测 NAV 曲线 | TradingView + IS/OOS 分割线标注 | 垂直分割线 |
| 回测历史列表 | 历史任务表格 + 点击查看结果 | 分页表格 |
| 回测页面 | `BacktestView.vue` | 组合上述组件 |

#### 4.5 暗色主题 (Day 14)

| 任务 | 产出 | 说明 |
|------|------|------|
| CSS 变量切换 | `composables/useTheme.ts` | Element Plus dark mode + ECharts theme |
| 主题切换按钮 | TopBar 右侧 sun/moon 图标 | localStorage 持久化 |
| 图表主题适配 | ECharts dark theme + Lightweight Charts dark theme | 跟随系统主题 |

### Sprint 4 交付验收标准

- [ ] 策略参数可编辑, 提交前必须经过 diff 展示 + 原因输入 + 二次确认
- [ ] 参数变更历史可追溯, 能看到每次变更的旧值/新值/原因/时间
- [ ] 策略启停有确认弹窗, 停用后策略卡片状态变为 "inactive"
- [ ] 回测页面可提交任务, 完成后展示指标卡片 + NAV 曲线
- [ ] 暗色/亮色主题一键切换, 所有图表正确适配

---

## 开发环境搭建指南

### 快速开始

```bash
cd frontend
npm install
npm run dev          # 启动 Vite dev server (http://localhost:5173)

# 确保后端在 8000 端口运行:
cd .. && poetry run main system server
```

### Vite 代理配置

```typescript
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      },
      '/ws': {
        target: 'ws://localhost:8000',
        ws: true,
      },
    },
  },
})
```

### 推荐 VS Code 插件

- Vue - Official (Volar)
- TypeScript Vue Plugin
- UnoCSS / Tailwind CSS IntelliSense
- ESLint + Prettier

---

## 依赖清单

```json
{
  "dependencies": {
    "vue": "^3.4.0",
    "vue-router": "^4.3.0",
    "pinia": "^2.1.0",
    "element-plus": "^2.7.0",
    "axios": "^1.7.0",
    "echarts": "^5.5.0",
    "vue-echarts": "^7.0.0",
    "lightweight-charts": "^4.1.0",
    "dayjs": "^1.11.0"
  },
  "devDependencies": {
    "vite": "^5.4.0",
    "@vitejs/plugin-vue": "^5.1.0",
    "typescript": "^5.5.0",
    "vue-tsc": "^2.0.0",
    "unocss": "^0.62.0",
    "eslint": "^9.0.0",
    "@antfu/eslint-config": "^3.0.0"
  }
}
```

---

## 风险与注意事项

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| 后端接口未就绪时前端无法开发 | Sprint 延期 | 前端使用 Mock 数据先行开发, 后端并行实现 |
| WebSocket 连接在弱网环境频繁断开 | 数据不实时 | 指数退避重连 + REST 轮询降级方案 |
| ECharts 大数据量渲染卡顿 | 波动率热力图 488 symbols | dataZoom 分页 + sampling 降采样 |
| TradingView Charts 在长时间序列中内存泄漏 | 浏览器 tab 崩溃 | 限制 NAV 曲线最大数据点 (1000), 超出按需加载 |
| 策略参数误操作 | 实盘风险 | 二次确认 + 变更审计 + 默认 dry_run=true |
