# PandaFactor 快速入门教程

## 🎯 项目概述

PandaFactor 是一个专业的量化因子开发和分析平台，为金融数据分析、技术指标计算和因子构建提供高性能工具。

### 核心特性
- 🚀 **高性能计算**: 基于NumPy和Pandas优化的量化算子
- 📊 **多数据源支持**: Tushare、RiceQuant、迅投等主流数据源
- 🔧 **灵活开发**: 支持Python类和公式两种因子开发方式
- 📈 **完整分析**: IC分析、分层回测、相关性分析等
- 🌐 **REST API**: 提供完整的因子管理和查询接口
- ⏰ **自动化更新**: 定时数据清洗和因子更新

## 📋 系统要求

- **Python**: 3.12+
- **操作系统**: Windows 10/11, macOS 10.15+, Ubuntu 20.04+
- **内存**: 8GB+ (推荐16GB+)
- **磁盘空间**: 50GB+ (用于历史数据存储)
- **数据库**: MongoDB 4.4+ (本地或云端)

## 🚀 快速开始

### 方式一：一键安装（推荐新手）

```bash
# 1. 下载预置数据库版本（包含5年历史数据）
# 联系助理获取百度网盘链接：
# 链接: https://pan.baidu.com/s/15jip2SATiORuqaBNMDm4fw?pwd=uupm

# 2. 下载并解压到本地目录

# 3. 启动数据库
# Windows: 双击 bin/db_start.bat
# macOS/Linux: 运行 bin/db_start.sh

# 4. 验证安装
# 浏览器访问: http://localhost:8111/docs
# 看到 API 文档页面即表示安装成功
```

### 方式二：源码安装（推荐开发者）

```bash
# 1. 克隆项目
git clone https://github.com/your-repo/panda_factor.git
cd panda_factor

# 2. 安装 uv (更快的包管理器)
pip install uv

# 3. 安装所有依赖
uv sync --all-extras

# 4. 配置数据库
# 编辑 panda_common/panda_common/config.yaml
# 修改 MongoDB 连接信息

# 5. 启动服务
# 终端1: 启动API服务器
uv run python -m panda_factor_server.__main__

# 终端2: 启动数据更新服务（可选）
uv run python -m panda_data_hub._main_auto_
```

## ⚙️ 配置指南

### 数据库配置

编辑 `panda_common/panda_common/config.yaml`：

```yaml
# MongoDB 配置
MONGO_USER: "panda"
MONGO_PASSWORD: "panda"
MONGO_URI: "127.0.0.1:27017"
MONGO_DB: "panda"
MONGO_TYPE: "single"  # 或 "replica_set"

# 数据源配置
DATASOURCE: "tushare"  # 可选: tushare, ricequant, xtquant, tqsdk

# Tushare 配置（如使用）
TS_TOKEN: "your_tushare_token"

# 米筐配置（如使用）
MUSER: "your_ricequant_username"
MPASSWORD: "your_ricequant_password"

# 日志配置
LOG_LEVEL: "INFO"
```

### 数据源选择

| 数据源 | 优势 | 适用场景 | 配置难度 |
|--------|------|----------|----------|
| **Tushare** | 数据全面、免费 | 个人开发者 | ⭐⭐ |
| **RiceQuant** | 数据质量高 | 专业机构 | ⭐⭐⭐ |
| **迅投** | 实时数据 | 量化团队 | ⭐⭐⭐⭐ |
| **TQSdk** | 期货数据专精 | 期货策略 | ⭐⭐ |

## 📝 第一个因子

### Python 方式（推荐）

