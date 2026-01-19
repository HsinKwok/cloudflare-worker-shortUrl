# cloudflare-worker-shortUrl
使用Trea生成的，部署于Cloudflare Workers部署的短网址生成器，附两个版本，演示为带API版本。
# Cloudflare Workers 短链接服务

## 功能概述
- 生成与管理短链接，支持过期时间、跳转类型、访问密码
- 管理后台（/admin）登录仅需密码，支持导入、删除、查看日志与统计
- API 创建短链接（/api/create），需先添加授权 API Key
- 访问短码时显示跳转页，默认 3 秒自动跳转；支持自定义 HTML 模板
- worker.js带API、跳转模板、有效期设定，
- noApi.js只有基础的短址功能，且有效期默认为90天，无模板设定

## 环境变量
- URL_KV：KV 命名空间绑定名称（必须在 Dashboard 里绑定）
- JWT_SECRET：JWT 签名密钥，必填
- ADMIN_PASSWORD：后台登录密码，必填
- ADMIN_USERNAME：后台展示用户名，选填，默认 admin
- **以上为noApi.js基础变量，使用worker.js还需加上以下变量**
- MAX_SHORTCODE_LENGTH：短码最大长度，选填，默认 16，最大 64
- DEFAULT_EXPIRE_DAYS：默认过期天数，选填，默认 90（0 表示永久）
- FRONTEND_EXPIRE_SELECT_ENABLED：前台是否显示过期天数输入，选填，默认 true
- MAX_EXPIRE_DAYS：过期天数上限，选填，默认 3650
- REDIRECT_HTML_TEMPLATE：跳转页模板（可选），支持占位符 {{URL}}、{{CODE}}、{{TYPE}}

## 存储键约定（KV）
- 短链：shortCode → { url, expireAt, password, redirectType }
- 访问统计：stats_{shortCode} → 访问次数
- 访问日志：log_{shortCode}_{timestamp} → { ip, userAgent, time }
- API Key：api_key_{key} → "1"

## 部署步骤（无需 wrangler.toml）
1. 创建 KV 命名空间（例如：url-kv）
2. 创建 Worker：将项目中的 `worker.js` 代码粘贴到 Cloudflare Dashboard → Workers → 创建服务
3. 绑定变量与 KV：
   - Settings → Variables → KV Namespace Bindings：添加绑定，变量名填 `URL_KV`，命名空间选择步骤 1 创建的命名空间
   - Settings → Variables → Add variable：添加并设置上方列出的环境变量（至少 `JWT_SECRET`、`ADMIN_PASSWORD`）
   - 可选：将 `sample_redirect.html` 的全部内容粘贴到 `REDIRECT_HTML_TEMPLATE`，作为自定义跳转页模板，包含 {{URL}}/{{CODE}}/{{TYPE}} 占位符
4. 保存并部署
5. 访问根路径 `/` 测试前台生成；访问 `/admin` 测试后台登录与管理

## 管理后台
- 路径：/admin
- 登录：POST /admin，body `{ action: "login", password: "<ADMIN_PASSWORD>" }`，成功返回 `token`
- 后续管理操作需在 `Authorization: Bearer <token>` 下执行
- 支持添加/编辑/删除短链、批量导入 CSV、查看访问日志、管理 API Key

## API
- 创建短链：GET 或 POST `/api/create`
  - 参数：`key`（后台添加的 API Key）、`url`、`expireDays`、`redirectType`（301 或 302）、`password`（可选）
  - 返回：`{ shortCode, shortUrl, expireAt }`
  - 说明：若未提供 `shortCode`，后端将按策略生成未占用的随机短码
- 查询短链：GET `/api/shorten?key={shortCode}`
  - 返回短链的完整存储内容（JSON）

## API 示例
- GET 创建（省略 shortCode，后端自动生成）

  ```bash
  curl "https://your-worker.example.workers.dev/api/create?key=MYCODE&url=https%3A%2F%2Fexample.com&expireDays=7&redirectType=302"
  ```

  返回示例：
  ```json
  { "shortCode": "abc123", "shortUrl": "https://your-worker.example.workers.dev/abc123", "expireAt": "2026-01-26T00:00:00.000Z" }
  ```

- POST 创建（JSON 请求体，自定义短码）

  ```bash
  curl -X POST "https://your-worker.example.workers.dev/api/create" \
    -H "Content-Type: application/json" \
    -d "{\"key\":\"MYCODE\",\"code\":\"custom01\",\"url\":\"https://example.com/page\",\"expireDays\":30,\"redirectType\":302}"
  ```

  返回示例：
  ```json
  { "shortCode": "custom01", "shortUrl": "https://your-worker.example.workers.dev/custom01", "expireAt": "2026-02-18T00:00:00.000Z" }
  ```

- 查询短链内容

  ```bash
  curl "https://your-worker.example.workers.dev/api/shorten?key=abc123"
  ```

  返回示例：
  ```json
  { "url": "https://example.com", "expireAt": "2026-01-26T00:00:00.000Z", "password": "", "redirectType": 302 }
  ```

- 错误示例（未授权的 API Key）

  ```bash
  curl "https://your-worker.example.workers.dev/api/create?key=BADKEY&url=https%3A%2F%2Fexample.com"
  ```

  响应：`未授权的 API key`，HTTP 403

## 跳转与密码
- 访问 `/{shortCode}`：
  - 若存在 `password`，需通过 `?pwd=...` 验证；否则返回密码验证页
  - 若已过期则返回 410 并清理相关 KV
  - 默认跳转页为内联样式，显示短码与目标地址，3 秒自动跳转
  - 若设置了 `REDIRECT_HTML_TEMPLATE`，且包含 HTML 内容，将优先渲染该模板（占位符 {{URL}}/{{CODE}}/{{TYPE}}）

## 字符与校验
- 短码字符集：大小写字母与数字 `[A-Za-z0-9]`
- 长度范围：2 至 `MAX_SHORTCODE_LENGTH`
- 过期天数：不得超过 `MAX_EXPIRE_DAYS`
- URL：必须为 http 或 https

## 本地示例
- `sample_redirect.html`：示例跳转页模板，包含占位符，可直接用于配置 `REDIRECT_HTML_TEMPLATE`

## 代码位置
- 核心逻辑与路由：`worker.js`
- 示例模板：`sample_redirect.html`
