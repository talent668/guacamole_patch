[English](README.md) | 简体中文

# guacamole_patch

这是针对 **Apache Guacamole 1.6.0** 的非官方下游补丁，重点改善浏览器切换标签页、页面长时间进入后台或恢复后出现的键盘与输入法失效问题。

本仓库不是 Apache Software Foundation 官方发布版本。

## 快速部署

拉取已经构建好的镜像：

```bash
docker pull ghcr.io/trilogys/guacamole_patch:1.6.0-recovery6
```

在 Docker Compose 中使用：

```yaml
services:
  guacamole:
    image: ghcr.io/trilogys/guacamole_patch:1.6.0-recovery6
```

只更新 Guacamole Web 容器：

```bash
docker compose pull guacamole
docker compose up -d --force-recreate --no-deps guacamole
docker compose ps
docker compose logs --tail=100 guacamole
```

Docker 会复用没有变化的镜像层，后续拉取通常只下载发生变化的部分。不要执行 `docker compose down -v`，也不要删除数据库卷。

## 镜像标签

- `1.6.0-recovery6`：本补丁包对应的当前具名恢复版本。
- `1.6.0-recovery5`：保留用于回滚的上一恢复版本。
- `1.6.0`：当前滚动发布镜像，每次正式构建都会更新这个标签。
- `main`：由 `main` 分支最新代码构建。
- `sha-<commit>`：对应特定源码提交的固定标签，适合精确部署和回滚。
- `dev`：由 `dev` 分支构建的开发镜像。

正式发布请选择 `main` 运行工作流。选择 `dev` 时也会更新共用的 `1.6.0` 标签。

## 修复内容

### 本机输入法 + Guacamole 文本输入

```text
远程 Windows：ENG
Guacamole 输入方式：文本输入
本机：中文输入法
```

补丁会处理浏览器遗漏的输入法合成结束事件，恢复当前远程连接的焦点以及文字、Backspace、Delete 的发送，并防止隐藏的原始键盘输入框抢占可见文本框焦点。

### 远程 Windows 微软拼音

```text
Guacamole 输入方式：无（None）
本机输入语言：ENG 或本机输入法
远程 Windows：微软拼音
RDP 键盘布局：en-us-qwerty
```

补丁会在页面隐藏时重置按键状态，返回页面后恢复键盘捕获，释放可能卡住的修饰键，处理长时间后台冻结和页面生命周期事件，并在首次操作时重建 Chromium 原生输入上下文。

仍可通过 `Ctrl+Alt+K` 或菜单中的“重新捕获键盘”进行手动恢复。

### 短暂网络提示

Guacamole 原本在大约 1.5 秒未收到隧道数据后就把连接标记为不稳定。补丁会额外确认 3 秒，只有页面可见且异常持续存在时才显示提示；页面隐藏期间不显示，返回页面后重新开始确认。

底层不稳定检测和 15 秒接收超时保持不变。真实且持续的网络或服务器故障仍会正常提示并断开连接。

“用户设置 → 连接恢复”以及当前连接侧边菜单提供“延迟自动刷新整个页面”开关，默认关闭。关闭时不会自动操作，右下角提示保留“刷新页面”按钮；开启后，确认异常时首次等待约 5 秒、第二次等待约 10 秒再刷新整个当前浏览器页面。每个标签页连续自动刷新最多两次，稳定运行一分钟后重置次数。

隧道仍显示正常，但连续三次有意点击后 8 秒内没有及时的远端画面同步时，同样会判断控制链路失效并请求恢复。补丁会使用 Guacamole 自带的画面统计和相对同步时间识别超过 3 秒的网络排队或浏览器渲染积压。存在进行中的文件传输、关闭自动刷新或点击“暂不刷新”时，会取消待执行操作。浏览器无法安全保存跨刷新计数时，自动刷新会停用，但手动按钮仍可用。

整页刷新会重新加载登录态、路由和所有平铺连接；负载均衡连接组可能重新选择后端，RDP 也可能打开不同的远程会话。若必须保持同一台主机，请保持开关关闭并优先使用具体连接。

