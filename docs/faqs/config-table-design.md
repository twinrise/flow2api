# 为什么用多张分组配置表，而不是一张 KV 配置表？

## 问题背景

翻一下 `data/flow.db` 的表结构会发现，系统里有 **8 张"单行配置表"**：

```
admin_config / proxy_config / generation_config / call_logic_config /
cache_config / debug_config / captcha_config / plugin_config
```

每张都是 `id INTEGER PRIMARY KEY DEFAULT 1`，永远只有一行。很自然会冒出一个疑问：

> 为什么不直接建一张 `config(key TEXT PRIMARY KEY, value TEXT)` 的 KV 表就完事了？

这里系统性地解释一下背后的设计取舍。

---

## 一句话结论

作者选了 **"配置即 schema"** 路线——把配置当成**强类型的单行记录**来建模，放弃 KV 的通用灵活性，换来类型安全、读写原子性、与前端模块的天然对齐，以及**几乎零成本的字段级迁移**。

在这个项目的体量和场景下（单机 SQLite、配置项由代码定义、后台按模块分页），这是务实选择。

---

## 多配置表的具体收益

### 1. 原生类型约束 vs 全部字符串化

KV 表的值列只能是 `TEXT` 或 `BLOB`：

```sql
-- KV 方案下的打码超时
INSERT INTO config(key, value) VALUES ('captcha.remote_browser_timeout', '60');
-- 读出来是字符串 "60"，需要 int() 转换；布尔要存 "0"/"1" 再转
```

而当前设计直接用数据库一级类型：

```sql
cache_timeout INTEGER DEFAULT 7200
cache_enabled BOOLEAN DEFAULT 0
image_timeout INTEGER DEFAULT 300
```

读出来就是正确类型，Pydantic 反序列化几乎零成本。布尔判断不会因为 `"false"` 字符串被当真值坑到。

---

### 2. `DEFAULT` + `ALTER TABLE ADD COLUMN` = 免费迁移

这是这套设计**最务实的一点**。

看 `database.py` 里的迁移逻辑：新加一个配置项时，只需要：

```sql
ALTER TABLE captcha_config
  ADD COLUMN personal_idle_tab_ttl_seconds INTEGER DEFAULT 600;
```

老库里**每一行自动带上默认值**，应用代码零 fallback。

换成 KV 表呢？老库里根本没这个 key 的记录，代码里要写：

```python
get_config("captcha.personal_idle_tab_ttl_seconds", default=600)
```

**默认值会散落到调用点**，每个读取位置都要维护自己那份默认值，非常容易漂移。要避免漂移就得再做一层"中央默认值表 + 启动时补齐"，等于自己重新实现了 `DEFAULT` 语义。

---

### 3. 一次读 = 一整组配置

业务路径上读打码配置：

```sql
-- 当前设计：一行拿齐 20+ 字段
SELECT * FROM captcha_config WHERE id = 1;
```

直接喂给 Pydantic 反序列化成 `CaptchaConfig` 对象，结束。

KV 方案：

```sql
SELECT key, value FROM config WHERE key LIKE 'captcha.%';
-- 或
SELECT key, value FROM config WHERE key IN ('captcha.method', 'captcha.api_key', ...);
```

还要在应用层手动拼成对象——**每次调用都干一次**。热路径上这是实打实的开销。

---

### 4. 原子更新一组相关配置

管理员后台保存「打码设置」页面时：

```sql
-- 当前设计：一条语句搞定
UPDATE captcha_config
   SET captcha_method = ?, yescaptcha_api_key = ?, browser_count = ?, ...
 WHERE id = 1;
```

KV 方案要么多条 `UPDATE` 包事务，要么 `INSERT OR REPLACE` 多行，外加"哪几个 key 属于这组"的隐式约定——而这个约定只存在于代码里，数据库无法校验。

---

### 5. 表结构即文档

8 张配置表正好对应 8 个功能模块：

