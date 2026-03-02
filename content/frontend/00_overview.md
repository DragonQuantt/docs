# 前端监控仪表盘 — 架构总览

> 编写时间: 2026-03-02
> 对接后端: `docs/architecture.md` (API Gateway + 12 个微服务)
> 前置阅读: `docs/architecture.md` 第 1.2 节整体架构图、第 2.2 节各服务职责

---

## 一、技术选型

| 类别 | 技术 | 版本 | 许可 | 选型理由 |
|------|------|------|------|---------|
| 框架 | Vue 3 | 3.4+ | MIT | Composition API + TypeScript 一等支持 |
| 构建 | Vite | 5.x | MIT | 毫秒级 HMR, 开箱即用 Vue 支持 |
| UI 库 | Element Plus | 2.x | MIT | 成熟的 Vue 3 组件库, 表格/表单/对话框丰富 |
| 状态管理 | Pinia | 2.x | MIT | Vue 官方状态库, TypeScript 友好 |
| 路由 | Vue Router | 4.x | MIT | 标准 SPA 路由 |
| 金融图表 | TradingView Lightweight Charts | 4.x | Apache 2.0 | 专业 K 线 / NAV 曲线, 高性能 Canvas 渲染, 免费 |
| 通用图表 | Apache ECharts | 5.x | Apache 2.0 | 热力图 / 雷达图 / 柱状图 / 桑基图, 免费 |
| HTTP 客户端 | Axios | 1.x | MIT | 拦截器 + 请求/响应类型化 |
| WebSocket | 原生 WebSocket + 自动重连封装 | - | - | 轻量, 无需额外依赖 |
| CSS 工具 | UnoCSS 或 TailwindCSS | - | MIT | 原子化 CSS, 与 Element Plus 互补 |
| 语言 | TypeScript | 5.x | Apache 2.0 | 类型安全, 接口契约强制 |

### 为什么不选其他方案?

| 方案 | 不选原因 |
|------|---------|
| Highcharts | 商业许可, 非免费 |
| D3.js 直接使用 | 学习曲线高, 量化场景 ECharts 已足够 |
| AG Grid | 社区版功能受限, 企业版付费 |
| Grafana 嵌入 | 适合基础设施监控, 不适合业务逻辑定制 (策略对比/参数管理) |

---

## 二、整体页面架构

```
┌─────────────────────────────────────────────────────────────────────┐
│  顶部状态条 (TopBar)                                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────┐ ┌──────────┐   │
│  │ NAV 净值  │ │ 日 PnL   │ │ 最大回撤  │ │ 告警数 │ │ 系统时间  │   │
│  │ $10,234  │ │ +1.2%    │ │ -5.3%    │ │ 🔴 2  │ │ UTC      │   │
│  └──────────┘ └──────────┘ └──────────┘ └───────┘ └──────────┘   │
├──────────┬──────────────────────────────────────────────────────────┤
│          │                                                          │
│  左侧    │  主内容区 (RouterView)                                    │
│  导航    │                                                          │
│  (Aside) │  根据路由渲染对应页面                                     │
│          │                                                          │
│  总览     │                                                          │
│  仓位     │                                                          │
│  收益     │                                                          │
│  风控     │                                                          │
│  订单     │                                                          │
│  策略     │                                                          │
│  信号     │                                                          │
│  波动率   │                                                          │
│  回测     │                                                          │
│  系统     │                                                          │
│          │                                                          │
└──────────┴──────────────────────────────────────────────────────────┘
```

---

## 三、页面清单与功能

