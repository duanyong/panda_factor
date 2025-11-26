# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

PandaFactor is a quantitative factor development and analysis platform for financial data. It provides high-performance operators for technical indicator calculation, factor construction, and visualization.

**Core Technologies:** Python 3.12, FastAPI, MongoDB, Redis, NumPy, Pandas

## Project Structure

```
panda_factor/
├── panda_common/          # Shared config, logging, database handler
├── panda_data/            # Data access layer (public API)
├── panda_data_hub/        # Data cleaning & auto-update scheduler
├── panda_factor/          # Core factor calculation engine
├── panda_factor_server/   # FastAPI REST API server
├── panda_llm/             # LLM integration (Deepseek/OpenAI)
└── panda_web/             # Vue.js frontend static files
```

## Commands

### Development Setup (uv workspace - Recommended)
```bash
# Install all workspace dependencies (uses pyproject.toml workspace config)
uv sync

# Install with dev dependencies
uv sync --all-extras
```

### Development Setup (uv manual)
```bash
# Install each module in editable mode with uv
uv pip install -e panda_common/
uv pip install -e panda_data/
uv pip install -e panda_data_hub/
uv pip install -e panda_factor/
uv pip install -e panda_llm/
uv pip install -e panda_factor_server/
```

### Development Setup (pip)
```bash
# Install each module in editable mode
pip install -e panda_common/
pip install -e panda_data/
pip install -e panda_data_hub/
pip install -e panda_factor/
pip install -e panda_llm/
pip install -e panda_factor_server/
```

### Running Services
```bash
# API Server (port 8111)
uv run python -m panda_factor_server.__main__

# Data Hub (auto-updates at 20:00 daily)
uv run python -m panda_data_hub._main_auto_

# LLM Service
uv run python -m panda_llm.__main__
```

### Docker
```bash
docker build -f Dockerfile.server -t panda_factor_server .
docker run -p 8111:8111 -e MONGO_URI=host.docker.internal:27017 panda_factor_server
```

## Architecture

### Data Flow
```
External Source (Tushare/RiceQuant/XTQuant)
    ↓
panda_data_hub (clean & validate)
    ↓
MongoDB (stock_market, factors collections)
    ↓
panda_data (read API)
    ↓
panda_factor (calculate)
    ↓
panda_factor_server (REST API)
```

### Data Access API (`panda_data`)
```python
import panda_data
panda_data.init()
panda_data.get_factor(factors, start, end)
panda_data.get_factor_by_name(name, start, end)
panda_data.get_market_data(symbols, start, end)
```

## Factor Development

### Python Mode (Recommended)
```python
from panda_factor.generate.factor_base import Factor

class CustomFactor(Factor):
    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']
        # Use inherited methods: RANK, STDDEV, DELAY, SUM, etc.
        result = self.RANK(close / self.DELAY(close, 20))
        return result  # Must return pd.Series with MultiIndex ['symbol','date']
```

### Formula Mode
```python
"RANK((CLOSE / DELAY(CLOSE, 20)) - 1) * STDDEV((CLOSE / DELAY(CLOSE, 1)) - 1, 20)"
```

### Built-in Functions
- Data: `CLOSE`, `OPEN`, `HIGH`, `LOW`, `VOLUME`, `AMOUNT`, `TURNOVER`, `MARKET_CAP`
- Time Series: `DELAY`, `SUM`, `MEAN`, `STD`, `MAX`, `MIN`, `RANK`
- Returns: `RETURNS`, `CUMSUM`, `CUMPROD`, `PCT_CHANGE`
- Advanced: `CORRELATION`, `COVARIANCE`, `ZSCORE`, `WINSORIZE`, `SCALE`
- Logic: `IF`, `ABS`, `LOG`, `POWER`, `SIGN`

## Key Conventions

### MultiIndex Format
All factor/data Series must use `pd.MultiIndex(['symbol', 'date'])`:
```python
index = pd.MultiIndex.from_product([symbols, dates], names=['symbol', 'date'])
series = pd.Series(data, index=index)
```

### Date Format
Dates use `YYYYMMDD` string format (e.g., `'20240320'`).

### Symbol Format
Shanghai: `SSXXXX`, Shenzhen: `SZXXXX`

### Database Handler
Uses singleton pattern - access via `DatabaseHandler()`.

### Configuration
`panda_common/config.yaml` with environment variable override support:
- `MONGO_URI` - MongoDB connection
- `DATASOURCE` - Primary data source (tqsdk, tushare, ricequant, xtquant)
- `LLM_API_KEY`, `LLM_MODEL`, `LLM_BASE_URL` - LLM configuration

## Key Files

- `panda_common/config.yaml` - Central configuration
- `panda_factor/generate/factor_base.py` - Factor abstract base class
- `panda_factor/generate/factor_utils.py` - 40+ built-in factor functions
- `panda_data/__init__.py` - Public data access API
- `panda_factor_server/routes/user_factor_pro.py` - REST API endpoints
