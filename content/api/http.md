# HTTP API 参考

本页采用最新 API 契约，与 `frontend/01_api_design.md` 保持一致。
说明：本页仅描述对外 REST 接口；服务间内部事件通信请参考 [Redis 事件总线](redis.md)。

## 基础信息

- **开发环境 Base URL**: `http://localhost:8000`
- **生产环境 Base URL**: `https://api.quant-trading.example.com`
- **协议**: HTTP/1.1
- **格式**: `application/json`

## 统一响应格式

成功响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {},
  "timestamp": "2026-03-02T12:00:00Z"
}
```

失败响应：

```json
{
  "code": 40001,
  "message": "Invalid request",
  "data": null,
  "timestamp": "2026-03-02T12:00:00Z"
}
```

错误码范围：

| 范围 | 含义 |
|------|------|
| `0` | 成功 |
| `40000-40099` | 请求参数错误 |
| `40100-40199` | 认证/授权错误 |
| `40400-40499` | 资源不存在 |
| `50000-50099` | 服务内部错误 |
| `50300-50399` | 上游服务不可用（Redis/交易所） |

## OpenAPI

- `GET /docs` - Swagger UI
- `GET /openapi.json` - OpenAPI Schema

浏览器访问：`http://localhost:8000/docs`

## 核心端点（分组）

### Health

- `GET /api/v1/health`
- `GET /api/v1/health/pipeline`

### Account

- `GET /api/v1/account/balance`
- `GET /api/v1/account/positions`
- `GET /api/v1/account/open-orders`

### Strategy

- `GET /api/v1/strategy/list`
- `GET /api/v1/strategy/{name}/config`
- `POST /api/v1/strategy/{name}/toggle`
- `POST /api/v1/strategy/{name}/config`
- `GET /api/v1/strategy/{name}/config/history`

### Risk

- `GET /api/v1/risk/status`
- `GET /api/v1/risk/alerts`
- `GET /api/v1/risk/config`
- `POST /api/v1/risk/symbols/disable`

### System

- `POST /api/v1/system/emergency-stop`

### Portfolio

- `GET /api/v1/portfolio/nav`
- `GET /api/v1/portfolio/pnl`
- `GET /api/v1/portfolio/drawdown`
- `GET /api/v1/portfolio/target-positions`
- `GET /api/v1/portfolio/position-deviation`
- `GET /api/v1/portfolio/volatility`

### Orders

- `GET /api/v1/orders/history`
- `GET /api/v1/orders/rebalance-history`
- `GET /api/v1/orders/trades`

### Signals

- `GET /api/v1/signals/latest`
- `GET /api/v1/signals/coverage`
- `GET /api/v1/signals/compare`

### Backtest

- `POST /api/v1/backtest/run`
- `GET /api/v1/backtest/{backtest_id}/result`
- `GET /api/v1/backtest/history`

## CORS 配置

开发环境可使用宽松策略；生产环境应限制 `allow_origins` 到前端实际域名。

## 兼容性说明

以下旧契约已废弃，不再作为主文档：

- `GET /health`
- `GET /v1/spot/top{top_k}/{days}days`
