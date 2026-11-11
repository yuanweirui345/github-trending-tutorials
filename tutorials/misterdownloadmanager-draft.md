# Mister Download Manager（MDM）中文付费教程草稿 —「下载全能工具包」

> 仓库：anefzaoui/misterdownloadmanager（https://github.com/anefzaoui/misterdownloadmanager）
> 定位：免费开源版 IDM 替代品的中文小白工具包教程，定价 **9.9 元**
> 开源协议：GPL-3.0（卖的是中文教程和配置经验，不是软件本身；软件请读者从官方渠道免费下载）
> 草稿日期：2026-11-10
> **实测状态声明（重要）：本草稿没有经过 Windows 真机实测。** 撰写环境是 Linux 沙箱，无法运行 Windows 的 setup.exe。全文所有安装步骤、界面路径均来自仓库 README、官方文档（docs/）和官网帮助页的逐项核实，属于「文档确认」；凡涉及真实点击效果、中文 Windows 弹窗原文、速度表现的内容，一律标注，发布前必须按文末复核清单在真机上跑一遍并替换截图后才能上架。

---

## 一、核实日志（2026-11-10）

数据源：GitHub 仓库主页与 Releases 页（已连接账号只读核实）、官网 misterdownloadmanager.com 及其功能/帮助页、仓库 docs/getting-started.md 与各 guide 文档、中文关键词搜索。