recovery6 在刷新前会使用 `Cache-Control: no-cache` 重新验证当前文档，清除 Guacamole 的旧构建刷新锁，并确保请求成功、失败或超时后只重载一次。连接侧边菜单始终提供“刷新页面”；等待远程首帧时的状态窗口也提供相同操作。

### 登录后远程画布黑屏

若 Guacamole 登录和菜单正常、只有远程画布黑屏，优先在侧边菜单执行“刷新页面”。若无效，请重启 `guacd`，并将连接中的颜色深度、DPI 留空，“禁用 GFX”“强制无损”及三个“禁用缓存”选项全部取消勾选。这些连接参数保存在数据库中，回退 Web 镜像不会自动恢复。

### 弱网下的鼠标响应

高频鼠标移动会以大约 30 Hz 合并为最新坐标。按下、松开、右键、滚轮和拖拽结束事件会先刷新最后坐标并立即发送，避免点击排在过期移动事件之后。连接被替换时会丢弃旧连接尚未发送的移动。

### 远端浏览器高画面负载

远程 Windows 中打开 Chrome、Edge、视频、动画或滚动复杂网页会产生大量 RDP 画面更新。补丁可以识别严重积压并恢复连接，但不能消除远端主机、网络带宽或 `guacd` 编码能力不足。建议在 Guacamole 的具体 RDP 连接中使用：

recovery6 保留 recovery4 的画面热路径优化：不再从实时同步回调中每 5 秒展开完整画布并生成缩略图；逐帧显示统计默认关闭，仅在用户点击后临时启用 1 秒窗口。缩略图仍会在首次连接和断开时保存。

- 保留 GFX 时，颜色深度先留空或使用 32 位；只有配合 `disable-gfx=true` 时才比较 16 位色，具体见下方部署优化说明；
- 不启用“强制无损”；
- 不启用壁纸、主题、字体平滑、拖动时显示窗口内容、桌面合成和菜单动画；
- 保持位图、离屏和字形缓存启用，即不要勾选对应的“禁用缓存”；
- 首次排查时把远程分辨率降到 `1280x720` 或 `1600x900`，DPI 设为 `96`。

## FRP 与高延迟网络的部署优化

以下是部署层的排查与优化经验，需要按实际拓扑手动应用；安装本补丁不会自动修改 Nginx、FRP、防火墙或 RDP 连接参数。

### 缩短服务器内部的转发路径

如果 HTTPS 入口、Guacamole Web 和 FRP 服务端位于同一台服务器，避免让它们通过服务器公网地址来回转发。一次实际部署将：

```text
浏览器 → 公网 HTTPS 入口 → 服务器公网地址 → 第二层 Nginx → Guacamole
```

调整为：

```text
浏览器 → 公网 HTTPS 入口 → 本机 Guacamole
guacd → 同机 frps 的内网地址:13389 → frpc → Windows:3389
```

下面是放入**现有 HTTPS server 块**的示例 location。它假设入口代理运行在宿主机或使用 host 网络，Guacamole 已通过宿主机端口 `32768` 提供 `/guacamole/`：

```nginx
location / {
    proxy_pass http://127.0.0.1:32768/guacamole/;
    proxy_http_version 1.1;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;

    proxy_buffering off;
    proxy_request_buffering off;
    proxy_cookie_path /guacamole/ /;
}
```

- 替换原有 location，不要重复添加。保留现有公网 TLS、认证、访问控制和有效上传限制；只有确认内层代理没有额外安全功能后，才移除该中转环节。
- 若入口代理在独立容器网络中，`127.0.0.1` 指向代理自身，应使用可达的 Docker 服务地址，例如 `http://guacamole:8080/guacamole/`。示例端口不是固定要求。
- 上游明文 HTTP 仅适用于本机或受信任的容器内网；跨主机转发应保留适当的传输保护。
- 示例把公网根路径映射到上游 `/guacamole/`，因此保留相应 Cookie 路径改写。公网路径不同的部署需要同步调整。
- 关闭响应缓冲对 HTTP 隧道尤其重要；它不会直接降低已建立 WebSocket 的网络 RTT。请求缓冲关闭后，请求体可流式传递。

