# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

x-ui 是一个支持多协议多用户的 xray 面板（Go 编写）。它通过 Web 界面管理 xray-core 进程：管理 inbound（vmess/vless/trojan/shadowsocks/dokodemo-door/socks/http）、流量统计与限额、到期管理、Telegram 通知、一键 SSL 等。本质上是一个「配置 + 拉起并监控 xray 子进程」的管理面板。

## 构建与运行

本项目部署运行于 Linux 服务器，构建在 Linux（或 macOS）上进行。

- **构建**：`CGO_ENABLED=1 go build main.go`
- **CGO 与 sqlite**：`gorm.io/driver/sqlite` 依赖 cgo 版 `mattn/go-sqlite3`，需要 gcc。注意——**关闭 CGO 不会导致编译失败**（`CGO_ENABLED=0` 也能编译通过），但生成的二进制在**运行时**操作数据库会 panic（sqlite 需要 cgo）。因此可运行/发版的二进制必须 `CGO_ENABLED=1`。
- **交叉编译 arm64**：`CC=aarch64-linux-gnu-gcc CGO_ENABLED=1 GOOS=linux GOARCH=arm64 go build main.go`。
- **运行 Web 面板**：`./main` 或 `./main run`（无参数默认等同 `run`）。
- **本地开发热加载模板/静态资源**：设置 `XUI_DEBUG=true`，此时 gin 从磁盘 `web/html`、`web/assets` 读取（而非 embed），并开启 DEBUG 日志和 gin DebugMode。生产模式使用 `//go:embed` 内嵌资源。
- **日志级别**：环境变量 `XUI_LOG_LEVEL`（debug/info/warn/error，默认 info；DEBUG 模式强制 debug）。
- **运行 xray 需要 bin/ 目录**：`bin/xray-{GOOS}-{GOARCH}`、`bin/geoip.dat`、`bin/geosite.dat`。仓库自带 `xray-linux-amd64/arm64`。运行时 xray 配置写入 `bin/config.json`。
- **数据库路径**：硬编码为 `/etc/x-ui/x-ui.db`（见 `config.GetDBPath()`，由 `config/name` 文件内容派生），启动时自动创建该目录。

### 命令行子命令（main.go）
- `setting`：读写设置，例如 `./main setting -port 8080 -username admin -password xxx`、`-reset`、`-show`、`-tgbottoken/-tgbotchatid/-tgbotRuntime/-enabletgbot`。直接操作 DB，不启动面板。
- `v2-ui -db <path>`：从旧版 v2-ui 数据库迁移 inbound 账号（`v2ui/` 包）。
- `-v`：打印版本（`config/version` 文件）。

### 测试
仓库当前**没有任何 `*_test.go` 测试文件**，也没有 lint 配置。新增测试时遵循全局规范。

## 架构

启动流程：`main.go:runWebServer` → `database.InitDB`（GORM 打开 sqlite + AutoMigrate）→ `web.NewServer().Start()`。

### 分层（web/）
- **controller/**：gin 路由与 HTTP 处理。`BaseController.checkLogin` 做会话鉴权中间件。`XUIController`(`/xui`) 挂载 `InboundController` 与 `SettingController`；`IndexController` 处理登录/登出；`ServerController` 提供系统状态/xray 日志等。所有受保护路由都在 `g.Use(a.checkLogin)` 下。
- **service/**：业务逻辑，**所有 service 都是无字段的空 struct**（如 `InboundService{}`、`SettingService{}`），按需 `new` 即可，数据库通过 `database.GetDB()` 全局获取。这是本项目的核心惯例——不要给 service 引入实例状态。
- **database/model/**：GORM 模型 `User`、`Inbound`、`Setting`。
- **entity/**：请求/响应 DTO（如 `AllSetting`）。
- **job/**：定时任务（见下）。
- **session/**：基于 gin-contrib/sessions（cookie store）的登录态。
- **network/**：`auto_https_listener` 自动将 HTTP 请求在 HTTPS 端口上重定向。

### xray 进程管理（xray/ + web/service/xray.go）
- `XrayService` 用**包级全局变量** `var p *xray.Process` + `sync.Mutex lock` 管理唯一的 xray 子进程。这是关键设计：整个进程只有一个 xray 实例。
- `GetXrayConfig()`：读取设置中的 xray 模板（`web/service/config.json`，存为 setting `xrayTemplateConfig`），把所有启用的 inbound 通过 `Inbound.GenXrayInboundConfig()` 追加进 `InboundConfigs`，生成完整 xray 配置。
- `RestartXray(isForce)`：生成新配置，若非强制且 `p.GetConfig().Equals(newConfig)` 相等则跳过重启（`Config.Equals` 逐字段比较 `json.RawMessage`）。修改 inbound/设置后通过 `SetToNeedRestart()` 标记，由定时任务择机重启。
- `Process.Start()`（xray/process.go）：将配置写入 `bin/config.json`，`exec.Command` 拉起 xray，goroutine 读取 stdout/stderr（环形缓冲 100 行）。
- 流量统计：通过 gRPC 连接 xray 的 stats command API（`apiPort` 从配置中 tag=="api" 的 inbound 解析），`trafficRegex` 解析 `inbound>>>tag>>>traffic>>>uplink/downlink`。

### 设置存储（web/service/setting.go）
设置以 **key/value 字符串行**存于 `Setting` 表。`GetAllSetting` 用反射把 DB 行按 json tag 映射到 `entity.AllSetting` 字段并做类型转换（int/string/bool）。`defaultValueMap` 定义默认值。新增设置项需同时改 `defaultValueMap`、`entity.AllSetting`、以及对应 getter/setter。

### 定时任务（web.startTask + web/job/）
`cron.New(WithSeconds())` 调度：
- `@every 30s` `CheckXrayRunningJob`：xray 挂了则拉起。
- `@every 10s`（启动延迟 5s）`XrayTrafficJob`：拉流量并入库。
- `@every 30s` `CheckInboundJob`：检查 inbound 流量超额/到期并禁用。
- Telegram 通知 `StatsNotifyJob`：仅当 `tgBotEnable` 为 true 时按用户配置的 cron（`tgRunTime`）注册。

### 进程信号
`main.go` 捕获信号：`SIGHUP` 触发**优雅重启**（Stop 旧 server → new server → Start，用于配置热生效），其它信号退出。

## 国际化
i18n 用 `go-i18n` + toml（`web/translation/translate.{en_US,zh_Hans,zh_Hant}.toml`），默认简体中文。模板里通过 `{{ i18n "key" }}` 调用，按 `Accept-Language` 头本地化。

## 前端
无构建步骤。`web/html/`（Go template）+ `web/assets/` 直接内嵌：Vue 2.6 + ant-design-vue 1.7 + element-ui 2.15 + axios，全部为预置静态文件（vendored）。

## 约定与注意事项
- 代码注释与用户可见字符串大量使用中文，保持一致。
- service 保持无状态空 struct；全局单例（xray 进程、db、webServer）通过包级变量管理。
- 版本号在 `config/version` 文件中（发版打 `0.*` tag 触发 `.github/workflows/release.yml`，自动下载最新 Xray-core 与 geo 数据打包）。
