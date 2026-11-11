# ha-mcp 中文付费教程 — 实测草稿

> 状态：实测完成（沙箱），待补真实 HAOS / 云端客户端截图后上架
> 实测日期：2026-11-10
> 项目：homeassistant-ai/ha-mcp — https://github.com/homeassistant-ai/ha-mcp
> 热度（抓取时）：6,319 ★，今日 +792（父任务第一轮筛选数据）；仓库当前页显示版本线 9.x
> 实测版本：**ha-mcp 9.1.2**（组件 manifest 与 PyPI 同版号）× **Home Assistant 2026.11.1**
> 许可证：MIT（卖的是中文实测教程与服务，不是卖开源代码本身；教程内保留原项目与作者致谢）

---

## 0. 一页结论（给选题/定价决策用）

- **推荐路径已经变了**：2026 年下半年起官方首推「进程内集成」（in-process server），不是装 App（add-on），更不是在电脑上跑 Docker/uvx。教程主线必须按进程内写，App / Docker / uvx 只作附录，否则小白会照着旧博客走弯路。
- **实测跑通的最完整链路**：手动拷贝组件 → HA 2026.11.1 首启 → 建专用管理员账号 → 配置流选 Secret URL → Webhook 连接 URL → MCP initialize / tools/list（77 个工具）/ ha_get_overview / ha_search / ha_get_state / 写待办并读回，**全部成功**；HA 重启后同一 URL 自动恢复，**再次验证成功**。
- **没跑通/没法在沙箱跑的**：Docker 路径（沙箱无 Docker）、App 路径（需要 HAOS/Supervised）、官方 Demo 服务器的实体查询（沙箱网络到 Demo 的 WebSocket 握手超时）、Claude.ai/ChatGPT 云端连接器实连（需要真实账号 + 公网域名）。这几处教程里必须标注「需真机补测/补图」，不能假装测过。
- **卖点成立**：真正的坑不在「装」，而在装完连不上、连上没工具、工具过期、URL 泄露——全是版本与客户端差异造成的，且中文资料几乎没覆盖 9.x 新流程。

---

## 1. 安装路径全梳理（先选一条，只能选一条）

官方原话：custom component、App、Docker/PyPI、local stdio 是「同一个服务器的四种独立跑法」，**每个客户端只配一种**。同一客户端里留两条条目（如一条 `uvx ha-mcp@latest` 的 stdio + 一条 App/集成 URL）是已知连接挂死原因（README 明示；stdio 传输本身还有已知问题，见 issue #1713）。

| # | 路径 | 适用 | 传输/端口 | 要不要令牌 | 定位 |
|---|------|------|-----------|------------|------|
| A | **进程内集成（首推）** | 所有安装类型：HAOS / Supervised / Container / Core | Webhook 走 HA 的 8123；直连端口默认 **9584**，路径 `/private_<random>` | 要：专用管理员账号登录或粘贴管理员长期令牌 | 教程主线 |
| B | HA App（原 add-on） | 仅 HAOS / Supervised | **9583**，日志里打印 `private_<secret>` URL；ingress 有 Open Web UI 按钮 | 不要，Supervisor 自动注入 | 附录一 |
| C | Docker（ha-mcp-web） | Container/Core，或单独一台机器 | Streamable HTTP **8086**，默认路径 `/mcp`（须改高熵 secret path） | 要：`HOMEASSISTANT_URL` + `HOMEASSISTANT_TOKEN` 环境变量 | 附录二 |
| D | PyPI / uvx（HTTP） | 同 C，不想用 Docker 时 | 同 C（`ha-mcp-web` 入口） | 同 C | 附录二 |
| E | 本地 stdio（uvx，**官方不推荐**，仅 demo/折腾） | 桌面客户端同机 | stdio，无端口 | 要：env 里写 URL+令牌 | 只作排障对照 |

补充：
- 旧的 SSE 入口 `ha-mcp-sse`（8087）**已被移除**；拿老教程的 SSE 配置连 `ha-mcp-web` 会返回 **405**。
- 进程内集成还有第二个条目类型 **HA-MCP File & YAML Tools**：只有想开 beta 的文件/YAML 编辑工具（`ha_list_files`、`ha_read_file`、`ha_write_file`、`ha_delete_file`、`ha_config_get_yaml`、`ha_config_set_yaml`）时才加，还要再打开 feature flags（`ENABLE_BETA_FEATURES` 总开关 + 子开关）。**小白主线直接跳过，教程里一句话带过即可，别让读者顺手全开。**

### 1.1 版本要求矩阵（教程第一章必须先讲，劝退一半售后）

| 你的 HA 版本 | 能怎么装 | 说明 |
|--------------|----------|------|
| **2026.11+** | 设置 → **Marketplace** 搜索 HA-MCP → Install | HACS 已内置改名 Marketplace；HA-MCP 在**默认目录**里，不用连 GitHub、不用加自定义仓库；Marketplace 每 6 小时自刷目录 |
| **2026.8.0 – 2026.10** | 打开 **HACS** 搜索 HA-MCP → Download | 流程与上面等价，只是入口叫 HACS |
| **2026.7** | 只能手动拷贝组件，且只能加 File & YAML 条目 + 外部服务器（App/Docker/stdio） | 配置流会**拦截**进程内 Server 条目（低于最低版本直接 abort，报错里写明 installed / required） |
| **< 2026.7** | **装不了** | manifest 依赖 `voluptuous-openapi` 在旧 Core 上无法解析，集成根本不加载；先升级 HA |
| 旧名字 | 搜不到别慌 | 老版本显示为 **HA-MCP Custom Component**，搜「HA-MCP」两个名字都能命中；从旧仓库 `homeassistant-ai/ha-mcp-integration` 装的继续收同样的更新（Marketplace 路径要保持 GitHub 已连接做后台检查） |

其他版本线：
- ha-mcp 本体（独立跑时）`requires-python = >=3.13,<3.15`（9.1.2 的 pyproject 实查）。Python 3.12 及以下装不到新版（见实测日志 §3.2）。
- Home Assistant Core（pip 装法）2026.10.1 的 `requires-python = >=3.14.2`；本次实测 HA 2026.11.1 跑在 Python 3.14.10 上。
- 教程版本基线：**ha-mcp ≥ 9.0.0（2026-10-28）**。9.0 的设置表单新增了「管理员账号」一步，网上 9.0 之前的截图和步骤都对不上，这是中文教程最大的信息差。

