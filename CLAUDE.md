# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

本仓库是 V2Board（代理协议管理面板：前台 + 后台 + 节点 API）的 fork，origin 为 `OpenRealmCn/v2board`，默认分支 `master`。

> 本目录常与另外两个仓库并排放在工作区 `D:\poject\OpenRealmWork\v2bx` 下（`v2bx/` 节点服务端、`V2bX-script/` 安装脚本），三者**各自独立带 `.git`**。上下文中若同时存在这三者，工作区根还有一份 CLAUDE.md 记录跨仓库契约。本文件自包含，单独 clone 本仓库时也能用。

## 技术栈与命令

PHP `^7.3.0 || ^8.0`、Laravel 8、Horizon（Redis 队列）、MySQL。框架版本见 `composer.json`。

```bash
composer install                      # 依赖尚未安装（仓库里没有 vendor/）
php artisan v2board:install           # 交互式安装：写 .env、导入 database/install.sql、建管理员
php artisan v2board:update            # 执行 database/update.sql 并重启队列
php artisan horizon                   # 队列 worker（生产用 pm2.yaml 托管）
php -c cli-php.ini webman.php start   # Workerman/AdapterMan 常驻 HTTP 服务，监听 127.0.0.1:6600
vendor/bin/phpunit                    # 测试（tests/ 里目前只有 Laravel 自带的 Example 测试）
vendor/bin/phpunit --filter testBasicTest tests/Unit/ExampleTest.php   # 跑单个测试
```

PHP 8 环境下会额外装 `joanhey/adapterman`（`init.sh` / `update.sh` 里的条件分支），走 webman 常驻进程模式；PHP 7 则用传统 FPM。

`init.sh` / `update.sh` 是面向宝塔等生产环境的部署脚本，本地开发不要直接跑（`update.sh` 会 `git reset --hard origin/master`）。`cli-php.ini` 是为常驻进程准备的 php.ini，注意它 `disable_functions` 掉了 `header`/`setcookie`/`session_*` 等函数（Adapterman 自行处理这些），所以**不能用它跑需要这些函数的 CLI 脚本**。

## 数据库变更约定

**不要用 Laravel migration**（`database/migrations/` 只有一个框架自带的 `failed_jobs`）。这个项目把 schema 维护成裸 SQL：

- 全新安装：`database/install.sql`
- 增量升级：`database/update.sql`

两个文件都由 `V2boardInstall` / `V2boardUpdate` 命令按 `;` 切分后逐条 `DB::select(DB::raw(...))` 执行，**且用空的 `catch` 吞掉所有异常**——所以 SQL 必须幂等（`CREATE TABLE IF NOT EXISTS`、`ALTER TABLE ... ADD COLUMN` 重复执行不报错）。加了新字段要同时改 `install.sql` 和 `update.sql`。

表名统一 `v2_` 前缀，模型类名是去前缀后的驼峰（`v2_server_v2node` → `App\Models\ServerV2node`）。节点类型各有一张表：`ServerVmess` / `ServerVless` / `ServerTrojan` / `ServerShadowsocks` / `ServerTuic` / `ServerHysteria` / `ServerAnytls` / `ServerV2node`。

## `config('v2board.*')` 的来源

代码里到处都是 `config('v2board.app_name')`、`config('v2board.server_token')`，但**仓库里没有 `config/v2board.php`——它在 `.gitignore` 中**。该文件由后台「系统设置」保存时生成：`Admin\ConfigController@save` 用 `var_export()` 把配置写进 `config/v2board.php`，然后 `opcache_reset()` + `artisan config:cache` + `posix_kill(WEBMANPID, 15)` 让常驻的 webman 进程重启。

推论：新 clone 的仓库在跑完 `v2board:install` 或后台保存设置之前，所有 `config('v2board.*')` 都返回默认值。

可配置的键由 `app/Http/Requests/Admin/ConfigSave.php` 的 `RULES` 常量白名单限定。**加新配置项要同时改三处**：`RULES`、`ConfigController@fetch`（后台回显）、`ConfigController@save`。

主题配置 `config/theme.php` 同理是运行时生成的（`ThemeService` 负责初始化）。

## 路由不在 `routes/`

`routes/` 下只有 `web.php` / `console.php` / `channels.php`。**API 路由由 `RouteServiceProvider@mapApiRoutes` 用 glob 动态注册**：扫描 `app/Http/Routes/V1/*.php` 和 `app/Http/Routes/V2/*.php`，按文件名 `new` 出同名类并调用其 `map($router)`，分别挂到 `/api/v1` 和 `/api/v2` 前缀下。新增路由 = 在对应目录放一个带 `map(Registrar $router)` 方法的类。

`ServerRoute` 用的是**通配派发**而非显式路由：

```php
$router->any('/{class}/{action}', function($class, $action) {
    $ctrl = \App::make("\\App\\Http\\Controllers\\V1\\Server\\" . ucfirst($class) . "Controller");
    return \App::call([$ctrl, $action]);
});
```

也就是说 URL 段直接决定实例化哪个控制器、调用哪个 public 方法——**加一个 public 方法就等于加了一个对外端点，注意鉴权**。`V2/ServerRoute.php` 则是一个显式的 `/api/v2/server/config`。

## 鉴权模型

四类身份，各有中间件（`app/Http/Kernel.php` 的 `$routeMiddleware`）：`user` / `admin` / `staff` / `client`。