| # | 核实项 | 结论 | 来源 | 可信度 |
|---|--------|------|------|--------|
| 1 | 最新版本 | **v0.5.2**（Releases Latest）。官网首页同步标注 0.5.2，Windows 安装包 17.1 MB | GitHub Releases Latest 页、官网首页 | 文档确认 |
| 2 | 版本迭代速度 | 仓库 2026-10-29 创建，约 12 天已发 8 个 Release / 9 个 Tag，从筛选时看到的 0.2.4 升到 0.5.2。**教程保鲜期极短，需标注版本号并预留更新机制** | 仓库主页（Created 2026-10-29、Releases 8、Tags 9） | 文档确认 |
| 3 | 热度 | 抓取时 4,948 Star、356 Fork、290 次提交 | 仓库主页 | 文档确认 |
| 4 | 系统要求 | 64 位 Windows 10 / 11，约 100 MB 磁盘空间；另有 Linux 版和实验性 macOS 版（本教程只做 Windows） | 官网、docs/getting-started.md | 文档确认 |
| 5 | 安装包文件名 | `MisterDownloadManager-0.5.2-setup.exe`（安装版）；另有 `-portable.zip`（便携版）和 `-x64.zip`（免安装版） | 官网下载区链接、getting-started 文件表 | 文档确认 |
| 6 | **安装包签名** | **未做代码签名（属实）**。README 原文：Windows 首次运行会提示 "Windows protected your PC"，原因是 "this early version isn't code-signed yet"，官方给的处理是点 **More info → Run anyway**；仓库 Status 一节也明确写 "Not yet: code signing" | README Download 一节、Status 一节 | 文档确认 |
| 7 | 杀毒误报与验真 | v0.5.2 新增了「杀毒/VirusTotal 误报怎么办、如何确认下载的是正版」帮助条目；官网安全页称：每个 Release 由 GitHub 服务器从公开源码构建，官网称每个文件都附 SHA-256 校验值；视频辅助工具在运行前会校验发布者签名/校验和 | v0.5.2 Release Notes、官网 FAQ Is it safe | 文档确认（校验流程本身未实测） |
| 8 | 中文界面 | **有简体中文和繁體中文**，共 86 种语言。安装时可选语言（默认跟随 Windows 显示语言），首次启动（非安装版）会弹语言列表，之后可在 **View → Language** 随时切换。⚠️注意：官方自述多数翻译为机器翻译、尚待母语者校对，中文界面个别术语可能不自然，教程截图时要注意 | README In your language 一节、getting-started | 文档确认（中文文案质量未实测） |
| 9 | 浏览器接管 | 通过官方扩展接管：Firefox 走 addons.mozilla.org，Chrome / Edge / Brave / Opera / Vivaldi 走 Chrome Web Store。安装版在安装时勾选 **Set up browser integration** 会注册浏览器通信小助手（native messaging host）；扩展会把地址、来源页、Cookie、User-Agent 一起交给 MDM，登录后才能下的文件也能接管。MDM 没在运行时，下载会还给浏览器 | README What it does、官网 Browser extension 功能页、帮助页 Browser integration | 文档确认 |
| 10 | 接管的生效条件（坑源） | 帮助页列明四个条件须同时满足：Options → General 里 **Use advanced browser integration** 开启、**Capture downloads from the following browsers** 勾了当前浏览器、Options → File types 的自动下载类型列表包含该文件类型、站点不在排除列表里；另有「小于某大小的文件不接管」的阈值（0 = 不限制）。扩展工具栏按钮带红色 **!** 时表示未连通，点开会说明原因 | 帮助页 Browser integration「What it takes」 | 文档确认 |
| 11 | 便携版限制 | **portable.zip 不能做浏览器接管的安装注册**，浏览器集成需要安装版（setup.exe）；便携版设置和下载列表存在程序旁的 data 文件夹、不写注册表 | getting-started、Browser integration 帮助页 | 文档确认 |
| 12 | 多线程加速 | 把文件分片、多连接同时下载并动态再平衡（某连接下完后接手剩余最大分片的一半，严重落后的连接会被断开重连）。**默认 8 个连接**，Options → Connection 可设 1–32；勾选 **Allow more than 32 connections** 后上限 128。服务器若拒绝多连接，MDM 会自动学到并记在该页 Exceptions 里 | 官网 Acceleration and resume 功能页 | 文档确认（提速幅度未实测） |
| 13 | 官方速度宣称的口径 | README/官网宣称 1 GiB 文件 1.0 秒 vs IDM 6.42 的 5.5 秒，但那是**同一台电脑对本地测试服务器**、交替运行 7 次取中位数的厂商自测。**教程和文案不得把它写成真实网络下的承诺速度**，只能转述为「官方基准测试」并注明测试环境 | README The benchmark in detail | 文档确认 |
| 14 | 断点续传 | 暂停、断网、Windows 或 MDM 崩溃后都能从断点续传；续传前会校验服务器上的文件还是同一个，避免把新文件拼到旧半成品上；每条连接直接写入目标文件的分片文件，没有最后合并步骤。若服务器不支持分片，进度窗口会提前显示 **Resume capability: No** | 官网 Acceleration and resume 功能页 | 文档确认（崩溃续传未实测） |
| 15 | 视频下载与 yt-dlp / ffmpeg 引导 | 内置引导属实，且是**首次启动就弹**：getting-started 写明首次启动后 MDM 会提供两个视频辅助工具——yt-dlp（解析视频页）和 ffmpeg（把画面和声音合并），点 **Yes, download them (recommended)** 后台下载一次、合计约 100 MB；选 Not now 则推迟到第一次下视频时再装。视频教程页给的分项大小：**Install yt-dlp 约 23 MB、Install ffmpeg 约 80 MB**（仅 Windows 需要装 ffmpeg，Linux/macOS 用系统的） | getting-started「The video helpers」、docs/guide-video-from-page.md、官网 Video 功能页 | 文档确认（国内网络下能否顺利下载这两个工具未实测·高风险点） |
| 16 | 没有 ffmpeg 时的降级 | 没有 ffmpeg 仍可下「自带声音的画质」，但拿不到高画质（画音分离的那些）和 MP3 提取；v0.5.2 还修了「只给音频却不说明原因」的问题，现在窗口会提示缺 ffmpeg | 官网 Video 功能页、v0.5.2 Release Notes | 文档确认 |
| 17 | 辅助工具的安全校验与更新 | yt-dlp 按其开发者的签名密钥校验、ffmpeg 按公布的校验和校验后才运行；也可以在 **Helpers...** 里改用自己已有的 yt-dlp / ffmpeg。⚠️MDM 目前**不会自动更新 yt-dlp**，某个站点突然下不了时通常要手动换新版 yt-dlp，帮助页有方法 | 官网 Video 功能页、FAQ Is it safe | 文档确认 |
| 18 | 登录状态视频与 JS 运行时 | 需要登录的视频可通过 Options → Sites Logins → Cookies for video sites 读浏览器 Cookie；但 Chrome / Edge / Brave 在 Windows 上会锁住自己的 Cookie，官方说 Firefox 可用、或用视频上的悬浮按钮。v0.5.2 还提到：yt-dlp 处理某些 YouTube 格式需要 JavaScript 运行时（它默认只找 Deno，装了 Node.js / Bun 时 MDM 会转告它）——这是小白很容易卡住且看不懂报错的一步 | 官网 Video 功能页、v0.5.2 Release Notes | 文档确认（未实测） |
| 19 | 视频功能边界 | 网页视频悬浮 **Download video** 按钮、播放列表/频道批量下、字幕（存旁文件或嵌入 MP4/MKV）、只要声音（MP3 / Opus / FLAC）、图库用 gallery-dl。**DRM 保护的视频下不了，MDM 也不尝试破解**；页面上受保护的流会标 Protected 且没有下载按钮 | README、官网 Video 功能页、Browser integration 帮助页 | 文档确认 |
| 20 | 定时下载 | 队列 + Scheduler：可设开始/停止时间、每天或一次、可唤醒电脑、下载完关机/睡眠/休眠（关机前有 30 秒倒计时可取消）；关主窗口没关系，它会缩到托盘继续跑，但不能用 Tasks → Exit 退出；且电脑不能在开始时间前先睡死 | docs/guide-download-overnight.md | 文档确认（唤醒/关机未实测） |
| 21 | 其他可写进工具包的功能 | 分类自动归档（按类型进不同文件夹）、下完自动解压（zip/7z/rar 等）、粘贴校验和验文件、从 IDM 导出文件导入下载列表、限速（全局/单文件）、批量链接抓取（Link grabber）、磁力/.torrent（选文件下、顺序下）、手机遥控（局域网）、便携版放 U 盘 | README What it does、官网功能索引 | 文档确认 |
| 22 | 中文教程空白度 | 2026-11-10 中文搜索「Mister Download Manager 中文 教程 下载」：前排是官网中文页（zh-hans，官方机器翻译站）、GitHub 官方文档和扩展商店页，**未见中文第三方实测/工具包类教程**。注意：不能宣传「全网没有中文介绍」——官方中文站是存在的，空白点在「中文手把手实测+避坑」这一层 | 中文关键词搜索结果 | 本次搜索确认 |

