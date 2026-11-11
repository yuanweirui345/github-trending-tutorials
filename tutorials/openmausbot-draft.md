# OpenMausBot 教程草稿（候选 6 号 · 差异化版）

> 定位声明：本教程**不做基础版**。基础中文教程已被中文社区抢跑（已有全量中文翻译仓 openmausbot-cn、免费「小白中文教程」仓 awesome-openmausbot-zh）。
> 本教程只卖一个差异化角度：**「VPS 24 小时部署实战 + 避坑」**——把 OpenMausBot 部署到一台 VPS 上全天候运行，关掉自己的电脑，机器人继续干活，手机随时配对指挥。
> 定价：9.9 元（2026-11-11 用户拍板，全线教程统一价；知识星球 / 小报童）。项目：milind-soni/OpenMausBot，https://github.com/milind-soni/OpenMausBot ，20,335 Star（2026-11-10 核实），Apache-2.0（`enterprise/` 目录除外，为 source-available 单独协议）。教程卖的是实测部署经验，不出售项目本身。

---

## 一、实测日志（2026-11-10，沙箱环境：Ubuntu 24.04 / Node v24.22.1 / npx 10.9.4）

### 跑通的部分 ✅
1. **npm 包可用**：`npx --yes openmausbot --help` 正常输出全部命令（当前版本 MausBot 0.1.151），`serve / pair / devices / status / login / service install / browser install` 等子命令齐全。
2. **服务端本体真实启动**：`npx openmausbot serve --data-dir <dir>` 成功拉起：
   - 主服务监听 `127.0.0.1:8799`，Webhook 接收器监听 `127.0.0.1:8800`；
   - `curl http://127.0.0.1:8799/api/health` 返回 `{"app":"openmausbot", ...}`，Web UI 首页 HTTP 200；
   - 数据目录自动生成（`~/.openmausbot` 等价物：bots.json、messages.db、runtime.db 等，配置文件权限 0600）。
3. **配对码真实生成**：服务运行时 `npx openmausbot pair` 成功生成一次性配对码（形如 `F4X2-T34P-ZJ9Z`，10 分钟有效、单次使用），与官方文档描述一致。
4. **Docker 路线做了逐文件静态核验**（见下）：`deploy/docker-compose.yml`、`deploy/Caddyfile`、`deploy/.env.example`、根目录 `compose.yaml` 与官方 `docs/deploy-vps.md` 路径 B、`docs/self-hosting.md` 完全对得上。

### 没跑通 / 未验证的部分 ❌（如实记录）
1. **Docker 路线未做运行时验证**：本沙箱**没有 Docker**（`docker: command not found`，apt 源里也没有 docker.io 包，无法安装），所以 `docker compose pull omb && docker compose up -d` 没有真实执行。Docker 部分结论仅来自官方文档 + 仓库 compose/Caddyfile 源码核验。
   - 附带发现：连官方自己的 Hetzner 实测记录（docs/deploy-vps.md，2026-09-07）都注明 **Docker 路径不在那次实测范围内**——市面教程大概率也没真跑过 Docker 路线，这正是本教程后续补实测的价值点（需一台真 VPS 补测）。
2. **全量克隆失败**：`git clone` 仓库（约 333 MB）在 512 MB 的 /tmp 里报 `No space left on device`；改用 sparse-checkout 只拉 `deploy/` + `docs/` 才成功。→ 写进坑清单：别在小磁盘/小内存机上全量拉仓库，VPS 路线根本不需要克隆源码（deploy 目录或直接用预构建镜像即可）。
3. **没有可用的引擎订阅，端到端对话未验证**：沙箱里没有已登录的 Claude / Codex / Grok 账号凭据，机器人建出来也无法真正回复。setup 的引擎登录环节、手机 App 真机配对、Caddy 在真实域名上签发 Let's Encrypt 证书，这三项都需要在真实 VPS + 真实订阅账号下补测后才能写成「已验证」步骤，草稿中已标注 ⚠️待真机补测。

