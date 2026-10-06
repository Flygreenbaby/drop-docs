# Drop 使用说明

面向"把它装到一台设备上、让家里几台机器能用"的人。不需要读源码，不需要装 Python。

文中示例路径都是**通用示例**（`/opt/lan-transfer/data`、`10.0.0.0/24`），照你自己的磁盘和网段替换即可。

---

## 0. 前提

- 一台常驻开机、局域网内地址稳定的设备（家用小服务器、ARM 单板机 / 机顶盒、NAS、旧笔记本都行），上面有 Docker。
- Docker 需要支持 `--network host`（Linux 宿主机天然支持；macOS / Windows 上的 Docker Desktop 不支持 host 网络，白名单会失效，见 §3）。
- 一个用于存放数据的目录，最好在某块独立挂载的盘上。
- 客户端不需要任何东西：能打开浏览器就行。

检查 Docker 是否可用：

```sh
docker version
docker info --format '{{.ServerVersion}}'
```

## 1. 取镜像

### 1.1 在线拉取（推荐）

```sh
docker pull ghcr.io/YOUR_GITHUB_USERNAME/lan-transfer:latest
```

镜像是多架构的，`docker pull` 会自动选择与本机匹配的架构。想看拉到的是哪一份：

```sh
docker image inspect ghcr.io/YOUR_GITHUB_USERNAME/lan-transfer:latest \
  --format '{{.Architecture}} / {{.Os}}'
# 32 位 ARM 设备上期望输出： arm / linux
```

`latest` 不表示"开发中的不稳定版"，它就是最近一次发布的版本。想要可复现的部署，用固定版本号标签（例如 `:2.0`）。

### 1.2 目标机不能上网：离线导入

在任何一台能拉到镜像、且有 Docker 的机器上：

```sh
docker pull --platform linux/arm/v7 ghcr.io/YOUR_GITHUB_USERNAME/lan-transfer:2.0
docker save -o lan-transfer-2.0-armv7.tar ghcr.io/YOUR_GITHUB_USERNAME/lan-transfer:2.0
```

把 tar 拷到目标机（scp / U 盘都行），然后：

```sh
docker load -i lan-transfer-2.0-armv7.tar
```

⚠️ `docker save` 一次只能导出**一个架构**。多架构镜像在本地 `docker pull` 时已经按本机架构选定了一份，所以导出前请用 `--platform` 明确指定，别指望一个 tar 里装着三种架构。

## 2. 数据目录

全部数据只住在宿主机挂进容器的那**一个**目录里：

```
<宿主机数据目录>/          容器内固定为 /data
├── config.json            唯一配置（首次启动时由镜像内默认配置写入）
├── database/drop.db       SQLite，消息记录
├── files/<YYYY-MM-DD>/    上传的文件本体
├── logs/drop.log          运行日志，按大小轮转
└── backups/               备份预留
```

```sh
mkdir -p /opt/lan-transfer/data
```

两个注意事项，都是踩过的坑：

- **不要只把 `config.json` 单个文件挂进去。** 宿主机上该文件不存在时，Docker 会创建一个**同名目录**，容器直接起不来，而报错只会指向一个看不出根因的 `IsADirectoryError`。挂目录。
- **确认数据目录落在你想要的那块盘上。** Docker 对不存在的宿主路径会自动 `mkdir` —— 外接盘没挂上时，这条命令不会报错，它会安安稳稳地把数据写进系统盘的目录里。启动前先 `df -h <数据目录所在路径>` 看一眼是不是独立挂载点。

## 3. 启动

```sh
docker run -d --name drop \
  -v /opt/lan-transfer/data:/data \
  --network host \
  --restart unless-stopped \
  --memory 320m \
  --log-opt max-size=2m --log-opt max-file=2 \
  ghcr.io/YOUR_GITHUB_USERNAME/lan-transfer:latest
```

每个参数为什么在这儿：