### 核实结论（给决策用）

- **值得做，但要快且预期管理要诚实**：功能面（加速+续传+浏览器接管+视频+定时+种子）确实撑得起「工具包」形态，定价统一 9.9 元；但版本 12 天从 0.2.4 冲到 0.5.2，教程截图和路径可能几周内过期，必须在教程里标版本、并承诺一个更新周期，否则差评风险高。
- **最大未验证风险是国内网络环境**：首次启动要从官方源下载 yt-dlp+ffmpeg 合计约 100 MB、浏览器扩展要访问 Chrome Web Store——这三件事在国内网络下都可能卡住。它们恰恰是最好的教程卖点（给出可行的替代路径：手动装扩展、改用自备 yt-dlp/ffmpeg），但替代路径本身**未经真机验证前不许写成肯定句**。
- **信任成本是真实存在的**：未签名 + 杀毒误报话题官方都承认了。教程的价值有相当一部分是「教用户安全地验真并放行」，这一章必须写透、且绝不能教人关杀毒/关 SmartScreen。

---

## 二、基于文档确认的安装步骤（Windows 安装版路线）

> 以下步骤译自官方 docs/getting-started.md 与帮助页，菜单英文名按官方原文保留（中文界面的对应译名，真机复核时逐项替换为实际中文文案）。

### 2.1 下载（先验真，再安装）

1. 只从两个官方入口下载：官网 misterdownloadmanager.com 的下载区，或 GitHub 仓库的 Releases 页。不要从任何「下载站」「网盘转载」拿安装包。
2. 选文件：认准 **`MisterDownloadManager-0.5.2-setup.exe`**（版本号以 Releases Latest 为准，教程发布时若已更新请同步改号）。
- 不要误选 portable.zip：便携版做不了浏览器接管注册（见核实 #11）。
- 不要误选 x64.zip：那是免安装版，浏览器接管要手动注册小助手，不适合小白。
3. 核对 SHA-256：官网称每个发布文件都附 SHA-256 校验值。在下载文件夹打开 PowerShell，运行 `Get-FileHash.\MisterDownloadManager-0.5.2-setup.exe -Algorithm SHA256`，与 Releases 页公布值逐字对比，一致才安装。这一步是本教程区别于免费搬运的核心信任动作，要配截图教。

### 2.2 安装向导（官方 5 步）

1. 双击运行 setup.exe。若先弹出 SmartScreen 蓝色拦截，按 2.3 节处理，不要慌、不要去关系统防护。
2. 选语言：安装器默认已选 Windows 的显示语言，MDM 会以你选的语言启动（选简体中文即可）。
3. 选安装范围：
- **仅为我安装（For you only）**：不需要管理员权限，装到 `%LOCALAPPDATA%\Programs\Mister Download Manager`。推荐小白选这个。
- 为所有用户：装到 Program Files，需要管理员权限。
4. 勾选安装选项（官方列出的四项）：
- ☑ **Set up browser integration (Chrome, Edge, Brave, Firefox)** — 浏览器接管，**必须勾**，工具包的一半价值在这里；
- ☑ **Open.torrent files and magnet links with Mister Download Manager** — 想要种子/磁力功能就勾；
- ☐ **Start Mister Download Manager when Windows starts** — 常用下载的可以勾，偶尔用的不勾；
- ☐ **Create a desktop shortcut** — 默认不勾选，想要桌面图标自己勾上。
5. 点 **Install** → **Finish**，MDM 会随即启动。若 MDM 已在运行，安装器会先让它保存并关闭。以后升级可用 Help → Check for updates，或直接跑新版安装包，下载列表和设置都会保留。

