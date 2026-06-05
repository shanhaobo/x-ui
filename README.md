# x-ui

> 支持多协议、多用户的 xray 面板，网页可视化操作，开箱即用。

[![releases](https://img.shields.io/github/v/release/shanhaobo/x-ui?style=flat-square)](https://github.com/shanhaobo/x-ui/releases)
[![license](https://img.shields.io/github/license/shanhaobo/x-ui?style=flat-square)](./LICENSE)

本仓库 fork 自 [vaxilu/x-ui](https://github.com/vaxilu/x-ui)，下载地址、自动构建流程已全部改为指向本仓库，可独立安装与发布。

---

## 目录

- [这是什么](#这是什么)
- [功能特性](#功能特性)
- [一、快速安装（小白首选）](#一快速安装小白首选)
- [二、安装后如何使用](#二安装后如何使用)
- [三、其他安装方式](#三其他安装方式)
- [四、进阶：用 GitHub Actions 发布你自己的版本](#四进阶用-github-actions-发布你自己的版本)
- [五、SSL 证书申请](#五ssl-证书申请)
- [六、Telegram 机器人通知](#六telegram-机器人通知)
- [七、从 v2-ui 迁移](#七从-v2-ui-迁移)
- [系统要求](#系统要求)
- [常见问题](#常见问题)

---

## 这是什么

x-ui 是一个跑在 **Linux 服务器**上的网页管理面板，用来管理代理工具 [xray-core](https://github.com/XTLS/Xray-core)。
装好后，你用浏览器打开一个网址就能在网页上添加账号、查看流量、限速限时，而不用手动改配置文件。

## 功能特性

- 系统状态监控（CPU / 内存 / 网络等）
- 多用户、多协议，网页可视化操作
- 支持协议：vmess、vless、trojan、shadowsocks、dokodemo-door、socks、http
- 流量统计、限制流量、限制到期时间
- 可自定义 xray 配置模板
- 支持 https 访问面板（自备域名 + ssl 证书）
- 支持一键申请 SSL 证书并自动续签
- 支持 Telegram 机器人通知

---

## 一、快速安装（小白首选）

> 你需要一台 **Linux 服务器**（VPS），并用 **root 用户**登录它的命令行（SSH）。

把下面这一整行复制到服务器命令行里，回车运行：

```bash
bash <(curl -Ls https://raw.githubusercontent.com/shanhaobo/x-ui/main/install.sh)
```

### 安装时会发生什么？

1. 脚本会自动下载并安装 x-ui 与 xray。
2. 出于安全考虑，安装完会**提示你修改端口和账号密码**：
   - 输入 `y` 回车 → 按提示依次输入新的「账号名 / 密码 / 面板端口」。
   - 输入 `n` 回车 → 使用默认值（**强烈建议安装后尽快自行修改**）。

| 项目 | 默认值 |
| --- | --- |
| 面板端口 | `54321` |
| 用户名 | `admin` |
| 密码 | `admin` |

### 打开面板

在浏览器访问：

```
http://你的服务器IP:端口
```

例如默认端口就是 `http://1.2.3.4:54321`，用你设置的账号密码登录即可。

> ⚠️ 如果打不开，多半是**服务器防火墙 / 云服务商安全组**没有放行该端口，请到云控制台放行你的面板端口。

---

## 二、安装后如何使用

安装完成后，服务器上多了一个 `x-ui` 命令。直接输入 `x-ui` 回车会出现**管理菜单**（功能最全，推荐小白用）：

```bash
x-ui
```

也可以直接用以下快捷命令：

| 命令 | 作用 |
| --- | --- |
| `x-ui` | 显示管理菜单（功能更多） |
| `x-ui start` | 启动面板 |
| `x-ui stop` | 停止面板 |
| `x-ui restart` | 重启面板 |
| `x-ui status` | 查看运行状态 |
| `x-ui enable` | 设置开机自启 |
| `x-ui disable` | 取消开机自启 |
| `x-ui log` | 查看日志 |
| `x-ui update` | 更新到最新版（数据不丢失） |
| `x-ui install` | 安装面板 |
| `x-ui uninstall` | 卸载面板 |
| `x-ui v2-ui` | 从本机 v2-ui 迁移账号数据 |

> 忘记账号密码 / 端口？在服务器运行 `x-ui` 进入菜单，里面有「重置用户名密码」「查看 / 修改面板端口」等选项。

---

## 三、其他安装方式

### Docker 安装

> Docker 配置思路参考 [Chasing66](https://github.com/Chasing66)。
> 这里**用项目自带的 `Dockerfile` 自行构建镜像**：镜像里的 x-ui 与 xray 全部来自本仓库源码，代码完全可审计，**不依赖任何第三方镜像**，避免供应链投毒风险。

```bash
# 1. 安装 docker
curl -fsSL https://get.docker.com | sh

# 2. 拉取本项目源码
git clone https://github.com/shanhaobo/x-ui.git
cd x-ui

# 3. 用自带 Dockerfile 构建镜像
docker build -t x-ui .

# 4. 运行（数据库与证书持久化到当前目录的 db/、cert/，容器重建也不丢数据）
docker run -itd --network=host \
    -v $PWD/db/:/etc/x-ui/ \
    -v $PWD/cert/:/root/cert/ \
    --name x-ui --restart=unless-stopped \
    x-ui
```

> **更新版本**：`git pull` 拉取最新源码后，重新构建并重建容器（挂载在宿主 `db/`、`cert/` 的数据不受影响）：
> ```bash
> cd x-ui && git pull
> docker build -t x-ui .
> docker rm -f x-ui
> # 再执行上面第 4 步的 docker run
> ```

### 手动安装 / 升级

1. 从 [Releases](https://github.com/shanhaobo/x-ui/releases) 下载最新压缩包（一般选 `amd64` 架构）。
2. 上传到服务器 `/root/` 目录，用 root 登录后执行：

```bash
cd /root/
rm x-ui/ /usr/local/x-ui/ /usr/bin/x-ui -rf
tar zxvf x-ui-linux-amd64.tar.gz
chmod +x x-ui/x-ui x-ui/bin/xray-linux-* x-ui/x-ui.sh
cp x-ui/x-ui.sh /usr/bin/x-ui
cp -f x-ui/x-ui.service /etc/systemd/system/
mv x-ui/ /usr/local/
systemctl daemon-reload
systemctl enable x-ui
systemctl restart x-ui
```

> CPU 架构不是 `amd64` 的，把命令里的 `amd64` 换成对应架构（如 `arm64`）。

---

## 四、进阶：用 GitHub Actions 发布你自己的版本

如果你 **fork（复制）了本项目**，想让上面的一键安装命令安装的是「你自己仓库的代码」，需要让 GitHub 帮你自动编译出安装包。下面是**零基础逐步教程**。

> 原理：GitHub Actions 是 GitHub 自带的「自动打包机器人」。你给仓库打一个版本号标签（tag），它就会自动编译 amd64 / arm64 / s390x 三种安装包，并发布到你仓库的 Releases 页面。一键安装脚本就是从那里下载安装包的。

### 步骤 1：Fork 本仓库

打开本仓库页面，点右上角 **Fork**，把它复制到你自己的账号下。
（如果这本来就是你自己的仓库，跳过这步。）

### 步骤 2：把下载地址改成你的用户名

脚本里写死了仓库地址。把以下文件中的 `shanhaobo` 全部替换成**你的 GitHub 用户名**：

- `install.sh`
- `x-ui.sh`
- `README.md`（本文件里的安装命令）

> 最简单的做法：在 GitHub 网页打开文件 → 点铅笔图标编辑 → 用浏览器查找替换 → 提交。

### 步骤 3：开启 Actions 并给它写权限（最容易漏！）

GitHub 对 fork 来的仓库**默认关闭 Actions**，必须手动打开：

1. 进入你的仓库 → 顶部 **Settings**（设置）。
2. 左侧菜单 **Actions → General**。
3. 「Actions permissions」选 **Allow all actions and reusable workflows**（允许所有），保存。
4. 同一页面往下找 「Workflow permissions」，选 **Read and write permissions**（读写权限），保存。

> 不做第 4 步，机器人会因为没权限发布失败。

### 步骤 4：打一个版本标签，触发自动构建

标签**必须以 `0.` 开头**（例如 `0.3.2`、`0.4.0`），否则不会触发。
在你电脑的命令行里（已 `git clone` 你的仓库）执行：

```bash
git tag 0.3.2
git push origin 0.3.2
```

> 不会用命令行？也可以在 GitHub 网页：进入 **Releases → Draft a new release → Choose a tag** 输入 `0.3.2` 新建标签并 Publish，同样会触发。

### 步骤 5：等待并查看结果

1. 进入仓库顶部 **Actions** 标签页，能看到一个名为 **Release X-ui** 的任务在运行。
2. 等待约 **3～5 分钟**，所有项变成绿色 ✅ 即成功。
3. 进入仓库 **Releases** 页面，应能看到新版本，下面挂着三个 `.tar.gz` 安装包。

### 步骤 6：用你自己的安装命令

把命令里的 `shanhaobo` 换成你的用户名，发给需要安装的人即可：

```bash
bash <(curl -Ls https://raw.githubusercontent.com/你的用户名/x-ui/main/install.sh)
```

### 常见报错对照

| 现象 | 原因 / 解决 |
| --- | --- |
| Actions 页面没有任何任务 | fork 仓库未启用 Actions，回到**步骤 3** 打开 |
| `Create Release` 失败，提示权限 / 403 | Workflow 权限是只读，按**步骤 3 第 4 步**改为读写 |
| 打了标签却没触发 | 标签没有以 `0.` 开头 |
| 安装时提示「检测 x-ui 版本失败」 | Releases 里还没有正式发布的版本，确认步骤 4、5 已成功 |

---

## 五、SSL 证书申请

> 此功能与教程由 [FranzKafkaYu](https://github.com/FranzKafkaYu) 提供。

脚本内置 SSL 证书申请功能，使用前需满足：

- 知晓 Cloudflare 注册邮箱
- 知晓 Cloudflare Global API Key
- 域名已通过 Cloudflare 解析到当前服务器

获取 Cloudflare Global API Key 的方法：

![](media/bda84fbc2ede834deaba1c173a932223.png)
![](media/d13ffd6a73f938d1037d0708e31433bf.png)

使用时只需输入 `域名`、`邮箱`、`API KEY` 即可：

![](media/2022-04-04_141259.png)

注意事项：

- 使用 DNS API 进行证书申请
- 默认使用 Let's Encrypt 作为 CA
- 证书安装目录为 `/root/cert`
- 申请的均为泛域名证书

---

## 六、Telegram 机器人通知

> 此功能与教程由 [FranzKafkaYu](https://github.com/FranzKafkaYu) 提供。

X-UI 支持通过 Telegram 机器人实现每日流量通知、面板登录提醒等。需自行申请机器人，
申请教程可参考[此博客](https://coderfan.net/how-to-use-telegram-bot-to-alarm-you-when-someone-login-into-your-vps.html)。

在面板后台填写以下参数：

- Tg 机器人 Token
- Tg 机器人 ChatId
- Tg 机器人运行周期（crontab 语法）

运行周期语法示例：

| 写法 | 含义 |
| --- | --- |
| `30 * * * * *` | 每一分钟的第 30 秒通知 |
| `@hourly` | 每小时通知 |
| `@daily` | 每天通知（凌晨零点） |
| `@every 8h` | 每 8 小时通知 |

通知内容：节点流量使用、面板登录提醒、节点到期提醒、流量预警提醒。

---

## 七、从 v2-ui 迁移

在装有 v2-ui 的服务器上先安装最新版 x-ui，然后执行以下命令迁移本机 v2-ui 的
**所有 inbound 账号数据**至 x-ui（`面板设置和用户名密码不会迁移`）：

```bash
x-ui v2-ui
```

> 迁移成功后请 **关闭 v2-ui** 并 **重启 x-ui**，否则两者的 inbound 会发生**端口冲突**。

---

## 系统要求

- CentOS 7+
- Ubuntu 16+
- Debian 8+

---

## 常见问题

- **打不开面板**：检查云服务商安全组 / 防火墙是否放行了面板端口。
- **忘记账号密码或端口**：服务器运行 `x-ui` 进入菜单，使用重置 / 查看相关选项。
- **更新会丢数据吗**：`x-ui update` 只更新程序，数据库（账号、设置）保留。

---

## Stargazers over time

[![Stargazers over time](https://starchart.cc/shanhaobo/x-ui.svg)](https://starchart.cc/shanhaobo/x-ui)
