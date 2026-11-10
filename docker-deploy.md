# MediaGet 在飞牛 fnOS（x86）上的 Docker 部署

目标设备：`<NAS_IP>`（飞牛 fnOS，Debian 12 bookworm，x86_64，4 核 / 7.8 GB 内存）
项目来源：https://github.com/hotier/MediaGet

---

## 一、部署结果速查

| 项目 | 值 |
|---|---|
| 访问地址（局域网） | `http://<NAS_IP>:3000` |
| 健康检查 | `http://<NAS_IP>:3000/api/health` → `{"status":"ok"}` |
| 容器名 | `mediaget`（`restart: unless-stopped`，主机重启后自动拉起） |
| 镜像 | `mediaget:latest`（526 MB，基于 node:22-alpine + yt-dlp + ffmpeg） |
| 项目目录 | `/vol1/MediaGet`（源码在 `src/`，编排文件在同级） |
| 端口映射 | 主机 `3000` → 容器 `3000` |
| 运行用户 | 容器内非 root（`node`，UID 1000），`init: true` 已启用 |
| 当前状态 | `Up (healthy)`，内存占用约 75 MB，CPU 空闲 ~0% |

一条命令看状态：

```bash
sudo docker ps --filter name=mediaget
sudo docker logs mediaget --tail 50
```

---

## 二、目录结构

```
/vol1/MediaGet/
├── docker-compose.yml   # 编排文件
├── .env                 # 运行配置，全部为可选项
├── update.sh            # 升级脚本
└── src/                 # 上游源码（含 Dockerfile）
```

---

## 三、实际执行的部署步骤（复现用）

> 前提：这台机器的 NAS 直连 `github.com` 的 `git clone` 会超时（`Failed to connect to github.com port 443`），
> 但 `raw.githubusercontent.com` 正常。所以源码是在一台能上网的电脑上下载压缩包再传上去的。

```bash
# 1. 电脑上下载源码包
#    https://codeload.github.com/hotier/MediaGet/tar.gz/refs/heads/main
# 2. 传到 NAS
scp mediaget-main.tar.gz <NAS_USER>@<NAS_IP>:/tmp/
# 3. NAS 上解压
sudo mkdir -p /vol1/MediaGet && cd /vol1/MediaGet
sudo tar -xzf /tmp/mediaget-main.tar.gz -C /vol1/MediaGet
sudo mv MediaGet-main src && sudo chown -R 1000:1001 src
# 4. 构建镜像（约 6 分钟，主要时间花在 pip 装 yt-dlp）
cd /vol1/MediaGet
sudo docker build --build-arg BUILD_ID=$(date +%Y%m%d-%H%M) -t mediaget:latest src
# 5. 启动
sudo docker compose up -d
```

> 基础镜像 `node:22-alpine` 走的是 NAS 自带的 registry 镜像（`docker.fnnas.com`），12 秒拉完，不需要额外配置加速。
> 构建时 npm 用的是 Dockerfile 内置的 npmmirror 源，Alpine apk 与 PyPI 均可直连（PyPI 偏慢，但能装完）。

`docker-compose.yml` 关键部分：

```yaml
services:
  mediaget:
    image: mediaget:latest
    container_name: mediaget
    init: true
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - TZ=Asia/Shanghai
      - VIDEO_PARSE_ENABLED=true
```

---

## 四、日常运维命令

```bash
cd /vol1/MediaGet

sudo docker compose ps                 # 状态
sudo docker compose logs -f --tail 100 # 实时日志
sudo docker compose restart            # 重启
sudo docker compose stop               # 停止
sudo docker compose start              # 启动
sudo docker compose down               # 删除容器（镜像和源码保留）
sudo docker compose up -d              # 按当前配置（重）启动
```

改 `.env` 后必须 `sudo docker compose up -d` 重建容器才生效（`restart` 不够）。

---

## 五、可选配置（`.env`）

默认什么都不填也能完整跑视频解析。以下按需打开，改完执行 `sudo docker compose up -d`：