### 2.3 Windows 拦截未签名安装包的处理（教程核心章节）

**先讲清原理（安抚+建立信任）**：MDM 目前没有购买代码签名证书，所以 Windows SmartScreen 不认识它，会弹出蓝色窗口 "Windows protected your PC"（中文系统文案）。这不等于软件有病毒，但**任何教程都不该让用户无脑放行**——放行前先做完 2.1 的来源核对和 SHA-256 校验，两个都对上了，才继续：

1. 在蓝色拦截窗口点 **More info（更多信息）**；
2. 核对窗口里显示的文件名确实是刚下载的 setup.exe；
3. 点 **Run anyway（仍要运行）**，进入安装向导。

红线（教程中明写）：
- 不教关闭 SmartScreen、不教关闭 Windows Defender、不教加全盘排除项；
- 若杀毒软件报警，先按 v0.5.2 帮助里的思路处理：核对 SHA-256 是否与官方一致，可把文件哈希拿到 VirusTotal 查一遍再决定；哈希对不上 = 立刻删掉重下，绝不放行；
- 只认官网和 GitHub Releases 两个来源，第三方站点的同名文件一律不要装。

### 2.4 首次启动三件事

1. **语言确认**：安装时选了中文一般直接就是中文界面；若不是，在 **View → Language** 里改。⚠️官方自述翻译多为机器翻译，个别菜单词可能别扭，教程里提前打预防针，并给出「中英对照速查」小表作为工具包附件。
2. **视频小助手弹窗**：首次启动后 MDM 会问要不要下载 yt-dlp 和 ffmpeg（合计约 100 MB，后台下载、只下一次）。
- 选 **Yes, download them (recommended)**。只下普通文件、从不下视频的人可以选 Not now，以后第一次下视频时还会再问。
- ⚠️这一步要从官方源下载，国内网络可能很慢或失败。教程需给出 Plan B 并真机验证后才能写死：① 在视频窗口单独点 Install yt-dlp（约 23 MB）/ Install ffmpeg（约 80 MB）重试；② 用自己已有的 yt-dlp / ffmpeg，在 **Helpers...** 里指定给 MDM 用（官方支持此路径）；③ 只用普通文件下载功能，视频功能暂缓。
3. **主窗口认路**：菜单栏（Tasks / File / Downloads / View / Help）、工具栏（Add URL、Add Torrent、Resume、Stop、Options、Scheduler…）、左侧分类、右侧下载列表，四块认清即可，不要逐项讲。

### 2.5 装浏览器扩展（接管的第二半，漏了就没效果）

安装版只注册了「小助手」，每个浏览器还要单独装扩展，MDM 每次启动会弹窗列出检测到的浏览器并给出商店入口（可选 Don't ask again 关掉提醒）：

- **Firefox（需 140 或更高版本）**：在 addons.mozilla.org 的 Mister Download Manager 页点 **Add to Firefox**；
- **Chrome**：在 Chrome Web Store 同名扩展页点 **Add to Chrome**；
- **Edge**：按钮叫 **Get**，会先要求 **Allow extensions from other stores（允许来自其他商店的扩展）**，点 Allow 后再 Add extension；
- **Brave / Opera / Vivaldi**：同样用 Chrome Web Store 的那个扩展。
- ⚠️打不开 Chrome Web Store 的用户：用安装目录里的扩展文件手动加载——打开 `chrome://extensions`（Edge 是 `edge://extensions`），开 **Developer mode（开发者模式）**，点 **Load unpacked（加载已解压的扩展程序）**，选安装目录 `extension` 文件夹下的 `chromium` 子文件夹（仅自己安装时在 `%LOCALAPPDATA%\Programs\Mister Download Manager\extension`，为所有用户安装时在 Program Files 对应目录）；每个 Release 也单独提供 `...-extension-chromium.zip`（先解压再加载）。装好后把扩展按钮固定到工具栏，看它是否显示已连通（红色! = 未连通）。
- 装完自检：随便找个.zip 直链点一下，若弹出 MDM 的 **Download File Info** 窗口而不是浏览器自带下载，就是接管成功。

### 2.6 第一次下载（普通文件）

译自官方 guide-first-download.md：

1. 在浏览器里右键文件链接 → 复制链接；
2. 在 MDM 点工具栏 **Add URL**（或按 Ctrl+N），地址一般已自动填好（它会读剪贴板）；
3. 点 OK，**Download File Info** 会显示文件名、大小和保存位置——MDM 按类型自动分文件夹（如 ZIP 去 `Downloads\Compressed`、视频去 `Downloads\Video`），可在这里改文件夹或分类；
4. 点 **Start Download**，进度窗口能看到速度、剩余时间和每条连接在干什么；
5. 完成后弹 **Download complete**，点 Open 打开文件或 Open folder 打开文件夹。

