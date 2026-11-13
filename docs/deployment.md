# DuMate 中转详细部署过程

本文记录一次完整的中转部署：把 DuMate 网关的访问从一台能直连百度的常开机
（下称“中转机”）中转到运行 Magpie 的机器（下称“Magpie 机器”）。
所有 `<...>` 均为占位符，请替换成你自己的值；不要把真实地址、端口或凭证
写进公开仓库。

> 背景：Magpie 机器直连 `dumate-svc.baidu.com` 不通（连接超时），而中转机
> 可以直连（`curl` 返回 400 即表示网关可达）。因此需要中转。

## 架构

```text
Magpie（dumate 插件，DUMATE_GATEWAY_BASE 指向本机反代）
  → Magpie 机器上的 nginx 反代（监听 <本机反代端口>）
  → SSH 反向隧道（Magpie 机器 <隧道端口> ← 中转机）
  → 中转机上的 Python 反代（监听 <中转机本地端口>）
  → https://dumate-svc.baidu.com
```

## 前置检查

在中转机上验证能直连百度网关：

```bash
curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" --max-time 12 \
  https://dumate-svc.baidu.com/gateway/apis/v1/chat/completions
# 返回 400 表示可达；返回 000（超时）表示不通，不能当中转机
```

在 Magpie 机器上确认直连不通（否则不需要中转）：

```bash
curl -s -o /dev/null -w "%{http_code}\n" --max-time 12 \
  https://dumate-svc.baidu.com/gateway/apis/v1/chat/completions
# 返回 000（超时）表示需要中转
```

## 1. 中转机：准备 SSH 密钥

中转机需要一把专用的 SSH 密钥，用来向 Magpie 机器建立反向隧道。
如果中转机上还没有，就生成一把（无口令，供隧道专用）：

```bash
ssh-keygen -t ed25519 -N "" -C "<中转机>-dumate-relay" -f ~/.ssh/id_ed25519
cat ~/.ssh/id_ed25519.pub
```

把输出的公钥追加到 **Magpie 机器**的 `~/.ssh/authorized_keys` 末尾：

```bash
# 在 Magpie 机器上执行
echo "<中转机公钥>" >> ~/.ssh/authorized_keys
```

## 2. 中转机：部署反代

反代只需把请求原样转发到百度网关。下面是一个最小的 Python 实现
（也可以用 nginx/Caddy 等，只要能转发即可）。把它保存为
`~/dumate-relay/proxy.py`：

```python
#!/usr/bin/env python3
"""Minimal reverse proxy: forward everything to dumate-svc.baidu.com."""
import http.server
import http.client
import threading

UPSTREAM_HOST = "dumate-svc.baidu.com"
LISTEN_HOST = "127.0.0.1"
LISTEN_PORT = <中转机本地端口>

class Handler(http.server.BaseHTTPRequestHandler):
    def _handle(self):
        length = int(self.headers.get("Content-Length") or 0)
        body = self.rfile.read(length) if length else None
        conn = http.client.HTTPSConnection(UPSTREAM_HOST, timeout=120)
        headers = {k: v for k, v in self.headers.items()
                   if k.lower() not in ("host", "connection")}
        headers["Host"] = UPSTREAM_HOST
        conn.request(self.command, self.path, body=body, headers=headers)
        resp = conn.getresponse()
        data = resp.read()
        self.send_response(resp.status)
        for k, v in resp.getheaders():
            if k.lower() not in ("transfer-encoding", "connection"):
                self.send_header(k, v)
        self.send_header("Content-Length", str(len(data)))
        self.end_headers()
        self.wfile.write(data)
        conn.close()

    do_GET = _handle
    do_POST = _handle
    do_PUT = _handle
    do_DELETE = _handle
    do_PATCH = _handle

    def log_message(self, fmt, *args):
        print("[proxy]", self.command, self.path, flush=True)

if __name__ == "__main__":
    srv = http.server.ThreadingHTTPServer(
        (LISTEN_HOST, LISTEN_PORT), Handler)
    print(f"[proxy] listening on {LISTEN_HOST}:{LISTEN_PORT}", flush=True)
    srv.serve_forever()
```

在中转机上验证反代：

```bash
cd ~/dumate-relay
nohup python3 proxy.py > proxy.log 2>&1 &
curl -s -o /dev/null -w "%{http_code}\n" --max-time 12 \
  http://127.0.0.1:<中转机本地端口>/gateway/apis/v1/chat/completions
# 应返回 400（网关可达）
```

## 3. 中转机：建立反向隧道

从**中转机**向 Magpie 机器建立 SSH 反向隧道，把 Magpie 机器的
`<隧道端口>` 映射到中转机的反代端口：

```bash
ssh -N -R 127.0.0.1:<隧道端口>:127.0.0.1:<中转机本地端口> \
  -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes \
  -o BatchMode=yes -i ~/.ssh/id_ed25519 \
  <Magpie 机器的 SSH 账号>@<Magpie 机器地址>
```

> 注意：如果中转机之前连过 Magpie 机器，而对方的 host key 换过，
> 会报 `REMOTE HOST IDENTIFICATION HAS CHANGED`。先在中转机上清掉旧 key：
> `ssh-keygen -R <Magpie 机器地址>`，再重连。

在 Magpie 机器上验证隧道端口已监听：

```bash
ss -tlnp | grep <隧道端口>
# 应看到 sshd-session 在监听该端口
```

