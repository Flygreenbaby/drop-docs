# Drop (lan-transfer)

局域网自托管的文本 / 文件中转站。所有设备打开同一个网址，就能互相发文字、链接、截图和小文件；新消息通过 SSE 实时出现在每台已打开的设备上。

不需要 App、不需要注册账号、不需要扫码配对——记住一个网址就够了。

> **Drop is a self-hosted LAN clipboard.** Open the same URL on every device in your network and drop text, links, screenshots and small files to each other, pushed live over SSE. Single container, no third-party runtime dependencies, no app to install, no accounts.
> **This is not open-source software.** The source repository is not published. Distribution here is a public Docker image plus project documentation, under a personal / non-commercial licence — see [LICENSE](LICENSE).

---

## ⚠️ 先读这一段：许可与源码可得性

- **本项目不是开源项目。** 不使用 MIT / Apache-2.0 / GPL 等 OSI 认可的许可证，也不授予商业使用权。许可条款见 [LICENSE](LICENSE)。
- **源代码仓库不公开**，也不接受外部贡献。这里公开的是 Docker 镜像与项目资料。
- 一个必须说清楚的技术事实：服务端是 Python 解释执行的，**镜像里必须带 `server.py` 才能运行**。也就是说 `docker pull` 之后，应用文件是可读的。本许可限制的是*使用权与再分发权*，不建立在"别人看不到代码"这个前提上。
- 镜像里的第三方组件（Debian 用户空间、Python 解释器）按它们各自的许可证走，不在本许可的管辖范围内，也不被本许可收紧。

## 它是什么 / 它不是什么

Drop 是一个**共享收件箱**：发出去的内容带着时间戳留在服务端的历史流里，而不是点对点直传完就消失。所以"手机拍了张图，两小时后才在电脑上想起要拿"这种场景成立。

它**不是**即时通讯，**不是**网盘，**不是**同步盘，也**不是**跨网传输工具。文件不会同步到某个指定文件夹，删除就是删除。

## 功能

**消息**

- 文本便签：Enter 发送、Shift+Enter 换行；输入法候选未上屏时的回车不会被当成发送
- 文本里的 `http(s)://` 自动变成可点击链接
- 文件收发：多文件顺序上传带进度条、页面拖拽投放、剪贴板粘贴（截图可直接发出）
- 图片消息在流里直接渲染内联缩略图
- 下载按 `/f/<id>` 取回，中文文件名按 RFC 5987 编码
- SSE 实时推送新消息与删除事件；断线由浏览器自动重连，重连后自动补齐断线期间的消息
- 历史分页，向上滚到顶部自动加载更早的记录；按日期插入分隔条
- 每条消息可删除（数据库标记 + 磁盘文件同步删除，并广播给其它设备）

**设备身份**

每条消息记录 `device_id`（浏览器首次打开时生成，存 `localStorage`）、`device_type`（7 种枚举，决定图标）、`device_name`（自定义名称，最长 24 字，允许中文）。身份存在客户端，服务端只校验格式并存档。用随机 id 而不是 IP 认设备，因为 DHCP 换一次地址就会把一台设备认成好几台。

**访问控制与容量**

- IP / CIDR 白名单，默认拒绝，`127.0.0.1` 不豁免
- 单文件上限（默认 25 MB）、文本字数上限（默认 8000 字）
- 收件箱总量兜底（默认 400 MB）与保留天数（默认 30 天）：超期清理，仍超限则从最旧一条开始删
- SSE 同时在线上限（默认 6，超出返回 503）

**界面**：移动优先的单页界面，适配安全区，无框架、无外部 JS、无构建步骤。界面目前**仅中文**。

## 支持的架构

镜像是多架构的，`docker pull` 会自动挑出匹配本机的那一份：

| 平台 | 状态 |
|---|---|
| `linux/arm/v7` | 主目标。产物结构经 33 项自检通过，并在 32 位 ARM 机顶盒上实机运行过 |
| `linux/amd64` | 由同一份 Dockerfile 构建，**未经实机运行验证** |
| `linux/arm64` | 由同一份 Dockerfile 构建，**未经实机运行验证** |

目标设备是 32 位 ARM（`armv7l`）时只能用 `linux/arm/v7`；换成 amd64 / arm64 的结果是 `exec format error`，不是"跑得慢一点"。

