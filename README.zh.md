# zpdf

[English](README.md) | 中文

ZPDF Tool 插件，为 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 提供高保真 PDF→Markdown 解析、保留版式的 PDF 翻译、Markdown 文档转换、费用预估和余额查询。

仓库：<https://github.com/nguyenvanhoang1/dsh-zpdf>

## 功能

| Tool | 功能 |
| --- | --- |
| `zpdf_parse_pdf` | PDF → Markdown，可选同步翻译，支持公式、表格、图片与 enrichment sidecar |
| `zpdf_translate_pdf` | 保留原版式翻译 PDF，可输出纯译文或双栏对照 |
| `zpdf_convert_markdown` | Markdown/ZIP → DOCX、HTML、PDF、LaTeX |
| `zpdf_estimate_cost` | 本地读取页数并结合账户余额估算 credits（不扣费） |
| `zpdf_check_balance` | 查询当前 credits 余额 |
| `zpdf_get_task_status` | 按 task id 排查超时或卡住的任务 |

- GUI 通过 DSH 凭据库保存 API Key（`ZPDF_API_KEY`），不会出现在设置 describe 响应里。
- Web GUI 的 **Settings → ZPDF** 是单独设置页，显示“已配置/未配置”并提供保存和清除，附带**实时积分**卡片与**任务总览**列表——积分与任务状态自动刷新（30 秒 / 10 秒），支持手动刷新与清除记录。任务历史保存在 `$DSH_HOME/zpdf/tasks.json`（最多 200 条）。
- Tool 调用时才检查 Key；没有配置时会提示打开设置页或使用 CLI。
- 也支持进程环境变量 `ZPDF_API_KEY`。环境变量优先，存在时设置页为只读。
- 上传、轮询、下载与 ZIP 解压都会响应 Tool 取消信号。

## 要求

- Node.js >= 20
- DeepSeek Harness `0.1.1-rc.2` 兼容版本（0.1.1-rc.2 client-modules 格式：`dsh.client` 清单 + `exports["./client"]` 惰性 CJS bundle）
- ZPDF Plus 或 Pro 账户
- 在 <https://www.zhiyipdf.com/api-keys> 创建 API Key

## 安装

先把包安装到需要使用的 DSH profile（以 `web` 为例）。推荐直接从 GitHub 安装：

```bash
# 从 GitHub 仓库安装（推荐，无需等 npm 发布）
dsh plugin --profile web add github:nguyenvanhoang1/dsh-zpdf

# 等价写法
dsh plugin --profile web add https://github.com/nguyenvanhoang1/dsh-zpdf.git
```

这是普通的 Git 依赖：仓库里带编好的 `lib/`，没有 `prepare`，因此不会触发 pnpm `allowBuilds`。同时声明了 `dsh.bundle`，所以 `dsh plugin add` 会把 `@nguyenvanhoang1/dsh-zpdf` 写进该 profile 的 `dsh.profile.bundles`，下次启动就会挂载——设置页和 Tool 会自动出现，不必手写 composition 行。

