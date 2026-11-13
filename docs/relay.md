# 经中转访问 DuMate 网关

## 什么时候需要中转

本插件默认直连百度官方网关 `https://dumate-svc.baidu.com/gateway/apis/v1`。
如果运行 Magpie 的机器直连该地址不通（例如出口网络被挡、连接超时），
而另一台机器可以连通，就需要经中转访问。

判断方法（在运行 Magpie 的机器上）：

```bash
curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" --max-time 12 \
  https://dumate-svc.baidu.com/gateway/apis/v1/chat/completions
```

返回 `000`（超时）说明直连不通；返回 `400` 说明网关可达（GET 无 body 被拒绝是正常的）。

## 架构

```text
Magpie（dumate 插件）
  → DUMATE_GATEWAY_BASE 指向本机反代地址
  → 反向代理（运行 Magpie 的机器上）
  → SSH 反向隧道
  → 中转机上的反代 / 直连
  → https://dumate-svc.baidu.com
```

要点：

- 中转只负责**聊天请求**。登录态换 token 仍直连 `console.bce.baidu.com`，
  该地址一般不受影响；如果它也不通，需要另外处理。
- 中转机必须能直连 `dumate-svc.baidu.com`（先在中转机上用上面的 curl 验证）。
- 插件通过环境变量 `DUMATE_GATEWAY_BASE` 指定中转地址，不设置时保持官方地址。

## 搭建步骤

以下 `<...>` 均为占位符，请替换成你自己的值；不要把真实地址、端口或凭证写进公开仓库。

### 1. 中转机上起反代

中转机需要一个能把请求转发到百度网关的反代。下面以 nginx 为例，
监听 `<中转机本地端口>`，转发到官方网关：

```nginx
server {
    listen 127.0.0.1:<中转机本地端口>;
    location / {
        proxy_pass https://dumate-svc.baidu.com;
        proxy_ssl_server_name on;
        proxy_ssl_name dumate-svc.baidu.com;
        proxy_set_header Host dumate-svc.baidu.com;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_connect_timeout 10s;
        proxy_read_timeout 120s;
        proxy_send_timeout 120s;
    }
}
```

也可以用其他反代实现（例如 Python、Caddy），只要能把
`/<路径>` 原样转发到 `https://dumate-svc.baidu.com/<路径>` 即可。

在中转机上验证：

```bash
curl -s -o /dev/null -w "%{http_code}\n" --max-time 15 \
  http://127.0.0.1:<中转机本地端口>/gateway/apis/v1/chat/completions
# 应返回 400（网关可达）
```

### 2. 从中转机向 Magpie 机器建反向隧道

在**中转机**上执行，把 Magpie 机器的 `<本地端口>` 映射到中转机的反代端口：

```bash
ssh -N -R 127.0.0.1:<本地端口>:127.0.0.1:<中转机本地端口> <Magpie 机器的 SSH 账号>@<Magpie 机器地址>
```

建议加上 `-o ServerAliveInterval=30 -o ExitOnForwardFailure=yes`，
并用 `setsid` / `nohup` / systemd 保持隧道常驻。
中转机重启、断网后隧道会断，需要重建。

在 Magpie 机器上验证隧道：

```bash
curl -s -o /dev/null -w "%{http_code}\n" --max-time 15 \
  http://127.0.0.1:<本地端口>/gateway/apis/v1/chat/completions
# 应返回 400
```

### 3. （可选）在 Magpie 机器上再加一层本机反代

如果希望插件连一个本机反代地址（例如为了统一入口或套一层 TLS），
可以在 Magpie 机器上再起一个 nginx，只做一层转发：

```nginx
server {
    listen 127.0.0.1:<本机反代端口>;
    location / {
        proxy_pass http://127.0.0.1:<本地端口>;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_connect_timeout 10s;
        proxy_read_timeout 120s;
        proxy_send_timeout 120s;
    }
}
```

不加这一层也可以，插件直接连 `http://127.0.0.1:<本地端口>` 即可。
本指南后续以“插件直连隧道端口”为例。

### 4. 给 Magpie 设置环境变量并重启

`DUMATE_GATEWAY_BASE` 必须对运行 Magpie 的进程可见。
Magpie 启动时插件会读取它，改了要重启 Magpie 才生效：

```bash
DUMATE_GATEWAY_BASE=http://127.0.0.1:<本地端口>/gateway/apis/v1 magpie tui
```

或写进 Magpie 的启动方式（systemd 服务的 `Environment=`、启动脚本等），
然后重启 Magpie。注意结尾不要带 `/`，插件会自动处理。

### 5. 验证一次真实调用

```bash
magpie provider test dumate deepseek-v41-flash
```

看到模型正常返回即成功。也可以用 Codex 指定该模型实测一次：

```bash
codex exec --skip-git-repo-check -c model="dumate/deepseek-v41-flash" "Reply with the single word OK."
```

## 排查

| 现象 | 可能原因 |
| --- | --- |
| `magpie provider test` 报 `no answer in 1m0s` | 中转链路不通：先按第 1、2 步的 curl 逐段验证（中转机直连、隧道端口）。 |
| 返回 401 `Incorrect API Key` | 链路已通，但网关 token 无效：重新登录 DuMate（`magpie plugin login dumate`），或检查登录态是否过期。 |
| 返回 400 `DuMate only accepts its official chat endpoint` | 请求地址与 `DUMATE_GATEWAY_BASE` 不一致：检查环境变量是否真正生效（重启 Magpie），以及地址结尾是否多了 `/`。 |
| 中转机能通、Magpie 机器不通 | 隧道断了：重建反向隧道；检查中转机反代是否还在运行。 |
| 一开始能用，过段时间不行 | 登录态（Cookie / token）过期：重新登录；或中转机、隧道因重启/断网而中断。 |

## 注意

- 中转机、隧道、Magpie 都是单点：任一环节中断调用就失败；重要用途请考虑固定中转机与自动重连。
- 不要把真实服务器地址、端口、Cookie、token 写进公开仓库或公开文档。
- 本插件不修改、不上传 DuMate 桌面端的登录文件；中转只转发聊天请求，不经过作者或第三方的服务器（除非你自己搭的中转机）。
