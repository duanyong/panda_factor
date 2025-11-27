# Panda Factor Server 教程

## 项目概述

Panda Factor Server 是一个基于 FastAPI 构建的 REST API 服务器，专门为量化因子开发和分析系统提供服务。它提供用户因子管理、因子计算、数据分析等核心功能，并集成了 LLM 服务和前端静态资源。

## 项目结构

```
panda_factor_server/
├── panda_factor_server/       # 服务器主包
│   ├── __main__.py           # 服务器启动入口 (主要文件)
│   ├── server.py             # 备用服务器配置
│   ├── api/                  # API 模块
│   ├── core/                 # 核心功能
│   ├── models/               # 数据模型定义
│   │   ├── common.py         # 通用模型
│   │   ├── request_body.py   # 请求体模型
│   │   ├── response_body.py  # 响应体模型
│   │   └── result_data.py    # 结果数据模型
│   ├── routes/               # 路由定义
│   │   ├── __init__.py
│   │   └── user_factor_pro.py # 用户因子专业版路由
│   ├── services/             # 业务逻辑服务
│   │   ├── models/           # 服务数据模型
│   │   └── user_factor_service.py # 用户因子服务
│   └── utils/                # 工具函数
├── setup.py                  # 安装配置
├── pyproject.toml           # 项目配置
└── README.md                # 项目说明
```

## __main__.py 启动入口详解

`__main__.py` 是 Panda Factor Server 的主要启动入口文件，位于 `panda_factor_server/__main__.py:1`。

### 1. 核心导入

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from panda_factor_server.routes import user_factor_pro
from panda_llm.routes import chat_router
import mimetypes
from pathlib import Path
from starlette.staticfiles import StaticFiles
```

### 2. FastAPI 应用初始化

```python
app = FastAPI(
    title="Panda Server",
    description="Server for Panda AI Factor System",
    version="1.0.0"
)
```

应用配置包含：
- **title**: "Panda Server" - 服务器名称
- **description**: "Server for Panda AI Factor System" - 服务描述
- **version**: "1.0.0" - API 版本

### 3. CORS 中间件配置

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 允许所有来源
    allow_credentials=True,
    allow_methods=["*"],  # 允许所有 HTTP 方法
    allow_headers=["*"],  # 允许所有请求头
)
```

CORS (Cross-Origin Resource Sharing) 配置允许前端应用跨域访问 API。

### 4. 路由注册

```python
# Include routers
app.include_router(user_factor_pro.router, prefix="/api/v1", tags=["user_factors"])
app.include_router(chat_router.router, prefix="/llm", tags=["panda_llm"])
```

- **user_factor_pro**: 用户因子专业版 API，挂载在 `/api/v1` 路径下
- **chat_router**: LLM 聊天服务 API，挂载在 `/llm` 路径下

### 5. 前端静态资源配置

```python
# 获取根目录下的panda_web
frontend_folder = Path(__file__).resolve().parent.parent.parent / "panda_web" / "panda_web" / "static"
print(f"前端静态资源文件夹路径:{frontend_folder}")

# 显式设置 .js .css 的 MIME 类型
mimetypes.add_type("text/css", ".css")
mimetypes.add_type("application/javascript", ".js")

# Mount the Vue dist directory at /factor path
app.mount("/factor", StaticFiles(directory=frontend_folder, html=True), name="static")
```

前端静态资源挂载：
- **路径**: `/factor` - 访问前端应用
- **静态文件目录**: 自动查找 `panda_web/panda_web/static` 目录
- **MIME 类型**: 明确设置 CSS 和 JavaScript 文件类型

### 6. 根路径处理器

```python
@app.get("/")
async def home():
    return {"message": "Welcome to the Panda Server!"}
```

提供简单的欢迎消息，用于测试服务器是否正常运行。

### 7. 主函数和启动配置

```python
def main():
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8111)

if __name__ == "__main__":
    main()
```

- **host**: "0.0.0.0" - 监听所有网络接口
- **port**: 8111 - 服务器端口
- **uvicorn**: ASGI 服务器，用于运行 FastAPI 应用

## 启动方式

### 方式一：直接运行模块

```bash
uv run python -m panda_factor_server.__main__
```

### 方式二：使用安装的命令

```bash
# 安装后可使用
panda_factor_server
```

### 方式三：Python 直接执行

```bash
python -m panda_factor_server.__main__
```

### 方式四：Docker 部署

```bash
# 构建镜像
docker build -f Dockerfile.server -t panda_factor_server .

# 运行容器
docker run -p 8111:8111 -e MONGO_URI=host.docker.internal:27017 panda_factor_server
```

## API 路由结构

### 用户因子 API (`/api/v1`)

主要接口包括：

#### 因子管理
- `GET /api/v1/user_factor_list` - 获取用户因子列表
- `POST /api/v1/create_factor` - 创建新因子
- `GET /api/v1/delete_factor` - 删除因子
- `POST /api/v1/update_factor` - 更新因子
- `GET /api/v1/query_factor` - 查询因子详情
- `GET /api/v1/query_factor_status` - 查询因子状态

