# `personal_project_pool_size` 字段详解

> 位置：`captcha_config.personal_project_pool_size`
> 默认值：`4`，硬约束范围 `1–50`
> 相关代码：`src/services/token_manager.py`、`src/services/generation_handler.py`、`src/core/config.py`

这个字段的名字容易引起误解，实际作用比字面意思宽。下面把常见疑问集中回答。

---

## Q1：这个字段到底是什么意思？

**含义**：**单个 Token 预先维护的 VideoFX project 轮换池的大小**——即一个账号底下同时持有几个可用的 `project_id`，供生成请求轮流使用。

需要特别注意，它**不是**：

- ❌ Token 数量
- ❌ 浏览器标签数量
- ❌ 浏览器实例数量

`config/setting_example.toml` 里的注释也专门强调过：

> 仅影响项目轮换，不决定打码标签页数量。

---

## Q2：为什么要建这个池？单个 Token 一个 project 不够用吗？

VideoFX 里每个 `project_id` 是生成任务的**独立容器**，有自己的 scene、配额、状态。单 project 运行有三类问题：

1. **并发瓶颈**：高并发请求串在同一个 project 上，容易被 Google 侧做单 project 限频。
2. **状态耦合**：同一 project 并行发起多个生成，scene 状态可能互相干扰。
3. **故障放大**：一个 project 一旦被标记/限频，整个 Token 就废了；有池可以切换到兄弟 project 继续用。

更关键的一个背景：**personal 打码模式下，reCAPTCHA token 和 `project_id` 强绑定**。想让本机浏览器多 Tab 并发过码，就必须让单个 Token 有多个 project 可以轮换——这才是字段名带 `personal_` 前缀的历史原因。

---

## Q3：它在什么时候被使用？

共有 4 条触发路径：

### 1. 添加 Token 时批量建池 — `token_manager.py:244-297`

```
add_token()
  → _get_project_pool_size() 读配置（当前为 N）
  → 复用/创建 project P1
  → 循环创建 P2, P3, ..., PN（调 Flow API create_project）
  → 全部 INSERT 到 projects 表
```

命名约定：`<base_name> P1`、`<base_name> P2`……

### 2. 每次生成请求时按需补齐 + 轮询选一个 — `token_manager.py:594-625` (`ensure_project_exists`)

这是**运行时的主要作用点**：

```
  读 projects 表里该 Token 的激活 project
  若数量 < pool_size → 现场补齐（再调 Flow API 建 project）
  取前 pool_size 个作为可选集
  按 tokens.current_project_id 的下一个做 round-robin
  写回 tokens.current_project_id / current_project_name
```

既保证池规模，又做 project 级负载均衡。

### 3. personal 模式启动预热 — `token_manager.py:65-104` (`get_personal_warmup_project_ids`)

personal 模式的浏览器池启动时，**遍历所有激活 Token，每个 Token 最多吐出 `pool_size` 个 project_id**，交给浏览器服务预打开 Tab，避免第一次请求的冷启动延迟。**这条路径仅在 personal 模式下生效**。

### 4. 运行时热更新 — `config.py:412-439`

管理员后台改值后，`set_personal_project_pool_size` 立刻钳制到 1–50 写入内存 config，下一次读取即生效。不需要重启。

---

## Q4：如果我不用 personal 模式，这个字段是不是就没用了？

**不是，但作用确实会大幅缩水。**

关键事实：**`ensure_project_exists` 在每次生成请求里都被调用（`generation_handler.py:1010`），和打码模式无关**。也就是说不管你用的是 yescaptcha / capmonster / capsolver / browser / remote_browser 还是 personal，**每次生成都会在 `pool_size` 个 project 之间轮换**。

字段名带 `personal_` 前缀有点误导——是历史命名问题，它的**实际作用域比名字暗示的要宽**。

### 各模式下的实际影响对比

| 模式 | `pool_size` 的作用 | 调到 1 会怎样 |
|------|---------------------|---------------|
| **personal** | **核心作用**：多 Tab 并发打码的基础；启动预热也用它 | 并发打码能力退化，Tab 排队串行 |
| `browser` | 中等：project 级负载分散 | 并发无明显变化，但同 project 被限频的风险增加 |
| `yescaptcha` / `capmonster` / `ezcaptcha` / `capsolver` | **弱**：第三方打码与 project 无关 | 几乎无感知，偶尔遇到 project 级限频会更脆弱 |
| `remote_browser` | 中等：远程浏览器池按 project 缓存 token | 同 browser |

### 为什么 personal 模式下它不可替代

在 personal 模式下，本机 Chromium 打开的每个 Tab 过一次 reCAPTCHA 拿到一个 token，**这个 token 只对特定 project 有效**。如果只有一个 project：

- 多 Tab 也只能拿同一 project 的 token
- 并发打码退化为串行（因为同一 project 的 scene 状态冲突）
- 浏览器池子的"多 Tab"设计失去意义

所以 personal 模式下，`personal_project_pool_size` 和 `personal_max_resident_tabs` 是要**配套考虑**的——池子太小，Tab 开再多也跑不满。

### 为什么其他模式下它作用变弱

其他模式的打码 token 由第三方服务/远程浏览器签发，**不跟具体 project 绑定**。所以池的"打码并发"价值消失，只剩下通用的 project 级反限频、反状态冲突——还在，但没那么不可或缺。

**结论**：如果你的部署只用三方打码，把值留在默认 4 其实也无伤大雅，代价只是每个 Token 添加时会在 Flow API 真的建 4 个 project，占用 Google 侧账号资源。想省资源的话调到 1 也能跑。

---

## Q5：把值调大或调小会发生什么？

| 操作 | 效果 | 副作用 |
|------|------|--------|
| 调大（如 4 → 10） | 下次 `add_token` / `ensure_project_exists` 自动补建到 10 个 | **会真实调用 Flow API 建 project**，消耗账号侧的资源配额 |
| 调小（如 10 → 4） | 取用时只从前 4 个 project 里选 | **不会自动删除**已有的 P5–P10，它们留在 `projects` 表里变成"孤儿"，占位不被选 |

> 想彻底清理多余 project 得手动操作（admin API 或直接 DB），字段本身没做收缩回收逻辑。

---

## Q6：它和相邻的 personal 配置字段怎么区分？

这是 personal 模式配置里最容易搞错的一组，**三者维度完全不同**：

| 字段 | 维度 | 作用层 |
|------|------|--------|
| `personal_project_pool_size` | **每个 Token** 的 **project 数** | 业务资源（Flow API 侧的 `project_id`） |
| `personal_max_resident_tabs` | **全局共享** 的浏览器 **Tab 数** | 运行时资源（本机 Chromium tab） |
| `browser_count` | **浏览器实例数** | 进程级资源（Chromium 进程） |

三者没有等值关系。举例："10 Token × pool=4 × tabs=5 × browser=1" = **40 个 project、共享 5 个 Tab、跑在 1 个浏览器里**。

记忆诀窍：
- **project 是数据侧的资源**（Google 账号里的虚拟资源）
- **tab / browser 是计算侧的资源**（本机进程/标签）

两边是**多对多复用关系**，不是一一对应。

---

## 一句话总结

> `personal_project_pool_size` 控制**单个账号持有多少个可轮换的 VideoFX project**，目的是在高并发下分散 project 级限频风险、避免 scene 状态冲突。在 personal 打码模式下它是"必调参数"（决定多 Tab 并发能力上限），在其他模式下它是"可选参数"（设为 1 也能跑，只损失一些 project 轮换的健壮性）。

**字段名带 `personal_` 前缀只是历史命名，实际作用域覆盖所有打码模式。**
