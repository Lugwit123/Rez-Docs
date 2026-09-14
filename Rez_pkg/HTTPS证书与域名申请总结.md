# HTTPS 证书与域名申请总结

日期：2026-09-14 · 状态：**已上线并验证**
背景文档（同样内容的过程稿，保留路线对比）：
- `Rez-Docs/Rez_pkg/HTTPS证书与nginx配置执行手册.md`（三条路线总览）
- `Rez-Docs/Rez_pkg/HTTPS证书_IP证书执行清单.md`（纯 IP 各路线执行清单）

本文只讲**实际怎么做的**、**现在的生产事实**、**怎么自愈与排错**。

---

## 1. 结论速览（当前生产事实）

| 项 | 值 |
|---|---|
| 域名 | `lugwit.duckdns.org`（DuckDNS 免费子域）→ A 记录 `121.196.144.88` |
| 证书①（IP 访问） | 自签 `C:/certs/lugwit/{fullchain.pem,privkey.pem}`，CN=IP，SAN=本机 IPv4，3650 天 |
| 证书②（域名访问） | Let's Encrypt（EC P-256）`C:/certs/lugwit_le/{fullchain.pem,privkey.pem}`，90 天自动续 |
| nginx 443 | 双 server 块按 SNI 分流：默认块=自签（裸 IP），域名块=LE（`lugwit.duckdns.org`） |
| 公网入口 | **只有 443**；8080 已收成 `listen 127.0.0.1:8080`（仅本机，供 443 反代） |
| 客户端 | App（Capacitor WebView）用 `https://lugwit.duckdns.org`；不再需要内置 CA（LE 天然可信） |

---

## 2. 为什么是这套（决策依据）

| 事实 | 排除的路线 |
|---|---|
| 服务器 **80 端口被 IIS 10.0 占用**（`/.well-known/...` 返回 404） | ✗ HTTP-01（Let's Encrypt / ZeroSSL / win-acme 文件验证都走 80） |
| 服务器是 **Windows**，没有域名也不打算备案 | ✗ 常规 CA 的域名验证；✓ 自签（IP）+ 免费域名（DNS-01） |
| 只有 IP 的免费 DV 证书要么 90 天（ZeroSSL，仍要 80）、要么 ~6 天（LE shortlived） | ✗ 纯 IP 走正式 CA 不划算 |
| DuckDNS 免费、提供 DNS 更新 API、支持 DNS-01 | ✓ 用它拿域名 + 自动签 LE，**完全不碰 80/IIS** |

两条腿并行：**IP 自签**保证"裸 IP 也能通"（10 年不用管），**域名 + LE** 保证"全平台天然可信"（App/浏览器零告警）。

---

## 3. 域名申请（DuckDNS）

1. 用 Google 账号登录 <https://www.duckdns.org> → 创建子域 `lugwit` → 得到 `<token>`。
2. 指向服务器（浏览器直接开，或服务器上 curl）：
   `https://www.duckdns.org/update?domains=lugwit&token=<token>&ip=121.196.144.88` → 返回 `OK`。
3. ⚠️ **两个运维要点**
   - DuckDNS 免费子域要求**每 30 天内有活动**（网页登录或调用一次 update），否则会被回收；
     建议让服务器**每天定时调一次 update 接口**（既保活、又顺带把动态 IP 追回来）。
   - token 等同密码（可改 DNS）：别进仓库、别贴聊天记录。

---

## 4. 证书获取

### 4.1 IP 自签（10 年，给裸 IP 访问）

```powershell
wuwor l_nginx -- python <l_nginx包>\999.0\tools\make_self_signed_cert.py --ip 121.196.144.88 --out C:/certs/lugwit
```

产物（`--help` 还支持 `--dns` / `--cn` / `--days` / `--bits`）：

| 文件 | 用途 |
|---|---|
| `privkey.pem` / `fullchain.pem` | nginx `ssl_certificate_key` / `ssl_certificate` |
| `ca.pem` | App 内置信任用（`res/raw/lugwit_ca.pem` + `network_security_config.xml`）——**改用域名后这一步可选** |

