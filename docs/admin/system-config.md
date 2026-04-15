# 系统配置详解（Admin → 系统配置）

本文档对应管理后台 `manage.html` 的「系统配置」标签页（截图见 `docs/uid-151.png`），逐项解释每个配置项的含义、数据存放位置、保存与生效链路，以及在核心业务流程中的使用方式。

> 关键点：**配置以 SQLite 为真源（source of truth）**，`config/setting.toml` 只作为首次启动的默认值来源；保存后通过 `db.reload_config_to_memory()` 热加载到内存单例 `config`，大部分无需重启。

---

## 架构概览

```
[前端 manage.html]
      │ HTTP (Bearer 会话 Token)
      ▼
[FastAPI src/api/admin.py]
      │ 写 SQLite
      ▼
[src/core/database.py]
      │ reload_config_to_memory()
      ▼
[内存单例 src/core/config.py :: Config]
      │
      ▼
[业务模块: flow_client / token_manager / generation_handler / captcha / cache / proxy_manager]
```

关键文件：

| 文件 | 作用 |
|---|---|
| `src/core/config.py` | 内存 `Config` 单例，所有字段的 getter/setter |
| `src/core/models.py` | 各配置表的 Pydantic 模型（AdminConfig / ProxyConfig / CaptchaConfig …） |
| `src/core/database.py` | SQLite 读写、迁移、`reload_config_to_memory()` |
| `src/api/admin.py` | 后台全部路由（登录、配置读写、Token 管理、日志） |
| `src/services/proxy_manager.py` | 代理 URL 解析与归一化 |
| `src/services/browser_captcha_personal.py` | 内置浏览器验证码服务 |
| `static/manage.html` | 后台单页 UI |
| `config/setting.toml` | 初次启动默认值 |

鉴权：

- **管理员会话**：`POST /api/admin/login` 成功后生成 `admin-<token_urlsafe(32)>`，保存在 `admin.py` 内存集合 `active_admin_tokens`（`admin.py:36`）。所有后台接口通过 `Authorization: Bearer <token>` 验证（`admin.py:562-573`）。改密时会 `clear()` 全部会话，强制重新登录（`admin.py:629`）。
- **API Key**：与会话完全独立，供外部客户端调用业务接口时使用，见下文「API 密钥配置」。

---

## 1. 安全配置

**UI 字段**：管理员用户名（默认 `admin`）、旧密码、新密码。

**数据存放**：
- 表：`admin_config(username, password, api_key, error_ban_threshold)`
- 模型：`AdminConfig`（`src/core/models.py:115`）
- 默认值：`config/setting.toml` 第 3–4 行（`admin / admin`）

**接口**：
- `POST /api/admin/login` → `LoginRequest(username, password)`，返回会话 Token（`admin.py:578-596`）
- `POST /api/admin/logout`（`admin.py:599-603`）
- `POST /api/admin/change-password` → `ChangePasswordRequest(username?, old_password, new_password)`（`admin.py:606-631`），别名 `POST /api/admin/password`（`admin.py:1340-1346`）

**校验/安全**：
- 密码使用 bcrypt 哈希（`src/core/auth.py :: AuthManager.verify_admin`）
- 改密成功后：写库 → `reload_config_to_memory()` → 清空全部会话 Token

**业务使用**：仅后台登录鉴权，不参与请求代理业务。

---

## 2. API 密钥配置

**UI 字段**：当前 API Key、Bi API Key、「更新 API Key」。

> 说明：截图中「当前 API Key」与「Bi API Key」两个输入框，从当前代码看 `admin_config.api_key` 为单一字段；如确需双 Key，可能是前端保留位/额外实现，需结合最新代码确认（见文末「待核对」）。

**数据存放**：`admin_config.api_key`，默认 `han1234`（`config/setting.toml:2`）。

**接口**：
- `GET /api/admin/config`（`admin.py:1315-1325`）
- `POST /api/admin/apikey` → `UpdateAPIKeyRequest(new_api_key)`（`admin.py:1349-1361`）

**业务使用**：外部调用 `/v1/*`、`/images`、`/videos` 等业务路由时，通过请求头 `X-API-Key` 或查询参数 `api_key` 进行校验（路由实现见 `src/api/routes.py`）。

**热加载**：写库后 `reload_config_to_memory()` → `config.api_key` 更新（`config.py:228-233`）。

---

## 3. 代理配置

