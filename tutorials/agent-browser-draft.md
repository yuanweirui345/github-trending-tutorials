# agent-browser 中文付费教程草稿（候选 2 号）

> 项目：vercel-labs/agent-browser — https://github.com/vercel-labs/agent-browser
> 定位：**不卖安装，卖场景实战包**。安装只有两条命令，值钱的是：① 国内网络下装不上 Chrome 的解法；② 自动填表 / 网页巡检 / 数据抓取 / E2E 测试四套能直接套用的配方；③ 让 Claude Code / Codex 等 AI Agent 真正调通它的接入方法。
> 实测日期：2026-11-10。协议：Apache-2.0（卖的是自有实测与配方，教程中保留原项目署名与协议说明）。

---

## 一、实测结论：跑通到哪一步

| 步骤 | 结果 | 证据 |
|---|---|---|
| `npm install -g agent-browser` | ✅ 成功，41 秒 | 版本 `agent-browser 0.42.0`（linux x86_64 原生 Rust 二进制） |
| `agent-browser install --with-deps` | ⚠️ 卡死后手动终止 | apt 阶段反复重试一个字体包 13 分钟以上（见坑 3） |
| `agent-browser install`（直连下载 Chrome） | ❌ 失败，30 秒超时 | 报错原文见坑 1 —— 这是本教程最大的卖点 |
| 镜像离线方案（npmmirror 下载 Chrome for Testing） | ✅ 成功，195MB / 41 秒 | 放进官方缓存目录后 `doctor` 显示 Chrome pass |
| 最小 Agent 场景：开网页 → 快照 → 填表 → 提交 → 验证 | ✅ 全通 | 页面回显「提交成功，欢迎你，小明」（用 @ref 和 CSS 选择器各跑通一次） |
| 数据抓取场景：表格 → 结构化 JSON | ✅ 全通 | `eval` 提取 3 件商品的名称/价格/库存为 JSON 数组 |
| `read` 把网页转成 Agent 可读文本 | ✅ 全通 | 本地测试页完整转出 Markdown 式文本 |
| 截图留证 | ✅ 全通 | `screenshot` 保存 PNG 成功（29KB） |
| 打开公网站点 example.com | ❌ 沙箱内失败 | Chrome 报 `net::ERR_TUNNEL_CONNECTION_FAILED`（沙箱代理环境所致，见坑 7；非产品问题） |
| E2E 测试配方 | 📝 已按官方命令与 dogfood skill 整理成配方，未做完整项目级实测 | 教程中标注为「配方篇」，不要宣传为全量实测 |

实测环境：Linux x86_64 沙箱、Node v24.22.1、npm 10.9.4、npm 源为官方 registry.npmjs.org、内存约 8GB。注意：本沙箱的出网方式与国内家庭网络不完全相同，但「直连 Google 下载域名超时」这一现象与国内用户反馈的卡点一致，且镜像解法是在同一网络下实测跑通的。Windows / macOS 未在本轮实测，教程中对应章节须标注「原理相同、路径不同」并在正式售卖前补测。

---

## 二、实测日志（关键原文）

### 1. 安装 CLI 本体

```bash
$ npm install -g agent-browser
added 1 package in 41s
$ agent-browser --version
agent-browser 0.42.0
```

41 秒在国内直连官方 npm 源属于偏慢（包内含原生二进制）。把 npm 源换成 npmmirror 可明显加速，属于教程的「加速小节」。

### 2. 下载 Chrome for Testing —— 直连失败（核心卖点）

```bash
$ agent-browser install
⚠ Linux detected. If browser fails to launch, run:
  agent-browser install --with-deps

Installing Chrome...
✗ Failed to fetch version info: error sending request for url (https://googlechromelabs.github.io/chrome-for-testing/last-known-good-versions-with-downloads.json): client error (Connect): operation timed out
（30 秒后失败，退出码 1）
```

`agent-browser doctor` 给出了同样的结论：

```text
fail  Chrome for Testing CDN unreachable after 3038ms: error sending request for url (https://googlechromelabs.github.io/chrome-for-testing/last-known-good-versions-with-downloads.json)
      fix: check network connectivity / firewall / proxy settings
fail  No Chrome binary found
      fix: agent-browser install
```

即：它先去 `googlechromelabs.github.io` 查最新版本号，再去 `storage.googleapis.com/chrome-for-testing-public/...` 下载本体。两个域名在国内直连都不稳定，这一步会劝退绝大多数小白——教程的付费理由就成立在这里。