### 实测中真实触发的现象（直接变成教程素材）
- 不带任何「入口」参数直接 `serve`：服务能起，但**不生成配对码**，只打印五种入口让你选——新手最容易在这里以为「装好了」，其实任何设备都连不进来。
- `serve --lan` 在没有私有网段地址的机器上直接报错：`--lan: this machine has no private network address, so --lan would reach nobody.` → **纯公网 VPS 上 --lan 是死路**，必须用 --domain / --public-url / --tailscale / --tunnel 之一。
- 浏览器引擎版本警告：沙箱里预装的 agent-browser 0.42.0 与项目钉死的 0.37.0 不一致，启动即警告「版本不对可能让浏览器工具卡死」。→ 版本钉死是这类项目的通病，教程要教 `openmausbot browser install` 用官方钉死版本，不要自己装最新版。

---

## 二、教程大纲（VPS 24 小时实战版）

**标题（草案）**：《关掉电脑，AI 继续打工：OpenMausBot VPS 24 小时部署实战（含 12 个坑）》

### 第 0 章 先说清门槛：没有订阅，这教程对你没用（劝退章，反而建立信任）
- OpenMausBot 本体免费开源，但它只是「壳」：每个机器人背后必须跑一个引擎 CLI——`claude`（Claude Code）、`codex`（Codex）或 `grok`，用的是**你自己的订阅登录态**，没有第二个选择。
- 所以前置条件三选一，必须先有：Claude 订阅（Pro/Max，能登录 Claude Code）、ChatGPT 订阅（能登录 Codex）、或 Grok 订阅；或退而求其次用 API Key 按量付费（官方 setup 支持粘 API Key，但费用自担，setup 在发测试消息前会先征求确认）。
- 国内用户额外门槛：上述订阅的注册、付费和登录环节本身就有网络与支付门槛。**先在自己电脑上把 claude/codex 登录成功，再买 VPS、再往下看**，顺序反了全是坑。
- 成本清单（教程里做成表格）：VPS（官方口径 2 核 4G 足够）+ 引擎订阅 + 可选域名；机器人的云电脑（Boat）是第三方付费服务（试用后收费），Composio（连接 Gmail/Slack/GitHub 等应用）是第三方服务，各自有账号和条款。

### 第 1 章 选路：VPS 三条路线对比，只推荐一条给新手
| 路线 | 命令 | 需要 | 适合 |
|---|---|---|---|
| A 隧道（最快） | `npx openmausbot login` + `serve --tunnel` | OpenMausBot 账号（邮箱验证码） | 没域名、想今天就跑起来 |
| B 自有域名 Docker+Caddy（本教程主线） | deploy/ 的 docker compose | 域名 + 80/443 端口 | 想要长期固定地址、24 小时常驻 |
| C Tailscale | `serve --tailscale` | Tailscale 账号 | 最私密，公网完全不暴露 |
- 为什么主线选 B：地址是你自己的、compose 自带 `restart: unless-stopped` 重启自恢复、Caddy 自动签/续证书，一次搭好不用再管。

### 第 2 章 VPS 开荒（路径 B 全步骤）⚠️待真机补测后定稿
1. 买 VPS：Ubuntu 22.04/24.04，2 核 4G 起步；安全组放行 80/443（SSH 端口建议改非默认 + 密钥登录）。
2. DNS：给域名加 A 记录（如 `maus.example.com` → VPS 公网 IP），等解析生效（`ping` 验证）再往下，证书签发失败十有八九是这步没等。
3. 装 Docker：`curl -fsSL https://get.docker.com | sh`，验证 `docker compose version`（无输出就退出重登一次）。
4. 拿部署文件：`git clone https://github.com/milind-soni/OpenMausBot && cd OpenMausBot/deploy`（只用 deploy 目录；磁盘小就 sparse-checkout，别全量拉——见坑 9）。
5. 配 `.env`：`cp .env.example .env`，改两项：
   - `DOMAIN=maus.example.com`
   - `ENGINES=@anthropic-ai/claude-code @openai/codex`（空格分隔，写哪个引擎，镜像里就烤哪个；以后加引擎要改这里重建镜像）
6. 启动：`docker compose pull omb && docker compose up -d`，`docker compose ps` 看 omb 变 `healthy`、caddy `running`；Caddy 自动申请证书，给它一分钟。
7. 验证：浏览器打开 `https://你的域名`，看到「要求输入配对码」的页面就是对的——没配对之前什么都用不了，这是设计如此，不是装坏了。