### 1.2 HACS 路径 vs Marketplace 路径的区别（读者最容易混）

- 本质是同一个东西换代：2026.11 起 HACS 内置进 HA 并改名 **Marketplace**，入口从「HACS 面板」变成「设置 → Marketplace」。
- Marketplace 首次打开要**接受一次 first-run 警告**，不接受进不去——截图清单里要专门截这一屏。
- HA-MCP 已进默认目录：Marketplace 路径**不需要**「连接 GitHub」也不需要「添加自定义仓库」；只有装自定义仓库（或旧 integration 镜像仓库）时才需要连 GitHub 做 48 小时后台检查。网上老教程里「先加 custom repository 再下载」在新路径里是多余动作，但在 2026.10 及更早的 HACS 上、且目录里搜不到时仍是备用招（加 `https://github.com/homeassistant-ai/ha-mcp`，类别选 Integration）。
- App 商店路径同理改名：2026.2 起「Add-ons」改叫「Apps」，老教程的菜单名对不上号，教程里要给对照表。

### 1.3 标准安装步骤（主线，进程内集成）

1. 按 §1.1 用 Marketplace（2026.11+）或 HACS（2026.8–2026.10）安装 **HA-MCP** 集成；或手动把仓库的 `custom_components/ha_mcp_tools/` 拷到 HA 配置目录的 `custom_components/` 下（实测用的就是手动路径，见 §3.7）。
2. **重启 Home Assistant**（不重启新代码不生效、添加集成里搜不到/更新不生效——官方 troubleshooting 明示）。
3. 先建专用管理员账号（见 §1.4），再去 **设置 → 设备与服务 → 添加集成**，搜 HA-MCP，选 **HA-MCP Server** 条目（不要选成 File & YAML Tools）。
4. 表单里填凭据（二选一，见 §1.4），并选连接方式（见 §1.5）：远程推荐先选 **Secret URL**（最简单）；局域网直连按需开。**注意：v9 新装默认两个通道都是关的**（远程 webhook 关、直连端口只绑 127.0.0.1），不选就只能本机用，别的设备连不上。
5. 提交后等首次启动：HA 会按 manifest 固定版本安装 `ha-mcp==9.1.2` 等依赖，**首启要 1–2 分钟甚至更久**（实测体感一致），不是卡死。
6. 到条目的 **Configure（配置）** 屏复制连接 URL（见 §1.5）。通知里和日志里**故意不带 URL**（通知所有登录用户可见、日志会被 MCP 工具读到），找不到 URL 不是 bug。
7. 把 URL 按 §1.6 的格式填进 AI 客户端，重启/刷新客户端，开新对话问「Can you see my Home Assistant?」或让它列工具验证。

### 1.4 凭据：专用管理员账号 vs 自己的长期令牌（安全核心卖点）

| | 方式一：自己账号的长期令牌（快） | 方式二：专用管理员账号（推荐） |
|--|----------------------------------|--------------------------------|
| 做法 | 左下角头像 → **Security（安全）** → **Long-lived access tokens** → Create token，当场复制（**只显示一次**）后粘贴到表单的 token 欄 | 设置 → 人员 → 添加人员：开 **Allow login**、**Local access only**、**Administrator**；把它的**用户名/密码**填进表单（不是填你自己的！），HA-MCP 自动为它签发名为 “HA-MCP Server” 的令牌，密码不保存 |
| 泄露后果 | **在任何网络都能用**，等于把管理员钥匙挂公网 | 令牌**只在本地网络/回环可用**（服务器本来就在 HA 内部回环连接），泄露面小得多 |
| 审计痕迹 | 服务器做的所有改动记在**你名下** | 记在专用账号名下，一眼分清哪些是 AI 改的 |
| 红线 | — | **Never use your own login for the second option**：表单的用户名/密码欄绝不能填自己的账号；用自己账号就走左边粘令牌 |

实测验证（§3.8）：专用账号（local_only=true、system-admin 组）+ 用户名/密码路径**一次通过**；服务器代码校验很严——账号不是 active 管理员或不是 local_only，会在验密码之前就以同一个 `invalid_login` 拒绝（不泄露具体原因）。非管理员令牌的后果也要讲：ha-mcp 会直接拒绝管理员类请求而不发给 HA，因为 HA 会把非管理员的 401 计入 `http.ban` 失败登录次数，**反复试可能把自己 IP 封了**；已发出的请求则报 `AUTH_INSUFFICIENT_PERMISSIONS`。

令牌自查清单：不要带引号粘贴、不要多空格；过期/被吊销就删掉重建；吊销后服务器下次启动校验失败，会出修复提示 **“HA-MCP needs an administrator access token”**（设置 → 修复）。

### 1.5 远程访问选型（一张表讲清，别让读者全开）

进程内集成的身份模式（Authentication mode）三选一，外加直连端口：

| 模式 | 是什么 | 适合 | 代价/风险 |
|------|--------|------|-----------|
| `none` = **Secret URL** | webhook id（`mcp_` + 长随机串）本身就是密码，URL 形如 `https://<你的HA域名>/api/webhook/<webhook-id>`；局域网版 `http://<HA地址>:8123/api/webhook/<webhook-id>` | 新手首选、最快连通 Claude 等 | URL 泄露 = 钥匙泄露；**绝不能出现在截图、公开日志、公开配置里**；泄露后在 Configure 里勾 **Regenerate connect secrets now** 立即轮换（旧 URL 立刻失效） |
| `ha_auth` = **Home Assistant sign-in** | 客户端跳转用 HA 账号登录（Core 做 OAuth 授权服务器），只接受管理员账号 | 暴露到公网时**官方建议优先** | 客户端要支持 OAuth；某些客户端的回调 URL 要加进 Configure 里的 **Allowed OAuth callback URLs** 白名单（默认已含 claude.ai / claude.com / ChatGPT / Gemini Enterprise 等，列表外会被拦在说明页） |
| `legacy` | 自托管 OAuth，静态 Client ID + Secret 粘贴到客户端 | 少数必须要一对凭据的客户端（如旧版 Copilot CLI、Gemini Spark 备用路） | Secret 就是钥匙；**改凭据/开关此模式后必须完整重启 HA**（只重启条目/App 不生效，否则报 Invalid client id） |
| 直连端口 **9584** | `http://<HA-IP>:9584/private_<random>`，绕过 webhook | 同局域网的桌面客户端 | 网络访问选项要选 Local network 才绑局域网；新装默认只绑 127.0.0.1；明文 HTTP 只在可信内网用 |
| 完全本地 | 关掉 Remote access via webhook | 只在本机/内网玩 | 不注册 webhook，Nabu Casa 也进不来；侧边栏面板和直连仍可用 |