**UI 字段**：
- 「启用请求代理」+ 「请求代理地址」（例 `http://127.0.0.1:7890`、`socks5://127.0.0.1:1080`）
- 「媒体上传下载代理」（独立开关/地址）
- 「测试代理」按钮

**数据存放**：
- 表：`proxy_config(enabled, proxy_url, media_proxy_enabled, media_proxy_url)`
- 模型：`ProxyConfig`（`src/core/models.py:125`）
- 管理器：`ProxyManager`（`src/services/proxy_manager.py:7-149`）

**接口**：
- `GET/POST /api/config/proxy`（`admin.py:1001-1061`）及别名 `GET/POST /api/proxy/config`（`admin.py:1016-1043`）
- `POST /api/proxy/test` → `ProxyTestRequest(proxy_url, test_url?, timeout_seconds?)`（`admin.py:1064-1124`）
  - 使用 `curl_cffi.AsyncSession` + `impersonate="chrome120"`，默认请求 `https://labs.google/`，超时 5–60s（默认 15s），返回 `elapsed_ms / final_url / status_code`

**支持的代理格式**（`ProxyManager._parse_proxy_line`）：
- `http://user:pass@host:port`、`https://…`、`socks5://…`、`socks5h://…`
- `host:port`（自动补 `http://`）
- `host:port:user:pass`（转成带鉴权的 http）
- `st5 host:port:user:pass`（SOCKS5 简写）

**业务使用**：
- 请求代理（Google Labs 接口）：`FlowClient` 通过 `await proxy_manager.get_request_proxy_url()` 获取，注入 `curl_cffi` 请求
- 媒体代理（图/视频上传下载）：`await proxy_manager.get_media_proxy_url()`；未配置则回退到请求代理
- 浏览器打码还有**独立的** `browser_proxy_enabled / browser_proxy_url`（见验证码配置），仅作用于打码浏览器

---

## 4. 生成超时配置

**UI 字段**：图片生成超时（300s）、视频生成超时（1500s）。

**数据存放**：
- 表：`generation_config(image_timeout, video_timeout)`
- 模型：`GenerationConfig`（`src/core/models.py:135`）
- 默认：300 / 1500（`config/setting.toml:41-43`）
- 内存：`config.image_timeout / video_timeout`（`config.py:258-277`）

**接口**：
- `GET/POST /api/config/generation`（`admin.py:1127-1151`）
- 别名 `GET/POST /api/generation/timeout`（`admin.py:1381-1398`）

**范围校验**：
- `image_timeout`：60–3600 秒
- `video_timeout`：60–7200 秒

**业务使用**：
- `/api/images`、`/api/videos` 生成路由及 `generation_handler` 轮询任务状态时使用；超时返回 504，并释放 Token 锁

---

## 5. Token 轮询配置

**UI 字段**：轮询模式（随机轮询 / 顺序轮询）。

**数据存放**：
- 表：`call_logic_config(call_mode, polling_mode_enabled)`
- 模型：`CallLogicConfig`（`src/core/models.py:143`）
- 默认：`default`（随机轮询）（`config/setting.toml:46`）

**接口**：
- `GET/POST /api/call-logic/config`（`admin.py:1154-1190`）

**模式**：
- `default` — 随机（按负载优先策略挑选）
- `polling` — 顺序（稳定顺序循环）

**业务使用**：`TokenManager.select_token()` 根据 `config.polling_mode_enabled` 决定挑选策略（`src/services/token_manager.py`）。

---

## 6. 错误处理配置

**UI 字段**：拉黑阈值（默认 3）。

**数据存放**：`admin_config.error_ban_threshold`，默认 3（`config/setting.toml:49`）。

**接口**：
- `GET /api/admin/config` / `POST /api/admin/config` → `UpdateAdminConfigRequest(error_ban_threshold)`（`admin.py:1315-1337`）

**业务使用**：
- `TokenManager` 在每次请求后累加 `consecutive_error_count`，成功则清零
- 达到阈值时 `is_active = False`，该 Token 自动停用、不再参与轮询

---

## 7. 缓存配置

**UI 字段**：启用缓存（开关）。

> 后端还额外支持 `cache_timeout`、`cache_base_url`，但截图面板当前仅显示开关。

**数据存放**：
- 表：`cache_config(cache_enabled, cache_timeout, cache_base_url)`
- 默认：关闭，`cache_timeout=7200`（0 表示永不过期）（`config/setting.toml:51-54`）

**接口**：
- `GET /api/cache/config`（`admin.py:1434-1450`）
- `POST /api/cache/enabled` / `POST /api/cache/config` / `POST /api/cache/base-url`（`admin.py:1453-1509`）