### 3. 镜像离线方案（已实测跑通）

思路：版本号和文件在国内镜像（npmmirror 同步了 chrome-for-testing 二进制）上都有，手动下载后放进 agent-browser 的缓存目录，它就不再联网下载。

```bash
# 当时查到的最新稳定版：157.0.8103.50（版本号会变，教程里教读者先查版本再拼链接）
# 官方地址形如：
#   https://storage.googleapis.com/chrome-for-testing-public/157.0.8103.50/linux64/chrome-linux64.zip
# 镜像地址只需换域名与路径前缀：
$ curl -LO https://cdn.npmmirror.com/binaries/chrome-for-testing/157.0.8103.50/linux64/chrome-linux64.zip
# 实测：195MB，41 秒下完，zip 完整性校验通过（官方直连则是超时失败）

$ unzip chrome-linux64.zip
# 放进缓存目录（目录名规律：chrome-<版本号>/chrome-linux64/chrome）
$ mkdir -p ~/.agent-browser/browsers/chrome-157.0.8103.50
$ mv chrome-linux64 ~/.agent-browser/browsers/chrome-157.0.8103.50/

$ agent-browser doctor
Chrome
  pass  Google Chrome for Testing 157.0.8103.50 at /home/hatch/.agent-browser/browsers/chrome-157.0.8103.50/chrome-linux64/chrome
```

备选方案（教程中一并给出，按易到难排）：
- **方案 B：指定已有 Chrome**。电脑上已装 Chrome/Chromium 时，不必再下载：启动时加 `--executable-path <chrome路径>`，或设环境变量 `AGENT_BROWSER_EXECUTABLE_PATH`（`--help` 原文确认支持）。实测用该参数指定镜像下载的 Chrome，同样完整跑通全部场景。
- **方案 C：走代理**。`--help` 确认支持 `--proxy "http://user:pass@127.0.0.1:7890"` 或环境变量 `AGENT_BROWSER_PROXY / HTTP_PROXY / HTTPS_PROXY`。有代理的读者让 `agent-browser install` 走代理即可；同时提醒用 `--proxy-bypass "localhost,*.internal.com"` 避免本地/内网站点被代理绕远。
- **方案 D：云浏览器**。doctor 输出显示它支持 Browserbase / Browserless / Browser Use / Kernel 等云浏览器（需各自 API Key），完全不想在本机装 Chrome 的读者可用，但要额外付费，教程中只做介绍不做主推。

平台对应关系（教程表格）：Linux 用 `linux64/chrome-linux64.zip`；Windows 用 `win64/chrome-win64.zip`，缓存目录同理为 `chrome-<版本号>/chrome-win64/chrome.exe`；Mac 按芯片选 `mac-arm64` 或 `mac-x64`。此对应关系按官方发布文件命名推导，Win/Mac 路径须补测后才能在教程里写「已验证」。

### 4. 最小 Agent 场景（本地演示商店页，全流程实测）

自建了一个带商品表格 + 订阅表单的中文测试页（本地 http 服务），用 agent-browser 跑官方推荐的核心闭环：`open → snapshot → 行动 → 再 snapshot 验证`。

```bash
$ agent-browser open http://127.0.0.1:8901/
✓ 演示商店 - 巡检测试页

$ agent-browser snapshot -i
- textbox "姓名" [ref=e16]
- textbox "邮箱" [ref=e17]
- button "提交订阅" [ref=e3]
#（表格单元格也都有 ref：机械键盘 ¥399 有货 / 无线鼠标 ¥159 有货 / 显示器支架 ¥229 缺货）

$ agent-browser fill @e16 '小明' && agent-browser fill @e17 'xiaoming@example.com' && agent-browser click @e3
✓ Done ×3
# 用 CSS 选择器 fill '#name' / fill '#email' / click '#submit' 同样跑通，两种定位方式都可教

$ agent-browser get text '#ok'
提交成功，欢迎你，小明        ← 提交结果验证成功

$ agent-browser eval "JSON.stringify([...document.querySelectorAll('#products tr')].slice(1).map(tr=>[...tr.cells].map(c=>c.textContent)))"
"[["机械键盘","¥399","有货"],["无线鼠标","¥159","有货"],["显示器支架","¥229","缺货"]]"   ← 抓取配方成立

$ agent-browser read http://127.0.0.1:8901/
# 演示商店 …（完整可读文本，适合直接喂给 AI 做摘要/巡检判断）

$ agent-browser screenshot demo.png
✓ Screenshot saved
```

