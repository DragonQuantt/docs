# Phase A 实施计划

> 编写时间: 2026-06-11
> 修订: 2026-06-11 **策略清零修订**——baseline_rev 与 crypto_pairs_mean_reversion 已从代码移除；唯一策略 monthly_majors_vs_alts（含信号契约与 Risk 真实 sizing）已在分支 `feat/majors_vs_alts_vol_target` 落地。本文按实施进度更新各项状态。
> 状态: **实施中**（A1/A2/A5 已完成）

---

## 一、目标

打通**唯一策略 [monthly_majors_vs_alts](../overview/strategy_5.md)**（月度大币-山寨轮动 + vol targeting，[已实现](../overview/strategy_5.md)）从 **信号 → sizing → 校验 → 真实下单 → 记录 → 可视化** 的真实交易闭环，全程 testnet 验证后切小资金实盘。

> 原计划的"双策略同机运行"（与 crypto pairs 并行）随 2026-06-11 策略清零取消；crypto pairs 实现与文档均已删除（git 历史可查），多策略并行机制（虚拟账本、策略级开关的多策略语义）转为未来预案。

Phase A 结束的系统标准：**策略真实运行、所有不可回填的数据在积累、出事会推送到手机、30 秒内可人工停止发单。**

## 二、范围与状态（13 项）

### G1 交易闭环核心（后端）

| # | 工作项 | 状态 | 内容 |
|---|--------|------|------|
| A1 | 信号契约升级 | ✅ **已完成（2026-06-11）** | `StrategySignal.target_weight`；批次 `signal_type` 与一等 `rebalance_id`（确定性 `rb-{strategy}-{YYYYMMDDTHHMM}`，重试同 id） |
| A2 | Risk 真实 sizing | ✅ **已完成（2026-06-11）** | `notional = |weight| × equity`；equity 读 `quant:account:balance`（过期回退 `default_equity_usdt`）；minNotional 剔腿；`max_gross_exposure` 比例缩减；`risk_rejected` 已实际发布。**余项归入 A10**：急停检查 |
| A3 | OrderService | ⬜ 待实施 | 订阅 `execution_approved`；差量调仓（先平后开）；ccxt 精度处理；幂等三道闸（SETNX + DB 唯一约束 + clientOrderId）；**默认 `dry_run: true`**；发单前急停二次检查。单策略阶段差量可直接对账户净持仓计算，**虚拟账本转为多策略预案** |
| A4 | 调度升级 | 🔽 **范围缩减** | croniter/`cron_schedule` **暂不需要**——现有日频调度器已满足（策略内部判断 30 天网格日）。仍需实施：**宕机错过补偿（catch_up）** |
| A5 | 策略本体 | ✅ **已完成（2026-06-11）** | monthly_majors_vs_alts：REST 日线、30 天网格、每日止损检查、NAV vol targeting、无状态重推；163 测试 + 真实行情冒烟通过 |
| A6 | 命名资产池 | ❌ **取消** | 策略 self-feeding 自发现宇宙（24h 成交额预筛 + 30 日 dvol 排名），不依赖资产池服务。asset_pool 服务保留供数据采集链路使用 |
| A7 | 持久化三表 | ⬜ 待实施 | `signal_records` / `rebalance_records` / `orders` 模型 + repository（表结构见[数据库设计](../architecture/database.md)） |
| A8 | Execution 修整 | ⬜ 待实施（范围修订） | 全批拒绝 → **按腿独立校验**（月度篮子单腿盘口问题不应否决整批；pair 原子组语义随策略退役转为预案）；订单簿快照过期回退 REST |

### G2 数据与可观测

| # | 工作项 | 状态 | 内容 |
|---|--------|------|------|
| A9 | ArchiveService | ⬜ 待实施 | orderbook 快照 + `bar_normalized` → 按日 parquet → S3。优先级：orderbook（不可回填）> bar > aggTrades（不落，Binance Vision 可回填） |
| A10 | AlertService + 急停 | ⬜ 待实施 | 聚合 `risk_rejected` / `execution_rejected` / `order_failed` + 心跳超时 → Redis 报警历史 + **Telegram 推送**；急停全链路（API 置位 → Risk 拒批 → Order 二次检查）；策略开关 `quant:strategy:enabled:{name}` |

### G3 API 与前端

