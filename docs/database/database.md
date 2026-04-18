# Flow2API 数据库设计文档

> 版本: 1.0
> 更新日期: 2026-04-17
> 数据库文件: `data/flow.db`

## 目录

1. [技术栈与总体说明](#1-技术栈与总体说明)
2. [数据库初始化与迁移](#2-数据库初始化与迁移)
3. [表结构一览](#3-表结构一览)
4. [核心业务表](#4-核心业务表)
5. [配置表（单行配置）](#5-配置表单行配置)
6. [索引与约束](#6-索引与约束)
7. [实体关系（ER）](#7-实体关系er)
8. [关键业务流程中的数据操作](#8-关键业务流程中的数据操作)
9. [连接参数与并发策略](#9-连接参数与并发策略)

---

## 1. 技术栈与总体说明

| 项目 | 内容 |
|------|------|
| 数据库引擎 | SQLite 3 |
| 异步驱动 | `aiosqlite`（async/await） |
| 数据模型 | Pydantic `BaseModel`（序列化层，非 ORM） |
| 访问入口 | `src/core/database.py` 中的 `Database` 类 |
| 模型定义 | `src/core/models.py` |
| 数据库文件 | `data/flow.db`（启动时自动创建） |
| 日志模式 | WAL（Write-Ahead Logging） |
| 外键约束 | `PRAGMA foreign_keys = ON` 启用 |
| 忙超时 | 30000ms（30 秒） |

设计要点：

- 整个系统 **没有使用 ORM**：表结构通过 `CREATE TABLE IF NOT EXISTS` 原生 SQL 声明，Python 层用 Pydantic 模型做字段校验与序列化。
- 所有配置类表都是 **单行配置表**（`id INTEGER PRIMARY KEY DEFAULT 1`），通过 `id=1` 保证全局唯一。
- 业务核心围绕 `tokens` 表展开，其余业务表均通过 `token_id` 外键关联。

---

## 2. 数据库初始化与迁移

### 2.1 初始化流程

应用启动时会顺序执行：

1. `Database.__init__()`：确保 `data/` 目录存在。
2. `init_db()`：执行全部 `CREATE TABLE IF NOT EXISTS` 语句，创建索引。
3. `check_and_migrate_db(config_dict)`：按需执行列/表结构迁移，写入配置默认值。

### 2.2 迁移机制

项目 **没有使用 Alembic 等迁移工具**，而是代码驱动的轻量迁移：

- **加列迁移**：通过 `ALTER TABLE ... ADD COLUMN` 在启动时补齐新增字段，实现版本间向后兼容升级。
- **结构迁移**：对 `request_logs` 等历史 schema 变化较大的表，走单独的 `_migrate_request_logs` 路径，按需重建。
- **默认行写入**：所有单行配置表启动时会写入一行 `id=1` 默认数据，确保读取不会因空表报错。

---

## 3. 表结构一览

全库共 **13 张表**，分为两类：

| 分类 | 表名 | 说明 |
|------|------|------|
| 业务表 | `tokens` | 账户池核心表（ST/AT、并发、启停状态） |
| 业务表 | `projects` | VideoFX 项目池（Token 级别） |
| 业务表 | `token_stats` | Token 使用统计（累计 + 当日） |
| 业务表 | `tasks` | 图片/视频生成任务元数据 |
| 业务表 | `request_logs` | API 请求审计日志 |
| 配置表 | `admin_config` | 管理员账号、API Key、禁用阈值 |
| 配置表 | `proxy_config` | 请求代理与媒体代理 |
| 配置表 | `generation_config` | 生成超时与重试 |
| 配置表 | `call_logic_config` | Token 调用策略（随机/轮询） |
| 配置表 | `cache_config` | 生成结果缓存配置 |
| 配置表 | `debug_config` | 调试日志开关 |
| 配置表 | `captcha_config` | 打码服务配置 |
| 配置表 | `plugin_config` | 浏览器插件连接配置 |

---

## 4. 核心业务表

### 4.1 `tokens` — 账户池核心表

存储 Google VideoFX/Gemini 账户的认证凭证与运行时状态，是整个系统的核心。

| 字段 | 类型 | 约束 / 默认 | 说明 |
|------|------|-------------|------|
| `id` | INTEGER | PK AUTOINCREMENT | Token ID |
| `st` | TEXT | UNIQUE NOT NULL | Session Token（`__Secure-next-auth.session-token`） |
| `at` | TEXT | NULL | Access Token（由 ST 换取，用于 API 调用） |
| `at_expires` | TIMESTAMP | NULL | AT 过期时间 |
| `email` | TEXT | NOT NULL | Google 账号邮箱 |
| `name` | TEXT | NULL | 用户显示名称 |
| `remark` | TEXT | NULL | 管理员备注 |
| `is_active` | BOOLEAN | 1 | 是否激活（0 禁用 / 1 激活） |
| `created_at` | TIMESTAMP | CURRENT_TIMESTAMP | 创建时间 |
| `last_used_at` | TIMESTAMP | NULL | 最近一次被选中时间 |
| `use_count` | INTEGER | 0 | 累计使用次数 |
| `credits` | INTEGER | 0 | 剩余点数（VideoFX Credits） |
| `user_paygate_tier` | TEXT | NULL | 账号等级（如 `PAYGATE_TIER_ONE`） |
| `current_project_id` | TEXT | NULL | 当前使用的项目 UUID |
| `current_project_name` | TEXT | NULL | 当前项目名称 |
| `image_enabled` | BOOLEAN | 1 | 图片生成功能开关 |
| `video_enabled` | BOOLEAN | 1 | 视频生成功能开关 |
| `image_concurrency` | INTEGER | -1 | 图片并发上限（-1 无限制） |
| `video_concurrency` | INTEGER | -1 | 视频并发上限（-1 无限制） |
| `captcha_proxy_url` | TEXT | NULL | Token 级打码代理（覆盖全局） |
| `ban_reason` | TEXT | NULL | 禁用原因（如 `429_rate_limit`） |
| `banned_at` | TIMESTAMP | NULL | 禁用时间戳 |

**关键约束**：`st` 全局唯一；被 `projects`、`token_stats`、`tasks`、`request_logs` 引用。

### 4.2 `projects` — 项目池

每个 Token 可以绑定多个 VideoFX 项目，用于浏览器打码场景下的项目隔离。

| 字段 | 类型 | 约束 / 默认 | 说明 |
|------|------|-------------|------|
| `id` | INTEGER | PK AUTOINCREMENT | 记录 ID |
| `project_id` | TEXT | UNIQUE NOT NULL | Google VideoFX 项目 UUID |
| `token_id` | INTEGER | NOT NULL，FK → `tokens.id` | 所属 Token |
| `project_name` | TEXT | NOT NULL | 项目名称 |
| `tool_name` | TEXT | 'PINHOLE' | 工具名（通常固定） |
| `is_active` | BOOLEAN | 1 | 是否可用 |
| `created_at` | TIMESTAMP | CURRENT_TIMESTAMP | 创建时间 |

**关系**：`tokens 1 — N projects`。

### 4.3 `token_stats` — Token 使用统计

记录每个 Token 的累计 / 当日统计与连续错误状态，用于负载均衡和自动禁用判定。

| 字段 | 类型 | 约束 / 默认 | 说明 |
|------|------|-------------|------|
| `id` | INTEGER | PK AUTOINCREMENT | 记录 ID |
| `token_id` | INTEGER | NOT NULL，FK → `tokens.id` | 所属 Token |
| `image_count` | INTEGER | 0 | 累计图片生成次数 |
| `video_count` | INTEGER | 0 | 累计视频生成次数 |
| `success_count` | INTEGER | 0 | 累计成功请求数 |
| `error_count` | INTEGER | 0 | 累计错误数（永不重置） |
| `last_success_at` | TIMESTAMP | NULL | 上次成功时间 |
| `last_error_at` | TIMESTAMP | NULL | 上次出错时间 |
| `today_image_count` | INTEGER | 0 | 当日图片数（每日重置） |
| `today_video_count` | INTEGER | 0 | 当日视频数（每日重置） |
| `today_error_count` | INTEGER | 0 | 当日错误数（每日重置） |
| `today_date` | DATE | NULL | 当日日期（判断是否需要重置） |
| `consecutive_error_count` | INTEGER | 0 | 连续错误计数（触发自动禁用） |

**关系**：`tokens 1 — 1 token_stats`（Token 新增时自动创建对应记录）。

### 4.4 `tasks` — 生成任务

图片/视频生成任务的状态与结果持久化表。

| 字段 | 类型 | 约束 / 默认 | 说明 |
|------|------|-------------|------|
| `id` | INTEGER | PK AUTOINCREMENT | 记录 ID |
| `task_id` | TEXT | UNIQUE NOT NULL | Flow API 返回的 operation name |
| `token_id` | INTEGER | NOT NULL，FK → `tokens.id` | 执行任务的 Token |
| `model` | TEXT | NOT NULL | 使用的模型名 |
| `prompt` | TEXT | NOT NULL | 生成提示词 |
| `status` | TEXT | 'processing' | `processing` / `completed` / `failed` |
| `progress` | INTEGER | 0 | 进度百分比（0–100） |
| `result_urls` | TEXT | NULL | 结果 URL 列表（JSON 数组） |
| `error_message` | TEXT | NULL | 错误信息 |
| `scene_id` | TEXT | NULL | Flow API 的 sceneId |
| `created_at` | TIMESTAMP | CURRENT_TIMESTAMP | 创建时间 |
| `completed_at` | TIMESTAMP | NULL | 完成时间 |

### 4.5 `request_logs` — 请求日志

所有 API 请求的审计日志，用于监控、调试与故障排查。

| 字段 | 类型 | 约束 / 默认 | 说明 |
|------|------|-------------|------|
| `id` | INTEGER | PK AUTOINCREMENT | 记录 ID |
| `token_id` | INTEGER | FK → `tokens.id`（可为 NULL） | 发起请求的 Token |
| `operation` | TEXT | NOT NULL | 操作类型（如 `text2image`、`video_generation`） |
| `request_body` | TEXT | NULL | 原始请求体（JSON，可被掩码） |
| `response_body` | TEXT | NULL | 响应体摘要（长度受限） |
| `status_code` | INTEGER | NOT NULL | HTTP 状态码 |
| `duration` | FLOAT | NOT NULL | 耗时（秒） |
| `status_text` | TEXT | '' | 文本描述（`completed`/`failed`/…） |
| `progress` | INTEGER | 0 | 进度百分比 |
| `created_at` | TIMESTAMP | CURRENT_TIMESTAMP | 请求创建时间 |
| `updated_at` | TIMESTAMP | CURRENT_TIMESTAMP | 最近更新时间 |

> `token_id` 允许为 NULL，用于记录未绑定具体 Token 的请求（如登录失败、系统级调用）。

---

## 5. 配置表（单行配置）

以下 8 张配置表均采用 **`id INTEGER PRIMARY KEY DEFAULT 1`** 的单行模式，读写接口通过 `WHERE id=1` 访问。运行时修改后：

- 数据库持久化最新值；
- 内存 `Config` 对象同步更新，支持热加载；
- 读取优先级：**数据库 > `config/setting.toml` > 硬编码默认值**。

### 5.1 `admin_config` — 管理员与全局阈值

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `id` | INTEGER | 1 | 固定 1 |
| `username` | TEXT | 'admin' | 管理员用户名 |
| `password` | TEXT | 'admin' | 管理员密码（哈希存储） |
| `api_key` | TEXT | 'han1234' | 对外 API Key |
| `error_ban_threshold` | INTEGER | 3 | 自动禁用连续错误阈值 |
| `updated_at` | TIMESTAMP | CURRENT_TIMESTAMP | 更新时间 |

### 5.2 `proxy_config` — 代理配置

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `id` | INTEGER | 1 | 固定 1 |
| `enabled` | BOOLEAN | 0 | 请求代理开关 |
| `proxy_url` | TEXT | NULL | HTTP/SOCKS 代理地址 |
| `media_proxy_enabled` | BOOLEAN | 0 | 媒体代理开关 |
| `media_proxy_url` | TEXT | NULL | 媒体代理地址 |
| `updated_at` | TIMESTAMP | CURRENT_TIMESTAMP | 更新时间 |

### 5.3 `generation_config` — 生成超时与重试

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `id` | INTEGER | 1 | 固定 1 |
| `image_timeout` | INTEGER | 300 | 图片超时（秒） |
| `video_timeout` | INTEGER | 1500 | 视频超时（秒） |
| `max_retries` | INTEGER | 3 | 最大重试次数 |
| `updated_at` | TIMESTAMP | CURRENT_TIMESTAMP | 更新时间 |

### 5.4 `call_logic_config` — 调用策略

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `id` | INTEGER | 1 | 固定 1 |
| `call_mode` | TEXT | 'default' | `default` 随机轮询 / `polling` 顺序轮询 |
| `polling_mode_enabled` | BOOLEAN | 0 | 轮询模式开关 |
| `updated_at` | TIMESTAMP | CURRENT_TIMESTAMP | 更新时间 |

### 5.5 `cache_config` — 结果缓存

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `id` | INTEGER | 1 | 固定 1 |
| `cache_enabled` | BOOLEAN | 0 | 缓存开关 |
| `cache_timeout` | INTEGER | 7200 | 缓存有效期（秒，0 表示永不过期） |
| `cache_base_url` | TEXT | NULL | 缓存文件访问基础 URL |
| `created_at` | TIMESTAMP | CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | TIMESTAMP | CURRENT_TIMESTAMP | 更新时间 |

> 缓存文件本体落在 `tmp/` 目录，数据库仅存配置元数据。

### 5.6 `debug_config` — 调试日志

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `id` | INTEGER | 1 | 固定 1 |
| `enabled` | BOOLEAN | 0 | 调试模式总开关 |
| `log_requests` | BOOLEAN | 1 | 是否记录请求体 |
| `log_responses` | BOOLEAN | 1 | 是否记录响应体 |
| `mask_token` | BOOLEAN | 1 | 日志中是否掩码 Token |
| `created_at` | TIMESTAMP | CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | TIMESTAMP | CURRENT_TIMESTAMP | 更新时间 |

### 5.7 `captcha_config` — 打码服务

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `id` | INTEGER | 1 | 固定 1 |
| `captcha_method` | TEXT | 'browser' | `yescaptcha`/`capmonster`/`ezcaptcha`/`capsolver`/`browser`/`personal`/`remote_browser` |
| `yescaptcha_api_key` | TEXT | '' | YesCaptcha API Key |
| `yescaptcha_base_url` | TEXT | 'https://api.yescaptcha.com' | YesCaptcha 基础 URL |
| `capmonster_api_key` | TEXT | '' | CapMonster API Key |
| `capmonster_base_url` | TEXT | 'https://api.capmonster.cloud' | CapMonster 基础 URL |
| `ezcaptcha_api_key` | TEXT | '' | EzCaptcha API Key |
| `ezcaptcha_base_url` | TEXT | 'https://api.ez-captcha.com' | EzCaptcha 基础 URL |
| `capsolver_api_key` | TEXT | '' | CapSolver API Key |
| `capsolver_base_url` | TEXT | 'https://api.capsolver.com' | CapSolver 基础 URL |
| `remote_browser_base_url` | TEXT | '' | 远程浏览器服务地址 |
| `remote_browser_api_key` | TEXT | '' | 远程浏览器服务 Key |
| `remote_browser_timeout` | INTEGER | 60 | 远程浏览器超时（秒） |
| `website_key` | TEXT | `6LdsFiUsAAA...` | reCAPTCHA website key |
| `page_action` | TEXT | 'IMAGE_GENERATION' | 页面操作名 |
| `browser_proxy_enabled` | BOOLEAN | 0 | 浏览器打码代理开关 |
| `browser_proxy_url` | TEXT | NULL | 浏览器打码代理 URL |
| `browser_count` | INTEGER | 1 | 浏览器打码实例数 |
| `personal_project_pool_size` | INTEGER | 4 | 单 Token 维护的项目池大小 |
| `personal_max_resident_tabs` | INTEGER | 5 | 内置浏览器共享 Tab 上限 |
| `personal_idle_tab_ttl_seconds` | INTEGER | 600 | 空闲 Tab 超时（秒） |
| `created_at` | TIMESTAMP | CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | TIMESTAMP | CURRENT_TIMESTAMP | 更新时间 |

### 5.8 `plugin_config` — 浏览器插件

| 字段 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `id` | INTEGER | 1 | 固定 1 |
| `connection_token` | TEXT | '' | 插件连接 Token |
| `auto_enable_on_update` | BOOLEAN | 1 | 更新 Token 时是否自动启用 |
| `created_at` | TIMESTAMP | CURRENT_TIMESTAMP | 创建时间 |
| `updated_at` | TIMESTAMP | CURRENT_TIMESTAMP | 更新时间 |

---

## 6. 索引与约束

### 6.1 唯一约束

| 表 | 字段 | 作用 |
|----|------|------|
| `tokens` | `st` | 避免同一 Session Token 重复入库 |
| `projects` | `project_id` | VideoFX 项目 UUID 全局唯一 |
| `tasks` | `task_id` | Flow API operation 唯一 |

### 6.2 外键（声明在 DDL 中；运行时 `PRAGMA foreign_keys=ON`）

| 子表 | 字段 | 指向 |
|------|------|------|
| `projects.token_id` | INTEGER | `tokens.id` |
| `token_stats.token_id` | INTEGER | `tokens.id` |
| `tasks.token_id` | INTEGER | `tokens.id` |
| `request_logs.token_id` | INTEGER（可空） | `tokens.id` |

### 6.3 性能索引

| 索引名 | 表 | 字段 | 用途 |
|--------|----|------|------|
| `idx_token_st` | `tokens` | `st` | 按 ST 定位 Token |
| `idx_tokens_email` | `tokens` | `email` | 邮箱查询 |
| `idx_tokens_is_active_last_used_at` | `tokens` | `(is_active, last_used_at)` | **负载均衡核心索引**：找"激活且最久未用"的 Token |
| `idx_project_id` | `projects` | `project_id` | 按项目 UUID 查询 |
| `idx_token_stats_token_id` | `token_stats` | `token_id` | 统计联查 |
| `idx_task_id` | `tasks` | `task_id` | 按任务 ID 查询 |
| `idx_request_logs_created_at` | `request_logs` | `created_at DESC` | 时间线分页 |
| `idx_request_logs_token_id_created_at` | `request_logs` | `(token_id, created_at DESC)` | 单 Token 日志分页 |

---

## 7. 实体关系（ER）

```
                       ┌──────────────┐
                       │    tokens    │  (核心表)
                       └──────┬───────┘
                              │
        ┌──────────────┬──────┼──────┬────────────────┐
        │ 1:1          │ 1:N  │ 1:N  │ 1:N (可空)
        ▼              ▼             ▼                ▼
  ┌───────────┐ ┌────────────┐ ┌─────────┐ ┌────────────────┐
  │token_stats│ │  projects  │ │  tasks  │ │  request_logs  │
  └───────────┘ └────────────┘ └─────────┘ └────────────────┘

配置表（全部独立，单行 id=1）：
  admin_config / proxy_config / generation_config / call_logic_config /
  cache_config / debug_config / captcha_config / plugin_config
```

- `tokens` 是唯一的业务中心，其余业务表全部通过 `token_id` 归属到某个 Token。
- 配置表之间无关联，彼此独立。
- 删除 Token 时，业务侧会联动清理 `token_stats`、`projects`、`tasks`，并将 `request_logs.token_id` 置为历史 NULL（保留日志可审计）。

---

## 8. 关键业务流程中的数据操作

### 8.1 添加 Token

```
INSERT tokens(st, email, ...)     -- 新行，is_active=1
  → 同事务 INSERT token_stats(token_id=新 id)   -- 初始化统计
```

### 8.2 负载均衡选择 Token

命中 `idx_tokens_is_active_last_used_at`：

```sql
SELECT * FROM tokens
WHERE is_active = 1
  AND (image_enabled = 1 或 video_enabled = 1)
ORDER BY last_used_at ASC NULLS FIRST
LIMIT N;
```

选中后 `UPDATE tokens SET last_used_at=..., use_count=use_count+1`。

### 8.3 生成任务生命周期

```
INSERT tasks(status='processing', progress=0, token_id=?)
  → 长轮询 Flow API
  → UPDATE tasks SET progress=..., status='completed'|'failed', result_urls=..., completed_at=now
  → UPDATE token_stats SET image_count/video_count 累加、today_* 累加
```

### 8.4 错误与自动禁用

```
请求失败 → UPDATE token_stats
  SET error_count+=1, consecutive_error_count+=1, today_error_count+=1

若 consecutive_error_count >= admin_config.error_ban_threshold
  → UPDATE tokens SET is_active=0, ban_reason='xxx', banned_at=now
```

请求成功或手动启用时 `consecutive_error_count` 重置为 0。

### 8.5 每日计数重置

每次写入统计前比对 `token_stats.today_date`：若不等于当天，则先把 `today_image_count / today_video_count / today_error_count` 清零并更新 `today_date`。

---

## 9. 连接参数与并发策略

| 项目 | 值 / 说明 |
|------|----------|
| 连接超时 | 30 秒 |
| 忙超时 | `PRAGMA busy_timeout = 30000` |
| 日志模式 | `PRAGMA journal_mode = WAL` |
| 同步级别 | `PRAGMA synchronous = NORMAL` |
| 外键 | `PRAGMA foreign_keys = ON` |
| 写并发 | Python 侧 `asyncio.Lock()` 串行化写入 |
| 读并发 | WAL 下多读并发，无显式锁 |
| 提交策略 | 显式 `await db.commit()`，无自动提交 |

> SQLite 在 WAL 模式下能较好支撑读多写少的场景；本项目"写"主要集中在 `request_logs`、`token_stats`，通过 `asyncio.Lock` + 忙超时保证不冲突。

---

## 附：相关路径

| 路径 | 说明 |
|------|------|
| `data/flow.db` | SQLite 数据库文件 |
| `src/core/database.py` | DDL、迁移与 CRUD 实现 |
| `src/core/models.py` | Pydantic 数据模型 |
| `config/setting.toml` | 初始配置来源（数据库未初始化时的默认值） |
| `tmp/` | 生成结果缓存与浏览器状态文件（非数据库管理） |