**行为**：
- 启用后，生成的图片/视频本地落盘，返回给客户端的 URL 基于 `cache_base_url`（为空则用 `http://127.0.0.1:8000`）
- `cache_timeout>0`：到期由 `FileCache` 清理任务删除
- 保存后调用 `_sync_runtime_cache_config()`（`admin.py:1425-1430`）重置清理任务

---

## 8. 插件连接配置

**UI 字段**：
- 「连接地址」：形如 `http://localhost:8000/api/plugin/update_token`
- 「连接 Token」：可「随机」生成 / 「复制」
- 「通用 Token 同步」复选框：对应 `auto_enable_on_update`

**数据存放**：
- 表：`plugin_config(connection_token, auto_enable_on_update)`
- 模型：`PluginConfig`（`src/core/models.py:203`）
- 默认：`connection_token=""`（首次保存由 `secrets.token_urlsafe(32)` 生成，`admin.py:1689`），`auto_enable_on_update=True`

**接口**：
- `GET/POST /api/plugin/config`（`admin.py:1644-1701`）
- **Chrome 插件回传**：`POST /api/plugin/update-token`（`admin.py:1704-1797`）
  - 连接地址由请求 `Host` 动态生成（`admin.py:1652-1666`）
  - 请求头：`Authorization: Bearer <connection_token>`
  - Body：`{ "session_token": "<Google Labs Cookie 中的 ST>" }`

**端到端流程**：

1. 浏览器插件在 `labs.google` 域抓取 ST 并 POST 到 `/api/plugin/update-token`
2. 后端校验 Bearer 与 `plugin_config.connection_token` 一致（否则 401）
3. `token_manager.flow_client.st_to_at(session_token)` 将 ST 交换为 AT，解析邮箱与 AT 过期时间
4. 按邮箱 upsert：
   - 已存在 → 更新 ST / AT / AT_expires；若 `auto_enable_on_update=True` 且原先被禁用，则重新启用
   - 不存在 → 新增，备注 `Added by Chrome Extension`
5. 响应 `{success, action: "updated"|"added", auto_enabled?}`

---

## 9. 验证码配置

**UI 字段**（截图当前为 `内置浏览器自动打码` 模式）：
- 打码方式（下拉）
- 「带 Token 最大数量」= 4
- 「最大标签数量」= 5
- 「标签空闲超时」= 600 秒
- 「启用调度」复选框

**数据存放**：
- 表：`captcha_config`（字段较多，包括各三方服务的 `*_api_key` / `*_base_url`，浏览器代理，及 personal 模式专属字段）
- 模型：`CaptchaConfig`（`src/core/models.py:175-200`）
- 默认：`captcha_method="personal"`（`config/setting.toml:57`）

**接口**：
- `GET /api/captcha/config`（`admin.py:1607-1630`）
- `POST /api/captcha/config`（`admin.py:1512-1604`）
- `POST /api/captcha/score-test`（`admin.py:1633-1639`）

**打码方式一览**：

| method | 需要字段 | 说明 |
|---|---|---|
| `yescaptcha` | `yescaptcha_api_key`, `yescaptcha_base_url` | 三方 reCAPTCHA 服务 |
| `capmonster` | `capmonster_api_key`, `capmonster_base_url` | 三方 |
| `ezcaptcha` | `ezcaptcha_api_key`, `ezcaptcha_base_url` | 三方 |
| `capsolver` | `capsolver_api_key`, `capsolver_base_url` | 三方 |
| `browser` | — | 本机 Playwright 有头浏览器 |
| `personal` | — | 本机 Nodriver 浏览器（默认，截图所示） |
| `remote_browser` | `remote_browser_base_url`, `remote_browser_api_key`, `remote_browser_timeout(5–300)` | 远程有头打码服务，`POST {base}/api/v1/custom-score` |

**personal 模式三个参数**（`config.py:392-417`，`database.py:1725-1728`）：

| UI 名 | 字段 | 范围 | 默认 | 含义 |
|---|---|---|---|---|
| 带 Token 最大数量 | `personal_project_pool_size` | 1–50 | 4 | 单 Token 的 project_id 轮换池大小，影响同一账号并发可用的 project 数 |
| 最大标签数量 | `personal_max_resident_tabs` | 1–50 | 5 | **全局共享**的浏览器标签上限；并发打码最多占用这么多标签，超出需排队 |
| 标签空闲超时 | `personal_idle_tab_ttl_seconds` | ≥60 | 600 | 标签空闲超过该秒数将被回收，下次再用时重新打开 |

