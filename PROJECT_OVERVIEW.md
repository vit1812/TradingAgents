# TradingAgents Project Overview

## Tong quan

TradingAgents la mot Python package dung de phan tich giao dich tai chinh bang multi-agent LLM. Project su dung LangGraph de dieu phoi cac agent, LangChain de ket noi LLM/tool, va cac nguon du lieu nhu yfinance hoac Alpha Vantage.

Project co 2 cach chay chinh:

- CLI tuong tac: `tradingagents` hoac `python -m cli.main`
- Goi truc tiep trong Python qua `TradingAgentsGraph().propagate(...)`

## Cau truc thu muc

```text
TradingAgents/
├── main.py                    # Vi du chay truc tiep bang Python
├── pyproject.toml             # Metadata package, dependencies, CLI entrypoint
├── requirements.txt           # Danh sach dependency theo pip
├── Dockerfile                 # Docker image
├── docker-compose.yml         # Chay bang Docker Compose
├── cli/                       # Giao dien dong lenh
├── tradingagents/             # Package loi
│   ├── agents/                # Cac agent phan tich, tranh luan, quan ly
│   ├── graph/                 # Dieu phoi workflow bang LangGraph
│   ├── dataflows/             # Lay du lieu thi truong, news, fundamentals
│   ├── llm_clients/           # Adapter cho cac LLM provider
│   └── default_config.py      # Cau hinh mac dinh va env overrides
├── tests/                     # Test suite
├── scripts/                   # Script phu tro
└── assets/                    # Hinh anh cho README va CLI
```

## Luong chay chinh

Workflow nghiep vu nam chu yeu trong `tradingagents/graph/trading_graph.py`.

Khi chay mot phien phan tich:

1. Khoi tao `TradingAgentsGraph`.
2. Nap config tu `tradingagents/default_config.py`.
3. Tao LLM clients qua `tradingagents/llm_clients/factory.py`.
4. Dung LangGraph workflow trong `tradingagents/graph/setup.py`.
5. Tao state ban dau trong `tradingagents/graph/propagation.py`.
6. Chay lan luot cac agent phan tich, nghien cuu, trader, risk, portfolio.
7. Ghi log, memory log, va tra ve quyet dinh cuoi.

Luong agent tong quat:

```text
Analyst Team
  ├─ Market Analyst
  ├─ Sentiment Analyst
  ├─ News Analyst
  └─ Fundamentals Analyst

        ↓

Research Team
  ├─ Bull Researcher
  ├─ Bear Researcher
  └─ Research Manager

        ↓

Trader

        ↓

Risk Management
  ├─ Aggressive Analyst
  ├─ Conservative Analyst
  └─ Neutral Analyst

        ↓

Portfolio Manager
  └─ final_trade_decision
```

## Luong Analyst chi tiet

Analyst Team la tang dau tien cua pipeline. Tang nay thu thap va chuan hoa 4 nhom insight dau vao truoc khi chuyen sang Research Team:

- Technical/market insight tu `Market Analyst`
- Sentiment insight tu `Sentiment Analyst`
- News/macro insight tu `News Analyst`
- Fundamental insight tu `Fundamentals Analyst`

Danh sach analyst duoc chon tu CLI hoac khi khoi tao `TradingAgentsGraph(selected_analysts=...)`. CLI chuan hoa thu tu theo `ANALYST_ORDER = ["market", "social", "news", "fundamentals"]`, sau do `tradingagents/graph/analyst_execution.py` tao `AnalystExecutionPlan`.

Mapping noi bo cua Analyst layer:

```text
market       -> Market Analyst        -> tools_market       -> market_report
social       -> Sentiment Analyst     -> tools_social       -> sentiment_report
news         -> News Analyst          -> tools_news         -> news_report
fundamentals -> Fundamentals Analyst  -> tools_fundamentals -> fundamentals_report
```

Key `social` van duoc giu de backward-compatible voi saved config va caller cu, nhung ten hien thi va node hien tai la `Sentiment Analyst`.

