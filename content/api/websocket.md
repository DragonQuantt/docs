# WebSocket API 参考

本页采用最新 API 契约，与 `frontend/01_api_design.md` 保持一致。
说明：本页仅描述对外 WebSocket 推送；服务间内部事件通信请参考 [Redis 事件总线](redis.md)。

## 连接端点（开发环境）

- `ws://localhost:8000/ws/positions`
- `ws://localhost:8000/ws/portfolio`
- `ws://localhost:8000/ws/alerts`
- `ws://localhost:8000/ws/services`

## 通用消息结构

所有推送消息统一为：

```json
{
  "type": "positions_update",
  "data": {},
  "timestamp": "2026-03-02T23:59:00Z"
}
```

客户端心跳：

```json
{ "type": "ping" }
```

服务端响应：

```json
{ "type": "pong", "timestamp": "2026-03-02T23:59:00Z" }
```

## 端点说明

### `/ws/positions`

- 推送频率：每 30 秒
- 数据来源：`quant:account:positions` + `quant:signal:latest`
- 用途：仓位快照与目标偏差

示例：

```json
{
  "type": "positions_update",
  "data": {
    "positions": [
      {
        "symbol": "BTC/USDT:USDT",
        "side": "long",
        "actual_weight": 0.0155,
        "target_weight": 0.0167,
        "deviation_pct": -7.2
      }
    ]
  },
  "timestamp": "2026-03-02T23:59:00Z"
}
```

### `/ws/portfolio`

- 推送频率：每 60 秒
- 数据来源：`quant:portfolio:snapshot`
- 用途：NAV、PnL、回撤、波动率核心指标

### `/ws/alerts`

- 推送频率：实时（事件触发）
- 数据来源：Pub/Sub `quant:risk_rejected`, `quant:order_failed`
- 用途：风控与执行告警

### `/ws/services`

- 推送频率：每 30 秒
- 数据来源：`quant:heartbeat:{service}`
- 用途：服务健康与延迟状态

## 重连建议

- 客户端收到 `close/error` 后做指数退避重连
- 建议退避窗口 `3s -> 6s -> 12s`，并设置最大重试次数
