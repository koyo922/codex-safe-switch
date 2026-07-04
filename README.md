# codex-safe-switch

[![PyPI](https://img.shields.io/pypi/v/codex-safe-switch.svg)](https://pypi.org/project/codex-safe-switch/)
[![CI](https://github.com/kadaliao/codex-safe-switch/actions/workflows/ci.yml/badge.svg)](https://github.com/kadaliao/codex-safe-switch/actions/workflows/ci.yml)

中文 | [English](README.en.md)

在官方 OpenAI provider、第三方 relay、多组 API key 之间一键切换 [Codex CLI](https://github.com/openai/codex) 的 provider 配置。CLI + 可选 Alfred workflow。

## 第一次用？先看这里

**这个工具解决什么问题:** Codex CLI 同一时间只能连一个 provider。手动改 `~/.codex/config.toml` 切换 relay / 官方账号时,很容易把本地状态(尤其是**历史会话 metadata**)弄乱,导致切回去后**历史会话列表消失**。这个工具把每个 provider 存成一个 profile,切换时只动 provider 字段、并自动对齐历史，所以切来切去历史都还在。

**30 秒上手:**

```bash
uv tool install codex-safe-switch   # 1. 安装（需要先有 uv）
codex-safe-switch                   # 2. 直接运行 = 交互选择器，首次会自动导入你现在的配置
codex-safe-switch save myrelay      # 3. 把当前 provider 存成名为 myrelay 的 profile
codex-safe-switch official          # 4. 一键切回官方 OpenAI
```

如果你第一次运行时当前就是官方 OpenAI 登录，工具会自动把 provider 片段保存到隐藏 profile `~/.codex/profiles/.official/`，active 名称显示为 `official`。也可以在官方配置生效时手动刷新一次：

```bash
codex-safe-switch save official     # 只保存官方 provider 配置，不保存 auth.json
```

**如果你是因为「切换后历史会话消失」才找到这里：** 别慌，历史文件通常没丢，只是 metadata 和当前 provider 对不上。

```bash
uv tool install codex-safe-switch
codex-safe-switch doctor-history    # 只读检查，看看现在历史指向哪个 provider/model
codex-safe-switch use <profile>     # 切到历史对应的那个 provider，会自动对齐历史（use/official 都会）
# 不想切换、只想原地修复 metadata：
codex-safe-switch merge-history --dry-run   # 先预览改动
codex-safe-switch merge-history             # 确认无误后写入
```

> 全程不会动 `~/.codex/auth.json`，所以不会覆盖你的官方 ChatGPT 登录。原理见下面「它怎么保证 safe 切换」一节。

## 安装

```bash
uv tool install codex-safe-switch
```

需要先有 [`uv`](https://github.com/astral-sh/uv)。安装后 `codex-safe-switch` 会被放到 `$PATH`（默认 `~/.local/bin/`）。

## 快速开始

```bash
codex-safe-switch           # 交互选择（↑/↓，回车切换）
codex-safe-switch ls        # 列出所有 profile，★ 表示当前
codex-safe-switch save dev  # 把当前 ~/.codex 状态存成 dev
codex-safe-switch official  # 切回官方 OpenAI provider
```

首次运行会自动把当前 `~/.codex/config.toml` 导入成 profile，不会丢配置。

<details>
<summary><strong>完整命令</strong></summary>

```text
codex-safe-switch              # 交互式选择器
codex-safe-switch ls           # 列出 profiles，★ 表示当前 active
codex-safe-switch current      # 打印当前 active profile
codex-safe-switch official     # 切回官方 OpenAI provider（别名：openai）
codex-safe-switch use [name]   # 加载 <name>；不传 name 时进入选择器
codex-safe-switch save <name>  # 把当前 provider 配置保存成 <name>
codex-safe-switch save <name> --openai-auth-bearer-env RELAY_TOKEN
                               # 保存为 ChatGPT 登录态 + relay bearer token 的 profile
codex-safe-switch save official
                               # 当前配置是官方 OpenAI 时，刷新隐藏官方快照
codex-safe-switch show <name>  # 打印 <name> 的 provider.toml 和 session-state
codex-safe-switch state <name> # 查看/设置 profile 的 session-state 作用域
codex-safe-switch rm <name>    # 删除 profile（不允许删除 active）
codex-safe-switch restart-codex
                               # 终止 Codex app/server 进程，让配置立即生效
codex-safe-switch merge-history --dry-run
                               # 预览历史 metadata 修复，不写入文件
codex-safe-switch doctor-history
                               # 只读检查历史 provider/model 状态
codex-safe-switch alfred-list  # Alfred Script Filter JSON
```

`use` / `official` 都可以加 `--restart-codex`，顺手重启 Codex app/server。

当 stdin/stdout 不是 TTY（管道、脚本）时，选择器会自动降级成数字菜单。

</details>

<details>
<summary><strong>Alfred 工作流</strong></summary>

执行 `uv tool install` 后，双击 `alfred/codex-safe-switch.alfredworkflow` 导入 Alfred。触发关键词是 `cx`。

workflow 默认调用 `$HOME/.local/bin/codex-safe-switch`。如果你的 `uv tool install` 把命令装到了别处，可以用 `uv tool dir --bin` 查看路径，然后修改 workflow plist 里的两个 script block。

</details>

<details>
<summary><strong>它怎么保证"safe"切换</strong></summary>

**只接管 provider，不动本地状态。** 每个 profile 接管 `~/.codex/config.toml` 里的下面这些字段；其他内容（trusted projects、plugins、marketplaces、MCP servers、TUI 偏好等）切换时完整保留：

- `model`, `model_provider`, `model_reasoning_effort`, `model_reasoning_summary`, `model_verbosity`
- `wire_api`, `disable_response_storage`, `preferred_auth_method`
- `[model_providers.*]`

**不接管 `auth.json`。** Profile 只保存 provider 相关配置；`~/.codex/auth.json` 由 Codex 自己维护。`save` / `use` / `official` 都不会保存或写回 `auth.json`，所以切换 provider 不会覆盖你的官方 ChatGPT 登录缓存或本地认证状态。

**只保存当前启用的 provider 块。** 如果你把 `model_provider = "..."` 注释掉了，即使下面还留着 `[model_providers.<name>]` 块，`save` 也不会把那块当成当前 profile 保存；如果启用了 `model_provider = "relay"`，只保存 `[model_providers.relay]`，不会顺手带走其他 provider 块。

**历史会话默认对齐。** 每次 `use` / `official` 后自动把本地历史 metadata 对齐到当前 provider 和 model，所以 relay 和官方账号之间切换时历史不会消失：

- 自动修复 rollout 文件 + `state_5.sqlite` 里的 provider/model 列。
- 如果 `session_index.jsonl` 落后于 SQLite 最新 thread，会补追加索引，避免移动端历史停在旧时间点。
- 已用过 Codex remote-control 的机器会顺手检查 managed app-server 链路，处理旧的 unix socket / SSH proxy 残留。
- `merge-history --keep-models` 可以只修 provider 不改 model；`--dry-run` 预览；`doctor-history` 只读诊断。

**官方 OpenAI 一键回退。** `codex-safe-switch official` 切回官方 OpenAI provider，工具维护隐藏 provider 快照 `~/.codex/profiles/.official/`，第一次从官方切走时自动刷新。

你也可以在当前配置就是官方 OpenAI 时运行 `codex-safe-switch save official` 主动刷新这个隐藏快照；如果当前配置不是官方 OpenAI，会直接拒绝，避免把 relay 误存成 official。

**进程隔离。** `restart-codex`（以及 `--restart-codex`）精确跳过 `codex-safe-switch` 自身进程，不会自杀。

**Remote 登录态风险提示。** 如果当前 `auth.json` 是 ChatGPT 登录态，但 active provider 使用 `env_key` 或 `[model_providers.<name>.auth]` 这类 API-key / bearer-only 认证形态，`use` 和 `restart-codex` 会提示风险。普通 API-key relay 仍然可以正常切换；但如果你希望手机 Remote、插件和 Codex App 继续使用 ChatGPT 登录态，请使用 `--openai-auth-bearer-env` 保存 profile。

</details>

<details>
<summary><strong>Profile 格式 + 添加 relay</strong></summary>

```text
~/.codex/profiles/
├── .active                       # 明文：当前 active profile 名
├── .official/
│   └── provider.toml             # 官方 OpenAI provider 片段
└── myrelay/
    └── provider.toml             # 只包含 provider 字段（见 examples/）
```

**添加普通 API-key relay profile**

1. 在 `~/.codex/config.toml` 里配置 relay，确认 `codex` 能跑。
2. 如果 key 来自环境变量，在 provider 里配置 `env_key = "..."`；profile 不需要也不会保存 `auth.json`。
3. `codex-safe-switch save <name>` 把 provider 片段存成 profile。
4. 之后用 `cx`（Alfred）或 `codex-safe-switch use <name>` 随时切。

**添加需要保留 ChatGPT 登录态 / Codex Remote 的 relay profile**

有些 relay 仍希望 Codex 使用 ChatGPT/OpenAI 登录链路，这样 Codex App、插件、手机 Remote 等依赖 ChatGPT 登录态的能力不会掉线。先把 relay token 放进环境变量，然后用：

```bash
codex-safe-switch save myrelay --openai-auth-bearer-env MYRELAY_TOKEN
```

这个命令会把当前 active provider 保存成下面的形态：

- `requires_openai_auth = true`
- `experimental_bearer_token = "<MYRELAY_TOKEN 的当前值>"`
- `preferred_auth_method = "chatgpt"`

同时会移除该 provider 里的 `env_key` 和 `[model_providers.<name>.auth]`，避免 Codex 进入 API-key / bearer-only 认证链路。注意：这种模式会把 bearer token 写入 profile 文件和激活后的 `~/.codex/config.toml`，请像保护 `auth.json` 一样保护这些文件。

也可以手写 profile 文件，参考 `examples/relay-profile/`。

</details>

<details>
<summary><strong>环境变量 / 开发版 / 发版</strong></summary>

**环境变量**

| 变量                 | 默认值              | 用途                    |
| -------------------- | ------------------ | ----------------------- |
| `CODEX_PROFILE_ROOT` | `~/.codex/profiles` | profiles 存放位置        |
| `CODEX_HOME`         | `~/.codex`          | 要写入的 Codex 配置目录   |

**免安装试用**

```bash
uvx --from codex-safe-switch codex-safe-switch ls
```

**装开发版**

```bash
uv tool install git+https://github.com/kadaliao/codex-safe-switch.git
```

**发版**

push `v*` tag 触发 `Publish to PyPI` workflow,会校验 tag 版本和 `pyproject.toml` 一致,跑测试 + build + twine check 后发布。

```bash
git tag vX.Y.Z && git push origin vX.Y.Z
```

</details>

## License

MIT