State ban dau duoc tao trong `tradingagents/graph/propagation.py`:

```text
messages = [("human", ticker)]
company_of_interest
asset_type
trade_date
past_context
market_report = ""
sentiment_report = ""
news_report = ""
fundamentals_report = ""
investment_debate_state
risk_debate_state
```

`tradingagents/graph/setup.py` dung `StateGraph` de them moi analyst thanh 3 node:

```text
Agent node: Market Analyst / Sentiment Analyst / News Analyst / Fundamentals Analyst
Tool node:  tools_market / tools_social / tools_news / tools_fundamentals
Clear node: Msg Clear Market / Msg Clear Sentiment / Msg Clear News / Msg Clear Fundamentals
```

Luong edge cua moi analyst:

```text
START
  -> Analyst 1
      -> neu LLM tra ve tool_calls: tools_x -> quay lai Analyst 1
      -> neu khong con tool_calls: Msg Clear X
  -> Analyst 2
  -> ...
  -> Analyst cuoi
  -> Bull Researcher
```

Dieu kien nay nam trong `tradingagents/graph/conditional_logic.py`: moi `should_continue_xxx` kiem tra `state["messages"][-1].tool_calls`. Neu co tool call thi di sang `ToolNode`; neu khong thi di sang clear node va tiep tuc analyst ke tiep.

Hien tai `analyst_concurrency_limit` duoc luu trong `AnalystExecutionPlan`, nhung graph van noi cac analyst theo chuoi trong `setup.py`. Nghia la cac analyst chay tuan tu, chua chay song song.

Chi tiet tung analyst:

- `Market Analyst` (`tradingagents/agents/analysts/market_analyst.py`): dung `get_stock_data` va `get_indicators`, yeu cau lay du lieu gia truoc roi chon cac technical indicators phu hop, ghi ket qua vao `market_report`.
- `Sentiment Analyst` (`tradingagents/agents/analysts/sentiment_analyst.py`): prefetch Yahoo Finance news, StockTwits, va Reddit, inject du lieu vao prompt, khong dung `bind_tools`, goi LLM mot lan va ghi vao `sentiment_report`.
- `News Analyst` (`tradingagents/agents/analysts/news_analyst.py`): dung `get_news` va `get_global_news` de phan tich tin lien quan den ticker/asset va macro, ghi vao `news_report`.
- `Fundamentals Analyst` (`tradingagents/agents/analysts/fundamentals_analyst.py`): dung `get_fundamentals`, `get_balance_sheet`, `get_cashflow`, va `get_income_statement`, ghi vao `fundamentals_report`.

### Market Analyst

`Market Analyst` la node technical-analysis nam trong `tradingagents/agents/analysts/market_analyst.py`. Node nay duoc tao bang `create_market_analyst(llm)` va co luong noi bo:

```text
state
  -> doc trade_date, asset_type, company_of_interest
  -> tao instrument_context de giu dung ticker/suffix
  -> bind tools: get_stock_data, get_indicators
  -> LLM quyet dinh tool_calls
  -> neu con tool_calls: tools_market chay tool roi quay lai Market Analyst
  -> neu khong con tool_calls: ghi result.content vao market_report
```

Prompt cua `Market Analyst` yeu cau chon toi da 8 technical indicators co tinh bo sung nhau, tranh tin hieu trung lap, sau do viet report chi tiet va them bang Markdown cuoi bai. Cac indicator duoc prompt neu ro:

```text
Moving averages: close_50_sma, close_200_sma, close_10_ema
MACD:            macd, macds, macdh
Momentum:        rsi
Volatility:      boll, boll_ub, boll_lb, atr
Volume:          vwma
```