| # | 页面 | 路由 | 核心组件 | 数据来源 | Phase |
|---|------|------|---------|---------|-------|
| 1 | **总览 Dashboard** | `/` | KPI 卡片 + 小 NAV 曲线 + 服务状态灯 + 最近告警 | 聚合多接口 | P1 |
| 2 | **仓位监控** | `/positions` | 目标 vs 实际仓位表格, 偏差高亮, 多空分布饼图 | `/api/v1/account/positions`, `/api/v1/portfolio/target-positions` | P1 |
| 3 | **收益监控** | `/pnl` | NAV 曲线 (Lightweight Charts), 日 PnL 柱状图, 回撤曲线 | `/api/v1/portfolio/nav`, `/api/v1/portfolio/pnl` | P1 |
| 4 | **风控面板** | `/risk` | 风控规则表 + 当前指标值 + 触发状态, 告警历史时间线 | `/api/v1/risk/*` | P2 |
| 5 | **订单审计** | `/orders` | 分页表格 (时间/symbol/方向/数量/价格/状态), 筛选器 | `/api/v1/orders/*` | P2 |
| 6 | **策略管理** | `/strategy` | 策略卡片列表, 参数只读, 启停开关, 参数变更表单 | `/api/v1/strategy/*` | P1 (只读) / P4 (写入) |
| 7 | **信号分析** | `/signals` | 覆盖率/缺失率环形图, symbol 级信号热力图 | `/api/v1/signals/*` | P3 |
| 8 | **波动率监控** | `/volatility` | 组合级/策略级/symbol级三层 Tab, ECharts 热力图 | `/api/v1/portfolio/volatility` | P3 |
| 9 | **策略对比** | `/compare` | 双策略 NAV 叠加曲线, Sharpe/MDD/Calmar 对比卡片 | `/api/v1/signals/compare` | P3 |
| 10 | **系统状态** | `/system` | 服务心跳表, 任务运行状态, 数据管线延迟 | `/api/v1/health/*` | P1 |
| 11 | **回测结果** | `/backtest` | 回测参数表单, 结果指标卡片, NAV 曲线, IS/OOS 分割线 | `/api/v1/backtest/*` | P4 |

---

## 四、前端目录结构

```
frontend/                          # 项目根目录 (monorepo 中的子目录)
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── env.d.ts
│
├── public/
│   └── favicon.ico
│
└── src/
    ├── main.ts                    # 入口: createApp + 挂载插件
    ├── App.vue                    # 根组件: Layout 骨架
    │
    ├── router/
    │   └── index.ts               # 路由定义
    │
    ├── stores/                    # Pinia stores
    │   ├── portfolio.ts           # NAV / PnL / 回撤
    │   ├── positions.ts           # 当前仓位 + 目标仓位
    │   ├── strategy.ts            # 策略列表 + 参数
    │   ├── risk.ts                # 风控状态 + 告警
    │   ├── orders.ts              # 订单历史
    │   ├── signals.ts             # 信号覆盖率
    │   ├── system.ts              # 服务心跳 + 任务状态
    │   └── volatility.ts          # 波动率数据
    │
    ├── api/                       # 后端接口封装
    │   ├── client.ts              # Axios 实例 (baseURL, 拦截器)
    │   ├── ws.ts                  # WebSocket 连接管理 (自动重连)
    │   ├── health.ts              # /api/v1/health/*
    │   ├── account.ts             # /api/v1/account/*
    │   ├── strategy.ts            # /api/v1/strategy/*
    │   ├── risk.ts                # /api/v1/risk/*
    │   ├── portfolio.ts           # /api/v1/portfolio/*
    │   ├── orders.ts              # /api/v1/orders/*
    │   ├── signals.ts             # /api/v1/signals/*
    │   └── backtest.ts            # /api/v1/backtest/*
    │
    ├── composables/               # Vue 组合式函数
    │   ├── useWebSocket.ts        # WebSocket 连接 + 自动重连
    │   ├── usePolling.ts          # 定时轮询 + 清理
    │   └── useTheme.ts            # 暗色/亮色主题切换
    │
    ├── components/                # 可复用组件
    │   ├── layout/
    │   │   ├── AppLayout.vue      # 主布局 (侧边栏 + 顶栏 + 内容区)
    │   │   ├── TopBar.vue         # 顶部 KPI 状态条
    │   │   └── SideNav.vue        # 左侧导航菜单
    │   │
    │   ├── charts/
    │   │   ├── NavChart.vue       # TradingView Lightweight Charts 封装
    │   │   ├── PnlBarChart.vue    # ECharts 日 PnL 柱状图
    │   │   ├── DrawdownChart.vue  # ECharts 回撤曲线
    │   │   ├── VolHeatmap.vue     # ECharts 波动率热力图
    │   │   └── SignalRadar.vue    # ECharts 信号雷达图
    │   │
    │   ├── tables/
    │   │   ├── PositionTable.vue  # 仓位表格 (目标 vs 实际)
    │   │   ├── OrderTable.vue     # 订单审计表格
    │   │   └── AlertTable.vue     # 告警历史表格
    │   │
    │   ├── cards/
    │   │   ├── KpiCard.vue        # 通用 KPI 卡片 (标题 + 值 + 变化率)
    │   │   ├── ServiceCard.vue    # 服务状态卡片 (名称 + 心跳 + 延迟)
    │   │   └── StrategyCard.vue   # 策略卡片 (名称 + 参数 + 状态)
    │   │
    │   └── common/
    │       ├── StatusDot.vue      # 状态指示灯 (绿/黄/红)
    │       ├── TimeAgo.vue        # 相对时间显示
    │       └── ConfirmDialog.vue  # 二次确认弹窗
    │
    ├── views/                     # 页面级组件
    │   ├── DashboardView.vue
    │   ├── PositionsView.vue
    │   ├── PnlView.vue
    │   ├── RiskView.vue
    │   ├── OrdersView.vue
    │   ├── StrategyView.vue
    │   ├── SignalsView.vue
    │   ├── VolatilityView.vue
    │   ├── CompareView.vue
    │   ├── SystemView.vue
    │   └── BacktestView.vue
    │
    └── types/                     # TypeScript 类型定义
        ├── api.ts                 # API 响应类型 (与后端 JSON Schema 对齐)
        ├── portfolio.ts           # TargetPortfolio / TargetPosition
        ├── strategy.ts            # StrategyInfo / StrategyConfig
        ├── risk.ts                # RiskRule / RiskStatus / Alert
        └── order.ts               # Order / Trade
```