外部可达性怎么来的（三条路，教程里按读者情况分流）：
- **已有 Nabu Casa 或反向代理指向 HA**：什么都不用多装，webhook URL 直接套现有域名。进程内集成**不需要** Webhook Proxy App（那是给 App 路径用的，老教程最容易在这儿多装一个）。
- **什么公网入口都没有**：Cloudflare Tunnel / 自建反代把 HA 送上公网 443（云端客户端的硬性要求见坑 #7）。
- **只为 ChatGPT 且不想暴露公网**：社区的 **OpenAI Tunnel for HA-MCP** 集成（第三方，出站连接把本地 MCP URL 接到 OpenAI 隧道，无需端口转发；出问题找该项目报，不要找 ha-mcp 官方）。
- App 路径的用户走另一套：装 **Webhook Proxy App** 把流量借道 8123 的 webhook 出去，URL 从该 App 日志复制；其 OAuth 改动同样要整机重启 HA 才生效。

另外两个保命开关（放教程「安全收尾」节）：
- 条目设备页有 **Access 开关**（默认关着的一类实体需先启用；关掉 = webhook、直连、对话代理三条路一起断，侧边栏面板还在但显示未运行）。文档特别警告：别把这个开关实体暴露给 Assist，否则语音/对话代理能自己把访问打開；自动化用 `entity_id: all` 也会误伤它。若读者装完发现完全连不上，先来这里看开关状态（本次实测在 setup 时选了远程模式，条目起来后直接可连，未被此开关拦截——不同版本/默认状态以读者现场为准，教程表述为「连不上先查这里」）。
- **Read Only Mode** + 按工具启停/安全策略（侧边栏 HA-MCP 面板里管理）：建议新手连通后先开只读跑一周，再放开写权限。改完工具开关要**重启服务器并刷新客户端工具列表**才生效。

### 1.6 各 AI 客户端的配置格式差异（排障重灾区，逐个给）

总原则：**URL 是从 Configure 屏复制的那一条**；格式错一位就连不上，而且多数客户端不报明确错误。

