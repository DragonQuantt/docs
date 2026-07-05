# 账户注册与管理

用户注册、登录认证、交易账户创建与管理的完整流程。

相关文件：

1. `src/quant_trading/controller/routes/auth/auth.py`
2. `src/quant_trading/controller/routes/account_mgmt/account_mgmt.py`
3. `src/quant_trading/common/utils/auth.py`
4. `src/quant_trading/common/utils/crypto.py`
5. `src/quant_trading/controller/dependencies.py`

---

## 架构概览

```
用户注册/登录                          交易账户管理
POST /api/v1/auth/register            POST   /api/v1/accounts
POST /api/v1/auth/login               GET    /api/v1/accounts
         │                            GET    /api/v1/accounts/{id}
         │  JWT access_token          PUT    /api/v1/accounts/{id}
         ▼                            DELETE /api/v1/accounts/{id}
┌──────────────────┐                         │
│  users 表        │◄── owner_id ──► accounts 表
│  email           │                 │ exchange_id
│  password_hash   │                 │ api_key_enc (AES-256-GCM)
│  role            │                 │ api_secret_enc
└──────────────────┘                 └────────────────────────
```

---

## 1. 用户注册

### 请求

```
POST /api/v1/auth/register
Content-Type: application/json
```

```json
{
  "email": "tom@example.com",
  "display_name": "Tom",
  "password": "your-password",
  "role": "admin"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `email` | string | YES | 登录邮箱，全局唯一 |
| `display_name` | string | YES | 显示名称 |
| `password` | string | YES | 密码（服务端 bcrypt 哈希后存储） |
| `role` | string | NO | `admin` / `trader` / `viewer`，默认 `viewer` |

### 响应

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": 1,
    "email": "tom@example.com",
    "display_name": "Tom",
    "role": "admin",
    "is_active": true
  }
}
```

### 错误码

| code | 说明 |
|------|------|
| `40900` | 邮箱已注册 |

---

## 2. 用户登录

### 请求

```
POST /api/v1/auth/login
Content-Type: application/json
```

```json
{
  "email": "tom@example.com",
  "password": "your-password"
}
```

### 响应

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "bearer",
    "user_id": 1,
    "role": "admin"
  }
}
```

### 错误码

| code | 说明 |
|------|------|
| `40100` | 邮箱或密码错误 |
| `40300` | 账号已停用 |

### Token 使用

后续所有需要认证的请求，在 Header 中携带：

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

Token 有效期 24 小时，过期后需重新登录。

---

## 3. 创建交易账户

### 请求

```
POST /api/v1/accounts
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "account_id": "binance_main",
  "account_name": "Binance 主账户",
  "exchange_id": "binance",
  "exchange_type": "future",
  "is_sandbox": true,
  "base_ccy": "USDT",
  "api_key": "your-binance-api-key",
  "api_secret": "your-binance-api-secret",
  "key_type": "hmac"
}
```

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `account_id` | string | YES | | 唯一标识，如 `binance_main`、`okx_sub1` |
| `account_name` | string | YES | | 显示名称 |
| `exchange_id` | string | NO | `binance` | ccxt 交易所 ID：`binance`、`okx`、`bybit` 等 |
| `exchange_type` | string | NO | `future` | `spot` / `future` / `margin` |
| `is_sandbox` | bool | NO | `true` | 是否使用测试网 |
| `base_ccy` | string | NO | `USDT` | 基准币种 |
| `api_key` | string | YES | | 交易所 API Key（明文传入，服务端加密存储） |
| `api_secret` | string | YES | | 交易所 API Secret（明文传入，服务端加密存储） |
| `key_type` | string | NO | `hmac` | `hmac` / `rsa`（Binance RSA 签名模式） |

### 权限

`admin` 或 `trader` 角色。创建后 `owner_id` 自动绑定为当前登录用户。

### 响应

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "account_id": "binance_main",
    "account_name": "Binance 主账户",
    "owner_id": 1,
    "exchange_id": "binance",
    "exchange_type": "future",
    "is_sandbox": true,
    "key_type": "hmac",
    "has_credentials": true,
    "base_ccy": "USDT",
    "status": "active",
    "status_message": null,
    "last_connected_at": null,
    "created_at": "2026-03-14T10:00:00Z",
    "updated_at": "2026-03-14T10:00:00Z"
  }
}
```