### 2.7 第一次下视频（带 yt-dlp / ffmpeg 的完整链路）

译自官方 guide-video-from-page.md（只下你有权下载的内容，见合规一节）：

1. 在浏览器打开视频页，复制地址栏链接；
2. MDM 里 **Add URL**（Ctrl+N）→ OK。MDM 识别出是视频页后会自动打开 **Download video from a page** 窗口（也可以走 Tasks → Download video from a page...，粘贴后点 Look up）；
3. 第一次用时若还没装 yt-dlp，点 **Install yt-dlp**（约 23 MB），MDM 从官方发布下载并校验后才用；
4. 等解析完成，画质列表按从好到差排列、标有大小，末尾是 Audio only；
5. 想要最高画质或 MP3，就需要 **ffmpeg**（画面和声音是分开存的，要靠它合并）：Windows 上 MDM 会提供 **Install ffmpeg**（约 80 MB）。不装就只能选自带声音的那些画质；
6. 选画质，确认 **Save to** 和 **File name**（默认用视频标题命名）；
7. 要字幕就在 **Subtitles** 里选语言，可存成旁文件或勾选嵌入进 MP4/MKV；
8. 点 **Download Now** 立即下，或 **Later** 先放进列表。之后它和普通下载一样出现在列表里。
9. 播放列表/频道：粘贴播放列表地址，窗口会列出全部视频（默认全勾），统一选一个画质，下载到以列表命名的文件夹；频道是每 50 个一批加载。

---

## 三、坑清单（每条标注核实状态）

标记说明：= 官方 README/文档/帮助页明写；= 步骤或现象未在 Windows 真机验证，发布前必须复核，不得对读者写成亲测结论。

1. **SmartScreen 蓝屏拦截，以为是病毒不敢装**
现象：双击 setup.exe 弹 "Windows protected your PC"。原因：安装包未做代码签名。解法：先核来源+SHA-256，再 More info → Run anyway（见 2.3）。严禁教关 SmartScreen/杀毒。
2. **杀毒软件误报**
现象：个别杀毒对未签名的新程序报警。原因：无签名+下载器行为特征。解法：v0.5.2 已内置相关帮助；核对 SHA-256、必要时查 VirusTotal；哈希不一致直接删除重下。
3. **选错文件：下了便携版，发现浏览器不接管**
原因：portable.zip 无法注册浏览器小助手。解法：卸载便携版改装 setup.exe；或只把便携版当 U 盘单机下载器用，别指望接管。
4. **只装了主程序、没装扩展，接管完全不生效**
原因：接管是「小助手 + 扩展」两件套。解法：按 2.5 给每个常用浏览器装扩展；MDM 启动弹窗就是干这个的，别手贱点 Don't ask again 又找不到入口（扩展商店搜 Mister Download Manager 也可）。
5. **扩展工具栏按钮有红色!**
含义：浏览器没连上 MDM。解法：点按钮看它给的原因和 How to fix it；常见是 MDM 没在运行、或安装时没勾浏览器集成（重跑安装包勾上）。
6. **某些文件接管、某些不接管，像坏了一样**
原因：四个条件任一不满足就还给浏览器——文件类型不在自动下载列表、站点在排除列表、文件小于「不接管」阈值、扩展被设为仅在点击时读取站点（那样传不了登录状态）。解法：对照帮助页条件逐项查 Options → General / File types；任何文件都想强制接管时，用右键 **Download with Mister Download Manager**（不受类型和排除列表限制）。
7. **Edge 装扩展先弹「允许来自其他商店」**
解法：点 Allow → 再点 Get/Add extension，不是出错，别在这里放弃。
8. **国内打不开 Chrome Web Store**
解法：开发者模式 Load unpacked 加载安装目录 extension\chromium（见 2.5）。注意此路扩展不会自动更新，升级 MDM 后可能需要重新加载——。
9. **首次启动的 100 MB 小助手下载卡住/失败**
原因推测：yt-dlp/ffmpeg 从官方源下载，国内网络不稳定。解法框架见 2.4 的 Plan B（单独重试、Helpers... 指定自备副本）。**此条的每种解法都必须真机验证后才许写进正式教程。**
10. **下视频只给音频、或最高画质不见了，却不报错**
原因：没装 ffmpeg 时画音分离的画质不可选。解法：装 ffmpeg（约 80 MB）；v0.5.2 起窗口会直接提示缺 ffmpeg，若读者界面没提示，让其先升级版本。
11. **站点改版导致某个视频突然下不了**
原因：yt-dlp 需要跟进站点变化，而 MDM 不自动更新 yt-dlp。解法：按帮助页方法手动换新版 yt-dlp，或在 Helpers... 里指向自己维护的新版。
12. **需要登录的视频解析失败**
原因：未给 Cookie，或浏览器 Cookie 被锁（Chrome/Edge/Brave 在 Windows 上会锁）。解法：Options → Sites Logins → Cookies for video sites 配置；改用 Firefox 取 Cookie，或用网页上视频悬浮按钮发起（该路径可绕过此限制，官方原话）。
13. **某些 YouTube 格式报「格式不可用」**
原因链：登录 Cookie 让 yt-dlp 只用需要 JS 运行时的播放器，而机器上没有 Deno/Node/Bun。解法：装 Node.js 或 Bun 后重试；v0.5.2 已做无 Cookie 重查兜底，先升级再排查。
14. **开满连接并不更快，还可能被服务器限速/拒绝**
原因：提速只在「服务器对单连接限速」时成立；多数服务器只允许少量连接。解法：默认 8 个就够用；别一上来拉满 32/128；MDM 会把学到的服务器上限记在 Options → Connection 的 Exceptions 里，不要乱删。
15. **以为能续传，服务器根本不支持**
解法：开下前看进度窗口的 **Resume capability**，显示 No 就是不支持分片，断了只能重下——大文件先确认再走人。
16. **定时下载到点没启动**
常见原因：用了 Tasks → Exit 把 MDM 真退出了（关窗口只是缩托盘，没事）；或电脑在开始时间前已睡眠且唤醒没成功。解法：只关窗口不退出、电源设置里别让它在开始前睡死；关机结束选项有 30 秒倒计时，人在电脑前能取消。
17. **下完找不到文件**
原因：MDM 按类型自动分文件夹（Compressed / Video 等），不是全扔进浏览器下载夹。解法：在 Download File Info 里改保存位置；或完成弹窗点 Open folder；教程里给出分类文件夹对照表。
18. **中文界面个别词看不懂**
原因：官方自述翻译多为机器翻译。解法：教程附「中英菜单对照速查」；教读者实在找不到某功能时切回英文界面按英文名找（View → Language）。
19. **教程刚买就过期：版本更新太快**
12 天 0.2.4 → 0.5.2。解法（对卖方）：教程标注适用版本、页面路径以功能名而非截图位置为准、承诺至少跟一轮大版本更新答疑；Help → Check for updates 升级后列表和设置不丢，可放心让读者升级。
20. **32 位系统 / 老 Windows 装不了**
只支持 64 位 Windows 10/11。购买页须提前写明系统要求，避免退款纠纷。

