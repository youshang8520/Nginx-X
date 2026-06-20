# Nginx-X

一个基于 Bash 的 Nginx 自动化管理交互脚本（Ubuntu / Debian / CentOS）。

## 项目目标

通过数字菜单统一管理 Nginx，重点保证稳定性：
- 所有配置修改后都会先执行 `nginx -t`
- 校验通过才会 `reload`
- 校验失败会自动回滚，避免把服务改挂

## 安装方式

### 方式一：一键安装（推荐）

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/youshang8520/Nginx-X/main/install.sh)"
```

### 方式二：手动安装

```bash
git clone https://github.com/youshang8520/Nginx-X.git
cd Nginx-X
bash install.sh
```

安装说明：
- 一键安装首次会克隆到 `/opt/Nginx-X`
- 再次执行会自动拉取最新代码
- 如果 `/opt/Nginx-X` 已存在但不是 Git 仓库，安装器会先提示确认，再决定是否清空重装
- 若安装、拉取、依赖安装或软件源刷新失败，安装器会直接提示网络/软件源相关报错，不会静默中断

> **安装完成后可直接运行：`nx`**

## 本 Fork 与上游差异

本仓库基于 `Xiuyixx/Nginx-X`，保留上游功能的同时增加了一些兼容和部署增强，适合需要在 CentOS 7、老 Bash 或已安装过上游版本的服务器上使用。

主要差异：

- **安装源固定为本仓库**：一键安装会克隆 `https://github.com/youshang8520/Nginx-X.git`，避免继续拉取上游仓库。
- **已安装目录来源检查**：如果 `/opt/Nginx-X` 已存在但 `remote.origin.url` 不是本仓库，安装器会提示并自动切换到本仓库版本。
- **远程安装兼容**：修复 `curl | bash` 场景下误判当前目录为本地仓库的问题，支持 `FORCE_BOOTSTRAP=1` 强制走引导安装。
- **CentOS 7 / 老 Bash 兼容**：替换依赖 Bash 新特性的数组读取写法，避免老系统中 `mapfile`、nameref 等兼容问题。
- **Nginx 升级提示增强**：当无法访问或解析 nginx.org 最新版本时，提示更明确，并回退到包管理器升级检查。
- **ACME / IPv6 增强**：HTTP-01 临时验证入口在系统 IPv6 可用时会同时监听 `80` 和 `[::]:80`，降低有 AAAA 记录时证书验证失败的概率。
- **IPv6-only HTTPS 自动适配**：当用户选择 `443`，但 IPv4 `443` 已被其他服务占用时，脚本会自动检测 IPv6、AAAA 解析和证书条件，条件满足后只监听 `[::]:443 ipv6only=on`。
- **端口自动放行与回滚**：配置成功落地后会尝试自动开放 `80`、`443` 或用户输入的自定义 HTTPS 端口；CentOS 7/firewalld 环境优先使用 `systemctl` 启动 `firewalld`，再用 `firewall-cmd` 放行。脚本只记录并回滚自己新增的系统防火墙规则，不关闭用户原本已开放的端口；云厂商安全组仍需在服务商面板确认。
- **保留上游修复**：已融合上游导入配置失败自动回滚的修复，避免导入失败后留下半成品配置。

如果你想使用本 fork 的兼容版，请使用本文档里的 `youshang8520/Nginx-X` 安装命令，不要使用上游 `Xiuyixx/Nginx-X` 的 raw 安装地址。

## 当前功能

### 稳定性设计

- 所有配置变更都会先执行 `nginx -t`
- 校验失败自动回滚，避免把在线 Nginx 配挂
- 443 / HTTPS 端口复用场景会优先走更安全的落地流程
- 证书申请前会做 HTTP-01 自检；自检失败会取消申请并给出排查命令，避免继续触发 CA 验证失败
- 临时文件使用安全随机文件名，降低冲突和误覆盖风险

1. **安装升级Nginx**
   - 自动检查是否已安装 Nginx
   - 未安装时自动安装依赖：`curl` `wget` `socat` `cron`
   - 自动安装 Nginx 官方 stable 版本（默认使用 nginx.org 官方源，HTTPS）
   - 安装后自动停用 `default.conf`（改名为 `default.conf.bak`）避免与自定义配置冲突
   - 自动创建证书目录：`/etc/nginx/ssl/`
   - 对比本地版本与 Nginx 官网最新版本（若 nginx.org 不可达，会回退为包管理器升级检查）
   - 有新版本时先备份 `/etc/nginx/`，再执行升级
   - 升级后自动校验并平滑重载

