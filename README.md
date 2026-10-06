# Drop

局域网自托管的文本 / 文件中转站。所有设备打开同一个网址，就能互相发文字、链接、截图和小文件；新消息通过 SSE 实时出现在每台已打开的设备上。

Drop 是一个**共享收件箱**：发出去的内容带着时间戳留在服务端的历史流里，而不是点对点传完就消失。所以"手机拍了张图，两小时后才在电脑上想起要拿"这种场景成立。它**不是**即时通讯，**不是**网盘，也**不是**跨网传输工具。

不需要 App、不需要注册账号、不需要扫码配对——记住一个网址就够了。一个容器跑起全部功能，零第三方依赖。

![Drop 的使用界面](https://img.ahyun.org.cn/ahyun%E7%9A%84%E7%88%B1%E5%AD%A4%E5%B2%9B%E4%B9%8B%E5%8D%9A%E5%AE%A2%E8%AE%B0%E5%BD%95/images/20261006202154630.png)

许可：个人 / 非商业使用，条款见 [LICENSE](LICENSE)。本项目不是开源软件，源代码仓库不公开。

## Docker 镜像

镜像地址：

```
ghcr.io/flygreenbaby/drop
```

支持架构：

- `linux/amd64`
- `linux/arm64`
- `linux/arm/v7`

`docker pull` 会自动选出匹配本机的那一份。拉取最新版本：

```bash
docker pull ghcr.io/flygreenbaby/drop:latest
```

## Docker

```bash
docker pull ghcr.io/flygreenbaby/drop:latest

mkdir -p /opt/drop/data

docker run -d \
  --name drop \
  --restart unless-stopped \
  --network host \
  -v /opt/drop/data:/data \
  ghcr.io/flygreenbaby/drop:latest
```

服务端口是 `8099`，局域网内任意设备的浏览器打开 `http://<宿主机地址>:8099` 即可使用。数据全部落在挂进容器 `/data` 的那一个目录里（配置、数据库、收到的文件、日志）。

`--network host` 是功能要求，不是风格选择：默认的 bridge 网络加端口映射会对局域网请求做 SNAT，容器里看到的来源 IP 全部变成网桥网关地址，IP 白名单当场失效。

首次启动会把镜像内的默认配置写进 `/data/config.json`，把其中的 `allow` 改成你自己的局域网网段，再重启容器，否则浏览器会被白名单挡在 403。

## Docker Compose

```yaml
version: "3.3"

services:
  drop:
    image: ghcr.io/flygreenbaby/drop:latest
    network_mode: host

    volumes:
      - /opt/drop/data:/data

    restart: unless-stopped

    logging:
      driver: json-file
      options:
        max-size: "2m"
        max-file: "2"
```

```bash
docker-compose up -d
docker-compose down
docker-compose ps
docker-compose logs --tail 50
```

这里使用 `docker-compose`，而不是 `docker compose`。