---

## 四、教程大纲 —「下载全能工具包」（定价 9.9 元）

**一句话定位**：软件免费，9.9 元买的是「安全下对、一次装好、接管+视频+定时全配置完、踩坑有人替你踩过」的中文手把手包。

- **第 0 章 先说清三件事**（免费试读建议）：它是什么/不是什么（独立开源项目，与 IDM 无隶属、不是破解版）；系统要求（64 位 Win10/11）；本教程适用版本（0.5.2）与更新说明；合规声明（只下你有权下载的内容）。
- **第 1 章 安全下载与验真**：两个官方入口、三个文件的区别、SHA-256 核对图文、SmartScreen 正确放行、杀毒误报处理。——本包的信任核心章。
- **第 2 章 安装与首次启动**：安装向导五步逐图、安装范围选择、四个勾选项怎么选、语言设置、100 MB 小助手弹窗怎么答（含下载失败的 Plan B）。
- **第 3 章 浏览器接管全配置**：Chrome / Edge / Firefox 各自装扩展图文、打不开商店时的手动加载法、接管四条件自检表、右键强制接管、红色! 排障。
- **第 4 章 普通下载与提速设置**：Add URL 全流程、分类文件夹规则、连接数怎么设才真有用（默认 8 的道理）、限速、批量链接抓取。
- **第 5 章 断点续传实战**：Resume capability 怎么看、暂停/断网/重启后续传、链接过期怎么办、校验和验文件。
- **第 6 章 视频下载全配置**：yt-dlp / ffmpeg 作用与安装、选画质、字幕、只要 MP3、播放列表批量下、登录视频与 Cookie、常见报错对照（只给音频/格式不可用/站点失效）。
- **第 7 章 定时与自动化**：队列、Scheduler 定时开始/停止、完成关机、关窗口与真退出的区别、手机局域网遥控（加餐）。
- **第 8 章 进阶加餐**：磁力/.torrent 选文件下、图库批量下、下完自动解压、从 IDM 导入列表、便携版放 U 盘的正确用法与限制。
- **附录 A 故障速查表**：坑清单 1–20 的一页表格式整理（现象→先查什么→解法所在章节）。
- **附录 B 中英菜单对照速查**：高频菜单/选项的中英对照（机器翻译不准时的救命表）。
- **附录 C 配置推荐表**：三类人（普通下文件 / 经常下视频 / NAS·定时党）的 Options 推荐值一览。