2. **配置管理**
   - 二级菜单包含：`添加配置` / `外部反代` / `配置列表` / `导入已有配置`
   - 添加配置：输入域名或本机IPv4、监听端口、后端端口，支持端口复用确认
   - 外部反代：输入域名或本机IPv4、监听端口、外部 `http/https` 上游 URL
   - 外部反代支持模式选择：
     - `标准模式`
     - `Stream 模式`
     - `Emby 分离 HTTP 推流`
     - `Emby 分离 HTTPS 推流`
     - `LilyEmby 方案（访问/推流分离，支持 sub_filter 响应体替换）`
   - Emby/Lily 模式支持配置主站上游、一个或多个推流节点 URL（多个用英文逗号分隔）、源站公开 URL、Referer URL
   - 同端口多域名场景自动适配：若目标端口已用于 HTTPS，会自动处理为可通过 `nginx -t` 的流程
   - 自动生成标准 Proxy Header，配置写入 `/etc/nginx/conf.d/域名-监听端口.conf`
   - 配置列表中的“修改”支持外部反代配置切换方案，并保留证书/HTTPS 联动逻辑
   - 修改配置后自动检测证书状态并提示启用 HTTPS（流程与添加配置一致）
   - 若修改配置时更换域名且发现没有证书，可直接在流程中申请证书并启用 HTTPS
   - 添加完成后自动检测证书：
     - 若已有证书：仅需确认是否启用 HTTPS
     - 若无证书：可一键“自动申请证书 + 自动启用 HTTPS（80→目标 HTTPS 端口）”
   - 如果 `443` 已被 sing-box 等服务占用，可输入自定义 HTTPS 端口。脚本会临时使用 `80` 完成 HTTP-01 证书验证，证书成功后在自定义端口启用 HTTPS，并配置 `80` 跳转到该 HTTPS 端口。
   - 配置列表：统一管理上述两类配置，按状态浏览并进入三级菜单执行 启用 / 停用 / 修改 / 编辑 / 删除
   - 导入已有配置：自动扫描 `conf.d/`、`sites-enabled/`、`sites-available/` 三个目录，发现未纳管的 Nginx 配置后逐个确认导入
   - 导入时自动提取域名、端口、后端地址、HTTPS 状态等元数据，不修改原有 Nginx 指令
   - `sites-available` 的配置会复制到 `conf.d/`，并自动移除 `sites-enabled` 中对应的软链接（避免重复加载）
   - 导入成功后会询问是否删除原始配置文件（推荐删除，避免重复扫描）
   - 已导入到 `conf.d` 的配置不会被重复扫描（按域名+端口去重，含 `.bak` 等停用文件）
   - 首次安装 Nginx 后会自动检测并提示导入已有配置

### IPv6 支持说明

- **对外监听（客户端通过 IPv6 访问）**：脚本在生成配置时会进行最佳努力检测，若系统 IPv6 已启用，会额外写入：
  - `listen [::]:80;`
  - `listen [::]:443 ssl;`（或对应的 HTTPS 监听端口）
- **ACME HTTP-01 验证**：临时验证入口在 IPv6 可用时也会写入 `listen [::]:80;`，适配有 AAAA 记录的域名。
- **IPv6-only HTTPS**：如果 IPv4 `443` 已被其他服务占用，而 IPv6 `443` 可用，脚本会自动引导使用 IPv6-only HTTPS，不需要用户手动编辑 Nginx 配置。
- **上游反代到 IPv6**：支持上游 URL 使用 IPv6（需要按 URL 规范使用方括号），例如：
  - `http://[2001:db8::1]:8080`
  - `https://[2400:3200::1]`

> 注意：若你的系统内核禁用了 IPv6（例如 `net.ipv6.conf.all.disable_ipv6=1`），脚本会自动跳过 IPv6 的 `listen` 行，避免 `nginx -t` 失败。

### IPv6 用户使用步骤

如果服务器本身支持 IPv6，建议按下面顺序确认：

1. 确认系统 IPv6 已启用：

   ```bash
   sysctl net.ipv6.conf.all.disable_ipv6
   ip -6 addr
   ```

   `net.ipv6.conf.all.disable_ipv6 = 0` 表示系统未禁用 IPv6；`ip -6 addr` 应能看到公网 IPv6 地址。

2. 在 DNS 服务商处添加记录：

   - `A` 记录：域名指向服务器 IPv4
   - `AAAA` 记录：域名指向服务器 IPv6

   如果只想先测试 IPv4 证书签发，可以暂时不加 AAAA；如果加了 AAAA，就必须确保 IPv6 的 80/443 也能公网访问。