| 变量 | 作用 |
|---|---|
| `VIDEO_PARSE_ENABLED` | 视频解析入口总开关，**必须为 `true`**（未配置时默认关闭，首页会隐藏解析入口） |
| `DOUYIN_COOKIE` / `BILIBILI_COOKIE` / `XHS_COOKIE` / `KUAISHOU_COOKIE` / `IG_COOKIE` | 平台 Cookie，提升成功率、补全作者简介等信息 |
| `MUSIC_API_BASE` / `MUSIC_API_BASES` | 音乐上游基址（见第七节，默认公共上游可能不可用） |
| `PROXY_ALLOWED_HOSTS` | 媒体代理追加白名单（自建图床/CDN 域名，逗号分隔） |
| `TURSO_DB_URL` / `TURSO_AUTH_TOKEN` | 缓存/统计/设置页持久化（不配则缓存只在内存，重启即失效） |
| `SETTINGS_API_KEY` | `/music/settings` 音乐设置页登录密钥 |
| `STATS_API_KEY` | `/api/stats` 统计接口的 Bearer 鉴权 |

配置文件写好后注意权限：`sudo chmod 600 /vol1/MediaGet/.env`（含 Cookie 时尤其要收紧）。

---

## 六、升级到新版本

```bash
# 1. 电脑上下载最新的源码包（同上，保存为 mediaget-main.tar.gz）
# 2. 传到 NAS
scp mediaget-main.tar.gz <NAS_USER>@<NAS_IP>:/tmp/
# 3. 执行升级脚本（自动备份旧源码、重建镜像、重建容器、等健康检查）
sudo bash /vol1/MediaGet/update.sh /tmp/mediaget-main.tar.gz
```

升级脚本失败时会保留 `src.old` 便于回滚，日志看 `sudo docker logs mediaget --tail 100`。

---

## 七、实测结果与已知限制

**实测通过**

| 检查项 | 结果 |
|---|---|
| `/api/health` | 200，`status: ok` |
| 首页 `/` | 200，24 KB |
| `/api/config` | `videoParseEnabled: true` |
| 抖音解析 | 200，返回标题/作者/播放地址等完整字段 |
| B 站解析 | 200，返回直链、封面、UP 主信息 |
| 图片代理 `/api/image` | 200，651 KB 正常回源 |
| 视频代理 `/api/video-proxy` | 206 分段请求正常，实测拉回 1 MB |
| 容器健康 | `Up (healthy)`，内存 ~75 MB |

**已知限制（与部署无关，属项目/网络环境本身）**

1. **音乐搜索可能不可用**：默认公共上游 `music-api.gdstudio.xyz` 返回 Cloudflare `521`，直连也超时。恢复方式是把 `MUSIC_API_BASE` / `MUSIC_API_BASES` 指向一个可直连的同契约实例（gdstudio 的 `types=search/url/pic/lyric` 参数）。
2. **NAS 直连 GitHub 可能不通**：`git clone` 会超时，`raw.githubusercontent.com` 通常正常。所以升级按第六节用源码包，不要在 NAS 上直接 clone。
3. **TikTok** 依赖镜像内的 yt-dlp，可用；**YouTube** 官方直链通道在上游已下线，不可用时降级为官方嵌入播放。
4. **小红书 / Instagram** 是登录墙平台，匿名成功率低，配 Cookie 才有明显改善。
5. 付费/DRM 平台（腾讯视频、爱奇艺、优酷、芒果 TV、Netflix 等）本项目明确不支持。

---

## 八、常见问题排查

| 现象 | 处理 |
|---|---|
| 首页能开但没解析框 | `.env` 里 `VIDEO_PARSE_ENABLED` 不是 `true`，改完 `sudo docker compose up -d` |
| 容器 `unhealthy` | `sudo docker logs mediaget --tail 100`；多为端口被占或构建产物异常 |
| 3000 端口被占用 | 改 `docker-compose.yml` 左侧端口，如 `"3999:3000"`，然后 `up -d`（右边 3000 不要动） |
| 解析报 429 | 平台级节流/IP 级限流，稍等再试；配 Cookie 可缓解 |
| NAS 重启后没起来 | 确认 `restart: unless-stopped` 仍在，`sudo docker compose up -d` |

---

## 九、可选：外网访问 / HTTPS

局域网 `http://<NAS_IP>:3000` 已可用。要外网访问，建议做反向代理而不是直接暴露 3000：

1. 用 1Panel / Nginx Proxy Manager / fnOS 自带的端口转发能力，把域名反代到 `127.0.0.1:3000` 并配 HTTPS 证书；
2. 注意 `/api/video-proxy` 会中转较大文件，反代需放开 `client_max_body_size` 与较长超时（单文件可达数百 MB）；
3. 暴露公网前先想清楚：解析服务容易被滥用，建议加访问控制（Basic Auth 或仅内网/VPN 访问）。
