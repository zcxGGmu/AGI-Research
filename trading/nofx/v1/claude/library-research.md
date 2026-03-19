# 港股 + A股 交易系统 Python 库研究报告

---

## 1. AKShare (akshare)

### 安装

```bash
pip install akshare --upgrade
```

### 1.1 stock_zh_a_spot_em() — A股实时行情

```python
import akshare as ak

df = ak.stock_zh_a_spot_em()
```

**参数**: 无

**返回 DataFrame 列**:

| 列名 | 类型 | 说明 |
|------|------|------|
| 序号 | int | 行号 |
| 代码 | str | 股票代码 (e.g. "000001") |
| 名称 | str | 股票名称 |
| 最新价 | float | 最新价格 |
| 涨跌幅 | float | 涨跌百分比 |
| 涨跌额 | float | 涨跌金额 |
| 成交量 | int | 成交量(手) |
| 成交额 | float | 成交额(元) |
| 振幅 | float | 振幅(%) |
| 最高 | float | 最高价 |
| 最低 | float | 最低价 |
| 今开 | float | 开盘价 |
| 昨收 | float | 昨日收盘价 |
| 量比 | float | 量比 |
| 换手率 | float | 换手率(%) |
| 市盈率-动态 | float | 动态PE |
| 市净率 | float | PB |
| 总市值 | float | 总市值(元) |
| 流通市值 | float | 流通市值(元) |
| 涨速 | float | 涨速 |
| 5分钟涨跌 | float | 5分钟涨跌幅 |
| 60日涨跌幅 | float | 60日涨跌幅 |
| 年初至今涨跌幅 | float | YTD涨跌幅 |

**已知问题**: 自动化调用时可能只返回 100-200 条数据而非全部 5000+。建议加重试逻辑并校验返回行数。

**生产代码示例**:

```python
import akshare as ak
import time

def fetch_a_share_realtime(max_retries: int = 3) -> "pd.DataFrame":
    """获取A股实时行情，带重试逻辑"""
    for attempt in range(max_retries):
        try:
            df = ak.stock_zh_a_spot_em()
            if len(df) > 1000:  # 正常应有5000+条
                return df
            time.sleep(2)
        except Exception as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(3)
    return df
```

### 1.2 stock_zh_a_hist() — A股历史K线

```python
df = ak.stock_zh_a_hist(
    symbol="000001",       # 纯数字代码，不带 sh/sz 前缀
    period="daily",        # "daily" | "weekly" | "monthly"
    start_date="20230101", # YYYYMMDD
    end_date="20240101",   # YYYYMMDD
    adjust="qfq"           # "" (不复权) | "qfq" (前复权) | "hfq" (后复权)
)
```

**返回 DataFrame 列**:

| 列名 | 说明 |
|------|------|
| 日期 | 交易日期 |
| 开盘 | 开盘价 |
| 收盘 | 收盘价 |
| 最高 | 最高价 |
| 最低 | 最低价 |
| 成交量 | 成交量 |
| 成交额 | 成交额 |
| 振幅 | 振幅(%) |
| 涨跌幅 | 涨跌幅(%) |
| 涨跌额 | 涨跌额 |
| 换手率 | 换手率(%) |

**备用数据源** (东财失败时降级):

```python
# 备用1: 新浪 (注意 symbol 格式不同: "sz000001")
df_sina = ak.stock_zh_a_daily(
    symbol="sz000001",
    start_date="20230101",
    end_date="20240101",
    adjust="qfq"
)
# 列名为英文: date, open, high, low, close, volume

# 备用2: 腾讯
df_tx = ak.stock_zh_a_hist_tx(
    symbol="sz000001",
    start_date="20230101",
    end_date="20240101",
    adjust="qfq"
)
# 列名为英文: date, open, high, low, close, amount
```

**生产级多源降级示例**:

