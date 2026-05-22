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