| 表 | 模块 |
|----|------|
| `admin_config` | 管理员与全局阈值 |
| `proxy_config` | 代理 |
| `generation_config` | 生成超时与重试 |
| `call_logic_config` | 调用策略 |
| `cache_config` | 结果缓存 |
| `debug_config` | 调试日志 |
| `captcha_config` | 打码服务 |
| `plugin_config` | 浏览器插件 |

**Schema 本身就是文档**：看一眼表名和列名就知道有哪些可配项，IDE 能做 schema-aware 补全，DB 工具（DBeaver / sqlite-browser）里一目了然。

KV 表是个黑盒：不读代码你根本不知道有哪些合法 key、合法值域、有没有拼错。

---

### 6. 前端/API 天然对齐

看后台 UI 的常见 pattern——每个设置 tab 对应一个 endpoint：

```
GET/POST /api/config/captcha    -> captcha_config 整行
GET/POST /api/config/proxy      -> proxy_config  整行
GET/POST /api/config/generation -> generation_config 整行
```

请求体就是整张表的一行，后端几乎不用写 mapping，`Pydantic model ↔ 行记录` **1:1 对应**。

KV 方案要额外做"把一堆 key 归拢成一个响应结构"的序列化层——多一层胶水意味着多一处潜在 bug。

---

### 7. `updated_at` 精确到模块

每张配置表有自己的 `updated_at`，可以单独看「代理最近改过没」「打码最近改过没」，便于排查"到底是哪个配置触发了行为变化"。

KV 表要么给每个 key 带 `updated_at`（存储成本翻倍），要么只能知道「总配置最后改时间」，粒度过粗。

---

## KV 表更合适的场景（这里为什么不适用）

KV 配置表的**甜蜜点**是：

- **配置项集合是动态的**：用户自定义 feature flag、多租户个性化设置、运营后台下发开关
- **不需要强类型**：大量同构的布尔/字符串开关
- **跨进程/跨服务共享**一份中心化配置（如 etcd、Consul、Apollo）

而 Flow2API 的情况恰好相反：

| 维度 | Flow2API 的实际情况 | KV 适用场景 |
|------|---------------------|-------------|
| 配置项来源 | **代码定义，有限且稳定** | 用户定义，动态增减 |
| 值类型 | 大量带语义的数值（超时、阈值、URL、枚举） | 同构开关 |
| 部署形态 | 单机 SQLite | 多服务共享 |
| 前端形态 | 按模块分页的后台 | 无前端或搜索式 |
| 修改频率 | 低频，运维/管理员操作 | 中高频，运营下发 |

**在这种场景硬上 KV，等于白白多出一层"序列化 / 反序列化 / 默认值 fallback / 分组约定"的胶水代码，拿通用性换来的灵活性根本用不上。**

---

## 代价（作者承担但可接受）

当前设计也不是没有成本，只是这些成本在项目体量下不痛：

| 代价 | 为什么可以接受 |
|------|----------------|
| 加一组配置要建新表 + 写迁移 | 13 张表已覆盖所有模块，频率极低 |
| 没法一条 SQL 导出全部配置 | 后台按模块分页，不需要这个能力 |
| DAO 层要为每张表写 get/update 方法 | Pydantic 让这件事非常薄，模板化 |
| 表多了看起来"乱" | 表名语义化，反而比 KV 的上百个 key 更好理解 |

---

## 总结

这是一个**有意识放弃通用性、押注于"配置项稳定 + 类型安全 + 模块对齐"**的设计。决策链条是：

```
配置项由代码定义、有限且稳定
        ↓
可以为每组配置写一个明确的 schema
        ↓
选择表级建模（强类型、DEFAULT、原子更新）
        ↓
付出"加组要建表"的成本
        ↓
换得"零 fallback 代码、零序列化胶水、schema 即文档"
```

放在这个体量（单机、单产品、配置项 < 100）的项目里是合适的；如果是多租户 SaaS 或者 feature flag 平台，结论就会反过来。

**判断一个项目应该选哪种方案的核心提问**：

> 你的配置项**集合**是代码已知的，还是运行时动态生长的？

- **集合固定** → 选多张分组表（本项目）
- **集合动态** → 选 KV 表（或专门的配置中心）