3. 确认 80/443 或自定义 HTTPS 端口可访问：

   脚本会在配置成功落地时自动放行系统防火墙端口（firewalld/ufw），包括 `80`、`443` 或你输入的自定义 HTTPS 端口。云厂商安全组不在服务器系统内，仍需要到服务商面板确认已放行。

   CentOS 7/firewalld 可用下面命令手动确认或排查：

   ```bash
   firewall-cmd --permanent --add-service=http
   firewall-cmd --permanent --add-service=https
   HTTPS_PORT=你的HTTPS端口
   firewall-cmd --permanent --add-port=${HTTPS_PORT}/tcp
   firewall-cmd --reload
   firewall-cmd --list-all
   ```

4. 安装或更新本 fork：

   ```bash
   bash -c "$(curl -fsSL https://raw.githubusercontent.com/youshang8520/Nginx-X/main/install.sh)"
   ```

5. 运行脚本并添加配置：

   ```bash
   nx
   ```

   在“配置管理”里选择“添加配置”或“外部反代”。脚本检测到 IPv6 可用时，会自动生成 IPv6 监听行。

   如果你选择监听 `443`，但 IPv4 的 `443` 已经被其他服务占用，脚本会自动尝试 IPv6-only HTTPS：

   - 检测系统 IPv6 是否可用
   - 检测 IPv6 `443` 是否空闲
   - 如果有多个公网 IPv6，列出地址并让用户选择，默认使用第一个
   - 检查域名 AAAA 是否指向选中的 IPv6
   - 如果未解析，会显示需要添加的 IPv6 地址，用户添加后按回车重新检测
   - 证书申请成功、Nginx 校验通过后才落地最终配置
   - 任一步失败或用户取消，都不会写入半成品站点配置

   如果 `443` 的 IPv4 和 IPv6 都被占用，可以直接输入自定义 HTTPS 端口。这不会影响证书签发，因为 HTTP-01 证书验证只要求域名的 `80` 端口可达；最终 HTTPS 会监听你输入的端口。

6. 申请证书前做外部连通性检查：

   ```bash
   curl -4 -I http://你的域名/
   curl -6 -I http://你的域名/
   ```

   如果 `curl -6` 失败，而域名已经有 AAAA 记录，Let's Encrypt 可能会通过 IPv6 访问失败，导致 HTTP-01 验证失败。此时应先修复 IPv6 防火墙/安全组，或者临时删除 AAAA 记录后再申请证书。

7. 申请证书后再检查 HTTPS：

   ```bash
   curl -4 -I https://你的域名/
   curl -6 -I https://你的域名/
   HTTPS_PORT=你的HTTPS端口
   curl -4 -I https://你的域名:${HTTPS_PORT}/
   curl -6 -I https://你的域名:${HTTPS_PORT}/
   ```

端口和证书排查命令：

```bash
DOMAIN=你的域名
HTTPS_PORT=你的HTTPS端口
ss -lntp '( sport = :80 )'
ss -lntp6 '( sport = :80 )'
ss -lntp '( sport = :443 )'
ss -lntp6 '( sport = :443 )'
ss -lntp "( sport = :${HTTPS_PORT} )"
ss -lntp6 "( sport = :${HTTPS_PORT} )"
grep -R -n "listen .*80\|server_name ${DOMAIN}" /etc/nginx/
nginx -t
systemctl status nginx --no-pager
firewall-cmd --get-active-zones
firewall-cmd --list-all
curl -4 -I "http://${DOMAIN}/"
curl -6 -I "http://${DOMAIN}/"
curl -4 -I "https://${DOMAIN}:${HTTPS_PORT}/"
curl -6 -I "https://${DOMAIN}:${HTTPS_PORT}/"
```

常见问题：

- 有 AAAA 记录但 IPv6 80 端口不通：证书申请可能失败。
- Nginx 配置里没有 `[::]` 监听：通常是系统禁用了 IPv6，检查 `sysctl net.ipv6.conf.all.disable_ipv6`。
- 后端是 IPv6 地址：上游 URL 必须使用方括号，例如 `http://[2400:3200::1]:8096`。
- 使用 CDN 时：确认 CDN 的 IPv6 回源、HTTP-01 路径和 80 端口策略没有拦截。
- 如果服务器有多个 IPv6：脚本会列出地址，默认选第一个，也可以输入编号指定某个 IPv6；域名 AAAA 必须解析到选中的地址。
- 如果系统使用 firewalld：脚本会优先用 `systemctl enable --now firewalld` 启动服务，再用 `firewall-cmd --permanent --add-service=http/https` 或 `--add-port=端口/tcp` 放行。