修改前备份文件；先运行 `nginx -t`，通过后平滑重载。检查首页、静态资源、登录和 WebSocket 升级，再重新连接远程桌面。仅看到 HTTP `101` 不能证明 RDP 登录或操作体验正常。回滚时恢复原文件，重新检查并平滑重载。

### RDP 画面配置：可比较的方案，不是通用最优值

Guacamole 1.6.0 的 GFX 路径要求 32 位色。保留 GFX 时，颜色深度可先留空或设为 32 位；手动填写 16 位不会覆盖 GFX 的要求。

若需要比较传统画面路径，可以在具体 RDP 连接中尝试以下参数。它们是 **Guacamole 连接参数**，不是 `frpc.toml` 或 `guacamole.properties` 配置：

```properties
disable-gfx=true
color-depth=16
width=1280
height=720
resize-method=display-update
force-lossless=false
dpi=96
enable-wallpaper=false
enable-theming=false
enable-font-smoothing=false
enable-desktop-composition=false
enable-full-window-drag=false
enable-menu-animations=false
```

这个组合降低色彩精度，并不保证比 GFX 更快；应在相同内容、分辨率和网络条件下对比。继续保留位图、离屏和字形缓存。更改连接参数后重新连接生效；`display-update` 允许分辨率随浏览器窗口变化，所以 `1280×720` 是初始请求尺寸，不是固定上限。

`force-lossless=false` 不等于禁用所有无损压缩，而是允许 Guacamole 按画面选择压缩方式。优先排查延迟时，不建议同时强制无损。

### FRP 路由与入口限制

- guacd 与 frps 同机时，可以将 RDP 目标改为 **frps 可达的内网地址与映射端口**。不要把 Windows 的本机 `127.0.0.1:3389` 填到另一台服务器上的 guacd 中。
- 只通过 Guacamole 使用 RDP 时，将映射端口（如 `13389`）限制为受信任的 guacd/服务器来源。先核对 NAT 后实际来源，并把专用规则放在已有整段端口放行规则之前，同时覆盖 IPv4 和 IPv6。
- Docker 地址、网络模式或公网 NAT 地址改变后，应复核允许规则；不要用一条广泛拒绝规则破坏同一 frpc 的其他业务。
- 如果**运行 frpc 的 Windows**被自己的 Mihomo TUN 接管，可以在现有 `tun.route-exclude-address` 列表中追加 FRP 服务器 IP 的 `/32` 地址。例如 `198.51.100.10/32` 只是文档示例，必须替换为实际服务器地址，并保留已有排除项。
- Mihomo 的 `DIRECT` 规则仍可能经过 TUN。保存后核对持久配置、运行配置和实际路由，例如 Windows 的 `Find-NetRoute -RemoteIPAddress <FRPS_IP>`。
- 被控 Windows 的 TUN 排除不会改变另一台客户端电脑的公司出口、代理或 VPN；该案例保持公司网络现状。
- 不要为消除 `work connection pool is full` 日志而盲目增大连接池。代理注册成功也不代表本机后端已启动；公网 RDP 端口按策略被拦截时，外部探测失败可能正是预期结果。

### 验证结果与边界

2026-10-27 的一次实际部署中，取消服务器公网回环和额外代理、关闭请求与响应缓冲后：

| 检查 | 结果 |
| --- | --- |
| Nginx 配置校验与平滑重载 | 通过 |
| HTTPS 首页与修改前内容比较 | HTTP 200，内容一致 |
| WebSocket 升级测试 | HTTP 101；这只是传输握手检查 |
| 同机 HTTPS 首页请求，前后各 5 次 | 总耗时中位数约 41.6 ms → 30.2 ms |
| 后续使用反馈 | 用户报告操作比之前顺畅很多 |