> **安全说明**：API Key 和 Secret 使用 AES-256-GCM 加密后存储在 `accounts` 表的 `api_key_enc` / `api_secret_enc` 字段中。响应中不返回凭据原文，仅通过 `has_credentials: true` 表示已配置。解密需要 `CREDENTIAL_MASTER_KEY` 环境变量。

### 错误码

| code | 说明 |
|------|------|
| `40900` | account_id 已存在 |

---

## 4. 查看账户列表

### 请求

```
GET /api/v1/accounts
Authorization: Bearer <token>
```

### 权限

- `admin`：返回所有账户
- `trader` / `viewer`：仅返回自己名下的账户

### 响应

```json
{
  "code": 0,
  "message": "success",
  "data": [
    {
      "account_id": "binance_main",
      "account_name": "Binance 主账户",
      "owner_id": 1,
      "exchange_id": "binance",
      "exchange_type": "future",
      "is_sandbox": true,
      "key_type": "hmac",
      "has_credentials": true,
      "base_ccy": "USDT",
      "status": "active",
      "status_message": null,
      "last_connected_at": "2026-03-14T12:30:00Z",
      "created_at": "2026-03-14T10:00:00Z",
      "updated_at": "2026-03-14T12:30:00Z"
    },
    {
      "account_id": "okx_test",
      "account_name": "OKX 测试",
      "owner_id": 1,
      "exchange_id": "okx",
      "exchange_type": "future",
      "is_sandbox": true,
      "key_type": "hmac",
      "has_credentials": true,
      "base_ccy": "USDT",
      "status": "active",
      "status_message": null,
      "last_connected_at": null,
      "created_at": "2026-03-14T11:00:00Z",
      "updated_at": "2026-03-14T11:00:00Z"
    }
  ]
}
```

---

## 5. 查看单个账户

### 请求

```
GET /api/v1/accounts/{account_id}
Authorization: Bearer <token>
```

### 权限

`admin` 可查看任意账户，其他角色仅可查看自己名下的。

### 错误码

| code | 说明 |
|------|------|
| `40400` | 账户不存在 |
| `40300` | 无权访问该账户 |

---

## 6. 修改账户

### 请求

```
PUT /api/v1/accounts/{account_id}
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "account_name": "新名称",
  "api_key": "new-api-key",
  "api_secret": "new-api-secret"
}
```

所有字段均为可选，仅传需要修改的字段。更换 API Key 时需同时传 `api_key` 和 `api_secret`。

### 权限

`admin` 或 `trader`，且仅可修改自己名下的账户（admin 无此限制）。

---

## 7. 删除账户

### 请求

```
DELETE /api/v1/accounts/{account_id}
Authorization: Bearer <token>
```

### 权限

仅 `admin`。

### 错误码

| code | 说明 |
|------|------|
| `40400` | 账户不存在 |

---

## 角色权限总结

| 操作 | admin | trader | viewer |
|------|-------|--------|--------|
| 注册用户 | YES | YES | YES |
| 登录 | YES | YES | YES |
| 创建交易账户 | YES | YES | NO |
| 查看账户列表 | 全部 | 自己的 | 自己的 |
| 修改账户 | 任意 | 自己的 | NO |
| 删除账户 | YES | NO | NO |

---

## 环境变量

| 变量 | 说明 |
|------|------|
| `CREDENTIAL_MASTER_KEY` | AES-256-GCM 加密密钥（base64 编码的 32 字节随机值），同时用于 JWT 签名 |

生成方式：

```bash
python -c "import secrets,base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

---

## 完整使用示例

```bash
# 1. 注册管理员
curl -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@example.com",
    "display_name": "Admin",
    "password": "secure-password",
    "role": "admin"
  }'

# 2. 登录获取 token
TOKEN=$(curl -s -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "secure-password"}' \
  | python -c "import sys,json; print(json.load(sys.stdin)['data']['access_token'])")

# 3. 创建 Binance 交易账户
curl -X POST http://localhost:8000/api/v1/accounts \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": "binance_main",
    "account_name": "Binance 主力账户",
    "exchange_id": "binance",
    "exchange_type": "future",
    "is_sandbox": false,
    "api_key": "your-api-key",
    "api_secret": "your-api-secret"
  }'

# 4. 查看账户列表
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/v1/accounts
```