---

## 三、坑清单（教程的核心价值，按小白踩中概率排序）

**坑 1：`agent-browser install` 一直转圈/超时，Chrome 装不上**
- 现象：`Failed to fetch version info: error sending request for url (https://googlechromelabs.github.io/chrome-for-testing/...)：operation timed out`，或 doctor 报 `Chrome for Testing CDN unreachable`。
- 原因：它要直连 Google 的两个域名查版本、下本体（约 195MB，解压后约 409MB），国内网络大多超时。
- 解法：用第二节的镜像离线方案（首选）、`--executable-path` 复用本机 Chrome、或 `--proxy` 走代理。这一条就是整套教程的定价锚点。

**坑 2：Linux 上加了 `--with-deps` 反而卡死在系统依赖**
- 现象：实测中 `apt-get install` 阶段在 `fonts-freefont-ttf` 一个包上反复出现 `Ign:` 重试，13 分钟未完成；此前 `apt-get update` 还出现镜像源 GPG 校验错误 `Splitting up ... into data and signature failed`、`The repository ... is not signed`（沙箱镜像源问题）。
- 原因：`--with-deps` 会先跑一遍 apt 装 30 多个图形/字体库，任何一个源不稳定就整体卡住；而实测中除一个字体包外其余库都已是最新版本，本来就不需要装。
- 解法：先不带 `--with-deps` 直接装 Chrome 试启动；真报缺库（doctor/启动报错会点名）再单独 `apt-get install <缺的库>`；字体包缺失只影响个别页面渲染，不阻塞主流程。Windows/Mac 无此坑。

**坑 3：`Unknown ref: e16` / 快照是 `(empty page)`，以为工具坏了**
- 现象：上一条命令 snapshot 里明明有的 @e16，下一个终端里 fill 就报 `Unknown ref`；或 snapshot 返回 `(no interactive elements)`、`(empty page)`。
- 原因：ref 只对「当前会话、当前页面状态」有效。换了 `--session`、页面跳转/刷新过、或浏览器守护进程已重启，旧 ref 全部作废。本沙箱里守护进程不跨终端会话存活，跨终端必现此坑；普通电脑上守护进程空闲约 1 小时才退出（官方文档），但换 session、关过浏览器同样会中招。
- 解法：① 每条命令都带同一个 `--session <名字>`（官方 core skill 特别强调：多 Agent 环境不要用默认会话，会互相抢页面）；② 每次导航或页面变化后重新 `snapshot` 再用新 ref；③ 一连串动作尽量写成一个脚本一次跑完（官方自带 `templates/form-automation.sh` 等模板就是这个思路）；④ 嫌 ref 麻烦可改用 CSS 选择器 / `find role button click --name "提交"` 这类语义定位，跨页面更稳。

**坑 4：点击「成功了」但其实没点上，或直接报错**
- 现象与原因：官方 README 明确——目标被遮挡（Cookie 同意横幅最常见）、隐藏、禁用或在视口外时，click 会失败而不是假装成功。
- 解法：先关掉遮挡元素 → 重新 snapshot → 再点；或先 `scrollintoview` 再点。教程里要把这条做成巡检/填表配方的固定检查步，别让读者对着报错猜。

**坑 5：有代理时，本地/内网页面打不开**
- 现象：沙箱实测中 Chrome 打开公网页面报 `net::ERR_TUNNEL_CONNECTION_FAILED`，且命令输出带 `⚠ proxy-password: <redacted>` 提示——它读到了环境里的代理设置。
- 原因：agent-browser 会继承 `HTTP_PROXY/HTTPS_PROXY` 等环境变量（也可用 `--proxy` 显式指定），代理不可用或把本地地址也绕进代理时就会失败。
- 解法：本地测试页、内网系统要加 `--proxy-bypass "localhost,*.internal.com"` 或确保 `NO_PROXY` 包含它们；代理本身不通时先修代理，别怀疑工具。

