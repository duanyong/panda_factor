# PandaFactor 实用代码示例

## 📖 目录

1. [基础数据操作](#基础数据操作)
2. [经典因子实现](#经典因子实现)
3. [技术指标因子](#技术指标因子)
4. [基本面因子](#基本面因子)
5. [高频数据因子](#高频数据因子)
6. [多因子组合](#多因子组合)
7. [因子分析工具](#因子分析工具)
8. [实用工具函数](#实用工具函数)

## 📊 基础数据操作

### 初始化和数据获取

```python
import pandas as pd
import numpy as np
import panda_data
from panda_factor.generate.factor_base import Factor

# 初始化系统
panda_data.init()

# 获取市场数据
def get_sample_data():
    """获取样本数据用于测试"""
    symbols = ['000001.SZ', '000002.SZ', '600000.SS', '600036.SS']
    return panda_data.get_market_data(
        start_date='20240101',
        end_date='20240320',
        symbols=symbols,
        fields=['open', 'close', 'high', 'low', 'volume', 'amount', 'market_cap']
    )

# 数据预处理
def preprocess_data(factors):
    """数据预处理通用函数"""
    for key, value in factors.items():
        # 前向填充缺失值
        factors[key] = value.groupby('symbol').fillna(method='ffill')
        # 删除仍为缺失的行
        factors[key] = factors[key].dropna()
    return factors

# 数据验证
def validate_data(factors):
    """验证数据完整性"""
    issues = []

    for key, value in factors.items():
        missing_count = value.isna().sum()
        if missing_count > 0:
            issues.append(f"{key}: {missing_count} 缺失值")

        if value.min() < 0 and key in ['close', 'open', 'high', 'low']:
            issues.append(f"{key}: 存在负价格")

    if issues:
        print("数据验证问题:")
        for issue in issues:
            print(f"  - {issue}")
    else:
        print("数据验证通过")

    return len(issues) == 0
```

## 🎯 经典因子实现

### 1. 动量因子系列

```python
class Momentum20D(Factor):
    """20日动量因子 - 经典动量策略"""
    def calculate(self, factors):
        close = factors['close']
        return self.RANK((close / self.DELAY(close, 20)) - 1)

class Momentum5D(Factor):
    """5日动量因子 - 短期动量"""
    def calculate(self, factors):
        close = factors['close']
        return self.RANK((close / self.DELAY(close, 5)) - 1)

class MomentumComposite(Factor):
    """复合动量因子 - 多周期动量组合"""
    def calculate(self, factors):
        close = factors['close']

        # 不同周期动量
        momentum_5d = self.RANK((close / self.DELAY(close, 5)) - 1)
        momentum_10d = self.RANK((close / self.DELAY(close, 10)) - 1)
        momentum_20d = self.RANK((close / self.DELAY(close, 20)) - 1)

        # 等权组合
        return (momentum_5d + momentum_10d + momentum_20d) / 3

class MomentumAccelerator(Factor):
    """动量加速度因子"""
    def calculate(self, factors):
        close = factors['close']

        # 一阶动量
        momentum_10d = (close / self.DELAY(close, 10)) - 1
        momentum_20d = (close / self.DELAY(close, 20)) - 1

        # 二阶动量（加速度）
        acceleration = momentum_10d - self.DELAY(momentum_10d, 10)

        return self.RANK(acceleration)

class PriceMomentumConsistency(Factor):
    """动量一致性因子 - 多时间框架动量一致性"""
    def calculate(self, factors):
        close = factors['close']

        # 不同周期动量符号
        momentum_5d_sign = self.SIGN((close / self.DELAY(close, 5)) - 1)
        momentum_10d_sign = self.SIGN((close / self.DELAY(close, 10)) - 1)
        momentum_20d_sign = self.SIGN((close / self.DELAY(close, 20)) - 1)

        # 一致性得分
        consistency = momentum_5d_sign + momentum_10d_sign + momentum_20d_sign

        # 标准化
        return self.SCALE(consistency)
```

### 2. 反转因子系列

```python
class Reversal1D(Factor):
    """1日反转因子 - 短期反转"""
    def calculate(self, factors):
        close = factors['close']
        return -self.RANK((close / self.DELAY(close, 1)) - 1)

class Reversal5D(Factor):
    """5日反转因子"""
    def calculate(self, factors):
        close = factors['close']
        return -self.RANK((close / self.DELAY(close, 5)) - 1)

class Reversal20D(Factor):
    """20日反转因子 - 长期反转"""
    def calculate(self, factors):
        close = factors['close']
        return -self.RANK((close / self.DELAY(close, 20)) - 1)

class OvernightReversal(Factor):
    """隔夜反转因子"""
    def calculate(self, factors):
        close = factors['close']
        open_price = factors['open']

        # 隔夜收益率
        overnight_return = (open_price / self.DELAY(close, 1)) - 1

        # 反转逻辑
        return -self.RANK(overnight_return)

class IntradayReversal(Factor):
    """日内反转因子"""
    def calculate(self, factors):
        close = factors['close']
        open_price = factors['open']
        high = factors['high']
        low = factors['low']

        # 日内收益率
        intraday_return = (close / open_price) - 1
        # 日内振幅
        intraday_range = (high - low) / open_price

        # 反转因子
        return -self.RANK(intraday_return * intraday_range)
```

## 📈 技术指标因子

### 1. 均线相关因子

```python
class MACrossover(Factor):
    """均线交叉因子 - 短期均线相对长期均线的位置"""
    def calculate(self, factors):
        close = factors['close']

        # 计算均线
        ma5 = self.MEAN(close, 5)
        ma10 = self.MEAN(close, 10)
        ma20 = self.MEAN(close, 20)

        # 多组均线相对位置
        ma_diff1 = (ma5 / ma10) - 1
        ma_diff2 = (ma10 / ma20) - 1

        # 综合信号
        return self.RANK(ma_diff1 + ma_diff2)

class MAVolumeWeighted(Factor):
    """成交量加权均线因子"""
    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']

        # 20日成交量加权均价
        volume_sum = self.SUM(volume, 20)
        volume_price_sum = self.SUM(close * volume, 20)
        vwap = volume_price_sum / (volume_sum + 1e-6)

        # 当前价格相对VWAP的位置
        return self.RANK((close / vwap) - 1)

class MADeviation(Factor):
    """均线偏离度因子"""
    def calculate(self, factors):
        close = factors['close']

        # 计算不同周期均线
        ma5 = self.MEAN(close, 5)
        ma10 = self.MEAN(close, 10)
        ma20 = self.MEAN(close, 20)

        # 计算偏离度
        deviation_5 = (close / ma5) - 1
        deviation_10 = (close / ma10) - 1
        deviation_20 = (close / ma20) - 1

        # 标准化偏离度
        dev_std_5 = self.STDDEV(deviation_5, 20)
        dev_std_10 = self.STDDEV(deviation_10, 20)
        dev_std_20 = self.STDDEV(deviation_20, 20)

        # Z-score
        zscore_5 = deviation_5 / (dev_std_5 + 1e-6)
        zscore_10 = deviation_10 / (dev_std_10 + 1e-6)
        zscore_20 = deviation_20 / (dev_std_20 + 1e-6)

        # 综合信号
        return self.RANK(zscore_5 + zscore_10 + zscore_20)
```

### 2. RSI 和随机指标

```python
class RSIIndicator(Factor):
    """RSI相对强弱指标因子"""
    def calculate(self, factors):
        close = factors['close']

        # 计算价格变化
        delta = close - self.DELAY(close, 1)

        # 分离上涨和下跌
        gain = self.IF(delta > 0, delta, 0)
        loss = self.IF(delta < 0, -delta, 0)

        # 计算平均涨幅和跌幅
        avg_gain = self.MEAN(gain, 14)
        avg_loss = self.MEAN(loss, 14)

        # 计算RS和RSI
        rs = avg_gain / (avg_loss + 1e-6)
        rsi = 100 - (100 / (1 + rs))

        # 超买超卖信号
        rsi_signal = self.IF(rsi > 70, -(rsi - 70),  # 超买，做空信号
                   self.IF(rsi < 30, 30 - rsi, 50 - rsi))  # 超卖或中性

        return self.SCALE(rsi_signal)

class StochasticIndicator(Factor):
    """随机指标因子"""
    def calculate(self, factors):
        close = factors['close']
        high = factors['high']
        low = factors['low']

        # 计算20日最高最低价
        highest_20 = self.MAX(high, 20)
        lowest_20 = self.MIN(low, 20)

        # 计算K值
        k_percent = 100 * (close - lowest_20) / (highest_20 - lowest_20 + 1e-6)

        # 计算3日K值均值作为D值
        d_percent = self.MEAN(k_percent, 3)

        # 买卖信号
        signal = k_percent - d_percent

        return self.RANK(signal)

class WilliamsR(Factor):
    """威廉指标因子"""
    def calculate(self, factors):
        close = factors['close']
        high = factors['high']
        low = factors['low']

        # 计算14日最高最低价
        highest_14 = self.MAX(high, 14)
        lowest_14 = self.MIN(low, 14)

        # 计算威廉指标
        williams_r = -100 * (highest_14 - close) / (highest_14 - lowest_14 + 1e-6)

        # 信号反转（威廉指标是反转指标）
        return self.RANK(-williams_r)
```

### 3. 波动率因子

```python
class Volatility20D(Factor):
    """20日波动率因子"""
    def calculate(self, factors):
        close = factors['close']

        # 计算日收益率
        daily_return = (close / self.DELAY(close, 1)) - 1

        # 计算20日标准差
        volatility = self.STDDEV(daily_return, 20)

        # 波动率排序（低波动率通常表现更好）
        return -self.RANK(volatility)

class VolatilityDecay(Factor):
    """波动率衰减因子 - 波动率变化趋势"""
    def calculate(self, factors):
        close = factors['close']

        # 计算不同周期波动率
        vol_5d = self.STDDEV((close / self.DELAY(close, 1)) - 1, 5)
        vol_10d = self.STDDEV((close / self.DELAY(close, 1)) - 1, 10)
        vol_20d = self.STDDEV((close / self.DELAY(close, 1)) - 1, 20)

        # 波动率变化率
        vol_change_5_10 = (vol_5d / vol_10d) - 1
        vol_change_10_20 = (vol_10d / vol_20d) - 1

        # 波动率衰减信号
        return self.RANK(vol_change_5_10 + vol_change_10_20)

class RangeVolatility(Factor):
    """价格区间波动率因子"""
    def calculate(self, factors):
        high = factors['high']
        low = factors['low']
        close = factors['close']

        # 真实波动幅度
        tr = self.MAX(
            high - self.DELAY(close, 1),
            self.MAX(
                low - self.DELAY(close, 1),
                high - low
            )
        )

        # 20日平均真实波动幅度
        atr = self.MEAN(tr, 20)

        # 相对波动率
        relative_vol = atr / close

        # 反转逻辑（低波动率偏好）
        return -self.RANK(relative_vol)

class ParkinsonVolatility(Factor):
    """Parkinson波动率因子 - 基于高低价的波动率估计"""
    def calculate(self, factors):
        high = factors['high']
        low = factors['low']

        # Parkinson波动率估计
        hl_ratio = high / low
        log_hl = self.LOG(hl_ratio)
        parkinson_vol = self.SQRT(0.361 * self.MEAN(log_hl * log_hl, 20))

        # 标准化
        return self.SCALE(parkinson_vol)
```

## 💰 基本面因子

### 1. 估值因子

```python
class PEInverse(Factor):
    """市盈率倒数因子"""
    def calculate(self, factors):
        pe_ratio = factors['pe_ratio']

        # 过滤异常值
        pe_clean = self.WINSORIZE(pe_ratio, 5)

        # 市盈率倒数
        pe_inverse = 1 / (pe_clean + 1e-6)

        # 标准化
        return self.SCALE(pe_inverse)

class PBInverse(Factor):
    """市净率倒数因子"""
    def calculate(self, factors):
        pb_ratio = factors['pb_ratio']

        # 过滤异常值
        pb_clean = self.WINSORIZE(pb_ratio, 5)

        # 市净率倒数
        pb_inverse = 1 / (pb_clean + 1e-6)

        # 标准化
        return self.SCALE(pb_inverse)

class ValueComposite(Factor):
    """复合价值因子"""
    def calculate(self, factors):
        pe_ratio = factors['pe_ratio']
        pb_ratio = factors['pb_ratio']
        ps_ratio = factors.get('ps_ratio', None)  # 如果有市销率数据

        # 数据清洗
        pe_clean = self.WINSORIZE(pe_ratio, 5)
        pb_clean = self.WINSORIZE(pb_ratio, 5)

        # 转换为倒数
        pe_signal = 1 / (pe_clean + 1e-6)
        pb_signal = 1 / (pb_clean + 1e-6)

        # 标准化
        pe_norm = self.SCALE(pe_signal)
        pb_norm = self.SCALE(pb_signal)

        # 等权组合
        if ps_ratio is not None:
            ps_clean = self.WINSORIZE(ps_ratio, 5)
            ps_signal = 1 / (ps_clean + 1e-6)
            ps_norm = self.SCALE(ps_signal)
            return (pe_norm + pb_norm + ps_norm) / 3
        else:
            return (pe_norm + pb_norm) / 2

class DividendYield(Factor):
    """股息率因子"""
    def calculate(self, factors):
        dividend_yield = factors.get('dividend_yield', None)

        if dividend_yield is None:
            return pd.Series(0, index=factors['close'].index)

        # 数据清洗
        dividend_clean = self.WINSORIZE(dividend_yield, 5)

        # 股息率越高越好
        return self.RANK(dividend_clean)

class EarningsYield(Factor):
    """盈利收益率因子"""
    def calculate(self, factors):
        pe_ratio = factors['pe_ratio']

        # 盈利收益率 = E/P = 1/PE
        earnings_yield = 1 / (pe_ratio + 1e-6)

        # 过滤异常值
        earnings_clean = self.WINSORIZE(earnings_yield, 5)

        return self.RANK(earnings_clean)
```

### 2. 成长因子

```python
class RevenueGrowth(Factor):
    """营收增长率因子"""
    def calculate(self, factors):
        revenue = factors.get('revenue', None)

        if revenue is None:
            return pd.Series(0, index=factors['close'].index)

        # 同比增长率
        revenue_yoy = (revenue / self.DELAY(revenue, 252)) - 1

        # 数据清洗
        growth_clean = self.WINSORIZE(revenue_yoy, 5)

        return self.RANK(growth_clean)

class ProfitGrowth(Factor):
    """利润增长率因子"""
    def calculate(self, factors):
        net_profit = factors.get('net_profit', None)

        if net_profit is None:
            return pd.Series(0, index=factors['close'].index)

        # 同比增长率
        profit_yoy = (net_profit / self.DELAY(net_profit, 252)) - 1

        # 数据清洗
        growth_clean = self.WINSORIZE(profit_yoy, 5)

        return self.RANK(growth_clean)

class ROEChange(Factor):
    """ROE变化因子"""
    def calculate(self, factors):
        roe = factors.get('roe', None)

        if roe is None:
            return pd.Series(0, index=factors['close'].index)

        # ROE变化
        roe_change = roe - self.DELAY(roe, 252)

        # 数据清洗
        change_clean = self.WINSORIZE(roe_change, 5)

        return self.RANK(change_clean)

class GrowthConsistency(Factor):
    """成长一致性因子"""
    def calculate(self, factors):
        revenue = factors.get('revenue', None)

        if revenue is None:
            return pd.Series(0, index=factors['close'].index)

        # 计算多季度增长率
        growth_q1 = (revenue / self.DELAY(revenue, 63)) - 1
        growth_q2 = (revenue / self.DELAY(revenue, 126)) - 1
        growth_q3 = (revenue / self.DELAY(revenue, 189)) - 1
        growth_q4 = (revenue / self.DELAY(revenue, 252)) - 1

        # 增长率一致性（方差越小一致性越好）
        growth_mean = (growth_q1 + growth_q2 + growth_q3 + growth_q4) / 4
        growth_var = ((growth_q1 - growth_mean) ** 2 +
                     (growth_q2 - growth_mean) ** 2 +
                     (growth_q3 - growth_mean) ** 2 +
                     (growth_q4 - growth_mean) ** 2) / 4

        # 综合得分：高增长 + 低方差
        consistency_score = growth_mean - self.SQRT(growth_var)

        return self.RANK(consistency_score)
```

### 3. 质量因子

```python
class ROE(Factor):
    """净资产收益率因子"""
    def calculate(self, factors):
        roe = factors.get('roe', None)

        if roe is None:
            return pd.Series(0, index=factors['close'].index)

        # 数据清洗
        roe_clean = self.WINSORIZE(roe, 5)

        return self.RANK(roe_clean)

class ROA(Factor):
    """总资产收益率因子"""
    def calculate(self, factors):
        roa = factors.get('roa', None)

        if roa is None:
            return pd.Series(0, index=factors['close'].index)

        # 数据清洗
        roa_clean = self.WINSORIZE(roa, 5)

        return self.RANK(roa_clean)

class DebtToEquity(Factor):
    """资产负债率因子（反转逻辑）"""
    def calculate(self, factors):
        debt_ratio = factors.get('debt_to_equity', None)

        if debt_ratio is None:
            return pd.Series(0, index=factors['close'].index)

        # 负债率越低越好
        return -self.RANK(debt_ratio)

class CurrentRatio(Factor):
    """流动比率因子"""
    def calculate(self, factors):
        current_ratio = factors.get('current_ratio', None)

        if current_ratio is None:
            return pd.Series(0, index=factors['close'].index)

        # 流动比率适中为好，过高或过低都不理想
        # 使用距离理想值的倒数作为信号
        ideal_ratio = 2.0  # 理想流动比率
        distance = self.ABS(current_ratio - ideal_ratio)

        return -self.RANK(distance)

class QualityComposite(Factor):
    """复合质量因子"""
    def calculate(self, factors):
        roe = factors.get('roe', None)
        roa = factors.get('roa', None)
        debt_ratio = factors.get('debt_to_equity', None)
        current_ratio = factors.get('current_ratio', None)

        signals = []

        # ROE信号
        if roe is not None:
            roe_clean = self.WINSORIZE(roe, 5)
            signals.append(self.SCALE(roe_clean))

        # ROA信号
        if roa is not None:
            roa_clean = self.WINSORIZE(roa, 5)
            signals.append(self.SCALE(roa_clean))

        # 负债率信号（反转）
        if debt_ratio is not None:
            debt_clean = self.WINSORIZE(debt_ratio, 5)
            signals.append(self.SCALE(-debt_clean))

        # 流动比率信号
        if current_ratio is not None:
            current_clean = self.WINSORIZE(current_ratio, 5)
            # 优化流动比率信号
            ideal_ratio = 2.0
            distance = self.ABS(current_clean - ideal_ratio)
            signals.append(self.SCALE(-distance))

        if not signals:
            return pd.Series(0, index=factors['close'].index)

        # 等权组合
        return sum(signals) / len(signals)
```

## ⚡ 高频数据因子

### 1. 分钟级动量因子

```python
class IntradayMomentum1H(Factor):
    """1小时动量因子"""
    def calculate(self, factors):
        close = factors['close']

        # 60分钟动量
        momentum_60m = (close / self.DELAY(close, 60)) - 1

        return self.RANK(momentum_60m)

class IntradayReversal5M(Factor):
    """5分钟反转因子"""
    def calculate(self, factors):
        close = factors['close']

        # 5分钟反转
        reversal_5m = -((close / self.DELAY(close, 5)) - 1)

        return self.RANK(reversal_5m)

class VolumeSpike(Factor):
    """成交量异动因子"""
    def calculate(self, factors):
        volume = factors['volume']

        # 相对成交量（当前相对于过去1小时均值）
        volume_ma_60 = self.MEAN(volume, 60)
        volume_ratio = volume / (volume_ma_60 + 1e-6)

        # 成交量突增
        return self.RANK(volume_ratio)

class PriceVolumeTrend(Factor):
    """价量趋势因子"""
    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']

        # 价格变化
        price_change = (close / self.DELAY(close, 1)) - 1

        # 成交量变化
        volume_ma = self.MEAN(volume, 20)
        volume_ratio = volume / (volume_ma + 1e-6)

        # 价量配合度
        price_volume_corr = self.CORRELATION(price_change, volume_ratio, 20)

        return self.RANK(price_volume_corr)

class IntradayVolatility(Factor):
    """日内波动率因子"""
    def calculate(self, factors):
        high = factors['high']
        low = factors['low']
        close = factors['close']

        # 真实波动幅度
        tr = high - low
        atr = self.MEAN(tr, 30)

        # 相对波动率
        relative_vol = atr / close

        # 反转逻辑（低波动率偏好）
        return -self.RANK(relative_vol)
```

## 🔗 多因子组合

### 1. 简单多因子模型

```python
class SimpleMultiFactor(Factor):
    """简单多因子组合 - 动量+价值+质量"""
    def calculate(self, factors):
        close = factors['close']
        pe_ratio = factors.get('pe_ratio', None)
        roe = factors.get('roe', None)

        # 动量因子
        momentum = self.RANK((close / self.DELAY(close, 20)) - 1)

        # 价值因子
        if pe_ratio is not None:
            value_signal = self.SCALE(1 / (pe_ratio + 1e-6))
        else:
            value_signal = pd.Series(0, index=close.index)

        # 质量因子
        if roe is not None:
            quality_signal = self.SCALE(roe)
        else:
            quality_signal = pd.Series(0, index=close.index)

        # 等权组合
        composite = (momentum + value_signal + quality_signal) / 3

        return self.RANK(composite)

class WeightedMultiFactor(Factor):
    """加权多因子组合"""
    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']
        pe_ratio = factors.get('pe_ratio', None)
        roe = factors.get('roe', None)

        # 因子权重配置
        weights = {
            'momentum': 0.3,
            'value': 0.2,
            'quality': 0.2,
            'volume': 0.3
        }

        # 各因子计算
        momentum = self.RANK((close / self.DELAY(close, 20)) - 1)
        volume_ma = self.MEAN(volume, 20)
        volume_signal = self.RANK(volume / (volume_ma + 1e-6))

        if pe_ratio is not None:
            value_signal = self.SCALE(1 / (pe_ratio + 1e-6))
        else:
            value_signal = pd.Series(0, index=close.index)

        if roe is not None:
            quality_signal = self.SCALE(roe)
        else:
            quality_signal = pd.Series(0, index=close.index)

        # 加权组合
        composite = (weights['momentum'] * momentum +
                    weights['value'] * value_signal +
                    weights['quality'] * quality_signal +
                    weights['volume'] * volume_signal)

        return self.RANK(composite)

class DynamicWeightMultiFactor(Factor):
    """动态权重多因子组合 - 根据市场环境调整权重"""
    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']

        # 市场状态识别
        market_return = (self.MEAN(close, 500) / self.DELAY(self.MEAN(close, 500), 20)) - 1
        market_vol = self.STDDEV((close / self.DELAY(close, 1)) - 1, 20)

        # 牛市增强动量，熊市增强价值
        if market_return > 0.02:  # 强势市场
            momentum_weight = 0.5
            value_weight = 0.1
        elif market_return < -0.02:  # 弱势市场
            momentum_weight = 0.1
            value_weight = 0.5
        else:  # 震荡市场
            momentum_weight = 0.3
            value_weight = 0.3

        # 各因子计算
        momentum = self.RANK((close / self.DELAY(close, 20)) - 1)

        # 动态组合
        composite = (momentum_weight * momentum +
                    value_weight * pd.Series(0, index=close.index))  # 简化版本

        return self.RANK(composite)
```

## 📊 因子分析工具

### 1. IC分析工具

```python
class FactorAnalyzer:
    """因子分析工具类"""

    def __init__(self):
        self.cache = {}

    def calculate_ic(self, factor_values, returns, period=1):
        """计算IC值"""
        # 对齐数据
        aligned_factor, aligned_returns = self._align_data(factor_values, returns, period)

        # 按日期分组计算IC
        ic_values = aligned_factor.groupby('date').apply(
            lambda x: x.corr(aligned_returns.loc[x.index])
        )

        return ic_values.dropna()

    def calculate_ic_ir(self, factor_values, returns, period=1):
        """计算IC_IR值"""
        ic_values = self.calculate_ic(factor_values, returns, period)

        ic_mean = ic_values.mean()
        ic_std = ic_values.std()
        ic_ir = ic_mean / (ic_std + 1e-6)

        return {
            'ic_mean': ic_mean,
            'ic_std': ic_std,
            'ic_ir': ic_ir,
            'ic_positive_ratio': (ic_values > 0).mean()
        }

    def layer_analysis(self, factor_values, returns, layers=5, period=1):
        """分层分析"""
        # 对齐数据
        aligned_factor, aligned_returns = self._align_data(factor_values, returns, period)

        # 按日期分组
        layer_returns = []

        for date in aligned_factor.index.get_level_values('date').unique():
            date_factor = aligned_factor.loc[date]
            date_returns = aligned_returns.loc[date]

            # 分层
            quantiles = date_factor.quantile([i/layers for i in range(layers+1)])

            layer_daily_returns = []
            for i in range(layers):
                if i == layers - 1:
                    mask = (date_factor >= quantiles.iloc[i]) & (date_factor <= quantiles.iloc[i+1])
                else:
                    mask = (date_factor >= quantiles.iloc[i]) & (date_factor < quantiles.iloc[i+1])

                layer_return = date_returns[mask].mean()
                layer_daily_returns.append(layer_return)

            layer_returns.append(layer_daily_returns)

        # 转换为DataFrame
        layer_df = pd.DataFrame(layer_returns,
                               columns=[f'Layer_{i+1}' for i in range(layers)],
                               index=aligned_factor.index.get_level_values('date').unique())

        return layer_df

    def _align_data(self, factor_values, returns, period):
        """对齐因子值和收益率数据"""
        # 确保收益率数据是未来period期的收益率
        future_returns = self.FUTURE_RETURNS(returns, period)

        # 对齐索引
        common_index = factor_values.index.intersection(future_returns.index)

        aligned_factor = factor_values.loc[common_index]
        aligned_returns = future_returns.loc[common_index]

        return aligned_factor, aligned_returns

    def factor_analysis_report(self, factor_name, factor_values, returns):
        """生成因子分析报告"""
        report = {}

        # 不同持有期的IC分析
        for period in [1, 5, 10, 20]:
            ic_stats = self.calculate_ic_ir(factor_values, returns, period)
            report[f'IC_{period}D'] = ic_stats

        # 分层分析
        layer_returns = self.layer_analysis(factor_values, returns, layers=5)

        # 计算各层年化收益
        annual_returns = {}
        for layer in layer_returns.columns:
            annual_return = (1 + layer_returns[layer].mean()) ** 252 - 1
            annual_returns[layer] = annual_return

        report['layer_annual_returns'] = annual_returns
        report['layer_returns'] = layer_returns

        return report

# 使用示例
def analyze_factor_example():
    """因子分析示例"""
    import panda_data

    # 初始化数据
    panda_data.init()

    # 创建分析器
    analyzer = FactorAnalyzer()

    # 获取样本数据
    market_data = panda_data.get_market_data(
        start_date='20240101',
        end_date='20240320',
        symbols=['000001.SZ', '600000.SS'],
        fields=['close']
    )

    # 计算收益率
    returns = market_data['close'].groupby('symbol').pct_change(1)

    # 示例因子
    close = market_data['close']
    factor_values = close.groupby('symbol').apply(lambda x: (x / x.shift(20)) - 1)
    factor_values = factor_values.groupby('date').rank(pct=True) - 0.5

    # 生成分析报告
    report = analyzer.factor_analysis_report("Momentum_20D", factor_values, returns)

    # 打印结果
    print("=== 因子分析报告 ===")
    for period, stats in report.items():
        if period.startswith('IC_'):
            print(f"\n{period}:")
            print(f"  IC均值: {stats['ic_mean']:.4f}")
            print(f"  IC标准差: {stats['ic_std']:.4f}")
            print(f"  IC_IR: {stats['ic_ir']:.4f}")
            print(f"  IC胜率: {stats['ic_positive_ratio']:.2%}")

    print("\n分层年化收益:")
    for layer, annual_ret in report['layer_annual_returns'].items():
        print(f"  {layer}: {annual_ret:.2%}")

    return report
```

## 🛠️ 实用工具函数

### 1. 数据处理工具

```python
class DataUtils:
    """数据处理工具类"""

    @staticmethod
    def remove_outliers(series, method='winsorize', n=3):
        """异常值处理"""
        if method == 'winsorize':
            q1 = series.quantile(0.25)
            q3 = series.quantile(0.75)
            iqr = q3 - q1
            lower_bound = q1 - n * iqr
            upper_bound = q3 + n * iqr
            return series.clip(lower_bound, upper_bound)
        elif method == 'zscore':
            z_scores = (series - series.mean()) / series.std()
            return series[z_scores.abs() <= n]
        else:
            return series

    @staticmethod
    def neutralize(factor, industry_factors=None, style_factors=None):
        """因子中性化处理"""
        if industry_factors is None and style_factors is None:
            return factor

        # 构建回归变量
        X = []
        if industry_factors is not None:
            X.extend(industry_factors)
        if style_factors is not None:
            X.extend(style_factors)

        if not X:
            return factor

        # 行业中性化处理
        X_matrix = pd.concat(X, axis=1)

        # 逐日期回归
        neutralized_factor = factor.copy()

        for date in factor.index.get_level_values('date').unique():
            date_mask = factor.index.get_level_values('date') == date
            date_factor = factor.loc[date_mask]
            date_X = X_matrix.loc[date_mask]

            # 线性回归
            try:
                from sklearn.linear_model import LinearRegression
                model = LinearRegression()
                model.fit(date_X, date_factor)

                # 残差作为中性化后的因子
                residuals = date_factor - model.predict(date_X)
                neutralized_factor.loc[date_mask] = residuals
            except:
                # 回归失败则保持原值
                continue

        return neutralized_factor

    @staticmethod
    def standardize_factor(factor, method='zscore'):
        """因子标准化"""
        if method == 'zscore':
            return (factor - factor.mean()) / factor.std()
        elif method == 'minmax':
            return (factor - factor.min()) / (factor.max() - factor.min())
        elif method == 'rank':
            return factor.groupby('date').rank(pct=True) - 0.5
        else:
            return factor

    @staticmethod
    def calculate_turnover(factor, threshold=0.5):
        """计算因子换手率"""
        # 按日期分组计算换手率
        daily_turnover = []

        dates = sorted(factor.index.get_level_values('date').unique())

        for i in range(1, len(dates)):
            prev_date = dates[i-1]
            curr_date = dates[i]

            prev_factor = factor.loc[prev_date]
            curr_factor = factor.loc[curr_date]

            # 获取交集股票
            common_stocks = prev_factor.index.intersection(curr_factor.index)

            if len(common_stocks) == 0:
                continue

            prev_values = prev_factor.loc[common_stocks]
            curr_values = curr_factor.loc[common_stocks]

            # 计算换手率（简化版）
            prev_top = prev_values.nlargest(int(len(prev_values) * threshold))
            curr_top = curr_values.nlargest(int(len(curr_values) * threshold))

            # 计算新增和移除的股票数量
            new_stocks = set(curr_top.index) - set(prev_top.index)
            removed_stocks = set(prev_top.index) - set(curr_top.index)

            turnover = (len(new_stocks) + len(removed_stocks)) / len(prev_top)
            daily_turnover.append(turnover)

        return pd.Series(daily_turnover, index=dates[1:])

    @staticmethod
    def factor_correlation_matrix(factors_dict):
        """计算因子相关性矩阵"""
        # 确保所有因子有相同的索引
        common_index = None
        for factor in factors_dict.values():
            if common_index is None:
                common_index = factor.index
            else:
                common_index = common_index.intersection(factor.index)

        # 构建因子DataFrame
        factor_df = pd.DataFrame({
            name: factor.loc[common_index]
            for name, factor in factors_dict.items()
        })

        # 计算相关性
        correlation_matrix = factor_df.corr()

        return correlation_matrix
```

### 2. 因子评估工具

```python
class FactorEvaluator:
    """因子评估工具"""

    def __init__(self):
        self.analyzer = FactorAnalyzer()
        self.data_utils = DataUtils()

    def comprehensive_evaluation(self, factor_name, factor_values, returns,
                                factor_description="", benchmark=None):
        """综合因子评估"""
        evaluation = {
            'factor_name': factor_name,
            'description': factor_description,
            'evaluation_date': pd.Timestamp.now(),
            'performance_metrics': {},
            'risk_metrics': {},
            'stability_metrics': {}
        }

        # 1. 性能指标
        for period in [1, 5, 10, 20]:
            ic_stats = self.analyzer.calculate_ic_ir(factor_values, returns, period)
            evaluation['performance_metrics'][f'IC_{period}D'] = ic_stats

        # 2. 分层分析
        layer_returns = self.analyzer.layer_analysis(factor_values, returns, layers=5)
        evaluation['layer_analysis'] = layer_returns

        # 3. 风险指标
        evaluation['risk_metrics'] = self._calculate_risk_metrics(factor_values, returns)

        # 4. 稳定性指标
        evaluation['stability_metrics'] = self._calculate_stability_metrics(factor_values)

        # 5. 总体评分
        evaluation['overall_score'] = self._calculate_overall_score(evaluation)

        return evaluation

    def _calculate_risk_metrics(self, factor_values, returns):
        """计算风险指标"""
        # 最大回撤
        layer_analysis = self.analyzer.layer_analysis(factor_values, returns, layers=5)
        top_layer_returns = layer_analysis['Layer_1']
        cumulative_returns = (1 + top_layer_returns).cumprod()
        rolling_max = cumulative_returns.expanding().max()
        drawdown = (cumulative_returns / rolling_max) - 1
        max_drawdown = drawdown.min()

        # 波动率
        volatility = top_layer_returns.std()

        # 夏普比率（简化版）
        sharpe_ratio = top_layer_returns.mean() / (volatility + 1e-6)

        return {
            'max_drawdown': max_drawdown,
            'volatility': volatility,
            'sharpe_ratio': sharpe_ratio
        }

    def _calculate_stability_metrics(self, factor_values):
        """计算稳定性指标"""
        # 因子换手率
        turnover = self.data_utils.calculate_turnover(factor_values)

        # 因子衰减度
        factor_autocorr = []
        for lag in [1, 5, 10, 20]:
            autocorr = factor_values.groupby('symbol').apply(
                lambda x: x.autocorr(lag=lag)
            ).mean()
            factor_autocorr.append(autocorr)

        return {
            'avg_turnover': turnover.mean(),
            'turnover_std': turnover.std(),
            'factor_decay': {
                f'lag_{lag}': corr
                for lag, corr in zip([1, 5, 10, 20], factor_autocorr)
            }
        }

    def _calculate_overall_score(self, evaluation):
        """计算总体评分"""
        # 简化的评分逻辑
        performance_score = 0
        risk_score = 0
        stability_score = 0

        # 性能评分 (40%)
        ic_1d = evaluation['performance_metrics']['IC_1D']
        performance_score = (ic_1d['ic_mean'] * 0.4 +
                            ic_1d['ic_ir'] * 0.3 +
                            ic_1d['ic_positive_ratio'] * 0.3) * 100

        # 风险评分 (30%)
        risk_metrics = evaluation['risk_metrics']
        risk_score = (risk_metrics['sharpe_ratio'] * 50 +
                     (1 + risk_metrics['max_drawdown']) * 50)

        # 稳定性评分 (30%)
        stability_metrics = evaluation['stability_metrics']
        stability_score = ((1 - stability_metrics['avg_turnover']) * 50 +
                         stability_metrics['factor_decay']['lag_1'] * 50)

        overall_score = (performance_score * 0.4 +
                         risk_score * 0.3 +
                         stability_score * 0.3)

        return round(overall_score, 2)

    def generate_report(self, evaluation, output_format='text'):
        """生成评估报告"""
        if output_format == 'text':
            return self._generate_text_report(evaluation)
        elif output_format == 'html':
            return self._generate_html_report(evaluation)
        else:
            return evaluation

    def _generate_text_report(self, evaluation):
        """生成文本报告"""
        report = f"""
# {evaluation['factor_name']} 因子评估报告

## 基本信息
- 因子名称: {evaluation['factor_name']}
- 描述: {evaluation['description']}
- 评估日期: {evaluation['evaluation_date'].strftime('%Y-%m-%d %H:%M:%S')}
- 总体评分: {evaluation['overall_score']}/100

## 性能指标
"""
        # IC指标
        for period, stats in evaluation['performance_metrics'].items():
            report += f"""
### {period}
- IC均值: {stats['ic_mean']:.4f}
- IC标准差: {stats['ic_std']:.4f}
- IC_IR: {stats['ic_ir']:.4f}
- IC胜率: {stats['ic_positive_ratio']:.2%}
"""

        # 风险指标
        risk = evaluation['risk_metrics']
        report += f"""
## 风险指标
- 最大回撤: {risk['max_drawdown']:.2%}
- 年化波动率: {risk['volatility']:.2%}
- 夏普比率: {risk['sharpe_ratio']:.4f}
"""

        # 稳定性指标
        stability = evaluation['stability_metrics']
        report += f"""
## 稳定性指标
- 平均换手率: {stability['avg_turnover']:.2%}
- 换手率标准差: {stability['turnover_std']:.2%}
- 因子衰减度 (1日滞后): {stability['factor_decay']['lag_1']:.4f}
"""

        return report

# 使用示例
def factor_evaluation_example():
    """因子评估示例"""
    evaluator = FactorEvaluator()

    # 模拟因子值和收益率
    dates = pd.date_range('20240101', '20240320', freq='D')
    symbols = ['000001.SZ', '600000.SS']

    index = pd.MultiIndex.from_product([symbols, dates], names=['symbol', 'date'])
    factor_values = pd.Series(np.random.randn(len(index)), index=index)
    returns = pd.Series(np.random.randn(len(index)) * 0.02, index=index)

    # 综合评估
    evaluation = evaluator.comprehensive_evaluation(
        factor_name="Test_Momentum_Factor",
        factor_values=factor_values,
        returns=returns,
        factor_description="测试动量因子"
    )

    # 生成报告
    report = evaluator.generate_report(evaluation)
    print(report)

    return evaluation
```

## 📝 使用建议

1. **因子选择**: 根据投资目标选择合适的因子类型
2. **参数调优**: 使用历史数据优化因子参数
3. **组合策略**: 多因子组合通常比单一因子更稳定
4. **风险控制**: 重视因子换手率和最大回撤
5. **定期评估**: 定期评估因子表现并调整策略

## 🚀 进阶应用

这些示例代码可以作为基础，进一步扩展：

1. **机器学习集成**: 将因子值作为ML模型特征
2. **实时计算**: 用于高频交易中的实时因子计算
3. **风险管理**: 集成到风险模型中进行风险控制
4. **组合优化**: 使用现代投资组合理论优化因子权重

希望这些示例能帮助您更好地使用PandaFactor平台！