---

## 五、数据流架构

```
┌───────────────────────────────────────────────────────────────┐
│                        前端 (Vue 3 SPA)                        │
│                                                               │
│  ┌─────────┐     ┌──────────┐     ┌──────────────────────┐  │
│  │  Views   │◄───│  Pinia   │◄───│  api/ + composables/ │  │
│  │  (页面)  │    │  Stores  │    │  (数据获取层)         │  │
│  └─────────┘     └──────────┘     └───────────┬──────────┘  │
│                                                │              │
│                                   ┌────────────┴───────────┐ │
│                                   │ REST (Axios) │ WS 推送  │ │
│                                   └────────────┬───────────┘ │
└────────────────────────────────────────────────┼─────────────┘
                                                 │
                                                 ▼
┌────────────────────────────────────────────────────────────────┐
│                 后端 API Gateway (FastAPI)                      │
│                                                                │
│  /api/v1/health/*      ← Redis (heartbeat keys)               │
│  /api/v1/account/*     ← Redis (account:balance, positions)   │
│  /api/v1/strategy/*    ← Redis (strategy:config) + YAML       │
│  /api/v1/risk/*        ← Redis (risk:status, alerts)          │
│  /api/v1/portfolio/*   ← Redis (portfolio:nav) + TimescaleDB  │
│  /api/v1/orders/*      ← TimescaleDB (orders table)           │
│  /api/v1/signals/*     ← Redis (signal:latest) + TimescaleDB  │
│                                                                │
│  /ws/positions         → 订阅 Redis Pub/Sub, 推送给前端        │
│  /ws/portfolio         → 定时从 Redis 读取, 推送给前端          │
│  /ws/alerts            → 订阅 Redis risk_rejected 通道         │
│  /ws/services          → 定时扫描 heartbeat keys, 推送给前端    │
└────────────────────────────────────────────────────────────────┘
```

### 数据刷新策略

| 数据类型 | 方式 | 频率 | 说明 |
|---------|------|------|------|
| 服务心跳 / 任务状态 | WebSocket `/ws/services` | 30s | 后端定时推送 |
| 当前仓位 | WebSocket `/ws/positions` | 30s | AccountService 更新后推送 |
| NAV / PnL | WebSocket `/ws/portfolio` | 60s | 每分钟计算推送 |
| 风控告警 | WebSocket `/ws/alerts` | 实时 | risk_rejected 事件触发 |
| 订单历史 | REST 轮询 | 用户刷新 | 按需加载, 分页 |
| 策略参数 | REST | 页面加载 | 低频变更, 不需推送 |
| 波动率 | REST 轮询 | 5min | 计算密集, 不适合高频推送 |
| 信号覆盖率 | REST 轮询 | 每日 | 日级汇总 |
| 回测结果 | REST | 按需 | 用户触发回测后查询 |

---

## 六、WebSocket 连接管理

### 自动重连封装

