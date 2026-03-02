# WebSocket API 参考

WebSocket 实时数据接口文档。

## 连接端点

```
ws://localhost:8080/ws
```

使用 CCXT Pro WebSocket 实现，提供实时市场数据流。

---

## 查询参数

| 参数 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `symbol` | string | 否 | BTC/USDT | 交易对符号（如 BTC/USDT） |
| `exchange` | string | 否 | binance | CCXT 交易所 ID |
| `test_mode` | boolean | 否 | false | 测试模式（使用模拟数据） |

**说明**:

- `symbol`: 必须是交易所支持的标准交易对格式（如 `BTC/USDT`、`ETH/USDT`）
- `exchange`: 必须是 CCXT Pro 支持的交易所 ID（如 `binance`、`okx`、`bybit`）
- `test_mode`: 开启后将返回模拟数据，无需实际连接交易所

---

## 连接示例

### 基本连接

```
ws://localhost:8080/ws?symbol=BTC/USDT&exchange=binance
```

### 测试模式连接

```
ws://localhost:8080/ws?symbol=ETH/USDT&exchange=binance&test_mode=true
```

### 不同交易所

```
ws://localhost:8080/ws?symbol=BTC/USDT&exchange=okx
ws://localhost:8080/ws?symbol=ETH/USDT&exchange=bybit
```

---

## 消息格式

### Ticker 数据

服务器推送的实时行情数据格式：

```json
{
  "type": "ticker",
  "symbol": "BTC/USDT",
  "data": {
    "price": "43234.56",
    "volume": "156.78",
    "ts": 1705676130
  }
}
```

**字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | string | 消息类型（固定为 "ticker"） |
| `symbol` | string | 交易对符号 |
| `data.price` | string | 最新成交价（字符串格式，保留精度） |
| `data.volume` | string | 24小时成交量（优先报价币种成交量） |
| `data.ts` | integer | Unix 时间戳（秒） |

### 错误消息

连接或数据获取过程中的错误信息：

```json
{
  "type": "error",
  "message": "Network error: Connection timeout"
}
```

**常见错误类型**:

- `Network error: ...` - 网络连接问题（自动重试，5秒后重连）
- `Exchange error: ...` - 交易所 API 错误（自动重试，5秒后重连）
- `Unexpected error: ...` - 未知错误（自动重试，5秒后重连）

**说明**:

- 服务器端会自动处理错误并尝试重连
- 客户端收到错误消息后可以选择等待自动恢复或主动断开重连
- WebSocket 断开后需要客户端主动重新建立连接

---

## 客户端示例

### Python (websockets)

```python
import asyncio
import websockets
import json

async def connect_ticker():
    uri = "ws://localhost:8080/ws?symbol=BTC/USDT&exchange=binance"
    
    async with websockets.connect(uri) as websocket:
        print("✅ 已连接到 WebSocket")
        
        try:
            async for message in websocket:
                data = json.loads(message)
                
                if data['type'] == 'ticker':
                    ticker = data['data']
                    print(f"[{data['symbol']}] 价格: {ticker['price']}, "
                          f"成交量: {ticker['volume']}, 时间: {ticker['ts']}")
                
                elif data['type'] == 'error':
                    print(f"❌ 错误: {data['message']}")
                    
        except websockets.exceptions.ConnectionClosed:
            print("❌ 连接已关闭")
        except KeyboardInterrupt:
            print("⚠️ 用户中断")

# 运行客户端
asyncio.run(connect_ticker())
```

### JavaScript (浏览器)

```javascript
// 建立 WebSocket 连接
const ws = new WebSocket('ws://localhost:8080/ws?symbol=BTC/USDT&exchange=binance');

// 连接建立
ws.onopen = () => {
    console.log('✅ WebSocket 已连接');
};

// 接收消息
ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    
    if (data.type === 'ticker') {
        const ticker = data.data;
        console.log(`[${data.symbol}] 价格: ${ticker.price}, 成交量: ${ticker.volume}`);
        
        // 更新 UI
        document.getElementById('price').textContent = ticker.price;
        document.getElementById('volume').textContent = ticker.volume;
        
    } else if (data.type === 'error') {
        console.error('❌ 错误:', data.message);
    }
};

// 连接错误
ws.onerror = (error) => {
    console.error('❌ WebSocket 错误:', error);
};

// 连接关闭
ws.onclose = () => {
    console.log('❌ WebSocket 已断开');
    // 实现重连逻辑
    setTimeout(() => {
        console.log('🔄 尝试重连...');
        // 重新创建连接
    }, 5000);
};
```