### 第 3 章 引擎登录：在服务器上再登一次（最容易卡的一章）
- 原理先讲清：引擎登录态存在**服务运行账号的家目录**里（`~/.claude`、`~/.codex`），和 OpenMausBot 自己的 `~/.openmausbot` 是两套东西；Docker 路线里家目录是数据卷 `/data`，所以重启/升级不丢登录，但**换卷 = 全部重登**。
- Docker 路线：`docker compose exec omb claude` 在容器里登录 Claude；Codex 在无头服务器上用 `codex login --device-auth` 设备码方式，在本地浏览器完成授权。
- npx 路线（路径 A/C）：先建专用普通用户（如 `maus`），setup、serve、引擎登录**全程同一个用户**，千万别用 root 登引擎又用普通用户跑服务——凭据在不同家目录里，机器人会「失忆」。
- 浏览器引擎：Docker 镜像自带；npx 路线要两步（管理员装系统库 `sudo -H npx openmausbot browser install --with-deps` + 服务用户再装一遍 `npx openmausbot browser install`），只做一步必然用不了。

### 第 4 章 Caddy 反代与域名深挖（差异化核心章之一）
- 架构一句话：omb 服务**只监听 127.0.0.1:8799，永不直接对外**；Caddy 和它共享网络命名空间（compose 里 `network_mode: service:omb`），对外 80/443 由 omb 服务占端口、Caddy 终止 TLS。
- Caddyfile 逐段讲解（仓库 `deploy/Caddyfile` 已核验）：
  - `{$DOMAIN}` 站点块 + `encode zstd gzip`；
  - `/hooks/*` 单独转发到 Webhook 接收器 8800（它靠每个 hook 自己的密钥鉴权，所以放在登录之外）；
  - 其余流量转发 8799，并设 `flush_interval -1`——**这是 SSE 流式回复不卡顿的关键**，自己写反代（nginx/cloudflared 走 --public-url）时必须等价处理：保留原始 Host、设 `X-Forwarded-Proto`、关掉对事件流的缓冲，否则机器人回复会「憋着不吐字」。
  - 可选第二道墙：Caddyfile 里预留了 `basic_auth` 注释块，给配对页再加一层共享口令（口令哈希用 `caddy hash-password` 生成，写进 .env 时每个 `$` 要双写转义——这是个真实小坑，教程里单独标红）。
- `--domain` 与 Docker 路线的区别：npx 的 `--domain` 是程序自己下载钉死版 Caddy 当子进程跑，低权限用户绑 80/443 失败时它会打印确切的 `setcap` 命令，照抄执行一次即可，别去 chmod 瞎改。
- Reach 自检：`serve` 启动会回访你的公网地址验证「答话的是不是这台服务器」，报 `doesn't reach this server` 就是反代指错端口（应指向 `http://127.0.0.1:8799`，Webhook 指 8800），用 `openmausbot status` 复查。

### 第 5 章 手机配对：从配对码到 24 小时指挥（差异化核心章之二）
- 生成配对：终端前台运行时 serve 启动即打印配对块（含二维码）；服务/容器方式运行**日志里永远不会有配对码**（防泄漏设计），必须手动执行：Docker 路线 `docker compose exec omb openmausbot pair`，npx 路线 `npx openmausbot pair`。
- 配对规则（全部官方核实，写成清单）：
  - 配对码 12 字符、**10 分钟有效、只能用一次**，服务器重启后所有未用配对码作废；
  - 同一地址 5 分钟输错 5 次，锁定 5 分钟；
  - 手机专用码：`pair --phone` 生成的码只能聊天、审批、阅读，**不能建机器人、改模型、连应用**——给手机发这种码，别发全权码；
  - `--ttl 24h` 可把有效期放宽到一天（仍单次），适合「白天生成、晚上回家再配」；
  - 会话有效期 30 天，活跃使用自动续期，最长 180 天；无密码体系，配对码就是钥匙，**截图别乱发**。
- 手机端操作：iOS App 扫码或粘贴链接；Android 扫码（链接是 App Link）；也可以直接用手机浏览器打开配对链接，并把 Web UI「添加到主屏幕」当 App 用。桌面端：桌面 App 里 **＋ → Connect to a server** 粘贴链接。
-  localhost 大坑：手机**永远连不上** `127.0.0.1/localhost` 链接——那指的是手机自己。配对链接必须是通过第 1 章选的入口生成的公网/Tailscale 地址。

