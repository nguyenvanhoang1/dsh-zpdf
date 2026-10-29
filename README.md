# zpdf

English | [中文](README.zh.md)

ZPDF tools for [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness): high-fidelity PDF→Markdown parsing, layout-preserving PDF translation, Markdown conversion, credit estimates, and balance checks.

Repository: <https://github.com/nguyenvanhoang1/dsh-zpdf>

## Features

| Tool | Capability |
| --- | --- |
| `zpdf_parse_pdf` | PDF → Markdown, optional translation, formulas, tables, images, enrichment sidecars |
| `zpdf_translate_pdf` | Layout-preserving PDF translation (translated-only or side-by-side) |
| `zpdf_convert_markdown` | Markdown/ZIP → DOCX, HTML, PDF, LaTeX |
| `zpdf_estimate_cost` | Local page-count + balance estimate (does not spend credits) |
| `zpdf_check_balance` | Current credit balance |
| `zpdf_get_task_status` | Inspect a task by id |

- The GUI stores the API key through DSH credentials (`ZPDF_API_KEY`). The value never rides settings describe responses.
- **Settings → ZPDF** is a dedicated settings page (configured / missing, save, clear) with a **live credits** card and a **task overview** list — balance and task statuses refresh automatically (30 s / 10 s) and manually, with a clear-log action. Task history lives in `$DSH_HOME/zpdf/tasks.json` (newest 200).
- Tools check the key lazily. A missing key tells the user to open Settings or run the CLI.
- `ZPDF_API_KEY` in the process environment is also accepted. An env key shadows the GUI credential and makes the settings page read-only.
- HTTP upload, poll, download, and ZIP extract honor the tool abort signal.

## Requirements

- Node.js >= 20
- DeepSeek Harness `0.1.1-rc.2` compatible (0.1.1-rc.2 client-modules format: `dsh.client` manifest + `exports["./client"]` lazy-CJS bundle)
- ZPDF Plus or Pro account
- An API key from <https://www.zhiyipdf.com/api-keys>

## Install

Install the package into the DSH profile you use (example: `web`). GitHub is the recommended path:

```bash
# From GitHub (recommended; no npm publish required)
dsh plugin --profile web add github:nguyenvanhoang1/dsh-zpdf

# Equivalent
dsh plugin --profile web add https://github.com/nguyenvanhoang1/dsh-zpdf.git
```

This is a git dependency that ships prebuilt `lib/` (no `prepare` script, so `dsh plugin add` does not hit pnpm `allowBuilds`). The package also declares `dsh.bundle`, so `dsh plugin add` appends `@nguyenvanhoang1/dsh-zpdf` to the profile's `dsh.profile.bundles` and the plugin mounts on the next start — settings page and tools appear without a hand-written composition row.