| 客户端 | 类型 | 配置长什么样 | 关键差异 |
|--------|------|--------------|----------|
| **Claude.ai（网页）** | 远程连接器（UI 填） | 设置 → Connectors → Add custom connector，只粘 URL | 需 Pro/Max/Team/Enterprise；它是从 **Anthropic 云端**来连的（不是你的浏览器），所以 URL 必须公网 443 可达；OAuth 设置要让它**自动探测**，手动硬选（包括选 No sign in）常连不上；添加后还要点 **Connect** 并在同意页点 Allow |
| **ChatGPT** | 远程连接器（UI 填） | 设置里建 connector，粘公网 HTTPS URL | 必须公网可达（或走 OpenAI Tunnel）；工具列表被 ChatGPT **缓存**，更新后要点 **Refresh tools**，无效就删掉连接器**换个名字**重建（同名重建会继续用旧缓存）；有第三方文章称 ChatGPT 侧能「建自动化」受套餐档位限制——上架前需用真实账号再核实，不要写进正文当定论 |
| **Claude Desktop** | 本地 stdio-only | 不能直填 URL，要用桥：`{"mcpServers":{"home-assistant":{"command":"npx","args":["-y","mcp-remote","<URL>","--allow-http"]}}}` 写进 `claude_desktop_config.json`（macOS：`~/Library/Application Support/Claude/`；Windows 传统版：`%APPDATA%\Claude\`，商店版路径另查） | `--allow-http` 只在局域网 http:// 时加，https 去掉；需要 Node ≥ 20.18.1；App 路径的桥配置里**不写令牌**；保存后必须**完全退出**（Cmd+Q / Alt+F4）再开，不是关窗口；服务器条目的 **key 不能带括号**，如 `"Home Assistant (ha-mcp)"` 会让工具数变 0 且毫无报错，改成纯字母数字空格即可 |
| **Claude Code** | CLI | `claude mcp add-json home-assistant '{"url":"<URL>","type":"http"}'` 或向导生成的 JSON | 改完在会话里跑 `/mcp` 重连（或开新会话），旧会话不自动刷新工具 |
| **Codex** | CLI | `codex mcp add home-assistant --url <URL>` | 已知坑：早期 2026 的 HTTP MCP 初始化 bug 会「连上但工具为 0」，同 URL 在别的客户端正常；解法是改走本地 stdio（`uvx`） |
| **Cursor** | HTTP 原生 | `{"mcpServers":{"home-assistant":{"url":"<URL>"}}}` | 顶层键是 `mcpServers`、里面只有 `url`，自动识别 Streamable HTTP；改完重启 Cursor |
| **VS Code** | HTTP 原生 | `{"mcp":{"servers":{"home-assistant":{"type":"http","url":"<URL>"}}}}`（`.vscode/mcp.json` 或设置里） | **顶层是 `mcp.servers` 不是 `mcpServers`**——抄 Claude/Cursor 的格式到 VS Code 是新手第一大错 |
| **Gemini CLI** | HTTP 原生 | `~/.gemini/settings.json` 里 `{"mcpServers":{"home-assistant":{"httpUrl":"<URL>"}}}` | 键名是 **`httpUrl` 不是 `url`**；拿 SSE 式配置（`url`）指 `ha-mcp-web` 会 405 |
| **Windsurf** | HTTP 原生 | `{"mcpServers":{"home-assistant":{"serverUrl":"<URL>"}}}` | 键名是 `serverUrl`，又不一样 |
| **通用 stdio（附录路径）** | 本地命令 | `{"mcpServers":{"Home Assistant":{"command":"uvx","args":["ha-mcp@latest"],"env":{"HOMEASSISTANT_URL":"http://…:8123","HOMEASSISTANT_TOKEN":"…"}}}}` | Claude Desktop 不继承终端 PATH，`uvx not found` 时写**绝对路径**；GUI 应用同理 |

---

## 2. 小白踩坑清单（教程核心卖点，按售后频率排序）

> 格式：现象 → 原因 → 解决。每条都标注来源：[实测] 本次沙箱亲历 / [官方文档] README·FAQ·in-process 文档明示 / [实测+文档] 两边都有。

**坑 1：一个客户端配了两条 ha-mcp（最阴险）** [官方文档]
现象：连接转圈、挂死、时好时坏。
原因：以前用 uvx/Docker 装过没删，又加了集成/App 的 URL，两个独立服务器抢同一个 HA。
解决：**只留一条**。换路径前先去客户端把旧条目删干净再加新的；集成、App、Docker、stdio 四选一。

**坑 2：HA 版本不够还硬装** [官方文档]
现象：添加集成搜不到 HA-MCP；或日志/通知报 `Requirements for ha_mcp_tools not found`、Invalid config；或配置流直接 abort 并写明 required 2026.8.0。
原因：进程内 Server 条目最低 HA **2026.8.0**；低于 2026.7 集成根本加载不了。
解决：先升级 HA 再装；2026.7 只能走手动 + 外部服务器的降级路（见 §1.1 表）。

**坑 3：装完不重启 / 首次启动慢以为卡死** [官方文档+实测体感]
现象：Marketplace/HACS 显示装好了，添加集成里没有；或提交条目后半天没动静。
原因：自定义集成代码只在 HA 启动时加载；首启还要现下载固定版本的 ha-mcp 包（1–2 分钟起，慢网络更久）。
解决：装完先**重启 HA** 再添加集成；提交后耐心等，成功后有 “HA-MCP Server” 通知，URL 去 Configure 屏拿（通知和日志里没有 URL 是设计如此）。

**坑 4：在 HA 刚启动完之前就提交表单** [实测]
现象：本次实测首启期间调 onboarding 建号接口，请求长时间无响应；随后报原文 `Username "sandboxadmin" already exists`——账号其实已建成，但那次的授权码随断开的请求丢了。
原因：HA 的 HTTP 在 bootstrap 没完成时就可能先放行（页面能打开 ≠ 后台就绪），慢 CPU 上首启要数分钟。
解决（写给 Core/手动党读者）：重启/首启后**等到日志出现启动完成、页面稳定**再做建号和添加集成；普通 HAOS 用户对应场景是「刚重启完别连点」。

**坑 5：凭据二选一填错欄** [官方文档]
现象：表单报 `invalid_login`，但密码明明没错。
原因：专用账号欄只收「local_only + 管理员」账号，故意不告诉你具体哪项不对；把**自己的账号密码**填进去也会被拒（官方红线：Never use your own login for the second option）。
解决：用自己账号就改走「粘贴长期令牌」；用专用账号就回人员页核对三个勾：Allow login / Local access only / Administrator。

**坑 6：拿普通（非管理员）账号的令牌硬连** [官方文档]
现象：能连上但建自动化/改配置类工具报 `AUTH_INSUFFICIENT_PERMISSIONS`；更糟的是可能触发 HA 的 IP 封禁计数。
原因：非管理员调管理员接口会吃 401，被 `http.ban` 当失败登录；ha-mcp 干脆在本地拦下这类请求。
解决：只用管理员令牌；已中招先去 HA 看 ban 记录，换管理员凭据重来。

**坑 7：局域网能连、云端客户端连不上（结构性，不是玄学）** [官方文档]
现象：浏览器打开 URL 正常、同局域网的 Claude Code 正常，但 Claude.ai / ChatGPT 死活连不上。
原因（三条硬规则）：①连接器是从 AI 厂商云端发起的，URL 必须**公网可达**；②托管连接器只走 **443、无显式端口**——`https://ha.example.com:8123/...` 在自己浏览器能开，对连接器等于不存在，要用反代/隧道把 443 转进来，HA 本体继续留在 8123；③Cloudflare 的 **Block AI training bots**、地域封锁、mTLS 客户端证书都会拦 AI 厂商出口段（如 Anthropic 的 `160.79.104.0/21`），且发现/令牌交换有 **10 秒**超时，慢反代会间歇失败。Tailscale Funnel 同理必须用 443。
解决：手机关 Wi-Fi 用流量打开 URL 自测；Cloudflare 关 AI bots 封锁并放行厂商 IP 段；外部 URL 欄里填的地址**不能带端口**（官方 options 说明：any port breaks remote MCP clients）。

**坑 8：把 Secret URL 当普通链接分享/截图** [官方文档]
现象：发帖求助把完整 webhook URL 截进去 = 把管理员钥匙公开了。
原因：`none` 模式下 URL 本身就是唯一凭据；webhook 通知、日志都刻意不打印它，唯独 Configure 屏可见（且仅管理员可开）。
解决：求助截图先打码 `/api/webhook/` 后面部分；已泄露立刻 Configure → **Regenerate connect secrets now**（webhook 和直连路径一起换新，旧的即刻失效），再逐个客户端更新 URL。删条目重加也会轮换。

**坑 9：客户端格式一家一个样，抄错键名** [官方文档+向导源码实查]
现象：JSON 没报错但就是不出现服务器/工具。
原因：VS Code 是 `mcp.servers`+`type:"http"`，Gemini CLI 是 `httpUrl`，Windsurf 是 `serverUrl`，Cursor 是 `mcpServers.url`，Zed 是 `context_servers`——没有两家完全一样；Claude Desktop 还根本不能填 URL，要 `mcp-remote` 桥。
解决：只用官方 Setup Wizard 按「你的客户端 × 你的安装方式」现生成配置，别抄博客里的万能 JSON。本草稿 §1.6 的表可直接做成教程对照卡。

**坑 10：Claude Desktop 连上了但工具数为 0** [官方文档]
现象：服务器显示 connected，`initialize` 和 `tools/list` 在日志里都成功，模型侧一个工具都没有，无任何报错。
原因：配置里服务器的 key 带了括号（如 `Home Assistant (ha-mcp)`），派生出的工具名不符合 `^[a-zA-Z0-9_-]{1,64}$`，被客户端在调 API 前静默丢弃。
解决：key 只用字母、数字、空格、`_`、`-`，改完完全退出重开。另排查：Codex 的 HTTP 初始化 bug 也会工具为 0，改 stdio（坑见 §1.6）。