> 「标签」= 浏览器 Tab，不隶属某个 Token，而是被全局 personal 打码池复用。`BrowserCaptchaService`（`src/services/browser_captcha_personal.py`）负责池子的创建、复用、回收。

**浏览器代理（独立于全局代理）**：
- `browser_proxy_enabled` + `browser_proxy_url` 仅作用于 `browser` / `personal` 打码时启动的浏览器实例
- 分数测试 `_score_test_with_*`（`admin.py:310+`、`admin.py:410-446`）优先使用浏览器代理

**保存副作用**：personal 模式保存后，额外调用 `BrowserCaptchaService.reload_config()`（`admin.py:1596-1602`）立刻应用新参数。

---

## 10. 调试配置

**UI 字段**：启用调试模式（会生成 `logs.txt`）。

**数据存放**：
- 表：`debug_config(enabled, log_requests, log_responses, mask_token)`
- 默认：关闭（`config/setting.toml:32`）
- 内存：`config.debug_enabled`（`config.py:211-212`）

> **重启后自动关闭**：Debug 目前在保存后主要作用于内存，服务重启不保留启用状态（以代码实际行为为准，若你需要持久化，见「待核对」）。

**接口**：`POST /api/admin/debug` → `UpdateDebugConfigRequest(enabled)`（`admin.py:1364-1378`）

**业务使用**：日志层在记录请求/响应前检查 `config.debug_enabled`，开启后 `flow_client.py` 会把所有上游调用、响应（按需脱敏 Token）写入 `logs.txt`。

---

## 热加载与持久化统一约定

保存流程模板：

```
POST /api/xxx/config
  └─ admin.py 路由处理（参数校验 + 归一化）
     └─ db.update_xxx_config(...)           ← 写 SQLite
     └─ db.reload_config_to_memory()        ← 回填内存 Config
     └─ [可选] service.reload_config()      ← 如 BrowserCaptchaService / FileCache
     └─ 返回 {success: True, ...}
```

`reload_config_to_memory()`（`database.py:1502-1562`）会依次调用 `config.set_*()` 方法，把数据库中全部 11 张配置/状态表的最新值回写入内存单例。

**不写 TOML**：`config/setting.toml` 仅在数据库为空时作为初始值来源，运行期所有修改只进 SQLite。

---

## 校验/约束速查

| 项 | 约束 | 来源 |
|---|---|---|
| 图片超时 | 60–3600s | `config.py:72-82` |
| 视频超时 | 60–7200s | `config.py:269-277` |
| 缓存超时 | 0–86400s（0 永不过期） | `admin.py:1484-1485` |
| 拉黑阈值 | ≥1 | `models.py:122` |
| personal 最大标签 | 1–50 | `config.py:397`, `database.py:1727` |
| personal project 池 | 1–50 | `config.py:406`, `database.py:1726` |
| personal 空闲 TTL | ≥60s | `config.py:415`, `database.py:1728` |
| 远程打码超时 | 5–300s | `admin.py:1552` |
| 代理 URL | http(s) / socks5(h) / host:port / host:port:user:pass / `st5 ...` | `proxy_manager.py:13-109` |
| 密码 | 明文入参，bcrypt 存储 | `auth.py :: AuthManager` |

---

## 待你审核/核对的不确定项

1. **API Key 是否双 Key**：UI 有「当前 API Key」「Bi API Key」两个输入框，但当前代码 `admin_config` 仅单一 `api_key` 字段。若已扩展为双 Key，请核对最新 `models.py / database.py / admin.py`。
2. **验证码配置「启用调度」复选框** 对应哪个字段：未在调研中明确落到字段名，疑似控制 personal 模式的标签回收/调度开关，需对照 `captcha_config` 表结构确认。
3. **Debug 持久化**：目前按「写库 + 热加载」理解，但若实际为「仅内存、重启失效」，请按实际表现修正本文档对应段落。
4. **插件连接配置「通用 Token 同步」** 文案对应 `auto_enable_on_update`，是否同时还控制其他同步行为（如覆盖备注等）需再确认。

---

## 附录：前端定位

- 入口页面：`static/manage.html`
- 关键函数：`loadAdminConfig / saveProxyConfig / testProxyConfig / saveCaptchaConfig / savePluginConfig / ...`（均为 vanilla JS，调用上文列出的 REST 路由）
- 会话 Token 存储：浏览器 `localStorage`，每次请求带入 `Authorization: Bearer ...`