```python
# 创建文件 my_first_factor.py
from panda_factor.generate.factor_base import Factor

class MomentumFactor(Factor):
    """20日动量因子"""
    def calculate(self, factors):
        close = factors['close']
        # 计算20日收益率
        returns = (close / self.DELAY(close, 20)) - 1
        # 横截面排名
        result = self.RANK(returns)
        return result

# 使用因子
import panda_data
panda_data.init()

# 创建因子实例
factor = MomentumFactor()

# 获取数据并计算因子
factors_data = panda_data.get_market_data(
    start_date='20240101',
    end_date='20240320',
    symbols=['000001.SZ', '600000.SS'],
    fields=['close']
)

# 计算因子值
factor_values = factor.calculate(factors_data)
print(factor_values.head())
```

### 公式方式（快速原型）

```python
# 简单动量因子
factor_formula = "RANK((CLOSE / DELAY(CLOSE, 20)) - 1)"

# 复杂多步骤因子
factor_formula = """
returns = (CLOSE / DELAY(CLOSE, 20)) - 1
volume_ma = SUM(VOLUME, 20) / 20
volume_ratio = VOLUME / volume_ma
momentum = RANK(returns)
final_result = momentum * SCALE(volume_ratio)
"""

# 价格成交量相关性
factor_formula = "CORRELATION(CLOSE, VOLUME, 20)"
```

## 🔧 核心功能详解

### 内置函数速查

#### 时间序列函数
```python
self.DELAY(series, n)        # 延迟n期
self.SUM(series, n)           # n期求和
self.MEAN(series, n)          # n期均值
self.STDDEV(series, n)        # n期标准差
self.MAX(series, n)           # n期最大值
self.MIN(series, n)           # n期最小值
```

#### 收益率函数
```python
self.RETURNS(series, n)       # n期收益率
self.FUTURE_RETURNS(series, n)  # n期未来收益率
```

#### 截面函数
```python
self.RANK(series)              # 横截面排名 [-0.5, 0.5]
self.SCALE(series)             # 标准化
self.ZSCORE(series, n)         # n期Z-score
```

#### 逻辑函数
```python
self.IF(condition, x, y)       # 条件判断
self.ABS(series)               # 绝对值
self.LOG(series)               # 对数
self.POWER(series, n)          # n次方
self.SIGN(series)              # 符号函数
```

#### 高级函数
```python
self.CORRELATION(series1, series2, n)  # n期相关性
self.COVARIANCE(series1, series2, n)    # n期协方差
self.WINSORIZE(series, n)               # n期缩尾处理
```

### 可用数据字段

| 字段 | 说明 | 数据类型 |
|------|------|----------|
| `close` | 收盘价 | float |
| `open` | 开盘价 | float |
| `high` | 最高价 | float |
| `low` | 最低价 | float |
| `volume` | 成交量 | float |
| `amount` | 成交额 | float |
| `turnover` | 换手率 | float |
| `market_cap` | 总市值 | float |
| `pe_ratio` | 市盈率 | float |
| `pb_ratio` | 市净率 | float |

## 📊 数据获取API

### 基础市场数据

```python
import panda_data

# 初始化系统
panda_data.init()

# 获取日级数据
daily_data = panda_data.get_market_data(
    start_date='20240101',
    end_date='20240320',
    symbols=['000001.SZ', '600000.SS'],  # 支持多股票
    fields=['open', 'close', 'high', 'low', 'volume', 'amount']
)

# 获取分钟级数据
minute_data = panda_data.get_market_min_data(
    start_date='20240320',
    end_date='20240320',
    symbol='000001.SZ',  # 单股票
    fields=['close', 'volume']
)

# 获取所有A股代码
all_symbols = panda_data.get_all_symbols()
print(f"总计 {len(all_symbols)} 只股票")
```

### 因子数据获取

```python
# 获取预定义因子
factor_data = panda_data.get_factor(
    factors=['momentum_20d', 'volatility_20d'],
    start_date='20240101',
    end_date='20240320'
)

# 按因子名称获取
single_factor = panda_data.get_factor_by_name(
    factor_name="VH03cc651",  # 因子唯一ID
    start_date='20240320',
    end_date='20250325'
)

# 查看因子统计信息
print(f"因子均值: {single_factor.mean():.4f}")
print(f"因子标准差: {single_factor.std():.4f}")
print(f"缺失值数量: {single_factor.isna().sum()}")
```

