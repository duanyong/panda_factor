# PandaFactor 入门教程

## 目录
1. [系统概述](#系统概述)
2. [环境安装与配置](#环境安装与配置)
3. [快速开始](#快速开始)
4. [因子开发详解](#因子开发详解)
5. [数据获取API](#数据获取api)
6. [因子分析](#因子分析)
7. [最佳实践](#最佳实践)
8. [常见问题](#常见问题)

## 系统概述

PandaFactor是一个专业的量化因子开发和分析平台，为金融数据分析、技术指标计算和因子构建提供高性能工具。

### 核心特性
- **高性能因子计算**: 基于NumPy和Pandas优化的量化算子
- **多种数据源支持**: Tushare、RiceQuant、迅投等主流数据源
- **灵活的因子开发**: 支持Python类和公式两种开发方式
- **完整的分析工具**: IC分析、分层回测、相关性分析等
- **REST API服务**: 提供完整的因子管理和查询接口
- **自动化数据更新**: 定时数据清洗和因子更新

### 技术架构
```
数据源 (Tushare/RiceQuant/XTQuant)
    ↓
panda_data_hub (数据清洗与更新)
    ↓
MongoDB (stock_market, factors 集合)
    ↓
panda_data (数据读取API)
    ↓
panda_factor (因子计算引擎)
    ↓
panda_factor_server (REST API服务)
```

## 环境安装与配置

### 系统要求
- Python 3.12+
- MongoDB 4.4+
- 内存: 建议8GB+
- 磁盘空间: 50GB+ (用于历史数据存储)

### 方式一：快速安装 (推荐)

```bash
# 克隆项目
git clone https://github.com/your-repo/panda_factor.git
cd panda_factor

# 使用uv工作空间安装所有依赖
uv sync --all-extras

# 启动数据库 (如果使用预置数据库)
# 下载并解压预置数据库，执行 bin/db_start.bat
```

### 方式二：手动安装

```bash
# 安装各个模块
pip install -e panda_common/
pip install -e panda_data/
pip install -e panda_data_hub/
pip install -e panda_factor/
pip install -e panda_llm/
pip install -e panda_factor_server/

# 或者使用uv
uv pip install -e panda_common/
uv pip install -e panda_data/
# ... 其他模块类似
```

### 配置文件设置

编辑 `panda_common/config.yaml`：

```yaml
# 数据库配置
MONGO_USER: "panda"
MONGO_PASSWORD: "panda"
MONGO_URI: "127.0.0.1:27017"
MONGO_DB: "panda"
MONGO_TYPE: "single"  # 或 "replica_set"

# 数据源配置
DATASOURCE: "tushare"  # 可选: tushare, ricequant, xtquant, tqsdk
TS_TOKEN: "your_tushare_token"  # 如果使用Tushare
MUSER: "your_ricequant_username"  # 如果使用RiceQuant
MPASSWORD: "your_ricequant_password"

# LLM配置 (可选)
LLM_API_KEY: "your_api_key"
LLM_MODEL: "deepseek-chat"
LLM_BASE_URL: "https://api.deepseek.com/v1"

# 日志配置
LOG_LEVEL: "INFO"
```

## 快速开始

### 1. 启动服务

```bash
# 启动API服务器 (端口 8111)
uv run python -m panda_factor_server.__main__

# 启动数据更新服务 (可选，用于自动更新数据)
uv run python -m panda_data_hub._main_auto_
```

### 2. 初始化数据访问

```python
import panda_data

# 初始化系统
panda_data.init()

# 获取基础市场数据
market_data = panda_data.get_market_data(
    start_date='20240101',
    end_date='20240320',
    symbols=['000001.SZ', '600000.SS'],
    fields=['open', 'close', 'high', 'low', 'volume']
)

print(market_data.head())
```

### 3. 创建第一个因子

```python
from panda_factor.generate.factor_base import Factor

# 简单动量因子
class MomentumFactor(Factor):
    def calculate(self, factors):
        close = factors['close']
        # 计算20日收益率
        returns = (close / self.DELAY(close, 20)) - 1
        # 横截面排名
        result = self.RANK(returns)
        return result
```

## 因子开发详解

### Python模式 (推荐)

Python模式提供更好的可维护性和扩展性，适合复杂因子开发。

#### 基础语法结构

```python
from panda_factor.generate.factor_base import Factor

class CustomFactor(Factor):
    def calculate(self, factors):
        # 获取基础数据
        close = factors['close']
        open = factors['open']
        volume = factors['volume']

        # 因子计算逻辑
        # ...

        return result  # 必须返回pd.Series，索引为MultiIndex(['symbol', 'date'])
```

#### 可用基础数据字段

- **价格数据**: `close`, `open`, `high`, `low`
- **成交量数据**: `volume`, `amount`, `turnover`
- **市值数据**: `market_cap`, `circulating_market_cap`
- **其他指标**: `pe_ratio`, `pb_ratio`, `dividend_yield`

#### 内置函数 (部分常用函数)

```python
# 时间序列函数
self.DELAY(series, n)        # 延迟n期
self.SUM(series, n)           # n期求和
self.MEAN(series, n)          # n期均值
self.STDDEV(series, n)        # n期标准差
self.MAX(series, n)           # n期最大值
self.MIN(series, n)           # n期最小值

# 收益率函数
self.RETURNS(series, n)       # n期收益率
self.FUTURE_RETURNS(series, n)  # n期未来收益率

# 截面函数
self.RANK(series)              # 横截面排名 ([-0.5, 0.5])
self.SCALE(series)             # 标准化

# 逻辑函数
self.IF(condition, x, y)       # 条件判断
self.ABS(series)               # 绝对值
self.LOG(series)               # 对数
self.POWER(series, n)          # n次方
self.SIGN(series)              # 符号函数

# 高级函数
self.CORRELATION(series1, series2, n)  # n期相关性
self.COVARIANCE(series1, series2, n)    # n期协方差
self.ZSCORE(series, n)                  # n期Z-score
self.WINSORIZE(series, n)               # n期缩尾处理
```

#### 复杂因子示例

```python
class ComplexFactor(Factor):
    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']
        high = factors['high']
        low = factors['low']

        # 20日收益率
        returns = (close / self.DELAY(close, 20)) - 1
        # 20日波动率
        volatility = self.STDDEV((close / self.DELAY(close, 1)) - 1, 20)
        # 价格振幅
        price_range = (high - low) / close
        # 成交量比率
        volume_ratio = volume / self.DELAY(volume, 1)
        # 20日成交量均值
        volume_ma = self.SUM(volume, 20) / 20
        # 动量信号
        momentum = self.RANK(returns)
        # 波动率信号
        vol_signal = self.IF(volatility > self.DELAY(volatility, 1), 1, -1)
        # 最终因子
        result = momentum * vol_signal * self.SCALE(volume_ratio / volume_ma)
        return result
```

### 公式模式

公式模式适合快速原型开发和简单因子，使用类SQL语法。

#### 基础语法

```python
# 简单因子公式
"RANK((CLOSE / DELAY(CLOSE, 20)) - 1)"

# 多行公式 (最后一行作为因子值)
returns = "(CLOSE / DELAY(CLOSE, 20)) - 1"
momentum = "RANK(returns)"
volatility = "STDDEV((CLOSE / DELAY(CLOSE, 1)) - 1, 20)"
final_factor = "momentum * volatility"
```

#### 公式函数对照表

| Python函数 | 公式函数 | 说明 |
|-----------|----------|------|
| `close` | `CLOSE` | 收盘价 |
| `volume` | `VOLUME` | 成交量 |
| `self.DELAY(series, n)` | `DELAY(series, n)` | 延迟 |
| `self.RANK(series)` | `RANK(series)` | 排名 |
| `self.STDDEV(series, n)` | `STDDEV(series, n)` | 标准差 |
| `self.SUM(series, n)` | `SUM(series, n)` | 求和 |
| `self.IF(condition, x, y)` | `IF(condition, x, y)` | 条件 |

#### 复杂公式示例

```python
# 复杂因子组合
"""
RANK((CLOSE / DELAY(CLOSE, 20)) - 1) *
STDDEV((CLOSE / DELAY(CLOSE, 1)) - 1, 20) *
IF(CLOSE > DELAY(CLOSE, 1), 1, -1)
"""

# 价格成交量相关性
"""
CORRELATION(CLOSE, VOLUME, 20)
"""

# 多步骤计算
"""
returns = (CLOSE / DELAY(CLOSE, 5)) - 1
volume_ma = SUM(VOLUME, 20) / 20
volume_ratio = VOLUME / volume_ma
momentum = RANK(returns)
final_result = momentum * SCALE(volume_ratio)
"""
```

## 数据获取API

### 基础市场数据

```python
import panda_data

# 初始化
panda_data.init()

# 获取日级数据
daily_data = panda_data.get_market_data(
    start_date='20240101',
    end_date='20240320',
    symbols=['000001.SZ', '600000.SS'],
    fields=['open', 'close', 'high', 'low', 'volume', 'amount']
)

# 获取分钟级数据
minute_data = panda_data.get_market_min_data(
    start_date='20240320',
    end_date='20240320',
    symbol='000001.SZ',
    fields=['close', 'volume']
)

# 获取所有股票代码
all_symbols = panda_data.get_all_symbols()
```

### 因子数据获取

```python
# 获取预定义因子
factor_data = panda_data.get_factor(
    factors=['momentum_20d', 'volatility_20d'],
    start_date='20240101',
    end_date='20240320'
)

# 按名称获取因子
single_factor = panda_data.get_factor_by_name(
    factor_name="VH03cc651",  # 因子ID
    start_date='20240320',
    end_date='20250325'
)
```

### 数据格式说明

所有返回的数据都使用标准的MultiIndex格式：

```python
# 数据结构示例
index = pd.MultiIndex.from_product(
    [['000001.SZ', '600000.SS'], ['20240101', '20240102']],
    names=['symbol', 'date']
)
data = pd.Series([1.2, 1.3, 0.8, 0.9], index=index)

# 访问特定股票的数据
stock_data = data.loc['000001.SZ']

# 访问特定日期的数据
date_data = data.xs('20240101', level='date')
```

## 因子分析

### 因子性能评估

PandaFactor提供完整的因子分析工具：

```python
from panda_factor.analysis.factor_analysis import FactorAnalysis

# 创建分析器
analyzer = FactorAnalysis()

# 计算因子收益
returns_data = analyzer.calculate_factor_returns(
    factor_values=factor_data,
    price_data=market_data['close'],
    periods=[1, 5, 10, 20]  # 持有期
)

# IC分析
ic_analysis = analyzer.ic_analysis(
    factor_values=factor_data,
    returns_data=returns_data
)

# 分层回测
layer_analysis = analyzer.layer_backtest(
    factor_values=factor_data,
    returns_data=returns_data,
    layers=5  # 分5层
)
```

### 可视化分析

```python
import matplotlib.pyplot as plt

# IC衰减分析
analyzer.plot_ic_decay(ic_data)

# 因子分布
analyzer.plot_factor_distribution(factor_data)

# 分层收益曲线
analyzer.plot_layer_returns(layer_analysis)
```

## 最佳实践

### 1. 因子开发原则

#### 保持简单性
- 优先选择简单、直观的因子逻辑
- 避免过度复杂的数学变换
- 遵循Golang设计哲学：简单性优于复杂性

#### 数据完整性处理
```python
class RobustFactor(Factor):
    def calculate(self, factors):
        close = factors['close']

        # 处理缺失值
        close = close.fillna(method='ffill')  # 前向填充

        # 检查数据充足性
        if len(close) < 20:
            return pd.Series(0, index=close.index)

        # 因子计算
        result = self.RANK((close / self.DELAY(close, 20)) - 1)

        # 异常值处理
        result = self.WINSORIZE(result, 5)

        return result
```

#### 性能优化
```python
# 使用向量化操作，避免循环
def calculate(self, factors):
    close = factors['close']
    volume = factors['volume']

    # 好的做法：向量化操作
    returns = close.pct_change(20)
    volume_ratio = volume / volume.rolling(20).mean()

    # 避免：逐股票循环
    # for symbol in close.index.get_level_values('symbol').unique():
    #     symbol_data = close.loc[symbol]
    #     # 逐个计算...

    return self.RANK(returns) * self.SCALE(volume_ratio)
```

### 2. 因子验证

#### 样本外测试
```python
def validate_factor(factor_class, train_period, test_period):
    # 训练期分析
    train_data = calculate_factor(factor_class, train_period)
    train_ic = calculate_ic(train_data)

    # 测试期验证
    test_data = calculate_factor(factor_class, test_period)
    test_ic = calculate_ic(test_data)

    # 比较结果
    return {
        'train_ic_mean': train_ic.mean(),
        'test_ic_mean': test_ic.mean(),
        'ic_decay': train_ic.mean() - test_ic.mean()
    }
```

#### 相关性检查
```python
def check_factor_correlation(new_factor, existing_factors):
    correlations = {}
    for name, factor_data in existing_factors.items():
        corr = new_factor.corr(factor_data)
        correlations[name] = corr
    return correlations
```

### 3. 因子组合

#### 简单等权组合
```python
def combine_factors(factors_dict, weights=None):
    if weights is None:
        weights = {name: 1.0/len(factors_dict) for name in factors_dict.keys()}

    combined = pd.Series(0, index=next(iter(factors_dict.values())).index)
    for name, factor_data in factors_dict.items():
        combined += weights[name] * factor_data

    return combined
```

## 常见问题

### Q1: 如何处理停牌股票的数据缺失？
```python
def handle_missing_data(factors):
    # 前向填充价格数据
    price_fields = ['open', 'high', 'low', 'close']
    for field in price_fields:
        if field in factors:
            factors[field] = factors[field].groupby('symbol').fillna(method='ffill')

    # 成交量缺失填充为0
    if 'volume' in factors:
        factors['volume'] = factors['volume'].fillna(0)

    return factors
```

### Q2: 如何提高因子计算性能？
```python
# 使用pandas的groupby优化
def efficient_rank(series):
    return series.groupby('date').rank(pct=True) - 0.5

# 避免不必要的中间变量
def calculate_momentum(close, period=20):
    return (close / close.groupby('symbol').shift(period)) - 1
```

### Q3: 如何处理因子计算中的未来函数？
```python
# 确保只使用当前及之前的数据
def safe_factor_calculation(factors, current_date):
    # 过滤到当前日期
    current_data = {k: v.loc[v.index.get_level_values('date') <= current_date]
                   for k, v in factors.items()}

    # 进行因子计算
    # ...
```

### Q4: 如何调试因子计算错误？
```python
class DebugFactor(Factor):
    def calculate(self, factors):
        close = factors['close']

        # 打印数据形状
        print(f"Close data shape: {close.shape}")
        print(f"Date range: {close.index.get_level_values('date').min()} - {close.index.get_level_values('date').max()}")

        # 检查缺失值
        missing_count = close.isna().sum()
        print(f"Missing values: {missing_count}")

        # 逐步计算并检查
        delayed_close = self.DELAY(close, 20)
        returns = (close / delayed_close) - 1
        print(f"Returns statistics: mean={returns.mean():.4f}, std={returns.std():.4f}")

        result = self.RANK(returns)
        return result
```

### Q5: 如何保存和加载自定义因子？
```python
import pickle

# 保存因子
def save_factor(factor_instance, filename):
    with open(filename, 'wb') as f:
        pickle.dump(factor_instance, f)

# 加载因子
def load_factor(filename):
    with open(filename, 'rb') as f:
        return pickle.load(f)

# 或者保存因子计算结果
def save_factor_results(factor_data, filename):
    factor_data.to_csv(filename)
```

## 总结

PandaFactor提供了一个功能完整、性能优秀的量化因子开发平台。通过本教程，您应该能够：

1. **搭建环境**: 完成系统安装和配置
2. **开发因子**: 使用Python类或公式方式创建自定义因子
3. **获取数据**: 通过API访问历史市场数据和因子数据
4. **分析验证**: 评估因子性能和进行回测分析
5. **优化部署**: 遵循最佳实践开发高质量因子

### 下一步学习建议

1. **深入因子理论**: 学习经典因子模型和量化投资理论
2. **机器学习集成**: 探索将PandaFactor与ML模型结合使用
3. **实盘交易**: 了解如何将因子集成到实际交易系统中
4. **社区贡献**: 参与开源项目，分享您的因子和经验

更多资源和最新更新请关注项目GitHub和官方文档。