Host packages such as `@deepseek-ai/dsh-tools` are **optional peer** dependencies (same pattern as [dsh-ads](https://github.com/Nagi-ovo/dsh-ads)): every host peer is marked optional in `peerDependenciesMeta`, so pnpm never reports "missing peer" at install time, while the runtime contract still requires them to resolve to the running Harness copy (module fallback stack). Shipping a second copy inside this plugin makes every tool call fail with `Cannot read properties of undefined (reading 'prepare')`.

For local development:

```bash
# Local checkout
dsh plugin --profile web add D:/code/work-relate/dsh-zpdf
```

`dsh plugin add` already activates the bundle layer. Only add a host-plane row yourself if you installed the package some other way:

```yaml
- insert:
    - id: zpdf
      name: '@nguyenvanhoang1/dsh-zpdf'
```

## Publishing

npm publishing is paused. Install from GitHub using the commands above; the new package name is `@nguyenvanhoang1/dsh-zpdf` and its npm scope has not been configured for publishing.

Before enabling `Release & Publish` (`.github/workflows/release.yml`), establish npm publishing access for this scope and configure npm Trusted Publishing for owner `nguyenvanhoang1`, repository `dsh-zpdf`, and workflow `release.yml`. The workflow uses OIDC, not an `NPM_TOKEN` secret. Previously published npm packages are not changed by this repository migration.

`CI` runs typechecks, tests, and builds. `Rebuild lib` keeps committed `lib/` artifacts in sync for Git installations, which do not run prepare scripts.

See [`examples/cordis.patch.yml`](examples/cordis.patch.yml) and the package-root [`cordis.patch.yml`](cordis.patch.yml).

Restart the profile:

```bash
dsh web
```

> `dsh plugin ... add` installs the dependency **and**, because this package declares `dsh.bundle`, joins the profile layer stack. A plain `pnpm add` without `dsh plugin` does not.

## Configure the API key

### Option 1: Web Settings (recommended)

In the DeepSeek Harness Web GUI:

1. Open **Settings**.
2. Open the **ZPDF** section.
3. Enter the API key and save.

The page writes the key into the DSH credential store (`$DSH_HOME/.credentials.yaml`, reference `ZPDF_API_KEY`). It does not go through the settings-document allowlist. If `ZPDF_API_KEY` is already in the process environment, the page is read-only (env wins and cannot be overwritten).

A missing key does not prevent the plugin from starting. The first authenticated tool call returns an actionable prompt.

### Option 2: CLI

After install, run the CLI through DeepSeek Harness so it resolves from the profile (a bare `zpdf` is not on PATH):

```bash
dsh plugin --profile web exec zpdf -- config set-key
```

This reads the key with a masked prompt and writes `zpdf.apiKey` in `$DSH_HOME/settings.yaml` (default `~/.dsh/settings.yaml`).

```bash
# Non-interactive (lands in shell history; not recommended)
dsh plugin --profile web exec zpdf -- config set-key sk-xxxxxxxxxxxxxxxx

# Scripts / CI: read from stdin
printf '%s' "$ZPDF_API_KEY" | dsh plugin --profile web exec zpdf -- config set-key

# Status (never prints the key)
dsh plugin --profile web exec zpdf -- config status

# Settings file path
dsh plugin --profile web exec zpdf -- config path

# Clear the stored key
dsh plugin --profile web exec zpdf -- config clear-key

# Custom settings file
dsh plugin --profile web exec zpdf -- config set-key --file D:/path/to/settings.yaml
```

The CLI preserves YAML comments, uses the same `<file>.lock` writer lock and atomic replace as DSH (`@deepseek-ai/dsh-atomic-write`), and sets owner-only permissions (`0600`; Windows still uses ACLs). A running DSH with file watch enabled hot-reloads the change.

### Option 3: Environment variable

```bash
export ZPDF_API_KEY=sk-xxxxxxxxxxxxxxxx
```

PowerShell:

```powershell
$env:ZPDF_API_KEY = 'sk-xxxxxxxxxxxxxxxx'
dsh web
```

Set the variable before starting DSH. You can change the name with `apiKeyEnv` in the composition.

Resolution order: CLI `settings.apiKey` → credential / environment (`ZPDF_API_KEY`).

## Optional config

Composition `config` is the base layer; the user settings document still overrides it:

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

| Field | Default | Notes |
| --- | --- | --- |
| `apiKey` | unset | Secret. Prefer GUI credentials or CLI; do not commit this |
| `apiKeyEnv` | `ZPDF_API_KEY` | Environment / credential reference name |
| `baseUrl` | `https://www.zhiyipdf.com` | ZPDF API origin |
| `outputDir` | `./zpdf-output` | Result directory (relative to DSH cwd) |
| `pollIntervalMs` | `2000` | Status poll interval |
| `maxPollMinutes` | `30` | Maximum poll duration |
| `httpTimeoutMs` | `60000` | Ordinary HTTP timeout |
| `uploadTimeoutMs` | `600000` | Upload / download timeout |

## Development

```bash
corepack enable pnpm
pnpm install
pnpm check
```

`pnpm check` runs TypeScript, Vitest, and the production build.

The optional host-composition test needs the installed Harness package directory:

```bash
DSH_ROOT=/path/to/node_modules/@deepseek-ai/dsh pnpm test:host
```

## Security

- Do not put a real API key in the repo, README, screenshots, or issues.
- The Web credentials API reports only configured / writable state, never the secret.
- The CLI `status` command only prints configured / not configured.
- Tools read local PDF/Markdown/ZIP paths from the model and write under `outputDir`. Enable this plugin only in a trusted composition.
- Output directories are checked with `realpath` so `output_subdir` and symlinks cannot escape `outputDir`.
- ZIP extraction rejects absolute paths and `../` entries. Caps: 10,000 entries, 4 GiB uncompressed, 2 GiB download.
- Inputs are capped at 300 MB and 800 pages (same as the ZPDF service) and validated locally before upload.

## License

MIT