### 数据格式说明

所有数据都使用标准的 `MultiIndex` 格式：

```python
# 数据结构示例
import pandas as pd

index = pd.MultiIndex.from_product(
    [['000001.SZ', '600000.SS'], ['20240101', '20240102']],
    names=['symbol', 'date']
)
data = pd.Series([1.2, 1.3, 0.8, 0.9], index=index)

# 访问特定股票数据
stock_data = data.loc['000001.SZ']
# 20240101    1.2
# 20240102    1.3

# 访问特定日期数据
date_data = data.xs('20240101', level='date')
# 000001.SZ    1.2
# 600000.SS    0.8
```

## 🎯 实战案例

### 案例1：技术指标因子

```python
class RSIFactor(Factor):
    """RSI相对强弱指标"""
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
        # 计算RSI
        rs = avg_gain / (avg_loss + 1e-6)  # 避免除零
        rsi = 100 - (100 / (1 + rs))
        # 标准化到[-0.5, 0.5]
        return (rsi / 100) - 0.5
```

### 案例2：成交量因子

```python
class VolumePriceFactor(Factor):
    """量价配合因子"""
    def calculate(self, factors):
        close = factors['close']
        volume = factors['volume']
        high = factors['high']
        low = factors['low']

        # 价格动量
        price_momentum = self.RANK((close / self.DELAY(close, 20)) - 1)

        # 成交量放大
        volume_ma = self.SUM(volume, 20) / 20
        volume_ratio = volume / volume_ma
        volume_signal = self.RANK(volume_ratio)

        # 价格振幅
        price_range = (high - low) / close
        volatility_signal = self.RANK(price_range)

        # 综合信号
        return price_momentum * volume_signal * volatility_signal
```

### 案例3：基本面因子

```python
class ValueFactor(Factor):
    """价值因子"""
    def calculate(self, factors):
        pe_ratio = factors['pe_ratio']
        pb_ratio = factors['pb_ratio']
        market_cap = factors['market_cap']

        # 处理极端值
        pe_clean = self.WINSORIZE(pe_ratio, 5)
        pb_clean = self.WINSORIZE(pb_ratio, 5)

        # 取倒数（值越小越好）
        pe_inverse = 1 / (pe_clean + 1e-6)
        pb_inverse = 1 / (pb_clean + 1e-6)

        # 标准化
        pe_signal = self.SCALE(pe_inverse)
        pb_signal = self.SCALE(pb_inverse)

        # 等权组合
        return (pe_signal + pb_signal) / 2
```

## 📈 因子分析

### IC分析

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

print(f"IC均值: {ic_analysis.mean():.4f}")
print(f"IC标准差: {ic_analysis.std():.4f}")
print(f"IC_IR: {ic_analysis.mean() / ic_analysis.std():.4f}")
```

### 分层回测

```python
# 分层回测
layer_analysis = analyzer.layer_backtest(
    factor_values=factor_data,
    returns_data=returns_data,
    layers=5  # 分5层
)

# 查看各层收益
for layer, returns in layer_analysis.items():
    annual_return = (1 + returns.mean()) ** 252 - 1
    print(f"第{layer}层年化收益: {annual_return:.2%}")
```

## 🛠️ 常见问题解决

### Q1: 如何处理数据缺失？

```python
class RobustFactor(Factor):
    def calculate(self, factors):
        close = factors['close']

        # 前向填充缺失值
        close = close.groupby('symbol').fillna(method='ffill')

        # 检查数据充足性
        if len(close) < 20:
            return pd.Series(0, index=close.index)

        # 正常计算
        return self.RANK((close / self.DELAY(close, 20)) - 1)