| 参数 | 作用 | 不加会怎样 |
|---|---|---|
| `-v <目录>:/data` | 数据持久化的唯一位置 | 容器的启动检查会发现 `/data` 不是挂载卷并**拒绝启动**（这是故意的）。镜像没有声明 `VOLUME`，因为一旦声明，漏带 `-v` 时 Docker 会静默建一个匿名卷，数据写进容器可写层，界面上一切正常，删容器即丢光 |
| `--network host` | 容器直接绑宿主机端口，能看到**真实来源 IP** | bridge + `-p` 会对局域网请求做 SNAT，容器看到的来源 IP 全是网桥网关，于是 IP 白名单当场失效（谁都在名单里）。这不是风格问题，是功能问题。**端口因此由 `config.json` 的 `port` 决定，没有 `-p` 这一说** |
| `--restart unless-stopped` | 开机 / 异常退出后自动拉起 | 前提是宿主机的 dockerd 自己会随系统启动 |
| `--memory` | 内存上限 | 内存受限设备上，真出 bug 时被杀的是这一个容器，而不是整台设备。常驻实测约 27 MB，给几百 MB 余量足够 |
| `--log-opt max-size/max-file` | 容器日志封顶 | 小闪存设备上"容器日志堆满闪存"是真实发生过的事故 |

`--network host` 在 macOS / Windows 的 Docker Desktop 上不可用。那些平台上跑起来能通，但白名单会因 SNAT 而失效，只适合临时试一下，不适合长期部署。

看启动日志：

```sh
docker logs drop
```

正常会打印配置路径、数据路径、监听地址与端口，以及逐个数据子目录的可写确认。

## 4. 第一次访问：先改白名单

镜像里出厂的默认配置带的是一个示例私网网段，**几乎不可能正好等于你的网络**。此时所有设备都会被 403 挡在门外 —— 那不是坏了，是访问控制正在按设计工作。

```sh
# 编辑宿主机上的 <数据目录>/config.json
vi /opt/lan-transfer/data/config.json
```

把 `allow` 改成你自己的网段，例如整个家用网段 `10.0.0.0/24`，或只放某一台 `10.0.0.23`。想在宿主机自己的浏览器里打开，把 `127.0.0.1` 也写进去 —— 本机没有暗规则豁免。

改完重启容器（配置无热加载）：

```sh
docker restart drop
```

然后局域网内任意设备的浏览器打开：

```
http://<宿主机局域网地址>:8099
```

## 5. 配置项

| 键 | 默认 | 含义 |
|---|---|---|
| `host` | `0.0.0.0` | 监听地址 |
| `port` | `8099` | 监听端口（host 网络下即宿主机端口） |
| `allow` | 示例网段 | **必填**。IP / CIDR 白名单，默认拒绝一切不在列表内的地址，含 `127.0.0.1` |
| `max_file_mb` | `25` | 单个文件大小上限 |
| `inbox_max_mb` | `400` | 收件箱文件总占用上限，超出从最旧开始删除 |
| `text_max_chars` | `8000` | 单条文本字数上限 |
| `retention_days` | `30` | 消息保留天数 |
| `max_sse_clients` | `6` | 同时在线的 SSE 连接上限，超出返回 503 |
| `history_page_size` | `60` | 历史分页大小（请求可用 `limit` 覆盖，上限 200） |
| `log_max_mb` / `log_keep` | `2` / `2` | 日志轮转单文件上限与保留份数 |

`allow` 里条目无法解析、或列表为空时，服务**拒绝启动**并打印具体原因。静默失效的访问控制比没有访问控制更危险，所以宁可起不来。

## 6. 日常使用

| 动作 | 操作 |
|---|---|
| 发文本 | 输入框内输入，Enter 发送，Shift+Enter 换行 |
| 发文件 | 点 📎 选择，或直接把文件拖到页面上，或截图后 Ctrl/Cmd+V 粘贴 |
| 取文件 | 点文件卡片下载；图片会同时显示缩略图 |
| 复制文本 | 消息下方「复制」 |
| 删除 | 消息下方「删除」（连带删除磁盘文件，**不可恢复**） |
| 改设备名 / 类型 | 点右上角设备按钮 |
| 翻更早的记录 | 向上滚动到顶部自动加载 |

每台设备用浏览器里的 `device_id` 认身份。换浏览器、换地址访问、清除站点数据都会被认成新设备。

## 7. docker-compose 写法

与上面的 `docker run` 等价：