```python
import akshare as ak
import pandas as pd

def fetch_a_hist(symbol: str, start: str, end: str, adjust: str = "qfq") -> pd.DataFrame:
    """多数据源降级获取A股历史K线"""
    # 统一列名映射
    UNIFIED_COLS = ["date", "open", "high", "low", "close", "volume", "amount"]

    # 源1: 东财
    try:
        df = ak.stock_zh_a_hist(
            symbol=symbol, period="daily",
            start_date=start, end_date=end, adjust=adjust
        )
        df = df.rename(columns={
            "日期": "date", "开盘": "open", "最高": "high",
            "最低": "low", "收盘": "close", "成交量": "volume", "成交额": "amount"
        })
        return df[UNIFIED_COLS]
    except Exception:
        pass

    # 源2: 新浪
    prefix = "sh" if symbol.startswith("6") else "sz"
    try:
        df = ak.stock_zh_a_daily(
            symbol=f"{prefix}{symbol}",
            start_date=start, end_date=end, adjust=adjust
        )
        df = df.rename(columns={"volume": "volume"})
        if "amount" not in df.columns:
            df["amount"] = 0
        return df[UNIFIED_COLS]
    except Exception:
        pass

    # 源3: 腾讯
    df = ak.stock_zh_a_hist_tx(
        symbol=f"{prefix}{symbol}",
        start_date=start, end_date=end, adjust=adjust
    )
    df = df.rename(columns={"amount": "amount"})
    if "volume" not in df.columns:
        df["volume"] = 0
    return df[UNIFIED_COLS]
```

### 1.3 stock_hk_spot_em() — 港股实时行情

```python
df = ak.stock_hk_spot_em()
```

**参数**: 无

**返回 DataFrame 列**: 序号, 代码, 名称, 最新价, 涨跌额, 涨跌幅, 今开, 最高, 最低, 昨收, 成交量, 成交额

### 1.4 stock_hk_hist() — 港股历史K线

```python
df = ak.stock_hk_hist(
    symbol="00700",        # 港股代码
    period="daily",        # "daily" | "weekly" | "monthly"
    start_date="20230101",
    end_date="20240101",
    adjust="qfq"           # "" | "qfq" | "hfq"
)
```

**返回 DataFrame 列**: 日期, 开盘, 收盘, 最高, 最低, 成交量, 成交额, 振幅, 涨跌幅, 涨跌额, 换手率

(与 A 股 `stock_zh_a_hist` 列结构一致)

### 1.5 stock_board_industry_name_em() — 行业板块

```python
df = ak.stock_board_industry_name_em()
```

**返回 DataFrame 列**: 排名, 板块名称, 板块代码, 最新价, 涨跌幅, 涨跌额, 总市值, 换手率, 上涨家数, 下跌家数, 领涨股票, 领涨涨跌幅

### 1.6 stock_hsgt_north_net_flow_in_em() — 北向资金

```python
df = ak.stock_hsgt_north_net_flow_in_em(symbol="北向资金")
# symbol: "北向资金" | "沪股通" | "深股通"
```

**返回 DataFrame 列**: 日期, 当日净流入(亿), 当日余额(亿)

### 1.7 stock_zh_a_gdhs() — 股东户数 (基本面)

```python
df = ak.stock_zh_a_gdhs(symbol="000001")
```

**返回 DataFrame 列**: 股东户数统计截止日, 区间涨跌幅, 股东户数(户), 股东户数增减, 户均持股市值, 户均持股数量, 总市值, 总股本

### 1.8 stock_news_em() — 个股新闻

```python
df = ak.stock_news_em(symbol="000001")
```

**返回 DataFrame 列**: 关键词, 新闻标题, 新闻内容, 发布时间, 文章来源, 新闻链接

### Rate Limits & Caveats

- 无官方 rate limit，但底层爬取东方财富，频繁调用会被封 IP
- 建议每次请求间隔 1-3 秒
- `stock_zh_a_spot_em()` 在自动化环境中经常只返回 200 条数据 (已知 bug)
- 接口变动频繁，建议锁定版本并定期测试
- 所有列名为中文，需要 rename 后使用
- 数据源为东方财富网页爬虫，非官方 API，稳定性无保证

---

## 2. Futu OpenAPI (futu-api)

### 安装