### 4.2 域名证书（Let's Encrypt，DNS-01 自动续）

```bash
# 服务器 Git Bash，acme.sh 完整仓库（必须含 dnsapi/dns_duckdns.sh）
export DuckDNS_Token=<token>          # ⚠ 变量名就是这个大小写，写成 DUCKDNS_TOKEN 会静默失败
/c/lugwit_ops/acme-repo/acme.sh --issue --dns dns_duckdns \
    -d lugwit.duckdns.org --server letsencrypt --keylength ec-256 \
    --home /c/certs/acme-home

/c/lugwit_ops/acme-repo/acme.sh --install-cert -d lugwit.duckdns.org --ecc \
    --key-file       C:/certs/lugwit_le/privkey.pem \
    --fullchain-file C:/certs/lugwit_le/fullchain.pem \
    --reloadcmd      "<l_nginx包>\999.0\bin\nginx.exe -s reload -p <runtime> -c <conf>" \
    --home /c/certs/acme-home
```

自动续期（acme.sh 的 `--cron` 每天跑一次，到期自动续 + reload nginx）：

```
schtasks /Create /TN lugwit_acme_renew /SC DAILY /ST 03:30 /RU SYSTEM /RL HIGHEST ^
  /TR "<git-bash.exe> -lc \"sh /c/lugwit_ops/acme-repo/acme.sh --cron --home /c/certs/acme-home\""
```

---

## 5. nginx 接线

| 文件 | 作用 |
|---|---|
| `conf/lugwit.conf` | 主配置（8080 只绑回环）+ `include https.conf;` |
| `conf/https.conf` | 443 两个 server：默认块（自签，裸 IP）/ 域名块（LE） |
| `conf/https_common.conf` | 两块共用：TLS 参数、`client_max_body_size 100g`、反代到 `127.0.0.1:8080`、`X-Forwarded-Proto https` |

```nginx
# conf/https.conf（要点）
server { listen 443 ssl default_server; server_name _;
         ssl_certificate     C:/certs/lugwit/fullchain.pem;
         ssl_certificate_key C:/certs/lugwit/privkey.pem;
         include .../https_common.conf; }
server { listen 443 ssl; server_name lugwit.duckdns.org;
         ssl_certificate     C:/certs/lugwit_le/fullchain.pem;
         ssl_certificate_key C:/certs/lugwit_le/privkey.pem;
         include .../https_common.conf; }
```

⚠️ 证书路径是**绝对路径**，而这份 conf 是**开发机与服务器共用**（同一仓库）——
所以"别人的机器"上一定没有这些文件。见 §6。

---

## 6. 证书缺失自愈（2026-09-14 新增，重要）

**踩过的坑**：`lugwit.conf` 里启用 `include https.conf;` 后，新机器/开发机没有 `C:/certs/lugwit*`
→ `nginx -t` 直接

```
nginx: [emerg] cannot load certificate "C:/certs/lugwit/fullchain.pem": ... No such file
```

而托盘"一键启动"是 `pythonw` + 隐藏窗口 → **无声失败**（nginx 起不来但看不到报错）。

**修法**：`l_nginx/999.0/src/l_nginx/nginx_cli_with_homepage.py` 新增 `ensure_https_certs()`，
在 `nginx_start` 分支里先于 `_ensure_homepage()` 执行：

- 只在 conf 里存在**生效的** `include https.conf;` 时才检查；
- 两套证书（`C:/certs/lugwit`、`C:/certs/lugwit_le`）任缺 → 用包内 `tools/make_self_signed_cert.py`
  就地自签（SAN 取本机全部 IPv4；域名那套 `CN/SAN=lugwit.duckdns.org`）；
- **幂等**：文件已在就整段跳过，**绝不覆盖服务器上的正式证书**。

自检（任何机器都能跑）：

