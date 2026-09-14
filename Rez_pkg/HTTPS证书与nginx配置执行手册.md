# HTTPS 证书申请与 nginx 配置执行手册

日期：2026-09-14 · 相关：`Rez-Docs/Rez_pkg/l_WChat_上传直连百度改造计划.md`（M0 前置）
配套改动：`l_nginx/999.0/conf/lugwit.conf` 末尾已写好**注释版** 443 段与 `/baidu-direct/` 模板（取消注释即用）

---

## 0. 先说清楚：能自动做的 / 必须你做的

| 环节 | 谁做 | 说明 |
|---|---|---|
| 写 nginx 443 配置、acme 申请脚本、续期任务、验收清单 | **我** | 已经写进本手册与 `lugwit.conf` |
| **证明你控制该域名/IP**（HTTP-01 放文件 / DNS-01 加 TXT） | **你**（在服务器上跑命令） | 这是 CA 的硬要求，任何人都不能代做；我这边只能操作你的开发机，连不到服务器 80 端口 |
| 在服务器上执行签发、重载 nginx | **你** | 命令我按 Linux / Windows 两种都写了 |
| 出错时把报错贴回来 | **你** | 我按报错继续改 |

> 我无法替你"申请"证书，也**不建议**用浏览器 MCP 去点 CA 控制台（要你的账号密码、且验证这步还是要落到服务器上）。

---

## 1. 先回答三个前提（决定走哪条路）

| 问题 | 影响 |
|---|---|
| **有没有域名？**（或愿不愿意弄一个免费的） | 有域名 → 走 **路线 A**（最省心，90 天自动续）；只有 IP → 路线 B |
| **服务器 80 端口能否从公网访问？** | HTTP-01 验证要它；不能开 80 → 用 **DNS-01**（需要域名 DNS 的 API key）或路线 C |
| **服务器是什么系统？** | Linux → acme.sh 标准流程；Windows → 见 §3.3 |

---

## 2. 路线 A：有域名 → Let's Encrypt（推荐）

**最省心、零成本、90 天自动续期。**

### 2.1 准备
- 域名 A 记录指向服务器公网 IP
- 80 端口（HTTP-01）或 DNS API（DNS-01）可用

### 2.2 安装 + 签发（Linux，acme.sh）
```bash
# 1) 安装 acme.sh（不依赖 root，装到 ~/.acme.sh）
curl https://get.acme.sh | sh -s email=you@example.com
export PATH="$HOME/.acme.sh:$PATH"

# 2) HTTP-01 签发（webroot 方式，nginx 已经在跑就用这个）
#    先在 nginx 里给 /.well-known/acme-challenge/ 指一个目录，例如 /var/www/acme
acme.sh --issue -d wchat.example.com --webroot /var/www/acme

#    或者：临时占用 80 端口（需要先停 nginx）
# acme.sh --issue -d wchat.example.com --standalone

# 3) DNS-01（80 端口不可用时；以阿里云 DNS 为例，需要 API key）
# export Ali_Key="xxx" Ali_Secret="yyy"
# acme.sh --issue --dns dns_ali -d wchat.example.com

# 4) 安装证书到固定路径 + 续期后自动重载 nginx
mkdir -p /etc/nginx/certs/lugwit
acme.sh --install-cert -d wchat.example.com \
  --key-file       /etc/nginx/certs/lugwit/privkey.pem \
  --fullchain-file /etc/nginx/certs/lugwit/fullchain.pem \
  --reloadcmd      "nginx -s reload"
```

nginx 里给 HTTP-01 留位置（加入 8080 那个 server 块）：
```nginx
location ^~ /.well-known/acme-challenge/ {
    root /var/www/acme;
    default_type "text/plain";
}
```

### 2.3 打开 HTTPS
- 编辑 `l_nginx/999.0/conf/lugwit.conf` 末尾那段注释：
  选**方案 A**（443 直接绑同一套 location，少一跳）或**方案 B**（443 → 127.0.0.1:8080）
- 把 `ssl_certificate` 路径改成你实际的（Linux 上是 `/etc/nginx/certs/lugwit/...`）
- 有域名就把 `server_name _;` 改成域名
- `nginx -t && nginx -s reload`
- 确认无问题后再加 `return 301 https://...`（强制跳转）与 HSTS

---

## 3. 路线 B：只有 IP（无域名）

| CA | 有效期 | 验证 | 备注 |
|---|---|---|---|
| **ZeroSSL** | **90 天** | 仅 HTTP-01（需 80 端口） | 免费额度可无限签；ACME 支持，需在控制台拿 EAB key |
| Let's Encrypt IP 证书 | ~6 天（160h，2026-01 起 GA） | 仅 HTTP-01 | 续期太频繁，脚本要很可靠 |

