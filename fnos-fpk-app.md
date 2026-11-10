# MediaGet 飞牛 FPK 应用说明

> 目标：把 MediaGet 以飞牛官方应用包（FPK）形式装进 fnOS，在桌面/应用中心出现可点击的图标，点开即是 Web UI。
> 访问地址：`http://<NAS_IP>:3000`　应用中心名：**MediaGet 视频解析**（appname `mediaget`）

---

## 一、安装后的状态

| 项目 | 结果 |
| --- | --- |
| 安装方式 | `appcenter-cli install-fpk mediaget.fpk -v 1`（等价于应用中心「手动安装」上传 fpk） |
| 应用中心状态 | `appcenter-cli list` → mediaget / MediaGet 视频解析 / 1.0.0 / **running** |
| 容器 | `mediaget`，`Up (healthy)`，`0.0.0.0:3000->3000/tcp` |
| 接口自检 | `GET /api/health` → `{"status":"ok"}`；`GET /api/config` → `videoParseEnabled:true` |
| 桌面入口 | 已登记：`service_name=mediaget.Application`、`type=iframe`、`url=http://${host}:3000/`、`icon=ui/images/icon_{0}.png` |
| 停止 / 启动 | `appcenter-cli stop mediaget` → 容器 Exited(143)；`start` → 重新 Up (healthy)，接口恢复 |
| 日志 | `/var/log/apps/mediaget.log`（应用中心侧）；`docker logs mediaget`（容器） |

装完桌面没看到图标就先 **F5** 刷新飞牛桌面；仍不出来执行
`sudo systemctl restart trim_app_center.service`（应用中心重启几秒，不影响容器）。

---

## 二、包内结构（安装后的落点）

```
/var/apps/mediaget/                       ← 应用根（安装目录）
├── manifest                              ← appname=mediaget / version=1.0.0 / service_port=3000 ...
├── ICON.PNG / ICON_256.PNG               ← 应用中心列表图标（64 / 256）
├── cmd/                                  ← 生命周期脚本
│   ├── main                              ← start/stop/status（status: 0=运行中, 3=未运行）
│   ├── install_init                      ← 装前检查端口是否被别的容器占用
│   ├── install_callback                   ← 装后确保镜像存在，缺失则用包内源码构建
│   └── uninstall_/upgrade_/config_*       ← 占位脚本（docker 应用由应用中心托管）
├── config/privilege                      ← run-as: root（安装后脚本要调 docker）
├── config/resource                       ← docker-project: mediaget → docker/；data-share
├── target -> /vol1/@appcenter/mediaget   ← app/ 的落点（TRIM_APPDEST）
│   ├── docker/docker-compose.yaml         ← 真正跑容器的编排
│   ├── ui/config                          ← 桌面入口定义
│   ├── ui/images/icon_64.png、icon_256.png ← 桌面图标
│   └── src/                               ← 随包源码（仅镜像缺失时兜底构建，约 4.6 MB）
└── var/ etc/ home/ tmp/ shares/            ← 系统创建的运行时目录
```

`app/ui/config`：

```json
{
  ".url": {
    "mediaget.Application": {
      "title": "MediaGet",
      "icon": "images/icon_{0}.png",
      "type": "iframe",
      "protocol": "http",
      "port": "3000",
      "url": "/",
      "allUsers": true
    }
  }
}
```

要点：`.url` 的 key 必须以 appname 开头，且与 manifest 的 `desktop_applaunchname` **完全一致**；
`icon` 里的 `{0}` 会被系统替换成 64/256，所以 `images/` 里两个尺寸都要有。

---

## 三、常用运维

```bash
appcenter-cli status mediaget
appcenter-cli list | grep media

appcenter-cli stop mediaget           # 等同界面上的停止按钮
appcenter-cli start mediaget

docker ps --filter name=mediaget
docker logs --tail 100 mediaget

cd /vol1/@appcenter/mediaget/docker && docker compose restart   # 绕过应用中心重启容器
sudo systemctl restart trim_app_center.service                  # 桌面图标不刷新时
```