Tool `get_stock_data` nam trong `tradingagents/agents/utils/core_stock_tools.py`. Tool nay route qua `route_to_vendor("get_stock_data", symbol, start_date, end_date)` de lay OHLCV theo khoang ngay. Vendor mac dinh la `yfinance`; implementation `get_YFin_data_online` trong `tradingagents/dataflows/y_finance.py` goi `yf.Ticker(symbol.upper()).history(...)`, lam sach timezone, round cac cot gia, va tra ve CSV string kem metadata.

Tool `get_indicators` nam trong `tradingagents/agents/utils/technical_indicators_tools.py`. Tool nay route qua `route_to_vendor("get_indicators", symbol, indicator, curr_date, look_back_days)`. Mac dinh `look_back_days = 30`. Neu LLM truyen nhieu indicator bang chuoi phan tach dau phay, code se split va chay tung indicator rieng.

Vendor routing nam trong `tradingagents/dataflows/interface.py`:

```text
get_stock_data:
  yfinance      -> get_YFin_data_online
  alpha_vantage -> get_alpha_vantage_stock

get_indicators:
  yfinance      -> get_stock_stats_indicators_window
  alpha_vantage -> get_alpha_vantage_indicator
```

Config mac dinh trong `tradingagents/default_config.py` dung `yfinance` cho ca `core_stock_apis` va `technical_indicators`. Co the override theo category bang `data_vendors` hoac theo tung tool bang `tool_vendors`; `tool_vendors` co uu tien cao hon.

Voi `yfinance`, indicator duoc tinh bang `stockstats`. Ham `load_ohlcv` trong `tradingagents/dataflows/stockstats_utils.py` download/cache khoang 5 nam OHLCV vao `data_cache_dir`, sau do loc `Date <= curr_date` de tranh look-ahead bias. `get_stock_stats_indicators_window` tinh indicator cho tung ngay trong cua so lookback va tra ve chuoi gia tri kem mo ta indicator.

Mot so luu y implementation:

- Prompt yeu cau LLM goi `get_stock_data` truoc, nhung day la instruction cho LLM, khong phai constraint bat buoc trong code.
- `get_indicators` khong su dung truc tiep CSV tu `get_stock_data`; no tu route/fetch/cache du lieu rieng qua vendor.
- Yfinance implementation con support them `mfi`, nhung prompt hien khong liet ke `mfi`, nen LLM thuong se khong chon indicator nay.
- `market_report` chi duoc ghi khi response cuoi cua LLM khong con `tool_calls`; neu response van la tool call thi report rong va graph tiep tuc vong tool.

Sau khi Analyst Team hoan tat, cac report trong `AgentState` duoc Research Team, Trader, Risk Management, va Portfolio Manager su dung de tranh luan va tao `final_trade_decision`.

## CLI

CLI nam trong `cli/main.py`, dung:

- `typer` de khai bao command
- `questionary` de hoi input tu nguoi dung
- `rich` de hien thi giao dien terminal

Entrypoint duoc khai bao trong `pyproject.toml`:

```toml
[project.scripts]
tradingagents = "cli.main:app"
```

Lenh chinh:

```bash
tradingagents analyze
tradingagents analyze --checkpoint
tradingagents analyze --clear-checkpoints
```

## Graph layer

Thu muc `tradingagents/graph/` la phan dieu phoi workflow.

- `trading_graph.py`: class orchestration chinh, khoi tao LLM, tools, graph, chay propagate.
- `analyst_execution.py`: dinh nghia analyst spec, execution plan, node/report mapping, va wall-time tracker cho analyst.
- `setup.py`: dung `StateGraph`, them node agent, tool node, conditional edges.
- `propagation.py`: tao initial state va cau hinh invoke graph.
- `conditional_logic.py`: quyet dinh khi nao agent goi tool, chuyen node, hoac dung debate.
- `checkpointer.py`: checkpoint/resume bang LangGraph + sqlite.
- `reflection.py`: tao reflection tu ket qua giao dich cu.
- `signal_processing.py`: rut gon quyet dinh giao dich cuoi.

State dung chung duoc dinh nghia trong `tradingagents/agents/utils/agent_states.py`.

