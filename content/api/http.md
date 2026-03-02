# HTTP API 参考

RESTful HTTP 接口文档。

## 基础信息

- **Base URL**: `http://localhost:8080`
- **协议**: HTTP/1.1
- **格式**: JSON

## 通用端点

### 健康检查

```
GET /health
```

检查服务是否正常运行。

**响应**:

```json
{
  "status": "healthy"
}
```

**状态码**: `200 OK`

---

### 根端点

```
GET /
```

获取网关基本信息。

**响应**:

```json
{
  "message": "Quant Trading Gateway",
  "status": "running"
}
```

**状态码**: `200 OK`

---

## API 文档

### OpenAPI 文档

```
GET /docs
```

交互式 API 文档（Swagger UI），提供可视化的接口测试界面。

**浏览器访问**: `http://localhost:8080/docs`

---

### OpenAPI Schema

```
GET /openapi.json
```

获取 OpenAPI 3.0 规范文件。

---

## 资产池 API

### 获取现货交易对排名

```
GET /v1/spot/top{top_k}/{days}days
```

根据过去 N 天的成交额获取 Top K 现货交易对列表。

**路径参数**:

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `top_k` | integer | 是 | 返回前 K 个交易对（必须 > 0） |
| `days` | integer | 是 | 统计天数（必须 > 0） |

**查询参数**:

| 参数 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `exchange` | string | 否 | binance | CCXT 交易所 ID |
| `quote_asset` | string | 否 | USDT | 计价币种过滤（如 USDT） |
| `max_workers` | integer | 否 | 8 | 线程数（1-32） |

**请求示例**:

```bash
# 获取币安交易所过去 30 天成交额前 10 的 USDT 计价交易对
GET /v1/spot/top10/30days?exchange=binance&quote_asset=USDT

# 获取 OKX 交易所过去 7 天成交额前 20 的交易对
GET /v1/spot/top20/7days?exchange=okx&max_workers=16
```

**响应示例**:

```json
{
  "exchange": "binance",
  "days": 30,
  "top_k": 10,
  "count": 10,
  "items": [
    {
      "symbol": "BTC/USDT",
      "base": "BTC",
      "quote": "USDT",
      "turnover": 125678901234.56
    },
    {
      "symbol": "ETH/USDT",
      "base": "ETH",
      "quote": "USDT",
      "turnover": 98765432109.87
    }
  ]
}
```

**响应字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `exchange` | string | 交易所 ID |
| `days` | integer | 统计天数 |
| `top_k` | integer | 请求的数量 |
| `count` | integer | 实际返回数量 |
| `items` | array | 交易对列表 |
| `items[].symbol` | string | 交易对符号（如 BTC/USDT） |
| `items[].base` | string | 基础币种 |
| `items[].quote` | string | 计价币种 |
| `items[].turnover` | float | 成交额 |

**状态码**:

- `200` - 成功
- `400` - 参数错误（如 top_k ≤ 0 或 days ≤ 0）
- `500` - 服务器错误

---

## CORS 配置

API 已启用 CORS（跨域资源共享），允许所有来源访问：

- **允许来源**: `*`（所有来源）
- **允许凭证**: `true`
- **允许方法**: `*`（所有 HTTP 方法）
- **允许头部**: `*`（所有请求头）

---

## 错误响应

标准错误格式：

```json
{
  "detail": "错误描述信息"
}
```

**常见错误示例**:

```json
{
  "detail": "top_k must be greater than 0"
}
```

```json
{
  "detail": "days must be greater than 0"
}
```

**HTTP 状态码**:

- `200` - 成功
- `400` - 请求错误（参数验证失败）
- `404` - 资源未找到
- `422` - 实体无法处理（参数格式错误）
- `500` - 服务器内部错误