```typescript
// composables/useWebSocket.ts 设计
interface UseWebSocketOptions {
  url: string
  onMessage: (data: any) => void
  reconnectInterval?: number   // 默认 3000ms
  maxRetries?: number          // 默认 10
}

function useWebSocket(options: UseWebSocketOptions) {
  // 1. 建立连接
  // 2. 收到消息 → 解析 JSON → 调用 onMessage
  // 3. 连接断开 → 指数退避重连
  // 4. 组件卸载 → 自动关闭连接 (onUnmounted)
  // 5. 暴露 { status, send, close } 给组件
}
```

### 后端 WebSocket 端点设计

后端每个 WebSocket 端点在 FastAPI 中实现, 模式统一:

```python
@router.websocket("/ws/positions")
async def ws_positions(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            # 从 Redis 读取最新仓位
            data = await redis.get("quant:account:positions")
            await websocket.send_json(json.loads(data))
            await asyncio.sleep(30)
    except WebSocketDisconnect:
        pass
```

---

## 七、主题与样式规范

### 配色方案

| 元素 | 暗色模式 | 亮色模式 |
|------|---------|---------|
| 背景 | `#0d1117` | `#ffffff` |
| 卡片背景 | `#161b22` | `#f6f8fa` |
| 正值 (盈利/多头) | `#3fb950` (绿) | `#1a7f37` |
| 负值 (亏损/空头) | `#f85149` (红) | `#cf222e` |
| 主色调 | `#58a6ff` (蓝) | `#0969da` |
| 警告 | `#d29922` (黄) | `#9a6700` |
| 文本主色 | `#e6edf3` | `#1f2328` |
| 文本次色 | `#8b949e` | `#656d76` |

### 响应式断点

| 断点 | 宽度 | 布局 |
|------|------|------|
| Desktop | >= 1280px | 侧边栏 + 完整内容区 |
| Tablet | 768-1279px | 折叠侧边栏 + 内容区 |
| Mobile | < 768px | 底部导航 + 全宽内容 (优先级低) |

---

## 八、部署方案

### 方案 A: S3 + CloudFront (推荐)

```
GitHub Actions
    │
    ├── npm run build → 生成 dist/
    │
    ├── aws s3 sync dist/ s3://quant-dashboard-{env}/
    │
    └── aws cloudfront create-invalidation → 刷新 CDN 缓存

成本: ~$1-3/月 (极低流量)
优势: 零运维, 全球 CDN, HTTPS 自动
```

### 方案 B: Nginx 容器 (与后端同 ECS 集群)

```
Dockerfile.frontend:
    FROM node:20-alpine AS build
    COPY . .
    RUN npm ci && npm run build

    FROM nginx:alpine
    COPY --from=build /app/dist /usr/share/nginx/html
    COPY nginx.conf /etc/nginx/conf.d/default.conf

部署为 ECS Fargate Service, 256 CPU / 512MB
通过 ALB 路由: /api/* → 后端, /* → 前端
```

### 推荐: 开发阶段用 Vite dev server (代理后端), 生产用方案 A。

### CORS 配置

后端 `gateway.py` 已有 CORS 中间件 (allow_origins=["*"]), 生产环境应限制为:

```python
allow_origins=[
    "https://dashboard.quant-trading.example.com",
    "http://localhost:5173",  # Vite 开发服务器
]
```

---

## 九、与后端架构对接点

| 后端服务 (architecture.md) | 前端对接方式 | 前端页面 |
|---------------------------|-------------|---------|
| AccountService | REST `/api/v1/account/*` + WS `/ws/positions` | 仓位监控, 总览 |
| StrategyService | REST `/api/v1/strategy/*` | 策略管理, 策略对比 |
| RiskService | REST `/api/v1/risk/*` + WS `/ws/alerts` | 风控面板 |
| OrderService | REST `/api/v1/orders/*` | 订单审计 |
| MonitorService | REST `/api/v1/health/*` + WS `/ws/services` | 系统状态, 总览 |
| FeatureService | REST `/api/v1/signals/*` | 信号分析 |
| BacktestEngine | REST `/api/v1/backtest/*` | 回测结果 |
| API Gateway (FastAPI) | 所有接口的统一入口 | 全部页面 |

**关键**: 前端不直接访问 Redis 或交易所 API。所有数据通过 API Gateway 中转, 后端从 Redis 读取缓存数据。