**坑 6：磁盘/临时目录空间不足，下载一半损坏**
- 现象：实测中下载到空间受限的临时目录时 195MB 的 zip 只下到 114MB 就写失败（curl rc 23），解压必然报错。
- 原因：Chrome 本体 zip 约 195MB、解压后约 409MB，加上浏览器缓存，至少要预留 1GB。
- 解法：下载和解压都放在空间充足的磁盘目录，先 `df -h` 看一眼；zip 下完先校验完整性再解压，别对着损坏包排查半天。

**坑 7：照着网上旧文章敲命令，对不上**
- 现象与原因：这个工具迭代极快（实测版本 0.42.0，仓库 121 个 release），博客里的命令和参数很快过时。
- 解法：只认 CLI 自带的版本匹配文档——`agent-browser skills get core`（核心闭环）、`agent-browser skills get core --full`（全量命令与模板）、`agent-browser skills list`（electron、slack、dogfood 等专项 skill）。教程的方法论一节要教读者「让你的 AI 先跑这条命令再干活」，这比记命令耐用得多。

**坑 8：装好 CLI 却没接上自己的 AI Agent，白装**
- 现象：命令行能跑，但 Claude Code / Codex 不会自己调用它。
- 解法（官方路径，教程接入篇）：`npx skills add vercel-labs/agent-browser` 把发现 skill 装进 Agent；或在项目 `AGENTS.md` 写明「Use agent-browser for browser automation. Before the first use, run `agent-browser skills get core` and follow it.」；MCP 客户端则用 `agent-browser mcp` 起 stdio MCP 服务。三种接法在教程里各给一份可抄的配置。

---

## 四、教程大纲（场景实战包结构）

**第 0 篇（免费引流）**：两条命令装好 agent-browser——以及 90% 的人在第二条卡死的原因。放出坑 1 的报错截图和镜像解法的一半，完整步骤在付费版。

**第 1 篇 · 安装避坑全解（付费）**
- npm 安装与换国内源加速；Node 版本要求
- Chrome for Testing 下载失败的三种解法：镜像离线包（附版本查询与各平台路径对照表）、复用本机 Chrome、代理模式
- Linux `--with-deps` 的正确用法与缺库单独补法
- 用 `agent-browser doctor` 自检：每一项 pass/fail 怎么看

**第 2 篇 · 核心闭环：让 AI「看懂」网页**
- open / snapshot / @ref 机制讲透，配本地演示页全程实操
- ref 失效的三种情况与选择器/语义定位备选（`find`）
- session 管理：为什么必须带 `--session`、多任务并行互不干扰

**第 3 篇 · 配方一：自动填表**
- 以订阅表单为例的完整脚本（fill/type/select/check/click + 提交后验证）
- 批量填表：把 CSV 里的数据循环填进后台表单的脚本模板
- 遮挡、验证码、登录态的边界说明（能做/不能做/风险）

- 表格数据从哪来：`read` 先把目标页结构看清再写脚本，别盲填

**第 4 篇 · 配方二：网页巡检**
- 定时打开指定页面，用 snapshot/get 检查关键字、价格、库存状态（如演示页的「缺货」），变化时截图留证
- `screenshot` / `pdf` / 前后 `diff` 的用法；巡检脚本模板（配合系统定时任务）
- 巡检结果怎么交给 AI 判断：read 输出直接喂给 Agent 做摘要与告警文案

**第 5 篇 · 配方三：数据抓取**
- 三档抓法：`read` 整页转文本（最省事）→ snapshot 按 ref 取局部 → `eval` 跑 JS 把表格/列表提取成 JSON（实测演示：商品表三行变 JSON 数组）
- 抓取结果落盘为 JSON/CSV 的脚本模板；分页、多页循环的写法要点
- 合规边界：只抓公开数据、控制频率、遵守目标站点条款，不碰登录后他人数据

**第 6 篇 · 配方四：E2E 测试入门**
- 用「打开—操作—断言」三段式测自家网站：`get`/`is` 校验标题、URL、元素状态，`wait` 等异步内容
- 官方 dogfood skill 做探索性测试找 bug 的流程介绍
- 失败截图与录像（`record`）留证，测试脚本模板一份
- （本篇为配方整理，完整项目级实测在正式版补做，销售页不得宣传为「全量实测」）

**第 7 篇 · 接入篇：让你的 AI Agent 自己会用**
- `npx skills add vercel-labs/agent-browser` / AGENTS.md 写法 / `agent-browser mcp` 三种接法对照
- 给 Claude Code / Codex 的一段可直接粘贴的任务提示词模板（如「用 agent-browser 打开 X，抓 Y 存成 CSV」）
- 多 Agent 并行时的 session 隔离规范