**工具包交付物清单**：图文教程正文 + 附录 A/B/C 三张可打印速查表 + 安装前检查清单（系统/网络/浏览器版本） + 版本更新日志页（承诺跟版更新）。

---

## 五、卖点文案草案（合规版）

### 5.1 主文案（知识星球 / 小报童详情页）

> **IDM 要收费，浏览器下载又慢又爱断——这个免费开源的平替，我帮你一次装明白。**
>
> Mister Download Manager（MDM）最近在 GitHub 上爆火：免费、开源、无广告，多线程加速、断网自动续传，还能接管浏览器下载、下网页视频、定时夜里慢慢下。用 IDM 的老用户打开它会觉得格外眼熟。
>
> 但小白装它有三道坎：① 官网和商店全是英文，三个安装文件不知道选哪个；② 它没买代码签名，Windows 会弹蓝色拦截，杀毒还可能误报，不敢点；③ 浏览器扩展、视频小助手（约 100 MB）这两步，国内网络下最容易卡住，卡住还没人告诉你怎么办。
>
> 这份「下载全能工具包」就是替你过这三道坎的：从**验真下载**（教你核对官方 SHA-256，确认是正版才放行）到**安装五步**、**Chrome/Edge/Firefox 接管设置**、**视频下载全配置**、**定时下载**，全程中文图文，附三张速查表：故障速查、中英菜单对照、三类人配置推荐。
>
> 软件本身免费、官方下载，本教程不卖软件，卖的是让你少折腾两小时的中文路线图。定价 9.9 元，适用版本 0.5.2，后续跟版更新。
>
> 系统要求：64 位 Windows 10 / 11。请只下载你有权下载的内容（自己购买的课程、公开资料、开源软件等）；受版权或 DRM 保护的内容，本工具和本教程都不提供下载方法。

### 5.2 短文案（一句话版）

> 免费开源的 IDM 平替 MDM，中文手把手装到能用：验真、过拦截、浏览器接管、视频配置、定时下载，一套 9.9 元。

### 5.3 合规红线（文案禁用清单）

- ❌ 不写「全网最快 / 保证 5 倍速度」——官方 1.0s vs 5.5s 是本地测试服务器的厂商基准，只能注明出处地转述。
- ❌ 不写「任何视频都能下 / 免费下付费课程 / 破解 DRM」——官方明确 DRM 不下；只写「下你有权下载的视频」。
- ❌ 不写「种子/磁力资源大全、下片神器」——种子功能只描述为「支持磁力与.torrent 文件管理」，不提供、不暗示任何资源。
- ❌ 不写「绕过会员/绕过付费墙」——登录视频功能只表述为「用你自己的登录状态下载你已购买/有权访问的内容」。
- ❌ 不写「官方中文教程」冒充官方——要写明是第三方中文教程；也不写「全网没有中文介绍」（官网有中文站），卖点是「中文实测避坑」这一层（注：真机复核完成前，详情页不得使用「实测」二字，先用「逐项核对官方文档」）。
- ❌ 不教关闭杀毒/SmartScreen，不提供任何「报毒解决=加白名单全盘」式危险操作。

---

## 六、需截图清单（全部需 Windows 真机拍摄，不可直接搬官方图商用）

> 说明：官网/GitHub 的官方截图仅作版式参考。付费教程应使用自拍截图（避免图片授权争议，也才配叫手把手）。以下每张都标了拍摄要点；带 ⚠️ 的是决定教程可信度的关键图。