3. **证书管理**
   - 设置邮箱（持久化到 `${XDG_CONFIG_HOME:-$HOME/.config}/nginxx/email.conf`）
   - 自动安装 acme.sh 并申请证书（HTTP 验证）
   - 申请前自动执行 HTTP-01 自检（DNS/80监听/challenge 路径/域名回源），失败时给出更明确提示
   - 证书列表：按编号展示已有证书，并显示续期任务状态
   - 证书操作：支持按编号执行 重新申请 / 启停续期 / 删除证书
   - 启用证书时先从配置列表选择目标配置
   - 若配置已启用 HTTPS，再次选择该配置可按确认执行“停用 HTTPS”
   - 自动检查是否已有证书：有证书直接启用；无证书先申请再启用
   - 一键启用 HTTPS（自动继承原监听端口；80 自动跳转到对应 HTTPS 端口）

4. **实时信息**
   - 二级菜单包含：`实时信息` / `流量统计` / `健康检查`
   - 实时信息：展示连接状态、请求统计、QPS、系统资源、Nginx 信息、网络流量
   - 流量统计：展示系统总流量与当前启用配置的流量估算
   - 若存在 `/var/log/nginx/access.host.log`，会优先按 Host 专用日志做更精确统计
   - 若不存在 Host 专用日志，则回退为基于最近 5000 条 `access.log` 的估算
   - 健康检查支持 `检查所有站点` / `检查单个站点`
   - 健康检查内容包括：入口 URL、HTTP 状态码、DNS 解析结果、命中 IP
   - HTTPS 站点会额外显示证书剩余天数（可获取时）
   - 外部反代会额外显示主上游与推流上游（支持多个推流上游），便于排查
   - 每 5 秒自动刷新，按回车返回上一级

5. **卸载**
   - 选项1：卸载脚本（彻底卸载本脚本并清理）
   - 选项2：卸载 Nginx（彻底卸载并清空 Nginx 及配置）
   - 选项3：卸载 Acme（彻底卸载并清空 Acme 配置/邮箱信息）
   - 选项4：全部卸载（脚本 + Nginx + Acme 一并清理）
   - 高风险卸载操作会显示删除摘要并要求二次确认

## 已知限制

- 证书签发当前只覆盖 `HTTP-01` 场景，不支持 DNS API 自动签发通配符证书。
- 若服务器前面有 CDN、四层转发、NAT 回环限制，HTTP-01 自检可能出现“软失败但实际可签发”或“本机通过但公网不通”的差异，需要结合实际网络环境判断。
- 流量统计默认是轻量实现。若未配置 `/var/log/nginx/access.host.log`，统计结果属于估算值，不适合作为精准计费依据。
- 当前没有完整的端到端自动化测试，仓库内提供的是基础语法检查、ShellCheck，以及 HTTPS 配置回归脚本。

## 交互与失败处理说明

- 菜单类入口在缺少前置条件时，会提示原因并返回上一级，不会因为 `set -e` 直接退出整个脚本。
- 安装 / 升级链路里若遇到网络不通、GitHub 不可达、软件源异常、签名密钥下载失败等情况，会给出更明确的错误提示。
- 证书邮箱配置保存在用户配置目录，而不是脚本安装目录，避免因 `/usr/local/bin` 不可写导致保存失败。

## 建议的 Host 专用日志格式

如果你希望“流量统计”更准确，可在 Nginx 主配置里增加类似：

```nginx
log_format nginxx_host '$host $body_bytes_sent $remote_addr [$time_local] '
                      '"$request" $status $http_referer "$http_user_agent"';
access_log /var/log/nginx/access.host.log nginxx_host;
```

这样脚本会优先读取 `access.host.log`，按域名做更准确的请求数和下行统计。

## 开发校验

本仓库已包含 GitHub Actions 基础 CI：

- `bash -n nx.sh`
- `bash -n install.sh`
- `shellcheck -x nx.sh install.sh`
- `bash tests/https_config_regression.sh`

本地也可以直接运行：

```bash
bash -n nx.sh
bash -n install.sh
shellcheck -x nx.sh install.sh tests/https_config_regression.sh
bash tests/https_config_regression.sh
```

## 交互规范

- 主菜单使用数字编号
- `0` 表示退出或返回上一级
- 颜色反馈：
  - 绿色：成功
  - 黄色：警告
  - 红色：错误