**第 8 篇 · 排错手册**
- doctor 输出逐项解读；坑 1–8 的速查表（报错原文 → 原因 → 解法）；版本升级 `agent-browser upgrade` 与文档以 `skills get core` 为准的原则

**附赠**：填表/巡检/抓取三份脚本模板打包 + 本地演示页源码（让读者先在练习页跑通再上真实站点）。

---

## 五、卖点文案草案与定价（9.9 元）

**标题候选**
- 《让 AI 自己上网干活：agent-browser 场景实战包——填表、巡检、抓取、测网站，四套配方直接抄》
- 《卡在 Chrome 下载那一步？agent-browser 国内安装避坑 + AI Agent 实战配方》

**销售页开头（草案）**
> `npm install -g agent-browser` 谁都会敲。真正劝退人的是下一步：`agent-browser install` 连 Google 下载 Chrome，国内网络转圈 30 秒直接超时——网上没有一篇中文教程讲怎么绕过去。这套实战包的第一篇就解决它：镜像离线安装（我们实测 41 秒下完、一次装通）、复用本机 Chrome、代理模式，三种解法任选。
> 装好只是开始。包里还有四套拿来就能用的配方：自动填表（附批量脚本）、网页巡检（价格/库存变了自动截图留证）、数据抓取（网页表格一键变 JSON/CSV）、E2E 测试（让 AI 替你点一遍自家网站找 bug）。再加一章接入指南，把它接进你的 Claude Code / Codex，之后就是一句话让 AI 自己开网页干活。

**定价**：统一 9.9 元（2026-11-11 用户拍板，全线教程统一价），全部篇章 + 三份脚本模板 + 演示页源码全内容不分档，含一个版本周期内的勘误免费更新。

---

## 六、需截图清单（正式制作时按此拍摄）

1. `npm install -g agent-browser` 成功输出 + `--version` 显示 0.42.0
2. ⭐ `agent-browser install` 超时报错原文特写（坑 1，销售页也要用）
3. `doctor` 三项 fail 的输出（Chrome not found / CDN unreachable）
4. 镜像下载命令与 195MB 文件下载完成画面
5. 解压后放进 `~/.agent-browser/browsers/chrome-<版本号>/` 的目录结构
6. ⭐ `doctor` Chrome 一项变 pass 的对比图（与第 3 张成对，前后对比最有说服力）
7. 本地演示商店页原貌（表格 + 表单）
8. `snapshot -i` 输出：姓名/邮箱/提交按钮带 @ref 的画面
9. 填表后页面回显「提交成功，欢迎你，小明」
10. `eval` 抓取输出的 JSON（三件商品结构化数据）
11. `screenshot` 命令截下的完整页面图
12. `skills get core` 输出开头（证明文档随 CLI 版本自带）
13. （可选）`--with-deps` 卡在 apt 重试的画面（坑 2 佐证）
14. （可选）接入 Claude Code / Codex 后，AI 自行调用 agent-browser 完成一次抓取的对话画面——正式版补测后拍摄

---

## 七、边界与未验证事项（写进教程，也约束销售宣传）

- 本轮实测平台只有 Linux x64 沙箱；Windows / macOS 的镜像路径为按官方文件命名推导，**正式售卖前必须各补测一台真机**，未补测前销售页不得写「全平台实测」。
- 沙箱网络与国内家庭宽带不完全一致：公网站点在沙箱内因代理隧道失败（`ERR_TUNNEL_CONNECTION_FAILED`），未作为产品缺陷记录；国内直连 Google 域名超时与用户实际环境吻合，镜像解法已在同网络验证有效。
- E2E 配方篇基于官方命令体系与 dogfood skill 整理，未做完整项目级实测，已在正文标注。
- 自动化合规：教程只教对自己有权操作或公开可访问的页面做填表、巡检、抓取与测试；不教绕过验证码、批量注册、撞库、抓取他人隐私数据等用途，并提醒遵守目标网站服务条款。
- 版本时效：agent-browser 发版很快，教程须标注适用版本（本草稿基于 0.42.0 / Chrome for Testing 157.0.8103.50），并教读者用 `skills get core` 对照自己版本，这也是勘误免费更新的具体承诺内容。