```powershell
wuwor l_nginx -- python -c "from l_nginx.nginx_cli_with_homepage import ensure_https_certs as f; f()"
<l_nginx包>\999.0\bin\nginx.exe -t -p <runtime> -c <conf>     # 期望 test is successful
```

---

## 7. 验证与验收

```powershell
# 服务器侧：443 在听 + SNI 返回对的证书
netstat -ano -p tcp | findstr ":443 "
curl.exe -k https://127.0.0.1/nginx-health                      # ok
curl.exe -k --resolve lugwit.duckdns.org:443:127.0.0.1 https://lugwit.duckdns.org/nginx-health

# 外网侧（浏览器；注意开发机可能被办公网挡住 443，见 §8）
https://lugwit.duckdns.org/homepage      → 302 → 登录页
https://lugwit.duckdns.org/script_editor/status → 200
```

---

## 8. 排错手册（都是实际踩过的）

| 症状 | 真因 | 处理 |
|---|---|---|
| `[emerg] cannot load certificate ... No such file` | 该机器没有 `C:/certs/**` | 跑 §6 的自愈，或手动执行 §4.1 |
| 某个端口 HTTPS 全 `000`（连 `/nginx-health` 都 000），但浏览器能打开 | **本机出站 443 被网络策略封**，不是服务器问题 | 用对照组判定：`curl -s -o NUL -w "%{http_code}" https://www.baidu.com` 同样 000 → 本机问题；换浏览器/换网络验证 |
| 443 昨天还好、今天完全不通 | conf 里**生效的** `include https.conf;` 被覆盖成注释态（同步/仓库覆盖）→ reload 后 443 消失 | `findstr /I "include https.conf" <conf>` 确认不是 `#` 注释；改回后 `nginx -t && reload` |
| `nginx -t` 通过但域名打不开 | 域名块 `server_name` 拼错 / 证书签发域名不一致 | 用 `--resolve` 到 127.0.0.1 试 SNI，或看 §7 的 SNI 自检 |
| acme.sh 续期失败 | ① `DuckDNS_Token` 变量名/值不对 ② 缺 `dnsapi/dns_duckdns.sh`（只下了 acme.sh 主文件） ③ 计划任务里 powershell 不可用 | 手动跑 `--cron` 看输出；确认仓库完整；任务用 **cmd/git-bash** 而非 powershell |

---

## 9. 关键路径清单

```
C:/certs/lugwit/{fullchain,privkey,ca}.pem     自签（IP 访问 + App 内置 CA）
C:/certs/lugwit_le/{fullchain,privkey}.pem      Let's Encrypt（域名访问）
C:/certs/acme-home/                            acme.sh 状态（账号/域名配置/续期参数，含 DuckDNS token，勿外传）
C:/lugwit_ops/acme-repo/                       acme.sh 完整仓库（含 dnsapi）
l_nginx/999.0/conf/{lugwit.conf,https.conf,https_common.conf}
l_nginx/999.0/tools/make_self_signed_cert.py   自签生成器
l_nginx/999.0/src/l_nginx/nginx_cli_with_homepage.py   启动前补证书（ensure_https_certs）
```

---

## 10. 仍待办 / 已知风险

| 项 | 说明 |
|---|---|
| 未开 301 强制跳转与 HSTS | 确认 HTTPS 稳定后再开；开了要能回滚（注释掉即可） |
| `/nginx-health` 未设 `default_type text/plain` | 浏览器会当下载，加一行即可 |
| DuckDNS 30 天活动要求 | 建议服务器每日定时调 update 接口（顺带追动态 IP） |
| 8764（脚本编辑器）公网暴露 | 过渡期=直连 + 令牌 + 待加防火墙源 IP 限制；终态=只绑回环，远端走 `https://lugwit.duckdns.org/script_editor/` |
| 443 出站被封的开发机 | 该机器上的工具（`remote_exec.py`）需 `--host` 指定可达通道；默认值已是网关基址 |