**改端口**：端口写在三处且必须一致 —— `manifest` 的 `service_port`、`app/ui/config` 的 `port`、
`app/docker/docker-compose.yaml` 的 `ports`。改完重新 `fnpack build` 并覆盖安装。

---

## 四、重装 / 卸载 / 换包

```bash
# 覆盖安装同一个 appname
appcenter-cli install-fpk /vol1/mediaget.fpk -v 1

# 卸载（var/ 与 shares/ 会被保留）
appcenter-cli uninstall mediaget
```

也可以在飞牛网页「应用中心 → 手动安装 → 选择 fpk」上传安装。

**重新打包**：

```bash
cd <工程目录>
fnpack build -d .
```

---

## 五、三个踩过的坑

### 1. 应用用户没有 docker 权限

第一次用 `privilege.run-as: package`，装到 `install_callback` 就报
`APP_INSTALL_FAILED_INSTALL_CALLBACK_EXCEPTION`，CLI 再抛 `code 10238` + 一段 nil pointer panic。
实测 `sudo -u mediaget docker image inspect` 是拒绝访问。改成 `run-as: root`（并且**不写**
`username/groupname`，避免安装器尝试「创建 root 用户」导致"本地用户已存在"）后一次装过。

### 2. 镜像不进包

FPK 里只放 compose 配置，镜像必须本机已有。所以 compose 写 `pull_policy: never`
（避免应用中心去 Docker Hub 拉），并由 `install_callback` 在缺镜像时用包内 `src/` 现场
`docker build`（约 5–10 分钟）。另注意：目标机器上只有 `docker compose` v2，没有 `docker-compose` v1。

### 3. 桌面图标显示成灰色占位框

不是权限也不是缓存，而是**文件名和配置模板对不上**：
`app/ui/config` 写 `"icon": "images/icon_{0}.png"`，`{0}` 替换成 64/256 得到 `icon_64.png`（**下划线**），
而包里放的是 `icon-64.png`（**连字符**）→ 前端取图 404 → 占位图。

约定很简单：**占位符之间的分隔符就是文件名里的分隔符**。参考同机其它应用：

- 某个应用 config 写 `images/icon_{0}.png`，目录里是 `icon_64.png`、`icon_256.png`（下划线）
- 另一个应用 config 写 `images/icon-{0}.png`，目录里是 `icon-64.png`、`icon-256.png`（连字符）

省心做法是两套名字都给。改图标后：

```bash
cp icon_64.png icon_256.png /vol1/@appcenter/<appname>/ui/images/
chmod 644 /vol1/@appcenter/<appname>/ui/images/*.png
sudo systemctl restart trim_app_center.service    # 然后浏览器 Ctrl+F5
```

排查时注意：图标经 nginx 的 `/app-center-static` 路由下发，未登录直接 curl 会返回 **401**，
别误判成 404；而 `/app/<appname>/...` 走的是另一条 `trim_http_cgi` 路由，返回 200 也不是图标本体。

---

## 六、已知限制

- **音乐搜索可能不可用**：默认上游 `music-api.gdstudio.xyz` 返回 Cloudflare 521 / 直连超时，
  `/api/music?action=search` 会 502 `sources-down`。把 `MUSIC_API_BASE`（或 `MUSIC_API_BASES`）
  指向可直连实例即可；变量加在 compose 的 `environment` 里。
- **HTTPS 访问混流**：桌面入口写 `protocol: "http"` + 固定端口只适合 http 访问；改走 HTTPS 域名反代时
  要去掉 `port`、把 `protocol` 改成 `https`（或不声明），交给飞牛网关接管，否则浏览器会拦混合内容。
- 解析的可用性取决于上游平台风控，平台改规则时重建镜像再 `appcenter-cli stop/start` 即可。
