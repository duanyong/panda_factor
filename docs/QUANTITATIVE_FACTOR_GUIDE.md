# PandaFactor 量化因子计算完全指南

## 目录
1. [系统架构概述](#系统架构概述)
2. [因子开发方式](#因子开发方式)
3. [内置函数详解](#内置函数详解)
4. [技术指标函数](#技术指标函数)
5. [常用因子模板](#常用因子模板)
6. [高级因子技巧](#高级因子技巧)
7. [因子计算实践](#因子计算实践)
8. [性能优化建议](#性能优化建议)

## 系统架构概述

### 核心组件架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PandaFactor量化因子计算平台                    │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │   Python类模式  │  │   公式模式     │  │   数据处理层     │    │
│  │   (Factor基类)   │  │   (字符串公式)   │  │   (MarketDataReader) │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │   因子计算引擎   │  │   内置函数库   │  │   因子分析工具     │    │
│  │   (FactorUtils)   │  │   (40+函数)     │  │   (IC/回测分析)   │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │   数据存储管理   │  │   Web API接口   │  │   可视化展示       │    │
│  │   (MongoDB)       │  │   (RESTful)     │  │   (图表/统计)     │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### 数据模型设计

```python
# 核心数据字段
BASE_FIELDS = {
    'close': '收盘价',      # 基础OHLC数据
    'open': '开盘价',
    'high': '最高价',
    'low': '最低价',
    'volume': '成交量',      # 成交量(股)
    'amount': '成交额',      # 成交额(元)
    'pre_close': '昨收价',   # 前一日收盘价
    'turnover': '换手率',     # 换手率(%)
    'market_cap': '总市值',   # 总市值(元)
    'pe_ratio': '市盈率',    # 市盈率
    'pb_ratio': '市净率',    # 市净率
    'industry': '行业',       # 行业分类
    'index_component': '指数成分'  # 指数成分股标识
}

# 数据索引结构
INDEX_STRUCTURE = pd.MultiIndex.from_arrays(
    [symbols, dates],           # 股票代码和日期数组
    names=['symbol', 'date']      # MultiIndex列名
)
```

## 因子开发方式

### 方式一：Python类模式（推荐）

Python模式提供更强的可扩展性和更好的错误处理，适合复杂因子开发。

#### 基础结构

```python
from panda_factor.generate.factor_base import Factor
from panda_factor.generate.factor_utils import FactorUtils
import pandas as pd
import numpy as np

class MomentumFactor(Factor):
    """
    动量因子：计算N期收益率并标准化

    逻辑：(当前价格 / N期前价格) - 1
    优势：简单直观，反应价格趋势
    缺点：容易受短期噪声影响
    """

    def calculate(self, factors):
        """
        因子计算主函数

        Args:
            factors: Dict[str, pd.Series] - 包含基础数据的字典
                     键为字段名，值为MultiIndex格式的Series

        Returns:
            pd.Series: 因子值，索引为MultiIndex(['symbol', 'date'])
        """
        # 获取收盘价
        close = factors['close']

        # 计算20期收益率
        returns = (close / self.DELAY(close, 20)) - 1

        # 横截面排名标准化到[-0.5, 0.5]范围
        result = self.RANK(returns)

        return result

# 使用示例
factor_instance = MomentumFactor()
result = factor_instance.calculate(factors_dict)
```

#### 复合因子示例

```python
class VolumePriceMomentumFactor(Factor):
    """
    价量动量因子：结合价格动量和成交量变化

    逻辑：价格动量 × 成交量变化率
    优势：考虑交易活跃度，过滤虚假信号
    应用：适合趋势跟踪策略
    """

    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']

        # 价格动量 (20日)
        price_momentum = self.RANK(
            (close / self.DELAY(close, 20)) - 1
        )

        # 成交量变化率 (20日)
        volume_ratio = volume / self.SUM(volume, 20)
        volume_rank = self.RANK(volume_ratio)

        # 复合因子：价量动量的加权平均
        result = (price_momentum + volume_rank) / 2

        return result
```

### 方式二：公式模式（快速开发）

公式模式使用字符串表达式，适合快速原型开发和简单因子。

#### 基础语法

```python
# 1. 简单因子公式
simple_momentum = "RANK((CLOSE / DELAY(CLOSE, 20)) - 1)"

# 2. 多步计算
multi_step = """
returns = (CLOSE / DELAY(CLOSE, 20)) - 1
volatility = STDDEV(DELTA(CLOSE, 1), 20)
momentum = RANK(returns) * IF(volatility > STDDEV(DELTA(CLOSE, 1), 20), 1, -1)
"""

# 3. 复合技术指标
complex_indicator = """
# 布林带突破
upper = MA(CLOSE, 20) + 2 * STDDEV(CLOSE, 20)
lower = MA(CLOSE, 20) - 2 * STDDEV(CLOSE, 20)
bollinger_breakout = IF(CLOSE > upper, 1, IF(CLOSE < lower, -1, 0))
"""

# 4. 条件逻辑
conditional_factor = """
# 多重时间框架动量
momentum_5d = RANK((CLOSE / DELAY(CLOSE, 5)) - 1)
momentum_20d = RANK((CLOSE / DELAY(CLOSE, 20)) - 1)
momentum_60d = RANK((CLOSE / DELAY(CLOSE, 60)) - 1)

# 加权组合
final_factor = 0.5 * momentum_5d + 0.3 * momentum_20d + 0.2 * momentum_60d
"""
```

## 内置函数详解

### Level 0: 基础数学函数

```python
# 基础数学运算
abs_value = self.ABS(series)                    # 绝对值：|x|
log_value = self.LOG(series)                      # 自然对数：ln(x)
power_value = self.POWER(series, 2)              # 幂运算：x²
sqrt_value = self.SQRT(series)                    # 平方根：√x
sign_value = self.SIGN(series)                    # 符号函数：sign(x)

# 比较运算
min_value = self.MIN(series1, series2)          # 最小值：min(x1, x2)
max_value = self.MAX(series1, series2)          # 最大值：max(x1, x2)

# 条件选择
result = self.IF(condition, true_value, false_value)  # 条件：condition ? true_value : false_value
```

### Level 1: 时间序列函数

```python
# 时间延迟和差分
delayed = self.DELAY(series, 5)              # 延迟5期：x[t-5]
diff_1 = self.DELTA(series, 1)                # 一阶差分：x[t] - x[t-1]

# 移动统计
ma_20 = self.MA(series, 20)                    # 20日移动平均：mean(x[t-19:t])
ema_20 = self.EMA(series, 20)                   # 20日指数移动平均
sma_20 = self.SMA(series, 20)                   # 20日平滑移动平均
wma_20 = self.WMA(series, 20)                   # 20日加权移动平均

# 滚动极值
ts_max_20 = self.TS_MAX(series, 20)            # 20日内最大值位置
ts_min_20 = self.TS_MIN(series, 20)            # 20日内最小值位置
ts_argmax_20 = self.TS_ARGMAX(series, 20)        # 20日内最大值出现位置
ts_argmin_20 = self.TS_ARGMIN(series, 20)        # 20日内最小值出现位置

# 滚动统计（窗口）
sum_20 = self.SUM(series, 20)                # 20日滚动求和
mean_20 = self.MEAN(series, 20)              # 20日滚动平均
std_20 = self.STDDEV(series, 20)              # 20日滚动标准差
```

### Level 2: 横截面函数

```python
# 横截面排名（最常用）
rank = self.RANK(series)                      # 截面排名，归一化到[-0.5, 0.5]

# 截面标准化
zscore = (series - self.MEAN(series)) / self.STDDEV(series)  # Z-score标准化
scale = self.SCALE(series)                    # 归一化到[-1, 1]范围
industry_neutral = self.INDUSTRY_NEUTRALIZE(series)      # 行业中性化处理
```

### Level 3: 高级统计函数

```python
# 相关系数
correlation = self.CORRELATION(series1, series2, 20)  # 20期滚动相关系数
covariance = self.COVARIANCE(series1, series2, 20)      # 20期滚动协方差

# 未来收益
future_returns = self.FUTURE_RETURNS(series, 5)   # 未来5期收益率
past_returns = self.RETURNS(series, 20)           # 过去20期收益率
```

## 技术指标函数

### 趋势类指标

```python
# MACD指标
class MACDFactor(Factor):
    def calculate(self, factors):
        close = factors['close']

        # 快线(12日)、慢线(26日)
        ema_12 = self.EMA(close, 12)
        ema_26 = self.EMA(close, 26)

        # MACD线 = 快线 - 慢线
        macd = ema_12 - ema_26

        # 信号线 = MACD的9日EMA
        signal = self.EMA(macd, 9)

        # MACD柱状图 = (MACD - 信号线) * 2
        histogram = (macd - signal) * 2

        # 最终因子：MACD柱状图
        result = self.RANK(histogram)

        return result

# RSI相对强弱指标
class RSIFactor(Factor):
    def calculate(self, factors):
        close = factors['close']

        # 计算价格变化
        delta = self.DELTA(close, 1)  # 每日价格变化

        # 分离上涨和下跌
        gains = self.IF(delta > 0, delta, 0)  # 上涨时取变化，否则为0
        losses = self.IF(delta < 0, -delta, 0)  # 下跌时取变化的相反数，否则为0

        # 14日平均涨幅和跌幅
        avg_gains = self.MEAN(gains, 14)
        avg_losses = self.MEAN(losses, 14)

        # RSI计算
        rs = 100 * avg_gains / (avg_gains + avg_losses)

        return rs
```

### 动量类指标

```python
# KDJ随机指标
class KDJFactor(Factor):
    def calculate(self, factors):
        high = factors['high']
        low = factors['low']
        close = factors['close']

        # 计算9日内的最高价和最低价
        high_9 = self.TS_MAX(high, 9)
        low_9 = self.TS_MIN(low, 9)

        # RSV = (收盘价 - 9日内最低价) / (9日内最高价 - 9日内最低价) * 100
        rsv = (close - low_9) / (high_9 - low_9) * 100

        # K值 = RSV的3日EMA
        k = self.EMA(rsv, 3)

        # D值 = K值的3日EMA
        d = self.EMA(k, 3)

        # J值 = 3*K - 2*D
        j = 3 * k - 2 * d

        # 最终因子：J值（通常认为J>80为超买，J<20为超卖）
        result = j / 100  # 归一化处理

        return result

# Bollinger布林带指标
class BollingerFactor(Factor):
    def calculate(self, factors):
        close = factors['close']

        # 中轨：20日移动平均
        middle = self.MA(close, 20)

        # 标准差：20日
        std = self.STDDEV(close, 20)

        # 上下轨：中轨 ± 2倍标准差
        upper = middle + 2 * std
        lower = middle - 2 * std

        # 布林带位置：(收盘价 - 下轨) / (上轨 - 下轨)
        bb_position = (close - lower) / (upper - lower)

        # 最终因子：布林带位置
        return bb_position
```

### 波动率类指标

```python
# ATR平均真实波幅
class ATRFactor(Factor):
    def calculate(self, factors):
        high = factors['high']
        low = factors['low']
        close = factors['close']

        # 真实波幅：max(高-昨收, 昨收-低)
        tr1 = self.MAX(high - self.DELAY(close, 1),
                      self.DELAY(close, 1) - self.DELAY(low, 1))

        # 14日真实波幅的移动平均
        atr = self.MEAN(tr1, 14)

        # 归一化处理
        result = 1 / atr  # 波幅越大，因子值越小

        return result

# 历史波动率
class VolatilityFactor(Factor):
    def calculate(self, factors):
        close = factors['close']

        # 计算日收益率
        returns = self.RETURNS(close, 1)

        # 20日收益率标准差作为波动率
        volatility = self.STDDEV(returns, 20)

        return volatility
```

### 成交量类指标

```python
# OBV能量潮指标
class OBVFactor(Factor):
    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']

        # 价格变化方向
        price_change = self.DELTA(close, 1)

        # 成交量：价格上涨时为正，下跌时为负
        obv = self.IF(price_change > 0, volume,
                     self.IF(price_change < 0, -volume, 0))

        # 累积OBV
        cumulative_obv = self.SUM(obv, len(close))

        # 标准化处理
        result = self.RANK(cumulative_obv)

        return result

# 量价趋势指标
class VolumePriceTrendFactor(Factor):
    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']

        # 成交量加权平均价格(VWAP)
        vwap = self.VWAP(close, volume)

        # 量价背离：价格新高但成交量萎缩，或价格新低但成交量放大
        price_high_20 = self.TS_MAX(close, 20)
        volume_ma_20 = self.MEAN(volume, 20)

        # 价格动量信号
        price_signal = (close / self.DELAY(close, 20)) - 1

        # 成交量变化信号
        volume_signal = volume / volume_ma_20 - 1

        # 背离信号：价格上涨但成交量萎缩
        divergence = self.IF((price_signal > 0) & (volume_signal < 0), 1, 0)

        return divergence
```

### 市值率类指标

```python
# 市盈率
class PERatioFactor(Factor):
    def calculate(self, factors):
        close = factors['close']
        market_cap = factors.get('market_cap')  # 市值数据
        earnings = factors.get('earnings')      # 净利润数据

        # 市盈率 = 股价 / 每股收益
        pe_ratio = market_cap / earnings

        # 处理异常值：市盈率过高或负值
        pe_ratio = self.IF(pe_ratio > 100, 100,
                         self.IF(pe_ratio <= 0, 50, pe_ratio))

        # 市盈率倒数（值越大越便宜）
        result = 1 / pe_ratio

        return result

# 市净率
class PBRatioFactor(Factor):
    def calculate(self, factors):
        market_cap = factors.get('market_cap')
        book_value = factors.get('book_value')

        # 市净率 = 市值 / 净资产价值
        pb_ratio = market_cap / book_value

        # 合理范围处理
        result = self.IF(pb_ratio > 10, 10,
                         self.IF(pb_ratio < 0.5, 0.5, pb_ratio))

        return result

# 股息率
class DividendYieldFactor(Factor):
    def calculate(self, factors):
        close = factors['close']
        dividend = factors.get('dividend')

        # 股息率 = 年度股息 / 股价
        dividend_yield = dividend / close

        return dividend_yield
```

## 常用因子模板

### 动量类因子模板

```python
# 多周期动量因子
class MultiMomentumFactor(Factor):
    """
    多时间框架动量因子
    短期(5日)捕捉短期趋势
    中期(20日)捕捉中期趋势
    长期(60日)捕捉长期趋势
    """

    def calculate(self, factors):
        close = factors['close']

        # 短期动量 (5日)
        mom_5 = (close / self.DELAY(close, 5)) - 1
        rank_5 = self.RANK(mom_5)

        # 中期动量 (20日)
        mom_20 = (close / self.DELAY(close, 20)) - 1
        rank_20 = self.RANK(mom_20)

        # 长期动量 (60日)
        mom_60 = (close / self.DELAY(close, 60)) - 1
        rank_60 = self.RANK(mom_60)

        # 复合：根据市场状态分配权重
        # 牛市：更多权重给长期动量
        # 熊市：更多权重给短期动量
        composite = 0.5 * rank_5 + 0.3 * rank_20 + 0.2 * rank_60

        return composite

# 加权动量因子
class WeightedMomentumFactor(Factor):
    """
    加权动量因子：根据波动率调整动量权重

    逻辑：动量 / 波动率
    优势：在波动率较低的时期，相同动量获得更高权重
    应用：风险调整后的动量策略
    """

    def calculate(self, factors):
        close = factors['close']

        # 基础动量
        momentum = (close / self.DELAY(close, 20)) - 1

        # 20日波动率
        volatility = self.STDDEV(self.DELTA(close, 1), 20)

        # 风波率倒数作为权重
        weight = 1 / (volatility + 0.01)  # 避免除零

        # 加权动量
        weighted_momentum = momentum * weight

        # 横截面标准化
        result = self.RANK(weighted_momentum)

        return result

# 偏差动量因子
class LaggedMomentumFactor(Factor):
    """
    偏差动量因子：使用滞后动量避免未来函数

    使用t-1日动量预测t日收益率
    避免未来函数偏误
    """

    def calculate(self, factors):
        close = factors['close']

        # t-1日的20日动量
        lag_momentum = self.DELAY(close, 20) / self.DELAY(close, 41) - 1

        # 当日收益率
        returns = self.RETURNS(close, 1)

        # 偏差动量标准化
        result = self.RANK(lag_momentum)

        return result
```

### 质量类因子模板

```python
# 质量因子
class QualityFactor(Factor):
    """
    质量因子：衡量数据质量，识别异常数据点

    逻辑：通过多种指标评估数据质量
    应用：数据清洗和异常值检测
    """

    def calculate(self, factors):
        close = factors['close']
        high = factors['high']
        low = factors['low']
        volume = factors['volume']

        # 价格跳跃：当日最高价与昨日收盘价的比值
        price_gap = high / self.DELAY(close, 1) - 1

        # 波动率异常：当日波动率与历史平均的比值
        daily_vol = self.ABS(self.DELTA(close, 1) / self.DELAY(close, 1))
        avg_vol = self.MEAN(daily_vol, 60)
        vol_anomaly = daily_vol / (avg_vol + 0.01)

        # 成交量异常：当日成交量与历史平均的比值
        volume_ratio = volume / self.MEAN(volume, 60)
        volume_anomaly = self.IF(volume_ratio > 3, 1,
                               self.IF(volume_ratio < 0.3, -1, 0))

        # 综合质量评分
        quality_score = (self.RANK(price_gap) +
                        self.RANK(vol_anomaly) +
                        volume_anomaly) / 3

        return quality_score

# 数据完整性因子
class DataCompletenessFactor(Factor):
    """
    数据完整性因子：识别数据缺失和异常

    逻辑：检查价格序列的完整性和一致性
    应用：数据质量控制
    """

    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']

        # 缺失值标识
        close_missing = close.isna().astype(int)
        volume_missing = volume.isna().astype(int)

        # 价格一致性：最高价 >= 最低价 >= 收盘价
        price_consistent = (
            (high >= low) &
            (low <= close) &
            (close <= high)
        ).astype(int)

        # 成交量合理性：成交量为正数
        volume_positive = (volume > 0).astype(int)

        # 完整性得分（越高越好）
        completeness_score = (
            close_missing * -1 +
            volume_missing * -1 +
            price_consistent * 1 +
            volume_positive * 1 + 2  # 基础分2分
        ) / 4

        return completeness_score
```

### 价值类因子模板

```python
# 相对价值因子
class RelativeValueFactor(Factor):
    """
    相对价值因子：结合多个估值指标的综合价值评估

    逻辑：市盈率、市净率、股息率的综合评分
    应用：价值投资策略
    """

    def calculate(self, factors):
        pe_ratio = factors.get('pe_ratio')
        pb_ratio = factors.get('pb_ratio')
        dividend_yield = factors.get('dividend_yield')

        # 各指标标准化排序（排名越小越好）
        pe_rank = self.RANK(self.IF(pe_ratio > 0, pe_ratio, 100))  # PE越小排名越高
        pb_rank = self.RANK(self.IF(pb_ratio > 0, pb_ratio, 100))  # PB越小排名越高
        div_rank = self.RANK(dividend_yield)                    # 股息率越高排名越高

        # 综合价值得分
        value_score = (pe_rank + pb_rank + div_rank) / 3

        return value_score

# 成长价值因子
class GrowthValueFactor(Factor):
    """
    成长价值因子：基于历史收益数据的成长性评估

    逻辑：通过多期收益变化评估成长性
    应用：成长股投资策略
    """

    def calculate(self, factors):
        close = factors['close']

        # 计算不同期间的收益率
        return_1m = self.RETURNS(close, 21)    # 1个月收益率
        return_3m = self.RETURNS(close, 63)    # 3个月收益率
        return_6m = self.RETURNS(close, 126)   # 6个月收益率
        return_12m = self.RETURNS(close, 252)   # 12个月收益率

        # 成长性得分：各期收益率的排名平均
        growth_score = (
            self.RANK(return_1m) +
            self.RANK(return_3m) +
            self.RANK(return_6m) +
            self.RANK(return_12m)
        ) / 4

        return growth_score
```

### 技术面因子模板

```python
# 技术面因子
class TechnicalFactor(Factor):
    """
    技术面因子：结合多个技术指标的综合技术分析

    逻辑：RSI、MACD、KDJ等技术指标的综合评分
    应用：技术分析策略
    """

    def calculate(self, factors):
        close = factors['close']
        high = factors['high']
        low = factors['low']

        # RSI指标
        rsi = self.RSI(close)

        # MACD指标
        macd_line = self.MACD(close)

        # KDJ指标
        kdj_k, kdj_d = self.KDJ(close, high, low)

        # 布林带位置
        bb_position = self.BOLLINGER_POSITION(close)

        # 技术面综合得分
        technical_score = (
            self.RANK(rsi) / 4 +           # RSI权重25%
            self.RANK(macd_line) / 4 +      # MACD权重25%
            self.RANK(kdj_k) / 4 +           # KDJ-K权重25%
            (kdj_k - kdj_d) / 4            # KDJ-KD权重25%
        )

        return technical_score
```

## 高级因子技巧

### 1. 因子中性化处理

```python
class IndustryNeutralizedFactor(Factor):
    """
    行业中性化因子：消除行业偏好

    方法：因子值减去行业均值
    """

    def calculate(self, factors):
        factor_values = self.calculate_raw_factor(factors)
        industry = factors['industry']

        # 按行业分组计算均值
        industry_means = factor_values.groupby(industry).transform('mean')

        # 行业中性化：原始值 - 行业均值
        neutralized = factor_values - industry_means

        return neutralized

class MarketCapNeutralizedFactor(Factor):
    """
    市值中性化因子：消除市值偏好

    方法：对市值进行分组处理
    """

    def calculate(self, factors):
        factor_values = self.calculate_raw_factor(factors)
        market_cap = factors['market_cap']

        # 按市值分为10组
        market_cap_groups = pd.qcut(market_cap, 10, labels=False, duplicates='drop')

        # 每组内进行标准化
        def standardize_in_group(group):
            return (group - group.mean()) / group.std()

        neutralized = factor_values.groupby(market_cap_groups).transform(standardize_in_group)

        return neutralized
```

### 2. 因子正交化处理

```python
class OrthogonalFactor(Factor):
    """
    正交化因子：通过Gram-Schmidt正交化消除因子间相关性

    应用：多因子模型构建
    """

    def calculate(self, factors):
        # 假设有多个原始因子
        factor1 = self.calculate_raw_factor1(factors)
        factor2 = self.calculate_raw_factor2(factors)

        # 构建因子矩阵
        factor_matrix = pd.concat([factor1, factor2], axis=1)
        factor_matrix = factor_matrix.dropna()

        # Gram-Schmidt正交化
        orthogonal_factor = self.gram_schmidt(factor_matrix)

        return orthogonal_factor

    def gram_schmidt(self, matrix):
        """Gram-Schmidt正交化实现"""
        Q, R = np.linalg.qr(matrix.T)
        orthogonal = Q.T @ Q.T
        return orthogonal
```

### 3. 自适应参数调整

```python
class AdaptiveFactor(Factor):
    """
    自适应因子：根据市场状态动态调整参数

    方法：根据波动率调整窗口长度
    应用：适应不同市场环境
    """

    def calculate(self, factors):
        close = factors['close']

        # 计算短期和长期波动率
        short_vol = self.STDDEV(self.RETURNS(close, 5), 20)
        long_vol = self.STDDEV(self.RETURNS(close, 60), 60)

        # 动态调整窗口长度
        adaptive_window = self.IF(short_vol > long_vol, 10, 30)

        # 使用自适应窗口计算动量
        momentum = (close / self.DELAY(close, adaptive_window)) - 1

        return momentum

class VolatilityScaledFactor(Factor):
    """
    波动率缩放因子：根据市场波动率调整因子值

    方法：在低波动期间放大信号，高波动期间缩小信号
    应用：波动率目标策略
    """

    def calculate(self, factors):
        factor_values = self.calculate_raw_factor(factors)

        # 计算20日波动率
        volatility = self.STDDEV(factors['close'], 20)

        # 计算波动率的分位数
        vol_25 = volatility.groupby('date').quantile(0.25)
        vol_75 = volatility.groupby('date').quantile(0.75)
        vol_median = volatility.groupby('date').median()

        # 动态缩放因子
        scaled_factor = factor_values.groupby('date').apply(
            lambda x: x / vol_median[x.name] *
                    (vol_25[x.name] / vol_75[x.name])
        )

        return scaled_factor
```

### 4. 风险管理增强

```python
class RiskManagedFactor(Factor):
    """
    风险管理因子：结合风险指标调整因子暴露

    方法：根据波动率和相关性调整因子权重
    应用：风险控制策略
    """

    def calculate(self, factors):
        factor_values = self.calculate_raw_factor(factors)

        # 计算个股波动率
        volatility = self.STDDEV(factors['close'], 20)

        # 计算因子与基准的相关性
        market_returns = self.MEAN(factors['close'].groupby('symbol').apply(
            lambda x: self.RETURNS(x, 20)
        ))

        factor_returns = self.RETURNS(factor_values, 20)
        correlations = {}

        for symbol in factor_values.index.get_level_values('symbol').unique():
            if symbol in market_returns and symbol in factor_returns:
                corr = factor_returns[symbol].corr(market_returns[symbol])
                correlations[symbol] = corr

        # 风险调整：降低高风险、低相关性因子的权重
        risk_adjusted = factor_values.copy()
        for symbol in factor_values.index.get_level_values('symbol').unique():
            if symbol in correlations:
                risk_weight = 1 / (1 + abs(correlations[symbol]))
                vol_adjustment = 1 / (1 + volatility[symbol])

                risk_adjusted.loc[symbol] = (
                    factor_values[symbol] * risk_weight * vol_adjustment
                )

        return risk_adjusted
```

## 因子计算实践

### 完整的因子开发流程

```python
# 1. 因子定义
class ComprehensiveMomentumFactor(Factor):
    """
    综合动量因子：结合多种动量计算方法

    计算包括：
    - 传统动量
    - 风险调整动量
    - 行业中性化动量
    - 多时间框架动量
    """

    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']
        industry = factors['industry']

        # 基础动量计算
        momentum_20d = (close / self.DELAY(close, 20)) - 1

        # 风险调整
        volatility = self.STDDEV(self.DELTA(close, 1), 20)
        risk_adj_momentum = momentum_20d / (1 + volatility)

        # 行业中性化
        industry_momentum = momentum_20d.groupby(industry).transform(
            lambda x: x - x.mean()
        )

        # 多时间框架
        momentum_5d = (close / self.DELAY(close, 5)) - 1
        momentum_60d = (close / self.DELAY(close, 60)) - 1

        # 成交量加权
        volume_weight = volume / self.MEAN(volume, 20)
        volume_momentum = momentum_20d * volume_weight

        # 最终合成
        composite_momentum = (
            self.RANK(industry_momentum) * 0.4 +      # 行业中性40%
            self.RANK(risk_adj_momentum) * 0.3 +      # 风险调整30%
            self.RANK(momentum_5d) * 0.2 +           # 短期20%
            self.RANK(momentum_60d) * 0.1           # 长期10%
        )

        return composite_momentum

# 2. 因子验证和测试
def test_factor_completeness(factor_data):
    """测试因子计算的完整性"""

    # 基础统计
    print(f"因子数据量: {len(factor_data):,}")
    print(f"缺失值数量: {factor_data.isna().sum():,}")
    print(f"无穷值数量: {np.isinf(factor_data).sum():,}")

    # 分布统计
    print(f"因子均值: {factor_data.mean():.4f}")
    print(f"因子标准差: {factor_data.std():.4f}")
    print(f"因子偏度: {factor_data.skew():.4f}")
    print(f"因子峰度: {factor_data.kurtosis():.4f}")

    # 相关性检查
    if 'close' in factor_data.columns:
        correlation = factor_data['factor'].corr(factor_data['close'])
        print(f"与收益率的相关性: {correlation:.4f}")

    return factor_data.describe()

# 3. 因子性能评估
def evaluate_factor_performance(factor_data, returns_data):
    """评估因子的预测性能"""

    from scipy.stats import ttest_1samp
    from scipy.stats import pearsonr

    # IC分析
    ic_values = []
    for date in factor_data.index.get_level_values('date').unique():
        if date in returns_data and date in factor_data:
            factor_slice = factor_data.loc[date]
            returns_slice = returns_data.loc[date]

            if len(factor_slice) > 10 and len(returns_slice) > 10:
                ic, _ = pearsonr(factor_slice, returns_slice)
                ic_values.append(ic)

    ic_mean = np.mean(ic_values)
    ic_std = np.std(ic_values)
    ir = ic_mean / ic_std

    print(f"IC均值: {ic_mean:.4f}")
    print(f"IC标准差: {ic_std:.4f}")
    print(f"信息比率: {ir:.4f}")

    return {
        'ic_mean': ic_mean,
        'ic_std': ic_std,
        'ir': ir,
        'ic_values': ic_values
    }
```

## 性能优化建议

### 1. 计算优化技巧

```python
# 向量化计算替代循环
def optimized_momentum(factors):
    """优化的动量计算"""
    close = factors['close']

    # ❌ 低效：循环计算
    # result = pd.Series(index=close.index, dtype=float)
    # for symbol in close.index.get_level_values('symbol').unique():
    #     symbol_data = close[symbol]
    #     for i in range(1, len(symbol_data)):
    #         if i >= 20:
    #             result[symbol.name + '_' + str(i)] = (symbol_data.iloc[i] / symbol_data.iloc[i-20]) - 1

    # ✅ 高效：向量化滚动操作
    returns = close.groupby('symbol').pct_change(20)
    result = returns.dropna()

    return result

# 批量操作优化
def batch_factor_calculation(factors_list, factor_class):
    """批量因子计算优化"""
    results = []

    # 批量处理多个因子
    for factors in factors_list:
        factor_instance = factor_class()
        result = factor_instance.calculate(factors)
        results.append(result)

    # 合并结果
    combined_results = pd.concat(results, axis=1)

    return combined_results
```

### 2. 内存管理技巧

```python
# 内存高效的数据处理
def memory_efficient_processing(data):
    """内存高效的数据处理"""

    # 使用适当的数据类型
    data = data.astype({
        'close': 'float32',
        'volume': 'float32',
        'high': 'float32',
        'low': 'float32'
    })

    # 及时删除不需要的中间变量
    intermediate = data['close'] * data['volume']
    del intermediate  # 立即释放内存

    # 分块处理大数据集
    chunk_size = 10000
    results = []

    for i in range(0, len(data), chunk_size):
        chunk = data.iloc[i:i+chunk_size]
        chunk_result = process_chunk(chunk)  # 处理数据块
        results.append(chunk_result)

        # 合并结果
    return pd.concat(results, ignore_index=True)

# 使用生成器减少内存占用
def generator_factor_calculation(factors, window_sizes):
    """使用生成器的因子计算"""

    def calculate_momentum_generator(close_series, window):
        for i in range(window, len(close_series)):
            yield close_series.iloc[i] / close_series.iloc[i-window] - 1

    # 使用生成器计算多窗口动量
    momentum_5d = list(calculate_momentum_generator(factors['close'], 5))
    momentum_20d = list(calculate_momentum_generator(factors['close'], 20))

    return momentum_5d, momentum_20d
```

### 3. 缓存和预计算

```python
# 因子结果缓存
class FactorCache:
    def __init__(self):
        self.cache = {}

    def get_factor(self, factor_class, factors, date):
        cache_key = f"{factor_class.__name__}_{date}"

        if cache_key not in self.cache:
            # 计算并缓存因子
            factor_instance = factor_class()
            result = factor_instance.calculate(factors)

            # 只缓存当日的结果
            if len(result) > 0:
                self.cache[cache_key] = result

        return self.cache.get(cache_key)

# 预计算常用指标
class PrecomputedIndicators:
    def __init__(self):
        self.indicators = {}

    def precompute_indicators(self, factors):
        """预计算常用技术指标"""
        close = factors['close']

        # 预计算各种窗口的移动平均
        for window in [5, 10, 20, 60]:
            self.indicators[f'ma_{window}'] = close.groupby('symbol').rolling(window).mean()
            self.indicators[f'ema_{window}'] = close.groupby('symbol').ewm(span=window).mean()
            self.indicators[f'std_{window}'] = close.groupby('symbol').rolling(window).std()

# 懒惰计算
def lazy_factor_evaluation(factors, factor_names):
    """因子评估的懒加载机制"""

    def evaluate_single_factor(factor_name):
        if factor_name not in factors:
            # 按需计算
            factor_instance = globals()[f"{factor_name}Factor"]()
            return factor_instance.calculate(factors)
        return pd.Series()

    # 按需计算和评估
    results = {}
    for factor_name in factor_names:
        results[factor_name] = evaluate_single_factor(factor_name)

    return results
```

### 4. 错误处理和日志

```python
import logging
from functools import wraps

# 日志装饰器
def log_factor_errors(func):
    @wraps(func)
    def wrapper(self, factors):
        try:
            result = func(self, factors)

            # 记录因子计算统计
            if hasattr(self, 'logger') and self.logger:
                self.logger.info(f"因子 {func.__name__}计算完成: "
                              f"数据量={len(result):, "
                              f"均值={result.mean():.4f}, "
                              f"标准差={result.std():.4f}")

            return result

        except Exception as e:
            # 记录错误信息
            if hasattr(self, 'logger') and self.logger:
                self.logger.error(f"因子 {func.__name__}计算失败: {str(e)}")

            # 返回空结果
            return pd.Series(index=factors.index.get_level_values('date').unique(),
                             dtype=float)

    return wrapper

# 数据质量检查
def validate_factor_data(factors):
    """验证输入数据质量"""

    # 检查必需字段
    required_fields = ['close', 'open', 'high', 'low']
    missing_fields = [field for field in required_fields if field not in factors]

    if missing_fields:
        raise ValueError(f"缺少必需字段: {missing_fields}")

    # 检查数据类型
    if not isinstance(factors['close'], pd.Series):
        raise ValueError("close字段必须是pandas Series")

    # 检查索引结构
    if not isinstance(factors['close'].index, pd.MultiIndex):
        raise ValueError("数据索引必须是MultiIndex(['symbol', 'date'])")

    # 检查数据范围
    print(f"数据范围: {factors.index.get_level_values('date').min()}")
    print(f"股票数量: {len(factors.index.get_level_values('symbol').unique())}")

    return True
```

这份指南提供了PandaFactor支持的完整因子计算体系，从基础概念到高级技巧，帮助您快速构建和优化量化因子。