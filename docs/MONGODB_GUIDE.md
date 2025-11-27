# PandaFactor MongoDB 使用完全指南

## 目录
1. [MongoDB架构概述](#mongodb架构概述)
2. [数据库连接和配置](#数据库连接和配置)
3. [数据模型设计](#数据模型设计)
4. [索引优化策略](#索引优化策略)
5. [查询性能优化](#查询性能优化)
6. [数据访问模式](#数据访问模式)
7. [监控和维护](#监控和维护)
8. [生产环境最佳实践](#生产环境最佳实践)
9. [故障排除和调优](#故障排除和调优)

## MongoDB架构概述

### 系统架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    PandaFactor应用层                        │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   数据获取    │  │   因子计算    │  │   Web API    │    │
│  │   模块       │  │   模块       │  │   模块       │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
├─────────────────────────────────────────────────────────────┤
│                 DatabaseHandler (单例模式)                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              连接池管理                            │    │
│  │  - readPreference: secondaryPreferred             │    │
│  │  - writeConcern: majority                        │    │
│  │  - retryWrites: true                             │    │
│  │  - socketTimeoutMS: 30000                        │    │
│  │  - connectTimeoutMS: 20000                       │    │
│  │  - serverSelectionTimeoutMS: 30000                 │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│                    MongoDB集群                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Primary    │  │  Secondary 1 │  │  Secondary 2 │    │
│  │   (主节点)    │  │   (从节点)    │  │   (从节点)    │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 核心组件

1. **DatabaseHandler**: 单例模式的数据库连接管理器
2. **连接池**: 复用连接，提高性能
3. **读写分离**: 优先从从节点读取数据
4. **故障转移**: 自动重试和节点选择
5. **数据分片**: 支持水平扩展（规划中）

### 部署模式

```python
# 单节点模式 (适合开发环境)
MONGO_TYPE: "single"
MONGO_URI: "127.0.0.1:27017"

# 副本集模式 (适合生产环境)
MONGO_TYPE: "replica_set"
MONGO_URI: "127.0.0.1:27017"
MONGO_REPLICA_SET: "rs0"

# 分片模式 (未来扩展)
MONGO_TYPE: "sharded"  # 规划中
```

## 数据库连接和配置

### 1. 配置文件详解

```yaml
# panda_common/config.yaml
# 数据库连接配置
MONGO_USER: "panda"
MONGO_PASSWORD: "panda"
MONGO_URI: "127.0.0.1:27017"
MONGO_AUTH_DB: "admin"        # 认证数据库
MONGO_DB: "panda"             # 业务数据库

# 部署模式配置
MONGO_TYPE: "replica_set"      # single/replica_set/sharded
MONGO_REPLICA_SET: "rs0"      # 副本集名称

# 连接参数 (可在DatabaseHandler中看到完整配置)
readPreference: 'secondaryPreferred'  # 读取偏好
w: 'majority'                     # 写入确认级别
retryWrites: true                   # 自动重试写入
socketTimeoutMS: 30000             # Socket超时
connectTimeoutMS: 20000            # 连接超时
serverSelectionTimeoutMS: 30000     # 服务器选择超时
authSource: "admin"                # 认证源
```

### 2. DatabaseHandler 设计模式

```python
# panda_common/handlers/database_handler.py
import pymongo
import urllib.parse
from typing import Optional, Dict, List

class DatabaseHandler:
    """
    单例模式的数据库连接处理器

    特性:
    - 单例模式确保连接唯一性
    - URL编码处理特殊字符
    - 支持多种部署模式
    - 自动故障重试
    - 连接池管理
    """

    _instance = None  # 单例实例

    def __new__(cls, *args, **kwargs):
        """实现单例模式"""
        if not cls._instance:
            cls._instance = super(DatabaseHandler, cls).__new__(cls)
        return cls._instance

    def __init__(self, config):
        """初始化数据库连接"""
        if not hasattr(self, 'initialized'):  # 防止重复初始化

            # URL编码密码（处理特殊字符）
            encoded_password = urllib.parse.quote_plus(config["MONGO_PASSWORD"])

            # 构建连接字符串
            MONGO_URI = f'mongodb://{config["MONGO_USER"]}:{encoded_password}@{config["MONGO_URI"]}/{config["MONGO_AUTH_DB"]}'

            # 根据部署类型配置连接
            if config['MONGO_TYPE'] == 'single':
                self.mongo_client = pymongo.MongoClient(
                    MONGO_URI,
                    readPreference='secondaryPreferred',
                    w='majority',
                    retryWrites=True,
                    socketTimeoutMS=30000,
                    connectTimeoutMS=20000,
                    serverSelectionTimeoutMS=30000,
                    authSource=config["MONGO_AUTH_DB"],
                )
            elif config['MONGO_TYPE'] == 'replica_set':
                MONGO_URI += f'?replicaSet={config["MONGO_REPLICA_SET"]}'
                self.mongo_client = pymongo.MongoClient(
                    MONGO_URI,
                    readPreference='secondaryPreferred',
                    w='majority',
                    retryWrites=True,
                    socketTimeoutMS=30000,
                    connectTimeoutMS=20000,
                    serverSelectionTimeoutMS=30000,
                    authSource=config["MONGO_AUTH_DB"],
                )

            # 测试连接
            try:
                self.mongo_client.admin.command('ping')
                print(f"Successfully connected to MongoDB: {self._mask_uri(MONGO_URI)}")
            except Exception as e:
                print(f"MongoDB connection failed: {e}")
                raise

            self.initialized = True

    def _mask_uri(self, uri: str) -> str:
        """隐藏连接字符串中的密码"""
        import urllib.parse
        parsed = urllib.parse.urlparse(uri)
        masked_netloc = parsed.netloc.replace(urllib.parse.quote_plus(
            self.config.get("MONGO_PASSWORD", "")), "****")
        masked_uri = uri.replace(parsed.netloc, masked_netloc)
        return masked_uri
```

### 3. 环境变量配置

```python
# 支持通过环境变量覆盖配置
import os

# 生产环境环境变量
os.environ['MONGO_USER'] = 'production_user'
os.environ['MONGO_PASSWORD'] = 'secure_password_123!@#'
os.environ['MONGO_URI'] = 'mongo-cluster.mongodb.com:27017'
os.environ['MONGO_TYPE'] = 'replica_set'
os.environ['MONGO_REPLICA_SET'] = 'prod-rs'

# 配置加载优先级：环境变量 > 配置文件
def load_config():
    config = load_from_yaml()  # 从YAML文件加载
    for key, env_value in os.environ.items():
        if key in config:
            # 类型转换
            original_type = type(config[key])
            if original_type == bool:
                config[key] = env_value.lower() in ('true', '1', 'yes')
            else:
                config[key] = original_type(env_value)
    return config
```

### 4. 连接池和性能调优

```python
# 连接池配置优化
class OptimizedDatabaseHandler(DatabaseHandler):
    """优化的数据库连接处理器"""

    def __init__(self, config):
        # 连接池配置
        max_pool_size = config.get('MONGO_MAX_POOL_SIZE', 100)      # 最大连接数
        min_pool_size = config.get('MONGO_MIN_POOL_SIZE', 10)       # 最小连接数
        max_idle_time_ms = config.get('MONGO_MAX_IDLE_TIME', 30000)  # 最大空闲时间

        # 读取偏好配置
        read_preference = config.get('MONGO_READ_PREFERENCE', 'secondaryPreferred')

        # 写入关注配置
        write_concern = config.get('MONGO_WRITE_CONCERN', 'majority')

        # 重试配置
        retry_reads = config.get('MONGO_RETRY_READS', True)
        retry_writes = config.get('MONGO_RETRY_WRITES', True)

        # 连接字符串构建
        connection_params = {
            'maxPoolSize': max_pool_size,
            'minPoolSize': min_pool_size,
            'maxIdleTimeMS': max_idle_time_ms,
            'readPreference': read_preference,
            'w': write_concern,
            'retryReads': retry_reads,
            'retryWrites': retry_writes,
            'socketTimeoutMS': 30000,
            'connectTimeoutMS': 20000,
            'serverSelectionTimeoutMS': 30000,
            'heartbeatFrequencyMS': 10000,      # 心跳频率
            'appName': 'panda_factor',          # 应用标识
        }

        if config.get('MONGO_TYPE') == 'replica_set':
            connection_params['replicaSet'] = config['MONGO_REPLICA_SET']

        self.mongo_client = pymongo.MongoClient(MONGO_URI, **connection_params)
```

## 数据模型设计

### 1. 核心集合设计

#### stocks集合 - 股票基础信息

```javascript
{
  "_id": ObjectId("6607a8f9d5a8b2c3d4e5f6a"),
  "symbol": "000001.SZ",           // 标准化股票代码
  "name": "平安银行",             // 股票名称
  "industry": "银行",              // 行业 (可选)
  "market": "深交所主板",          // 交易所 (可选)
  "list_date": "19910403",         // 上市日期 (可选)
  "expired": false,                // 是否退市
  "created_at": ISODate("2024-03-27T10:00:00Z"),
  "updated_at": ISODate("2024-03-27T10:00:00Z")
}

// 索引设计
db.stocks.createIndex({"symbol": 1}, {unique: true, name: "symbol_unique"})
db.stocks.createIndex({"expired": 1}, {name: "expired_idx"})
db.stocks.createIndex({"created_at": 1}, {name: "created_at_idx"})
```

#### stock_market集合 - 日线行情数据

```javascript
{
  "_id": ObjectId("6607a8f9d5a8b2c3d4e5f6b"),
  "date": "20240327",              // 交易日期 YYYYMMDD
  "symbol": "000001.SZ",          // 股票代码
  "open": 12.50,                  // 开盘价
  "high": 12.80,                  // 最高价
  "low": 12.40,                   // 最低价
  "close": 12.75,                 // 收盘价
  "volume": 15000000,              // 成交量(股)
  "amount": 191250000,             // 成交额(元)
  "pre_close": 12.60,             // 昨收价
  "change": 0.15,                 // 涨跌额
  "pct_chg": 1.19,                // 涨跌幅(%)
  "limit_up": 13.86,               // 涨停价
  "limit_down": 11.34,             // 跌停价
  "index_component": "hs300",       // 指数成分股标识
  "turnover": 0.025,              // 换手率 (可选)
  "pe_ratio": 8.5,                // 市盈率 (可选)
  "pb_ratio": 0.8,                // 市净率 (可选)
  "market_cap": 247500000000,      // 总市值(元) (可选)
  "created_at": ISODate("2024-03-27T15:00:00Z"),
  "updated_at": ISODate("2024-03-27T15:00:00Z")
}

// 索引设计 (核心查询模式)
db.stock_market.createIndex(
  {"symbol": 1, "date": 1},
  {unique: true, name: "symbol_date_unique"}
)

// 常用查询索引
db.stock_market.createIndex({"date": 1}, {name: "date_idx"})
db.stock_market.createIndex({"symbol": 1}, {name: "symbol_idx"})
db.stock_market.createIndex(
  {"index_component": 1, "date": 1},
  {name: "index_component_date_idx"}
)

// 部分索引 (只索引活跃股票)
db.stock_market.createIndex(
  {"symbol": 1, "date": 1},
  {
    name: "active_stocks_idx",
    partialFilterExpression: { "date": { "$gte": "20240101" } }
  }
)
```

#### factor_base集合 - 基础因子数据

```javascript
{
  "_id": ObjectId("6607a8f9d5a8b2c3d4e5f6c"),
  "date": "20240327",              // 交易日期
  "symbol": "000001.SZ",          // 股票代码
  "open": 12.50,                  // 开盘价
  "high": 12.80,                  // 最高价
  "low": 12.40,                   // 最低价
  "close": 12.75,                 // 收盘价
  "volume": 15000000,              // 成交量
  "market_cap": 247500000000,     // 总市值
  "turnover": 0.025,              // 换手率
  "amount": 191250000,             // 成交额
  "created_at": ISODate("2024-03-27T15:00:00Z"),
  "updated_at": ISODate("2024-03-27T15:00:00Z")
}

// 索引设计
db.factor_base.createIndex(
  {"date": 1, "symbol": 1},
  {unique: true, name: "date_symbol_unique"}
)
db.factor_base.createIndex({"symbol": 1, "date": 1}, {name: "symbol_date_idx"})
```

#### factors集合 - 计算因子数据

```javascript
{
  "_id": ObjectId("6607a8f9d5a8b2c3d4e5f6d"),
  "date": "20240327",              // 交易日期
  "symbol": "000001.SZ",          // 股票代码
  "factor_name": "momentum_20d",   // 因子名称
  "factor_value": 0.125,           // 因子值
  "user_id": 12345,                // 用户ID (自定义因子)
  "factor_type": "technical",      // 因子类型 (可选)
  "calculation_date": ISODate("2024-03-27T15:00:00Z"), // 计算时间
  "created_at": ISODate("2024-03-27T15:00:00Z"),
  "updated_at": ISODate("2024-03-27T15:00:00Z")
}

// 复合唯一索引
db.factors.createIndex(
  {"date": 1, "symbol": 1, "factor_name": 1},
  {unique: true, name: "date_symbol_factor_unique"}
)

// 用户因子查询索引
db.factors.createIndex(
  {"user_id": 1, "factor_name": 1, "date": -1},
  {name: "user_factor_date_idx"}
)

// 因子性能分析索引
db.factors.createIndex(
  {"factor_name": 1, "date": -1},
  {name: "factor_name_date_idx"}
)
```

### 2. 数据分区策略

#### 按日期分区 (推荐用于历史数据)

```python
# 历史数据按年份分区
YEAR_PARTITIONS = [
    "stock_market_2020",
    "stock_market_2021",
    "stock_market_2022",
    "stock_market_2023",
    "stock_market_2024"
]

# 当前年份数据存储在主集合
CURRENT_YEAR_COLLECTION = "stock_market"

def get_collection_name(date: str) -> str:
    """根据日期获取集合名称"""
    year = int(date[:4])
    if year == datetime.now().year:
        return CURRENT_YEAR_COLLECTION
    return f"stock_market_{year}"

# 查询示例
def query_market_data(symbol: str, start_date: str, end_date: str):
    """跨分区查询"""
    # 确定涉及的年份
    years = range(int(start_date[:4]), int(end_date[:4]) + 1)

    results = []
    for year in years:
        collection_name = get_collection_name(f"{year}0101")
        collection = db[collection_name]

        # 查询该年份的数据
        year_data = list(collection.find({
            "symbol": symbol,
            "date": {"$gte": start_date, "$lte": end_date}
        }))
        results.extend(year_data)

    return results
```

#### 按股票代码分区 (适用于超大集合)

```python
# 按股票代码首字母分区
SYMBOL_PARTITIONS = {
    "stock_market_0": ["0*", "1*", "2*", "3*"],
    "stock_market_3": ["3*"],  # 300xxx创业板单独分区
    "stock_market_6": ["6*"],  # 600xxx主板单独分区
    "stock_market_other": ["*"]  # 其他股票
}

def get_partition_collection(symbol: str) -> str:
    """根据股票代码获取分区集合"""
    first_char = symbol[0]

    for partition, patterns in SYMBOL_PARTITIONS.items():
        for pattern in patterns:
            if pattern.endswith('*'):
                if first_char == pattern[0]:
                    return partition
            elif pattern == '*':
                return partition

    return "stock_market_other"
```

### 3. 数据建模最佳实践

#### 文档设计原则

```python
# 好的实践：扁平化结构，避免深层嵌套
class MarketDataSchema:
    """
    设计原则：
    1. 扁平化结构，避免深层嵌套
    2. 合理的数据类型选择
    3. 预留扩展字段
    4. 统一的时间戳字段
    5. 适当的字段命名约定
    """

    # 核心字段 (必需)
    REQUIRED_FIELDS = {
        "date": str,           # 交易日期 YYYYMMDD
        "symbol": str,         # 股票代码
        "open": float,         # 开盘价
        "high": float,         # 最高价
        "low": float,          # 最低价
        "close": float,        # 收盘价
        "volume": int,         # 成交量
    }

    # 扩展字段 (可选)
    OPTIONAL_FIELDS = {
        "amount": float,       # 成交额
        "pre_close": float,    # 昨收价
        "change": float,       # 涨跌额
        "pct_chg": float,      # 涨跌幅
        "turnover": float,     # 换手率
        "pe_ratio": float,     # 市盈率
        "pb_ratio": float,     # 市净率
        "market_cap": int,     # 总市值
        "limit_up": float,     # 涨停价
        "limit_down": float,    # 跌停价
        "index_component": str, # 指数成分股
    }

    # 元数据字段
    METADATA_FIELDS = {
        "created_at": datetime,
        "updated_at": datetime,
        "data_source": str,    # 数据来源
        "version": int,        # 数据版本
    }

# 避免的反模式
class AntiPatternSchema:
    """
    避免以下设计模式：
    1. 深层嵌套结构
    2. 数组存储大量数据
    3. 动态字段名
    4. 冗余数据存储
    """

    # ❌ 错误：深层嵌套
    BAD_NESTED = {
        "price_data": {
            "open": {"value": 12.50, "currency": "CNY"},
            "high": {"value": 12.80, "currency": "CNY"},
            # ...
        }
    }

    # ✅ 正确：扁平化
    GOOD_FLAT = {
        "open": 12.50,
        "high": 12.80,
        # ...
    }

    # ❌ 错误：动态字段名
    BAD_DYNAMIC = {
        "prices": {
            "20240327": 12.75,
            "20240326": 12.60,
            "20240325": 12.45,
        }
    }

    # ✅ 正确：结构化数组
    GOOD_STRUCTURED = [
        {"date": "20240327", "close": 12.75},
        {"date": "20240326", "close": 12.60},
        {"date": "20240325", "close": 12.45}
    ]
```

## 索引优化策略

### 1. 索引设计原则

#### 基于查询模式的索引设计

```python
class IndexDesignStrategy:
    """
    索引设计策略：
    1. 基于实际查询模式
    2. 复合索引字段顺序优化
    3. 使用部分索引减少索引大小
    4. 考虑索引选择性
    5. 平衡读写性能
    """

    # 核心查询模式分析
    QUERY_PATTERNS = {
        # 模式1: 按股票代码和日期范围查询 (最常见)
        "stock_date_range": {
            "query": {"symbol": "000001.SZ", "date": {"$gte": "20240101", "$lte": "20240327"}},
            "frequency": "very_high",
            "selectivity": "high",
            "recommended_index": {"symbol": 1, "date": 1}
        },

        # 模式2: 按日期范围查询多只股票
        "date_symbol_list": {
            "query": {"date": {"$gte": "20240101", "$lte": "20240327"}, "symbol": {"$in": ["000001.SZ", "000002.SZ"]}},
            "frequency": "high",
            "selectivity": "medium",
            "recommended_index": {"date": 1, "symbol": 1}
        },

        # 模式3: 按指数成分股查询
        "index_component": {
            "query": {"index_component": "hs300", "date": {"$gte": "20240101"}},
            "frequency": "medium",
            "selectivity": "low",
            "recommended_index": {"index_component": 1, "date": 1}
        },

        # 模式4: 因子数据查询
        "factor_query": {
            "query": {"factor_name": "momentum_20d", "date": {"$gte": "20240101"}},
            "frequency": "high",
            "selectivity": "medium",
            "recommended_index": {"factor_name": 1, "date": 1, "symbol": 1}
        }
    }

    @staticmethod
    def create_optimized_indexes(db):
        """创建优化索引"""
        # stock_market集合索引
        stock_market = db["stock_market"]

        # 1. 主要索引：symbol + date (支持最常见查询)
        stock_market.create_index(
            [("symbol", 1), ("date", 1)],
            name="symbol_date_idx",
            unique=True,
            background=True
        )

        # 2. 辅助索引：date + symbol (支持日期范围查询)
        stock_market.create_index(
            [("date", 1), ("symbol", 1)],
            name="date_symbol_idx",
            background=True
        )

        # 3. 单字段索引：date (支持纯日期查询)
        stock_market.create_index(
            [("date", 1)],
            name="date_idx",
            background=True
        )

        # 4. 部分索引：只索引当年数据 (减少索引大小)
        current_year = datetime.now().strftime("%Y")
        stock_market.create_index(
            [("symbol", 1), ("date", 1)],
            name="current_year_idx",
            partialFilterExpression={"date": {"$regex": f"^{current_year}"}},
            background=True
        )

        # 5. 稀疏索引：只索引有换手率的数据
        stock_market.create_index(
            [("turnover", 1), ("date", -1)],
            name="turnover_date_idx",
            partialFilterExpression={"turnover": {"$exists": True, "$ne": None}},
            background=True
        )
```

### 2. 高级索引策略

#### 复合索引字段顺序优化

```python
class CompoundIndexOptimizer:
    """复合索引优化器"""

    @staticmethod
    def analyze_query_selectivity(collection, query_patterns):
        """分析查询字段的选择性"""
        analysis_results = {}

        for pattern_name, query in query_patterns.items():
            # 计算各字段的选择性
            field_selectivity = {}

            for field in query.keys():
                if field in ["$gte", "$lte", "$gt", "$lt"]:
                    continue

                # 计算字段的唯一值比例
                distinct_count = collection.distinct(field)
                total_count = collection.count_documents({})
                selectivity = len(distinct_count) / total_count

                field_selectivity[field] = {
                    "distinct_count": len(distinct_count),
                    "total_count": total_count,
                    "selectivity_ratio": selectivity
                }

            analysis_results[pattern_name] = field_selectivity

        return analysis_results

    @staticmethod
    def optimize_index_order(selectivity_analysis):
        """优化索引字段顺序"""
        recommendations = {}

        for pattern, field_selectivity in selectivity_analysis.items():
            # 按选择性排序（选择性高的字段放前面）
            sorted_fields = sorted(
                field_selectivity.items(),
                key=lambda x: x[1]["selectivity_ratio"]
            )

            # 排除特殊操作符字段
            index_fields = [field for field, _ in sorted_fields
                          if not field.startswith("$")]

            recommendations[pattern] = {
                "optimal_order": index_fields,
                "selectivity_analysis": field_selectivity
            }

        return recommendations

# 使用示例
def create_optimized_compound_indexes(db):
    """创建优化的复合索引"""
    collection = db["stock_market"]

    # 分析查询模式
    query_patterns = [
        {"symbol": "000001.SZ", "date": {"$gte": "20240101"}},
        {"date": {"$gte": "20240101"}, "symbol": {"$in": ["000001.SZ", "000002.SZ"]}},
        {"index_component": "hs300", "date": {"$gte": "20240101"}}
    ]

    optimizer = CompoundIndexOptimizer()
    selectivity = optimizer.analyze_query_selectivity(collection, query_patterns)
    recommendations = optimizer.optimize_index_order(selectivity)

    # 根据建议创建索引
    for pattern, rec in recommendations.items():
        optimal_fields = rec["optimal_order"]

        if len(optimal_fields) >= 2:
            # 创建最优顺序的复合索引
            index_spec = [(field, 1) for field in optimal_fields]
            collection.create_index(
                index_spec,
                name=f"optimal_{pattern}_idx",
                background=True
            )

            print(f"Created optimal index for {pattern}: {index_spec}")
```

#### 部分索引和稀疏索引

```python
class AdvancedIndexStrategy:
    """高级索引策略"""

    @staticmethod
    def create_partial_indexes(db):
        """创建部分索引，减少索引大小"""
        stock_market = db["stock_market"]

        # 1. 只索引活跃股票的数据
        active_symbols = [
            "000001.SZ", "000002.SZ", "600000.SS",
            "600036.SS", "000858.SZ"  # 活跃股票示例
        ]

        stock_market.create_index(
            [("symbol", 1), ("date", 1)],
            name="active_stocks_idx",
            partialFilterExpression={
                "symbol": {"$in": active_symbols}
            },
            background=True
        )

        # 2. 只索引高成交量的数据
        stock_market.create_index(
            [("date", 1), ("volume", -1)],
            name="high_volume_idx",
            partialFilterExpression={
                "volume": {"$gte": 10000000}  # 成交量大于1000万股
            },
            background=True
        )

        # 3. 只索引有市值的数据
        stock_market.create_index(
            [("market_cap", -1), ("symbol", 1)],
            name="market_cap_idx",
            partialFilterExpression={
                "market_cap": {"$exists": True, "$ne": None}
            },
            background=True
        )

        # 4. 只索引特定日期范围的数据
        current_year = datetime.now().year
        stock_market.create_index(
            [("date", 1), ("symbol", 1)],
            name="recent_data_idx",
            partialFilterExpression={
                "date": {"$gte": f"{current_year}0101"}
            },
            background=True
        )

    @staticmethod
    def create_sparse_indexes(db):
        """创建稀疏索引，跳过缺失字段的文档"""
        stock_market = db["stock_market"]

        # 1. 稀疏索引：只索引有换手率的文档
        stock_market.create_index(
            [("turnover", 1), ("date", -1)],
            name="sparse_turnover_idx",
            sparse=True,  # 稀疏索引
            background=True
        )

        # 2. 稀疏索引：只索引有市盈率的文档
        stock_market.create_index(
            [("pe_ratio", 1), ("symbol", 1)],
            name="sparse_pe_ratio_idx",
            sparse=True,
            background=True
        )

        # 3. 稀疏索引：只索引有指数成分标识的文档
        stock_market.create_index(
            [("index_component", 1), ("date", 1)],
            name="sparse_index_component_idx",
            sparse=True,
            background=True
        )
```

### 3. 索引监控和维护

```python
class IndexMonitor:
    """索引监控器"""

    def __init__(self, db_handler):
        self.db_handler = db_handler
        self.db = db_handler.mongo_client[db_handler.config["MONGO_DB"]]

    def analyze_index_usage(self, collection_name, days=7):
        """分析索引使用情况"""
        collection = self.db[collection_name]

        # 获取集合统计信息
        stats = self.db.command("collStats", collection_name)

        # 获取索引信息
        indexes = collection.list_indexes()

        # 分析索引大小和使用情况
        index_analysis = []

        for index in indexes:
            if index["name"] == "_id_":
                continue

            # 获取索引统计
            index_stats = self.db.command("collStats", collection_name, indexDetails=True)
            index_detail = None

            for detail in index_stats.get("indexDetails", []):
                if detail.get("name") == index["name"]:
                    index_detail = detail
                    break

            analysis = {
                "name": index["name"],
                "key": index["key"],
                "unique": index.get("unique", False),
                "sparse": index.get("sparse", False),
                "partial": "partialFilterExpression" in index,
                "size_bytes": index_detail.get("size", 0) if index_detail else 0,
                "size_mb": (index_detail.get("size", 0) if index_detail else 0) / (1024*1024)
            }

            index_analysis.append(analysis)

        return {
            "collection": collection_name,
            "total_size_mb": stats.get("size", 0) / (1024*1024),
            "total_index_size_mb": stats.get("totalIndexSize", 0) / (1024*1024),
            "document_count": stats.get("count", 0),
            "avg_doc_size": stats.get("avgObjSize", 0),
            "indexes": index_analysis
        }

    def find_unused_indexes(self, collection_name):
        """查找未使用的索引"""
        # 注意：需要MongoDB 4.2+和适当的监控配置
        collection = self.db[collection_name]

        # 模拟索引使用分析 (实际环境需要使用$indexStats)
        all_indexes = [idx["name"] for idx in collection.list_indexes() if idx["name"] != "_id_"]

        # 常用查询模式 (需要从应用日志收集)
        common_query_patterns = [
            {"symbol": 1, "date": 1},
            {"date": 1, "symbol": 1},
            {"date": 1},
            {"symbol": 1}
        ]

        used_indexes = set()
        for pattern in common_query_patterns:
            for idx in all_indexes:
                # 简化的索引匹配逻辑
                if any(field in str(pattern) for field in idx.split("_")):
                    used_indexes.add(idx)

        unused_indexes = set(all_indexes) - used_indexes

        return {
            "total_indexes": len(all_indexes),
            "used_indexes": len(used_indexes),
            "unused_indexes": list(unused_indexes),
            "unused_count": len(unused_indexes)
        }

    def recommend_index_optimizations(self, collection_name):
        """推荐索引优化"""
        analysis = self.analyze_index_usage(collection_name)
        unused = self.find_unused_indexes(collection_name)

        recommendations = []

        # 1. 删除未使用的索引
        if unused["unused_count"] > 0:
            recommendations.append({
                "type": "remove_unused",
                "description": f"删除 {unused['unused_count']} 个未使用的索引",
                "indexes": unused["unused_indexes"],
                "space_saving_mb": sum(
                    idx["size_mb"] for idx in analysis["indexes"]
                    if idx["name"] in unused["unused_indexes"]
                )
            })

        # 2. 重复索引检测
        index_keys = [str(idx["key"]) for idx in analysis["indexes"]]
        duplicate_keys = [key for key in set(index_keys) if index_keys.count(key) > 1]

        if duplicate_keys:
            recommendations.append({
                "type": "remove_duplicates",
                "description": f"删除重复的索引: {duplicate_keys}",
                "duplicate_keys": duplicate_keys
            })

        # 3. 索引大小过大警告
        large_indexes = [idx for idx in analysis["indexes"] if idx["size_mb"] > 1000]

        if large_indexes:
            recommendations.append({
                "type": "optimize_large_indexes",
                "description": f"优化 {len(large_indexes)} 个过大的索引",
                "large_indexes": large_indexes
            })

        # 4. 索引覆盖率分析
        total_index_size = analysis["total_index_size_mb"]
        data_size = analysis["total_size_mb"]

        if total_index_size > data_size * 2:  # 索引大小超过数据2倍
            recommendations.append({
                "type": "reduce_index_overhead",
                "description": f"索引大小({total_index_size:.1f}MB)过大，考虑使用部分索引",
                "index_ratio": total_index_size / data_size
            })

        return {
            "collection": collection_name,
            "analysis": analysis,
            "unused_analysis": unused,
            "recommendations": recommendations
        }

# 使用示例
def monitor_and_optimize_indexes(db_handler):
    """监控和优化索引"""
    monitor = IndexMonitor(db_handler)

    # 分析主要集合
    collections = ["stock_market", "factor_base", "factors"]

    for collection_name in collections:
        print(f"\n=== 分析集合: {collection_name} ===")

        # 基本分析
        analysis = monitor.analyze_index_usage(collection_name)
        print(f"文档数量: {analysis['document_count']:,}")
        print(f"数据大小: {analysis['total_size_mb']:.1f} MB")
        print(f"索引大小: {analysis['total_index_size_mb']:.1f} MB")
        print(f"索引数量: {len(analysis['indexes'])}")

        # 优化建议
        recommendations = monitor.recommend_index_optimizations(collection_name)

        if recommendations["recommendations"]:
            print("\n优化建议:")
            for rec in recommendations["recommendations"]:
                print(f"- {rec['description']}")
```

## 查询性能优化

### 1. 查询模式分析

```python
class QueryPerformanceAnalyzer:
    """查询性能分析器"""

    def __init__(self, db_handler):
        self.db_handler = db_handler
        self.db = db_handler.mongo_client[db_handler.config["MONGO_DB"]]

    def analyze_query_performance(self, collection_name, queries):
        """分析查询性能"""
        collection = self.db[collection_name]
        results = []

        for query_info in queries:
            query_name = query_info["name"]
            query = query_info["query"]

            # 1. 执行查询统计
            start_time = time.time()
            count_result = collection.count_documents(query)
            count_time = time.time() - start_time

            # 2. 获取执行计划
            explain_result = collection.find(query).explain("executionStats")

            # 3. 提取性能指标
            execution_stats = explain_result.get("executionStats", {})
            winning_plan = execution_stats.get("executionStages", {})

            analysis = {
                "query_name": query_name,
                "query": query,
                "document_count": count_result,
                "count_time_ms": count_time * 1000,
                "total_docs_examined": execution_stats.get("totalDocsExamined", 0),
                "total_keys_examined": execution_stats.get("totalKeysExamined", 0),
                "execution_time_ms": execution_stats.get("executionTimeMillis", 0),
                "indexes_used": self._extract_used_indexes(explain_result),
                "collection_scan": winning_plan.get("stage") == "COLLSCAN",
                "efficiency": self._calculate_query_efficiency(execution_stats, count_result)
            }

            results.append(analysis)

        return results

    def _extract_used_indexes(self, explain_result):
        """提取使用的索引"""
        used_indexes = []

        def traverse_plan(stage):
            if isinstance(stage, dict):
                if "indexName" in stage:
                    used_indexes.append(stage["indexName"])

                for key, value in stage.items():
                    if key in ["inputStage", "inputStages", "innerStage", "outerStage"]:
                        if isinstance(value, list):
                            for sub_stage in value:
                                traverse_plan(sub_stage)
                        else:
                            traverse_plan(value)

        winning_plan = explain_result.get("queryPlanner", {}).get("winningPlan", {})
        traverse_plan(winning_plan)

        return list(set(used_indexes)) if used_indexes else ["COLLSCAN"]

    def _calculate_query_efficiency(self, execution_stats, result_count):
        """计算查询效率"""
        total_examined = execution_stats.get("totalDocsExamined", 0)

        if result_count == 0:
            return 0.0

        # 效率 = 结果数量 / 检查的文档数量
        efficiency = result_count / total_examined if total_examined > 0 else 0.0

        return {
            "efficiency_ratio": efficiency,
            "docs_examined_per_result": total_examined / result_count,
            "performance_grade": self._grade_performance(efficiency)
        }

    def _grade_performance(self, efficiency):
        """性能评级"""
        if efficiency >= 0.8:
            return "优秀"
        elif efficiency >= 0.5:
            return "良好"
        elif efficiency >= 0.2:
            return "一般"
        else:
            return "需要优化"

    def generate_optimization_suggestions(self, query_analysis):
        """生成优化建议"""
        suggestions = []

        for analysis in query_analysis:
            query_suggestions = []

            # 1. 检查是否使用了集合扫描
            if analysis["collection_scan"]:
                query_suggestions.append({
                    "type": "add_index",
                    "description": "查询使用了集合扫描，建议添加索引",
                    "recommendation": f"为查询字段创建复合索引: {analysis['query']}"
                })

            # 2. 检查查询效率
            efficiency = analysis["efficiency"]["efficiency_ratio"]
            if efficiency < 0.5:
                query_suggestions.append({
                    "type": "improve_selectivity",
                    "description": f"查询效率较低({efficiency:.2f})，检查索引选择性和查询条件",
                    "recommendation": "优化查询条件或创建更精确的索引"
                })

            # 3. 检查文档检查数量
            docs_per_result = analysis["efficiency"]["docs_examined_per_result"]
            if docs_per_result > 100:
                query_suggestions.append({
                    "type": "reduce_examined_docs",
                    "description": f"每个结果检查了{docs_per_result:.0f}个文档，效率较低",
                    "recommendation": "创建更好的索引或优化查询条件"
                })

            # 4. 检查执行时间
            execution_time = analysis["execution_time_ms"]
            if execution_time > 1000:  # 超过1秒
                query_suggestions.append({
                    "type": "slow_query",
                    "description": f"查询执行时间过长({execution_time:.0f}ms)",
                    "recommendation": "考虑添加索引或使用聚合管道优化"
                })

            suggestions.append({
                "query_name": analysis["query_name"],
                "suggestions": query_suggestions
            })

        return suggestions

# 使用示例
def analyze_and_optimize_queries(db_handler):
    """分析和优化查询"""
    analyzer = QueryPerformanceAnalyzer(db_handler)

    # 定义测试查询
    test_queries = [
        {
            "name": "单股票单日查询",
            "query": {
                "symbol": "000001.SZ",
                "date": "20240327"
            }
        },
        {
            "name": "单股票日期范围查询",
            "query": {
                "symbol": "000001.SZ",
                "date": {"$gte": "20240101", "$lte": "20240327"}
            }
        },
        {
            "name": "多股票日期范围查询",
            "query": {
                "symbol": {"$in": ["000001.SZ", "000002.SZ", "600000.SS"]},
                "date": {"$gte": "20240301", "$lte": "20240327"}
            }
        },
        {
            "name": "指数成分股查询",
            "query": {
                "index_component": "hs300",
                "date": {"$gte": "20240301"}
            }
        }
    ]

    # 分析查询性能
    results = analyzer.analyze_query_performance("stock_market", test_queries)

    # 生成优化建议
    suggestions = analyzer.generate_optimization_suggestions(results)

    # 输出分析结果
    print("=== 查询性能分析结果 ===")
    for result in results:
        print(f"\n查询: {result['query_name']}")
        print(f"文档数量: {result['document_count']:,}")
        print(f"执行时间: {result['execution_time_ms']:.2f} ms")
        print(f"使用索引: {', '.join(result['indexes_used'])}")
        print(f"效率: {result['efficiency']['performance_grade']}")
        print(f"检查文档/结果: {result['efficiency']['docs_examined_per_result']:.1f}")

    print("\n=== 优化建议 ===")
    for suggestion in suggestions:
        print(f"\n查询: {suggestion['query_name']}")
        for s in suggestion['suggestions']:
            print(f"- {s['description']}")
```

### 2. 聚合管道优化

```python
class AggregationOptimizer:
    """聚合管道优化器"""

    def __init__(self, db_handler):
        self.db_handler = db_handler
        self.db = db_handler.mongo_client[db_handler.config["MONGO_DB"]]

    def optimize_factor_aggregation(self, factor_name, start_date, end_date):
        """优化因子数据聚合"""
        factors = self.db["factors"]

        # 优化的聚合管道
        pipeline = [
            # 1. $match: 早期过滤，减少处理的数据量
            {
                "$match": {
                    "factor_name": factor_name,
                    "date": {"$gte": start_date, "$lte": end_date}
                }
            },

            # 2. $sort: 如果后续需要分组，先排序提高性能
            {
                "$sort": {"date": 1, "symbol": 1}
            },

            # 3. $group: 分组聚合
            {
                "$group": {
                    "_id": {
                        "symbol": "$symbol",
                        "date": "$date"
                    },
                    "factor_value": {"$first": "$factor_value"},
                    "calculation_date": {"$first": "$calculation_date"}
                }
            },

            # 4. $project: 只返回需要的字段
            {
                "$project": {
                    "_id": 0,
                    "symbol": "$_id.symbol",
                    "date": "$_id.date",
                    "factor_value": 1,
                    "calculation_date": 1
                }
            },

            # 5. $sort: 最终排序
            {
                "$sort": {"date": 1, "symbol": 1}
            }
        ]

        return list(factors.aggregate(pipeline, allowDiskUse=True))

    def optimize_market_data_aggregation(self, start_date, end_date, symbols=None):
        """优化市场数据聚合"""
        stock_market = self.db["stock_market"]

        # 构建查询条件
        match_condition = {
            "date": {"$gte": start_date, "$lte": end_date}
        }

        if symbols:
            match_condition["symbol"] = {"$in": symbols}

        # 优化的聚合管道
        pipeline = [
            # 1. $match: 早期过滤
            {"$match": match_condition},

            # 2. $addFields: 计算衍生字段
            {
                "$addFields": {
                    "price_change": {"$subtract": ["$close", "$open"]},
                    "price_change_pct": {
                        "$multiply": [
                            {"$divide": [
                                {"$subtract": ["$close", "$open"]},
                                "$open"
                            ]},
                            100
                        ]
                    }
                }
            },

            # 3. $group: 按股票分组计算统计指标
            {
                "$group": {
                    "_id": "$symbol",
                    "avg_close": {"$avg": "$close"},
                    "max_high": {"$max": "$high"},
                    "min_low": {"$min": "$low"},
                    "total_volume": {"$sum": "$volume"},
                    "total_amount": {"$sum": "$amount"},
                    "volatility": {"$stdDevPop": "$close"},
                    "price_changes": {"$push": "$price_change"},
                    "dates": {"$push": "$date"}
                }
            },

            # 4. $addFields: 计算额外统计指标
            {
                "$addFields": {
                    "avg_volume": {"$divide": ["$total_volume", {"$size": "$dates"}]},
                    "volatility_pct": {
                        "$multiply": [
                            {"$divide": ["$volatility", "$avg_close"]},
                            100
                        ]
                    }
                }
            },

            # 5. $project: 格式化输出
            {
                "$project": {
                    "_id": 0,
                    "symbol": "$_id",
                    "avg_close": {"$round": ["$avg_close", 2]},
                    "max_high": {"$round": ["$max_high", 2]},
                    "min_low": {"$round": ["$min_low", 2]},
                    "total_volume": 1,
                    "avg_volume": {"$round": ["$avg_volume", 0]},
                    "volatility_pct": {"$round": ["$volatility_pct", 2]},
                    "data_points": {"$size": "$dates"}
                }
            },

            # 6. $sort: 按成交量排序
            {"$sort": {"total_volume": -1}}
        ]

        return list(stock_market.aggregate(pipeline, allowDiskUse=True))

    def create_performance_materialized_view(self):
        """创建性能物化视图"""
        factors = self.db["factors"]

        # 创建聚合管道物化视图
        pipeline = [
            # 匹配最近一年的数据
            {
                "$match": {
                    "date": {"$gte": "20230101"}
                }
            },

            # 按因子名称和日期分组，计算统计指标
            {
                "$group": {
                    "_id": {
                        "factor_name": "$factor_name",
                        "date": "$date"
                    },
                    "mean": {"$avg": "$factor_value"},
                    "std": {"$stdDevPop": "$factor_value"},
                    "min": {"$min": "$factor_value"},
                    "max": {"$max": "$factor_value"},
                    "count": {"$sum": 1},
                    "values": {"$push": "$factor_value"}
                }
            },

            # 计算分位数
            {
                "$addFields": {
                    "sorted_values": {"$sortArray": ["$values", 1]},
                    "count_minus_one": {"$subtract": ["$count", 1]}
                }
            },

            {
                "$addFields": {
                    "p25": {
                        "$arrayElemAt": [
                            "$sorted_values",
                            {"$floor": {"$multiply": ["$count_minus_one", 0.25]}}
                        ]
                    },
                    "p50": {
                        "$arrayElemAt": [
                            "$sorted_values",
                            {"$floor": {"$multiply": ["$count_minus_one", 0.5]}}
                        ]
                    },
                    "p75": {
                        "$arrayElemAt": [
                            "$sorted_values",
                            {"$floor": {"$multiply": ["$count_minus_one", 0.75]}}
                        ]
                    }
                }
            },

            # 最终投影
            {
                "$project": {
                    "_id": 0,
                    "factor_name": "$_id.factor_name",
                    "date": "$_id.date",
                    "mean": {"$round": ["$mean", 4]},
                    "std": {"$round": ["$std", 4]},
                    "min": {"$round": ["$min", 4]},
                    "max": {"$round": ["$max", 4]},
                    "p25": {"$round": ["$p25", 4]},
                    "p50": {"$round": ["$p50", 4]},
                    "p75": {"$round": ["$p75", 4]},
                    "count": 1
                }
            }
        ]

        # 创建物化视图 (MongoDB 4.2+)
        try:
            self.db.command({
                "create": "factor_stats_mv",
                "viewOn": "factors",
                "pipeline": pipeline
            })
            print("成功创建因子统计物化视图")
        except Exception as e:
            print(f"创建物化视图失败: {e}")

        return pipeline
```

### 3. 批量操作优化

```python
class BatchOperationOptimizer:
    """批量操作优化器"""

    def __init__(self, db_handler):
        self.db_handler = db_handler
        self.db = db_handler.mongo_client[db_handler.config["MONGO_DB"]]

    def optimized_bulk_upsert(self, collection_name, documents, batch_size=1000):
        """优化的批量upsert操作"""
        collection = self.db[collection_name]

        # 分批处理，避免内存溢出
        total_processed = 0
        start_time = time.time()

        for i in range(0, len(documents), batch_size):
            batch = documents[i:i + batch_size]

            # 构建批量操作
            operations = []
            for doc in batch:
                if collection_name == "stock_market":
                    filter_key = {"date": doc["date"], "symbol": doc["symbol"]}
                elif collection_name == "factors":
                    filter_key = {
                        "date": doc["date"],
                        "symbol": doc["symbol"],
                        "factor_name": doc["factor_name"]
                    }
                else:
                    filter_key = {"_id": doc.get("_id")}

                operations.append(
                    UpdateOne(
                        filter_key,
                        {"$set": doc},
                        upsert=True
                    )
                )

            # 执行批量操作
            try:
                result = collection.bulk_write(
                    operations,
                    ordered=False,  # 无序执行，提高性能
                    bypass_document_validation=False
                )

                total_processed += len(batch)

                # 进度报告
                progress = (total_processed / len(documents)) * 100
                elapsed = time.time() - start_time
                rate = total_processed / elapsed if elapsed > 0 else 0

                print(f"进度: {progress:.1f}% ({total_processed:,}/{len(documents):,}) "
                      f"速度: {rate:.0f} docs/sec")

            except BulkWriteError as bwe:
                print(f"批量写入错误: {bwe.details}")
                # 可以选择重试失败的操作
                continue

        total_time = time.time() - start_time
        print(f"批量upsert完成: {total_processed:,} 文档, 耗时: {total_time:.2f}秒")

        return total_processed

    def optimized_bulk_insert(self, collection_name, documents, batch_size=5000):
        """优化的批量插入操作"""
        collection = self.db[collection_name]

        # 预处理文档：移除_id字段，让MongoDB自动生成
        clean_docs = []
        for doc in documents:
            clean_doc = doc.copy()
            clean_doc.pop("_id", None)
            clean_docs.append(clean_doc)

        # 分批插入
        total_inserted = 0
        start_time = time.time()

        for i in range(0, len(clean_docs), batch_size):
            batch = clean_docs[i:i + batch_size]

            try:
                result = collection.insert_many(
                    batch,
                    ordered=False,  # 无序插入，提高性能
                    bypass_document_validation=False
                )

                total_inserted += len(result.inserted_ids)

                # 进度报告
                progress = (total_inserted / len(clean_docs)) * 100
                elapsed = time.time() - start_time
                rate = total_inserted / elapsed if elapsed > 0 else 0

                print(f"插入进度: {progress:.1f}% ({total_inserted:,}/{len(clean_docs):,}) "
                      f"速度: {rate:.0f} docs/sec")

            except BulkWriteError as bwe:
                print(f"批量插入错误: {bwe.details}")
                continue

        total_time = time.time() - start_time
        print(f"批量插入完成: {total_inserted:,} 文档, 耗时: {total_time:.2f}秒")

        return total_inserted

    def parallel_bulk_operations(self, operations_list, max_workers=4):
        """并行批量操作"""
        from concurrent.futures import ThreadPoolExecutor, as_completed

        results = []

        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            # 提交所有操作
            future_to_op = {
                executor.submit(self._execute_operation, op): op
                for op in operations_list
            }

            # 收集结果
            for future in as_completed(future_to_op):
                operation = future_to_op[future]

                try:
                    result = future.result()
                    results.append({
                        "operation": operation["name"],
                        "status": "success",
                        "result": result,
                        "time": result.get("time", 0)
                    })
                except Exception as e:
                    results.append({
                        "operation": operation["name"],
                        "status": "error",
                        "error": str(e),
                        "time": 0
                    })

        return results

    def _execute_operation(self, operation):
        """执行单个批量操作"""
        start_time = time.time()

        collection_name = operation["collection"]
        operation_type = operation["type"]
        documents = operation["documents"]

        if operation_type == "upsert":
            result = self.optimized_bulk_upsert(collection_name, documents)
        elif operation_type == "insert":
            result = self.optimized_bulk_insert(collection_name, documents)
        else:
            raise ValueError(f"不支持的操作类型: {operation_type}")

        elapsed_time = time.time() - start_time

        return {
            "count": result,
            "time": elapsed_time
        }

# 使用示例
def demonstrate_optimized_operations(db_handler):
    """演示优化批量操作"""
    optimizer = BatchOperationOptimizer(db_handler)

    # 准备测试数据
    test_documents = []
    for i in range(10000):
        doc = {
            "date": "20240327",
            "symbol": f"00000{i % 1000:03d}.SZ",
            "close": 10.0 + (i % 100) * 0.1,
            "volume": 1000000 + i * 100
        }
        test_documents.append(doc)

    print("=== 批量操作性能测试 ===")

    # 1. 批量upsert
    print("\n1. 批量upsert测试:")
    start_time = time.time()
    upserted = optimizer.optimized_bulk_upsert("stock_market", test_documents[:5000])
    upsert_time = time.time() - start_time
    print(f"Upserted {upserted} documents in {upsert_time:.2f} seconds")

    # 2. 批量insert
    print("\n2. 批量insert测试:")
    start_time = time.time()
    inserted = optimizer.optimized_bulk_insert("test_collection", test_documents)
    insert_time = time.time() - start_time
    print(f"Inserted {inserted} documents in {insert_time:.2f} seconds")

    # 3. 并行批量操作
    print("\n3. 并行批量操作测试:")
    operations = [
        {
            "name": "stock_market_upsert",
            "collection": "stock_market",
            "type": "upsert",
            "documents": test_documents[:2500]
        },
        {
            "name": "factor_data_upsert",
            "collection": "factors",
            "type": "upsert",
            "documents": [{"factor_name": "test", "factor_value": i,
                           "date": "20240327", "symbol": f"00000{i%100:03d}.SZ"}
                          for i in range(2500)]
        }
    ]

    start_time = time.time()
    parallel_results = optimizer.parallel_bulk_operations(operations, max_workers=2)
    parallel_time = time.time() - start_time

    print(f"并行操作完成，耗时: {parallel_time:.2f}秒")
    for result in parallel_results:
        print(f"- {result['operation']}: {result['status']}, "
              f"时间: {result['time']:.2f}秒")
```

## 数据访问模式

### 1. 高性能数据读取器

```python
class OptimizedMarketDataReader:
    """优化的市场数据读取器"""

    def __init__(self, db_handler, use_partitioned=True):
        self.db_handler = db_handler
        self.db = db_handler.mongo_client[db_handler.config["MONGO_DB"]]
        self.use_partitioned = use_partitioned
        self.query_cache = {}  # 查询结果缓存

    def get_market_data(self, symbols, start_date, end_date, fields=None):
        """获取市场数据（优化版本）"""
        # 1. 参数验证和标准化
        if isinstance(symbols, str):
            symbols = [symbols]

        if not fields:
            fields = ["date", "symbol", "open", "high", "low", "close", "volume"]

        # 2. 缓存检查
        cache_key = self._generate_cache_key(symbols, start_date, end_date, fields)
        if cache_key in self.query_cache:
            print("从缓存获取数据")
            return self.query_cache[cache_key]

        # 3. 构建投影字段
        projection = {field: 1 for field in fields}
        projection["_id"] = 0  # 排除_id字段

        # 4. 构建查询条件
        query = {
            "symbol": {"$in": symbols},
            "date": {"$gte": start_date, "$lte": end_date}
        }

        # 5. 执行优化查询
        if self.use_partitioned:
            results = self._partitioned_query("stock_market", query, projection)
        else:
            results = self._standard_query("stock_market", query, projection)

        # 6. 数据后处理
        processed_results = self._post_process_results(results, fields)

        # 7. 缓存结果
        self.query_cache[cache_key] = processed_results

        return processed_results

    def _partitioned_query(self, base_collection, query, projection):
        """分区查询"""
        all_results = []

        # 确定涉及的分区
        date_range = query["date"]
        start_year = int(list(date_range.values())[0]["$gte"][:4])
        end_year = int(list(date_range.values())[0]["$lte"][:4])

        years = range(start_year, end_year + 1)

        # 并行查询各分区
        from concurrent.futures import ThreadPoolExecutor, as_completed

        with ThreadPoolExecutor(max_workers=min(4, len(years))) as executor:
            future_to_year = {}

            for year in years:
                # 确定集合名称
                if year == datetime.now().year:
                    collection_name = base_collection
                else:
                    collection_name = f"{base_collection}_{year}"

                # 提交查询任务
                future = executor.submit(
                    self._execute_collection_query,
                    collection_name,
                    query,
                    projection
                )
                future_to_year[future] = year

            # 收集结果
            for future in as_completed(future_to_year):
                year = future_to_year[future]

                try:
                    year_results = future.result()
                    all_results.extend(year_results)
                    print(f"分区 {year}: 查询到 {len(year_results)} 条记录")
                except Exception as e:
                    print(f"分区 {year} 查询失败: {e}")

        return all_results

    def _standard_query(self, collection_name, query, projection):
        """标准查询"""
        collection = self.db[collection_name]

        # 使用批量游标，处理大数据集
        cursor = collection.find(
            query,
            projection=projection,
            batch_size=10000  # 批量大小
        ).sort("date", 1)  # 按日期排序

        return list(cursor)

    def _execute_collection_query(self, collection_name, query, projection):
        """执行单个集合查询"""
        try:
            collection = self.db[collection_name]
            results = list(collection.find(query, projection))
            return results
        except Exception as e:
            print(f"集合 {collection_name} 查询失败: {e}")
            return []

    def _generate_cache_key(self, symbols, start_date, end_date, fields):
        """生成缓存键"""
        symbols_str = "_".join(sorted(symbols))
        fields_str = "_".join(sorted(fields))
        return f"{symbols_str}_{start_date}_{end_date}_{fields_str}"

    def _post_process_results(self, results, fields):
        """后处理查询结果"""
        if not results:
            return []

        # 1. 按日期排序
        results.sort(key=lambda x: x.get("date", ""))

        # 2. 数据类型转换
        processed = []
        for result in results:
            processed_result = {}

            for field in fields:
                value = result.get(field)

                # 数值字段转换
                if field in ["open", "high", "low", "close", "volume", "amount"]:
                    processed_result[field] = float(value) if value is not None else 0.0
                else:
                    processed_result[field] = value

            processed.append(processed_result)

        return processed

    def get_factor_data(self, factor_names, start_date, end_date, symbols=None):
        """获取因子数据"""
        if isinstance(factor_names, str):
            factor_names = [factor_names]

        # 构建查询条件
        query = {
            "factor_name": {"$in": factor_names},
            "date": {"$gte": start_date, "$lte": end_date}
        }

        if symbols:
            if isinstance(symbols, str):
                symbols = [symbols]
            query["symbol"] = {"$in": symbols}

        # 投影字段
        projection = {
            "date": 1,
            "symbol": 1,
            "factor_name": 1,
            "factor_value": 1,
            "_id": 0
        }

        # 执行查询
        collection = self.db["factors"]
        results = list(collection.find(query, projection).sort("date", 1))

        return results

    def clear_cache(self):
        """清除查询缓存"""
        self.query_cache.clear()
        print("查询缓存已清除")

# 使用示例
def demonstrate_optimized_reading(db_handler):
    """演示优化读取"""
    reader = OptimizedMarketDataReader(db_handler, use_partitioned=True)

    # 1. 获取单股票数据
    print("=== 获取单股票数据 ===")
    single_stock_data = reader.get_market_data(
        symbols=["000001.SZ"],
        start_date="20240301",
        end_date="20240327",
        fields=["date", "close", "volume"]
    )
    print(f"获取到 {len(single_stock_data)} 条记录")

    # 2. 获取多股票数据
    print("\n=== 获取多股票数据 ===")
    multi_stock_data = reader.get_market_data(
        symbols=["000001.SZ", "000002.SZ", "600000.SS"],
        start_date="20240301",
        end_date="20240327"
    )
    print(f"获取到 {len(multi_stock_data)} 条记录")

    # 3. 获取因子数据
    print("\n=== 获取因子数据 ===")
    factor_data = reader.get_factor_data(
        factor_names=["momentum_20d", "volatility_20d"],
        start_date="20240301",
        end_date="20240327",
        symbols=["000001.SZ"]
    )
    print(f"获取到 {len(factor_data)} 条因子记录")
```

## 监控和维护

### 1. 数据库性能监控

```python
class DatabasePerformanceMonitor:
    """数据库性能监控器"""

    def __init__(self, db_handler):
        self.db_handler = db_handler
        self.db = db_handler.mongo_client[db_handler.config["MONGO_DB"]]

    def get_database_stats(self):
        """获取数据库统计信息"""
        stats = self.db.command("dbStats")

        return {
            "collections": stats.get("collections", 0),
            "objects": stats.get("objects", 0),
            "data_size": stats.get("dataSize", 0) / (1024 * 1024),  # MB
            "index_size": stats.get("indexSize", 0) / (1024 * 1024),  # MB
            "storage_size": stats.get("storageSize", 0) / (1024 * 1024),  # MB
            "avg_obj_size": stats.get("avgObjSize", 0),
            "indexes": stats.get("indexes", 0)
        }

    def get_collection_performance(self, collection_name):
        """获取集合性能指标"""
        collection = self.db[collection_name]

        # 集合统计
        coll_stats = self.db.command("collStats", collection_name)

        # 索引统计
        indexes = collection.list_indexes()
        index_stats = []

        for index in indexes:
            index_name = index["name"]
            if index_name == "_id_":
                continue

            # 尝试获取索引使用统计 (需要MongoDB 4.2+)
            try:
                index_usage = self.db.command(
                    "collStats",
                    collection_name,
                    indexDetails=True
                )

                index_detail = None
                for detail in index_usage.get("indexDetails", []):
                    if detail.get("name") == index_name:
                        index_detail = detail
                        break

                index_stats.append({
                    "name": index_name,
                    "key": index["key"],
                    "size": index_detail.get("size", 0) / (1024 * 1024) if index_detail else 0,  # MB
                    "usage_count": index_detail.get("accesses", {}).get("ops", 0) if index_detail else 0
                })
            except:
                # 如果无法获取详细统计，使用基本信息
                index_stats.append({
                    "name": index_name,
                    "key": index["key"],
                    "size": 0,
                    "usage_count": 0
                })

        return {
            "collection": collection_name,
            "document_count": coll_stats.get("count", 0),
            "data_size_mb": coll_stats.get("size", 0) / (1024 * 1024),
            "avg_doc_size": coll_stats.get("avgObjSize", 0),
            "index_size_mb": coll_stats.get("totalIndexSize", 0) / (1024 * 1024),
            "index_count": len(indexes) - 1,  # 排除_id索引
            "indexes": index_stats
        }

    def monitor_slow_queries(self, threshold_ms=100):
        """监控慢查询"""
        # 获取当前操作 (需要适当的权限)
        try:
            inprog = self.db.command("currentOp")

            slow_operations = []
            for op in inprog.get("inprog", []):
                if op.get("secs", 0) * 1000 > threshold_ms:
                    slow_operations.append({
                        "opid": op.get("opid"),
                        "operation": op.get("op"),
                        "namespace": op.get("ns"),
                        "query": op.get("query"),
                        "duration_ms": op.get("secs", 0) * 1000,
                        "client": op.get("client"),
                        "app": op.get("appName")
                    })

            return slow_operations
        except Exception as e:
            print(f"无法获取慢查询信息: {e}")
            return []

    def analyze_data_distribution(self, collection_name, field):
        """分析数据分布"""
        collection = self.db[collection_name]

        # 使用聚合管道分析字段分布
        pipeline = [
            {
                "$group": {
                    "_id": f"${field}",
                    "count": {"$sum": 1}
                }
            },
            {
                "$sort": {"count": -1}
            },
            {
                "$limit": 20  # 只返回前20个
            }
        ]

        distribution = list(collection.aggregate(pipeline))

        # 计算统计指标
        total_docs = collection.count_documents({})
        analyzed_docs = sum(item["count"] for item in distribution)
        coverage = (analyzed_docs / total_docs) * 100 if total_docs > 0 else 0

        return {
            "field": field,
            "total_documents": total_docs,
            "analyzed_documents": analyzed_docs,
            "coverage_percent": coverage,
            "distribution": distribution
        }

    def generate_performance_report(self):
        """生成性能报告"""
        print("=== MongoDB 性能监控报告 ===")

        # 1. 数据库总体统计
        db_stats = self.get_database_stats()
        print(f"\n数据库统计:")
        print(f"  集合数量: {db_stats['collections']}")
        print(f"  文档总数: {db_stats['objects']:,}")
        print(f"  数据大小: {db_stats['data_size']:.1f} MB")
        print(f"  索引大小: {db_stats['index_size']:.1f} MB")
        print(f"  存储大小: {db_stats['storage_size']:.1f} MB")
        print(f"  平均文档大小: {db_stats['avg_obj_size']} bytes")
        print(f"  索引数量: {db_stats['indexes']}")

        # 2. 主要集合性能
        main_collections = ["stock_market", "factor_base", "factors", "stocks"]

        print(f"\n主要集合性能:")
        for collection_name in main_collections:
            try:
                perf = self.get_collection_performance(collection_name)
                print(f"\n  {collection_name}:")
                print(f"    文档数量: {perf['document_count']:,}")
                print(f"    数据大小: {perf['data_size_mb']:.1f} MB")
                print(f"    索引大小: {perf['index_size_mb']:.1f} MB")
                print(f"    平均文档大小: {perf['avg_doc_size']} bytes")
                print(f"    索引数量: {perf['index_count']}")

                # 显示最大索引
                if perf['indexes']:
                    largest_index = max(perf['indexes'], key=lambda x: x['size'])
                    print(f"    最大索引: {largest_index['name']} ({largest_index['size']:.1f} MB)")

            except Exception as e:
                print(f"  {collection_name}: 分析失败 - {e}")

        # 3. 慢查询监控
        slow_queries = self.monitor_slow_queries(1000)  # 1秒以上
        if slow_queries:
            print(f"\n慢查询 ({len(slow_queries)}个):")
            for query in slow_queries[:5]:  # 显示前5个
                print(f"  操作: {query['operation']}")
                print(f"  集合: {query['namespace']}")
                print(f"  持续时间: {query['duration_ms']:.0f} ms")
                print(f"  客户端: {query.get('client', 'N/A')}")
        else:
            print(f"\n慢查询: 无")

        # 4. 数据分布分析
        print(f"\n数据分布分析:")

        # 分析股票代码分布
        symbol_dist = self.analyze_data_distribution("stock_market", "symbol")
        print(f"  股票代码分布 (前10个):")
        for item in symbol_dist['distribution'][:10]:
            print(f"    {item['_id']}: {item['count']:,} 条记录")

        # 分析日期分布
        date_dist = self.analyze_data_distribution("stock_market", "date")
        print(f"  日期分布 (前10个):")
        for item in date_dist['distribution'][:10]:
            print(f"    {item['_id']}: {item['count']:,} 条记录")

# 使用示例
def run_performance_monitoring(db_handler):
    """运行性能监控"""
    monitor = DatabasePerformanceMonitor(db_handler)

    # 生成完整报告
    monitor.generate_performance_report()
```

### 2. 自动化维护任务

```python
class DatabaseMaintenanceManager:
    """数据库维护管理器"""

    def __init__(self, db_handler):
        self.db_handler = db_handler
        self.db = db_handler.mongo_client[db_handler.config["MONGO_DB"]]

    def cleanup_expired_data(self, days_to_keep=365):
        """清理过期数据"""
        cutoff_date = (datetime.now() - timedelta(days=days_to_keep)).strftime("%Y%m%d")

        collections_to_clean = ["stock_market", "factor_base"]

        total_deleted = 0

        for collection_name in collections_to_clean:
            print(f"清理集合 {collection_name} 的过期数据...")

            collection = self.db[collection_name]

            # 统计要删除的文档数量
            delete_query = {"date": {"$lt": cutoff_date}}
            count_to_delete = collection.count_documents(delete_query)

            if count_to_delete > 0:
                # 分批删除，避免长时间锁定
                batch_size = 10000
                deleted_count = 0

                while deleted_count < count_to_delete:
                    batch_query = {"date": {"$lt": cutoff_date}}
                    result = collection.delete_many(batch_query).limit(batch_size)

                    batch_deleted = result.deleted_count
                    deleted_count += batch_deleted
                    total_deleted += batch_deleted

                    print(f"  已删除 {deleted_count:,}/{count_to_delete:,} 条记录")

                    # 短暂暂停，避免过度占用资源
                    time.sleep(0.1)

                print(f"集合 {collection_name} 清理完成: 删除了 {deleted_count:,} 条记录")
            else:
                print(f"集合 {collection_name} 无过期数据")

        print(f"总计删除 {total_deleted:,} 条过期记录")
        return total_deleted

    def optimize_indexes(self):
        """优化索引"""
        print("开始索引优化...")

        # 获取所有集合
        collections = self.db.list_collection_names()

        for collection_name in collections:
            if collection_name.startswith("system."):
                continue

            print(f"\n优化集合: {collection_name}")
            collection = self.db[collection_name]

            # 获取现有索引
            existing_indexes = collection.list_indexes()
            index_info = {idx["name"]: idx for idx in existing_indexes}

            # 分析索引使用情况
            self._analyze_collection_indexes(collection_name, collection, index_info)

            # 推荐新索引
            self._recommend_missing_indexes(collection_name, collection)

    def _analyze_collection_indexes(self, collection_name, collection, index_info):
        """分析集合索引"""
        print(f"  分析现有索引...")

        # 统计信息
        stats = self.db.command("collStats", collection_name)
        total_docs = stats.get("count", 0)
        data_size = stats.get("size", 0)
        index_size = stats.get("totalIndexSize", 0)

        print(f"    文档数量: {total_docs:,}")
        print(f"    数据大小: {data_size / (1024*1024):.1f} MB")
        print(f"    索引大小: {index_size / (1024*1024):.1f} MB")

        # 分析每个索引
        for idx_name, idx_data in index_info.items():
            if idx_name == "_id_":
                continue

            print(f"    索引: {idx_name}")
            print(f"      键: {idx_data['key']}")
            print(f"      唯一: {idx_data.get('unique', False)}")

            # 计算索引效率 (简化版本)
            # 实际环境需要使用 $indexStats 获取准确信息
            index_efficiency = self._estimate_index_efficiency(
                collection, idx_data['key'], total_docs
            )
            print(f"      估计效率: {index_efficiency:.2f}")

    def _estimate_index_efficiency(self, collection, index_key, total_docs):
        """估算索引效率"""
        # 简化的效率计算
        # 实际应该使用查询模式分析和索引统计

        if isinstance(index_key, list):
            # 复合索引
            first_field = index_key[0][0]
        else:
            # 单字段索引
            first_field = index_key

        # 计算字段的选择性
        distinct_values = len(collection.distinct(first_field))
        selectivity = distinct_values / total_docs if total_docs > 0 else 0

        return selectivity

    def _recommend_missing_indexes(self, collection_name, collection):
        """推荐缺失的索引"""
        print(f"  分析缺失索引...")

        # 基于常见查询模式推荐索引
        common_patterns = {
            "stock_market": [
                {"fields": [("symbol", 1), ("date", 1)], "reason": "股票日期查询"},
                {"fields": [("date", 1), ("symbol", 1)], "reason": "日期股票查询"},
                {"fields": [("date", 1)], "reason": "日期范围查询"},
                {"fields": [("symbol", 1)], "reason": "单股票查询"}
            ],
            "factors": [
                {"fields": [("factor_name", 1), ("date", 1)], "reason": "因子日期查询"},
                {"fields": [("date", 1), ("symbol", 1), ("factor_name", 1)], "reason": "完整因子查询"}
            ]
        }

        existing_indexes = set(idx["name"] for idx in collection.list_indexes())

        if collection_name in common_patterns:
            for recommendation in common_patterns[collection_name]:
                # 简化的索引名称生成
                idx_name = "_".join([field[0] for field in recommendation["fields"]]) + "_idx"

                if idx_name not in existing_indexes:
                    print(f"    推荐索引: {recommendation['fields']} - {recommendation['reason']}")

    def compact_collections(self):
        """压缩集合，回收存储空间"""
        print("开始集合压缩...")

        collections = ["stock_market", "factor_base", "factors"]

        for collection_name in collections:
            try:
                print(f"压缩集合: {collection_name}")

                # 获取压缩前统计
                stats_before = self.db.command("collStats", collection_name)

                # 执行压缩
                result = self.db.command("compact", collection_name)

                # 获取压缩后统计
                stats_after = self.db.command("collStats", collection_name)

                # 计算空间节省
                space_saved = stats_before.get("size", 0) - stats_after.get("size", 0)
                space_saved_mb = space_saved / (1024 * 1024)

                print(f"  压缩完成，节省空间: {space_saved_mb:.1f} MB")

            except Exception as e:
                print(f"  压缩失败: {e}")

    def backup_critical_data(self, backup_path="./backups"):
        """备份关键数据"""
        print("开始关键数据备份...")

        backup_dir = Path(backup_path)
        backup_dir.mkdir(exist_ok=True)

        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")

        # 备份股票基础信息
        self._backup_collection("stocks", backup_dir / f"stocks_{timestamp}.json")

        # 备份最新一天的市场数据
        latest_date = datetime.now().strftime("%Y%m%d")
        market_query = {"date": latest_date}
        self._backup_collection_query(
            "stock_market",
            market_query,
            backup_dir / f"stock_market_{latest_date}_{timestamp}.json"
        )

        print(f"备份完成，文件保存在: {backup_dir}")

    def _backup_collection(self, collection_name, output_file):
        """备份整个集合"""
        collection = self.db[collection_name]

        # 使用批量游标
        cursor = collection.find({}, batch_size=10000)

        with open(output_file, 'w', encoding='utf-8') as f:
            f.write('[\n')
            first = True

            for doc in cursor:
                if not first:
                    f.write(',\n')
                else:
                    first = False

                # 移除_id字段
                doc_copy = doc.copy()
                doc_copy.pop('_id', None)

                json.dump(doc_copy, f, ensure_ascii=False, default=str)

            f.write('\n]')

        print(f"  集合 {collection_name} 备份完成: {output_file}")

    def _backup_collection_query(self, collection_name, query, output_file):
        """备份查询结果"""
        collection = self.db[collection_name]

        cursor = collection.find(query, batch_size=10000)

        with open(output_file, 'w', encoding='utf-8') as f:
            f.write('[\n')
            first = True

            for doc in cursor:
                if not first:
                    f.write(',\n')
                else:
                    first = False

                doc_copy = doc.copy()
                doc_copy.pop('_id', None)

                json.dump(doc_copy, f, ensure_ascii=False, default=str)

            f.write('\n]')

        print(f"  查询结果备份完成: {output_file}")

# 使用示例
def run_maintenance_tasks(db_handler):
    """运行维护任务"""
    maintenance = DatabaseMaintenanceManager(db_handler)

    print("=== 数据库维护任务 ===")

    # 1. 清理过期数据
    print("\n1. 清理过期数据:")
    maintenance.cleanup_expired_data(days_to_keep=730)  # 保留2年

    # 2. 优化索引
    print("\n2. 优化索引:")
    maintenance.optimize_indexes()

    # 3. 压缩集合
    print("\n3. 压缩集合:")
    maintenance.compact_collections()

    # 4. 备份关键数据
    print("\n4. 备份数据:")
    maintenance.backup_critical_data()

    print("\n维护任务完成!")
```

## 生产环境最佳实践

### 1. 连接池配置优化

```python
class ProductionConnectionManager:
    """生产环境连接管理器"""

    @staticmethod
    def get_production_config():
        """获取生产环境配置"""
        return {
            # 连接池配置
            "maxPoolSize": 50,           # 最大连接数
            "minPoolSize": 5,            # 最小连接数
            "maxIdleTimeMS": 300000,     # 最大空闲时间 5分钟
            "waitQueueTimeoutMS": 5000,  # 等待队列超时

            # 读取配置
            "readPreference": "secondaryPreferred",
            "readConcern": {"level": "majority"},

            # 写入配置
            "writeConcern": {"w": "majority", "j": True},
            "retryWrites": True,
            "retryReads": True,

            # 超时配置
            "socketTimeoutMS": 30000,         # Socket超时 30秒
            "connectTimeoutMS": 20000,        # 连接超时 20秒
            "serverSelectionTimeoutMS": 30000,  # 服务器选择超时 30秒
            "heartbeatFrequencyMS": 10000,     # 心跳频率 10秒

            # 压缩配置
            "compressors": ["zlib", "snappy"],
            "zlibCompressionLevel": 6,

            # 应用标识
            "appName": "panda_factor_prod",
        }

    @staticmethod
    def create_client_with_config(base_uri, config):
        """使用配置创建客户端"""
        from pymongo import MongoClient

        # 构建连接字符串
        connection_string = f"{base_uri}?"

        # 添加连接参数
        params = []
        for key, value in config.items():
            if isinstance(value, bool):
                params.append(f"{key}={str(value).lower()}")
            elif isinstance(value, dict):
                for sub_key, sub_value in value.items():
                    params.append(f"{key}={sub_key}={sub_value}")
            else:
                params.append(f"{key}={value}")

        connection_string += "&".join(params)

        return MongoClient(connection_string)
```

### 2. 读写分离策略

```python
class ReadWriteStrategy:
    """读写分离策略"""

    def __init__(self, db_handler):
        self.db_handler = db_handler
        self.db = db_handler.mongo_client[db_handler.config["MONGO_DB"]]

        # 不同操作的读取偏好
        self.read_preferences = {
            "realtime": "primary",           # 实时查询使用主节点
            "analytics": "secondary",        # 分析查询使用从节点
            "balanced": "secondaryPreferred"  # 平衡模式
        }

    def get_collection_for_operation(self, collection_name, operation_type="balanced"):
        """根据操作类型获取集合实例"""
        read_pref = self.read_preferences.get(operation_type, "secondaryPreferred")

        # 创建带有特定读取偏好的集合实例
        collection = self.db.get_collection(
            collection_name,
            read_preference=read_pref
        )

        return collection

    def execute_realtime_query(self, collection_name, query, projection=None):
        """执行实时查询（强一致性）"""
        collection = self.get_collection_for_operation(collection_name, "realtime")

        result = collection.find_one(query, projection)
        return result

    def execute_analytics_query(self, collection_name, pipeline, allow_disk_use=True):
        """执行分析查询（最终一致性）"""
        collection = self.get_collection_for_operation(collection_name, "analytics")

        # 对于大型聚合操作，允许使用磁盘
        result = list(collection.aggregate(
            pipeline,
            allowDiskUse=allow_disk_use,
            batchSize=1000
        ))

        return result

    def execute_batch_write(self, collection_name, operations):
        """执行批量写入（强一致性）"""
        # 写入操作总是使用主节点
        collection = self.db[collection_name]

        result = collection.bulk_write(
            operations,
            ordered=False,  # 并行执行提高性能
            write_concern=WriteConcern(w="majority", j=True)
        )

        return result
```

### 3. 监控和告警

```python
class ProductionMonitoring:
    """生产环境监控"""

    def __init__(self, db_handler, alert_webhook=None):
        self.db_handler = db_handler
        self.db = db_handler.mongo_client[db_handler.config["MONGO_DB"]]
        self.alert_webhook = alert_webhook

    def monitor_database_health(self):
        """监控数据库健康状态"""
        health_status = {
            "timestamp": datetime.now().isoformat(),
            "status": "healthy",
            "issues": [],
            "metrics": {}
        }

        try:
            # 1. 连接测试
            self.db.command("ping")
            health_status["metrics"]["connection"] = "ok"
        except Exception as e:
            health_status["status"] = "critical"
            health_status["issues"].append(f"连接失败: {str(e)}")
            health_status["metrics"]["connection"] = "failed"

        try:
            # 2. 副本集状态
            if hasattr(self.db_handler.mongo_client, 'RS'):
                rs_status = self.db.command("replSetGetStatus")
                health_status["metrics"]["replica_set"] = {
                    "primary": rs_status.get("primary"),
                    "secondaries": len([m for m in rs_status.get("members", [])
                                       if m.get("stateStr") == "SECONDARY"]),
                    "members": rs_status.get("members", [])
                }
        except Exception as e:
            health_status["issues"].append(f"副本集状态获取失败: {str(e)}")

        try:
            # 3. 服务器状态
            server_status = self.db.command("serverStatus")
            health_status["metrics"]["server"] = {
                "version": server_status.get("version"),
                "uptime": server_status.get("uptime"),
                "connections": server_status.get("connections", {}),
                "memory": server_status.get("mem", {})
            }

            # 检查连接数
            connections = server_status.get("connections", {})
            current_connections = connections.get("current", 0)
            available_connections = connections.get("available", 0)

            if current_connections > available_connections * 0.8:
                health_status["issues"].append(f"连接数过高: {current_connections}/{available_connections}")
                health_status["status"] = "warning"

        except Exception as e:
            health_status["issues"].append(f"服务器状态获取失败: {str(e)}")

        # 发送告警
        if health_status["status"] != "healthy":
            self.send_alert(health_status)

        return health_status

    def monitor_performance_metrics(self):
        """监控性能指标"""
        metrics = {
            "timestamp": datetime.now().isoformat(),
            "slow_queries": [],
            "large_operations": [],
            "index_usage": {}
        }

        try:
            # 1. 慢查询监控
            inprog = self.db.command("currentOp")
            for op in inprog.get("inprog", []):
                if op.get("secs", 0) > 1.0:  # 超过1秒的操作
                    metrics["slow_queries"].append({
                        "operation": op.get("op"),
                        "namespace": op.get("ns"),
                        "duration": op.get("secs", 0),
                        "query": op.get("query", {})
                    })

            # 2. 大操作监控
            for op in inprog.get("inprog", []):
                if op.get("command", {}).get("docsExamined", 0) > 10000:
                    metrics["large_operations"].append({
                        "operation": op.get("op"),
                        "namespace": op.get("ns"),
                        "docs_examined": op.get("command", {}).get("docsExamined", 0)
                    })

            # 3. 集合性能统计
            main_collections = ["stock_market", "factor_base", "factors"]
            for collection_name in main_collections:
                try:
                    coll_stats = self.db.command("collStats", collection_name)

                    metrics["index_usage"][collection_name] = {
                        "document_count": coll_stats.get("count", 0),
                        "index_size_mb": coll_stats.get("totalIndexSize", 0) / (1024*1024),
                        "data_size_mb": coll_stats.get("size", 0) / (1024*1024),
                        "avg_doc_size": coll_stats.get("avgObjSize", 0)
                    }
                except Exception as e:
                    metrics["index_usage"][collection_name] = {"error": str(e)}

        except Exception as e:
            metrics["error"] = str(e)

        return metrics

    def send_alert(self, status_data):
        """发送告警"""
        if not self.alert_webhook:
            return

        alert_message = {
            "level": "critical" if status_data["status"] == "critical" else "warning",
            "service": "mongodb",
            "message": f"MongoDB健康检查失败: {status_data['status']}",
            "details": status_data,
            "timestamp": datetime.now().isoformat()
        }

        try:
            import requests
            response = requests.post(
                self.alert_webhook,
                json=alert_message,
                timeout=10
            )

            if response.status_code == 200:
                print("告警发送成功")
            else:
                print(f"告警发送失败: {response.status_code}")
        except Exception as e:
            print(f"告警发送异常: {e}")

# 定时监控任务
def setup_monitoring_scheduler(db_handler, alert_webhook=None):
    """设置监控调度器"""
    monitor = ProductionMonitoring(db_handler, alert_webhook)

    # 每分钟检查数据库健康状态
    def health_check():
        health = monitor.monitor_database_health()
        if health["status"] != "healthy":
            print(f"⚠️  数据库健康告警: {health['issues']}")

    # 每5分钟检查性能指标
    def performance_check():
        metrics = monitor.monitor_performance_metrics()

        if metrics.get("slow_queries"):
            print(f"⚠️  发现 {len(metrics['slow_queries'])} 个慢查询")

        if metrics.get("large_operations"):
            print(f"⚠️  发现 {len(metrics['large_operations'])} 个大操作")

    # 使用APScheduler设置定时任务
    from apscheduler.schedulers.background import BackgroundScheduler

    scheduler = BackgroundScheduler()
    scheduler.add_job(health_check, 'interval', minutes=1)
    scheduler.add_job(performance_check, 'interval', minutes=5)

    scheduler.start()
    print("数据库监控调度器已启动")

    return scheduler
```

## 故障排除和调优

### 1. 常见问题诊断

```python
class DatabaseTroubleshooter:
    """数据库故障诊断器"""

    def __init__(self, db_handler):
        self.db_handler = db_handler
        self.db = db_handler.mongo_client[db_handler.config["MONGO_DB"]]

    def diagnose_connection_issues(self):
        """诊断连接问题"""
        diagnosis = {
            "issue_type": "connection",
            "checks": [],
            "recommendations": []
        }

        try:
            # 1. 检查连接字符串
            config = self.db_handler.config
            connection_string = f"mongodb://{config['MONGO_USER']}:****@{config['MONGO_URI']}"

            diagnosis["checks"].append({
                "test": "connection_string",
                "status": "checked",
                "result": connection_string
            })

            # 2. 检查网络连通性
            import socket
            host, port = config['MONGO_URI'].split(':')
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(5)

            try:
                result = sock.connect_ex((host, int(port)))
                sock.close()

                if result == 0:
                    diagnosis["checks"].append({
                        "test": "network_connectivity",
                        "status": "pass",
                        "result": f"网络连接正常 {host}:{port}"
                    })
                else:
                    diagnosis["checks"].append({
                        "test": "network_connectivity",
                        "status": "fail",
                        "result": f"网络连接失败 {host}:{port}"
                    })
                    diagnosis["recommendations"].append("检查网络连接和防火墙设置")
            except Exception as e:
                diagnosis["checks"].append({
                    "test": "network_connectivity",
                    "status": "error",
                    "result": f"网络测试异常: {str(e)}"
                })
                diagnosis["recommendations"].append("验证主机名和端口号")

            # 3. 检查认证
            try:
                self.db.command("ping")
                diagnosis["checks"].append({
                    "test": "authentication",
                    "status": "pass",
                    "result": "认证成功"
                })
            except Exception as e:
                diagnosis["checks"].append({
                    "test": "authentication",
                    "status": "fail",
                    "result": f"认证失败: {str(e)}"
                })
                diagnosis["recommendations"].extend([
                    "检查用户名和密码",
                    "验证用户权限",
                    "检查认证数据库设置"
                ])

        except Exception as e:
            diagnosis["checks"].append({
                "test": "general_diagnosis",
                "status": "error",
                "result": f"诊断过程异常: {str(e)}"
            })

        return diagnosis

    def diagnose_performance_issues(self):
        """诊断性能问题"""
        diagnosis = {
            "issue_type": "performance",
            "checks": [],
            "recommendations": []
        }

        try:
            # 1. 检查慢查询
            slow_queries = self.db.command("currentOp", {
                "secs": {"$gt": 1}
            })

            if slow_queries.get("inprog"):
                diagnosis["checks"].append({
                    "test": "slow_queries",
                    "status": "fail",
                    "result": f"发现 {len(slow_queries['inprog'])} 个慢查询"
                })
                diagnosis["recommendations"].append("优化查询和索引")
            else:
                diagnosis["checks"].append({
                    "test": "slow_queries",
                    "status": "pass",
                    "result": "无慢查询"
                })

            # 2. 检查索引使用情况
            for collection_name in ["stock_market", "factor_base", "factors"]:
                collection = self.db[collection_name]

                # 获取集合统计
                stats = self.db.command("collStats", collection_name)

                # 检查索引覆盖率
                doc_count = stats.get("count", 0)
                index_size = stats.get("totalIndexSize", 0)
                data_size = stats.get("size", 0)

                if doc_count > 0 and index_size > data_size * 2:
                    diagnosis["checks"].append({
                        "test": f"index_coverage_{collection_name}",
                        "status": "warning",
                        "result": f"{collection_name} 索引大小过大: {index_size/(1024*1024):.1f}MB"
                    })
                    diagnosis["recommendations"].append(f"优化 {collection_name} 集合的索引")

            # 3. 检查内存使用
            server_status = self.db.command("serverStatus")
            memory_info = server_status.get("mem", {})

            resident_mb = memory_info.get("resident", 0)
            mapped_mb = memory_info.get("mapped", 0)

            if resident_mb > 8000:  # 超过8GB
                diagnosis["checks"].append({
                    "test": "memory_usage",
                    "status": "warning",
                    "result": f"内存使用过高: {resident_mb}MB"
                })
                diagnosis["recommendations"].append("考虑增加内存或优化查询")

        except Exception as e:
            diagnosis["checks"].append({
                "test": "performance_diagnosis",
                "status": "error",
                "result": f"性能诊断异常: {str(e)}"
            })

        return diagnosis

    def diagnose_data_integrity(self):
        """诊断数据完整性"""
        diagnosis = {
            "issue_type": "data_integrity",
            "checks": [],
            "recommendations": []
        }

        try:
            # 1. 检查重复数据
            for collection_name in ["stock_market", "factors"]:
                collection = self.db[collection_name]

                # 检查stock_market重复
                if collection_name == "stock_market":
                    pipeline = [
                        {
                            "$group": {
                                "_id": {"symbol": "$symbol", "date": "$date"},
                                "count": {"$sum": 1}
                            }
                        },
                        {"$match": {"count": {"$gt": 1}}}
                    ]

                # 检查factors重复
                elif collection_name == "factors":
                    pipeline = [
                        {
                            "$group": {
                                "_id": {
                                    "symbol": "$symbol",
                                    "date": "$date",
                                    "factor_name": "$factor_name"
                                },
                                "count": {"$sum": 1}
                            }
                        },
                        {"$match": {"count": {"$gt": 1}}}
                    ]

                duplicates = list(collection.aggregate(pipeline))

                if duplicates:
                    diagnosis["checks"].append({
                        "test": f"duplicate_data_{collection_name}",
                        "status": "fail",
                        "result": f"{collection_name} 发现 {len(duplicates)} 组重复数据"
                    })
                    diagnosis["recommendations"].append(f"清理 {collection_name} 重复数据")
                else:
                    diagnosis["checks"].append({
                        "test": f"duplicate_data_{collection_name}",
                        "status": "pass",
                        "result": f"{collection_name} 无重复数据"
                    })

            # 2. 检查数据完整性
            stock_count = self.db["stocks"].count_documents({"expired": False})
            market_latest = self.db["stock_market"].distinct("date")

            if market_latest:
                latest_date = max(market_latest)
                market_count = self.db["stock_market"].count_documents({
                    "date": latest_date
                })

                coverage_ratio = market_count / stock_count if stock_count > 0 else 0

                if coverage_ratio < 0.8:  # 覆盖率低于80%
                    diagnosis["checks"].append({
                        "test": "data_coverage",
                        "status": "warning",
                        "result": f"数据覆盖率低: {coverage_ratio:.2%}"
                    })
                    diagnosis["recommendations"].append("检查数据更新流程")
                else:
                    diagnosis["checks"].append({
                        "test": "data_coverage",
                        "status": "pass",
                        "result": f"数据覆盖率正常: {coverage_ratio:.2%}"
                    })

        except Exception as e:
            diagnosis["checks"].append({
                "test": "data_integrity_check",
                "status": "error",
                "result": f"完整性检查异常: {str(e)}"
            })

        return diagnosis

    def generate_comprehensive_report(self):
        """生成综合诊断报告"""
        print("=== MongoDB 综合诊断报告 ===\n")

        # 1. 连接诊断
        print("1. 连接诊断:")
        conn_diagnosis = self.diagnose_connection_issues()
        for check in conn_diagnosis["checks"]:
            status_icon = "✅" if check["status"] == "pass" else "❌"
            print(f"   {status_icon} {check['test']}: {check['result']}")

        if conn_diagnosis["recommendations"]:
            print("   💡 建议:")
            for rec in conn_diagnosis["recommendations"]:
                print(f"      - {rec}")

        # 2. 性能诊断
        print("\n2. 性能诊断:")
        perf_diagnosis = self.diagnose_performance_issues()
        for check in perf_diagnosis["checks"]:
            if check["status"] == "pass":
                status_icon = "✅"
            elif check["status"] == "warning":
                status_icon = "⚠️"
            else:
                status_icon = "❌"
            print(f"   {status_icon} {check['test']}: {check['result']}")

        if perf_diagnosis["recommendations"]:
            print("   💡 建议:")
            for rec in perf_diagnosis["recommendations"]:
                print(f"      - {rec}")

        # 3. 数据完整性诊断
        print("\n3. 数据完整性诊断:")
        integrity_diagnosis = self.diagnose_data_integrity()
        for check in integrity_diagnosis["checks"]:
            if check["status"] == "pass":
                status_icon = "✅"
            elif check["status"] == "warning":
                status_icon = "⚠️"
            else:
                status_icon = "❌"
            print(f"   {status_icon} {check['test']}: {check['result']}")

        if integrity_diagnosis["recommendations"]:
            print("   💡 建议:")
            for rec in integrity_diagnosis["recommendations"]:
                print(f"      - {rec}")

        # 4. 总体评估
        all_checks = (conn_diagnosis["checks"] +
                      perf_diagnosis["checks"] +
                      integrity_diagnosis["checks"])

        pass_count = sum(1 for check in all_checks if check["status"] == "pass")
        total_count = len(all_checks)

        health_score = (pass_count / total_count) * 100

        print(f"\n=== 总体评估 ===")
        print(f"健康评分: {health_score:.1f}% ({pass_count}/{total_count})")

        if health_score >= 80:
            print("🟢 数据库状态: 良好")
        elif health_score >= 60:
            print("🟡 数据库状态: 一般，需要关注")
        else:
            print("🔴 数据库状态: 需要立即处理")

        return {
            "health_score": health_score,
            "connection": conn_diagnosis,
            "performance": perf_diagnosis,
            "integrity": integrity_diagnosis
        }

# 使用示例
def run_database_diagnosis(db_handler):
    """运行数据库诊断"""
    troubleshooter = DatabaseTroubleshooter(db_handler)
    report = troubleshooter.generate_comprehensive_report()
    return report
```

---

通过本指南，您应该能够：

1. **理解MongoDB架构**: 掌握PandaFactor中MongoDB的整体设计和连接管理
2. **优化数据模型**: 设计高效的文档结构和索引策略
3. **实现查询优化**: 使用聚合管道、批量操作等高级技术
4. **监控系统性能**: 建立完整的监控和维护体系
5. **处理生产环境**: 配置连接池、读写分离、故障转移
6. **故障排除**: 诊断和解决常见的数据库问题

这套MongoDB使用指南为PandaFactor提供了企业级的数据库解决方案，确保系统的高性能、高可用性和数据安全性。