### 3.1 ZeroSSL + acme.sh（推荐给纯 IP）
```bash
# 1) 到 ZeroSSL 控制台注册 → Developer → 生成 EAB (KID + HMAC key)
export EAB_KID="..."
export EAB_HMAC_KEY="..."

# 2) 用 ZeroSSL 作为 ACME 服务器签发 IP 证书
acme.sh --register-account --server zerossl \
        --eab-kid "$EAB_KID" --eab-hmac-key "$EAB_HMAC_KEY"
acme.sh --issue --server zerossl -d 121.196.144.88 --webroot /var/www/acme

# 3) 安装 + 续期重载（同路线 A 第 4 步）
```

### 3.2 Let's Encrypt IP 证书（6 天）
```bash
acme.sh --issue --server letsencrypt --profile shortlived \
        -d 121.196.144.88 --webroot /var/www/acme
# 续期要跑得很勤（acme.sh 会自动按证书有效期排 cron；确认 cron 生效）
```

### 3.3 Windows 服务器（如果 121.196.144.88 是 Windows）
- acme.sh 需要 bash；Windows 上更实际的是：
  - **win-acme**（https://www.win-acme.com/）：图形/命令行向导，支持 HTTP-01 与 DNS，自动装到 IIS 或导出 pem 给 nginx
  - 或 **ZeroSSL 控制台手动签发 + 手动上传**（90 天，记得建提醒）
- nginx（Windows 版，你们的 `l_nginx/bin/nginx.exe`）证书路径用正斜杠：
  `ssl_certificate C:/certs/lugwit/fullchain.pem;`

---

## 4. 路线 C：不想开 80/443、也不想管证书 → Cloudflare Tunnel

适合"服务器不想暴露入站端口"的场景：服务器只**出站**连 Cloudflare，CF 给你一个 HTTPS 域名。

```bash
# 服务器上（需要域名托管在 Cloudflare）
cloudflared tunnel login
cloudflared tunnel create lugwit
cloudflared tunnel route dns lugwit wchat.example.com
# 配置 config.yml：ingress 指向本机 8080
cloudflared tunnel run lugwit
```
- 优点：不用证书、不用开 80/443、CF 自动证书
- 缺点：需要一个域名；流量经 CF（境外节点时延敏感场景要评估）

---

## 5. 上线后要同步改的地方

| 位置 | 改动 |
|---|---|
| `wchat-android/capacitor.config.json` | `server.url` → `https://<域名或IP>`；删掉 `"cleartext": true`（或改 false） |
| 安卓壳 | 重新打包发版（Capacitor `npx cap sync android && gradle assembleRelease`） |
| `server_config.py`（各包） | 若有硬编码 `http://121.196.144.88:8080` 的地方，改 https（注意别把服务端内部 `127.0.0.1:xxxx` 也改掉） |
| nginx | 打开 `return 301 https://$host$request_uri;` + HSTS（确认稳定后再开） |
| 证书续期 | `acme.sh` 自带 cron；确认 `--reloadcmd "nginx -s reload"` 生效，并**加一个到期告警**（到期前 7 天） |

---

## 6. 验收清单

1. `openssl s_client -connect <host>:443 -servername <host> | openssl x509 -noout -subject -dates` → 证书主体/有效期正确
2. 浏览器/手机访问 `https://<host>/l_wchat/...` 无证书告警；`http://` 自动 301 到 https
3. 抓包（手机侧）确认 **cookie `lugwit_token` 为密文**（TLS 内），不再是明文
4. App（WebView）能正常加载页面、登录、上传、下载
5. `curl -I http://<host>:8080` → 301 到 https（若已开强制跳转）
6. nginx 日志里 `/baidu-direct/` 的 access_log 已关闭（配了的话）

---

## 7. 回滚

- nginx：注释掉 443 段 → `nginx -t && nginx -s reload`（8080 一直是好的，随时可退回）
- App：`capacitor.config.json` 的 `server.url` 改回 `http://121.196.144.88:1234` + `cleartext: true`，重新发版
- 证书：不续期即自然过期，不影响 8080 的 HTTP 服务

---

## 8. 需要你提供的（我据此把脚本填成可直接跑的版本）

1. 有没有域名？域名是什么、DNS 服务商是哪家（决定 HTTP-01 / DNS-01）
2. `121.196.144.88` 是 Linux 还是 Windows？80/443 能否从公网访问？
3. 你能否在那台服务器上执行命令（能的话我按"你执行、我改脚本"的节奏推进）
4. 选哪条路线（A 域名+LE / B 纯 IP ZeroSSL / C Cloudflare Tunnel）

---

## 9. 我已完成的配置准备

- `l_nginx/999.0/conf/lugwit.conf` 末尾：**注释版** 443 段（方案 A / 方案 B 二选一）、
  `/baidu-direct/` 转发模板（含 `access_log off` + `auth_request` 两个必做项）、
  HTTP-01 challenge location 使用说明
- 注释不影响现有 nginx 启动；证书就位后取消注释即可
