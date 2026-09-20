# HTTPS 证书与域名申请总结

日期：2026-09-17 · 状态：**已上线并验证（现行主文档）· 2026-09-17 更新**

本文说明当前生产事实、决策依据、替代路线边界、自愈与排错。早期路线调研与 Windows/纯 IP 候选清单已完成合并并删除。网络路由规则见[../Nginx反向代理机制.md](../Nginx反向代理机制.md)。

> **2026-09-17 方向变更（重要）**
> - `lugwit.duckdns.org` **已被国内网络按 `duckdns.org` 关键字做 SNI 拦截**：对该域名（含任意不存在的 `*.duckdns.org` 子域）TLS 握手一律被 RST；而同机器上任意其它未知 SNI（如 `no-such-host.example.com`）却能正常握手 → 这是**网络侧 SNI 关键字拦截**，非服务器/证书问题，改 nginx 也救不了。**DuckDNS 域名路线弃用**。
> - 域名新方向（2026-09-17 拍板）：**购买 `lugwit.cn`，用非标端口 `8443` + Let's Encrypt（阿里云 DNS-01 自动续）**，理由是**免 ICP 备案**；443 暂时保留 IP 自签证书（纯 IP 访问不受备案影响）。见 §4.3。
> - 若要"域名 + 443 不带端口"，必须走 **ICP 备案**（国内 ECS 未备案域名访问 80/443 会被拦）。
> - 客户端/运维工具默认入口已从失效域名改为 IP：**`https://121.196.144.88/script_editor`**（见 §10、§11.5）。

---

## 1. 结论速览（当前生产事实）

| 项 | 值 |
|---|---|
| 生产访问入口 | IP `https://121.196.144.88`（443，自签证书，2026-09-16/17） |
| 域名 | `lugwit.duckdns.org`（DuckDNS 免费子域）→ A 记录 `121.196.144.88` —— **已弃用**：国内网络按 `duckdns.org` SNI 关键字拦截（任意 `*.duckdns.org` 握手被 RST），域名方向见 §4.3 |
| 证书①（IP 访问） | 自签 `C:/certs/lugwit/{fullchain.pem,privkey.pem}`，CN=IP，SAN=本机 IPv4，3650 天 |
| 证书②（域名访问） | Let's Encrypt（EC P-256）`C:/certs/lugwit_le/{fullchain.pem,privkey.pem}`，90 天自动续 —— 但 `lugwit.duckdns.org` 当前被 SNI 拦截、不可用 |
| nginx 443 | 双 server 块按 SNI 分流：默认块=自签（裸 IP），域名块=LE（`lugwit.duckdns.org`） |
| 公网入口 | **只有 443**；8080 已收成 `listen 127.0.0.1:8080`（仅本机，与 443 共用 `conf/routes.conf`） |
| 客户端 | App（Capacitor WebView）走 `https://121.196.144.88`，**必须内置自签 CA**（`res/raw/lugwit_ca.pem`，与线上 443 证书 sha256 前 8 位 `1d2e7cd6` 一致，2026-09-16/17 实测） |

---

## 2. 为什么是这套（决策依据）

| 事实 | 排除的路线 |
|---|---|
| 服务器 **80 端口被 IIS 10.0 占用**（`/.well-known/...` 返回 404） | ✗ HTTP-01（Let's Encrypt / ZeroSSL / win-acme 文件验证都走 80） |
| 服务器是 **Windows**，没有域名也不打算备案 | ✗ 常规 CA 的域名验证；✓ 自签（IP）+ 免费域名（DNS-01） |
| 只有 IP 的免费 DV 证书要么 90 天（ZeroSSL，仍要 80）、要么 ~6 天（LE shortlived） | ✗ 纯 IP 走正式 CA 不划算 |
| DuckDNS 免费、提供 DNS 更新 API、支持 DNS-01 | ✗ DuckDNS 域名现被 SNI 拦截（见顶部）；✓ 改用自有域名 `lugwit.cn` + DNS-01（见 §4.3） |

两条腿并行：**IP 自签**保证"裸 IP 也能通"（10 年不用管），**域名 + LE** 保证"全平台天然可信"（App/浏览器零告警）。

> 2026-09-17 追加：DuckDNS 虽能签出 LE 证书，但域名**被国内网络 SNI 拦截**，故"域名 + LE"这条腿改用自有域名 `lugwit.cn` + 非标端口 8443（见 §4.3）。

---

## 3. 域名申请（DuckDNS）——已弃用（保留作历史）

