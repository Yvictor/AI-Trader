# 🇹🇼 AI-Trader 台股支援整合計劃

> **版本**: v2.1（方案 B - 統一 MCP 工具）
> **更新日期**: 2025-10-31
> **目標**: 在保持美股功能的基礎上，新增台股交易支援

---

## 📋 目錄

1. [專案目標](#專案目標)
2. [設計原則](#設計原則)
3. [架構設計](#架構設計)
4. [檔案異動清單](#檔案異動清單)
5. [配置設計](#配置設計)
6. [實作階段規劃](#實作階段規劃)
7. [技術細節](#技術細節)
8. [測試策略](#測試策略)
9. [使用指南](#使用指南)
10. [風險評估](#風險評估)

---

## 專案目標

### 核心目標
- ✅ 使用 **Shioaji API** 取代 Alpha Vantage（僅限台股）
- ✅ 支援**台股標的**（全市場 + 成交額過濾）
- ✅ 支援**混合模式**（模擬交易 / 真實交易）
- ✅ **保持美股功能完整可用**（向下相容）
- ✅ 保持原有的 **MCP 架構**與 **AI Agent 框架**
- ✅ **統一 MCP 工具**，避免代碼重複

### 非目標
- ❌ 不移除或破壞現有美股功能
- ❌ 不強制要求使用 Shioaji
- ❌ 不改變核心 AI Agent 邏輯

---

## 設計原則

### 1. 向下相容性優先
```
保留原有功能 + 新增台股功能 = 雙軌並行
```

### 2. DRY 原則（Don't Repeat Yourself）
- **統一 MCP 工具**而非分離
- 透過 `market` 參數在工具內部切換邏輯
- 避免維護兩份幾乎相同的代碼

### 3. 擴展式架構
- 資料層分離（Alpha Vantage vs Shioaji）
- 工具層統一（透過 runtime_env.json 傳遞 market）
- 配置層切換（config.market 參數）

---

## 架構設計

### 系統架構對比

#### 資料流（保持分離）
```
【美股資料流】
Alpha Vantage API
    ↓ get_daily_price.py
daily_prices_AAPL.json
    ↓ merge_jsonl.py
merged.jsonl

【台股資料流】
Shioaji API
    ↓ get_daily_price_tw.py
tw_daily_prices_2330.json
    ↓ merge_jsonl_tw.py
merged_tw.jsonl
```

#### MCP 工具層（統一服務）
```
【統一 MCP 工具 - 根據 runtime_env.json 的 MARKET 參數切換】

tool_get_price_local.py (Port 8003)
    ├─ market="us" → 讀取 merged.jsonl
    └─ market="tw" → 讀取 merged_tw.jsonl

tool_trade.py (Port 8002)
    ├─ market="us" → 美股邏輯（任意股數）
    └─ market="tw" → 台股邏輯（1000 股倍數）
```

### 完整架構圖

```
┌─────────────────────────────────────────┐
│            Main Entry                    │
│             main.py                      │
│      (讀取 config.market 參數)            │
│      (寫入 runtime_env.json)             │
└────────────────┬────────────────────────┘
                 │
         ┌───────┴────────┐
         │                │
         ▼                ▼
┌─────────────┐    ┌──────────────┐
│  BaseAgent  │    │Config System │
│(market參數) │    │  (us/tw)     │
└──────┬──────┘    └──────────────┘
       │
       │ 使用統一的 MCP 配置（不需切換）
       │
       ▼
統一 MCP 工具（內部根據 MARKET 參數切換）
├── Math (8000)         - 通用
├── Search (8001)       - 通用
├── Trade (8002)        - 讀取 MARKET 參數
│   ├─ market="us" → 美股邏輯
│   └─ market="tw" → 台股邏輯
└── Price (8003)        - 讀取 MARKET 參數
    ├─ market="us" → merged.jsonl
    └─ market="tw" → merged_tw.jsonl
```

### 參數傳遞機制

```
main.py (或 main_parallel.py)
  ↓ 設定 os.environ["RUNTIME_ENV_PATH"]
  ↓ = "data/agent_data/{signature}/.runtime_env.json"
  ↓ 讀取 config.get("market", "us")
  ↓ write_config_value("MARKET", market)

data/agent_data/{signature}/.runtime_env.json
  (每個 signature 有獨立的配置檔案)

  ↓ get_config_value("MARKET", "us")

MCP Tools (tool_trade.py, tool_get_price_local.py)
  ↓ 根據 market 參數選擇邏輯
  ↓ if market == "us": 使用 merged.jsonl
  ↓ if market == "tw": 使用 merged_tw.jsonl

執行對應的市場邏輯
```

**重要特性**：
- 每個 signature 有獨立的 `.runtime_env.json`
- 支援使用不同 config 並行執行不同市場
- 例如：Terminal 1 執行 `configs/default_config.json` (美股)，Terminal 2 執行 `configs/tw_config.json` (台股)

---

## 檔案異動清單

### 🆕 新增檔案（7 個）

#### 連線管理層
```
tools/shioaji_client.py          # Shioaji 單例連線管理器
```

#### 數據獲取層
```
data/get_tw_stocks.py            # 獲取台股清單（含篩選）
data/get_daily_price_tw.py       # 下載台股 K 線數據
data/merge_jsonl_tw.py           # 合併台股數據
```

#### 台股專屬工具
```
tools/tw_stock_filter.py         # 台股篩選邏輯
```

#### Agent 層
```
prompts/agent_prompt_tw.py       # 台股 Prompt 模板
```

#### 配置層
```
configs/tw_config.json           # 台股配置檔案
```

### 🔧 修改檔案（7 個）

```
pyproject.toml                       # 新增 shioaji 依賴
.env.example                         # 新增 Shioaji 環境變數
configs/default_config.json          # 新增 market: "us" 欄位
agent/base_agent/base_agent.py       # 新增 market 參數支援（簡化版）
agent_tools/tool_trade.py            # 🔥 新增 market 邏輯判斷
agent_tools/tool_get_price_local.py  # 🔥 新增 market 邏輯判斷
tools/price_tools.py                 # 🔥 新增 market 參數支援
main.py                              # 讀取並寫入 market 參數
```

### ✅ 保留檔案（完全不動）

```
data/get_daily_price.py          ✅ 美股版本保留
data/merge_jsonl.py              ✅ 美股版本保留
prompts/agent_prompt.py          ✅ 美股版本保留
tools/general_tools.py           ✅ 保留（通用工具）
agent_tools/tool_math.py         ✅ 保留（通用工具）
agent_tools/tool_jina_search.py  ✅ 保留（通用工具）
agent_tools/start_mcp_services.py ✅ 保留（不需要 market 參數）
```

### ❌ 方案 A 中需要但方案 B 不需要的檔案

```
❌ agent_tools/tool_trade_tw.py          (改為在 tool_trade.py 內部判斷)
❌ agent_tools/tool_get_price_local_tw.py (改為在 tool_get_price_local.py 內部判斷)
```

---

## 配置設計

### 環境變數設計（.env）

```bash
# === 現有配置（保留不變）===
OPENAI_API_BASE=https://your-openai-proxy.com/v1
OPENAI_API_KEY=your_openai_key
ALPHAADVANTAGE_API_KEY=your_alpha_vantage_key
JINA_API_KEY=your_jina_api_key

# === MCP 服務端口（統一，不分市場）===
MATH_HTTP_PORT=8000
SEARCH_HTTP_PORT=8001
TRADE_HTTP_PORT=8002              # 美股和台股共用
GETPRICE_HTTP_PORT=8003           # 美股和台股共用

# === Shioaji 認證（新增）===
SHIOAJI_API_KEY=your_api_key
SHIOAJI_SECRET_KEY=your_secret_key
SHIOAJI_SIMULATION=True              # True=模擬模式, False=真實交易

# === 台股篩選條件（新增）===
TW_STOCK_FILTER_TOP_N=200            # 取成交金額前 N 名
TW_STOCK_MIN_VOLUME=10000000         # 最低成交額 1000 萬
TW_STOCK_EXCLUDE_ETF=True            # 排除 ETF

# === 系統配置 ===
# RUNTIME_ENV_PATH 由 main.py 自動設定為 data/agent_data/{signature}/.runtime_env.json
# 無需手動配置，每個 signature 有獨立的 runtime_env
AGENT_MAX_STEP=30
```

### 配置檔案設計

#### configs/default_config.json（美股配置）

```json
{
  "agent_type": "BaseAgent",
  "market": "us",
  "date_range": {
    "init_date": "2025-10-01",
    "end_date": "2025-10-21"
  },
  "models": [
    {
      "name": "gpt-5",
      "basemodel": "openai/gpt-5",
      "signature": "gpt-5",
      "enabled": true
    }
  ],
  "agent_config": {
    "max_steps": 30,
    "max_retries": 3,
    "base_delay": 1.0,
    "initial_cash": 10000.0
  },
  "log_config": {
    "log_path": "./data/agent_data"
  }
}
```

#### configs/tw_config.json（台股配置）

```json
{
  "agent_type": "BaseAgent",
  "market": "tw",
  "date_range": {
    "init_date": "2025-01-01",
    "end_date": "2025-10-31"
  },
  "models": [
    {
      "name": "gpt-5-tw",
      "basemodel": "openai/gpt-5",
      "signature": "gpt-5-tw",
      "enabled": true
    }
  ],
  "agent_config": {
    "max_steps": 30,
    "max_retries": 3,
    "base_delay": 1.0,
    "initial_cash": 1000000.0
  },
  "log_config": {
    "log_path": "./data/agent_data"
  },
  "tw_config": {
    "stock_filter_top_n": 200,
    "min_volume": 10000000,
    "exclude_etf": true,
    "exchanges": ["TSE", "OTC"]
  },
  "shioaji_config": {
    "simulation": true,
    "auto_login": true
  }
}
```

### 配置參數對比

| 項目 | default_config.json (us) | tw_config.json (tw) |
|------|--------------------------|---------------------|
| **market** | `"us"` | `"tw"` |
| **initial_cash** | `10000.0` (USD) | `1000000.0` (TWD) |
| **MCP 端口** | 8002 (trade), 8003 (price) | **相同** 8002, 8003 |
| **股票清單** | NASDAQ 100 (硬編碼) | 從 `tw_stocks_list.json` 載入 |
| **專屬配置** | 無 | `tw_config`, `shioaji_config` |

---

## 實作階段規劃

### 第一階段：基礎建設（3-4 天）

**目標**: 建立 Shioaji 連線與數據獲取能力

#### 檢查清單
- [ ] 安裝 shioaji 套件 (`pyproject.toml`)
- [ ] 設定環境變數 (`.env`)
- [ ] 建立 `tools/shioaji_client.py` 連線管理器
- [ ] 測試登入（模擬模式）
- [ ] 實作 `data/get_tw_stocks.py`（獲取台股清單）
- [ ] 實作 `tools/tw_stock_filter.py`（台股篩選邏輯）
- [ ] 實作 `data/get_daily_price_tw.py`（下載 K 線）
- [ ] 實作 `data/merge_jsonl_tw.py`（合併數據）
- [ ] 驗證數據格式正確

#### 產出
- ✅ 可登入 Shioaji
- ✅ 可獲取台股清單（篩選後 200 支）
- ✅ 可下載歷史 K 線數據
- ✅ 數據檔案：`merged_tw.jsonl`

---

### 第二階段：MCP 工具改造（3-4 天）

**目標**: 修改現有 MCP 工具支援雙市場

#### 檢查清單
- [ ] 修改 `tools/price_tools.py`
  - [ ] `get_open_prices()` 新增 `market` 參數
  - [ ] 根據 market 選擇資料檔案
- [ ] 修改 `agent_tools/tool_get_price_local.py`
  - [ ] 在開頭讀取 `market = get_config_value("MARKET", "us")`
  - [ ] 根據 market 選擇 `merged.jsonl` 或 `merged_tw.jsonl`
  - [ ] 根據 market 使用不同的欄位名稱
- [ ] 修改 `agent_tools/tool_trade.py`
  - [ ] 在 `buy()` 和 `sell()` 開頭讀取 market
  - [ ] 台股增加 1000 股倍數驗證
  - [ ] 記錄中新增 `market` 欄位
- [ ] 測試美股功能（確保向下相容）
- [ ] 測試台股功能（新增功能）

#### 產出
- ✅ MCP 工具支援雙市場
- ✅ 美股功能完全正常
- ✅ 台股功能正常運作

---

### 第三階段：Agent 整合（2-3 天）

**目標**: 調整 BaseAgent 與 Prompt 系統

#### 檢查清單
- [ ] 修改 `configs/default_config.json`（新增 `market: "us"`）
- [ ] 建立 `configs/tw_config.json`
- [ ] 修改 `agent/base_agent/base_agent.py`
  - [ ] 新增 `market` 參數
  - [ ] 實作 `_load_tw_stocks()`
  - [ ] ~~移除不必要的 `_get_tw_mcp_config()`~~（不需要了）
- [ ] 建立 `prompts/agent_prompt_tw.py`
- [ ] 修改 `main.py`
  - [ ] 讀取 `config.market` 參數
  - [ ] 呼叫 `write_config_value("MARKET", market)`
- [ ] 測試完整交易流程（模擬模式）

#### 產出
- ✅ AI Agent 可正常讀取台股資料
- ✅ AI Agent 可正常執行台股交易（模擬）
- ✅ 日誌與持倉記錄格式正確

---

### 第四階段：真實交易支援（2-3 天，選用）

**目標**: 加入真實下單功能

#### 檢查清單
- [ ] 在 `tool_trade.py` 加入真實下單邏輯
  - [ ] 檢查 `SHIOAJI_SIMULATION` 參數
  - [ ] 模擬模式：寫入 position.jsonl
  - [ ] 真實模式：呼叫 `api.place_order()`
- [ ] 實作訂單狀態回報處理
- [ ] 加入安全機制
  - [ ] 單筆交易金額上限
  - [ ] 每日交易次數限制
  - [ ] 人工確認流程
- [ ] 測試真實下單流程（小額測試）

#### 產出
- ✅ 支援真實下單
- ✅ 安全機制完善

---

## 技術細節

### 統一 MCP 工具的實作方式

#### 核心機制：runtime_env.json 參數傳遞

```python
# main.py 寫入市場參數
market = config.get("market", "us")
write_config_value("MARKET", market)

# MCP 工具讀取市場參數
market = get_config_value("MARKET", "us")
```

#### tool_trade.py 修改範例

```python
@mcp.tool()
def buy(symbol: str, amount: int) -> Dict[str, Any]:
    """買入股票（支援美股和台股）"""

    # 🔥 讀取市場參數
    market = get_config_value("MARKET", "us")

    signature = get_config_value("SIGNATURE")
    today_date = get_config_value("TODAY_DATE")

    # 🔥 台股特殊驗證
    if market == "tw" and amount % 1000 != 0:
        return {
            "error": f"台股交易必須是 1000 股的倍數。您輸入：{amount} 股",
            "symbol": symbol,
            "date": today_date,
            "market": market
        }

    # 獲取持倉
    current_position, current_action_id = get_latest_position(today_date, signature)

    # 🔥 獲取價格（price_tools.get_open_prices 會根據 market 選擇檔案）
    price_data = get_open_prices(today_date, [symbol], market=market)
    stock_price = price_data.get(f'{symbol}_price')

    # 檢查現金
    required_cash = stock_price * amount
    cash_left = current_position.get("CASH", 0) - required_cash

    if cash_left < 0:
        return {"error": "Insufficient cash!", ...}

    # 更新持倉（邏輯完全相同！）
    new_position = current_position.copy()
    new_position["CASH"] = cash_left
    new_position[symbol] = new_position.get(symbol, 0) + amount

    # 記錄交易
    with open(position_file_path, "a") as f:
        record = {
            "date": today_date,
            "id": current_action_id + 1,
            "this_action": {
                "action": "buy",
                "symbol": symbol,
                "amount": amount,
                "price": stock_price,
                "market": market  # 🔥 記錄市場
            },
            "positions": new_position
        }
        f.write(json.dumps(record) + "\n")

    write_config_value("IF_TRADE", True)
    return new_position
```

#### tool_get_price_local.py 修改範例

```python
@mcp.tool()
def get_price_local(symbol: str, date: str) -> Dict[str, Any]:
    """讀取股票歷史價格（支援美股和台股）"""

    # 🔥 讀取市場參數
    market = get_config_value("MARKET", "us")

    # 🔥 根據市場選擇資料檔案和欄位映射
    if market == "us":
        filename = "merged.jsonl"
        field_mapping = {
            "open": "1. buy price",
            "close": "4. sell price",
            # ...
        }
    elif market == "tw":
        filename = "merged_tw.jsonl"
        field_mapping = {
            "open": "open",
            "close": "close",
            # ...
        }

    # 讀取資料
    data_path = _workspace_data_path(filename)
    with data_path.open("r", encoding="utf-8") as f:
        for line in f:
            doc = json.loads(line)

            # 🔥 根據市場解析不同格式
            if market == "us":
                if doc.get("Meta Data", {}).get("2. Symbol") != symbol:
                    continue
                series = doc.get("Time Series (Daily)", {})
            elif market == "tw":
                if doc.get("symbol") != symbol:
                    continue
                series = doc.get("time_series", {})

            day = series.get(date)

            return {
                "symbol": symbol,
                "date": date,
                "market": market,
                "ohlcv": {
                    "open": day.get(field_mapping["open"]),
                    "close": day.get(field_mapping["close"]),
                    # ...
                },
            }
```

#### tools/price_tools.py 修改範例

```python
def get_open_prices(
    today_date: str,
    symbols: List[str],
    market: str = None,  # 🔥 新增參數
    merged_path: Optional[str] = None
) -> Dict[str, Optional[float]]:
    """從本地資料讀取開盤價"""

    # 🔥 如果沒傳入 market，從 runtime_env 讀取
    if market is None:
        from tools.general_tools import get_config_value
        market = get_config_value("MARKET", "us")

    # 🔥 根據市場選擇檔案
    if merged_path is None:
        base_dir = Path(__file__).resolve().parents[1]
        if market == "us":
            merged_file = base_dir / "data" / "merged.jsonl"
        elif market == "tw":
            merged_file = base_dir / "data" / "merged_tw.jsonl"

    # 讀取並解析（根據市場使用不同格式）
    # ...
```

### BaseAgent 修改重點（簡化版）

```python
class BaseAgent:
    def __init__(
        self,
        market: str = "us",  # 新增參數
        # ...
    ):
        self.market = market

        # 根據 market 選擇股票清單
        if market == "us":
            self.stock_symbols = self.DEFAULT_NASDAQ_SYMBOLS
        elif market == "tw":
            self.stock_symbols = self._load_tw_stocks()

        # 🔥 不需要根據 market 切換 MCP 配置（統一使用同一套）
        self.mcp_config = self._get_default_mcp_config()
```

### 數據格式對比

#### 美股數據格式（Alpha Vantage）
```json
{
  "Meta Data": {
    "2. Symbol": "AAPL"
  },
  "Time Series (Daily)": {
    "2025-10-21": {
      "1. buy price": "150.00",
      "4. sell price": "152.00",
      "5. volume": "1000000"
    }
  }
}
```

#### 台股數據格式（Shioaji）
```json
{
  "symbol": "2330",
  "name": "台積電",
  "exchange": "TSE",
  "time_series": {
    "2025-10-21": {
      "open": 500.00,
      "high": 510.00,
      "low": 495.00,
      "close": 505.00,
      "volume": 50000000
    }
  }
}
```

---

## 測試策略

### 單元測試

#### tools/shioaji_client.py
- [ ] 測試連線建立
- [ ] 測試模擬/真實模式切換
- [ ] 測試登入/登出

#### tools/tw_stock_filter.py
- [ ] 測試股票篩選邏輯
- [ ] 測試排名更新
- [ ] 測試 ETF 過濾

#### agent_tools/tool_get_price_local.py
- [ ] 測試美股價格查詢（market="us"）
- [ ] 測試台股價格查詢（market="tw"）
- [ ] 測試錯誤處理

#### agent_tools/tool_trade.py
- [ ] 測試美股買賣（任意股數）
- [ ] 測試台股買賣（1000 倍數）
- [ ] 測試台股股數驗證錯誤
- [ ] 測試現金不足錯誤
- [ ] 測試持股不足錯誤

### 整合測試

#### 美股完整流程測試（確保向下相容）
1. [ ] 設定 `config.market = "us"`
2. [ ] 啟動 MCP 服務
3. [ ] 初始化 BaseAgent
4. [ ] 執行單日交易
5. [ ] 驗證持倉記錄
6. [ ] 驗證功能與原版一致

#### 台股完整流程測試（新功能）
1. [ ] 設定 `config.market = "tw"`
2. [ ] 啟動 MCP 服務（同一套）
3. [ ] 初始化 BaseAgent
4. [ ] 執行單日交易
5. [ ] 驗證持倉記錄
6. [ ] 驗證日誌格式

---

## 使用指南

### 美股交易流程（完全不變）

```bash
# 1. 下載美股數據
uv run data/get_daily_price.py

# 2. 合併數據
uv run data/merge_jsonl.py

# 3. 啟動 MCP 服務（統一啟動，不需要指定市場）
uv run agent_tools/start_mcp_services.py

# 4. 執行美股交易（另一個終端）
# 單一模型順序執行
uv run main.py configs/default_config.json

# 或多模型並行執行（使用 main_parallel.py）
uv run main_parallel.py configs/default_config.json
```

### 台股交易流程（新增）

```bash
# 1. 獲取台股清單
uv run data/get_tw_stocks.py

# 2. 下載台股數據
uv run data/get_daily_price_tw.py

# 3. 合併數據
uv run data/merge_jsonl_tw.py

# 4. 啟動 MCP 服務（同一套服務）
uv run agent_tools/start_mcp_services.py

# 5. 執行台股交易（另一個終端）
# 單一模型順序執行
uv run main.py configs/tw_config.json

# 或多模型並行執行（使用 main_parallel.py）
uv run main_parallel.py configs/tw_config.json
```

### 模擬與真實模式切換

```bash
# .env
SHIOAJI_SIMULATION=True   # 模擬模式
# SHIOAJI_SIMULATION=False  # 真實模式（需謹慎）
```

---

## 風險評估

### 相容性風險

| 風險 | 機率 | 影響 | 緩解措施 |
|------|------|------|---------|
| 美股功能被破壞 | 極低 | 高 | 修改前完整測試美股流程 |
| runtime_env 讀取失敗 | 低 | 中 | 提供預設值 `"us"` |
| 數據格式轉換錯誤 | 中 | 中 | 完整單元測試 |

### 技術風險

| 風險 | 機率 | 影響 | 緩解措施 |
|------|------|------|---------|
| Shioaji API 不穩定 | 中 | 高 | 重試機制、錯誤處理 |
| 台股交易規則差異 | 高 | 中 | 詳細測試、文件說明 |
| 成交額排名變動 | 高 | 低 | 定期更新、快取機制 |

### 操作風險

| 風險 | 機率 | 影響 | 緩解措施 |
|------|------|------|---------|
| 真實交易誤操作 | 低 | 極高 | 預設模擬模式、多重確認 |
| 配置錯誤 | 中 | 中 | 配置驗證、範例檔案 |
| 同一 signature 重複執行 | 低 | 中 | 使用不同 signature，文件說明並行執行方式 |

### 架構限制與特性

| 項目 | 影響 | 說明 |
|------|------|------|
| 同一 signature 只能執行一個市場 | 低 | 每個 signature 的 runtime_env.json 只有一個 MARKET 值 |
| ✅ 支援並行執行不同市場 | 正面 | 使用不同 signature 和 config，可在不同終端並行執行美股和台股 |
| MCP 服務動態讀取配置 | 無 | get_config_value 每次都重新讀取，無需重啟服務 |

---

## 方案 B vs 方案 A 對比

### 方案 A（分離式 MCP）vs 方案 B（統一式 MCP）

| 項目 | 方案 A（分離） | 方案 B（統一）✅ |
|------|--------------|----------------|
| **MCP 工具檔案** | 4 個（trade, price, trade_tw, price_tw） | 2 個（trade, price） |
| **Port 數量** | 4 個（8002, 8003, 8004, 8005） | 2 個（8002, 8003） |
| **代碼行數** | ~800 行 | ~500 行 |
| **維護成本** | 高（兩份幾乎相同的代碼） | 低（一份代碼） |
| **擴展性** | 每加一市場 +2 檔案 | 每加一市場只需在工具內加判斷 |
| **BaseAgent 配置** | 需切換 MCP 配置 | 不需切換 |
| **Bug 修復** | 需改兩個地方 | 只需改一個地方 |
| **並行支援** | 可同時運行 | ✅ 支援（使用不同 config 和 signature） |

**選擇方案 B 的理由**：
1. ✅ 符合 DRY 原則
2. ✅ 維護成本低
3. ✅ 擴展性好
4. ✅ BaseAgent 邏輯簡化
5. ✅ 支援並行執行不同市場（每個 signature 獨立 runtime_env）

---

## 後續優化方向

### 第一優先
- [ ] 加入手續費與交易稅計算（0.1425% 手續費 + 0.3% 證交稅）
- [ ] 支援零股交易（< 1000 股）
- [ ] 台股交易日曆（排除休市日）

### 第二優先
- [ ] 支援期貨選擇權交易
- [ ] 加入技術指標計算
- [ ] 績效視覺化儀表板
- [ ] 風險控管機制

### 第三優先
- [ ] 支援其他市場（港股、A股）
- [ ] 多市場比較分析
- [ ] 策略回測框架
- [ ] 自動化報告生成

### 第四優先（改進使用體驗）
- [ ] 提供並行執行範例腳本
- [ ] 加入並行執行狀態監控
- [ ] 統一的配置管理工具

---

## 附錄

### A. 台灣 50 成分股（預設清單）

```python
DEFAULT_TW_STOCKS = [
    "2330", "2317", "2454", "2308", "2882", "2881", "2303", "1301", "1303", "2412",
    "2891", "2002", "2886", "2880", "2892", "2207", "2382", "3008", "2884", "2912",
    "2409", "2801", "2344", "2345", "2603", "1326", "2357", "2408", "3711", "2885",
    "2395", "2327", "5880", "2379", "2887", "2324", "2301", "3045", "2891", "2883",
    "2890", "2609", "2888", "3034", "2105", "2542", "2615", "5871", "9910", "2002"
]
```

### B. Shioaji API 參考資源

- **官方文件**: https://sinotrade.github.io/
- **GitHub**: https://github.com/Sinotrade/Shioaji
- **API 參考**: https://sinotrade.github.io/llms.txt

### C. 常見問題

**Q: 為什麼選擇方案 B（統一 MCP）而非方案 A（分離 MCP）？**
A: 因為 `buy()` 和 `sell()` 的核心邏輯不因市場而異，只有細節差異（股數驗證、資料來源）。統一工具可以避免維護兩份幾乎相同的代碼，符合 DRY 原則。

**Q: 如何在工具內部判斷是美股還是台股？**
A: 透過 `get_config_value("MARKET", "us")` 從 runtime_env.json 讀取，main.py 會在執行前寫入。

**Q: 台股資料多久更新一次？**
A: 建議每天收盤後執行 `get_daily_price_tw.py` 更新數據。

**Q: 可以同時交易美股和台股嗎？**
A: ✅ 可以！因為每個 signature 有獨立的 `.runtime_env.json` 檔案（位於 `data/agent_data/{signature}/` 目錄下），所以可以在不同終端並行執行：
   - Terminal 1: `uv run main.py configs/default_config.json` (signature="gpt-5", market="us")
   - Terminal 2: `uv run main.py configs/tw_config.json` (signature="gpt-5-tw", market="tw")

   限制：同一個 signature 不能同時執行兩個市場（因為共用同一個 runtime_env.json）

**Q: 模擬模式和真實模式有什麼差別？**
A: 模擬模式只寫入本地檔案，真實模式會實際呼叫 Shioaji API 下單到永豐金證券。

**Q: 如何切換回美股？**
A: 執行 `uv run main.py configs/default_config.json` 即可，美股功能完全保留。

**Q: 為什麼不需要修改 start_mcp_services.py？**
A: 因為 MCP 工具統一了，不需要根據市場啟動不同的服務。所有市場共用同一套 MCP 服務（Port 8002, 8003）。MCP 工具內部會透過 `get_config_value("MARKET")` 動態讀取當前市場參數。

**Q: main.py 和 main_parallel.py 有什麼差別？**
A:
- `main.py`：順序執行多個模型（config 中的 models 列表依序執行）
- `main_parallel.py`：並行執行多個模型（同時啟動多個子程序）
- 兩者都支援台股/美股切換，只是執行方式不同
- 使用 main_parallel.py 可節省總執行時間

**Q: runtime_env.json 存放在哪裡？**
A: 每個 signature 有獨立的配置檔案，路徑為 `data/agent_data/{signature}/.runtime_env.json`。這個路徑由 main.py 自動設定（main.py Line 179-182），不需要手動配置。

**Q: 如何確認目前使用的是哪個市場？**
A: 可以查看對應 signature 的 runtime_env.json 檔案：
```bash
cat data/agent_data/{signature}/.runtime_env.json
# 會看到 {"MARKET": "us"} 或 {"MARKET": "tw"}
```

---

## 變更歷史

| 版本 | 日期 | 變更內容 |
|------|------|---------|
| v1.0 | 2025-10-31 | 初版發布（方案 A - 分離式 MCP） |
| v2.0 | 2025-10-31 | 改為方案 B（統一式 MCP），減少代碼重複 |
| v2.1 | 2025-10-31 | 修正 runtime_env.json 路徑說明，澄清並行執行機制，新增 main_parallel.py 使用說明 |

---

**文件維護者**: AI-Trader Team
**最後更新**: 2025-10-31
**當前方案**: 方案 B - 統一 MCP 工具