### 第 6 章 让它 24 小时活着：常驻、重启与升级
- Docker 路线：compose 已有 `restart: unless-stopped`，VPS 重启后自动恢复，最省心。
- npx 路线：`openmausbot service install --tunnel`（或 `--tailscale` / `--public-url`）生成 systemd 单元；手写 systemd 的要点（官方示例已核验）：`User=` 必须是引擎登录的那个用户、`WorkingDirectory` 和 `HOME` 指向其家目录、`PATH` 带上 `~/.local/bin`、`Restart=always`。
- 升级：Docker 路线重新 `pull` + `up -d`（改 ENGINES 要 `--build`）；服务路线要先停服务、用服务用户装指定版本、再启动，**不要让它自动追最新**——这个项目 0.1.x 几乎每天发版（已 108 个 release），钉版本、看 changelog 再升。
- 备份：停服后备份数据目录（Docker 的 data 卷 / `~/.openmausbot` + 引擎家目录），官方实测过备份恢复后配对设备、机器人、对话都在（但服务商重新认证未在其测试范围内，教程如实标注）。

### 第 7 章 安全章（差异化核心章之三）
见下方第三节「安全风险与防护」全文，教程中独立成章，不许跳过。

### 第 8 章 常见报错速查表
直接用官方 Troubleshooting 表 + 本次实测现象整理成中文速查（Reach 报错、证书未就绪、配对码输错类型、地址填成配对码、http 公网未加密被拒等，见坑清单）。

---

## 三、安全风险与防护建议（教程第 7 章全文要点）

**先把话说透：OpenMausBot 里的机器人不是聊天玩具，它跑的是你本人的 Claude/Codex CLI，用你的账号权限执行命令、改文件、操作浏览器和电脑。把它放到 24 小时 VPS 上，等于雇了一个全天候拿着你钥匙的人——防护不是选修课。**

1. **⚠️ 头号大坑：默认就是 Full access（完全访问）**。官方 `docs/approval-levels.md` 明说：个人版每个机器人和每个对话**默认从 Full access 开始**——命令、文件修改、选定电脑的操作自动放行，破坏性/敏感操作也在内；通过已连接应用发邮件/消息时，若没单独设置，默认「不问直接发、无每日上限」。**部署完第一件事：把每个机器人改成 Ask for approval（事事问我）**，真信任某个固定任务了，再单独给它放级。
2. **回环信任模型**：服务器只听 127.0.0.1，本机上的任何程序都被当作主人。所以：永远不要自己把 8799 端口裸暴露到公网（别做路由器端口转发、别改绑 0.0.0.0 图省事）；对外只能走官方五种入口之一，它们全都要求配对。`--lan` 的明文 http 链接桌面端会直接拒绝公网地址，这道防线别绕。
3. **共享服务器更危险**：多人共用一台机器/一个服务时，本机回环调用方权限更大（官方 SECURITY.md 明确列了若干已知未完全关闭的越权面，例如 Full-access 线程里机器人 shell 可能绕过审批卡发消息）。结论：**VPS 上只跑你自己的机器人，别和不信任的人拼车共用一个实例**；要给别人用，用 `access add` 给邮箱登录并区分管理员/成员（成员只能聊天和审批），或用 fleet 做隔离租户。
4. **凭据是明文存的**：API Key 以明文保存在 `config.json`（靠文件权限 0600 保护，README 原话 “saved as plaintext, not encrypted”）；引擎登录态在服务用户家目录；隧道账号在 `~/.openmausbot/tunnel-account.json`（0600）。防护：服务用户专号专用、SSH 密钥登录并禁密码、数据卷/备份当密码本保管、别把家目录打包发人排障。
5. **配对码 = 钥匙**：10 分钟单次有效是好事，但 `--ttl 24h` 的长效码和配对链接截图一旦外泄，等于把门钥匙发出去。用 `openmausbot devices` 定期盘点已配对设备，不认识的立刻 `devices revoke <ID>`（立即下线并断流）。
6. **电脑操作能力分级开**：机器人操作宿主桌面只在 macOS / Ubuntu Xorg 且必须显式 opt-in，Wayland 被官方安全门直接禁用——别为了能用去关安全门或换回 Xorg 还不自知。VPS 场景优先用它的云电脑/容器电脑（隔离环境），宿主控制留到最后。
7. **连接应用（Composio）最小授权**：Gmail、Slack、GitHub 等一次 OAuth 后所有机器人共享。只连真正要用的应用；配合第 1 条，把「代表你发送」设为每次必问（Sending on your behalf → Ask every time）。
8. **Webhook 表面**：8800 接收器靠每个 hook 的密钥鉴权，官方建议用 Bearer 请求头传密钥（别放 URL 里进日志）；公网只经反代放行 `/hooks/*`，不要把整个 8800 裸露出去。
9. **供应链与版本**：只从官方 npm 包 / 官方 GHCR 镜像安装；项目方反复声明**没有任何加密货币代币**，任何打着 Maus 名号的币都是假冒——顺手写进教程防读者被骗。