#### 因子执行
- `GET /api/v1/run_factor` - 运行因子计算
- `GET /api/v1/query_task_status` - 查询任务状态
- `GET /api/v1/task_logs` - 获取任务日志

#### 分析图表
- `GET /api/v1/query_factor_excess_chart` - 因子超额收益图表
- `GET /api/v1/query_factor_analysis_data` - 因子分析数据
- `GET /api/v1/query_group_return_analysis` - 分组收益分析
- `GET /api/v1/query_ic_decay_chart` - IC 衰减图表
- `GET /api/v1/query_ic_density_chart` - IC 密度图表
- `GET /api/v1/query_ic_self_correlation_chart` - IC 自相关图表
- `GET /api/v1/query_ic_sequence_chart` - IC 序列图表

#### 收益分析
- `GET /api/v1/query_return_chart` - 收益图表
- `GET /api/v1/query_simple_return_chart` - 简单收益图表
- `GET /api/v1/query_last_date_top_factor` - 最新日期顶级因子
- `GET /api/v1/query_one_group_data` - 单组数据查询
- `GET /api/v1/query_rank_ic_decay_chart` - 排名 IC 衰减图表
- `GET /api/v1/query_rank_ic_density_chart` - 排名 IC 密度图表
- `GET /api/v1/query_rank_ic_self_correlation_chart` - 排名 IC 自相关图表
- `GET /api/v1/query_rank_ic_sequence_chart` - 排名 IC 序列图表

### LLM 服务 API (`/llm`)

提供 AI 聊天和智能分析功能。

## 数据模型

### 请求模型 (request_body.py)

`CreateFactorRequest` - 创建因子请求：

```python
class CreateFactorRequest(BaseModel):
    user_id: str                    # 用户ID
    name: str                       # 因子中文名称
    factor_name: str               # 因子英文名称 (唯一)
    factor_type: str               # 因子类型: future | stock
    is_persistent: bool            # 是否持久化
    cron: Optional[str]            # 定时任务表达式
    factor_start_day: Optional[str] # 持久化开始时间
    code: Text                     # 因子代码
    code_type: str                 # 代码类型: formula | python
    tags: str                      # 因子标签
    status: int                    # 状态: 0未运行, 1运行中, 2成功, 3失败
    describe: str                  # 因子描述
    params: Optional[Params]       # 参数配置
```

## 依赖配置

### 核心依赖

```python
dependencies = [
    "fastapi>=0.95.0",      # Web 框架
    "uvicorn>=0.21.0",      # ASGI 服务器
    "pymongo>=4.3.3",       # MongoDB 驱动
    "pydantic>=1.10.7",     # 数据验证
    "redis>=4.5.4",         # 缓存
    "apscheduler>=3.10.1",  # 任务调度
]
```

### 工作区依赖

```python
"panda-common",    # 通用配置和工具
"panda-data",      # 数据访问层
"panda-factor",    # 因子计算引擎
```

### 数据源依赖

```python
"tqsdk>=3.2.10",   # 天勤量化
"rqdatac>=2.9.0",  # 米筐数据
```

## 配置要求

### 环境变量

- `MONGO_URI` - MongoDB 连接字符串
- `DATASOURCE` - 数据源选择 (tqsdk, tushare, ricequant, xtquant)
- `LLM_API_KEY` - LLM API 密钥
- `LLM_MODEL` - LLM 模型名称
- `LLM_BASE_URL` - LLM 服务地址

### 端口配置

- **API 服务器**: 8111
- **前端访问**: http://localhost:8111/factor
- **API 文档**: http://localhost:8111/docs

## 日志和监控

### 日志配置

服务器支持日志记录到文件和控制台，便于调试和监控：

```python
# 在 server.py 中配置
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler('panda.log')
    ]
)
```

### 请求日志

记录所有 API 请求的响应时间和状态码，用于性能监控。

### 全局异常处理

统一的异常处理机制，确保错误信息的格式一致性和可追踪性。

## 最佳实践

### 1. 开发环境启动

```bash
# 确保依赖已安装
uv sync --all-extras

# 启动服务器
uv run python -m panda_factor_server.__main__
```

### 2. 生产环境部署

- 使用 Docker 容器化部署
- 配置反向代理 (Nginx)
- 设置环境变量管理敏感信息
- 启用日志轮转和监控

### 3. 安全配置

生产环境中应该：
- 限制 CORS 允许的域名
- 添加 API 认证和授权
- 配置 HTTPS
- 设置请求频率限制

### 4. 性能优化

- 启用 Redis 缓存
- 配置数据库连接池
- 使用异步数据库操作
- 实施请求压缩

## 总结

Panda Factor Server 是一个功能完整的量化因子服务后端，通过 FastAPI 提供高性能的 REST API 服务。`__main__.py` 作为启动入口，整合了用户因子管理、LLM 服务和前端静态资源，为量化因子开发和分析提供了完整的后端支持。

通过模块化设计，服务器具有良好的可扩展性和可维护性，支持多种数据源和部署方式，是量化因子系统的重要组成部分。