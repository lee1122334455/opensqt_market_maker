# OpenSQT Market Maker 系统学习指南

> 本文档由代码审计自动生成，帮助你理解做市商系统的架构和运行逻辑。

---

## 目录

1. [安全审计报告](#安全审计报告)
2. [系统架构总览](#系统架构总览)
3. [功能模块详解](#功能模块详解)
4. [做市商策略原理](#做市商策略原理)
5. [核心代码流程](#核心代码流程)
6. [风控机制详解](#风控机制详解)
7. [快速参考](#快速参考)

---

## 安全审计报告

### 审计结论：代码安全，无后门

经过对所有代码的全面审计，确认这是一个正规的开源做市商项目，代码安全可信。

### 安全检查清单

| 检查项 | 结果 | 说明 |
|--------|------|------|
| 硬编码后门地址 | ✅ 无 | 无隐藏的资金转账地址 |
| 隐藏API调用 | ✅ 无 | 所有API调用都是标准交易所接口 |
| 数据外泄 | ✅ 无 | 无向外部服务器发送敏感数据的代码 |
| 恶意订单逻辑 | ✅ 无 | 订单逻辑透明，仅执行用户配置的策略 |
| 密钥窃取 | ✅ 无 | API密钥仅用于交易所认证，不上传 |

### 代码特点说明

#### 返佣标识（合法商业行为）

文件位置：`utils/orderid.go:115-154`

```go
// 币安返佣前缀: x-zdfVM8vY (10字符)
prefix := "x-zdfVM8vY"  // Binance broker ID
// Gate.io 返佣前缀: t- (2字符)
prefix := "t-"
```

**说明**：这是交易所的返佣机制，用户通过此程序下单时，开发者会获得交易所返佣。这是正常的商业模式，不影响用户资金安全，用户的交易费率不会因此增加。

### 安全建议

1. **敏感信息保护**
   - 将 `config.yaml` 加入 `.gitignore`
   - 考虑支持环境变量读取密钥

2. **日志文件权限**
   - 建议将日志文件权限设置为 `0600`（当前为 `0644`）

### 代码质量评估

| 方面 | 评分 | 说明 |
|------|------|------|
| 错误处理 | ⭐⭐⭐⭐ | 完善的错误处理和重试机制 |
| 并发安全 | ⭐⭐⭐⭐⭐ | 正确使用 sync.Map、RWMutex、atomic |
| 风控设计 | ⭐⭐⭐⭐⭐ | 多层风控：启动检查、实时监控、对账、订单清理 |
| 日志记录 | ⭐⭐⭐⭐ | 详细的中文日志，便于调试 |

---

## 系统架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OpenSQT Market Maker                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │   配置加载    │───▶│   交易所适配  │───▶│     核心交易引擎      │  │
│  │ config.yaml  │    │ IExchange    │    │ SuperPositionManager │  │
│  └──────────────┘    └──────────────┘    └──────────────────────┘  │
│                              │                      │               │
│                              ▼                      ▼               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │   价格监控    │    │   订单执行    │    │      风控系统        │  │
│  │ PriceMonitor │◀──▶│ OrderExecutor│    │  RiskMonitor/Safety  │  │
│  └──────────────┘    └──────────────┘    └──────────────────────┘  │
│         │                   │                      │               │
│         └───────────────────┴──────────────────────┘               │
│                            │                                        │
│                     WebSocket 实时流                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 目录结构

```
opensqt_market_maker/
├── main.go                 # 入口文件，组件编排
├── config/
│   └── config.go           # 配置加载和验证
├── exchange/
│   ├── interface.go        # 交易所统一接口
│   ├── types.go            # 通用数据类型
│   ├── factory.go          # 交易所工厂
│   ├── binance/            # 币安适配器
│   ├── bitget/             # Bitget适配器
│   └── gate/               # Gate.io适配器
├── monitor/
│   └── price_monitor.go    # 价格监控
├── position/
│   └── super_position_manager.go  # 核心交易引擎
├── order/
│   └── executor_adapter.go # 订单执行器
├── safety/
│   ├── safety.go           # 启动安全检查
│   ├── risk_monitor.go     # 实时风控监控
│   ├── reconciler.go       # 持仓对账
│   └── order_cleaner.go    # 订单清理
├── logger/
│   └── logger.go           # 日志系统
├── utils/
│   └── orderid.go          # 订单ID生成
└── config.example.yaml     # 配置模板
```

---

## 功能模块详解

### 1. 配置模块 (`config/`)

**文件**: `config.go`

**功能**：
- 加载 `config.yaml` 配置文件
- 验证配置参数有效性
- 支持多交易所配置切换

**关键配置项**：

```yaml
trading:
  symbol: "ETHUSDT"         # 交易对
  price_interval: 2         # 网格间距（美元）
  order_quantity: 30        # 每单金额（USDT）
  buy_window_size: 10       # 买单窗口数量
  sell_window_size: 10      # 卖单窗口数量
```

---

### 2. 交易所抽象层 (`exchange/`)

**文件**: `interface.go`, `types.go`, `factory.go`

**设计模式**: 接口 + 适配器模式

```go
type IExchange interface {
    GetName() string
    PlaceOrder(ctx, req) (*Order, error)
    CancelOrder(ctx, symbol, orderID) error
    GetPositions(ctx, symbol) ([]*Position, error)
    StartOrderStream(ctx, callback) error
    StartPriceStream(ctx, symbol, callback) error
    GetPriceDecimals() int
    GetQuantityDecimals() int
    // ...更多方法
}
```

**支持的交易所**：

| 交易所 | 适配器文件 | 状态 |
|--------|------------|------|
| Binance | `binance/adapter.go` | ✅ 完整支持 |
| Bitget | `bitget/adapter.go` | ✅ 完整支持 |
| Gate.io | `gate/adapter.go` | ✅ 完整支持 |

**扩展新交易所步骤**：
1. 创建 `exchange/<name>/` 目录
2. 实现 `IExchange` 接口
3. 创建 wrapper 文件
4. 在 `factory.go` 中注册
5. 在 `config.go` 中添加配置结构

---

### 3. 价格监控 (`monitor/`)

**文件**: `price_monitor.go`

**功能**：
- 通过 WebSocket 订阅实时价格流
- 使用 `atomic.Value` 无锁存储最新价格
- 价格变化时通过 channel 通知交易引擎

**数据流**：

```
交易所 WebSocket → PriceMonitor → atomic.Value → priceChangeCh → 主交易循环
```

**关键代码**：

```go
type PriceMonitor struct {
    exchange     exchange.IExchange
    symbol       string
    lastPrice    atomic.Value    // 无锁存储最新价格
    priceChangeCh chan float64   // 价格变化通知
}

func (pm *PriceMonitor) onPriceUpdate(price float64) {
    pm.lastPrice.Store(price)
    select {
    case pm.priceChangeCh <- price:
    default:  // 非阻塞发送
    }
}
```

---

### 4. 核心交易引擎 (`position/`)

**文件**: `super_position_manager.go`

这是系统最核心的模块，实现了基于槽位的网格交易策略。

#### 核心概念：槽位状态机

每个价格点位对应一个「槽位」(Slot)：

```
┌─────────────────────────────────────────────────────────┐
│                    槽位状态机                            │
├─────────────────────────────────────────────────────────┤
│                                                          │
│    ┌──────┐   买单成交    ┌──────┐   卖单挂出   ┌──────┐ │
│    │ FREE │─────────────▶│FILLED│────────────▶│LOCKED│ │
│    └──────┘              └──────┘              └──────┘ │
│        ▲                                          │      │
│        │                  卖单成交                 │      │
│        └──────────────────────────────────────────┘      │
│                                                          │
│    ┌───────┐   买单挂出                                  │
│    │PENDING│◀───────────── 等待成交                      │
│    └───────┘                                             │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**状态说明**：

| 状态 | 含义 | 订单情况 |
|------|------|----------|
| `FREE` | 空闲 | 无订单、无持仓 |
| `PENDING` | 等待买入 | 买单已挂出，等待成交 |
| `FILLED` | 持仓中 | 买单已成交，持有仓位 |
| `LOCKED` | 等待卖出 | 卖单已挂出，等待成交 |

**槽位数据结构**：

```go
type Slot struct {
    Price          float64       // 价格点位
    PositionStatus string        // 持仓状态
    PositionQty    float64       // 持仓数量
    OrderID        int64         // 当前订单ID
    OrderSide      string        // 订单方向 BUY/SELL
    OrderStatus    string        // 订单状态
    OrderCreatedAt time.Time     // 订单创建时间
    mu             sync.Mutex    // 槽位独立锁
}
```

---

### 5. 订单执行器 (`order/`)

**文件**: `executor_adapter.go`

**功能**：
- 限速控制：25单/秒，突发30单
- 自动重试：最多5次
- PostOnly 降级：失败3次后自动降级为普通限价单
- 保证金不足检测：触发锁定机制

**重试策略**：

```go
maxRetries := 5
postOnlyFailCount := 0
degraded := false

for i := 0; i <= maxRetries; i++ {
    // PostOnly 失败3次后降级为普通单
    if postOnlyFailCount >= 3 && !degraded {
        degraded = true
        exchangeReq.PostOnly = false
        logger.Warn("PostOnly已失败3次，降级为普通限价单")
    }

    order, err := exchange.PlaceOrder(ctx, req)
    if err == nil {
        return order, nil
    }

    if isPostOnlyError(err) {
        postOnlyFailCount++
    }
}
```

**限速实现**：

```go
rateLimiter := rate.NewLimiter(rate.Limit(25), 30)  // 25/秒，突发30

func (oe *OrderExecutor) PlaceOrder(req *OrderRequest) (*Order, error) {
    // 等待令牌
    if err := oe.rateLimiter.Wait(context.Background()); err != nil {
        return nil, err
    }
    // 执行下单...
}
```

---

### 6. 风控系统 (`safety/`)

#### 6.1 启动安全检查 (`safety.go`)

```go
func CheckAccountSafety(ex, symbol, currentPrice, orderAmount,
                        priceInterval, feeRate, requiredPositions,
                        priceDecimals) error {
    // 1. 检查杠杆倍数（最大10倍）
    if leverage > MaxLeverage {
        return fmt.Errorf("杠杆倍率太高")
    }

    // 2. 检查保证金是否足够持有指定仓位数
    maxPositions := (accountBalance * leverage) / orderAmount
    if maxPositions < requiredPositions {
        return fmt.Errorf("保证金不足")
    }

    // 3. 检查净利润是否为正
    netProfit := profitPerTrade - totalFee
    if netProfit <= 0 {
        return fmt.Errorf("每笔净利润为负")
    }
}
```

#### 6.2 实时风控监控 (`risk_monitor.go`)

```go
type RiskMonitor struct {
    cfg           *config.Config
    exchange      exchange.IExchange
    symbolDataMap map[string]*SymbolData  // 多币种K线缓存
    triggered     bool                     // 是否触发风控
}
```

**触发条件**（全部监控币种同时满足）：
- 当前价格 < 移动平均价格
- 成交量 > 均值 × 配置倍数（默认3倍）

**恢复条件**（达到恢复阈值数量的币种满足）：
- 价格回到均线上方
- 成交量恢复正常

#### 6.3 持仓对账 (`reconciler.go`)

```go
func (r *Reconciler) Reconcile() error {
    // 1. 查询交易所实际持仓
    positions, _ := r.exchange.GetPositions(ctx, symbol)

    // 2. 查询所有挂单
    openOrders, _ := r.exchange.GetOpenOrders(ctx, symbol)

    // 3. 遍历本地槽位，统计对比
    r.pm.IterateSlots(func(price float64, slot interface{}) bool {
        // 统计本地状态...
    })

    // 4. 输出对账结果
    logger.Info("对账完成：本地持仓 vs 交易所持仓")
}
```

#### 6.4 订单清理 (`order_cleaner.go`)

```go
func (oc *OrderCleaner) CleanupOrders() {
    // 统计当前订单数
    if totalOrders >= threshold {
        // 策略：优先清理数量多的一方
        if len(buyOrders) > len(sellOrders) {
            // 清理价格最低的买单（离当前价格最远）
        } else {
            // 清理价格最高的卖单（离当前价格最远）
        }
    }
}
```

---

### 7. 日志系统 (`logger/`)

**文件**: `logger.go`

**日志级别**：

| 级别 | 说明 | 生产建议 |
|------|------|----------|
| DEBUG | 详细调试信息 | 仅开发使用 |
| INFO | 正常运行信息 | ✅ 推荐 |
| WARN | 警告信息 | - |
| ERROR | 错误信息 | - |
| FATAL | 致命错误 | 程序退出 |

**特性**：
- DEBUG 模式自动写入文件日志（`./log/opensqt-YYYY-MM-DD.log`）
- 日志文件按日期自动轮转

---

### 8. 工具模块 (`utils/`)

**文件**: `orderid.go`

**功能**：生成紧凑的 ClientOrderID（≤18字符）

**格式**：`{价格整数}_{方向}_{时间戳+序列号}`

**示例**：
- `65000_B_1702468800001` - 价格65000，买单
- `950_S_1702468800123` - 价格0.950，卖单

```go
func GenerateOrderID(price float64, side string, priceDecimals int) string {
    priceInt := int64(price * math.Pow(10, priceDecimals))
    sideCode := "B"  // or "S"
    timestampSeq := fmt.Sprintf("%d%03d", time.Now().Unix(), sequence)
    return fmt.Sprintf("%d_%s_%s", priceInt, sideCode, timestampSeq)
}
```

---

## 做市商策略原理

### 什么是做市商？

做市商（Market Maker）是为市场提供流动性的交易者，通过**同时挂出买单和卖单**，赚取买卖价差（Spread）。

### 本系统的策略：网格交易（Grid Trading）

这是一种**只做多（Long Only）**的网格策略，核心思想是：

> **低买高卖，网格覆盖**：在当前价格下方挂买单，买入后在更高价格挂卖单。

### 策略运行图解

假设配置：
- 当前价格：3000 USDT
- 网格间距：2 USDT
- 买单窗口：5个
- 卖单窗口：5个
- 每单金额：30 USDT

```
价格轴
   ▲
   │
3010│                           ┌───────┐
   │                           │ 卖单5 │ ← 如果持仓，挂卖单
3008│                           └───────┘
   │                           ┌───────┐
3006│                           │ 卖单4 │
   │                           └───────┘
3004│                           ┌───────┐
   │                           │ 卖单3 │
3002│                           └───────┘
   │                           ┌───────┐
3000│ ─ ─ ─ 当前价格 ─ ─ ─ ─ ─ ─│ 卖单2 │
   │                           └───────┘
2998│                           ┌───────┐
   │ ┌───────┐                 │ 卖单1 │
2996│ │ 买单1 │                 └───────┘
   │ └───────┘
2994│ ┌───────┐
   │ │ 买单2 │
2992│ └───────┘
   │ ┌───────┐
2990│ │ 买单3 │
   │ └───────┘
2988│ ┌───────┐
   │ │ 买单4 │
2986│ └───────┘
   │ ┌───────┐
   │ │ 买单5 │
   │ └───────┘
   └─────────────────────────────────────────▶ 时间
```

### 盈利逻辑

以 ETH 为例（配置：间距2U，每单30U）：

```
步骤1: 价格跌到 2996 → 买单1成交 → 买入 0.01 ETH（花费30U）
       同时在 2998 挂出卖单

步骤2: 价格涨到 2998 → 卖单成交 → 卖出 0.01 ETH（获得29.98U + 利润）

利润计算:
┌────────────────────────────────────────┐
│ 买入成本: 2996 × 0.01 = 29.96 USDT     │
│ 卖出收入: 2998 × 0.01 = 29.98 USDT     │
│ 毛利润: 0.02 USDT                      │
│ 手续费: 约 0.012 USDT (假设0.02%费率)  │
│ 净利润: 约 0.008 USDT / 次             │
└────────────────────────────────────────┘

收益预估:
- 每天震荡100次 = 0.8 USDT
- 每天震荡1000次 = 8 USDT
```

### 策略风险

1. **单边下跌风险**：价格持续下跌时，会不断买入，导致浮亏
2. **极端行情风险**：剧烈波动可能导致滑点或无法成交
3. **资金占用**：需要足够保证金支撑网格深度

---

## 核心代码流程

### 1. 初始化阶段

```
main.go
   │
   ├─▶ 加载配置 (config.Load)
   │       └─ 读取 config.yaml
   │
   ├─▶ 创建交易所适配器 (exchange.NewExchange)
   │       ├─ 根据配置选择交易所
   │       └─ 获取合约信息（精度等）
   │
   ├─▶ 执行安全检查 (safety.CheckAccountSafety)
   │       ├─ 检查杠杆倍数
   │       ├─ 检查保证金
   │       └─ 检查盈利可行性
   │
   ├─▶ 创建仓位管理器 (position.NewSuperPositionManager)
   │       └─ 初始化槽位 Map
   │
   ├─▶ 启动 WebSocket
   │       ├─ 价格流 (StartPriceStream)
   │       └─ 订单流 (StartOrderStream)
   │
   ├─▶ 启动后台 Goroutines
   │       ├─ 风控监控 (RiskMonitor)
   │       ├─ 持仓对账 (Reconciler)
   │       ├─ 订单清理 (OrderCleaner)
   │       └─ 状态打印
   │
   └─▶ 进入主循环
```

### 2. 主交易循环

```go
// main.go 核心循环（伪代码）
for {
    select {
    case <-ctx.Done():
        // 优雅关闭
        return

    case price := <-priceChangeCh:
        // 检查风控状态
        if riskMonitor.IsTriggered() {
            continue  // 风控触发，暂停交易
        }

        // 检查保证金锁定
        if time.Now().Before(marginLockUntil) {
            continue  // 保证金锁定中
        }

        // 核心：调整订单
        positionManager.AdjustOrders(price)
    }
}
```

### 3. 订单调整逻辑 (`AdjustOrders`)

```go
func (pm *SuperPositionManager) AdjustOrders(currentPrice float64) {
    // 1. 计算买单窗口范围
    buyLow := currentPrice - interval*buyWindowSize
    buyHigh := currentPrice - interval

    // 2. 计算卖单窗口范围
    sellLow := currentPrice + interval
    sellHigh := currentPrice + interval*sellWindowSize

    // 3. 遍历所有槽位
    pm.slots.Range(func(key, value interface{}) bool {
        price := key.(float64)
        slot := value.(*Slot)

        slot.mu.Lock()
        defer slot.mu.Unlock()

        // 买单逻辑：价格在买单窗口内且槽位空闲
        if price >= buyLow && price <= buyHigh {
            if slot.Status == FREE {
                placeBuyOrder(price)
            }
        }

        // 卖单逻辑：价格在卖单窗口内且持有仓位
        if price >= sellLow && price <= sellHigh {
            if slot.Status == FILLED {
                placeSellOrder(price)
            }
        }

        // 撤单逻辑：价格超出窗口范围
        if price < buyLow || price > sellHigh {
            if slot.HasPendingOrder() {
                cancelOrder(slot.OrderID)
            }
        }

        return true
    })
}
```

### 4. 订单更新回调（WebSocket）

```go
func (pm *SuperPositionManager) OnOrderUpdate(update OrderUpdate) {
    // 根据 ClientOrderID 解析价格，找到对应槽位
    price, side, _, valid := utils.ParseOrderID(update.ClientOrderID, priceDecimals)
    if !valid {
        return
    }

    slot := pm.getSlot(price)
    slot.mu.Lock()
    defer slot.mu.Unlock()

    switch update.Status {
    case "FILLED":
        if side == "BUY" {
            // 买单成交 → 更新为持仓状态
            slot.Status = FILLED
            slot.PositionQty = update.ExecutedQty
            slot.OrderID = 0
            pm.totalBuyQty += update.ExecutedQty
        } else {
            // 卖单成交 → 释放槽位
            slot.Status = FREE
            slot.PositionQty = 0
            slot.OrderID = 0
            pm.totalSellQty += update.ExecutedQty
        }

    case "CANCELED":
        // 订单取消 → 根据情况处理
        slot.OrderStatus = ""
        slot.OrderID = 0

    case "PARTIALLY_FILLED":
        // 部分成交 → 更新已成交数量
        slot.OrderStatus = "PARTIALLY_FILLED"
    }
}
```

---

## 风控机制详解

### 四层风控架构

```
┌─────────────────────────────────────────────────────────────┐
│                        风控系统                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  层级1: 启动前检查 (safety.go)                              │
│  ├─ 杠杆倍数 ≤ 10倍                                         │
│  ├─ 保证金 ≥ 配置仓位数 × 每仓成本                          │
│  └─ 净利润 > 0（扣除手续费后）                              │
│                                                              │
│  层级2: 实时市场监控 (risk_monitor.go)                      │
│  ├─ 监控 BTC、ETH、SOL 等主流币K线                          │
│  ├─ 触发: 全部币种 (价格<均线 && 成交量>3倍均量)            │
│  └─ 触发后: 暂停新订单，等待市场恢复                        │
│                                                              │
│  层级3: 持仓对账 (reconciler.go)                            │
│  ├─ 每60秒执行一次                                          │
│  ├─ 查询交易所实际持仓                                      │
│  └─ 对比本地槽位状态，修复差异                              │
│                                                              │
│  层级4: 订单清理 (order_cleaner.go)                         │
│  ├─ 订单数超过阈值时触发                                    │
│  ├─ 优先清理数量多的一方                                    │
│  └─ 清理离当前价格最远的订单                                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 保证金锁定机制

当遇到保证金不足错误时，系统会锁定一段时间避免频繁失败：

```go
if hasMarginError {
    marginLockUntil = time.Now().Add(10 * time.Second)
    logger.Warn("保证金不足，锁定10秒")
}

// 主循环中检查
if time.Now().Before(marginLockUntil) {
    continue  // 跳过本次调整
}
```

---

## 快速参考

### 核心文件路径

| 功能 | 文件路径 |
|------|----------|
| 程序入口 | `main.go` |
| 配置加载 | `config/config.go` |
| 交易引擎 | `position/super_position_manager.go` |
| 订单执行 | `order/executor_adapter.go` |
| 价格监控 | `monitor/price_monitor.go` |
| 风控监控 | `safety/risk_monitor.go` |
| 交易所接口 | `exchange/interface.go` |

### 运行命令

```bash
# 安装依赖
go mod download

# 编译
go build -o opensqt main.go

# 运行（使用默认 config.yaml）
./opensqt

# 运行（指定配置文件）
./opensqt /path/to/config.yaml

# 代码格式化
go fmt ./...

# 代码检查
go vet ./...
```

### 配置文件模板

```yaml
app:
  current_exchange: "bitget"  # binance / bitget / gate

exchanges:
  binance:
    api_key: "YOUR_API_KEY"
    secret_key: "YOUR_API_SECRET"
    fee_rate: 0.0002

  bitget:
    api_key: "YOUR_API_KEY"
    secret_key: "YOUR_API_SECRET"
    passphrase: "YOUR_PASSPHRASE"
    fee_rate: 0.0002

trading:
  symbol: "ETHUSDT"
  price_interval: 2           # 网格间距
  order_quantity: 30          # 每单金额(USDT)
  buy_window_size: 10         # 买单窗口
  sell_window_size: 10        # 卖单窗口

risk_control:
  enabled: true
  monitor_symbols:
    - "BTCUSDT"
    - "ETHUSDT"
  volume_multiplier: 3.0
  average_window: 20

system:
  log_level: "INFO"
  cancel_on_exit: true
```

### 日志级别说明

| 级别 | 配置值 | 说明 |
|------|--------|------|
| 调试 | DEBUG | 最详细，包含所有信息，写入文件 |
| 信息 | INFO | 正常运行信息（推荐生产环境） |
| 警告 | WARN | 需要注意但不影响运行 |
| 错误 | ERROR | 需要关注的问题 |
| 致命 | FATAL | 程序无法继续 |

---

## 常见问题

### Q: 为什么订单没有成交？

检查以下几点：
1. 价格是否在买单窗口内
2. 是否触发了风控暂停
3. 是否保证金不足被锁定
4. 查看日志确认订单状态

### Q: 如何调整网格密度？

修改 `config.yaml` 中的 `price_interval` 参数：
- 间距越小，交易越频繁，利润越低
- 间距越大，交易越少，利润越高

### Q: 如何查看实时状态？

系统会定期打印状态信息（默认每分钟）：
- 当前持仓数量
- 挂单数量
- 累计买入/卖出
- 预估盈利

---

*文档生成时间: 2025-12-25*
*代码版本: main branch*