**坑 11：工具列表过期，改名参数被拒** [官方文档+实测旁证]
现象：调用报 `VALIDATION_FAILED` / `unknown parameter`（实测中把 `ha_search` 的参数写成旧名 `search_query` 时，服务器原文回复：`` `search_query`: unknown parameter, did you mean `query`? `` 并列出 Valid parameters——9.1.0 曾把自动化工具的 `identifier` 改名为 `entity_id`，同类问题）。
原因：客户端缓存了旧工具列表，重启 HA/服务器**不会**刷新它。
解决：按客户端刷新——Claude Desktop 连接器断开重连；claude.ai 在连接器设置里 **Refresh tools list** 后开新对话；ChatGPT 进插件设置 **Refresh tools**，还不行就删连接器**换名重建**；Claude Code 跑 `/mcp` 重连。ChatGPT 新启用的工具不刷新永远不出现。

**坑 12：Python 版本坑（只影响自己跑服务器的附录路径）** [实测]
现象（原文）：
- pip（Python 3.12）：`ERROR: Could not find a version that satisfies the requirement ha-mcp==9.1.2 (from versions: 3.3.0, …, 3.5.1)` / `ERROR: No matching distribution found for ha-mcp==9.1.2`
- uvx 强指 3.12：`the current Python version (3.12.3) does not satisfy Python>=3.13,<3.15`
原因：ha-mcp 9.x 要 Python ≥3.13；更坑的是旧解释器上 uvx 可能默默装上 **3.x 的远古版**（FAQ 明示装到 outdated version with known bugs）。
解决：别用系统 pip 硬装；用 `uvx`（它会自己拉对的 Python，本次实测自动下载了 CPython 3.13.18 并跑通 9.1.2），或显式 `uvx --python 3.13 ha-mcp@latest`；用 `uvx ha-mcp@latest --version` 核对版本，工具变少先 `uv cache clean` + `uvx --refresh`。

**坑 13：Docker 用户设置全丢 / 改端口只改一处** [官方文档]
现象：容器一重建，工具开关、feature flags、OAuth 注册全没了；或改了端口容器在跑但永远连不上。
原因：ha-mcp 的数据在容器内 `~/.ha-mcp`（`/home/mcpuser/.ha-mcp`），不挂卷就随容器销毁；端口要在 `-p 主机:容器` 的**第二个数字**和 `MCP_PORT` **两处一致**。
解决：固定 `-v ha-mcp-data:/home/mcpuser/.ha-mcp`（compose 里还要给 volume 写死 `name: ha-mcp-data`）；绑挂目录注意 UID 999 的 chown；看到日志 `Cannot write ha-mcp data to … data will NOT persist across restarts` 就是没挂对。另：Docker 以 stdio 跑时设置页在容器内 127.0.0.1，外面打不开，要用 HTTP 模式。

**坑 14：ha-mcp-web 裸奔默认路径** [实测]
现象（原文）：`ha-mcp listening on 0.0.0.0:8086/mcp with default MCP_SECRET_PATH. … Either bind loopback (MCP_HOST=127.0.0.1) or set MCP_SECRET_PATH to a high-entropy value`
原因：HTTP 模式靠 URL 路径保密，默认 `/mcp` 等于没密码还绑了所有网卡。
解决：自托管 HTTP 时必须设 `MCP_SECRET_PATH=/private_<随机>` 或只绑回环；实测设了 secret path 后，错路径返回 **404**、对路径 GET 返回 **405（这是正常的，Streamable HTTP 端点只收 POST）**、设置页在 `<secret>/settings` 返回 200。把「GET 看到 405 以为坏了」也写进教程——Claude.ai 连接时日志里的 405 同样是正常预检。

**坑 15：Webhook Proxy / legacy OAuth 改完不整机重启** [官方文档]
现象：改了 OAuth 开关或凭据，客户端报 `Invalid client id`，怎么重连都没用。
原因：legacy OAuth 的视图在集成首次加载时绑进 HA 的 HTTP 层，reload/重启 App 都不会重注册（webhook 本身会）。
解决：动 legacy OAuth 后 **设置 → 系统 → 重启** 整个 HA，再删掉客户端连接器重加。另分清：进程内集成自带 webhook，**不需要**再装 Webhook Proxy App（它只服务 App 路径）。

**坑 16：端口/名字对不上号** [官方文档+实测]
速记：进程内直连 **9584** / App **9583** / Docker·web **8086** / 已删除的 SSE **8087**；集成在侧边栏叫 **HA-MCP** 面板；条目在设备页叫 “HA-MCP — Manage Home Assistant with AI” → HA-MCP Server。拿 9583 的老教程去连进程内安装，必然连不上。

**坑 17：代理/杀毒软件劫持 HTTPS（公司电脑党）** [官方文档+实测变体]
现象：`uvx` 下载时报 `invalid peer certificate: UnknownIssuer`，或客户端里服务器一闪就断（transport closed）。
原因：uv 用自带证书库、不认系统库，公司代理/Zscaler/杀毒的 HTTPS 扫描证书不被信任；Node（mcp-remote）同理不认系统库。
解决：uv 加环境变量 `UV_NATIVE_TLS=1`；Node 加 `NODE_EXTRA_CA_CERTS=<pem路径>`。实测旁证：本次沙箱的代理环境变量 `NO_PROXY` 里带 `[::1]` 这类括号 IPv6 写法，直接让 ha-mcp 的 httpx 在启动时抛 `httpx.InvalidURL: Invalid port: ':1]'` 崩掉，把 `NO_PROXY` 清成 `localhost,127.0.0.1` 后立刻恢复——**凡是「在本机能跑、换个环境就崩」的，先查代理环境变量**，这是通用排障顺序，值得单独成节。

**坑 18：第一次打开太慢** [官方文档+实测]
现象：uvx 首次运行、HA 首次加载组件、uvx 冷缓存起 web 服务，都要现下载几十 MB（实测冷启 uvx 拉了 Python 3.13 + 71 个包）。
解决：先在终端单独跑一次 `uvx ha-mcp@latest --version` 预热缓存，再回客户端里连；别在客户端首次拉起时因为超时就改配置，越改越乱。

---

## 3. 实测日志（2026-11-10，沙箱）

### 3.0 环境