```bash
pip install futu-api
```

**前置条件**: 必须安装并运行 [FutuOpenD](https://www.futunn.com/download/openAPI) 网关程序 (本地守护进程，默认监听 `127.0.0.1:11111`)。

### 2.1 连接与认证

```python
from futu import *

# ===== 行情上下文 =====
quote_ctx = OpenQuoteContext(host='127.0.0.1', port=11111)

# ===== 交易上下文 =====
# 港股交易
trd_ctx_hk = OpenSecTradeContext(
    filter_trdmarket=TrdMarket.HK,
    host='127.0.0.1',
    port=11111,
    security_firm=SecurityFirm.FUTUSECURITIES
)

# A股交易 (沪深通)
trd_ctx_cn = OpenSecTradeContext(
    filter_trdmarket=TrdMarket.CN,
    host='127.0.0.1',
    port=11111,
    security_firm=SecurityFirm.FUTUSECURITIES
)
```

**关键枚举**:

| 枚举 | 值 | 说明 |
|------|-----|------|
| `TrdMarket.HK` | 1 | 港股市场 |
| `TrdMarket.US` | 2 | 美股市场 |
| `TrdMarket.CN` | 3 | A股市场 |
| `TrdEnv.REAL` | 0 | 真实交易 |
| `TrdEnv.SIMULATE` | 1 | 模拟交易 |
| `TrdSide.BUY` | 1 | 买入 |
| `TrdSide.SELL` | 2 | 卖出 |
| `OrderType.NORMAL` | 0 | 普通限价单 |
| `OrderType.MARKET` | 1 | 市价单 |
| `OrderType.ABSOLUTE_LIMIT` | 5 | 竞价限价单 |
| `SecurityFirm.FUTUSECURITIES` | 1 | 富途证券 |

### 2.2 place_order() — 下单

```python
ret, data = trd_ctx_hk.place_order(
    price=350.0,                    # 价格 (市价单可设0)
    qty=100,                        # 数量
    code="HK.00700",                # 股票代码
    trd_side=TrdSide.BUY,           # 买卖方向
    order_type=OrderType.NORMAL,    # 订单类型
    trd_env=TrdEnv.SIMULATE,        # 交易环境
    adjust_limit=0,                 # 价格调整幅度(%)，0=不调整
    remark="strategy_alpha_001"     # 备注
)

if ret == RET_OK:
    print(data)  # DataFrame: order_id, code, trd_side, order_type, ...
    order_id = data['order_id'].iloc[0]
else:
    print(f"下单失败: {data}")  # data 为错误信息字符串
```

**返回 DataFrame 列**: order_id, code, stock_name, trd_side, order_type, order_status, qty, price, create_time, updated_time

### 2.3 position_list_query() — 持仓查询

```python
ret, data = trd_ctx_hk.position_list_query(
    code="",                    # 空字符串=查全部, 或指定 "HK.00700"
    pl_ratio_sort=None,         # 排序: SortDir.ASCEND / SortDir.DESCEND
    trd_env=TrdEnv.SIMULATE
)

if ret == RET_OK:
    for _, row in data.iterrows():
        print(f"{row['code']}: 持仓{row['qty']}股, "
              f"成本{row['cost_price']}, 市值{row['market_val']}, "
              f"盈亏{row['pl_val']} ({row['pl_ratio']*100:.1f}%)")
```

**返回 DataFrame 列**: position_side, code, stock_name, qty, can_sell_qty, cost_price, market_val, nominal_price, pl_val, pl_ratio, today_pl_val, today_buy_qty, today_sell_qty

### 2.4 accinfo_query() — 账户信息

```python
ret, data = trd_ctx_hk.accinfo_query(trd_env=TrdEnv.SIMULATE)

if ret == RET_OK:
    print(f"总资产: {data['total_assets'].iloc[0]}")
    print(f"现金: {data['cash'].iloc[0]}")
    print(f"持仓市值: {data['market_val'].iloc[0]}")
    print(f"可用资金: {data['avl_withdrawal_cash'].iloc[0]}")
```

**返回 DataFrame 列**: power, total_assets, cash, market_val, frozen_cash, avl_withdrawal_cash, max_power_short, net_cash_power, currency

### 2.5 行情接口

```python
# 获取实时报价
ret, data = quote_ctx.get_stock_quote(["HK.00700", "HK.09988"])
# 返回: code, name, last_price, open_price, high_price, low_price,
#        prev_close_price, volume, turnover, amplitude, turnover_rate, ...

# 获取K线
ret, data, _ = quote_ctx.request_history_kline(
    code="HK.00700",
    start="2024-01-01",
    end="2024-06-01",
    ktype=KLType.K_DAY,       # K_DAY | K_WEEK | K_MON | K_1M | K_5M | K_15M
    autype=AuType.QFQ,        # QFQ | HFQ | NONE
    max_count=1000
)
# 返回: code, time_key, open, close, high, low, volume, turnover,
#        pe_ratio, turnover_rate, change_rate

# 订阅实时行情推送
ret, data = quote_ctx.subscribe(["HK.00700"], [SubType.QUOTE, SubType.K_1M])
```

### 2.6 模拟 vs 真实交易

```python
# 模拟交易: 无需 unlock，直接下单
ret, data = trd_ctx_hk.place_order(
    price=350.0, qty=100, code="HK.00700",
    trd_side=TrdSide.BUY, order_type=OrderType.NORMAL,
    trd_env=TrdEnv.SIMULATE  # <-- 模拟环境
)

# 真实交易: 必须先 unlock_trade
ret, data = trd_ctx_hk.unlock_trade(
    password_md5="your_md5_password",  # 交易密码的MD5
    is_unlock=True
)
if ret == RET_OK:
    ret, data = trd_ctx_hk.place_order(
        price=350.0, qty=100, code="HK.00700",
        trd_side=TrdSide.BUY, order_type=OrderType.NORMAL,
        trd_env=TrdEnv.REAL  # <-- 真实环境
    )
```

### 2.7 完整生产示例

```python
from futu import *
import pandas as pd

class FutuTrader:
    def __init__(self, host='127.0.0.1', port=11111, market=TrdMarket.HK,
                 env=TrdEnv.SIMULATE):
        self.quote_ctx = OpenQuoteContext(host=host, port=port)
        self.trd_ctx = OpenSecTradeContext(
            filter_trdmarket=market, host=host, port=port,
            security_firm=SecurityFirm.FUTUSECURITIES
        )
        self.env = env

    def get_quote(self, codes: list[str]) -> pd.DataFrame:
        ret, data = self.quote_ctx.get_stock_quote(codes)
        if ret != RET_OK:
            raise RuntimeError(f"行情获取失败: {data}")
        return data

    def get_positions(self) -> pd.DataFrame:
        ret, data = self.trd_ctx.position_list_query(trd_env=self.env)
        if ret != RET_OK:
            raise RuntimeError(f"持仓查询失败: {data}")
        return data

    def get_account(self) -> dict:
        ret, data = self.trd_ctx.accinfo_query(trd_env=self.env)
        if ret != RET_OK:
            raise RuntimeError(f"账户查询失败: {data}")
        return data.iloc[0].to_dict()

    def buy(self, code: str, price: float, qty: int) -> str:
        ret, data = self.trd_ctx.place_order(
            price=price, qty=qty, code=code,
            trd_side=TrdSide.BUY, order_type=OrderType.NORMAL,
            trd_env=self.env
        )
        if ret != RET_OK:
            raise RuntimeError(f"买入失败: {data}")
        return data['order_id'].iloc[0]

    def sell(self, code: str, price: float, qty: int) -> str:
        ret, data = self.trd_ctx.place_order(
            price=price, qty=qty, code=code,
            trd_side=TrdSide.SELL, order_type=OrderType.NORMAL,
            trd_env=self.env
        )
        if ret != RET_OK:
            raise RuntimeError(f"卖出失败: {data}")
        return data['order_id'].iloc[0]

    def close(self):
        self.quote_ctx.close()
        self.trd_ctx.close()
```

### Rate Limits & Caveats

- 每 30 秒最多 15 次下单请求 (模拟环境)
- 真实环境需要先 `unlock_trade`，且有频率限制
- 必须本地运行 FutuOpenD 网关 (不支持纯云端)
- 行情订阅有额度限制: 免费用户 100 个标的
- 港股代码格式: `HK.00700`，A股: `SH.600000` / `SZ.000001`
- 模拟账户初始资金 100 万港币 / 100 万人民币
- FutuOpenD 需要登录富途账号，有 IP 白名单限制

---

## 3. Longbridge / LongPort OpenAPI (longport)

### 安装

```bash
pip install longport
```

> 注意: 旧包名 `longbridge` 已废弃，新包名为 `longport`。

### 3.1 认证与连接

需要在 [LongPort 开放平台](https://open.longbridge.com) 申请 API 凭证。

**环境变量方式** (推荐):

```bash
export LONGPORT_APP_KEY="your_app_key"
export LONGPORT_APP_SECRET="your_app_secret"
export LONGPORT_ACCESS_TOKEN="your_access_token"
```

```python
from longport.openapi import Config, QuoteContext, TradeContext

# 从环境变量加载配置
config = Config.from_env()

# 行情上下文
quote_ctx = QuoteContext(config)

# 交易上下文
trade_ctx = TradeContext(config)
```

**代码内配置**:

```python
config = Config(
    app_key="your_app_key",
    app_secret="your_app_secret",
    access_token="your_access_token"
)
```

### 3.2 QuoteContext — 行情接口

```python
from longport.openapi import Config, QuoteContext, Period, AdjustType

config = Config.from_env()
ctx = QuoteContext(config)

# 获取实时报价
quotes = ctx.quote(["700.HK", "AAPL.US", "600519.SH"])
for q in quotes:
    print(f"{q.symbol}: 最新价={q.last_done}, "
          f"开盘={q.open}, 最高={q.high}, 最低={q.low}, "
          f"成交量={q.volume}, 换手率={q.turnover_rate}")

# 获取K线 (candlesticks)
candles = ctx.candlesticks(
    symbol="700.HK",
    period=Period.Day,        # Day | Week | Month | Min_1 | Min_5 | Min_15 | Min_30 | Min_60
    count=100,                # 返回条数
    adjust_type=AdjustType.ForwardAdjust  # ForwardAdjust | NoAdjust
)
for c in candles:
    print(f"{c.timestamp}: O={c.open}, H={c.high}, L={c.low}, C={c.close}, V={c.volume}")

# 获取历史K线
history = ctx.history_candlesticks_by_date(
    symbol="700.HK",
    period=Period.Day,
    adjust_type=AdjustType.ForwardAdjust,
    start=date(2024, 1, 1),
    end=date(2024, 6, 1)
)
```

**Quote 对象属性**: symbol, last_done, prev_close, open, high, low, timestamp, volume, turnover, turnover_rate, trade_status

**Candlestick 对象属性**: close, open, low, high, volume, turnover, timestamp

### 3.3 TradeContext — 交易接口

```python
from decimal import Decimal
from longport.openapi import (
    Config, TradeContext,
    OrderSide, OrderType, TimeInForceType, OutsideRTH
)

config = Config.from_env()
ctx = TradeContext(config)

# ===== 下单 =====
resp = ctx.submit_order(
    symbol="700.HK",
    order_type=OrderType.LO,              # LO (限价) | MO (市价) | ELO | ALO
    side=OrderSide.Buy,                    # Buy | Sell
    submitted_quantity=200,                # 数量
    time_in_force=TimeInForceType.Day,     # Day | GoodTilCanceled | GoodTilDate
    submitted_price=Decimal("350.0"),      # 限价单价格
    remark="strategy_001"                  # 备注
)
order_id = resp.order_id
print(f"订单ID: {order_id}")

# ===== 查询账户余额 =====
balances = ctx.account_balance(currency="HKD")
for b in balances:
    print(f"币种={b.currency}, 总资产={b.total_cash}, "
          f"可用={b.available_cash}, 冻结={b.frozen_cash}, "
          f"持仓市值={b.market_value}")

# ===== 查询持仓 =====
positions = ctx.stock_positions()
for channel in positions.channels:
    for p in channel.positions:
        print(f"{p.symbol}: 数量={p.quantity}, "
              f"可卖={p.available_quantity}, "
              f"成本={p.cost_price}, 市值={p.market_value}")

# ===== 查询订单 =====
orders = ctx.today_orders()
for o in orders:
    print(f"订单{o.order_id}: {o.symbol} {o.side} "
          f"{o.quantity}@{o.price} 状态={o.status}")

# ===== 撤单 =====
ctx.cancel_order(order_id=order_id)
```

**关键枚举**:

| 枚举 | 值 | 说明 |
|------|-----|------|
| `OrderType.LO` | - | 限价单 (Limit Order) |
| `OrderType.MO` | - | 市价单 (Market Order) |
| `OrderType.ELO` | - | 增强限价单 |
| `OrderType.ALO` | - | 竞价限价单 |
| `OrderSide.Buy` | - | 买入 |
| `OrderSide.Sell` | - | 卖出 |
| `TimeInForceType.Day` | - | 当日有效 |
| `TimeInForceType.GoodTilCanceled` | - | 撤单前有效 |

**代码格式**: `700.HK` (港股), `AAPL.US` (美股), `600519.SH` / `000001.SZ` (A股)

### 3.4 完整生产示例

```python
from decimal import Decimal
from datetime import date
from longport.openapi import (
    Config, QuoteContext, TradeContext,
    OrderSide, OrderType, TimeInForceType, Period, AdjustType
)

class LongbridgeTrader:
    def __init__(self):
        self.config = Config.from_env()
        self.quote_ctx = QuoteContext(self.config)
        self.trade_ctx = TradeContext(self.config)

    def get_quote(self, symbols: list[str]) -> list:
        return self.quote_ctx.quote(symbols)

    def get_history(self, symbol: str, start: date, end: date) -> list:
        return self.quote_ctx.history_candlesticks_by_date(
            symbol=symbol,
            period=Period.Day,
            adjust_type=AdjustType.ForwardAdjust,
            start=start, end=end
        )

    def get_positions(self) -> list:
        resp = self.trade_ctx.stock_positions()
        positions = []
        for ch in resp.channels:
            positions.extend(ch.positions)
        return positions

    def get_balance(self, currency: str = "HKD") -> list:
        return self.trade_ctx.account_balance(currency=currency)

    def buy(self, symbol: str, price: Decimal, qty: int) -> str:
        resp = self.trade_ctx.submit_order(
            symbol=symbol,
            order_type=OrderType.LO,
            side=OrderSide.Buy,
            submitted_quantity=qty,
            time_in_force=TimeInForceType.Day,
            submitted_price=price
        )
        return resp.order_id

    def sell(self, symbol: str, price: Decimal, qty: int) -> str:
        resp = self.trade_ctx.submit_order(
            symbol=symbol,
            order_type=OrderType.LO,
            side=OrderSide.Sell,
            submitted_quantity=qty,
            time_in_force=TimeInForceType.Day,
            submitted_price=price
        )
        return resp.order_id

    def cancel(self, order_id: str):
        self.trade_ctx.cancel_order(order_id=order_id)
```

### Rate Limits & Caveats

- API 调用频率限制: 行情 10 次/秒，交易 5 次/秒
- 需要在 LongPort 开通 OpenAPI 权限 (需要实名认证)
- 支持港股、美股、A股 (沪深港通)
- 行情数据有延迟 (免费为 15 分钟延迟，实时需付费)
- Python SDK 底层为 Rust 实现，性能较好
- 支持 WebSocket 实时推送
- 模拟交易需要单独申请

---

## 4. TA-Lib (ta-lib)

### 安装 (Linux)

TA-Lib 需要先编译安装 C 库，再安装 Python 包装器:

```bash
# 1. 安装编译依赖
sudo apt-get update
sudo apt-get install -y build-essential wget

# 2. 下载并编译 C 库
wget http://prdownloads.sourceforge.net/ta-lib/ta-lib-0.4.0-src.tar.gz
tar -xzf ta-lib-0.4.0-src.tar.gz
cd ta-lib/
./configure --prefix=/usr
make
sudo make install
cd ..
rm -rf ta-lib ta-lib-0.4.0-src.tar.gz

# 3. 安装 Python 包
pip install TA-Lib
```

**快捷方式 (conda)**:
```bash
conda install -c conda-forge ta-lib
```

### 4.1 MACD — 移动平均收敛散度

```python
import talib
import numpy as np

# 签名
macd, signal, hist = talib.MACD(
    close,              # np.ndarray, 收盘价序列
    fastperiod=12,      # 快线周期 (默认12)
    slowperiod=26,      # 慢线周期 (默认26)
    signalperiod=9      # 信号线周期 (默认9)
)
# 返回: 3个 np.ndarray
# macd   - MACD线 (DIF)
# signal - 信号线 (DEA)
# hist   - 柱状图 (MACD柱)
```

### 4.2 RSI — 相对强弱指标

```python
rsi = talib.RSI(
    close,              # np.ndarray
    timeperiod=14       # 周期 (默认14)
)
# 返回: np.ndarray, 前 timeperiod 个值为 NaN
```

### 4.3 EMA — 指数移动平均

```python
ema = talib.EMA(
    close,              # np.ndarray
    timeperiod=30       # 周期 (默认30)
)
# 返回: np.ndarray
```

### 4.4 BBANDS — 布林带

```python
upper, middle, lower = talib.BBANDS(
    close,              # np.ndarray
    timeperiod=20,      # 周期 (默认5)
    nbdevup=2,          # 上轨标准差倍数 (默认2)
    nbdevdn=2,          # 下轨标准差倍数 (默认2)
    matype=0            # 均线类型: 0=SMA, 1=EMA, 2=WMA, 3=DEMA, 4=TEMA
)
# 返回: 3个 np.ndarray (上轨, 中轨, 下轨)
```

### 4.5 ATR — 平均真实波幅

```python
atr = talib.ATR(
    high,               # np.ndarray, 最高价
    low,                # np.ndarray, 最低价
    close,              # np.ndarray, 收盘价
    timeperiod=14       # 周期 (默认14)
)
# 返回: np.ndarray
```

### 4.6 与 AKShare + Pandas 集成的完整示例

```python
import akshare as ak
import talib
import numpy as np
import pandas as pd

def compute_indicators(symbol: str, start: str, end: str) -> pd.DataFrame:
    """获取A股数据并计算全套技术指标"""
    # 获取历史数据
    df = ak.stock_zh_a_hist(
        symbol=symbol, period="daily",
        start_date=start, end_date=end, adjust="qfq"
    )
    df = df.rename(columns={
        "日期": "date", "开盘": "open", "最高": "high",
        "最低": "low", "收盘": "close", "成交量": "volume"
    })

    close = df["close"].values.astype(float)
    high = df["high"].values.astype(float)
    low = df["low"].values.astype(float)

    # MACD
    df["macd"], df["macd_signal"], df["macd_hist"] = talib.MACD(
        close, fastperiod=12, slowperiod=26, signalperiod=9
    )

    # RSI
    df["rsi_14"] = talib.RSI(close, timeperiod=14)
    df["rsi_6"] = talib.RSI(close, timeperiod=6)

    # EMA
    df["ema_5"] = talib.EMA(close, timeperiod=5)
    df["ema_20"] = talib.EMA(close, timeperiod=20)
    df["ema_60"] = talib.EMA(close, timeperiod=60)

    # 布林带
    df["bb_upper"], df["bb_middle"], df["bb_lower"] = talib.BBANDS(
        close, timeperiod=20, nbdevup=2, nbdevdn=2, matype=0
    )

    # ATR
    df["atr_14"] = talib.ATR(high, low, close, timeperiod=14)

    # 额外常用指标
    df["adx"] = talib.ADX(high, low, close, timeperiod=14)
    df["cci"] = talib.CCI(high, low, close, timeperiod=14)
    df["willr"] = talib.WILLR(high, low, close, timeperiod=14)
    df["obv"] = talib.OBV(close, df["volume"].values.astype(float))

    return df
```

### 4.7 其他常用函数速查

| 函数 | 签名 | 说明 |
|------|------|------|
| `SMA` | `talib.SMA(close, timeperiod=30)` | 简单移动平均 |
| `WMA` | `talib.WMA(close, timeperiod=30)` | 加权移动平均 |
| `DEMA` | `talib.DEMA(close, timeperiod=30)` | 双指数移动平均 |
| `TEMA` | `talib.TEMA(close, timeperiod=30)` | 三指数移动平均 |
| `ADX` | `talib.ADX(high, low, close, timeperiod=14)` | 平均趋向指标 |
| `CCI` | `talib.CCI(high, low, close, timeperiod=14)` | 商品通道指标 |
| `WILLR` | `talib.WILLR(high, low, close, timeperiod=14)` | 威廉指标 |
| `OBV` | `talib.OBV(close, volume)` | 能量潮 |
| `STOCH` | `talib.STOCH(high, low, close, fastk=5, slowk=3, slowd=3)` | KDJ |
| `SAR` | `talib.SAR(high, low, acceleration=0.02, maximum=0.2)` | 抛物线SAR |
| `MFI` | `talib.MFI(high, low, close, volume, timeperiod=14)` | 资金流量指标 |

### Rate Limits & Caveats

- 纯本地计算，无网络请求，无 rate limit
- 输入必须是 `numpy.ndarray` (float64)，不接受 pandas Series (需 `.values`)
- 输出前 N 个值为 NaN (N = timeperiod - 1)
- C 库编译在某些 Linux 发行版上可能有问题，conda 安装更稳定
- 版本 0.4.0 的 C 库较老但稳定，Python 包装器持续更新
- 不支持 Apple Silicon 原生编译 (需要 Rosetta 或 conda)

---

## 5. 库对比与选型建议

| 维度 | AKShare | Futu OpenAPI | Longbridge | TA-Lib |
|------|---------|-------------|------------|--------|
| 用途 | 数据获取 | 行情+交易 | 行情+交易 | 技术指标计算 |
| 费用 | 免费 | 免费(有额度) | 免费(有额度) | 免费 |
| A股支持 | 全面 | 沪深港通 | 沪深港通 | N/A |
| 港股支持 | 全面 | 全面 | 全面 | N/A |
| 实时行情 | 有(爬虫) | 有(官方) | 有(官方) | N/A |
| 交易能力 | 无 | 有 | 有 | N/A |
| 模拟交易 | 无 | 有 | 有 | N/A |
| 稳定性 | 中(爬虫) | 高(官方API) | 高(官方API) | 高(本地) |
| 基本面数据 | 丰富 | 有限 | 有限 | N/A |
| 北向资金 | 有 | 无 | 无 | N/A |
| 新闻/舆情 | 有 | 无 | 无 | N/A |

### 推荐架构

```
AKShare (数据层)          TA-Lib (计算层)
  ├─ A股/港股历史数据  ──→  技术指标计算
  ├─ 基本面数据              │
  ├─ 北向资金                │
  └─ 新闻舆情                ↓
                         策略信号生成
                              │
                              ↓
              Futu / Longbridge (执行层)
                ├─ 模拟交易验证
                └─ 实盘交易执行
```

---

## 参考链接

- AKShare: https://github.com/akfamily/akshare | https://akshare.akfamily.xyz/data/stock/stock.html
- Futu OpenAPI: https://openapi.futunn.com/futu-api-doc/en/ | https://github.com/FutunnOpen/py-futu-api
- Longbridge: https://open.longbridge.com/docs/getting-started | https://pypi.org/project/longport/
- TA-Lib: https://ta-lib.github.io/ta-lib-python/ | https://github.com/TA-Lib/ta-lib-python
