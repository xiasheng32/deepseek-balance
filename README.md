# deepseek-balance

> 一个 [pi](https://pi.dev) 扩展：在页脚状态栏实时显示 DeepSeek 账户余额，并提供 `/deepseek-balance` 命令查看详情面板。

## 功能特性

- **页脚常驻状态**：显示 `🐳 DeepSeek 余额：¥110.00`，并按可用性着色（可用 / 不可用 / 未配置）。
- **`/deepseek-balance` 命令**：强制刷新并弹出详情面板，展示每个币种的**总余额、赠送余额、充值余额**与账户状态。
- **自动刷新**：会话启动时查询，之后每 60 秒轮询并在每次 Agent 结束后按 30 秒节流刷新；失败按指数退避重试，单次请求 10 秒超时。
- **无硬编码密钥**：凭据统一从 pi 的模型注册表读取，仓库中不含任何 key。

## 目录结构

```
deepseek-balance/
├── package.json                 # pi 包清单（keywords: pi-package）
├── extensions/
│   └── deepseek-balance.ts      # pi 接线：事件、命令、详情面板
├── src/
│   └── balance.ts               # 纯逻辑：刷新策略与状态文案（零外部依赖）
├── scripts/
│   └── sync-autoload.mjs        # 显式生成自包含副本；默认写入 autoload
├── tests/
│   └── deepseek-balance.test.mjs
├── .env.example                 # DEEPSEEK_API_KEY 占位示例
├── .gitignore
├── LICENSE
└── README.md
```

## 安装

需要 Node.js >=22.19 和提供 `modelRegistry.getApiKeyForProvider()` 的 pi；已验证的宿主版本为 **1.0.4**。无需额外安装运行时或测试依赖，pi 提供宿主包。

### 方式一：本地路径（开发 / 试用）

克隆项目后，在包含 `package.json` 的项目根目录运行（解开 `.tgz` 后则进入 `package/`）：

```bash
cd deepseek-balance

# 仅本次离线试用，不加载其他扩展、不写配置、不解析凭据或查询余额
pi --offline -ne -e .

# 仅本次在线试用：交互式启动会查询真实账户
pi -ne -e .

# 确认后安装为常驻扩展（这一步会写入 pi 包配置）
pi install .
```

安装后重新启动 pi，或在已有会话中执行 `/reload`；`pi list` 可查看已配置的包。

### 方式二：从 GitHub 安装

```bash
pi install git:github.com/xiasheng32/deepseek-balance
```

### 方式三：手动放置到 autoload 目录

`extensions/deepseek-balance.ts` 依赖相对的 `../src/balance.ts`，不能直接复制。请在项目根目录显式生成自包含文件：

```bash
npm run sync-autoload
# 默认覆盖 ~/.pi/agent/extensions/deepseek-balance.ts，然后在 pi 中执行 /reload

# 或仅生成指定本地文件，不修改启用中的插件
npm run sync-autoload -- /tmp/deepseek-balance.ts
```

**包安装与独立 autoload 副本只能择一启用**，否则可能重复注册命令、状态栏与查询任务。已存在独立副本时，先通过 `pi config` 禁用旧来源，或备份后移出 autoload 目录；试用命令的 `-ne` 会隔离其他扩展。同步脚本只更新指定输出文件，不修改其他扩展或配置。

## 凭据配置

凭据统一由 pi 的模型注册表解析，插件不直接读取 `auth.json`，也不自行保存 API Key。通常选择以下一种方式：

1. 在 pi 中执行 `/login deepseek`，由 pi 保存和管理凭据；
2. 在启动 pi 的 shell 中导出环境变量（替换占位值）：

```bash
export DEEPSEEK_API_KEY='YOUR_DEEPSEEK_API_KEY'
pi
```

`.env.example` 仅提供变量名称和空值示例。**pi 与本插件不会自动加载项目的 `.env`**；仅复制文件不会配置凭据。如自行使用 `.env`，需在启动 pi 前由 shell/工具将其导出到环境。

> 请勿把真实密钥、`auth.json` 或本机配置提交到仓库。`.env` 已在 `.gitignore` 中忽略，分发示例保持空值。

## 使用

- 启动 `pi` 后，页脚会显示 DeepSeek 余额。
- 输入 `/deepseek-balance` 强制刷新并打开详情面板，`Enter` / `Esc`（或 `q`）关闭；内容超出面板时用 `↑` / `↓` 滚动，`Home` / `End` 跳至首尾。
- 未配置凭据时页脚显示 `🐳 DeepSeek 未配置`，`/deepseek-balance` 也会提示未配置。

## 刷新与状态

刷新策略与 `codex-limit` 保持一致，但本包独立运行，不依赖相邻项目：

- 交互式会话启动时非阻塞查询，之后每 60 秒轮询一次，并在 `agent_settled` 时尝试刷新；自动请求统一至少间隔 30 秒。
- 一次仅有一个刷新请求（并发调用共享同一个 Promise）；每次重新通过 pi 解析凭据，不缓存 API Key。凭据变更、401/403 或退出登录会清空旧数据。
- 单次请求 10 秒超时。额度失败后的自动重试从 60 秒起指数退避（60 → 120 → 240 秒），最高 5 分钟；429 会遵守 `Retry-After`（秒数或 HTTP 日期）进入冷却，手动刷新也不能绕过冷却。
- 刷新中页脚显示 `🐳 DeepSeek[更新中] 余额：…`（保留上一次数值，accent 色）；网络/数据错误保留上次成功数据并以 `🐳 DeepSeek[过期] 余额：…` 标注（warning 色），详情顶部提示“以下为过期缓存，不代表当前余额。”。过期标注优先于更新中标注。
- 未配置凭据显示 `🐳 DeepSeek 未配置` 且不进入退避，补上 Key 后下一个节流窗口即可恢复；账户不可用时以 warning 色显示并附 `⚠`。
- 详情时间统一为本机时区的 `YYYY-MM-DD HH:MM`。
- 会话关闭、切换和重载时清理定时器、取消请求并忽略迟到结果；状态按会话隔离，不使用模块级单例。

离线模式会连凭据解析一起跳过，手动刷新也不会联网：

```bash
pi --offline -ne -e .
# 或 PI_OFFLINE=1 pi -ne -e .
```

## 开发与测试

在项目根目录运行刷新策略的单元测试（注入 fetch 与合成凭据，不发起真实网络请求）：

```bash
npm test
```

检查离线启动选项，或隔离试用本包的 HUD 和命令：

```bash
pi --offline -ne -e . --help
pi --offline -ne -e .
```

帮助命令不能单独证明扩展加载成功。离线试用时应看到 `🐳 DeepSeek 离线`，`/deepseek-balance` 也不会查询凭据或余额。不要用 `-p "ok"` 做离线检查，它会调用模型。

修改 `src/balance.ts` 或 `extensions/deepseek-balance.ts` 后，包方式加载只需 `/reload`；单文件部署才需要显式运行 `npm run sync-autoload`。

## 安全与接口限制

- 仅向固定的 `https://api.deepseek.com/user/balance` 发送 GET，拒绝重定向；不会消费余额。
- 不记录 API Key、认证异常原文或完整响应；网络/数据失败只显示脱敏状态。
- 扩展与 pi 拥有相同的系统权限，请只加载可信源码。真实账户与网络访问不属于离线测试。

## 本地打包与上传 GitHub

在项目根目录运行：

```bash
npm pack --offline --ignore-scripts --pack-destination ..
# 上一级目录生成 deepseek-balance-0.1.0.tgz，不安装插件、不发布 npm
```

上传 GitHub 时使用本项目目录的内容，让 `package.json` 位于仓库根目录；不要把整个父目录、`.tgz`、凭据或本机配置上传。源码、测试、README、许可证、空值 `.env.example` 和 `.gitignore` 均可保留。包清单使用精确文件白名单，不需要 `node_modules/`；压缩包解开后也可直接运行 `npm test` 和 `pi -ne -e .`。

## 许可

[MIT](./LICENSE)。Copyright (c) 2026 xiasheng32。
