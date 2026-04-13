# Flow2API 架构设计文档

> 版本: 1.0
> 更新日期: 2026-01-29

## 目录

1. [项目概述](#1-项目概述)
2. [系统架构](#2-系统架构)
3. [目录结构](#3-目录结构)
4. [核心模块详解](#4-核心模块详解)
5. [数据模型](#5-数据模型)
6. [API 接口设计](#6-api-接口设计)
7. [核心流程](#7-核心流程)
8. [配置管理](#8-配置管理)
9. [错误处理机制](#9-错误处理机制)
10. [部署方式](#10-部署方式)
11. [技术栈](#11-技术栈)

---

## 1. 项目概述

### 1.1 项目定位

Flow2API 是一个生产级别的 API 中间层服务，为 Google VideoFX/Gemini 图片和视频生成服务提供 **OpenAI 兼容接口**。

### 1.2 核心功能

- **OpenAI 兼容 API**：标准 `/v1/chat/completions` 接口
- **多 Token 管理**：支持数千个 Token 的负载均衡
- **并发控制**：每个 Token 独立的并发限制
- **自动容错**：429 自动解禁、连续错误自动禁用、AT 自动刷新
- **文件缓存**：生成结果本地缓存，自动清理
- **打码服务**：支持多种第三方验证码服务
- **Web 管理后台**：完整的 Token 和配置管理界面

### 1.3 项目规模

| 指标 | 数值 |
|------|------|
| 总代码行数 | ~8,700 行 Python |
| 核心服务组件 | 12 个 |
| 支持的模型数量 | 80+ 种 |
| 数据库表 | 9 个 |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端请求                               │
│              (OpenAI SDK / HTTP Client / Web UI)                │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      FastAPI 应用层                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ CORS 中间件 │  │  认证验证   │  │      路由处理           │  │
│  └─────────────┘  └─────────────┘  │  - /v1/models          │  │
│                                     │  - /v1/chat/completions│  │
│                                     │  - /api/admin/*        │  │
│                                     └─────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        服务层                                    │
│  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │
│  │ GenerationHandler│  │   TokenManager   │  │ LoadBalancer  │  │
│  │   (生成处理器)    │  │  (Token 管理器)  │  │ (负载均衡器)  │  │
│  └────────┬─────────┘  └────────┬─────────┘  └───────┬───────┘  │
│           │                     │                     │          │
│  ┌────────▼─────────┐  ┌────────▼─────────┐  ┌───────▼───────┐  │
│  │   FlowClient     │  │ConcurrencyManager│  │ ProxyManager  │  │
│  │ (Flow API客户端) │  │   (并发管理器)   │  │  (代理管理器) │  │
│  └────────┬─────────┘  └──────────────────┘  └───────────────┘  │
│           │                                                      │
│  ┌────────▼─────────┐  ┌──────────────────┐                     │
│  │   FileCache      │  │ BrowserCaptcha   │                     │
│  │   (文件缓存)     │  │   (打码服务)     │                     │
│  └──────────────────┘  └──────────────────┘                     │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        核心层                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │   Config     │  │   Database   │  │       Logger         │   │
│  │  (配置管理)  │  │  (数据库层)  │  │     (日志系统)       │   │
│  └──────────────┘  └──────────────┘  └──────────────────────┘   │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                       数据存储层                                 │
│  ┌──────────────────────────┐  ┌─────────────────────────────┐  │
│  │     SQLite (flow.db)     │  │      文件系统 (tmp/)        │  │
│  │  - tokens                │  │  - 缓存的视频/图片文件      │  │
│  │  - projects              │  │  - 浏览器状态数据           │  │
│  │  - token_stats           │  │                             │  │
│  │  - tasks                 │  │                             │  │
│  │  - 配置表 (6个)          │  │                             │  │
│  └──────────────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      外部服务                                    │
│  ┌──────────────────────┐  ┌─────────────────────────────────┐  │
│  │   Google VideoFX     │  │       打码服务提供商            │  │
│  │   - 图片生成 API     │  │  - YesCaptcha                   │  │
│  │   - 视频生成 API     │  │  - CapMonster                   │  │
│  │   - 认证 API         │  │  - EzCaptcha                    │  │
│  └──────────────────────┘  │  - CapSolver                    │  │
│                            └─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心设计原则

1. **异步优先**：基于 FastAPI + aiosqlite + curl_cffi 的全异步架构
2. **单例模式**：核心服务组件全局单例，通过依赖注入传递
3. **热加载**：配置可运行时修改，无需重启服务
4. **容错设计**：多层错误处理和自动恢复机制
5. **可观测性**：完整的日志、统计和请求追踪

---

## 3. 目录结构

```
flow2api/
├── main.py                          # 应用入口点
├── requirements.txt                 # Python 依赖
├── docker-compose.yml               # Docker 编排配置
│
├── config/
│   └── setting.toml                 # TOML 配置文件
│
├── data/
│   └── flow.db                      # SQLite 数据库文件
│
├── src/
│   ├── main.py                      # FastAPI 应用初始化和生命周期
│   │
│   ├── core/                        # 核心层
│   │   ├── config.py               # 配置管理（支持热加载）
│   │   ├── database.py             # 数据库 ORM 层 (1327行)
│   │   ├── models.py               # Pydantic 数据模型
│   │   ├── auth.py                 # 认证和密码管理
│   │   └── logger.py               # 调试日志系统
│   │
│   ├── api/                         # API 层
│   │   ├── routes.py               # OpenAI 兼容 API 路由
│   │   └── admin.py                # 管理后台 API
│   │
│   └── services/                    # 服务层
│       ├── flow_client.py          # Flow API 客户端 (1280行)
│       ├── token_manager.py        # Token 生命周期管理 (591行)
│       ├── generation_handler.py   # 生成处理核心 (1482行)
│       ├── load_balancer.py        # 负载均衡器
│       ├── concurrency_manager.py  # 并发控制管理
│       ├── proxy_manager.py        # 代理配置管理
│       ├── file_cache.py           # 文件缓存服务
│       ├── browser_captcha.py      # 浏览器打码服务
│       └── browser_captcha_personal.py  # 个人浏览器打码
│
├── static/                          # Web 管理界面静态资源
│
├── tmp/                             # 缓存文件目录
│
└── docs/                            # 文档目录
    └── architecture.md             # 本文档
```

---

## 4. 核心模块详解

### 4.1 核心层 (Core)

#### 4.1.1 配置管理 (config.py)

**职责**：管理应用配置，支持文件配置和数据库配置的双层架构

**配置来源**：
```
数据库配置 > 配置文件 (setting.toml) > 硬编码默认值
```

**主要配置项**：

| 配置组 | 说明 |
|--------|------|
| `global` | API Key、管理员账户 |
| `flow` | API 基础 URL、超时、轮询参数 |
| `server` | 监听地址和端口 |
| `debug` | 日志记录控制 |
| `proxy` | 代理配置 |
| `generation` | 图片/视频生成超时 |
| `cache` | 缓存策略 |
| `captcha` | 打码方式配置 |
| `admin` | 错误禁用阈值 |

**热加载机制**：
```python
# 管理员修改配置流程
API 请求 → 更新数据库 → reload_config_to_memory() → 内存更新 → 立即生效
```

#### 4.1.2 数据库层 (database.py)

**职责**：提供异步数据库访问和 ORM 操作

**技术实现**：
- 使用 `aiosqlite` 异步 SQLite 驱动
- 自动数据库迁移（检测并创建缺失的表和列）
- 完整的 CRUD 操作集

**数据库表**：

| 表名 | 说明 |
|------|------|
| `tokens` | Token 存储和管理 |
| `projects` | 项目管理 |
| `token_stats` | 统计信息 |
| `tasks` | 生成任务追踪 |
| `request_logs` | API 请求日志 |
| `admin_config` | 管理配置 |
| `proxy_config` | 代理配置 |
| `generation_config` | 生成超时配置 |
| `cache_config` | 缓存配置 |
| `debug_config` | 调试配置 |
| `captcha_config` | 打码配置 |
| `plugin_config` | 插件配置 |

#### 4.1.3 认证系统 (auth.py)

**两层认证机制**：

1. **API Key 认证**（用户层）
   - HTTPBearer 令牌验证
   - 配置文件中的 `api_key` 字段

2. **管理员认证**（管理层）
   - bcrypt 密码哈希
   - JWT-like Token 验证

#### 4.1.4 日志系统 (logger.py)

**功能特性**：
- 结构化调试日志（JSON 格式 + 时间戳）
- 自动令牌掩码保护
- 大字段截断处理（base64 图片数据）

**日志内容**：
```
[REQUEST]  方法、URL、请求头、请求体、代理信息
[RESPONSE] 状态码、持续时间、响应头、响应体
[ERROR]    错误消息、状态码、错误响应
```

---

### 4.2 服务层 (Services)

#### 4.2.1 Flow API 客户端 (flow_client.py)

**职责**：封装与 Google VideoFX API 的所有交互

**核心功能**：
- HTTP 请求统一处理（GET/POST）
- ST (Cookie) + AT (Bearer) 双认证
- User-Agent 生成（账号级别缓存）
- 代理集成
- 超时控制

**主要 API 方法**：

| 方法 | 说明 |
|------|------|
| `st_to_at()` | ST 转换为 AT（获取过期时间） |
| `get_credits()` | 查询用户余额 |
| `create_project()` | 创建新项目 |
| `get_projects()` | 列出项目 |
| `generate_image()` | 生成图片 |
| `generate_video()` | 生成视频 |
| `get_operation()` | 查询生成状态 |
| `upload_image()` | 上传参考图片 |

#### 4.2.2 Token 管理器 (token_manager.py)

**职责**：Token 生命周期的完整管理

**核心功能**：
- Token 增删改查
- ST 自动转换 AT
- AT 过期检测和自动刷新
- 429 速率限制自动解禁（每小时检查）
- 项目关联管理
- 并发限制配置

**Token 状态流转**：
```
添加 Token (ST)
    │
    ▼
ST → AT 转换
    │
    ▼
正常使用 ◄──────┐
    │           │
    ▼           │
AT 过期 → 自动刷新
    │
    ▼
429 限制 → 自动禁用 → 1小时后自动解禁
    │
    ▼
连续错误 → 自动禁用
```

#### 4.2.3 生成处理器 (generation_handler.py)

**职责**：统一的生成 API 入口点

**核心功能**：
- 模型配置管理（80+ 种模型）
- Token 选择和负载均衡
- 并发限制执行
- 流式响应生成
- 统计信息更新
- 错误处理和自动禁用

**支持的模型类型**：

| 类型 | 说明 | 模型示例 |
|------|------|----------|
| 图片 | 文本生成图片 | Gemini 2.5 Flash, Imagen 4.0 |
| T2V | 文本生成视频 | Veo 3.1, Veo 2.1, Veo 2.0 |
| I2V | 图片生成视频 | 首尾帧模式，1-2张图片 |
| R2V | 多图生成视频 | 支持无限图片 |
| 视频放大 | 分辨率提升 | 4K/1080P |

**模型配置结构**：
```python
{
    "type": "video",           # image / video
    "video_type": "t2v",       # t2v / i2v / r2v / upscale
    "model_key": "veo_3_1_t2v_fast",
    "aspect_ratio": "VIDEO_ASPECT_RATIO_LANDSCAPE",
    "supports_images": False
}
```

#### 4.2.4 负载均衡器 (load_balancer.py)

**算法**：随机选择

**过滤条件**：
1. Token 活跃状态
2. AT 有效性
3. 功能开关（图片/视频）
4. 并发限制

**选择流程**：
```
active_tokens → filter_by_status → filter_by_concurrency → random_select
```

#### 4.2.5 并发管理器 (concurrency_manager.py)

**职责**：为每个 Token 维护独立的并发计数器

**操作方法**：

| 方法 | 说明 |
|------|------|
| `can_use_image/video()` | 检查是否有可用插槽 |
| `acquire_image/video()` | 获取插槽 |
| `release_image/video()` | 释放插槽 |
| `get_remaining()` | 查询剩余插槽 |

**特殊值**：`-1` 表示无限制

#### 4.2.6 代理管理器 (proxy_manager.py)

**职责**：代理配置的 CRUD 和运行时管理

**功能**：
- 代理启用/禁用
- 为 HTTP 客户端提供代理 URL

#### 4.2.7 文件缓存服务 (file_cache.py)

**职责**：生成结果的本地缓存管理

**缓存策略**：
- 文件名：URL 的 MD5 哈希
- 默认超时：2 小时
- 清理间隔：5 分钟

**下载方式**：
- wget（默认）
- curl_cffi（备选）

#### 4.2.8 打码服务 (browser_captcha.py / browser_captcha_personal.py)

**两种模式**：

| 模式 | 实现 | 说明 |
|------|------|------|
| `browser` | Patchright | 无头浏览器 |
| `personal` | Nodriver | 常驻浏览器，持久化状态 |

**支持的打码提供商**：
- YesCaptcha
- CapMonster
- EzCaptcha
- CapSolver

---

## 5. 数据模型

### 5.1 Token 模型

```python
Token(
    id: int,                          # 数据库主键
    st: str,                          # Session Token（唯一）
    at: str,                          # Access Token
    at_expires: datetime,             # AT 过期时间
    email: str,                       # 用户邮箱
    name: str,                        # 用户名
    remark: str,                      # 备注
    is_active: bool = True,           # 活跃状态
    created_at: datetime,             # 创建时间
    last_used_at: datetime,           # 最后使用时间
    use_count: int,                   # 使用次数
    credits: int,                     # 剩余余额
    user_paygate_tier: str,           # 用户等级
    current_project_id: str,          # 当前项目 UUID
    current_project_name: str,        # 项目名称
    image_enabled: bool = True,       # 图片生成开关
    video_enabled: bool = True,       # 视频生成开关
    image_concurrency: int = -1,      # 图片并发限制 (-1=无限制)
    video_concurrency: int = -1,      # 视频并发限制 (-1=无限制)
    ban_reason: str,                  # 禁用原因
    banned_at: datetime               # 禁用时间
)
```

### 5.2 Token 统计模型

```python
TokenStats(
    token_id: int,
    image_count: int,                 # 历史图片生成数
    video_count: int,                 # 历史视频生成数
    error_count: int,                 # 历史错误数
    consecutive_error_count: int,     # 连续错误计数（用于自动禁用）
    today_image_count: int,           # 今日图片生成数
    today_video_count: int,           # 今日视频生成数
    today_error_count: int,           # 今日错误数
    today_date: date,                 # 统计日期
    success_count: int,               # 历史成功次数
    last_success_at: datetime,        # 最后成功时间
    last_error_at: datetime           # 最后错误时间
)
```

### 5.3 项目模型

```python
Project(
    id: int,
    project_id: str,                  # VideoFX 项目 UUID
    token_id: int,                    # 关联的 Token
    project_name: str,                # 项目名称
    tool_name: str = "PINHOLE",       # 工具名（固定值）
    is_active: bool = True,
    created_at: datetime
)
```

### 5.4 任务模型

```python
Task(
    id: int,
    task_id: str,                     # Flow API operation name（唯一）
    token_id: int,
    model: str,
    prompt: str,
    status: str,                      # processing / completed / failed
    progress: int,                    # 0-100
    result_urls: List[str],           # 生成结果 URL 列表
    error_message: str,
    scene_id: str,                    # Flow API scene ID
    created_at: datetime,
    completed_at: datetime
)
```

### 5.5 配置模型

```python
# 管理配置
AdminConfig(
    error_ban_threshold: int = 3,     # 连续错误禁用阈值
    log_requests: bool = False,       # 记录请求日志
    log_responses: bool = False       # 记录响应日志
)

# 代理配置
ProxyConfig(
    enabled: bool = False,
    url: str = ""
)

# 生成超时配置
GenerationConfig(
    image_timeout: int = 120,         # 图片生成超时（秒）
    video_timeout: int = 600          # 视频生成超时（秒）
)

# 缓存配置
CacheConfig(
    enabled: bool = True,
    ttl: int = 7200                   # 缓存时间（秒）
)

# 打码配置
CaptchaConfig(
    mode: str = "none",               # none / browser / personal
    provider: str = "",               # yescaptcha / capmonster 等
    api_key: str = ""
)

# 插件配置
PluginConfig(
    connection_token: str = "",
    auto_enable_on_update: bool = True
)
```

---

## 6. API 接口设计

### 6.1 OpenAI 兼容 API

#### 模型列表

```http
GET /v1/models
Authorization: Bearer <api_key>
```

**响应**：
```json
{
    "object": "list",
    "data": [
        {
            "id": "gemini-2.5-flash-image-landscape",
            "object": "model",
            "owned_by": "flow2api",
            "description": "Image generation - Gemini 2.5 Flash"
        }
    ]
}
```

#### 生成接口

```http
POST /v1/chat/completions
Authorization: Bearer <api_key>
Content-Type: application/json
```

**请求体**：
```json
{
    "model": "gemini-2.5-flash-image-landscape",
    "messages": [
        {
            "role": "user",
            "content": "一只猫"
        }
    ],
    "stream": true
}
```

**多模态请求**（带图片）：
```json
{
    "model": "veo-2.0-i2v-generate",
    "messages": [
        {
            "role": "user",
            "content": [
                {"type": "text", "text": "让这只猫动起来"},
                {"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,..."}}
            ]
        }
    ],
    "stream": true
}
```

**流式响应**：
```
data: {"choices":[{"delta":{"content":"processing..."}}]}
data: {"choices":[{"delta":{"content":"![image](https://...)"}}]}
data: [DONE]
```

### 6.2 管理 API

#### 认证相关

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/admin/login` | 管理员登录 |
| POST | `/api/admin/logout` | 登出 |
| POST | `/api/admin/change-password` | 修改密码 |

#### Token 管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/tokens` | 获取所有 Token |
| POST | `/api/add-token` | 添加 Token |
| POST | `/api/update-token` | 更新 Token |
| POST | `/api/delete-token` | 删除 Token |
| POST | `/api/enable-token` | 启用 Token |
| POST | `/api/disable-token` | 禁用 Token |

#### 配置管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/admin/config` | 获取管理配置 |
| POST | `/api/admin/config/update` | 更新管理配置 |
| POST | `/api/admin/proxy/config` | 代理配置 |
| POST | `/api/admin/generation/config` | 生成超时配置 |
| POST | `/api/admin/cache/config` | 缓存配置 |
| POST | `/api/admin/captcha/config` | 打码配置 |

#### 日志管理

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/admin/logs` | 获取请求日志 |
| POST | `/api/admin/logs/clear` | 清空日志 |

---

## 7. 核心流程

### 7.1 应用启动流程

```
1. 加载配置 (setting.toml)
         │
         ▼
2. 初始化数据库
   ├─ 检查首次启动
   ├─ 创建所有表结构
   └─ 数据库迁移检查
         │
         ▼
3. 从数据库加载配置
   ├─ 管理员账户
   ├─ 缓存设置
   ├─ 生成超时
   ├─ 调试模式
   └─ 打码配置
         │
         ▼
4. 初始化核心服务
   ├─ Database()
   ├─ ProxyManager()
   ├─ FlowClient()
   ├─ TokenManager()
   ├─ LoadBalancer()
   ├─ ConcurrencyManager()
   └─ GenerationHandler()
         │
         ▼
5. 初始化打码服务（如配置）
   ├─ browser 模式: BrowserCaptchaService
   └─ personal 模式: BrowserCaptchaService（持久化）
         │
         ▼
6. 启动后台任务
   ├─ 文件缓存清理任务（5分钟间隔）
   └─ Token 429 自动解禁任务（1小时间隔）
         │
         ▼
7. 服务就绪，监听 0.0.0.0:8000
```

### 7.2 生成请求处理流程

```
用户请求
    │
    ▼
[验证 API Key] → routes.create_chat_completion()
    │
    ▼
[解析请求] → 提取模型、提示词、图片
    │
    ▼
[选择 Token] → load_balancer.select_token()
    ├─ 获取活跃 Token 列表
    ├─ 检查 AT 有效性
    ├─ 检查功能开关
    └─ 检查并发限制
    │
    ▼
[获取并发许可] → concurrency_manager.acquire_image/video()
    │
    ▼
[发送生成请求] → flow_client.generate_image/video()
    ├─ 构建请求体
    ├─ 应用代理
    └─ 打码处理（如需要）
    │
    ▼
[轮询结果] → flow_client.get_operation()
    ├─ 间隔时间: poll_interval（默认 3 秒）
    ├─ 最大次数: max_poll_attempts（默认 200 次）
    └─ 超时时间: image_timeout / video_timeout
    │
    ▼
[缓存结果] → file_cache.download_and_cache()
    │
    ▼
[更新统计] → database.increment_*_count()
    ├─ 生成计数 +1
    ├─ 成功计数 +1
    └─ 连续错误计数 = 0
    │
    ▼
[流式返回] → StreamingResponse (SSE)
    │
    ▼
[释放并发] → concurrency_manager.release_image/video()
```

### 7.3 Token 刷新流程

```
定时检查 / 请求触发
         │
         ▼
[检查 AT 有效性] → token_manager.is_at_valid()
    ├─ 检查 AT 是否存在
    └─ 检查是否过期
         │
         ▼ (如果过期)
[刷新 AT] → token_manager.refresh_at()
    ├─ 调用 flow_client.st_to_at()
    ├─ 获取新 AT 和过期时间
    └─ 更新数据库
         │
         ▼ (如果 ST 也过期，personal 模式)
[浏览器打码]
    ├─ 自动登录
    ├─ 获取新 ST
    └─ 更新数据库
```

---

## 8. 配置管理

### 8.1 配置文件示例 (setting.toml)

```toml
[global]
api_key = "your-api-key"

[flow]
base_url = "https://labs.google/fx/api"
timeout = 30
poll_interval = 3
max_poll_attempts = 200

[server]
host = "0.0.0.0"
port = 8000

[debug]
enabled = false
log_requests = false
log_responses = false
mask_token = true

[proxy]
enabled = false
url = ""

[generation]
image_timeout = 120
video_timeout = 600

[admin]
username = "admin"
password = "your-password"
error_ban_threshold = 3

[cache]
enabled = true
ttl = 7200

[captcha]
mode = "none"
provider = ""
api_key = ""
```

### 8.2 配置热加载

配置修改后立即生效，无需重启服务：

```python
# 在管理 API 中
async def update_config(new_config):
    await db.update_config(new_config)       # 持久化到数据库
    await config.reload_config_to_memory()   # 重新加载到内存
    # 配置立即生效
```

---

## 9. 错误处理机制

### 9.1 错误分类

| 错误类型 | 处理方式 |
|----------|----------|
| 429 速率限制 | 禁用 Token，1小时后自动解禁 |
| 连续错误 | 达到阈值（默认3次）后自动禁用 |
| AT 过期 | 自动刷新 |
| 网络错误 | 记录日志，返回 500 |
| 参数错误 | 返回 400 |

### 9.2 429 处理流程

```
检测到 429
    │
    ▼
记录禁用原因: ban_reason = "429_rate_limit"
记录禁用时间: banned_at = now()
    │
    ▼
禁用 Token: is_active = False
    │
    ▼
后台任务（每小时）
    │
    ▼
检查 banned_at 是否超过 1 小时
    │
    ▼ (是)
清除禁用标记
启用 Token: is_active = True
```

### 9.3 连续错误禁用

```
请求失败
    │
    ▼
consecutive_error_count += 1
    │
    ▼
检查: consecutive_error_count >= error_ban_threshold?
    │
    ▼ (是)
禁用 Token
记录: ban_reason = "consecutive_errors"
    │
    ▼
(请求成功时重置)
consecutive_error_count = 0
```

### 9.4 HTTP 错误码

| 状态码 | 说明 |
|--------|------|
| 401 | API Key 无效或过期 |
| 400 | 请求格式错误（模型不存在、提示词为空） |
| 429 | 所有 Token 都不可用或并发已满 |
| 500 | 服务器错误（Flow API 失败、数据库错误） |

---

## 10. 部署方式

### 10.1 Docker 部署（推荐）

```yaml
# docker-compose.yml
version: '3.8'
services:
  flow2api:
    image: ghcr.io/thesmallhancat/flow2api:latest
    ports:
      - "38000:8000"
    volumes:
      - ./data:/app/data
      - ./config/setting.toml:/app/config/setting.toml
    environment:
      - PYTHONUNBUFFERED=1
    restart: unless-stopped
```

**启动命令**：
```bash
docker-compose up -d
```

### 10.2 本地部署

```bash
# 1. 克隆代码
git clone https://github.com/TheSmallHanCat/flow2api.git
cd flow2api

# 2. 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Linux/Mac
# 或 venv\Scripts\activate  # Windows

# 3. 安装依赖
pip install -r requirements.txt

# 4. 配置
cp config/setting.toml.example config/setting.toml
# 编辑 setting.toml

# 5. 启动
python main.py
```

### 10.3 访问服务

- **API 端点**: `http://localhost:8000/v1/chat/completions`
- **管理后台**: `http://localhost:8000/`

---

## 11. 技术栈

### 11.1 核心依赖

| 依赖 | 版本 | 说明 |
|------|------|------|
| FastAPI | 0.119.0 | Web 框架 |
| Uvicorn | 0.32.1 | ASGI 服务器 |
| Pydantic | 2.10.4 | 数据验证 |
| aiosqlite | 0.20.0 | 异步 SQLite 驱动 |
| bcrypt | 4.2.1 | 密码哈希 |
| curl_cffi | 0.7.3 | HTTP 客户端（浏览器伪装） |
| python-multipart | 0.0.20 | 多部分表单数据 |
| tomli | 2.2.1 | TOML 配置解析 |

### 11.2 可选依赖

| 依赖 | 版本 | 说明 |
|------|------|------|
| patchright | >= 0.10.0 | 无头浏览器（打码） |
| nodriver | >= 0.48.0 | 常驻浏览器（打码） |
| python-dateutil | 2.8.2 | 日期时间处理 |

### 11.3 架构特点总结

1. **全异步架构**：FastAPI + aiosqlite + curl_cffi
2. **多 Token 支持**：负载均衡 + 并发控制
3. **容错能力**：429 自动解禁、连续错误自动禁用、AT 自动刷新
4. **热加载配置**：运行时修改，立即生效
5. **完整可观测性**：详细日志、统计信息、请求追踪
6. **OpenAI 兼容**：标准 API 接口，易于集成
7. **高效缓存**：文件本地缓存，自动清理
8. **灵活打码**：支持多种第三方服务

---

## 附录

### A. 数据库 ER 图

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   tokens     │────<│   projects   │     │    tasks     │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id (PK)      │     │ id (PK)      │     │ id (PK)      │
│ st           │     │ project_id   │     │ task_id      │
│ at           │     │ token_id(FK) │     │ token_id(FK) │
│ email        │     │ project_name │     │ model        │
│ is_active    │     │ tool_name    │     │ prompt       │
│ ...          │     │ ...          │     │ status       │
└──────────────┘     └──────────────┘     │ ...          │
       │                                   └──────────────┘
       │
       ▼
┌──────────────┐
│ token_stats  │
├──────────────┤
│ token_id(FK) │
│ image_count  │
│ video_count  │
│ error_count  │
│ ...          │
└──────────────┘
```

### B. 支持的模型列表（部分）

**图片生成**：
- `gemini-2.5-flash-image-*`（landscape/portrait/square）
- `gemini-3.0-pro-image-*`
- `imagen-4.0-*`

**视频生成 (T2V)**：
- `veo-3.1-*-generate`（fast/standard）
- `veo-2.1-*-generate`
- `veo-2.0-*-generate`

**视频生成 (I2V)**：
- `veo-2.0-*-i2v-generate`

**视频放大**：
- `veo-2.0-*-upscale`

---

*文档结束*