- Ubuntu 24.04.5 LTS，共享 CPU（1.5 核配额）、内存约 6–8 GB；**无 Docker**（`docker: command not found`）、无 podman。
- 系统 Python **3.12.3**（故意保留，用来复现版本坑）；uv **0.13.7**（装进 3.12 的 venv 里）；uv 自动提供的 CPython **3.13.18** 与 **3.14.10**。
- 网络只能经沙箱出口代理出去；`/` 盘 150 GB，但 **/tmp 只有 512 MB tmpfs**（坑见 §3.6）。
- 做法总览：克隆官方仓库读源码与文档 → uvx 跑本体 → 连官方 Demo → 自起 ha-mcp-web → pip 装真 HA 2026.11.1 → 手动装组件 → API 走完建号/配置流 → Webhook 端到端调工具。GitHub 上**零写操作**。

### 3.1 仓库与文档精读 ✅

- README（抓取时仓库 6,319 ★、3,455 commits、MIT）主线已改为进程内集成；`docs/in-process-server.md` 为权威细节（版本三层门槛、凭据两种、三种 auth 模式、Access 开关、轮换、故障修复标题等均出自此文）。
- 组件 `manifest.json` 实查：domain `ha_mcp_tools`，version **9.1.2**，requirements 固定 `ha-mcp==9.1.2`（+ `mcp>=1.24.0`、`ruamel.yaml>=0.18.0`、`voluptuous-openapi>=0.4.1`）；`hacs.json` 写明 `"homeassistant": "2026.8.0"`。
- 配置流源码实查：第一步是菜单（server / tools 二选一）；server 表单字段为 `admin_token`（密码框，可选）、`admin_username`、`admin_password`、`remote_access`（disabled/ha_auth/legacy/none）、`bind_host`。与 UI 文档逐项对得上。
- Setup Wizard 源码实查各客户端生成格式（§1.6 的表据此写，非博客转述）：VS Code `mcp.servers`+`type:http`、Gemini CLI `httpUrl`、Windsurf `serverUrl`、Claude Desktop 走 `mcp-remote` 桥等。
- App 的 `config.yaml` 实查：ingress 端口 9583、`hassio_role: manager`、镜像 `ghcr.io/homeassistant-ai/ha-mcp-addon-{arch}`，仅 aarch64/amd64。

### 3.2 本体启动：uvx ✅ / pip 与旧 Python ❌（预期内失败）

| 试验 | 结果 |
|------|------|
| `uvx --from ha-mcp==9.1.2 ha-mcp --version` | ✅ 自动下载 CPython 3.13.18 + 71 个包，输出 `ha-mcp 9.1.2` |
| Python 3.12 的 pip 装 9.1.2 | ❌ `ERROR: No matching distribution found for ha-mcp==9.1.2`（可选版本只剩 3.3.0–3.5.1 的老版——印证 FAQ「旧 Python 会装到远古版」） |
| `uvx --python 3.12 …` | ❌ `the current Python version (3.12.3) does not satisfy Python>=3.13,<3.15 … requirements are unsatisfiable` |

### 3.3 stdio 连官方 Demo：协议层 ✅ / 实体层 ❌（网络卡点，如实记录）

Demo 按 FAQ 公开参数：URL `https://ha-mcp-demo-server.qc-h.net`、Token `demo`。

- 第一次启动直接崩：`httpx.InvalidURL: Invalid port: ':1]'`，定位为沙箱 `NO_PROXY` 含 `[::1]` 等括号 IPv6 条目触发 httpx 的 URLPattern 解析 bug；把 `NO_PROXY` 改为 `localhost,127.0.0.1` 后恢复（坑 #17）。
- 恢复后：MCP `initialize` ✅（serverInfo `ha-mcp 9.1.2`）；`tools/list` ✅ 返回 **77 个工具**（默认目录，不含 beta 文件/YAML 工具——README 宣称的 87 是含 opt-in 的总数，教程里要这样解释，免得读者数工具对不上发慌）；`ha_get_skill_guide` ✅（本地资源，不依赖 HA 连接）。
- 但凡是要真连 Demo HA 的调用都失败：日志原文 `WebSocket connection failed: timed out during opening handshake`；`ha_get_overview` 返回 `CONNECTION_TIMEOUT`（“Request timeout”）；`ha_get_system_health` 返回 `CONNECTION_FAILED`（“Failed to connect to Home Assistant WebSocket … timed out during opening handshake”）；`ha_search` 等待超时。同期 curl 直打 Demo 的 `/api/` 也超时、无代理直连返回 000。**结论：沙箱出口到 Demo 的 wss/https 链路不通（Demo 本身慢/不可达或代理不转发升级请求），不是本机部署问题**——后面 §3.9 用自建 HA 完成了同一批调用的对照验证。
- 附带收获：把参数写成旧名 `search_query` 时服务器明确纠错（`did you mean 'query'?` + Valid parameters + 提示工具列表可能过期），成为坑 #11 的一手素材。

### 3.4 ha-mcp-web（HTTP 模式） ✅

- 冷启第一次探活失败（18 秒内未绑端口，包还在下载）；预热后约 4 秒监听。
- 默认启动警告原文见坑 #14（默认 `/mcp` + 绑 0.0.0.0 的安全警告属实）。
- 设 `MCP_SECRET_PATH=/private_sandboxtest123`（测试值）后验证：POST initialize ✅ 返回 9.1.2（SSE event 格式）；GET 端点 **405**（正常）；`<secret>/settings` **200**；错路径 **404**。

### 3.5–3.6 装真 Home Assistant：第一次 ❌ / 第二次 ✅

- 第一次（venv 建在 /tmp）：uv 已下载完 91 个包，安装阶段报原文
  `error: Failed to install: pyric-0.1.6.3… cause: failed to copy file … No space left on device (os error 28)`
  原因：/tmp 是 512 MB tmpfs。HA 本体 venv 实测约 **1.3 GB**——写进教程提醒 Core 玩家别把 venv 放小分区/内存盘。
- 第二次（venv 建在主盘）：✅ `python -m homeassistant --version` → **2026.11.1**（Python 3.14.10）。

### 3.7 HA 首启 + 手动装组件 ✅（慢，且有一个真坑）

- 把仓库 `custom_components/ha_mcp_tools/` 拷入 `config/custom_components/` 后首启，日志原文：
  `We found a custom integration ha_mcp_tools which has not been tested by Home Assistant. This component might cause stability problems…`（手动装必出的警告，教程要提前打预防针：这是所有自定义集成的例行警告，不是报错。）