---

## 四、坑清单（现象 → 原因 → 解法）

1. **没订阅，机器人是空壳** → 引擎 CLI 没登录/没订阅时机器人无法回复 → 先在本机登录成功 claude/codex/grok 之一，再部署；API Key 路线要预算按量费用。
2. **serve 起来了却没有任何配对码** → 没选入口时官方设计就不发码 → 按第 1 章五选一重跑 serve。
3. **--lan 在 VPS 上报错 reach nobody** → 纯公网机器没有私有地址（实测复现）→ 改用 --domain / --public-url / --tailscale / --tunnel。
4. **手机打不开配对链接 / 链接是 localhost** → localhost 指手机自己 → 必须用入口生成的公网/Tailscale 链接重新 `pair`。
5. **服务方式运行，日志里找不到配对码** → 官方故意不让码进日志 → 在终端执行 `pair`（Docker：`docker compose exec omb openmausbot pair`）。
6. **反代后机器人回复不流式、整段蹦出来** → 代理缓冲了 SSE → Caddy 设 `flush_interval -1`；nginx 关 proxy_buffering 并保留 Host / X-Forwarded-Proto；上游必须指 127.0.0.1:8799（hooks 指 8800）。
7. **Reach 校验报 doesn't reach this server** → 隧道/反代指错端口或指到另一台服务器 → 改指 8799 后 `openmausbot status` 复查。
8. **域名打不开 / 证书报错** → DNS 没生效或 80/443 没放行 → 等 A 记录生效、查安全组，Caddy 证书随后自动签发（约一分钟）。
9. **git clone 报 No space left on device** → 仓库约 333 MB，/tmp/小盘装不下（实测复现）→ 只拉 deploy 目录或 sparse-checkout；VPS 路线优先用预构建镜像 `pull`，不必 build 源码。
10. **浏览器工具卡死/不可用** → agent-browser 版本与项目钉死版本不一致（实测触发 0.42 vs 0.37 警告）；或只做了两步装浏览器中的一步 → 用 `openmausbot browser install` 按官方两步走（管理员装系统依赖 + 服务用户装浏览器本体）。
11. **重启后机器人「不认识」引擎了** → setup/登录/运行用了不同用户（root vs 服务用户），凭据在家目录对不上；Docker 换了数据卷同理 → 全程同一用户、固定数据卷。
12. **机器人擅自执行了命令/发了消息** → 默认 Full access（见安全章第 1 条）→ 立刻改 Ask for approval，并把「代表发送」设为每次必问。
13. **Windows 安装被 SmartScreen 拦**（桌面版梳理）→ 安装包未做代码签名 → 「更多信息 → 仍要运行」；同时提醒只从官网/GitHub Releases 下载。
14. **Ubuntu 上机器人不能控制桌面** → Wayland 安全门未解（官方 issue #345），不是装坏了 → 用 Xorg 会话（且明确风险）或改用云电脑/本地 VM，别找偏方关安全门。
15. **升级后行为突变** → 0.1.x 几乎日更 → 钉版本升级、先看 changelog、升前备份数据目录。

---

## 五、卖点文案草案（定价 9.9 元）