```yaml
services:
  drop:
    image: ghcr.io/YOUR_GITHUB_USERNAME/lan-transfer:latest
    network_mode: host
    volumes:
      - /opt/lan-transfer/data:/data
    restart: unless-stopped
    mem_limit: 320m
    logging:
      driver: json-file
      options:
        max-size: "2m"
        max-file: "2"
```

注意 `image:` 标签要与 `pull` 下来的标签一致；写成 `latest` 时升级要靠 `docker compose pull` + 重建容器。

## 8. 备份、升级、回滚

**备份**：把数据目录整体打包即可（数据库是 SQLite，稳妥做法是先停容器或用 `sqlite3 .backup` 做一致性快照，再打包 —— 直接 tar 一个正在被写入的 `.db` 可能拷出半页数据）。备份默认与数据在同一块盘上，需要长期留存请自行复制到别的介质。

**升级**：

```sh
docker pull ghcr.io/YOUR_GITHUB_USERNAME/lan-transfer:2.1
docker stop drop && docker rm drop
docker run -d --name drop ... ghcr.io/YOUR_GITHUB_USERNAME/lan-transfer:2.1
```

数据、配置都在挂载卷里，**删容器不会删数据**，镜像里也没有任何用户数据。配置不会被新版本覆盖（首次启动只在卷里没有配置时才写入默认配置）。

**回滚**：用旧版本标签重新 `docker run` 就行。表结构演进只使用 `ALTER TABLE ADD COLUMN`，不做建新表搬数据，因此旧版本读新库不会因为缺表而崩。

**卸载**：`docker stop drop && docker rm drop`，然后自行决定是否删除数据目录。

## 9. 常见错误

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 浏览器 403 | 来源 IP 不在 `allow` 里 | 改 `config.json` 的 `allow`，然后 `docker restart drop` |
| 所有设备都"像在名单里" | 没用 `--network host`，SNAT 把来源 IP 变成了网关地址 | 按 §3 重跑；或自己 curl 一下 `docker logs` 里打印的来源地址确认 |
| `exec format error` | 镜像架构与设备不符（在 32 位 ARM 上拉了 amd64） | 确认 `docker image inspect` 输出 `arm / linux`；用 `--platform linux/arm/v7` 重新拉 |
| 容器起来又退出，日志说 `/data 不是宿主机挂载卷` | 漏带 `-v`，或挂的是单个文件 | 挂**目录**，见 §2、§3 |
| 日志报某个子目录不可写 | 属主 / 只读挂载 / 盘满 | 检查 `df -h`、挂载是否 `ro`、目录权限 |
| 上传返回 413 | 超过 `max_file_mb`，或收件箱触到 `inbox_max_mb` | 调大配置，或接受这是产品边界 |
| 页面打不开、日志空白 | 端口被占用，或宿主机防火墙拦了入站 | `ss -lntp \| grep 8099`；放行对应端口（仅入站该端口，不要关整个防火墙） |
| SSE 返回 503 | 同时在线连接数达到 `max_sse_clients` | 调大配置；上限存在是因为内存受限设备上每连接都占资源 |
| 下载返回 410 | 记录还在，但文件已被保留策略清理 | 正常行为，不是丢数据 bug |

## 10. 这份文档里还没验证过的部分

诚实标注，避免你替我踩：

- **在线 `docker pull` 这条路径尚未经过一次实际验证。** 镜像发布到 GHCR 并把包设为 Public 之后，才第一次有人（包括我）真的拉过。设成 Public 之前，这一步在文档里属于"按官方写法应该可行"。
- **`linux/amd64` 与 `linux/arm64` 变体未经实机运行验证。** 它们由同一份 Dockerfile 构建，但只有 `linux/arm/v7` 在真实设备上跑过。
- `--memory` / `--log-opt` 的具体数值是低资源设备上的保守建议，不同 Docker 版本对参数的支持程度未逐一实测。
- 老版本 Docker 对多架构 manifest 与构建证明（provenance attestation）的兼容性差异较大。本项目的发布构建刻意关闭了 provenance / sbom 附件以换取最大兼容性；若你的设备上 `docker pull` 报 manifest media type 相关错误，请把完整报错贴到 issue 里。