- 首启约 3 分钟 HTTP 才就绪（慢 CPU + 首次装配），期间 zeroconf/SSDP 的 `Operation not permitted` 是沙箱组播限制，**与普通家庭网络无关，不写进教程正文**，此处仅存档。
- 就绪前提交的 onboarding 建号请求长时间无响应，账号实际已创建但响应/授权码丢失，重提报 `Username "sandboxadmin" already exists`（坑 #4）。改为标准登录流（login_flow → auth/token）后拿到 access/refresh token ✅；`core_config`、`analytics` 两个引导步 API 提交 ✅。（注：引导状态里 `user` 步仍显示未完成，是先建号后补登录流的顺序产物，不影响使用；正常 UI 引导不会遇到。）

### 3.8 建专用管理员账号 ✅

用管理员 token 走 WebSocket API 等价于 UI 的「设置 → 人员 → 添加人员」：
- `config/auth/create` 建用户 “HA-MCP Server”，`group_ids=["system-admin"]`、`local_only=true` ✅
- `config/auth_provider/homeassistant/create` 给它挂用户名 `ha-mcp` + 密码 ✅
- 顺带用 `auth/long_lived_access_token` 给主账号签了一张长期令牌（长度 183）备用 ✅

### 3.9 配置流 + 进程内服务器 + Webhook 端到端 ✅✅（核心成果）

1. REST 发起配置流 `handler=ha_mcp_tools` → 返回菜单（server/tools）✅；选 server → 表单字段与源码/文档一致 ✅。
2. 提交 **专用账号用户名/密码** + `remote_access="none"`（Secret URL）+ `bind_host="127.0.0.1"` → `create_entry`，条目 **state=loaded**，options 为 `enable_webhook:true, webhook_auth:"none"` ✅。**HA 自动按 manifest 把 ha-mcp 9.1.2 装进自身环境并在进程内拉起**（无需单独 pip）。
3. 端口验证：HA 进程在 **127.0.0.1:9584** 监听 ✅（绑回环是本次的选择；选 Local network 时才会绑局域网）。
4. Configure（options 流）屏原文：`Component 9.1.2 - Server ha-mcp 9.1.2 (installed with this component release)`，并给出 Connect URL（webhook 形态，本沙箱的 HA 地址显示为虚拟 IP，家庭环境应为读者自己的 HA 域名/地址）；同屏确认 File & YAML 模块 `Not installed`（未加第二条目，符合预期）。OAuth 回调白名单默认值里已含 claude.ai、claude.com、localhost、chatgpt 等，与文档一致。
5. **通过 Webhook URL 调 MCP**（经 HA 8123，与远程客户端同一路径）：
   - initialize ✅ `ha-mcp 9.1.2`，无会话 ID（stateless，与 FAQ 说明一致）
   - tools/list ✅ **77 个**
   - `ha_get_overview` ✅ success：17 个实体、7 个域、245 个服务、HA 版本 2026.11.1
   - `ha_search`（query="sun"）✅ 精确命中一组 sun 传感器
   - `ha_get_state`（sun.sun）✅ 返回 above_horizon 及完整属性
   - **写操作**：`ha_set_todo_item` 往 `todo.shopping_list` 加一条测试项 ✅，`ha_get_todo` 读回 count=1 ✅（读→写闭环成立）
   - GET webhook URL 返回 **405**（正常，见坑 #14）
6. **重启 HA 后**：同一 webhook URL 再次 initialize/tools/list/overview **全部成功**，9584 自动重新监听 ✅（连接 URL 与密钥持久化，不是重启即失联）。
7. 本次未触发文档里「Access 开关默认关需先启用」的拦截（setup 时选了远程模式，条目起来即可连）；教程按「连不上先查设备页 Access 开关」表述，不写死。

### 3.10 未测清单（诚实边界）

| 未测项 | 原因 | 教程处理 |
|--------|------|----------|
| Docker 路径（ghcr 镜像 / compose） | 沙箱无 Docker | 附录只写官方命令 + 坑 #13，并标注「未实测，待真机复核」 |
| App（add-on）路径 | 需要 HAOS/Supervised，Core 环境装不了 App | 附录二选一时说明；截图需真机补 |
| Claude.ai / ChatGPT 云端连接器实连 | 需要真实付费账号 + 公网域名/Nabu Casa | 正文按文档写步骤，截图清单里列为**上架前必补**；公网 443 等规则来自官方 FAQ |
| Nabu Casa 远程 URL 实连 | 无 Nabu Casa 订阅环境 | 同上 |
| 语音/对话代理选 HA-MCP LLM API | 非教程主线 | 大纲里只留一节选读，不承诺实测 |

---

## 4. 教程大纲（可直接开写）

### 4.1 标题备选

1. 《让 Claude / ChatGPT 替你管 Home Assistant：ha-mcp 小白一键部署 · 2026.11 Marketplace 最新版》
2. 《不用写 YAML 了：ha-mcp 实测部署全记录（87 个工具、含全部踩坑）》
3. 《HA-MCP 中文保姆级教程：从 Marketplace 安装到 Claude、ChatGPT、Cursor 全客户端连通》

主推 1：带版本号和最新入口（Marketplace），天然和旧博客拉开差距——这是定价底气。

### 4.2 定价

统一售价 **9.9 元**（2026-11-11 用户拍板，全线教程统一价），全内容不分档；版本更新勘误免费跟进（ha-mcp 更新极快，156 个 release、版本号几乎每月跳）。

### 4.3 卖点文案草案（小报童/知识星球商品页）