- **`client`** 中间件是给**节点后端**用的，靠 `?token=` 比对 `config('v2board.server_token')`；`UniProxyController` / `V2\ServerController` 则在构造函数里自己校验 token，并且**校验失败时用 HTTP 200 + `{"status":"fail"}` 而不是抛异常**（避免异常路径泄露信息）——改这里要保持同样的返回形态。
- **`admin` / `staff`** 依赖 `AuthService::decryptAuthData()` 解出的 JWT。**这里有过真实的越权漏洞**（见 `git log` commit `e695843d`）：`auth_data[is_staff]=1` 会被 PHP 解析成数组，数组传给 `Cache::get()` 时未命中的键会把请求原值回显，于是 `Cache::has()` 认为「已有登录信息」、JWT 校验被跳过。动 `AuthService`、`Staff`/`Admin` 中间件或任何 `Cache::get(用户可控输入)` 时要格外小心类型。

全局中间件栈里有 `CORS` 和 `UAfilter`；`api` 组含 `ForceJson` 和 `Language`。注意 `web` 组的 session / CSRF 中间件**全被注释掉了**，`api` 组也是——这个项目不用 session。

## 节点 API 契约（与 v2bx 仓库成对）

`app/Http/Controllers/V1/Server/UniProxyController.php` 是 v2bx 所用的端点，路径 `/api/v1/server/UniProxy/{user,push,alivelist,alive,config}`。构造函数里做节点类型归一化：

```php
if ($this->nodeType === 'v2ray') $this->nodeType = 'vmess';
if ($this->nodeType === 'hysteria2') $this->nodeType = 'hysteria';
```

`config` 方法按 `nodeType` 分派到每种协议的配置结构（`shadowsocks` / `vmess` / `vless` / `trojan` / `tuic` / `hysteria` / `anytls`）。**改这个契约要同步改 v2bx 仓库的 `api/panel/`，反之亦然。**

`V2/Server/ServerController.php` 是给 v2node 用的新版聚合端点（`/api/v2/server/config`），返回扁平结构并用 ETag/304 做条件请求。

## 其他约定

- **订阅协议生成器**在 `app/Protocols/`，`ClientController@subscribe` 通过 `glob()` 遍历该目录、按每个类的 `$flag` 属性匹配 User-Agent（或 `?flag=`）来选实现，命中即返回。**加一个客户端支持 = 加一个带 `$flag` 和 `handle()` 的类，无需改控制器。** 注意 UA 含 `sing` 时走单独的 `Singbox` / `SingboxOld` 分支（按 sing-box 版本号 `1.12.0` 分流），不参与 glob 匹配。
- **支付网关**在 `app/Payments/`，同样是按目录约定发现（AlipayF2F、EPay、Stripe 系列、BTCPay、CoinPayments 等）。
- **定时任务**全部集中在 `app/Console/Kernel.php@schedule()`（`traffic:update`、`check:order`、`check:commission`、`reset:traffic`、`v2board:statistics` 等），命令实现在 `app/Console/Commands/`。
- **队列走 Horizon**（Redis），`config/horizon.php`；`pm2.yaml` 只托管 `php artisan horizon` 一个进程。Job 在 `app/Jobs/`（`TrafficFetchJob`、`OrderHandleJob`、`SendEmailJob`、`SendTelegramJob`、`StatServerJob`、`StatUserJob`）。
- **缓存键**统一由 `app/Utils/CacheKey.php` 生成（如 `SERVER_V2NODE_LAST_CHECK_AT`），不要手写字符串键。
- 业务逻辑放 `app/Services/`（`ServerService`、`UserService`、`OrderService`、`PaymentService`…），控制器保持薄。
- **`ServerService` 是节点数据的唯一出口**：每种节点类型一对 `getAvailableXxx(User $user)` / `getAllXxx()`（Vless/Vmess/Trojan/Tuic/Hysteria/Shadowsocks/AnyTLS/V2node）。**新增节点类型要把两处都加上，并注册进 `getAvailableServers()` 的汇总与 `getAllServers()`。**
- `library/` 是 `Library\` 命名空间下的非 Laravel 遗留代码（`AlipayF2F.php`），PSR-4 根与 `app/` 不同。
- `public/theme/default/assets/umi.js` 是**压缩后的 webpack 构建产物**，前端源码不在本仓库（在 `v2board-OpenRealm` 仓库）。改前端需要去那边改源码再构建；直接改这个产物只适合应急修改字符串。

## 已迁移到 OpenRealmCn

上游引用已全量切换。检查这几处未被上游改回：

```bash
grep -rn "wyx2685" --exclude-dir=.git --exclude=CLAUDE.md .   # 应为空
```

涉及的文件：

- `app/Services/ServerService.php` — v2node 的安装命令 URL，指向 `OpenRealmCn/v2node` 的 **`main`** 分支（该 fork 默认分支是 `main`，不是 `master`）。
- `readme.md` — 三处链接改为 `OpenRealmCn`。
- `public/theme/default/assets/umi.js` — 压缩产物里的 GitHub 链接（已定点字节替换，文件 +4 字节、行数不变）。

同步上游时这些会被改回，合并后需重新检查。

## 相关仓库

- `OpenRealmCn/v2bx`（canonical 名 `OpenRealmCn/V2bX`，默认分支 **`dev_new`**）— 节点服务端，消费本仓库的 `/api/v1/server/UniProxy/*`。
- `OpenRealmCn/v2node`（默认分支 `main`）— 另一种节点端实现，走 `/api/v2/server/config`。
- `OpenRealmCn/V2bX-script`（默认分支 `master`）— V2bX 一键安装脚本。