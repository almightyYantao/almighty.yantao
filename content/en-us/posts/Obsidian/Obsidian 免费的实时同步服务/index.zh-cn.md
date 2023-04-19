---
title: Obsidian 免费的实时同步服务  
date: 2023-03-06  
tags: ['Obsidian']  
featured_image: https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061309538.png
toc: true
---

使用 fly.io 免费计划部署或自托管数据库，进行 LiveSync 插件的一系列配置后实现各设备间 Obsidian 实时增量修改同步，可以和官方同步服务相媲美。

<!-- more -->

## 使用 fly.io
这次使用的是 [**fly.io**](https://fly.io/) 的免费计划，fly.io 是一个 SAAS（是（Platform as a Service）的缩写，是指平台即服务）平台，可以搭建如静态博客、Nextjs、Nuxtjs、Deno、Go、Python 等底层的各种各样的服务。但首先需要自己注册一个账号，这里可以直接使用 Github 登录。
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061259670.png"/>
> **注意**：fly.io 的使用需要绑卡，如果没有绑卡会在[**创建应用**](#%e5%88%9b%e5%bb%ba%e5%ba%94%e7%94%a8)章节出现 **Error** 提示。绑卡过程请在 [**fly.io 面板**](https://fly.io/dashboard)中 **Billing** 进行。正常使用国内的双币卡就可以，注意请**如实**填写信息。没有卡的朋友可以去各银行办一张（超好过的😗）或试试虚拟卡？（虚拟卡只是博主想到的一种方案，**没有**试过）

### 安装 flyctl
Windows 用户在本地打开 PowerShell 或 Windows 终端💻，输入：
```powershell
iwr https://fly.io/install.ps1 -useb | iex
```

> 注意：CMD 中不支持上面的命令，如果电脑中只有 CMD，或许你需要安装 [**PowerShell**](https://github.com/PowerShell/PowerShell/releases)（选择 latest 版本）或 [**Windows 终端**](https://aka.ms/terminal)。

### 本地登录
```bash
flyctl auth signup
```
会自动打开浏览器进行验证账户操作。

### 创建应用

在本地任意位置创建一个 fly.io 的工作目录（其实就是创建个能找到的文件夹，你不会放桌面上了吧😂？）。

```bash
mkdir fly.io
cd fly.io
mkdir couchdb
cd couchdb
```

进入 couchdb 目录后输入命令：

```cmd
flyctl launch --image couchdb
```

这一步将会启动一个向导，按自己的需求进行选择。
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061301932.png"/>
**Select region** 的意思是选择一个位置，尽量选择**靠近**自己的位置。
### 配置卷大小
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061301125.png"/>
我输入的配置卷大小命令为：
```cmd
flyctl volumes create --region nrt couchdata --size 1
```

**nrt** 的意思是东京地区，你需要改变为和[**上面选择的位置区域**](#%e5%88%9b%e5%bb%ba%e5%ba%94%e7%94%a8)一样的位置代码。这一行命令的意思是：在东京地区创建一个 1G 大小的卷。

### 调整配置信息
打开应用根目录的 `fly.toml` 文件，添加或修改如下信息：
> 根目录为在[**创建应用**](#%e5%88%9b%e5%bb%ba%e5%ba%94%e7%94%a8)章节中创建的新文件夹 **couchdb**

博主是全部配置完毕后写这篇文章的😎，所有可能有遗漏或一些错误？为了保证严谨性，把自己的配置全部贴一份做对照吧：
```bash
# fly.toml file generated for couch-db on 2022-09-30T16:06:18+08:00

app = "couch-db"
kill_signal = "SIGINT"
kill_timeout = 5
processes = []

[build]
  image = "couchdb:latest"

[env]
  COUCHDB_USER = "yantao"

[mounts]
  source="couchdata"
  destination="/opt/couchdb/data"

[experimental]
  allowed_public_ports = []
  auto_rollback = true

[[services]]
  http_checks = []
  internal_port = 5984
  processes = ["app"]
  protocol = "tcp"
  script_checks = []
  [services.concurrency]
    hard_limit = 25
    soft_limit = 20
    type = "connections"

  [[services.ports]]
    force_https = true
    handlers = ["http"]
    port = 80

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 443

  [[services.ports]]
    handlers = ["tls", "http"]
    port = 5984

  [[services.tcp_checks]]
    grace_period = "1s"
    interval = "15s"
    restart_limit = 0
    timeout = "2s"

```

请对比上面提供的配置文件修改自己的 **fly.toml** 文件。
### 设置密码
在终端中输入命令：
```cmd
flyctl secrets set COUCHDB_PASSWORD=你的密码
```
> **注意**：密码使用大小写字母与数字，**不要**使用特殊字符。实际测试后发现使用特殊字符时无法识别密码，会无法登录数据库。

如果想要修改密码，可以再次运行上面的命令。
### 部署
> 注意，如果提示 **Services defined at indexes: 0 require a dedicated IP address. You currently have no dedicated IPs allocated. Please allocate at least one dedicated IP before deploying**，那么需要运行 `fly ips allocate-v4`。

在终端中输入命令：
```cmd
flyctl deploy
```
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061303907.png"/>
稍作等待后提示部署成功🎉。使用下面命令可以打开网页登录：
```cmd
flyctl open
```
如果是 fly.io 部署，那么在网址后加 `/_utils/#/setup`，跳转后可以输入用户名与密码。
网页成功登录，好耶🎉

## 网页数据库配置
这一大章节都在网页中进行。如果不知道自己的地址，可以打开 [**fly.io 面板**](https://fly.io/dashboard)，点击自己的应用，`Hostname` 处为自己应用的网址。
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061303825.png"/>

### 创建数据库
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061304360.png"/>

点击右上角的 `Create Database`，创建一个数据库，`Database name` 为数据库名字，`Partitioned` 请**不要**勾选，然后点 `Create` 创建。

### 配置其他信息
打开 `Setup` 选项卡，填写相关信息。
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061304896.png"/>

第一行的 `Specify your Admin credentials` 为你在上面步骤中配置的用户信息。第二行的 `Bind address the node will listen on` 意思是监听的访问地址，设置为 `0.0.0.0` 为允许所有 ip 访问。第三行的 `Port that the node will use` 为你在 [**调整配置信息**](#%e8%b0%83%e6%95%b4%e9%85%8d%e7%bd%ae%e4%bf%a1%e6%81%af) 这一步中的 `fly.toml` 文件中配置的端口，如果和我设置的一样，那这里应该是 `5984`😚。设置完成后会显示 `Apache CouchDB is configured for production usage as a clustered node! Do you want to replicate data?`，代表配置成功。

### 启用 CORS
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061304975.png"/>

然后打开 `Configuration` 选项卡中的 `CORS` 标签，启用 **CORS**。完成网页端操作！
> 下方的 **Origin Domains** 需要设置为 `All domains`。

## Obsidian 设置

这一章节的操作都在 Obsidian 本体软件中进行。首先需要关闭 Obsidian 中的安全模式，在**插件市场**中搜索 `Self-hosted LiveSync` 下载并启用。在 Github 仓库中手动下载安装本插件可能会出现一些问题。
> 在移动端可能会**打不开**插件市场，是众所周知的网络原因，需要自己解决😑。

### 配置连接信息

打开 `Remote Database configuration` 选项卡。输入自己的数据库网址、用户名、密码与数据库名。

1.  **数据库网址 URI** 为 `https://你的应用.fly.dev` 的形式，如果找不到可以使用 `flyctl open` 或在 [**fly.io 面板**](https://fly.io/dashboard)中打开自己的应用查看 **Hostname** 项。
2.  **用户名**与**密码**为在[**调整配置信息**](#%e8%b0%83%e6%95%b4%e9%85%8d%e7%bd%ae%e4%bf%a1%e6%81%af)时填写的用户名与在[**设置密码**](#%e8%ae%be%e7%bd%ae%e5%af%86%e7%a0%81)中用命令设置的密码。
3.  **数据库名**为在[**创建数据库**](#%e5%88%9b%e5%bb%ba%e6%95%b0%e6%8d%ae%e5%ba%93)时创建的数据库名。

<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061305937.png"/>
### 修复连接

点击 `Test Database Connection`，在右上角出现 **Connect to 数据库名**，则为连接成功。然后点击 `Check database configuration`，会出现一堆日志，逐个点击后面的 `fix` 按钮修复即可。

<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061305723.png"/>

修复完成后重新点击 `Check database configuration`，没有出现 `fix` 按钮即为修复成功。
### 同步设置

打开 **Sync Settings** 选项卡，其中有所有的同步方式设置。**注意**：实时同步 (LiveSync) 与定时同步 (Periodic Sync) 互斥，无法同时打开。

<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061305828.png"/>

可能有朋友会纠结用哪个，在这里强烈推荐使用 **LiveSync**（实时同步）方式，毕竟用此插件就是为了这个。因为 Periodic Sync（定时同步）中包含有 Sync on Save (保存时同步)、Sync on File Open (文件打开时同步)、Sync on Start (打开软件时同步) 各种选项，合计一下完全就是变相的实时同步嘛！第二点是当我使用 `hugo server` 开启本地博客变更监听后发现，Obsidian 会自动在输入文字后保存一次，造成**频繁**的 Sync on Save (保存时同步) 操作。所以，还是使用 LiveSync（实时同步）吧👍。

### 其他设置

在 **Sync Settings** 选项卡中还包含有 `Use Trash for deleted files`（删除文件到回收站）配置强烈建议打开，防止文件意外丢失。

在 **Miscellaneous** 选项卡（一个小扳手图标）中，有选项 `Show staus inside editor`（在编辑器右上角显示当前同步状态），推荐打开。

同步状态将显示在状态栏，状态都有：

-   ⏹️ 准备就绪
-   💤 LiveSync 已启用，正在等待更改
-   ⚡️ 同步中
-   ⚠️ 出现错误

信息解释：

-   ↑ 上传的 chunk 和元数据数量
-   ↓ 下载的 chunk 和元数据数量
-   ⏳ 等待的过程的数量
-   🧩 正在等待 chunk 的文件数量 如果你删除或修改了文件名称，请等待 ⏳ 图标消失。

### 安装于其他设备
在插件 **Setup wizard** 选项卡中，点击 `Copy Setup URI`，弹出的对话框输入你的数据库密码，即可复制当前的配置信息。在其他如 Android、iOS 设备上安装此插件并点击 `Open Setup URI` 输入复制的链接即可。
<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061306999.png"/>

在这里选第一个，意思为**将此设备设置为辅助或后续设备**。稍等一会儿后即可同步完成。而且同步非常快，只需要几秒钟就好。

<img src="https://raw.githubusercontent.com/almightyYantao/blog-img/master/202303061306269.png"/>

这里放一张官网的同步动态图可以感受一下。

## 推荐方案

值得注意的是，**不应该**在 Obsidian 中存放太多媒体文件，如：Markdown 文件直接引用本地图片或音频。不然会显著拖慢**任何** WebDAV 或 LiveSync 的同步速度。做一个对比，我的 Obsidian 中大概有 80 篇左右笔记与博文，本地源文件只有 600KB，数据库占用只有 2MB 左右。如果使用本地引用图片，那么一张图片就能抵得上我全部笔记的大小，同步时间将成倍的增长🙄。所以，这些媒体文件**更应该放入图床或对象存储中**，使用 `![]()` 的形式引用。

## 后记

fly.io 部署的服务体验太好了💖。Obsidian 各端同步起来非常快，虽然看面板只是一个 256MB 的小机器，但是这个任务完全可以胜任。fly.io 每个账户的免费资源包括：总共 3GB 的卷、最多 3 个共享 CPU-1x 256MB 虚拟机、每月 160GB 出站数据传输。看起来还能做一些新玩法的样子。在对比使用 Remotely save 同步后发现，同步速度除了快就是快，刚在电脑上写完一句话，想起有事情准备走，拿起手机打开 Obsidian，完全可以接着继续编辑，无缝同步的体验真是太棒了！我宣布这是 Obsidian 非官方同步服务的**最佳方式**。