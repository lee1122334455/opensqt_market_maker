# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenSQT Market Maker is a high-performance, low-latency cryptocurrency market maker system written in Go. It implements a long-only grid trading strategy for perpetual contract markets, supporting Binance, Bitget, and Gate.io via WebSocket real-time streams.

## Build & Run Commands

```bash
# Install dependencies
go mod download

# Build
go build -o opensqt main.go

# Run (uses config.yaml by default)
./opensqt

# Run with custom config path
./opensqt /path/to/config.yaml

# Format code
go fmt ./...

# Vet for issues
go vet ./...
```

## Architecture

### Core Module Responsibilities

- **main.go**: Entry point, component orchestration, goroutine management, graceful shutdown
- **config/**: YAML configuration loading and validation
- **exchange/**: Unified exchange abstraction layer using interface + adapter pattern
- **monitor/**: Global WebSocket price source using atomic.Value for lock-free reads
- **position/**: Super Position Manager - core grid trading engine with slot-based state machine
- **order/**: Order execution with rate limiting (25/sec) and retry logic
- **safety/**: Four-layer risk control (pre-launch checks, active monitoring, reconciliation, order cleanup)
- **logger/**: Multi-level logging (DEBUG/INFO/WARN/ERROR/FATAL) with file + console output

### Key Design Patterns

**Exchange Abstraction** (`exchange/interface.go`):
```
IExchange interface
    ↑
    └─ binanceWrapper ↔ BinanceAdapter
    └─ bitgetWrapper ↔ BitgetAdapter
    └─ gateWrapper ↔ GateAdapter
```

**Slot State Machine** (`position/super_position_manager.go`):
```
FREE → PENDING (buy order) → FILLED (position held) → LOCKED (sell order) → FREE
```

**Concurrency Model**:
- `sync.Map` for slot storage
- `sync.RWMutex` for global and per-slot locks
- `atomic.Value` for price storage
- Channels for inter-goroutine communication
- Critical pattern: Release locks before blocking I/O, re-acquire after

### Data Flow

1. Exchange WebSocket → PriceMonitor (atomic update) → priceChangeCh (buffered channel)
2. Main loop receives price → RiskMonitor check → SuperPositionManager.AdjustOrders()
3. Order WebSocket → OnOrderUpdate() → Slot state transitions
4. Reconciler (every 5 min) → Compare local vs exchange state → Fix discrepancies

### Background Goroutines

1. Price stream (WebSocket)
2. Order stream (WebSocket callback)
3. Risk monitor (K-line WebSocket)
4. Reconciler (timer-based)
5. Order cleaner (timer-based)
6. Price change listener (main trading loop)
7. Status printer (periodic)
8. Shutdown handler

## Configuration

Runtime behavior controlled via `config.yaml`. Key sections:
- `app.current_exchange`: Switch between binance, bitget, gate
- `exchanges.<name>`: API credentials per exchange
- `trading`: Symbol, price_interval (grid spacing), order_quantity, window sizes
- `timing`: price_send_interval (ms), order_retry_delay, status_print_interval
- `risk_control`: Volume anomaly detection thresholds
- `system.log_level`: DEBUG enables file logging to `./log/`

## Adding a New Exchange

1. Create `exchange/<name>/` directory with adapter.go, websocket.go, kline_websocket.go
2. Implement IExchange interface in adapter
3. Create wrapper in `exchange/wrapper_<name>.go`
4. Register in `exchange/factory.go`
5. Add config section in `config/config.go`

## Important Conventions

- All prices use `float64` (not decimal library in hot paths)
- Client order IDs generated via `utils/orderid.go` for tracking
- Log messages: Chinese for user-facing, English for debug
- WebSocket callbacks use reflection to extract fields from exchange-specific structs
- PostOnly orders: 3 failures → automatic downgrade to regular orders
- Margin lock: 10-second cooldown on "insufficient margin" errors