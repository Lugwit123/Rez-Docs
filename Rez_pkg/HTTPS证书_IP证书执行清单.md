# HTTPS 证书执行清单（纯 IP 121.196.144.88 · Windows 服务器）

配套：`Rez-Docs/Rez_pkg/HTTPS证书与nginx配置执行手册.md`（三条路线总览）
实测前提（2026-09-14）：

| 项 | 结果 |
|---|---|
| 域名 | 没有（全仓搜索无域名，全是 IP） |
| 80 端口 | **IIS 10.0** 占用，`/.well-known/acme-challenge/*` 返回 404 |
| 443 端口 | 空闲（证书到位即可启用 HTTPS） |
| 8080 | nginx/1.27.5，`/nginx-health` 200 |

结论：纯 IP 只能用 **ZeroSSL IP 证书（90 天，HTTP-01 必须走 80）** → 需让 IIS 托管挑战目录。
下面按"路线①（IIS + win-acme）"和"路线③（免费域名 + DNS-01）"分别给步骤。

---

## 路线①：IIS 托管 ACME 挑战目录 + win-acme 申请 ZeroSSL IP 证书

### ①-1 建挑战目录并让 IIS 提供它（管理员 PowerShell）

```powershell
New-Item -ItemType Directory -Force C:\acme-webroot | Out-Null

Import-Module WebAdministration
if (-not (Test-Path 'IIS:\Sites\Default Web Site\.well-known')) {
  New-WebVirtualDirectory -Site "Default Web Site" -Name ".well-known" -PhysicalPath C:\acme-webroot
}
```

验证：
```powershell
"ok" | Out-File -Encoding ascii C:\acme-webroot\probe.txt
# 从任意机器：
curl.exe http://121.196.144.88/.well-known/acme-challenge/probe.txt   # 期望输出 ok
```

### ①-2 用 win-acme 申请 ZeroSSL IP 证书（90 天，自动续期）

```powershell
# 1) 下载 win-acme（免安装，解压即用）
#    https://github.com/win-acme/win-acme/releases  → win-acme.v2.x.x.x64.pluggable.zip
#    解压到 C:\tools\win-acme\

# 2) 首次运行向导（图形/命令行都行），要点：
#    - 目标：Manual input → 填 121.196.144.88
#    - 验证：HTTP-01（File system 或 IIS；选 IIS 时会自动把挑战文件放进默认站点）
#    - ACME 服务器：ZeroSSL（需在 ZeroSSL 控制台生成 EAB：KID + HMAC Key，填进向导）
#    - 存储：PEM 文件 → C:\certs\lugwit\
#    - 安装后任务：建计划任务自动续期 + 续期后执行命令（见 ①-4）
```

> win-acme 各版本的菜单/参数名会变，按向导走最稳；命令行等价形式（示例，需按安装向导生成的
> `settings.json` 校对）：
> ```powershell
> C:\tools\win-acme\wacs.exe --target manual --host 121.196.144.88 `
>   --validation filesystem --webroot C:\acme-webroot `
>   --store pemfiles --pemfilespath C:\certs\lugwit --accepttos
> ```

### ①-3 证书文件确认

期望得到两个文件（路径按向导里填的）：
```
C:\certs\lugwit\fullchain.pem     # 证书链
C:\certs\lugwit\privkey.pem       # 私钥
```

### ①-4 启用 nginx 443（我来改 conf，你执行 reload）

`l_nginx/999.0/conf/lugwit.conf` 末尾已备好**注释版** 443 段，取其一：
- **方案 A**：把 8080 那段整体复制到 443（少一跳）
- **方案 B**：443 只做 TLS 终止 → `proxy_pass http://127.0.0.1:8080`（改动最小，推荐先用它）

要改的只有两行：
```nginx
ssl_certificate     C:/certs/lugwit/fullchain.pem;
ssl_certificate_key C:/certs/lugwit/privkey.pem;
```

执行：
```powershell
cd <l_nginx 包目录>\999.0
.\bin\nginx.exe -t          # 语法检查
.\bin\nginx.exe -s reload
```

### ①-5 放行 443 + 验收

```powershell
# 入站放行 443
New-NetFirewallRule -DisplayName "nginx 443" -Direction Inbound `
  -Protocol TCP -LocalPort 443 -Action Allow

# 网关侧（你本机）验收：
curl.exe -k -I https://121.196.144.88/                     # 期望 200/302
openssl s_client -connect 121.196.144.88:443 -servername 121.196.144.88 < NUL | openssl x509 -noout -subject -dates
curl.exe -k https://121.196.144.88/nginx-health            # 期望 ok
```

> 注意：IP 证书的 `subject` 是 `CN=121.196.144.88`；浏览器/App 里访问 `https://121.196.144.88/...`
> 不再告警。若客户端仍报错，多半是用了 `https://<域名>` 访问（用 IP 访问即可）。