> **你的 Home Assistant 还在手写 YAML？让 AI 直接替你建自动化。**
>
> ha-mcp（GitHub 6,300+ Star、今日还在暴涨）能把 Home Assistant 的 87 个管理工具交给 Claude、ChatGPT、Cursor：说人话就能建自动化、改仪表盘、查日志排障。但它的官方流程在 2026 年 10 月底刚大改版（9.0 新凭据表单、2026.11 内置 Marketplace），网上的中文教程全是旧版，照着装只会连不上。
>
> 这份教程基于 **ha-mcp 9.1.2 × Home Assistant 2026.11.1 全程实测**（不是翻译 README）：
> ✅ Marketplace / HACS / 手动三种装法对照，先查版本表再动手，劝退装不了的环境
> ✅ 专用管理员账号的正确建法——不用你的主账号令牌冒险，泄露也不怕
> ✅ Secret URL / HA 登录 / legacy 三种远程模式一张表选清，含 Nabu Casa、Cloudflare、ChatGPT 打通法
> ✅ Claude、ChatGPT、Claude Desktop、Claude Code、Codex、Cursor、VS Code、Gemini CLI **八个客户端的配置逐个给**（每家的键名都不一样，这是连不上的头号原因）
> ✅ 18 个真实踩坑：装了两条服务器挂死、工具数为 0、工具列表过期、URL 泄露轮换、Cloudflare 拦 AI……每个都给现象和解法
>
> 买完 30 分钟，让你的 AI 第一次自己写出一条能跑的自动化。

（注：文案中的「今日还在暴涨」「87 个工具」要在上架当天复核数字再发；工具数正文按「默认 77、含 opt-in 共 87」解释。）

### 4.4 目录与每节要点

**第 0 章 先说清：ha-mcp 和 HA 自带 MCP 不是一回事**（要点：自带 MCP 只能控制已暴露给 Assist 的设备；ha-mcp 能建/改自动化、仪表盘、脚本、备份、注册表——选错的人先劝走，少一半售后）
**第 1 章 动手前查版本**（§1.1 版本矩阵；Marketplace 与 HACS 的关系；你的安装类型决定可选路径表）
**第 2 章 主线安装：进程内集成**（§1.3 七步；不重启的后果；首启慢的预期管理；通知里没 URL 不是 bug）
**第 3 章 凭据与安全姿态**（§1.4 两方式对照与红线；非管理员令牌与 IP ban；Read Only Mode 建议；Access 开关）
**第 4 章 选远程访问模式**（§1.5 五格表；Nabu Casa/反代/隧道分流；外部 URL 不带端口；URL 即密码与轮换操作）
**第 5 章 连你的第一个客户端（Claude.ai）**（连接器添加全流程；自动探测 OAuth；点 Connect；用 “Can you see my Home Assistant?” + 列工具验证；公网 443 三规则）
**第 6 章 其他客户端逐个连**（§1.6：Claude Desktop 的 mcp-remote 桥与完全退出、Claude Code、Codex 工具为 0 的备路、Cursor、VS Code 的 mcp.servers、Gemini 的 httpUrl、Windsurf 的 serverUrl）
**第 7 章 第一次让 AI 干活**（三个可复制的开场提示词：日落开灯自动化、加天气卡片、给现有自动化加动作；先开只读观察，再放开写；自动备份与改动回滚说明）
**第 8 章 排障手册**（§2 坑清单按症状索引重排：连不上 / 没工具 / 工具有但调用失败 / 更新后出问题 四组，每组一张流程图）
**附录 A App（add-on）路径**（9583、日志取 URL、Webhook Proxy、何时选它；标注未实测）
**附录 B Docker / uvx 自托管路径**（卷、端口两处一致、secret path、Python 版本；标注 Docker 未实测）
**附录 C File & YAML beta 工具**（第二条目 + feature flags + 风险提示，只给进阶读者）
**附录 D 名词对照表**（Add-ons→Apps、HACS→Marketplace、旧名 HA-MCP Custom Component、旧 SSE 已删除）

### 4.5 需要截图的位置清单（上架前拍摄；★ = 沙箱无法拍、必须真机/真账号补）

1. 设置 → Marketplace 首次打开的警告屏（接受那一下）
2. Marketplace 搜 “HA-MCP” 的结果页（新名） ★（沙箱是手动装，无此屏）
3. HACS（2026.8–2026.10）搜 HA-MCP → Download 页（有旧版本机器时补） ★
4. 设置 → 系统 → 重启 的位置（装完重启这一步）
5. 设置 → 人员 → 添加人员：Allow login / Local access only / Administrator 三个开关同框
6. 头像 → Security → Long-lived access tokens → Create token（令牌只显示一次的提示，**截图时令牌必须打码/用废令牌**）
7. 添加集成搜 HA-MCP 出现两个条目（Server / File & YAML Tools）的选择菜单
8. Server 表单全屏：凭据欄 + remote_access 四个选项 + bind_host（核心截图）
9. 提交后 “HA-MCP Server” 启动通知
10. Configure 屏：Connect URL 区域（**webhook id 打码示范**，顺带教读者打码）+ Regenerate connect secrets 的位置
11. 侧边栏 HA-MCP 面板首页（工具开关/只读模式）
12. 设备页 HA-MCP Server 的 Access 开关
13. Claude.ai 添加自定义连接器并 Connect 成功、工具列表展开 ★
14. ChatGPT 连接器设置页与 Refresh tools 按钮 ★
15. Claude Desktop 配置文件粘贴 mcp-remote 配置前后 + 重启后工具图标 ★（需桌面系统）
16. 第一条 AI 生成的自动化在 HA 里打开的效果页
17. 设置 → 修复 里与 HA-MCP 有关的修复条目示例（有条件时）
18. 排障对照：客户端里「工具数为 0」与「刷新后恢复」的同机对比 ★

### 4.6 交付与合规备忘

- 交付形态：图文正文（小报童/知识星球）+ §1.6 对照卡单独成页；所有含 URL/令牌的示例图一律打码或用测试值（如本文的 `/private_sandboxtest123`、Demo 的公开 token `demo`）。
- 合规：项目 MIT 协议；教程卖的是原创中文实测内容与排障服务，不打包分发其代码；正文保留项目链接与作者致谢；不承诺收益、不替读者保管任何令牌。
- 更新义务（免费勘误）：ha-mcp 发版极快（本次实测期间 README 已是 9.1.2），勘误帖至少跟到下一个大版本（10.x）或 HA 2026.12，重点盯三处：Marketplace 目录政策、配置流字段、客户端刷新方式。

---

## 5. 给父任务的一句话汇总

进程内推荐路径已在真 HA 2026.11.1 上端到端跑通（含写操作与重启恢复）；最大坑是「双装冲突、HA 版本门槛、凭据欄填错、云端客户端的公网 443 规则、客户端配置键名各异与工具列表过期」。Docker/App/云端连接器三项受沙箱条件所限未实测，已在草稿中诚实标注并列入真机补图清单。