`@deepseek-ai/dsh-tools` 等宿主包是 **可选 peer** 依赖（和 [dsh-ads](https://github.com/Nagi-ovo/dsh-ads) 同款模式）：所有宿主 peer 都在 `peerDependenciesMeta` 中标记为 optional，安装时 pnpm 不再报 "missing peer"；运行时契约不变——必须解析到当前 Harness 自带的那一份（模块回退栈）。如果插件再自带一份，任意 Tool 调用都会报 `Cannot read properties of undefined (reading 'prepare')`。

也可以从本地目录安装：

```bash
# 在本仓库直接进行本地测试
dsh plugin --profile web add D:/code/work-relate/dsh-zpdf
```

`dsh plugin add` 已经会激活 bundle 层。只有用别的方式装包时，才需要自己在 profile 的 `cordis.patch.yml` 里加一行：

```yaml
- insert:
    - id: zpdf
      name: '@nguyenvanhoang1/dsh-zpdf'
```

参见 [`examples/cordis.patch.yml`](examples/cordis.patch.yml) 和仓库根目录的 [`cordis.patch.yml`](cordis.patch.yml)。

## 发版

npm 发布暂未启用，请使用上面的 GitHub 安装命令。新包名为 `@nguyenvanhoang1/dsh-zpdf`，对应 npm scope 尚未配置发布授权。

启用 `Release & Publish`（`.github/workflows/release.yml`）前，需取得该 npm scope 的发布权限，并将 npm Trusted Publisher 配置为 owner `nguyenvanhoang1`、仓库 `dsh-zpdf`、workflow `release.yml`。工作流使用 OIDC，不使用 `NPM_TOKEN` secret。本次仓库迁移不修改此前已发布的 npm 包。

`CI` 执行类型检查、测试和构建；`Rebuild lib` 同步已提交的 `lib/`，确保不执行 prepare 脚本的 Git 安装可以直接使用成品。

重启 profile：

```bash
dsh web
```

> `dsh plugin ... add` 会安装依赖，并且因为本包声明了 `dsh.bundle`，会加入 profile 的 layer 栈。单独 `pnpm add`、不走 `dsh plugin` 则不会。

## 配置 API Key

### 方法一：Web 设置页（推荐）

打开 DeepSeek Harness Web GUI：

1. 点击左下角 **Settings / 设置**；
2. 进入独立的 **ZPDF** 设置项；
3. 输入 API Key 并保存。

设置页把 Key 写入 DSH 凭据库（`$DSH_HOME/.credentials.yaml` 的 `ZPDF_API_KEY` 引用），不经过设置文档 allowlist。若进程环境里已经有 `ZPDF_API_KEY`，设置页会显示为只读（环境变量优先，不能覆盖）。

如果尚未设置 Key，插件仍可启动；第一次需要鉴权的 Tool 调用会返回可操作提示。

### 方法二：CLI

安装包后，请通过 DeepSeek Harness 调用 CLI（直接打 `zpdf` 会提示找不到命令，因为它不在全局 PATH 上）：

```bash
dsh plugin --profile web exec zpdf -- config set-key
```

命令会用遮罩输入读取 Key，并写入 `$DSH_HOME/settings.yaml`（默认 `~/.dsh/settings.yaml`）中的 `zpdf.apiKey`。

```bash
# 非交互传参（会进入 shell history，不推荐）
dsh plugin --profile web exec zpdf -- config set-key sk-xxxxxxxxxxxxxxxx

# 脚本/CI：从 stdin 读取
printf '%s' "$ZPDF_API_KEY" | dsh plugin --profile web exec zpdf -- config set-key

# 查看状态（不会输出 Key）
dsh plugin --profile web exec zpdf -- config status

# 查看实际设置文件路径
dsh plugin --profile web exec zpdf -- config path

# 清除已保存 Key
dsh plugin --profile web exec zpdf -- config clear-key

# 使用自定义 settings 文件
dsh plugin --profile web exec zpdf -- config set-key --file D:/path/to/settings.yaml
```

CLI 会尽量保留 YAML 注释；写入使用与 DSH 自身 settings 提供者相同的 `<file>.lock` 写锁与原子替换（`@deepseek-ai/dsh-atomic-write`），并把设置文件权限设为 owner-only（`0600`；Windows 上仍受 ACL 控制）。正在运行且启用了文件监听的 DSH 会热加载该修改。

### 方法三：环境变量

```bash
export ZPDF_API_KEY=sk-xxxxxxxxxxxxxxxx
```

PowerShell：

```powershell
$env:ZPDF_API_KEY = 'sk-xxxxxxxxxxxxxxxx'
dsh web
```

环境变量需要在启动 DSH 前设置。可在 composition 中把 `apiKeyEnv` 改成其他变量名。

解析顺序：CLI 写入的 `settings.apiKey` → 凭据库 / 环境变量（`ZPDF_API_KEY`）。

## 可选配置

可直接在 composition 的插件行提供 base 配置；用户 settings 层仍会覆盖它：

```yaml
- insert:
    - id: zpdf
      name: '@nguyenvanhoang1/dsh-zpdf'
      config:
        outputDir: ./zpdf-output
        pollIntervalMs: 2000
        maxPollMinutes: 30
        httpTimeoutMs: 60000
        uploadTimeoutMs: 600000
```

| 字段 | 默认值 | 说明 |
| --- | --- | --- |
| `apiKey` | 未设置 | Secret；推荐由 GUI/CLI 写入，不要提交到 composition |
| `apiKeyEnv` | `ZPDF_API_KEY` | 环境变量 / 凭据引用名 |
| `baseUrl` | `https://www.zhiyipdf.com` | ZPDF 服务地址 |
| `outputDir` | `./zpdf-output` | 结果输出目录（相对 DSH 启动目录） |
| `pollIntervalMs` | `2000` | 轮询任务状态间隔 |
| `maxPollMinutes` | `30` | 最长轮询分钟数 |
| `httpTimeoutMs` | `60000` | 普通 HTTP 请求超时 |
| `uploadTimeoutMs` | `600000` | 上传/下载超时 |

## 开发

```bash
corepack enable pnpm
pnpm install
pnpm check
```

`pnpm check` 依次运行 TypeScript 类型检查、Vitest 测试和生产构建。

可选的宿主组合测试需要指定已安装的 Harness 包目录：

```bash
DSH_ROOT=/path/to/node_modules/@deepseek-ai/dsh pnpm test:host
```

## 安全说明

- 不要把真实 API Key 写入仓库、README、截图或 issue。
- Web 凭据 API 只返回是否已配置 / 是否可写，不返回 secret 内容。
- CLI 状态命令只显示 configured/not configured。
- 插件会读取 Tool 参数指定的本地 PDF/Markdown/ZIP，并把结果写入 `outputDir`；只在可信的 DSH composition 中启用它。
- 输出目录先按 `realpath` 校验：`output_subdir` 及其中的符号链接都不能逃出 `outputDir`。
- ZIP 下载使用路径穿越检查，拒绝绝对路径和 `../` 条目；解压上限为 10,000 个条目与 4 GiB 总解压大小，单次结果下载上限 2 GiB。
- 输入文件上限 300 MB、最多 800 页（与 ZPDF 服务端限制一致），并在本地先校验后再上传；上传、轮询、下载与解压都会响应取消信号。

## License

MIT