**主标题**：关掉电脑，AI 继续打工——OpenMausBot VPS 24 小时部署实战
**副标题**：市面上的中文教程都只教你在自己电脑上装着玩。这篇只讲一件他们没讲的事：把它搬上 VPS 全天候运行。
**卖点条目**：
- ✅ 三条 VPS 路线对比与选型（隧道 / 自有域名 Docker+Caddy / Tailscale），照着一条走到底，不用拼凑十篇博客
- ✅ Caddy 反代逐行拆解：证书自动签发、SSE 不缓冲、Webhook 单独放行——机器人回复卡顿的真凶在这
- ✅ 手机配对全规则：一次性配对码、手机专用受限码、设备盘点与一键踢下线，出门在外安全指挥
- ✅ 12+ 个实测坑：没订阅是空壳、--lan 在 VPS 上是死路、默认 Full access 有多危险……每个坑都有现象和解法
- ✅ 安全章单独成篇：默认完全访问怎么关、凭据明文存哪、为什么绝不能把 8799 裸暴露公网
- ✅ 常驻与升级：systemd / Docker 重启自恢复、钉版本升级、备份恢复清单
**适合谁**：已有 Claude Code / Codex 订阅、想让 AI 助手全天候在线的进阶玩家。
**不适合谁**（明写，降退款）：还没有任何 AI 订阅的小白——先去把订阅搞定，本教程不解决订阅注册与付费问题。
**定价**：统一 9.9 元，全内容不分档。教程承诺：后续 Docker 真机补测、版本大更新时免费同步修订版。

---

## 六、需截图清单（定稿前补拍；标注实测状态）

1. VPS 安全组放行 80/443 的设置页 ⚠️待真机
2. DNS A 记录添加与 `ping` 生效验证 ⚠️待真机
3. `.env` 修改前后对照（DOMAIN、ENGINES 两行高亮）— 可用本地文件截图 ✅可拍
4. `docker compose ps` 显示 omb healthy / caddy running ⚠️待真机（沙箱无 Docker）
5. Caddy 签发证书后浏览器打开域名出现配对页 ⚠️待真机
6. `docker compose exec omb openmausbot pair` 输出的配对块（码要打码）⚠️待真机；npx 版 pair 输出 ✅已实测可拍同类图
7. serve 不带入口时的五入口提示（新手困惑点）✅已实测可拍
8. --lan 报错原文特写 ✅已实测可拍
9. 浏览器引擎版本不一致警告特写 ✅已实测可拍
10. 桌面 App「＋ → Connect to a server」粘贴链接的界面 ⚠️待真机
11. 手机扫码配对全过程（iOS 扫码页 → 配对成功 → 聊天界面）⚠️待真机
12. 手机专用码 `pair --phone` 生成命令与手机端权限受限提示 ⚠️待真机
13. 机器人设置里把 Full access 改成 Ask for approval 的前后对照（安全章配图）⚠️待真机
14. 审批卡在手机上弹出、点 Allow/Deny 的样子 ⚠️待真机
15. `openmausbot devices` 设备列表与 revoke 操作 ⚠️可半实测（沙箱已生成本机配对）
16. systemd 服务状态 `systemctl status` + 重启 VPS 后机器人自动恢复并发消息 ⚠️待真机
17. /api/health 返回 JSON 的终端截图 ✅已实测可拍

---

## 七、与其他路线的文档梳理（只梳理，未实测）

- **桌面版**：macOS（Apple 芯片/Intel，签名公证的 .dmg）、Windows x64（未签名，SmartScreen 需手动放行）、Ubuntu 24.04 .deb/AppImage（Beta，GNOME；Xorg 可显式开启宿主控制，Wayland 禁用）。桌面版内嵌 harness，无需单独跑服务；还能以 companion 模式经 Tailscale 连到另一台主机。
- **npx 版**：`npm install -g openmausbot` 或 `npx openmausbot`，Node 24+；首次启动是方向键引导（选 AI 接入、登录或粘 API Key、选模型、可选连手机）；`openmausbot setup` 可重配而不清数据。注意：API Key 明文存 config.json（0600）。
- **源码版**：Node 24+、pnpm，`pnpm dev:server`（8799）+ `pnpm dev`（5199），仅开发者需要，教程不展开。

## 八、定稿前待办（给后续实测）
- [ ] 租一台真 VPS（2 核 4G）补测路径 B 全程：Docker 拉起、Caddy 真证书、引擎 device-auth 登录、手机真机配对、重启恢复
- [ ] 补测路径 A（--tunnel）与路径 B 的体验差异，确认选型表述
- [ ] 拍齐第六节标 ⚠️ 的截图
- [ ] 复核定价与上架渠道（本草稿未做任何发布/上架动作；未对 GitHub 做任何写操作）
