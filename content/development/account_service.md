# Account Service

## 对外通信
- GET  /api/v1/account/nav/snapshots?from=&to=  # 历史净值快照
- WS   /ws/account/portfolio                     # 实时推送完整 portfolio 状态
- GET  /api/v1/account/health                    # 健康检查
- GET /api/v1/portfolio/target-positions        # 策略输出的目标仓位。
## 内部 Pipeline
- 维护内存中最新 portfolio state
- portfolio 发生变化时发布到 quant:account_updated
- 其他服务（风控/信号/决策）订阅该 channel 或直接读 Redis 缓存