## 4. Magpie 机器：配置本机反代

在 Magpie 机器上起一个 nginx，只做一层转发（把请求交给隧道端口）。
以 Baota nginx 为例，写一个 vhost（路径按你的 nginx 安装调整）：

```nginx
server {
    listen 127.0.0.1:<本机反代端口>;
    location / {
        proxy_pass http://127.0.0.1:<隧道端口>;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_connect_timeout 10s;
        proxy_read_timeout 120s;
        proxy_send_timeout 120s;
    }
}
```

重载 nginx 后验证：

```bash
nginx -s reload
curl -s -o /dev/null -w "%{http_code}\n" --max-time 15 \
  http://127.0.0.1:<本机反代端口>/gateway/apis/v1/chat/completions
# 应返回 400
```

## 5. Magpie：设置环境变量并重启

`DUMATE_GATEWAY_BASE` 必须对运行 Magpie 的进程可见。启动 Magpie 时带上：

```bash
DUMATE_GATEWAY_BASE=http://127.0.0.1:<本机反代端口>/gateway/apis/v1 magpie tui
```

或写进 Magpie 的启动方式（systemd 服务的 `Environment=`、启动脚本等），
然后重启 Magpie。注意地址结尾不要带 `/`。

验证一次真实调用：

```bash
magpie provider test dumate deepseek-v41-flash
```

看到模型正常返回即成功。也可以用 Codex 实测：

```bash
codex exec --skip-git-repo-check -c model="dumate/deepseek-v41-flash" \
  "Reply with the single word OK."
```

## 6. 中转机：设置开机自启（systemd）

让反代和隧道在中转机重启后自动恢复。新建两个 service：

`/etc/systemd/system/dumate-relay-proxy.service`：

```ini
[Unit]
Description=DuMate gateway reverse proxy
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/root/dumate-relay
ExecStart=/usr/bin/python3 /root/dumate-relay/proxy.py
Restart=always
RestartSec=5
StandardOutput=append:/root/dumate-relay/proxy.log
StandardError=append:/root/dumate-relay/proxy.log

[Install]
WantedBy=multi-user.target
```

`/etc/systemd/system/dumate-relay-tunnel.service`：

```ini
[Unit]
Description=DuMate relay SSH reverse tunnel to Magpie machine
After=network-online.target dumate-relay-proxy.service
Wants=network-online.target
Requires=dumate-relay-proxy.service

[Service]
Type=simple
User=root
ExecStart=/usr/bin/ssh -N -R 127.0.0.1:<隧道端口>:127.0.0.1:<中转机本地端口> -o ServerAliveInterval=30 -o ServerAliveCountMax=3 -o ExitOnForwardFailure=yes -o BatchMode=yes -o StrictHostKeyChecking=accept-new -i /root/.ssh/id_ed25519 <Magpie 机器的 SSH 账号>@<Magpie 机器地址>
Restart=always
RestartSec=10
StandardOutput=append:/root/dumate-relay/tunnel.log
StandardError=append:/root/dumate-relay/tunnel.log

[Install]
WantedBy=multi-user.target
```

启用并启动：

```bash
systemctl daemon-reload
systemctl enable dumate-relay-proxy.service dumate-relay-tunnel.service
systemctl start dumate-relay-proxy.service
systemctl start dumate-relay-tunnel.service
systemctl is-active dumate-relay-proxy.service dumate-relay-tunnel.service
# 两个都应为 active；is-enabled 都应为 enabled
```

> 如果之前是用 `nohup` 手动起的反代/隧道，先把旧进程停掉再启动 service，
> 避免端口冲突（`kill <PID>`，用确切 PID，不要用 `pkill -f`，后者可能
> 把自己的 SSH 会话也杀掉）。

## 7. 排查

| 现象 | 可能原因 |
| --- | --- |
| `magpie provider test` 报 `no answer in 1m0s` | 中转链路不通：按第 2、3、4 步的 curl 逐段验证（中转机反代、隧道端口、本机反代）。 |
| 本机反代返回 502 | 隧道断了：检查中转机上的 tunnel service 是否 active，重启它；检查中转机反代是否还在运行。 |
| 隧道连不上，报 host key 变化 | 中转机 `known_hosts` 里存了 Magpie 机器的旧 key：`ssh-keygen -R <Magpie 机器地址>` 后重连。 |
| 返回 401 `Incorrect API Key` | 链路已通，但网关 token 无效：重新登录 DuMate（`magpie plugin login dumate`），或检查登录态是否过期。 |
| 返回 400 `DuMate only accepts its official chat endpoint` | 请求地址与 `DUMATE_GATEWAY_BASE` 不一致：检查环境变量是否真正生效（重启 Magpie），以及地址结尾是否多了 `/`。 |
| 一开始能用，过段时间不行 | 登录态（Cookie / token）过期：重新登录；或中转机、隧道因重启/断网而中断（已配 systemd 的会自动恢复）。 |

## 注意

- 中转机、隧道、Magpie 都是单点：任一环节中断调用就失败；重要用途请考虑固定中转机与自动重连（本文第 6 步已覆盖）。
- 不要把真实服务器地址、端口、Cookie、token 写进公开仓库或公开文档。
- 本插件不修改、不上传 DuMate 桌面端的登录文件；中转只转发聊天请求，不经过作者或第三方的服务器（除非你自己搭的中转机）。