### ①-6 续期后要重载 nginx

win-acme 建的计划任务里，把"安装后执行"设为：
```
<l_nginx 包目录>\999.0\bin\nginx.exe -s reload
```

---

## 路线③：免费域名 + DNS-01（不用碰 IIS、不用碰 80）

比 IP 证书省心，且 App 里可以用域名。

```powershell
# 1) DuckDNS 注册一个免费子域（如 lugwit.duckdns.org）→ 填服务器公网 IP
#    记下 token（用于 DNS-01 自动加 TXT 记录）

# 2) 用 lego（单文件 exe，Windows 免装）申请
lego.exe --server https://acme-v02.api.letsencrypt.org/directory `
  --email you@example.com `
  --dns duckdns `
  --domains lugwit.duckdns.org `
  --dns.resolvers 223.5.5.5 `
  --path C:\certs\lugwit `
  --accept-tos run
# DuckDNS 凭据：设置环境变量 DUCKDNS_TOKEN=<你的 token>

# 3) 得到 C:\certs\lugwit\certificates\lugwit.duckdns.org.pem 与 .key
#    或用 acme.sh（Git Bash）：
#    acme.sh --issue --dns dns_duckdns -d lugwit.duckdns.org
#    acme.sh --install-cert -d lugwit.duckdns.org --key-file C:/certs/lugwit/privkey.pem `
#            --fullchain-file C:/certs/lugwit/fullchain.pem --reloadcmd "nginx -s reload"
```

后续（443 启用、防火墙、验收）与 ①-4~①-6 相同，只是把 `server_name _;` 改成域名。

---

## 路线④：自签证书 + App 内置信任（**没有域名时的推荐路线**）

不需要域名、不碰 80/IIS、不用注册任何 CA 账号、**10 年不用续期**。
代价：只有"信任了这张证书的客户端"（我们的 App、装过 CA 的机器）不告警，浏览器裸访问会有警告。

### ④-1 生成证书（服务器上一条命令）

```powershell
# l_nginx 包的 requires 已加 cryptography，直接用它跑
wuwor l_nginx -- python <l_nginx包>\999.0\tools\make_self_signed_cert.py `
    --ip 121.196.144.88 --out C:/certs/lugwit
```
产物：`C:/certs/lugwit/{privkey.pem, fullchain.pem, ca.pem}`（CN=121.196.144.88，SAN 含该 IP，3650 天）

### ④-2 启用 443

```powershell
# 1) conf/lugwit.conf 里把 include https.conf; 这行的注释去掉
#    （https.conf 已按上面的证书路径写好，方案 B：443 → 127.0.0.1:8080）
cd <l_nginx包>\999.0
.\bin\nginx.exe -t           # 语法检查（证书不存在会在这里报错，先做 ④-1）
.\bin\nginx.exe -s reload

# 2) 放行 443
New-NetFirewallRule -DisplayName "nginx 443" -Direction Inbound `
  -Protocol TCP -LocalPort 443 -Action Allow
```

### ④-3 让 App 信任这张证书（否则 WebView 会拒绝）

1. 把 `C:/certs/lugwit/ca.pem` 拷到
   `l_WChat/999.0/src/l_WChat/wchat-android/android/app/src/main/res/raw/lugwit_ca.pem`
2. 新建 `android/app/src/main/res/xml/network_security_config.xml`：
   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <network-security-config>
     <base-config cleartextTrafficPermitted="false">
       <trust-anchors>
         <certificates src="system"/>
         <certificates src="@raw/lugwit_ca"/>   <!-- 自签 CA -->
       </trust-anchors>
     </base-config>
   </network-security-config>
   ```
3. `AndroidManifest.xml` 的 `<application>` 加：`android:networkSecurityConfig="@xml/network_security_config"`
4. `capacitor.config.json`：`server.url` 改 `https://121.196.144.88`，删掉 `"cleartext": true`
5. 重新打包发版（`npx cap sync android` → gradle assembleRelease）

> 开发机/浏览器要看页面：把 `ca.pem` 装进"受信任的根证书颁发机构"即可（或 curl 加 `-k`）。

### ④-4 验收

```powershell
curl.exe -k -I https://121.196.144.88/                 # 302 → /homepage
curl.exe -k https://121.196.144.88/nginx-health        # ok
# 用装了 CA 的机器（或 App）验证：无证书告警
```

---

## 附：三条路线对比（无域名场景）

| 路线 | 需要的账号/改动 | 有效期 | 客户端是否告警 |
|---|---|---|---|
| ① ZeroSSL IP 证书 | ZeroSSL 账号 + IIS 挑战目录 + win-acme | 90 天（自动续） | 否（全平台可信） |
| ② 临时停 IIS（standalone） | 每次续期停一次 IIS | 90 天 | 否 |
| **④ 自签 + App 内置信任** | **无外部账号**，只需生成 + 改 App | **3650 天** | 仅"未装 CA 的浏览器"告警 |