```

### Q2: 如何提高计算性能？

```python
# 好的做法：向量化操作
def efficient_factor(factors):
    close = factors['close']
    volume = factors['volume']

    # 向量化计算
    returns = close.groupby('symbol').pct_change(20)
    volume_ma = volume.groupby('symbol').rolling(20).mean()

    return returns * (volume / volume_ma)

# 避免：逐股票循环
def inefficient_factor(factors):
    results = []
    for symbol in factors['close'].index.get_level_values('symbol').unique():
        symbol_data = factors['close'].loc[symbol]
        # 逐个计算...
        results.append(symbol_result)
    return pd.concat(results)
```

### Q3: 如何调试因子计算？

```python
class DebugFactor(Factor):
    def calculate(self, factors):
        close = factors['close']

        # 打印调试信息
        print(f"数据形状: {close.shape}")
        print(f"时间范围: {close.index.get_level_values('date').min()} - {close.index.get_level_values('date').max()}")
        print(f"缺失值: {close.isna().sum()}")

        # 逐步计算
        delayed_close = self.DELAY(close, 20)
        returns = (close / delayed_close) - 1

        print(f"收益率统计: 均值={returns.mean():.4f}, 标准差={returns.std():.4f}")

        result = self.RANK(returns)
        return result
```

## 🌐 API服务使用

### 启动API服务

```bash
# 启动服务器
uv run python -m panda_factor_server.__main__

# 服务地址: http://localhost:8111
# API文档: http://localhost:8111/docs
```

### 主要API端点

```python
import requests

# 基础URL
BASE_URL = "http://localhost:8111"

# 1. 获取市场数据
response = requests.get(f"{BASE_URL}/market_data", params={
    "symbols": "000001.SZ,600000.SS",
    "start_date": "20240101",
    "end_date": "20240320",
    "fields": "close,volume"
})
market_data = response.json()

# 2. 计算自定义因子
factor_code = """
class MyFactor(Factor):
    def calculate(self, factors):
        return self.RANK((factors['close'] / self.DELAY(factors['close'], 20)) - 1)
"""

response = requests.post(f"{BASE_URL}/factor/calculate", json={
    "factor_code": factor_code,
    "start_date": "20240101",
    "end_date": "20240320",
    "symbols": "000001.SZ,600000.SS"
})
factor_result = response.json()

# 3. 获取因子列表
response = requests.get(f"{BASE_URL}/factors")
factors_list = response.json()
```

## 📚 学习路径建议

### 初级阶段（1-2周）
1. 熟悉Python基础和Pandas操作
2. 学习基础的量化概念（收益率、波动率等）
3. 完成系统安装和配置
4. 实现简单的技术指标因子（如均线、动量）

### 中级阶段（1-2个月）
1. 学习多因子模型理论
2. 掌握IC分析和分层回测方法
3. 实现复杂的多步骤因子
4. 学习基本面数据处理方法

### 高级阶段（3-6个月）
1. 研究机器学习在因子挖掘中的应用
2. 学习因子组合和权重优化方法
3. 掌握实时因子计算系统设计
4. 参与量化比赛和社区交流

## 🤝 社区与支持

### 官方资源
- **GitHub仓库**: https://github.com/your-repo/panda_factor
- **官方文档**: https://docs.pandaai.online
- **因子大赛**: https://www.pandaai.online/factorhub/factorcompetition

### 交流群组
- **微信交流群**: 扫描README中的二维码
- **QQ群**: 123456789
- **Discord**: https://discord.gg/pandaai

### 技术支持
- **Issue反馈**: GitHub Issues
- **邮件联系**: support@pandaai.online
- **在线客服**: 工作日 9:00-18:00

## 📄 许可证

本项目采用 GPL-3.0 许可证，详见 [LICENSE](LICENSE) 文件。

---

**开始量化，最好是十年前，其次是现在！** 🚀

如果您觉得这个项目对您有帮助，请给我们一个 ⭐ Star！