> ⚠️ **该路线已弃用（2026-09-17）**：`lugwit.duckdns.org` 被国内网络按 SNI 关键字拦截，握手被 RST，不可用。下面步骤仅作历史记录；域名方向见 §4.3。

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
| `ca.pem` | App 内置信任用（`res/raw/lugwit_ca.pem` + `network_security_config.xml`）——**域名弃用后必须内置**；域名路线若恢复才可选 |

### 4.2 域名证书（Let's Encrypt，DNS-01 自动续）——DuckDNS 已弃用

> ⚠️ **该路线（DuckDNS + LE）已弃用（2026-09-17）**：域名被 SNI 拦截，证书签得出来也用不了。下面命令仅作历史记录；新方向见 §4.3。

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

### 4.3 新方向：`lugwit.cn` + 8443 + LE（阿里云 DNS-01，2026-09-17 拍板）

DuckDNS 域名被 SNI 拦截后（见 §1 / 顶部），"域名 + LE"这条腿改为：

| 项 | 决定 | 理由 |
|---|---|---|
| 域名 | 购买 **`lugwit.cn`**（阿里云） | 自有域名、DNS 可控 |
| 端口 | **8443（非标端口）** | **免 ICP 备案**；国内 ECS 未备案域名走 80/443 会被拦 |
| 证书 | **Let's Encrypt** + **阿里云 DNS-01** 自动续 | DNS-01 不碰 80，绕开 IIS 占用 80 的问题 |
| 443 | 暂时**保留 IP 自签**（`C:/certs/lugwit`） | 纯 IP 访问不受备案影响 |
| 若要"域名 + 443 不带端口" | 必须 **ICP 备案** | 国内 ECS 未备案域名访问 80/443 会被拦 |

> 落地步骤（待执行，非现状）：买域名 → 阿里云 DNS API token → acme.sh 用 `dns_ali` 签发 `lugwit.cn` → nginx 加 `listen 8443 ssl` 的 server 块（`conf/https.conf`）→ 客户端/工具入口改 `https://lugwit.cn:8443/...`。

---

## 5. nginx 接线

| 文件 | 作用 |
|---|---|
| `conf/lugwit.conf` | 主配置（8080 只绑回环）+ `include https.conf;` + 8080 server 块 `include routes.conf;` |
| `conf/https.conf` | 443 两个 server：默认块（自签，裸 IP）/ 域名块（LE） |
| `conf/https_common.conf` | 两块共用：TLS 参数、`client_max_body_size 100g`、`include routes.conf;`（直接命中 location，不经 8080） |
| `conf/routes.conf` | **唯一一份 location 集合**，被 443 与 8080 共同 include |

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
curl.exe -k --resolve 121.196.144.88:443:121.196.144.88 https://121.196.144.88/nginx-health