---

## 路线⑤：免费域名（DuckDNS）+ Let's Encrypt —— **已实施（2026-09-14）**

比自签更好：全平台天然可信、90 天自动续期、App 不需要内置 CA。

### 已做完的事（可复现步骤）

| 步骤 | 内容 | 结果 |
|---|---|---|
| 1. 注册免费域名 | DuckDNS（Google 登录）→ 建子域 | **`lugwit.duckdns.org`** |
| 2. 指向服务器 | `https://www.duckdns.org/update?domains=lugwit&token=<token>&ip=121.196.144.88` | 返回 `OK`；解析到 `121.196.144.88` |
| 3. 签证书（服务器） | Git Bash 里 clone acme.sh 完整仓库（要 `dnsapi/dns_duckdns.sh`），`export DuckDNS_Token=<token>`（**注意变量名是这个**），`--issue --dns dns_duckdns -d lugwit.duckdns.org --server letsencrypt --keylength ec-256` | 证书落到 `C:/certs/acme-home/lugwit.duckdns.org_ecc/` |
| 4. 安装证书 | `--install-cert --ecc --key-file C:/certs/lugwit_le/privkey.pem --fullchain-file C:/certs/lugwit_le/fullchain.pem --reloadcmd "<nginx.exe> -s reload -p <runtime> -c <conf>"` | acme.sh 记住路径与 reload 命令，**已验证 "Reload successful"** |
| 5. nginx 双块 | `conf/https.conf`：①`listen 443 ssl default_server` + 自签（IP 访问）②`server_name lugwit.duckdns.org` + LE 证书；公共部分抽到 `conf/https_common.conf` | `nginx -t` 通过，reload 成功 |
| 6. 自动续期 | `schtasks /Create /TN lugwit_acme_renew /SC DAILY /ST 03:30 /RU SYSTEM /RL HIGHEST /TR "<git bash> -lc \"sh /c/lugwit_ops/acme-repo/acme.sh --cron --home /c/certs/acme-home\""` | 下次运行 03:30；到期自动续 + reload nginx |
| 7. App | `capacitor.config.json`：`server.url = https://lugwit.duckdns.org` | 重新打包发版即可 |

### 关键路径与文件

```
C:/certs/acme-home/                 acme.sh 状态（账号、域名配置、下次续期=Ari 窗口）
C:/certs/acme-home/lugwit.duckdns.org_ecc/*.conf   ← 含 DuckDNS_Token 与续期参数（勿外传）
C:/certs/lugwit_le/{fullchain.pem,privkey.pem}     ← LE 证书（EC P-256）
C:/certs/lugwit/{fullchain.pem,privkey.pem,ca.pem} ← 自签（IP 访问 + App 内置 CA 用）
C:/lugwit_ops/acme-repo/            acme.sh 完整仓库（含 dnsapi）
l_nginx/999.0/conf/{https.conf,https_common.conf}  nginx 443 两块的配置
```

### 验证（已通过）

```powershell
# 服务器侧（SNI 正确性 + 路由）
$t=New-Object Net.Sockets.TcpClient('127.0.0.1',443);$s=New-Object Net.Security.SslStream($t.GetStream(),$false,({$true}))
$s.AuthenticateAsClient('lugwit.duckdns.org');(New-Object Security.Cryptography.X509Certificates.X509Certificate2($s.RemoteCertificate)).Subject
# → CN=lugwit.duckdns.org（issuer: CN=YE1, O=Let's Encrypt）

# 外网侧（浏览器，无告警）
https://lugwit.duckdns.org/homepage   → 302 → /homepage/login（登录页）
https://lugwit.duckdns.org/nginx-health → ok（Content-Type 未设，浏览器会当下载，见下）
```

### 小尾巴（可选）

- `/nginx-health` 没设 Content-Type → 浏览器当下载；如要显示文本，在 location 里加 `default_type text/plain;`
- 域名有了之后，`_gateway_base()`（`l_homepage/homepage_cli.py:1675`）里"直连后端"分支仍拼 `http://host:8080`，建议改成 scheme 感知（详见对话记录）

---

## 现在需要你做的

| 选择 | 你要做的 | 我接着做的 |
|---|---|---|
| **④（推荐，无域名）** | 在服务器跑 ④-1 生成证书 → ④-2 启用 443（注释掉 include 那行的注释）→ 把 `curl -k -I https://121.196.144.88/` 结果贴我 | 改 App 侧（network_security_config + capacitor.config）、出重新打包清单 |
| ①（全平台可信） | 跑 ①-1（IIS 虚拟目录）+ 贴回 `curl http://121.196.144.88/.well-known/acme-challenge/probe.txt` 结果 | 给你 win-acme 精确参数 + 443 启用步骤 |
| ②（临时停 IIS） | 说一声 | 写一键脚本（停 IIS → 签发 → 启 IIS → reload） |