| # | 工作项 | 状态 | 内容 |
|---|--------|------|------|
| A11 | API 补全 | ⬜ 待实施 | `GET /signals/latest|history`、`GET /orders/history|rebalance-history(/{id})`、`POST /system/emergency-stop`、`GET /risk/alerts`、`GET /portfolio/target-positions` |
| A12 | 前端接线 | ⬜ 待实施 | 5 个现有页面接真数据 + RebalanceView 审计页 + 顶栏急停按钮与 dry_run/testnet/live 环境徽章；实时 NAV 复用 `WS /api/v1/account/portfolio/ws` |

### G4 基础设施

| # | 工作项 | 状态 | 内容 |
|---|--------|------|------|
| A13 | 部署 | ⬜ 待实施 | 单机 EC2 t4g.medium（东京）+ docker compose；SSM Parameter Store；S3；CI arm64 镜像；Telegram bot；CloudWatch + Budgets（详见[部署运维](../deployment/index.md)） |

## 三、关键架构决策（按 2026-06-11 修订）

1. **策略输出统一为目标权重契约**（✅ 已实现）：`signal_type=target_weight`，Risk/Execution/Order 策略无关。
2. **确定性 `rebalance_id`**（✅ 已实现）：策略名 + 计划触发时刻，重启重试同 id，幂等贯穿全链。
3. **策略虚拟账本** → **降级为多策略预案**：当前单策略，OrderService 差量直接对账户净持仓即可；未来第二个策略上线前必须启用（设计见 [architecture](../architecture/index.md) 多策略小节）。
4. **月度策略不依赖流式特征链**（✅ 已实现并强化为无状态设计）：每次运行从 REST 日线全量重推网格/入场价/止损/杠杆，重启不失同步，实盘 ≡ 回测口径。
5. **归档优先级 orderbook > bar > aggTrades**（不变）。
6. **cron 放进程内**（不变，且 croniter 暂不需要——日频调度器已满足）。
7. **前端定位审计型监控**（不变）。

## 四、与既有计划的映射

| 既有计划 | Phase A 覆盖 |
|----------|--------------|
| [后端 Sprint](back_end_sprint.md) V1 Sprint 1（最短闭环） | 服务链已实现，OrderService 为唯一余项 |
| V1 Sprint 3（恢复 + 幂等） | rebalance_id ✅；调度 catch_up 待做 |
| V1 Sprint 4（风控 + 执行可靠） | equity sizing ✅；急停、报警待做 |
| INTEGRATION_PLAN Phase 1.1 / 1.3 | ArchiveService + 三表，待做 |
| V1 Sprint 5/6、V2 | 不在 Phase A 范围 |

## 五、验收标准

1. testnet 上完整跑通一次月度调仓，RebalanceView 审计页能解释每一腿的最终成交（滑点、手续费、before/after 权重）
2. 调仓触发时刻杀进程再重启 → 自动补跑且**不重复下单**（幂等 + catch_up 验证）
3. 急停置位后：新信号批被 Risk 拒绝，在途批被 Order 拦截
4. 断开 Binance WS → 5 分钟内 Telegram 收到心跳报警
5. 实盘归档的 bar 与 T+1 从 data.binance.vision 重算的 bar 对账一致
6. ~~双策略互不踩仓位~~ → 改为：每日止损检查日与网格调仓日均正确产生/不产生差量订单（已部分验证：真实行情冒烟中止损与杠杆均正确）

## 六、明确不在 Phase A 范围

动态 T50 流动性池、回测一致性验证工具（INTEGRATION_PLAN Phase 2）、JWT 用户体系、多策略 ensemble 与虚拟账本启用、**crypto pairs 重新上线**、策略 2（dollar bar + ML）全链路、自动平仓（急停只停新单）。

## 七、文档变更索引

- 2026-06-11 第一轮：docs 全面对齐代码 + Phase A 标注（见 git 历史）
- 2026-06-11 第二轮（策略清零）：本文状态更新；[strategy_5.md](../overview/strategy_5.md) 由框架稿改实现稿；strategy_crypto_pairs.md 与策略 1~4 研究稿文档全部删除（git 历史可查）；[architecture/index.md](../architecture/index.md)、[api/redis.md](../api/redis.md)、[api/cli.md](../api/cli.md)、[configuration.md](../getting-started/configuration.md)、[back_end_sprint.md](back_end_sprint.md) 同步对齐