## Agents layer

Thu muc `tradingagents/agents/` chua cac agent:

```text
agents/
├── analysts/
│   ├── market_analyst.py
│   ├── sentiment_analyst.py
│   ├── news_analyst.py
│   └── fundamentals_analyst.py
├── researchers/
│   ├── bull_researcher.py
│   └── bear_researcher.py
├── trader/
│   └── trader.py
├── risk_mgmt/
│   ├── aggressive_debator.py
│   ├── conservative_debator.py
│   └── neutral_debator.py
├── managers/
│   ├── research_manager.py
│   └── portfolio_manager.py
└── utils/
```

Moi agent thuong co dang `create_xxx(llm)`, tra ve mot LangGraph node function. Agent nhan state, goi LLM/tool neu can, roi cap nhat mot phan state nhu `market_report`, `investment_plan`, `trader_investment_plan`, hoac `final_trade_decision`.

## Dataflows layer

Thu muc `tradingagents/dataflows/` phu trach lay du lieu. `interface.py` la lop routing trung tam giua cac vendor.

Vendor hien co:

- `yfinance`
- `alpha_vantage`

Nhom du lieu:

- OHLCV stock data
- Technical indicators
- Fundamentals
- Balance sheet
- Cashflow
- Income statement
- News
- Global news
- Insider transactions

Config vendor nam trong `DEFAULT_CONFIG["data_vendors"]`.

## LLM clients

Thu muc `tradingagents/llm_clients/` gom cac adapter ket noi provider LLM.

Provider duoc ho tro:

- OpenAI
- Anthropic
- Google Gemini
- Azure OpenAI
- xAI
- DeepSeek
- Qwen / Qwen CN
- GLM / GLM CN
- MiniMax / MiniMax CN
- OpenRouter
- Ollama

Factory trung tam la `create_llm_client(...)` trong `tradingagents/llm_clients/factory.py`.

## Config va bien moi truong

Config mac dinh nam trong `tradingagents/default_config.py`.

Mot so env var quan trong:

- `TRADINGAGENTS_LLM_PROVIDER`
- `TRADINGAGENTS_DEEP_THINK_LLM`
- `TRADINGAGENTS_QUICK_THINK_LLM`
- `TRADINGAGENTS_LLM_BACKEND_URL`
- `TRADINGAGENTS_OUTPUT_LANGUAGE`
- `TRADINGAGENTS_MAX_DEBATE_ROUNDS`
- `TRADINGAGENTS_MAX_RISK_ROUNDS`
- `TRADINGAGENTS_CHECKPOINT_ENABLED`
- `TRADINGAGENTS_RESULTS_DIR`
- `TRADINGAGENTS_CACHE_DIR`
- `TRADINGAGENTS_MEMORY_LOG_PATH`

Mac dinh du lieu runtime duoc ghi vao:

```text
~/.tradingagents/logs
~/.tradingagents/cache
~/.tradingagents/memory/trading_memory.md
```

## Tests

Thu muc `tests/` gom cac test cho:

- Config va env overrides
- LLM provider/model validation
- API key handling
- Checkpoint resume
- Structured output
- Ticker safety
- Crypto asset mode
- Analyst execution
- Memory log
- Dataflow config

Chay test:

```bash
pytest
```

## Ket luan

Project nay co kien truc theo tang kha ro:

- `cli/`: giao dien nguoi dung tren terminal
- `tradingagents/graph/`: dieu phoi workflow multi-agent
- `tradingagents/agents/`: logic tung agent
- `tradingagents/dataflows/`: lay va route du lieu tai chinh
- `tradingagents/llm_clients/`: ket noi cac LLM provider
- `tests/`: dam bao cac hanh vi chinh

Day la mot framework multi-agent trading research, khong phai mot trading bot dat lenh that. Ket qua phan tich mang tinh nghien cuu va can duoc kiem chung rieng truoc khi su dung trong thuc te.
