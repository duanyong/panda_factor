# PandaFactor 数据更新完全指南

## 目录
1. [数据更新架构概述](#数据更新架构概述)
2. [数据源配置](#数据源配置)
3. [自动更新机制](#自动更新机制)
4. [手动数据更新](#手动数据更新)
5. [数据清洗流程](#数据清洗流程)
6. [数据库存储结构](#数据库存储结构)
7. [监控和故障排除](#监控和故障排除)
8. [最佳实践](#最佳实践)

## 数据更新架构概述

PandaFactor的数据更新系统采用分层架构，支持多数据源和自动调度机制。

### 系统架构图

```
┌─────────────────┐    ┌─────────────────┐
│   数据源层      │    │   调度层        │
├─────────────────┤    ├─────────────────┤
│ Tushare        │    │ APScheduler     │
│ RiceQuant      │◄──►│ Cron触发器      │
│ 迅投(XTQuant)  │    │ BackgroundTasks │
│ QMT            │    └─────────────────┘
│ Wind           │             │
│ Choice         │             ▼
└─────────────────┘    ┌─────────────────┐
         │              │   业务逻辑层    │
         ▼              ├─────────────────┤
┌─────────────────┐    │ 数据清洗器      │
│   存储层        │    │ 因子计算器      │
├─────────────────┤    │ 元数据管理器    │
│ MongoDB         │    │ 进度监控器      │
│ stocks集合      │    └─────────────────┘
│ stock_market集合│             │
│ factor_base集合  │             ▼
│ factors集合     │    ┌─────────────────┐
│ logs集合        │    │   API接口层     │
└─────────────────┘    ├─────────────────┤
                       │ REST API        │
                       │ Web界面         │
                       │ 进度查询        │
                       └─────────────────┘
```

### 核心组件说明

1. **panda_data_hub**: 数据更新核心模块
2. **调度器**: APScheduler实现定时任务
3. **清洗器**: 各种数据源的专用清洗器
4. **数据库**: MongoDB存储清洗后的数据
5. **API服务**: FastAPI提供管理和监控接口

## 数据源配置

### 1. 支持的数据源

| 数据商 | 支持状态 | 需要配置 | 特殊说明 |
|--------|----------|----------|----------|
| Tushare | ✅ 已上线 | `TS_TOKEN` | 免费版有限制 |
| RiceQuant | ✅ 已上线 | `MUSER`, `MPASSWORD` | 需要账号 |
| 迅投(XTQuant) | ✅ 已上线 | `XT_TOKEN` | 不支持macOS |
| 天勤(Tqsdk) | 🚧 测试中 | `TQSDK_USERNAME`, `TQSDK_PASSWORD` | - |
| QMT | 🚧 测试中 | QMT配置 | - |
| Wind | 🔧 对接中 | Wind配置 | - |
| Choice | 🔧 对接中 | Choice配置 | - |

### 2. 配置文件详解

编辑 `panda_common/config.yaml`：

```yaml
# 数据源选择 (tushare/ricequant/xtquant/tqsdk)
DATASOURCE: "tushare"
DATAHUBSOURCE: "tushare"  # 数据清洗专用数据源

# Tushare配置
TS_TOKEN: "你的tushare_token"

# RiceQuant配置
MUSER: "your_ricequant_username"
MPASSWORD: "your_ricequant_password"

# 迅投配置
XT_TOKEN: "your_xt_token"

# 数据更新时间配置
UPDATE_TIME: '20:00'              # 通用更新时间
STOCKS_UPDATE_TIME: "20:00"       # 股票数据更新时间
FACTOR_UPDATE_TIME: "20:30"       # 因子数据更新时间

# 数据清洗时间范围
HUB_START_DATE: 20170101          # 历史数据开始日期
HUB_END_DATE: 20250321            # 历史数据结束日期
```

### 3. 数据源切换

```python
# 运行时切换数据源
import os

# 方法1：通过环境变量
os.environ['DATASOURCE'] = 'ricequant'
os.environ['MUSER'] = 'your_username'
os.environ['MPASSWORD'] = 'your_password'

# 方法2：修改配置文件后重启
# 编辑 panda_common/config.yaml
# 重启服务使配置生效
```

## 自动更新机制

### 1. 自动更新调度器

PandaFactor使用双重调度器分别处理股票数据和因子数据：

```python
# 启动自动更新服务
uv run python -m panda_data_hub._main_auto_
```

#### 股票数据调度器 (DataScheduler)

```python
# 配置的执行时间：默认每日20:00
STOCKS_UPDATE_TIME: "20:00"

# 执行流程：
1. 检查是否为交易日
2. 清洗股票元数据 (stocks集合)
3. 清洗市场行情数据 (stock_market集合)
4. 更新指数成分股信息
```

#### 因子数据调度器 (FactorCleanerScheduler)

```python
# 配置的执行时间：默认每日20:30
FACTOR_UPDATE_TIME: "20:30"

# 执行流程：
1. 基于当日市场数据
2. 计算基础因子数据
3. 计算扩展因子数据
4. 存储到factor_base集合
```

### 2. 调度器详细配置

```python
# panda_data_hub/task/data_scheduler.py
class DataScheduler:
    def schedule_data(self):
        # 解析配置的时间
        time = self.config["STOCKS_UPDATE_TIME"]  # "20:00"
        hour, minute = time.split(":")

        # 创建Cron触发器
        trigger = CronTrigger(
            minute=minute,      # 分钟
            hour=hour,          # 小时
            day='*',           # 每日
            month='*',         # 每月
            day_of_week='*'    # 每周
        )

        # 添加定时任务
        self.scheduler.add_job(
            self._process_data,
            trigger=trigger,
            id=f"data_{datetime.now().strftime('%Y%m%d')}",
            replace_existing=True
        )
```

### 3. 高级调度配置

```python
# 自定义调度时间
STOCKS_UPDATE_TIME: "19:30"    # 工作日19:30更新
FACTOR_UPDATE_TIME: "22:00"     # 工作日22:00更新

# 周末不执行调度 (需要代码修改)
def is_weekday():
    return datetime.now().weekday() < 5  # 0-4为周一到周五

# 在_process_data中添加检查
def _process_data(self):
    if not is_weekday():
        logger.info("周末跳过数据更新")
        return
    # 执行数据清洗...
```

### 4. 调度器监控

```python
# 查看调度器状态
import requests

# API端点 (需要启动panda_data_hub服务)
response = requests.get("http://localhost:8222/datahub/api/v1/get_progress")
print(response.json())

# 检查任务是否正在运行
scheduler.get_job(job_id="data_20250327")
```

## 手动数据更新

### 1. 启动数据更新服务

```bash
# 启动数据更新API服务
uv run python -m panda_data_hub._main_clean_

# 服务将在 http://localhost:8222 启动
```

### 2. Web界面操作

访问 `http://localhost:8222/docs` 查看API文档，或使用前端界面：
- `http://localhost:8080/factor/#/datahubsource` - 数据源配置
- `http://localhost:8080/factor/#/datahublist` - 数据列表查看
- `http://localhost:8080/factor/#/datahubdataclean` - 股票数据清洗
- `http://localhost:8080/factor/#/datahubFactorClean` - 因子数据清洗

### 3. API接口调用

#### 股票数据清洗

```python
import requests

# 清洗指定日期范围的股票数据
base_url = "http://localhost:8222/datahub/api/v1"

# 启动股票数据清洗
response = requests.get(
    f"{base_url}/upsert_stockmarket_final",
    params={
        "start_date": "20240101",
        "end_date": "20240320"
    }
)

# 检查进度
response = requests.get(f"{base_url}/get_progress")
print(response.json())
```

#### 因子数据清洗

```python
# 启动因子数据清洗
response = requests.get(
    f"{base_url}/upsert_factor_final",
    params={
        "start_date": "20240101",
        "end_date": "20240320"
    }
)

# 检查因子清洗进度
response = requests.get(f"{base_url}/get_progress_factor")
print(response.json())
```

### 4. 命令行手动更新

```python
# 直接调用清洗器 (高级用法)
from panda_data_hub.data.tushare_stock_market_cleaner import TSStockMarketCleaner
from panda_common.config import get_config

# 获取配置
config = get_config()

# 创建清洗器实例
cleaner = TSStockMarketCleaner(config)

# 执行当日数据清洗
cleaner.stock_market_clean_daily()

# 执行历史数据清洗
cleaner.clean_meta_market_data("20240320")
```

### 5. 批量历史数据更新

```python
# 批量更新历史数据
from datetime import datetime, timedelta

def batch_update(start_date, end_date, batch_size=30):
    current_date = datetime.strptime(start_date, "%Y%m%d")
    end_date_obj = datetime.strptime(end_date, "%Y%m%d")

    while current_date <= end_date_obj:
        batch_end = min(current_date + timedelta(days=batch_size-1), end_date_obj)

        # 调用API进行批量更新
        response = requests.get(
            f"{base_url}/upsert_stockmarket_final",
            params={
                "start_date": current_date.strftime("%Y%m%d"),
                "end_date": batch_end.strftime("%Y%m%d")
            }
        )

        print(f"Started batch: {current_date.strftime('%Y%m%d')} - {batch_end.strftime('%Y%m%d')}")

        # 移动到下一批次
        current_date = batch_end + timedelta(days=1)

        # 等待一段时间避免API限制
        time.sleep(10)

# 使用示例
batch_update("20240101", "20240320", batch_size=20)
```

## 数据清洗流程

### 1. 股票数据清洗流程

#### Tushare数据源清洗流程

```python
# 完整的股票数据清洗流程
class TSStockMarketCleaner:
    def stock_market_clean_daily(self):
        # 1. 判断是否为交易日
        date_str = datetime.now().strftime("%Y%m%d")
        if not self.is_trading_day(date_str):
            logger.info(f"跳过非交易日: {date_str}")
            return

        # 2. 获取基础行情数据
        price_data = self.pro.query('daily', trade_date=date_str)

        # 3. 获取指数成分股信息
        hs_300 = self.pro.query('index_weight', index_code='399300.SZ',
                                start_date=mid_date, end_date=last_date)
        zz_500 = self.pro.query('index_weight', index_code='000905.SH',
                                start_date=mid_date, end_date=last_date)
        zz_1000 = self.pro.query('index_weight', index_code='000852.SH',
                                 start_date=mid_date, end_date=last_date)

        # 4. 数据处理和清洗
        # - 添加股票名称
        # - 计算涨跌停价格
        # - 添加指数成分股标识
        # - 标准化股票代码格式

        # 5. 数据入库
        upsert_operations = []
        for record in price_data.to_dict('records'):
            upsert_operations.append(UpdateOne(
                {'date': record['date'], 'symbol': record['symbol']},
                {'$set': record},
                upsert=True
            ))

        self.db_handler.mongo_client[db_name]['stock_market'].bulk_write(upsert_operations)
```

#### 数据清洗详细步骤

```python
def clean_meta_market_data(self, date_str):
    """详细的股票市场数据清洗流程"""

    # Step 1: 获取基础数据
    price_data = self.pro.query('daily', trade_date=date_str)

    # Step 2: 数据预处理
    price_data.reset_index(drop=False, inplace=True)
    price_data['index_component'] = None

    # Step 3: 获取指数成分股
    mid_date, last_date = self.get_previous_month_dates(date)
    indices = {
        'hs300': self.pro.query('index_weight', index_code='399300.SZ',
                               start_date=mid_date, end_date=last_date),
        'zz500': self.pro.query('index_weight', index_code='000905.SH',
                               start_date=mid_date, end_date=last_date),
        'zz1000': self.pro.query('index_weight', index_code='000852.SH',
                                start_date=mid_date, end_date=last_date)
    }

    # Step 4: 处理每只股票的数据
    for idx, row in price_data.iterrows():
        # 标记指数成分股
        component = self.clean_index_components(
            data_symbol=row['ts_code'],
            date=date,
            **indices
        )
        price_data.at[idx, 'index_component'] = component

        # 添加股票名称
        price_data.at[idx, 'name'] = self.clean_stock_name(row['ts_code'])

    # Step 5: 数据转换和清洗
    price_data = self.transform_price_data(price_data)

    # Step 6: 计算衍生数据
    price_data = self.calculate_derived_fields(price_data)

    # Step 7: 数据验证
    price_data = self.validate_data(price_data)

    # Step 8: 入库
    self.upsert_to_database(price_data)
```

### 2. 因子数据清洗流程

```python
class TSFactorCleaner:
    def clean_daily_factor(self):
        """因子数据清洗流程"""

        # 1. 获取当日市场数据作为基础
        date = datetime.now().strftime('%Y%m%d')
        market_data = self.get_market_data(date)

        if market_data is None or len(market_data) == 0:
            logger.info(f"No market data for {date}")
            return

        # 2. 数据预处理
        data = pd.DataFrame(list(market_data))
        data = self.preprocess_factor_data(data)

        # 3. 获取市值和换手率数据
        logger.info("获取市值和换手率数据...")
        basic_data = self.pro.query('daily_basic',
                                   trade_date=date,
                                   fields=['ts_code','turnover_rate','total_mv'])

        # 4. 获取成交额数据
        logger.info("获取成交额数据...")
        amount_data = self.pro.query("daily",
                                    trade_date=date,
                                    fields=['ts_code', 'amount'])

        # 5. 数据合并和处理
        factor_data = self.merge_factor_sources(data, basic_data, amount_data)

        # 6. 数据标准化
        factor_data = self.normalize_factor_data(factor_data)

        # 7. 入库
        self.upsert_factor_data(factor_data)
```

### 3. 数据质量控制

```python
class DataQualityController:
    def validate_market_data(self, data):
        """市场数据质量检查"""

        # 1. 基础数据完整性检查
        required_fields = ['open', 'high', 'low', 'close', 'volume']
        for field in required_fields:
            if field not in data.columns:
                raise ValueError(f"Missing required field: {field}")

        # 2. 价格合理性检查
        invalid_prices = (
            (data['high'] < data['low']) |  # 最高价不能低于最低价
            (data['close'] > data['high']) |  # 收盘价不能高于最高价
            (data['close'] < data['low']) |  # 收盘价不能低于最低价
            (data['open'] > data['high']) |   # 开盘价不能高于最高价
            (data['open'] < data['low'])      # 开盘价不能低于最低价
        )

        if invalid_prices.any():
            logger.warning(f"Found {invalid_prices.sum()} records with invalid price relationships")
            data = data[~invalid_prices]  # 移除无效数据

        # 3. 成交量检查
        negative_volume = data['volume'] < 0
        if negative_volume.any():
            logger.warning(f"Found {negative_volume.sum()} records with negative volume")
            data.loc[negative_volume, 'volume'] = 0  # 修正为0

        # 4. 涨跌停检查
        extreme_changes = data['pct_chg'].abs() > 20  # 涨跌幅超过20%
        if extreme_changes.any():
            logger.info(f"Found {extreme_changes.sum()} records with extreme price changes")

        # 5. 异常值检测
        outliers = self.detect_outliers(data)
        if len(outliers) > 0:
            logger.info(f"Detected {len(outliers)} potential outliers")

        return data

    def detect_outliers(self, data, threshold=3):
        """使用IQR方法检测异常值"""
        outliers = []

        for column in ['open', 'high', 'low', 'close', 'volume']:
            Q1 = data[column].quantile(0.25)
            Q3 = data[column].quantile(0.75)
            IQR = Q3 - Q1

            lower_bound = Q1 - threshold * IQR
            upper_bound = Q3 + threshold * IQR

            column_outliers = data[(data[column] < lower_bound) | (data[column] > upper_bound)]
            outliers.extend(column_outliers.index.tolist())

        return list(set(outliers))
```

## 数据库存储结构

### 1. MongoDB集合结构

#### stocks集合 (股票基础信息)

```javascript
{
  "_id": ObjectId("..."),
  "symbol": "000001.SZ",        // 标准化股票代码
  "name": "平安银行",            // 股票名称
  "expired": false,             // 是否退市
  "created_at": ISODate("..."),
  "updated_at": ISODate("...")
}

// 索引
db.stocks.createIndex({"symbol": 1}, {unique: true})
db.stocks.createIndex({"expired": 1})
```

#### stock_market集合 (日线行情数据)

```javascript
{
  "_id": ObjectId("..."),
  "date": "20240320",           // 交易日期 YYYYMMDD
  "symbol": "000001.SZ",       // 股票代码
  "open": 10.50,               // 开盘价
  "high": 10.80,               // 最高价
  "low": 10.40,                // 最低价
  "close": 10.75,              // 收盘价
  "volume": 1000000,           // 成交量(股)
  "pre_close": 10.60,          // 昨收价
  "limit_up": 11.66,           // 涨停价
  "limit_down": 9.54,          // 跌停价
  "index_component": "hs300",  // 指数成分股标识
  "name": "平安银行",          // 股票名称
  "created_at": ISODate("..."),
  "updated_at": ISODate("...")
}

// 复合索引
db.stock_market.createIndex({"date": 1, "symbol": 1}, {unique: true})
db.stock_market.createIndex({"symbol": 1, "date": 1})
db.stock_market.createIndex({"date": 1})
```

#### factor_base集合 (基础因子数据)

```javascript
{
  "_id": ObjectId("..."),
  "date": "20240320",           // 交易日期
  "symbol": "000001.SZ",       // 股票代码
  "open": 10.50,               // 开盘价
  "high": 10.80,               // 最高价
  "low": 10.40,                // 最低价
  "close": 10.75,              // 收盘价
  "volume": 1000000,           // 成交量
  "market_cap": 50000000000,   // 总市值(元)
  "turnover": 0.05,            // 换手率
  "amount": 10750000,          // 成交额(元)
  "created_at": ISODate("..."),
  "updated_at": ISODate("...")
}

// 复合索引
db.factor_base.createIndex({"date": 1, "symbol": 1}, {unique: true})
db.factor_base.createIndex({"symbol": 1, "date": 1})
```

#### factors集合 (计算因子数据)

```javascript
{
  "_id": ObjectId("..."),
  "date": "20240320",           // 交易日期
  "symbol": "000001.SZ",       // 股票代码
  "factor_name": "momentum_20d", // 因子名称
  "factor_value": 0.125,        // 因子值
  "user_id": 123,              // 用户ID(自定义因子)
  "created_at": ISODate("..."),
  "updated_at": ISODate("...")
}

// 复合索引
db.factors.createIndex({"date": 1, "symbol": 1, "factor_name": 1}, {unique: true})
db.factors.createIndex({"factor_name": 1, "date": 1})
db.factors.createIndex({"user_id": 1, "factor_name": 1})
```

### 2. 数据库操作工具

```python
from pymongo import UpdateOne, DeleteOne
from panda_common.handlers.database_handler import DatabaseHandler

class DatabaseUtils:
    def __init__(self, config):
        self.db_handler = DatabaseHandler(config)
        self.db_name = config["MONGO_DB"]

    def bulk_upsert_stock_market(self, data):
        """批量更新股票市场数据"""
        collection = self.db_handler.mongo_client[self.db_name]['stock_market']

        operations = []
        for record in data.to_dict('records'):
            operations.append(UpdateOne(
                {'date': record['date'], 'symbol': record['symbol']},
                {'$set': record},
                upsert=True
            ))

        if operations:
            result = collection.bulk_write(operations)
            logger.info(f"Upserted {result.upserted_count} stock market records")
            return result

        return None

    def bulk_upsert_factors(self, factor_data, factor_name, user_id=None):
        """批量更新因子数据"""
        collection = self.db_handler.mongo_client[self.db_name]['factors']

        operations = []
        for (date, symbol), value in factor_data.items():
            record = {
                'date': date,
                'symbol': symbol,
                'factor_name': factor_name,
                'factor_value': value,
                'updated_at': datetime.now()
            }
            if user_id:
                record['user_id'] = user_id

            operations.append(UpdateOne(
                {
                    'date': date,
                    'symbol': symbol,
                    'factor_name': factor_name
                },
                {'$set': record},
                upsert=True
            ))

        if operations:
            result = collection.bulk_write(operations)
            logger.info(f"Upserted {result.upserted_count} factor records")
            return result

        return None

    def clean_stale_data(self, days_to_keep=365):
        """清理过期数据"""
        cutoff_date = (datetime.now() - timedelta(days=days_to_keep)).strftime("%Y%m%d")

        # 清理旧的stock_market数据
        result1 = self.db_handler.mongo_delete_many(
            self.db_name,
            'stock_market',
            {'date': {'$lt': cutoff_date}}
        )

        # 清理旧的factor_base数据
        result2 = self.db_handler.mongo_delete_many(
            self.db_name,
            'factor_base',
            {'date': {'$lt': cutoff_date}}
        )

        logger.info(f"Cleaned {result1.deleted_count} stock_market records")
        logger.info(f"Cleaned {result2.deleted_count} factor_base records")
```

### 3. 数据备份和恢复

```python
import json
from datetime import datetime

class DataBackup:
    def __init__(self, db_handler):
        self.db_handler = db_handler

    def export_stocks(self, filename=None):
        """导出股票基础信息"""
        if filename is None:
            filename = f"stocks_backup_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"

        stocks = self.db_handler.mongo_find("panda", "stocks", {})

        with open(filename, 'w', encoding='utf-8') as f:
            json.dump(list(stocks), f, ensure_ascii=False, indent=2, default=str)

        logger.info(f"Exported {len(stocks)} stocks to {filename}")
        return filename

    def export_market_data(self, start_date, end_date, filename=None):
        """导出指定日期范围的行情数据"""
        if filename is None:
            filename = f"market_data_{start_date}_{end_date}.json"

        query = {
            'date': {'$gte': start_date, '$lte': end_date}
        }
        market_data = self.db_handler.mongo_find("panda", "stock_market", query)

        # 分批导出避免内存溢出
        batch_size = 10000
        with open(filename, 'w', encoding='utf-8') as f:
            f.write('[\n')
            batch = []
            for i, record in enumerate(market_data):
                batch.append(record)
                if len(batch) >= batch_size:
                    json.dump(batch, f, ensure_ascii=False, indent=2, default=str)
                    batch = []
                    if i < batch_size:  # 不是最后一批
                        f.write(',\n')

            if batch:  # 最后一批
                json.dump(batch, f, ensure_ascii=False, indent=2, default=str)
            f.write('\n]')

        logger.info(f"Exported market data to {filename}")
        return filename

    def restore_from_backup(self, backup_file, collection_name):
        """从备份文件恢复数据"""
        with open(backup_file, 'r', encoding='utf-8') as f:
            data = json.load(f)

        if isinstance(data, list):
            operations = []
            for record in data:
                # 根据集合类型构建查询条件
                if collection_name == 'stocks':
                    filter_query = {'symbol': record['symbol']}
                elif collection_name in ['stock_market', 'factor_base']:
                    filter_query = {
                        'date': record['date'],
                        'symbol': record['symbol']
                    }
                else:
                    filter_query = {'_id': record.get('_id')}

                operations.append(UpdateOne(
                    filter_query,
                    {'$set': record},
                    upsert=True
                ))

            if operations:
                result = self.db_handler.mongo_client["panda"][collection_name].bulk_write(operations)
                logger.info(f"Restored {result.upserted_count} records to {collection_name}")
                return result

        return None
```

## 监控和故障排除

### 1. 日志监控系统

```python
import logging
from datetime import datetime, timedelta

class DataUpdateMonitor:
    def __init__(self, db_handler):
        self.db_handler = db_handler
        self.logger = logging.getLogger("DataUpdateMonitor")

    def check_daily_update_status(self, target_date=None):
        """检查指定日期的数据更新状态"""
        if target_date is None:
            target_date = datetime.now().strftime("%Y%m%d")

        # 检查股票市场数据
        market_count = self.db_handler.mongo_count(
            "panda", "stock_market",
            {"date": target_date}
        )

        # 检查因子基础数据
        factor_count = self.db_handler.mongo_count(
            "panda", "factor_base",
            {"date": target_date}
        )

        # 检查股票总数
        total_stocks = self.db_handler.mongo_count("panda", "stocks", {"expired": False})

        status = {
            "date": target_date,
            "market_data_records": market_count,
            "factor_data_records": factor_count,
            "total_stocks": total_stocks,
            "market_coverage": market_count / total_stocks if total_stocks > 0 else 0,
            "factor_coverage": factor_count / total_stocks if total_stocks > 0 else 0,
            "status": "success" if market_count > 0 else "failed"
        }

        self.logger.info(f"Data update status for {target_date}: {status}")
        return status

    def check_data_quality(self, date):
        """检查数据质量"""
        market_data = list(self.db_handler.mongo_find(
            "panda", "stock_market",
            {"date": date}
        ))

        if not market_data:
            return {"status": "no_data"}

        df = pd.DataFrame(market_data)

        quality_report = {
            "date": date,
            "total_records": len(df),
            "null_values": df.isnull().sum().to_dict(),
            "negative_volume": (df['volume'] < 0).sum(),
            "invalid_prices": (
                (df['high'] < df['low']) |
                (df['close'] > df['high']) |
                (df['close'] < df['low'])
            ).sum(),
            "extreme_changes": (df['pct_chg'].abs() > 20).sum() if 'pct_chg' in df.columns else 0
        }

        return quality_report

    def generate_weekly_report(self):
        """生成周度数据更新报告"""
        end_date = datetime.now()
        start_date = end_date - timedelta(days=7)

        date_range = pd.date_range(start=start_date, end=end_date, freq='D')
        trading_days = []

        for date in date_range:
            date_str = date.strftime("%Y%m%d")
            # 这里需要实际的交易日历判断逻辑
            if self.is_trading_day(date_str):
                trading_days.append(date_str)

        reports = []
        for date in trading_days:
            status = self.check_daily_update_status(date)
            quality = self.check_data_quality(date)
            reports.append({
                "date": date,
                "update_status": status,
                "data_quality": quality
            })

        return reports

    def alert_on_failure(self, date):
        """数据更新失败告警"""
        status = self.check_daily_update_status(date)

        if status["status"] == "failed":
            # 发送告警邮件/短信/钉钉消息
            alert_message = f"""
            数据更新失败告警
            日期: {date}
            预期股票数: {status['total_stocks']}
            实际更新数: {status['market_data_records']}
            覆盖率: {status['market_coverage']:.2%}
            请及时检查数据更新服务！
            """

            # 这里可以集成实际的告警系统
            self.logger.error(alert_message)

            # 发送到监控系统
            self.send_to_monitoring_system(alert_message)
```

### 2. 性能监控

```python
import time
import psutil
from functools import wraps

class PerformanceMonitor:
    def __init__(self):
        self.logger = logging.getLogger("PerformanceMonitor")

    def monitor_data_cleaning_performance(self, func):
        """装饰器：监控数据清洗性能"""
        @wraps(func)
        def wrapper(*args, **kwargs):
            start_time = time.time()
            start_memory = psutil.Process().memory_info().rss / 1024 / 1024  # MB

            try:
                result = func(*args, **kwargs)

                end_time = time.time()
                end_memory = psutil.Process().memory_info().rss / 1024 / 1024  # MB

                performance_data = {
                    "function": func.__name__,
                    "execution_time": end_time - start_time,
                    "memory_usage": end_memory - start_memory,
                    "status": "success"
                }

                self.logger.info(f"Performance: {performance_data}")
                return result

            except Exception as e:
                end_time = time.time()

                performance_data = {
                    "function": func.__name__,
                    "execution_time": end_time - start_time,
                    "status": "failed",
                    "error": str(e)
                }

                self.logger.error(f"Performance: {performance_data}")
                raise

        return wrapper

    def monitor_database_performance(self, operation_name):
        """上下文管理器：监控数据库操作性能"""
        class DatabasePerformanceMonitor:
            def __init__(self, parent, operation_name):
                self.parent = parent
                self.operation_name = operation_name
                self.start_time = None

            def __enter__(self):
                self.start_time = time.time()
                return self

            def __exit__(self, exc_type, exc_val, exc_tb):
                end_time = time.time()
                execution_time = end_time - self.start_time

                performance_data = {
                    "operation": self.operation_name,
                    "execution_time": execution_time,
                    "status": "success" if exc_type is None else "failed"
                }

                if exc_type is not None:
                    performance_data["error"] = str(exc_val)

                self.parent.logger.info(f"DB Performance: {performance_data}")

        return DatabasePerformanceMonitor(self, operation_name)
```

### 3. 常见问题诊断

```python
class DataUpdateTroubleshooter:
    def __init__(self, config, db_handler):
        self.config = config
        self.db_handler = db_handler
        self.logger = logging.getLogger("Troubleshooter")

    def diagnose_update_failure(self, date):
        """诊断数据更新失败原因"""
        diagnosis = {
            "date": date,
            "issues": [],
            "recommendations": []
        }

        # 1. 检查交易日状态
        if not self.is_trading_day(date):
            diagnosis["issues"].append(f"{date} 不是交易日")
            diagnosis["recommendations"].append("跳过非交易日的数据更新")
            return diagnosis

        # 2. 检查API配置
        if self.config['DATAHUBSOURCE'] == 'tushare':
            if not self.config.get('TS_TOKEN'):
                diagnosis["issues"].append("Tushare Token 未配置")
                diagnosis["recommendations"].append("配置有效的 Tushare Token")

        # 3. 检查数据库连接
        try:
            self.db_handler.mongo_client.admin.command('ping')
        except Exception as e:
            diagnosis["issues"].append(f"数据库连接失败: {str(e)}")
            diagnosis["recommendations"].append("检查MongoDB服务是否正常运行")

        # 4. 检查数据源API限制
        if self.config['DATAHUBSOURCE'] == 'tushare':
            # 检查今日API调用次数
            api_calls_today = self.get_tushare_api_calls_today()
            if api_calls_today > 20000:  # 假设每日限制
                diagnosis["issues"].append(f"Tushare API调用次数过多: {api_calls_today}")
                diagnosis["recommendations"].append("等待API限制重置或升级账户")

        # 5. 检查数据完整性
        market_count = self.db_handler.mongo_count("panda", "stock_market", {"date": date})
        expected_count = self.db_handler.mongo_count("panda", "stocks", {"expired": False})

        if market_count < expected_count * 0.9:  # 覆盖率低于90%
            diagnosis["issues"].append(f"数据不完整: {market_count}/{expected_count}")
            diagnosis["recommendations"].append("重新执行数据清洗任务")

        return diagnosis

    def auto_fix_common_issues(self, date):
        """自动修复常见问题"""
        fixes_applied = []

        # 1. 修复重复数据
        duplicates = self.find_duplicate_records(date)
        if duplicates:
            self.remove_duplicates(duplicates)
            fixes_applied.append(f"删除了 {len(duplicates)} 条重复记录")

        # 2. 修复错误的价格关系
        invalid_prices = self.find_invalid_price_records(date)
        if invalid_prices:
            self.fix_invalid_prices(invalid_prices)
            fixes_applied.append(f"修复了 {len(invalid_prices)} 条价格异常记录")

        # 3. 补充缺失的股票名称
        missing_names = self.find_missing_stock_names(date)
        if missing_names:
            self.supplement_stock_names(missing_names)
            fixes_applied.append(f"补充了 {len(missing_names)} 只股票的名称")

        return fixes_applied

    def find_duplicate_records(self, date):
        """查找重复记录"""
        pipeline = [
            {"$match": {"date": date}},
            {"$group": {
                "_id": {"date": "$date", "symbol": "$symbol"},
                "count": {"$sum": 1},
                "docs": {"$push": "$_id"}
            }},
            {"$match": {"count": {"$gt": 1}}}
        ]

        duplicates = list(self.db_handler.mongo_client["panda"]["stock_market"].aggregate(pipeline))
        return duplicates

    def find_invalid_price_records(self, date):
        """查找价格关系异常的记录"""
        query = {
            "date": date,
            "$or": [
                {"high": {"$lt": {"$ifNull": ["$low", 0]}}},
                {"close": {"$gt": {"$ifNull": ["$high", float('inf')]}}},
                {"close": {"$lt": {"$ifNull": ["$low", 0]}}},
                {"open": {"$gt": {"$ifNull": ["$high", float('inf')]}}},
                {"open": {"$lt": {"$ifNull": ["$low", 0]}}}
            ]
        }

        return list(self.db_handler.mongo_find("panda", "stock_market", query))
```

## 最佳实践

### 1. 数据更新策略

#### 增量更新策略

```python
class IncrementalUpdateStrategy:
    """增量更新策略，只更新缺失或错误的数据"""

    def __init__(self, config, db_handler):
        self.config = config
        self.db_handler = db_handler

    def get_missing_dates(self, start_date, end_date):
        """获取缺失数据的日期"""
        date_range = pd.date_range(start=start_date, end=end_date, freq='D')
        existing_dates = set()

        # 从数据库查询已存在的日期
        pipeline = [
            {"$match": {"date": {"$gte": start_date, "$lte": end_date}}},
            {"$group": {"_id": "$date", "count": {"$sum": 1}}},
            {"$sort": {"_id": 1}}
        ]

        results = list(self.db_handler.mongo_client["panda"]["stock_market"].aggregate(pipeline))
        expected_daily_count = 4000  # 预期每日股票数量

        for result in results:
            if result["count"] >= expected_daily_count * 0.9:  # 覆盖率超过90%
                existing_dates.add(result["_id"])

        missing_dates = []
        for date in date_range:
            date_str = date.strftime("%Y%m%d")
            if date_str not in existing_dates and self.is_trading_day(date_str):
                missing_dates.append(date_str)

        return missing_dates

    def smart_update(self, start_date, end_date):
        """智能更新：只更新缺失的数据"""
        missing_dates = self.get_missing_dates(start_date, end_date)

        if not missing_dates:
            logger.info("所有数据都是最新的，无需更新")
            return

        logger.info(f"发现 {len(missing_dates)} 个交易日数据缺失，开始增量更新")

        for date in missing_dates:
            try:
                self.update_single_date(date)
                logger.info(f"成功更新 {date} 的数据")
            except Exception as e:
                logger.error(f"更新 {date} 数据失败: {str(e)}")
                continue
```

#### 数据验证策略

```python
class DataValidationStrategy:
    """数据验证策略"""

    def __init__(self, db_handler):
        self.db_handler = db_handler

    def validate_batch_data(self, data_batch):
        """批量数据验证"""
        validation_results = {
            "total_records": len(data_batch),
            "valid_records": 0,
            "invalid_records": 0,
            "warnings": [],
            "errors": []
        }

        valid_records = []

        for record in data_batch:
            record_valid = True
            record_warnings = []

            # 基础字段检查
            required_fields = ['date', 'symbol', 'open', 'high', 'low', 'close', 'volume']
            for field in required_fields:
                if field not in record or record[field] is None:
                    validation_results["errors"].append(f"记录 {record.get('symbol', 'unknown')} 缺少必需字段: {field}")
                    record_valid = False
                    break

            if not record_valid:
                validation_results["invalid_records"] += 1
                continue

            # 价格合理性检查
            if record['high'] < record['low']:
                validation_results["errors"].append(f"{record['symbol']} 最高价低于最低价")
                record_valid = False

            if not (record['low'] <= record['close'] <= record['high']):
                validation_results["errors"].append(f"{record['symbol']} 收盘价超出当日价格区间")
                record_valid = False

            if not (record['low'] <= record['open'] <= record['high']):
                validation_results["errors"].append(f"{record['symbol']} 开盘价超出当日价格区间")
                record_valid = False

            # 成交量检查
            if record['volume'] < 0:
                validation_results["warnings"].append(f"{record['symbol']} 成交量为负数")
                record['volume'] = 0  # 自动修正
                record_warnings.append("成交量已修正为0")

            # 涨跌幅检查
            if 'pre_close' in record and record['pre_close'] > 0:
                pct_change = (record['close'] - record['pre_close']) / record['pre_close'] * 100
                if abs(pct_change) > 20:  # 假设涨跌幅限制为20%
                    validation_results["warnings"].append(f"{record['symbol']} 涨跌幅异常: {pct_change:.2f}%")

            if record_valid:
                valid_records.append(record)
                validation_results["valid_records"] += 1

                if record_warnings:
                    validation_results["warnings"].extend(record_warnings)
            else:
                validation_results["invalid_records"] += 1

        return valid_records, validation_results
```

### 2. 性能优化策略

#### 数据库操作优化

```python
class DatabaseOptimization:
    """数据库操作优化"""

    def __init__(self, db_handler):
        self.db_handler = db_handler

    def optimize_bulk_operations(self, data, collection_name, batch_size=1000):
        """优化批量操作"""
        from pymongo import UpdateOne

        collection = self.db_handler.mongo_client["panda"][collection_name]
        total_records = len(data)
        processed = 0

        while processed < total_records:
            batch = data[processed:processed + batch_size]

            operations = []
            for record in batch:
                if collection_name == 'stock_market':
                    filter_query = {'date': record['date'], 'symbol': record['symbol']}
                elif collection_name == 'factors':
                    filter_query = {
                        'date': record['date'],
                        'symbol': record['symbol'],
                        'factor_name': record['factor_name']
                    }
                else:
                    filter_query = {'_id': record.get('_id')}

                operations.append(UpdateOne(
                    filter_query,
                    {'$set': record},
                    upsert=True
                ))

            if operations:
                try:
                    result = collection.bulk_write(operations, ordered=False)
                    processed += len(batch)
                    logger.info(f"Processed {processed}/{total_records} records in {collection_name}")

                    # 添加延迟避免过载
                    time.sleep(0.1)

                except Exception as e:
                    logger.error(f"Bulk operation failed: {str(e)}")
                    # 可以选择重试或记录失败记录
                    break

    def create_optimal_indexes(self):
        """创建优化索引"""
        db = self.db_handler.mongo_client["panda"]

        # stock_market集合索引
        db.stock_market.create_index([("date", 1), ("symbol", 1)], unique=True)
        db.stock_market.create_index([("symbol", 1), ("date", 1)])
        db.stock_market.create_index([("date", 1)])
        db.stock_market.create_index([("symbol", 1)])

        # factors集合索引
        db.factors.create_index([("date", 1), ("symbol", 1), ("factor_name", 1)], unique=True)
        db.factors.create_index([("factor_name", 1), ("date", 1)])
        db.factors.create_index([("user_id", 1), ("factor_name", 1)])

        logger.info("Created optimal indexes for better query performance")
```

#### 内存优化策略

```python
class MemoryOptimization:
    """内存使用优化"""

    def process_large_dataset(self, data_generator, batch_processor):
        """处理大数据集的内存优化方法"""
        batch_size = 1000
        batch = []

        for record in data_generator:
            batch.append(record)

            if len(batch) >= batch_size:
                # 处理当前批次
                batch_processor(batch)

                # 清理内存
                batch.clear()

                # 强制垃圾回收
                import gc
                gc.collect()

        # 处理最后一批
        if batch:
            batch_processor(batch)

    def optimize_dataframe_memory(self, df):
        """优化DataFrame内存使用"""
        original_memory = df.memory_usage(deep=True).sum() / 1024 / 1024  # MB

        # 优化数值类型
        for col in df.select_dtypes(include=['int64']).columns:
            df[col] = pd.to_numeric(df[col], downcast='integer')

        for col in df.select_dtypes(include=['float64']).columns:
            df[col] = pd.to_numeric(df[col], downcast='float')

        # 优化字符串类型
        for col in df.select_dtypes(include=['object']).columns:
            if df[col].dtype == 'object':
                df[col] = df[col].astype('category')

        optimized_memory = df.memory_usage(deep=True).sum() / 1024 / 1024  # MB
        reduction = (original_memory - optimized_memory) / original_memory * 100

        logger.info(f"Memory optimized: {original_memory:.2f}MB -> {optimized_memory:.2f}MB ({reduction:.1f}% reduction)")

        return df
```

### 3. 容错和恢复策略

#### 自动重试机制

```python
import time
from functools import wraps
from random import uniform

class RetryMechanism:
    """自动重试机制"""

    @staticmethod
    def retry_on_failure(max_retries=3, base_delay=1, max_delay=60, backoff_factor=2):
        """重试装饰器"""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                last_exception = None

                for attempt in range(max_retries + 1):
                    try:
                        return func(*args, **kwargs)

                    except Exception as e:
                        last_exception = e

                        if attempt == max_retries:
                            logger.error(f"Function {func.__name__} failed after {max_retries} retries: {str(e)}")
                            raise

                        # 计算延迟时间（指数退避 + 随机抖动）
                        delay = min(base_delay * (backoff_factor ** attempt) + uniform(0, 1), max_delay)
                        logger.warning(f"Function {func.__name__} failed (attempt {attempt + 1}/{max_retries + 1}), retrying in {delay:.2f}s: {str(e)}")
                        time.sleep(delay)

                raise last_exception

            return wrapper
        return decorator

# 使用示例
@RetryMechanism.retry_on_failure(max_retries=3, base_delay=2)
def fetch_market_data(date, data_source):
    """获取市场数据（带重试机制）"""
    # 实际的数据获取逻辑
    pass
```

#### 数据恢复机制

```python
class DataRecovery:
    """数据恢复机制"""

    def __init__(self, config, db_handler):
        self.config = config
        self.db_handler = db_handler

    def create_recovery_point(self, date):
        """创建恢复点"""
        recovery_point = {
            "date": date,
            "timestamp": datetime.now(),
            "market_data_count": self.db_handler.mongo_count("panda", "stock_market", {"date": date}),
            "factor_data_count": self.db_handler.mongo_count("panda", "factor_base", {"date": date}),
            "custom_factors_count": self.db_handler.mongo_count("panda", "factors", {"date": date})
        }

        # 保存恢复点信息
        self.db_handler.mongo_insert("panda", "recovery_points", recovery_point)

        logger.info(f"Created recovery point for {date}")
        return recovery_point

    def rollback_to_recovery_point(self, date):
        """回滚到恢复点"""
        recovery_point = self.db_handler.mongo_find_one("panda", "recovery_points", {"date": date})

        if not recovery_point:
            raise ValueError(f"No recovery point found for {date}")

        # 删除指定日期的所有数据（通常用于数据损坏时重新处理）
        delete_result = self.db_handler.mongo_delete_many("panda", "stock_market", {"date": date})
        delete_result += self.db_handler.mongo_delete_many("panda", "factor_base", {"date": date})
        delete_result += self.db_handler.mongo_delete_many("panda", "factors", {"date": date})

        logger.info(f"Rollback completed for {date}, deleted {delete_result} records")

        return recovery_point

    def detect_data_corruption(self, date):
        """检测数据损坏"""
        corruption_indicators = []

        # 检查数据量异常
        market_count = self.db_handler.mongo_count("panda", "stock_market", {"date": date})
        expected_count = 4000  # 预期股票数量

        if market_count < expected_count * 0.5:  # 数据量过少
            corruption_indicators.append(f"Insufficient market data: {market_count}/{expected_count}")

        if market_count > expected_count * 1.5:  # 数据量过多（可能有重复）
            corruption_indicators.append(f"Excessive market data: {market_count}/{expected_count}")

        # 检查价格异常
        invalid_prices = self.db_handler.mongo_count("panda", "stock_market", {
            "date": date,
            "$or": [
                {"high": {"$lt": {"$ifNull": ["$low", 0]}}},
                {"$expr": {"$gt": ["$close", "$high"]}},
                {"$expr": {"$lt": ["$close", "$low"]}}
            ]
        })

        if invalid_prices > 0:
            corruption_indicators.append(f"Found {invalid_prices} records with invalid price relationships")

        return corruption_indicators
```

### 4. 安全策略

#### API访问控制

```python
import hashlib
import secrets
from datetime import datetime, timedelta

class APIAccessControl:
    """API访问控制"""

    def __init__(self, db_handler):
        self.db_handler = db_handler

    def generate_api_key(self, user_id, permissions=None):
        """生成API密钥"""
        if permissions is None:
            permissions = ["read_data", "write_factor"]

        api_key = secrets.token_urlsafe(32)
        api_secret = secrets.token_urlsafe(32)

        key_record = {
            "user_id": user_id,
            "api_key": api_key,
            "api_secret": api_secret,
            "permissions": permissions,
            "created_at": datetime.now(),
            "expires_at": datetime.now() + timedelta(days=365),
            "is_active": True,
            "rate_limit": 1000  # 每小时请求限制
        }

        self.db_handler.mongo_insert("panda", "api_keys", key_record)

        return {
            "api_key": api_key,
            "api_secret": api_secret,
            "permissions": permissions
        }

    def validate_api_request(self, api_key, api_secret, required_permission):
        """验证API请求"""
        key_record = self.db_handler.mongo_find_one("panda", "api_keys", {
            "api_key": api_key,
            "is_active": True
        })

        if not key_record:
            return False, "Invalid API key"

        # 验证密钥
        if key_record["api_secret"] != api_secret:
            return False, "Invalid API secret"

        # 检查权限
        if required_permission not in key_record["permissions"]:
            return False, f"Permission denied: {required_permission} required"

        # 检查过期时间
        if datetime.now() > key_record["expires_at"]:
            return False, "API key expired"

        # 检查速率限制
        current_hour = datetime.now().replace(minute=0, second=0, microsecond=0)
        request_count = self.db_handler.mongo_count("panda", "api_requests", {
            "api_key": api_key,
            "timestamp": {"$gte": current_hour}
        })

        if request_count >= key_record["rate_limit"]:
            return False, "Rate limit exceeded"

        # 记录请求
        self.db_handler.mongo_insert("panda", "api_requests", {
            "api_key": api_key,
            "permission": required_permission,
            "timestamp": datetime.now()
        })

        return True, "Access granted"
```

#### 数据备份策略

```python
import schedule
import threading

class BackupStrategy:
    """数据备份策略"""

    def __init__(self, db_handler, backup_path="./backups"):
        self.db_handler = db_handler
        self.backup_path = backup_path
        self.backup_thread = None

    def daily_backup(self):
        """每日备份策略"""
        today = datetime.now().strftime("%Y%m%d")

        # 备份关键数据
        collections_to_backup = ["stocks", "stock_market", "factor_base", "factors"]

        for collection in collections_to_backup:
            backup_file = f"{self.backup_path}/{collection}_{today}.json"
            self.export_collection_to_json(collection, backup_file)

        # 清理旧备份（保留30天）
        self.cleanup_old_backups(days_to_keep=30)

        logger.info(f"Daily backup completed for {today}")

    def export_collection_to_json(self, collection_name, filename):
        """导出集合到JSON文件"""
        try:
            # 使用mongodump的Python实现
            data = list(self.db_handler.mongo_find("panda", collection_name, {}))

            with open(filename, 'w', encoding='utf-8') as f:
                json.dump(data, f, ensure_ascii=False, indent=2, default=str)

            logger.info(f"Exported {len(data)} records from {collection_name} to {filename}")

        except Exception as e:
            logger.error(f"Failed to export {collection_name}: {str(e)}")

    def schedule_backup_tasks(self):
        """调度备份任务"""
        # 每日凌晨2点执行备份
        schedule.every().day.at("02:00").do(self.daily_backup)

        # 启动调度线程
        def run_scheduler():
            while True:
                schedule.run_pending()
                time.sleep(60)

        self.backup_thread = threading.Thread(target=run_scheduler, daemon=True)
        self.backup_thread.start()

        logger.info("Backup scheduler started")
```

---

通过本指南，您应该能够：

1. **理解数据更新架构**: 掌握PandaFactor的数据更新机制和组件
2. **配置数据源**: 正确配置各种数据源和API密钥
3. **设置自动更新**: 配置定时任务和调度器
4. **执行手动更新**: 使用API或命令行手动更新数据
5. **监控数据质量**: 实施监控和故障排除机制
6. **优化性能**: 应用性能优化和最佳实践
7. **确保数据安全**: 实施备份和恢复策略

这套数据更新机制确保了PandaFactor平台的数据时效性、准确性和可靠性，为量化因子研究提供了坚实的数据基础。