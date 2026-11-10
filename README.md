# MediaGet on fnOS (FPK)

把开源项目 [hotier/MediaGet](https://github.com/hotier/MediaGet)（短视频 / 音乐解析服务）以飞牛 fnOS 官方应用包 **FPK** 的形式部署，安装后在飞牛桌面/应用中心出现可点击的应用图标，点开即是 Web UI。

> 本仓库只记录**部署与打包过程**，不含上游源码。MediaGet 本体请从上游仓库获取。
> 所有机器相关的信息（内网地址、登录账号、带 UID 的内网路径）都已替换为占位符。

## 文档

| 文件 | 内容 |
| --- | --- |
| [docker-deploy.md](docker-deploy.md) | 第一步：用 `docker compose` 裸部署，跑通并验证接口 |
| [fnos-fpk-app.md](fnos-fpk-app.md) | 第二步：按 FPK 规范打包、安装、桌面图标，以及踩过的坑 |

## 环境

- 飞牛 fnOS（x86），Debian 12 bookworm，Docker Engine 28.x + compose v2
- 飞牛自带 `/usr/trim/bin/fnpack`（打包）与 `appcenter-cli`（安装/启停）
- 上游镜像基于 `node:22-alpine`，内置 yt-dlp 与 ffmpeg

## 大致流程

```bash
# 1. 构建镜像（拿到源码包后）
docker build -t mediaget:latest <源码目录>

# 2. 裸部署，先验证功能
docker compose up -d

# 3. 改造成 FPK 工程后打包
fnpack build -d <工程目录>          # 产出 mediaget.fpk

# 4. 安装（等价于应用中心「手动安装」上传 fpk）
appcenter-cli install-fpk mediaget.fpk -v 1
appcenter-cli status mediaget
```

## 三个结论（详细见文档）

1. **FPK 不带镜像**，包里只有 compose 配置；镜像必须本机已存在，或用 `install_callback` 现场构建。
2. **Docker 型 FPK 的 `config/privilege` 要用 `run-as: root`**（且不要写 username/groupname），否则安装回调里调 docker 会因权限失败，CLI 抛 `code 10238` 崩溃。
3. **桌面图标的文件名必须和 `ui/config` 的 `icon` 模板一致**：`icon_{0}.png` → `icon_64.png`（下划线），`icon-{0}.png` → `icon-64.png`（连字符）。不一致就是灰色占位图。

## 已知限制

- 默认公共音乐上游 `music-api.gdstudio.xyz` 可能返回 Cloudflare 521，音乐搜索会 502 `sources-down`，需要用 `MUSIC_API_BASE(S)` 指向可直连实例。
- 桌面入口若写 `protocol: http` + 固定端口，只适合 http 访问；改走 HTTPS 域名反代时要去掉 `port` 或改 `protocol`。
- 解析能否成功取决于各平台风控与 Cookie 配置，项目本身不支持付费/DRM 平台。