1. ⚠️ Releases 页三个文件并排（setup.exe / portable.zip / x64.zip），红框标 setup.exe
2. PowerShell 里 Get-FileHash 的输出与官网校验值对照（可打码部分字符但保留比对方法）
3. ⚠️ SmartScreen 蓝色拦截窗口（中文系统原文）→ More info 展开 → Run anyway，三连图
4. 杀毒软件若报警的实际弹窗（遇到了必拍；没遇到则此图缺省并在文中注明因杀毒版本而异）
5. 安装向导：语言选择页
6. 安装向导：仅为我 / 为所有用户选择页
7. ⚠️ 安装向导：四个勾选项页（标注每个勾的建议）
8. 首次启动：语言列表（如出现）
9. ⚠️ 首次启动：yt-dlp / ffmpeg 下载询问弹窗 + 下载中状态 + 失败时的样子（若复现失败，失败图最值钱）
10. 主窗口全景（中文界面），标注四区：菜单/工具栏/分类/列表
11. View → Language 菜单展开（证明可切换，并拍一两个机器翻译别扭的菜单词做对照表素材）
12. 启动时的「添加浏览器扩展」弹窗（列出浏览器那张）
13. Chrome Web Store 扩展页 Add to Chrome 按钮
14. Edge 的 Allow extensions from other stores 提示 + Get 按钮
15. Firefox Add-ons 页 Add to Firefox 按钮
16. chrome://extensions 开发者模式 + Load unpacked 选 extension\chromium 的手动加载全过程（2–3 张）
17. 扩展工具栏按钮正常态 vs 红色! 异常态对比
18. ⚠️ 接管成功瞬间：点.zip 链接弹出 MDM 的 Download File Info
19. Download File Info 窗口（文件名/大小/分类文件夹）
20. 下载进度窗口：多条连接的分片条 + 每连接状态表
21. Options → Connection 页：默认 8、范围 1–32、Allow more than 32 的勾选位置、Exceptions 列表
22. Resume capability 显示 Yes / No 的两处进度窗口细节
23. 大文件下载中途杀掉进程/断网后，重启续传前后的进度对比（两张）
24. 视频：网页上悬浮的 Download video 按钮
25. Download video from a page 窗口：画质列表 + Audio only 行
26. ⚠️ 视频窗口里的 Install yt-dlp / Install ffmpeg 按钮与安装过程
27. 没装 ffmpeg 时画质变少/只剩音频的对比图（能复现则拍）
28. 字幕选择与「嵌入视频」勾选处；播放列表窗口（全勾+统一画质）
29. Scheduler：开始/停止时间、Daily、完成后关机的设置页（两张）
30. 关窗口后托盘图标还在运行的状态
31. 分类文件夹实际落盘效果（Downloads 下的 Compressed / Video 等）
32. 从 IDM 导入列表的菜单路径（如复核者装有 IDM，可选拍）

---

## 七、Windows 真机复核清单（发布前必须全部打勾）

- [] 在中文 Windows 11（和尽可能一台 Win10）上完整跑 2.2 安装，记录 SmartScreen 中文弹窗原文并替换 2.3 文案、补图 3
- [] 核对 v0.5.2（或发布时最新版）setup.exe 的 SHA-256 公布位置，确认 2.1 的验真步骤读者能照做
- [] 确认安装器默认语言是否跟随中文系统、安装后首屏是否中文；记录机器翻译明显别扭的菜单词，填进附录 B
- [] yt-dlp + ffmpeg 首次下载在国内家庭宽带下实测：时长、是否成功；失败时把 Plan B 三条逐条验证（单独 Install、Helpers... 指定自备副本的完整路径）
- [] Chrome / Edge / Firefox 三浏览器各完成一次接管：装扩展 → 点真实.zip 直链 → 确认弹出 Download File Info；Edge 的 Allow 步骤截图
- [] Chrome Web Store 打不开的场景下，走通手动 Load unpacked 路径，并确认 MDM 升级后扩展是否需重装
- [] 续传实测：大文件下到一半分别做暂停、断网、结束进程、重启电脑四种打断，确认都能续传；找一个不支持分片的服务器验证 Resume capability: No 的提示
- [] 连接数实测：同一文件 8 / 32 两档的速度差异（只记录自家网络结果，文案仍不得承诺提速倍数）
- [] 视频实测：公开视频页走通悬浮按钮与 Add URL 两条路；确认装 ffmpeg 前后画质列表差异；字幕下载与 MP3 提取各成功一次（素材用无版权争议的公开视频）
- [] 定时实测：设一个 10 分钟后的队列任务，确认自动启动；完成关机只测到 30 秒倒计时出现即取消，不必真关机
- [] 复核完成后，把正文中所有标记逐条销项，并在详情页把「逐项核对官方文档」改为如实表述（如「已在 Windows 11 真机完整走通」并注明测试日期与版本）

---

## 八、信息来源（核实当日 2026-11-10）

- 仓库主页与 README：https://github.com/anefzaoui/misterdownloadmanager
- 最新版 Release（v0.5.2）：https://github.com/anefzaoui/misterdownloadmanager/releases/latest
- 官方入门文档：https://github.com/anefzaoui/misterdownloadmanager/blob/main/docs/getting-started.md
- 第一次下载 / 视频 / 浏览器接管 / 夜间下载教程（官方 docs 内 guide 系列）：https://github.com/anefzaoui/misterdownloadmanager/blob/main/docs/guide-first-download.md 等
- 官网首页（版本、安装包、功能索引）：https://misterdownloadmanager.com/
- 官网功能页：Acceleration and resume、Video downloads、Browser extension（misterdownloadmanager.com/features/ 下）
- 官网帮助页：Browser integration（misterdownloadmanager.com/help/browser-integration.html）
- 官网 FAQ：Is it safe、Which browsers does it work with（misterdownloadmanager.com/faq/ 下）
- 中文空白度：中文关键词「Mister Download Manager 中文 教程 下载」搜索结果（前排为官网中文站与官方文档，无中文第三方教程）