镜像构成：官方 `python:3.12-slim` 之上叠一层应用文件（`server.py`、`static/index.html`、`entrypoint.sh`、`config.default.json`），共 5 层、约 39 MB。构建过程不执行 `apt`、`pip`、`npm`，零第三方依赖；容器里只有一个进程 `python3 -u /app/server.py`。

## 快速开始

```sh
# 1. 拉镜像（把 YOUR_GITHUB_USERNAME 换成实际命名空间）
docker pull ghcr.io/YOUR_GITHUB_USERNAME/lan-transfer:latest

# 2. 准备数据目录，并把默认配置里的白名单改成你自己的网段
mkdir -p /opt/lan-transfer/data
#    首次启动时镜像内的默认配置会被放进 /opt/lan-transfer/data/config.json

# 3. 起容器
docker run -d --name drop \
  -v /opt/lan-transfer/data:/data \
  --network host \
  --restart unless-stopped \
  ghcr.io/YOUR_GITHUB_USERNAME/lan-transfer:latest
```

然后在局域网内任意设备的浏览器打开 `http://<宿主机地址>:8099`。

`--network host` 不是风格问题，是功能问题：默认的 bridge 网络 + `-p` 端口映射会对局域网请求做 SNAT，容器里看到的来源 IP 全部变成网桥网关地址，于是 IP 白名单当场失效（谁都在名单里）。详见 [USAGE.md](USAGE.md)。

## 技术取向

| 决策 | 选择 | 原因 |
|---|---|---|
| 实时通道 | SSE（`text/event-stream`） | 标准库没有 WebSocket 服务端；手写帧解析（掩码、分片、控制帧）易错难调。SSE 是一条普通 HTTP 长响应，浏览器原生自动重连 |
| 依赖 | 零第三方 | 无 `pip install`、无 `npm`。标准库自带 `sqlite3` 与 `ipaddress` 已够用 |
| 上传协议 | 原始字节直传（`application/octet-stream`） | 服务端不必解析 multipart（`cgi` 模块在 Python 3.13 已移除） |
| 存储 | SQLite + 磁盘文件 | 数据全在宿主机挂进去的那一个目录里 |
| 文件 I/O | 64 KB 分块流式读写 | 在内存受限设备上整份读入几十 MB 足以触发 OOM |

服务端实现是单个 `server.py`（Python 标准库 + SQLite + SSE），前端是单个 `index.html`。

## 安全边界（说清楚，不隐瞒）

请把它当作**局域网内部工具**来评估。

已实现：默认拒绝的 IP/CIDR 白名单（配置写错或为空则拒绝启动，不留"看起来生效实际没生效"的状态）；磁盘路径完全由服务端生成，原始文件名只入库展示，目录穿越在设计上不可能；前端全程 `textContent` / `createElement`，无一处 `innerHTML`，所以别人发 `<script>` 只会作为字符显示；请求尺寸上限校验；日志只记录本次传输自身的信息，按大小轮转封顶。

**未实现：没有 HTTPS，没有口令，没有账号体系。传输与存储都是明文，文件明文躺在磁盘上。** 白名单挡得住"连上 WiFi 就能翻你历史记录"，挡不住已经拿到白名单网段内任意 IP 的设备——局域网里地址不是身份。真要收紧需要的是 HTTPS + 口令，而不是往白名单里堆更多网段。

数据流向：服务端只监听、读写本地挂载卷，不向任何外部服务发起请求，无遥测、无第三方 SDK、无云端依赖；镜像取到之后可以完全离线运行。

## 当前限制（诚实清单）

- 不适合大文件与批量目录：默认单文件 25 MB、收件箱 400 MB、30 天保留、超量从最旧删除。这是产品边界，不是 bug。
- 单实例，无高可用、无横向扩展。
- 界面仅中文，无国际化。
- 宿主机 Docker 若不自启，重启设备后服务不会自动拉起（`--restart unless-stopped` 的前提是 dockerd 自己起来了）。
- `linux/amd64` / `linux/arm64` 变体未经实机运行验证（见上表）。
- 配置改动需重启容器生效，无热加载。

## 文档

- [USAGE.md](USAGE.md) —— 部署、配置项、常见错误、备份与升级
- [LICENSE](LICENSE) —— 个人 / 非商业使用条款

## 许可

© 2026 Drop / lan-transfer 作者。保留所有权利。

本项目的镜像与文档按 [LICENSE](LICENSE)（个人非商业使用许可）提供：**不是开源软件，不授予商业使用权**。需要商业用途请通过 [LICENSE](LICENSE) 中「商业授权」一节的方式联系作者。