# 外网侧（浏览器；注意开发机可能被办公网挡住 443，见 §8）
https://121.196.144.88/homepage      → 302 → 登录页
https://121.196.144.88/script_editor/status → 200
```

---

## 8. 排错手册（都是实际踩过的）

| 症状 | 真因 | 处理 |
|---|---|---|
| `[emerg] cannot load certificate ... No such file` | 该机器没有 `C:/certs/**` | 跑 §6 的自愈，或手动执行 §4.1 |
| 某个端口 HTTPS 全 `000`（连 `/nginx-health` 都 000），但浏览器能打开 | **本机出站 443 被网络策略封**，不是服务器问题 | 用对照组判定：`curl -s -o NUL -w "%{http_code}" https://www.baidu.com` 同样 000 → 本机问题；换浏览器/换网络验证 |
| 443 昨天还好、今天完全不通 | conf 里**生效的** `include https.conf;` 被覆盖成注释态（同步/仓库覆盖）→ reload 后 443 消失 | `findstr /I "include https.conf" <conf>` 确认不是 `#` 注释；改回后 `nginx -t && reload` |
| `nginx -t` 通过但域名打不开 | 域名块 `server_name` 拼错 / 证书签发域名不一致 | 用 `--resolve` 到 127.0.0.1 试 SNI，或看 §7 的 SNI 自检 |
| **域名 HTTPS 握手直接 RST，但裸 IP 正常**；任意 `*.duckdns.org` 都 RST、其它未知 SNI 却正常 | **国内网络按 `duckdns.org` SNI 关键字拦截** | 不是服务器/证书问题，改 nginx 无用；换自有域名（`lugwit.cn`，见 §4.3） |
| `urllib` 报 `CERTIFICATE_VERIFY_FAILED: self-signed certificate` | Python 默认信任库不认自签证书 | 消费 `l_qframelesswindow`（随包分发 CA，见 §11）；运维工具走 `tools/_tls.py` |
| acme.sh 续期失败 | ① `DuckDNS_Token` 变量名/值不对 ② 缺 `dnsapi/dns_duckdns.sh`（只下了 acme.sh 主文件） ③ 计划任务里 powershell 不可用 | 手动跑 `--cron` 看输出；确认仓库完整；任务用 **cmd/git-bash** 而非 powershell |

---

## 9. 关键路径清单

```
C:/certs/lugwit/{fullchain,privkey,ca}.pem     自签（IP 访问 + App 内置 CA）
C:/certs/lugwit_le/{fullchain,privkey}.pem      Let's Encrypt（域名访问）
C:/certs/acme-home/                            acme.sh 状态（账号/域名配置/续期参数，含 DuckDNS token，勿外传）
C:/lugwit_ops/acme-repo/                       acme.sh 完整仓库（含 dnsapi）
l_nginx/999.0/conf/{lugwit.conf,https.conf,https_common.conf,routes.conf}
l_nginx/999.0/tools/make_self_signed_cert.py   自签生成器
l_nginx/999.0/tools/_tls.py                    远程运维工具共用 HTTPS 请求层（自签证书降级）
l_nginx/999.0/src/l_nginx/nginx_cli_with_homepage.py   启动前补证书（ensure_https_certs）
l_qframelesswindow/999.0/src/l_qframelesswindow/config/ca_bundle.pem   随包分发 CA（自签 + 公网根，单文件）
l_qframelesswindow/999.0/src/l_qframelesswindow/config/ca.pem          随包分发 CA（仅自签）
```

---

## 10. 仍待办 / 已知风险

| 项 | 说明 |
|---|---|
| 未开 301 强制跳转与 HSTS | 确认 HTTPS 稳定后再开；开了要能回滚（注释掉即可） |
| `/nginx-health` 未设 `default_type text/plain` | 浏览器会当下载，加一行即可 |
| DuckDNS 30 天活动要求 | **已弃用**（域名被 SNI 拦截）；域名改用 `lugwit.cn`（见 §4.3） |
| 8764（脚本编辑器）端口暴露 | **8764 由各宿主固定端口 + 服务发现（`~/.Lugwit/run/<service>.json`）+ IPC 命名管道提供**，详见 `Rez_pkg/l_script_editor.md`（不再公网直连该端口） |
| 443 出站被封的开发机 | 该机器上的工具（`remote_exec.py` 等）需 `--host` 指定可达通道；**默认值已是网关基址 `https://121.196.144.88/script_editor`**（见 §11.5） |
| 自签证书轮换 | 服务器重签 `C:/certs/lugwit` 后，须同步更新包内 `ca_bundle.pem`（服务器当前证书与包内自签 CA 的 SHA-256 应一致），见 §11.4 |

---

## 11. Python 侧证书信任（`l_qframelesswindow` 随包分发 CA）

服务器 443 用的是**自签证书**，Python 默认信任库不认，裸 `urllib.request.urlopen` 会
`CERTIFICATE_VERIFY_FAILED: self-signed certificate`（登录 / token 校验全失败）。为此 `l_qframelesswindow` 随包分发 CA：

| 文件（包内 `src/l_qframelesswindow/config/`） | 内容 |
|---|---|
| `ca_bundle.pem` | **1 张自签 CA + 121 张公网根装在同一个文件**（必须单文件，见下） |
| `ca.pem` | 只含自签 CA |

### 11.1 查找顺序与安装

`ssl_support.ca_file()` 的查找顺序：

```
LUGWIT_CA_FILE（环境变量）
> 包内 config/ca_bundle.pem
> 包内 config/ca.pem
> C:/certs/lugwit/ca.pem
```

- `ssl_support.install_default_ca()`：用 `ssl.create_default_context(cafile=…)` **直接重建**默认 HTTPS context 工厂，让**裸 `urlopen` 也信任**自签证书。
- ⚠️ **实测坑（2026-09-17，Windows + OpenSSL 3）**：**先建默认 context 再 `load_verify_locations()` 追加自签 CA 无效**，否则会混入系统库里同 CN 的旧证书**毒化校验**。只有 `create_default_context(cafile=…)`（把自签 CA 与公网根放同一文件）才认。
- 同一份 bundle 也由 `package.py` 写进环境变量：`SSL_CERT_FILE` 与 `REQUESTS_CA_BUNDLE` 都指向 `ca_bundle.pem`（覆盖 ssl/OpenSSL 与 requests/urllib3 两条路径）。

### 11.2 零安装信任

只要消费方包声明了 `requires: l_qframelesswindow`（如 `l_notepad_client`、`l_notepad_server`、`lugwit_netdisk_client` 等），**零安装**即得信任（包 env 注入 `SSL_CERT_FILE` / `REQUESTS_CA_BUNDLE`，`ssl_support` 再兜底重建默认 context）。

### 11.3 网页/WebView 侧的缺口与解法（2026-09-19 补充）

**先分清"网页"有几种形态** —— 只有第 3、4 类没解决：

| # | 形态 | 状态 | 依据 |
|---|---|---|---|
| 1 | 桌面 Python（`urllib` / `requests`） | ✅ 已解决 | §11、§11.2：随包分发 `ca_bundle.pem` + `SSL_CERT_FILE` / `REQUESTS_CA_BUNDLE` + `install_default_ca()` |
| 2 | Android WebView | ✅ 已解决 | `l_WChat/.../res/xml/network_security_config.xml` + `res/raw/lugwit_ca.pem` |
| 3 | **内嵌 QtWebEngine 页面**（`l_notepad_client`、`lugwit_netdisk_client` 里的 QWebEngineView） | ❌ **未解决** | Chromium 用自己的信任库，Python 的 `ca_bundle` 对它**无效** → 指向 `https://121.196.144.88/...` 会**白屏 / 证书错误** |
| 4 | **外部浏览器**（Chrome/Edge 打 `/baidu` `/note` `/chat` `/homepage` `/docs/`） | ❌ **未解决** | 仓库内**无任何装根证书的手段**（全仓搜 `certutil` / `addstore` 只命中一条与 Appx 签名无关的文档） |
| 5 | 托盘 `l_tray` 的 Python | ⚠️ 缺口 | 不依赖 `l_qframelesswindow` → 拿不到 CA 分发（本机回环 `http://127.0.0.1:1028` 无碍，指远端 https 才失败） |

> 今天没炸的原因：内部桌面程序的页面一律走**本机回环 `http://127.0.0.1:8080`**（明文、无 TLS）。
> `l_notepad_server` 的 CHANGELOG 明确记着：服务端调认证固定走回环 8080，**就是为了绕开自签证书**。
> 只有"从别的机器用浏览器打开 IP 入口"才暴露第 3、4 类。

**解法（按推荐顺序）**

| 方案 | 覆盖 | 成本 | 说明 |
|---|---|---|---|
| **1. 把自签 CA 装进 Windows 根库** | 第 3、4 类（**一次解决**），顺带解掉第 5 类 | 一次性脚本 + 管理员权限 | 新脚本：**`l_nginx/999.0/tools/install_lugwit_ca.bat`**（安装 / `/uninstall` / `/y`）。Chrome、Edge、QtWebEngine 在 Windows 上读**系统根库** → 装一次这几类都好（**待实测确认**：装后重启浏览器/客户端再验） |
| 2. 接 `QWebEnginePage.certificateError` | 只管自家客户端内嵌页；外部浏览器无效 | 小 | ⚠️ 必须**按证书指纹白名单**放行，别写成无条件忽略（那是 MitM 敞口）。仓库先例：`ChatRoom/.../l_cgtw/maya_plugin/maya_plugin.py:427,522` |
| 3. 上域名 + Let's Encrypt（`8443`，免备案） | 根治 | 买域名 | 已拍板方向（§4.3、`Nginx反向代理机制` §11.4） |

**脚本 1 的实测记录（2026-09-19，本机 A/B/A 验证）**

**✅ 结论：装根库有效，且「用户库」就够 —— 不需要 `/all`、不需要管理员/UAC。**

| 阶段 | 动作 | 结果 |
|---|---|---|
| A（基线） | Chromium（Playwright，未忽略证书错）访问 `https://121.196.144.88/nginx-health` | `net::ERR_CERT_AUTHORITY_INVALID` |
| B（装用户库） | `certutil -addstore -user -f Root <ca.pem>` | `/baidu/` **200**、`/api/v1/health` **200** |
| A′（撤销） | PowerShell `Remove-Item` 删除 | `ERR_CERT_AUTHORITY_INVALID` **复现** |

据此脚本已改为**默认装 `CurrentUser` 库**（免 UAC），`/all` 才装机器库。

**实测发现的两个真实坑（都已写进脚本）**

1. **库里早有旧指纹残留**：`Cert:\CurrentUser\Root` 与 `Cert:\LocalMachine\Root` 里都存在
   `O=Lugwit Internal, CN=121.196.144.88`，但指纹是 **`478768654210E7F20DF95BC30E4BEBBAA4FF2A00`（旧）**，
   而服务器**现在出示的是 `831663D9907F224B0EA4AA01F746BD33FEB17314`**（等于包内 `ca.pem`）。
   → 这正是 §11.4 警告过的"重签未同步"，也是 `ssl_support` 注释里那句"三者指纹互不匹配"的来源。
   这种状态下"看着装了却仍不信任"。**脚本装前会列出同 Subject 不同指纹的条目**并提示清理。
2. **卸载不能用 `certutil -delstore`**：删根库时它弹确认框，非交互调用直接
   `ERROR_CANCELLED(1223)`，且**证书仍在**（实测：删完再查指纹还在）。脚本改用
   PowerShell `Remove-Item` 静默删除（实测输出 `REMOVED`、复查 `after=False`）。

**写 .bat 本身的两个坑（踩过，务必照着写）**

| 坑 | 现象 | 处置 |
|---|---|---|
| **文件编码** | 存成 UTF-8（无 BOM）时 cmd 按 GBK 解析 → 中文字节**吃掉后面的 ASCII**（`usebackq` 被啃成 `ebackq`、`echo` 成 `ho`），脚本整体崩 | **用 GBK/ANSI 保存**；不加 `chcp 65001`（会让 GBK 输出变乱码） |
| **注释/输出里的 `>`** | `rem/echo` 行出现大于号时 cmd **仍按重定向处理**（`rem ... -> 装后...` 触发重定向），后续行错位成命令 | rem/echo 里**不写重定向符号**，用 `→` 或文字表述 |
| 证书信息读取 | `certutil -dump` 对 **PEM 不输出** `Subject` / `Cert Hash(sha1)` 行（只有转 DER 才给）；临时文件 + `WriteAllLines` 传数组实测把 **4 行并成 1 行** | 改用 PowerShell `X509Certificate2` 读 PEM，**一次取一个值** |

**验收口径**：用 `https://121.196.144.88/baidu/` 或 `/api/v1/health`。
⚠️ **别用 `/nginx-health`**：该端点响应非 HTML，浏览器会当下载处理（`net::ERR_ABORTED`），与证书无关，容易误判。

**待办**：QtWebEngine（`l_notepad_client` / `lugwit_netdisk_client`）未单独验（与 Chromium 同源、同走系统库，预期一致）；

**⚠️ 装根证书的安全须知（必须一起讲清）**

1. 装完后，**凡持有这张 CA 私钥签发的证书者，都能对本机所有 HTTPS 做中间人**。
2. 当前自签形态下**「服务器证书私钥」与「CA 私钥」是同一把**（`C:/certs/lugwit/privkey.pem`，自签 = leaf 即 CA）→ 一旦外泄，全平台流量可被劫持。**只放服务器、严格保管**。
3. 只对可信内网机器推送；公用机/离职机建议先 `/uninstall`。
4. 长期更稳的做法：把"自签 CA"改成**只签服务器证书的中间 CA** + 短有效期，把私钥用途分离。

### 11.4 证书轮换

服务器重签 `C:/certs/lugwit` 后**必须同步更新包内 `ca_bundle.pem`**（核对方法：**服务器当前证书与包内自签 CA 的 SHA-256 应一致**）。

### 11.5 运维工具的证书处理

`l_nginx/999.0/tools/_tls.py`：远程运维工具（`remote_exec.py` / `remote_sync.py` / `remote_push.py`）共用的 HTTPS 请求层。这些工具常在 rez 环境外直接跑，拿不到包注入的 `SSL_CERT_FILE`，故：

- CA 查找顺序：`SSL_CERT_FILE` > `C:/certs/lugwit/ca.pem`；
- CA 校验优先，失败（如证书已轮换）**降级为不校验并警告一次**，保证运维通道不被证书问题卡死。
- 三个工具的**默认 `--host` 已改为 `https://121.196.144.88/script_editor`**（从失效域名改为 IP）。
- 注意：**8764 本身是明文 HTTP**，`--host http://…:8764` 直连不涉及证书；只有走 443 网关 `https://…/script_editor` 才用得上。
