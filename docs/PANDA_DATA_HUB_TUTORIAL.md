# PandaFactor Data Hub 模块教程

## 概述

`panda_data_hub` 是PandaFactor平台的数据维护中心，主要负责金融数据的自动更新、清洗和验证。该模块确保系统始终拥有最新、准确的金融数据，为因子计算和市场分析提供可靠的数据基础。

## 模块结构

```
panda_data_hub/
├── __main__.py          # 自动更新任务启动入口
├── _main_auto_.py       # 自动更新主程序
├── cleaner/             # 数据清洗模块
├── scheduler/           # 任务调度器
└── updater/             # 数据更新器
```

## 核心功能

### 1. 自动数据更新
- **定时任务**: 每日20:00自动执行数据更新
- **多数据源**: 支持Tushare、RiceQuant、XTQuant等数据源
- **增量更新**: 智能识别新增数据，避免重复下载

### 2. 数据清洗与验证
- **数据质量检查**: 自动检测异常值和缺失值
- **标准化处理**: 统一数据格式和命名规范
- **完整性验证**: 确保数据连续性和完整性

### 3. 数据存储管理
- **MongoDB集成**: 自动将清洗后的数据存储到MongoDB
- **版本控制**: 保留历史数据版本，支持回滚
- **索引优化**: 自动优化数据库查询性能

## 使用方法

### 启动自动更新服务

```bash
# 方法1: 使用uv (推荐)
uv run python -m panda_data_hub._main_auto_

# 方法2: 使用python直接运行
python -m panda_data_hub._main_auto_
```

### 手动执行数据更新

```bash
# 手动触发完整更新
uv run python -m panda_data_hub.__main__ --full-update

# 更新特定日期范围
uv run python -m panda_data_hub.__main__ --start-date 20240101 --end-date 20240131

# 仅更新特定股票
uv run python -m panda_data_hub.__main__ --symbols 000001.SZ 600000.SS
```

## 配置说明

### 环境变量配置

在`panda_common/config.yaml`中配置以下参数：

```yaml
# 数据源配置
datasource: "tqsdk"  # 可选: tqsdk, tushare, ricequant, xtquant

# MongoDB配置
mongo_uri: "localhost:27017"

# Tushare配置 (如果使用tushare)
tushare_token: "your_tushare_token"

# 更新时间配置
update_time: "20:00"  # 每日更新时间
```

### 调度器配置

```yaml
# 调度器设置
scheduler:
  enabled: true
  timezone: "Asia/Shanghai"
  max_retries: 3
  retry_delay: 300  # 秒
```

## 工作流程详解

### 1. 数据获取阶段
```
外部数据源 (Tushare/RiceQuant/XTQuant)
    ↓
数据获取器 (DataFetcher)
    ↓
原始数据缓存
```

### 2. 数据清洗阶段
```
原始数据
    ↓
数据清洗器 (DataCleaner)
    ↓
异常值处理 → 缺失值填充 → 数据标准化
    ↓
清洗后数据
```

### 3. 数据验证阶段
```
清洗后数据
    ↓
数据验证器 (DataValidator)
    ↓
完整性检查 → 一致性验证 → 质量评分
    ↓
验证通过的数据
```

### 4. 数据存储阶段
```
验证通过的数据
    ↓
数据库处理器 (DatabaseHandler)
    ↓
MongoDB存储
    ├── stock_market (市场数据)
    ├── factors (因子数据)
    └── metadata (元数据)
```

## 监控与日志

### 日志系统
```python
import logging

# 配置日志级别
logging.basicConfig(level=logging.INFO)

# 查看更新日志
tail -f logs/panda_data_hub.log
```

### 关键日志信息
- **INFO**: 正常的更新进度和状态
- **WARNING**: 数据质量问题警告
- **ERROR**: 严重错误和异常情况
- **DEBUG**: 详细的调试信息

### 监控指标
- **数据更新成功率**: 成功更新的天数/总天数
- **数据完整性**: 缺失数据的比例
- **更新耗时**: 每次更新的执行时间
- **存储空间**: 数据库占用的存储空间

## 故障排除

### 常见问题及解决方案

#### 1. 数据更新失败
**问题**: 自动更新任务执行失败
**解决方案**:
```bash
# 检查网络连接
ping api.tushare.pro

# 验证API token
python -c "import tushare as ts; ts.set_token('your_token'); print(ts.pro_api().daily(trade_date='20240101'))"

# 检查MongoDB连接
python -c "from pymongo import MongoClient; print(MongoClient('localhost:27017').server_info())"
```

#### 2. 数据质量问题
**问题**: 数据存在异常值或缺失值
**解决方案**:
```python
# 重新运行数据清洗
uv run python -m panda_data_hub.__main__ --clean-only

# 查看数据质量报告
uv run python -m panda_data_hub.__main__ --quality-report
```

#### 3. 存储空间不足
**问题**: MongoDB存储空间不足
**解决方案**:
```python
# 清理过期数据
uv run python -m panda_data_hub.__main__ --cleanup-old-data --days 365

# 压缩数据库
mongod --dbpath /path/to/db --repair
```

## 最佳实践

### 1. 定期维护
- 每周检查数据质量报告
- 每月清理过期数据
- 每季度备份重要数据

### 2. 性能优化
- 合理设置更新频率
- 使用增量更新减少数据传输
- 优化MongoDB索引提升查询性能

### 3. 安全考虑
- 定期更新API密钥
- 限制数据库访问权限
- 启用数据备份机制

## 扩展开发

### 添加新的数据源

```python
# 1. 在updater目录创建新的数据源适配器
# panda_data_hub/updater/custom_source.py

class CustomDataSource:
    def fetch_data(self, symbols, start_date, end_date):
        # 实现数据获取逻辑
        pass

    def validate_config(self):
        # 验证配置参数
        pass

# 2. 在配置文件中添加数据源配置
# panda_common/config.yaml
datasource: "custom_source"
custom_source:
    api_key: "your_api_key"
    base_url: "https://api.example.com"
```

### 自定义数据清洗规则

```python
# panda_data_hub/cleaner/custom_rules.py

class CustomCleaningRule:
    def apply(self, data):
        # 实现自定义清洗逻辑
        # 例如: 特定异常值处理
        return cleaned_data

    def validate(self, data):
        # 验证清洗结果
        return is_valid
```

## 相关文档

- [MongoDB数据存储指南](MONGODB_GUIDE.md)
- [数据更新操作指南](DATA_UPDATE_GUIDE.md)
- [量化因子开发指南](QUANTITATIVE_FACTOR_GUIDE.md)

## 技术支持

如遇到问题，请：
1. 查看日志文件获取详细错误信息
2. 检查配置文件是否正确
3. 参考故障排除章节
4. 提交Issue到项目仓库

---

*本教程基于PandaFactor v1.0编写，如有更新请参考最新版本。*