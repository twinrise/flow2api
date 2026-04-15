# TOML 是什么格式？为什么本项目用它？

## 一句话

**TOML** = *Tom's Obvious, Minimal Language*，由 GitHub 联合创始人 Tom Preston-Werner 在 2013 年设计，目标是"给人读的配置文件"。可以理解为 **INI 文件的升级版**：支持类型、嵌套、数组、注释，同时保持对人眼友好。

---

## 长什么样

```toml
# 这是注释
api_key = "han1234"
admin_username = "admin"
admin_password = "admin"

[generation]          # 段（table）
image_timeout = 300
video_timeout = 1500

[proxy]
enabled = false
url = "http://127.0.0.1:7890"

[[tokens]]            # 数组段（array of tables）
email = "a@x.com"
active = true

[[tokens]]
email = "b@x.com"
active = false
```

基本类型：字符串、整数、浮点、布尔、日期时间、数组、内联表。

---

## 和常见配置格式对比

| 格式 | 典型用途 | 优点 | 缺点 |
|---|---|---|---|
| **JSON** | API 数据交换 | 机器友好，解析器无处不在 | 不能写注释；引号/逗号繁琐；不适合人手改 |
| **YAML** | K8s / Docker Compose / Ansible | 最紧凑 | 依赖缩进，错一个空格就出错；`yes/no/on/off` 等隐式类型坑多 |
| **INI** | Windows 老配置 | 简单直观 | 没有数组、嵌套、类型；方言多 |
| **TOML** | 应用/工具配置 | 有注释、有类型、段落清晰、缩进无所谓 | 不适合深层嵌套的大型数据结构 |

经验法则：**配置文件选 TOML，数据交换选 JSON，大规模编排选 YAML**。

---

## 为什么本项目用 TOML

1. **给人改的** — `config/setting.toml` 是首次启动的默认值来源，可能被手动编辑，要求"可读、能写注释"。
2. **Python 生态原生支持** — Python 3.11+ 标准库自带 `tomllib`（只读），写入用 `tomli-w` / `tomlkit`，无需重量级依赖。
3. **社区标配** — `pyproject.toml`（Python 打包）、`Cargo.toml`（Rust）、`netlify.toml`、`fly.toml` 都用它，已是现代工具链的主流选择。
4. **类型安全** — `300` 是整数、`"300"` 是字符串、`true` 是布尔。没有 YAML 里的类型陷阱。

---

## 在本项目中的位置

- 路径：`config/setting.toml`
- 作用：**仅作为首次启动的默认值来源**。服务启动时若数据库为空，会把 TOML 里的默认值写入 SQLite。
- 运行期修改：**不会回写到 TOML**，全部通过后台接口写入 SQLite，并热加载到内存。
- 详情参见 [`docs/admin/system-config.md`](../admin/system-config.md) 的「热加载与持久化统一约定」段。

因此日常**不需要手动修改 `setting.toml`**，除非你想调整出厂默认值或在全新环境下初始化。

---

## 其他语言对 TOML 的支持

| 语言 | 支持方式 |
|---|---|
| Python | 3.11+ 标准库 `tomllib`（只读），写入用 `tomli-w` / `tomlkit` |
| Go | 第三方库：`github.com/BurntSushi/toml` 或 `github.com/pelletier/go-toml/v2` |
| Rust | `toml` crate（官方生态标配） |
| Node.js | `@iarna/toml`、`smol-toml` 等 |
| Java | `tomlj`、`night-config` |

---

## 延伸阅读

- 官方规范（含中文版）：<https://toml.io/cn/>
- 与 YAML/JSON 的对比讨论：<https://github.com/toml-lang/toml#comparison-with-other-formats>