这些是特定部署中的观测，不是通用性能承诺。首页请求耗时不是“按键到画面显示”延迟，也不能据此量化每一项配置的贡献；红色提示残留是否彻底解决仍需独立验证。

此前采样的网页侧 TCP RTT 约 179–180 ms，FRP 侧约 46–51 ms，服务器内部约 0.1–0.2 ms。若中间代理终止 TCP，服务器看到的 RTT 还不等于完整的客户端路径；两段 RTT 相加也只是粗略估算。Windows 的一次 12 秒采样未见资源不足丢帧、编码耗时最高约 1 ms，这只代表当时的负载。

排查时区分“整台远程桌面都慢”和“只有远端浏览器慢”，并在实际操作期间同步观察客户端、代理、guacd、FRP 和 Windows。先保留有效配置，一次只比较一个变量，不要仅凭红条、单次 ping 或低 CPU 占用认定根因。

参考：[Guacamole 反向代理](https://guacamole.apache.org/doc/1.6.0/gug/reverse-proxy.html)、[RDP 连接参数](https://guacamole.apache.org/doc/1.6.0/gug/configuring-guacamole.html)、[FRP 网络与连接池](https://gofrp.org/en/docs/features/common/network/network/)、[Mihomo TUN 排除路由](https://wiki.metacubex.one/config/inbound/tun/#route-exclude-address)。

## GitHub Actions 构建

进入 **Actions → Build and publish Guacamole recovery image → Run workflow**，选择 `main` 后运行。

构建成功会发布：

```text
ghcr.io/trilogys/guacamole_patch:1.6.0
ghcr.io/trilogys/guacamole_patch:1.6.0-recovery6
ghcr.io/trilogys/guacamole_patch:main
ghcr.io/trilogys/guacamole_patch:sha-<commit>
```

## 手动构建

需要：

```text
Docker
curl
patch
Python 3
tar
sha256sum
mktemp
```

克隆并构建：

```bash
git clone https://github.com/trilogys/guacamole_patch.git
cd guacamole_patch

IMAGE_NAME="ghcr.io/trilogys/guacamole_patch:1.6.0-recovery6" \
bash ./build.sh
```

排错时可以使用较快的构建参数：

```bash
MAVEN_ARGUMENTS="-T 1C -Dmaven.test.skip=true" \
IMAGE_NAME="ghcr.io/trilogys/guacamole_patch:1.6.0-recovery6" \
bash ./build.sh
```

`build.sh` 会验证补丁包和 Apache 源码压缩包、运行回归检查、试应用并正式应用补丁、使用官方 Dockerfile 构建镜像、检查镜像并执行 `initdb` 冒烟测试。

## 验证镜像

```bash
docker image inspect ghcr.io/trilogys/guacamole_patch:1.6.0-recovery6 \
  --format '{{index .Config.Labels "io.guacamole.recovery.patch-sha256"}}'
```

预期补丁 SHA-256：

```text
0345d83616eefb0c23c6c8aad328739260beeb2bf0f0d645fc436d4f87f4de5f
```

## 验收测试

完整步骤见 [TEST_MATRIX.md](TEST_MATRIX.md)。至少验证：

- 本机中文输入法、远程 Windows 为 ENG 时的输入；
- 远程微软拼音的候选和选词；
- Chrome 和 Edge 中反复切换标签页；
- Ctrl、Shift、Alt 不会卡键；
- Backspace、Delete、方向键、Enter 和数字选词；
- 同一账号断开后重新连接；
- 能够回滚到已知正常镜像。

## 影响范围与限制

补丁不修改 `guacd`、PostgreSQL 数据结构和数据、Nginx、FRP、连接账号、录像或共享目录。

`Win+Space`、`Alt+Tab` 等快捷键可能被本机操作系统或浏览器拦截。正式部署前仍需在实际 Windows RDP 环境中完成验收。

## 许可证

本补丁包遵循 Apache License 2.0。Apache Guacamole 的版权和 NOTICE 要求保持不变。
