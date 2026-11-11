# GBrain（garrytan/gbrain）中文付费教程草稿

> 状态：实测草稿，未发布、未上架。实测日期：2026-11-10。教程定位：卖「实测经验 + 个人知识库搭建」，不卖开源软件本身。
> 仓库：https://github.com/garrytan/gbrain ｜ 作者：Garry Tan（Y Combinator 总裁/CEO）｜ 协议：MIT（可自由自托管，教程须保留原作者与协议说明）
> 实测版本：gbrain 0.60.645.0（commit 358e347）｜ 运行环境：Linux x86_64、Bun 1.4.4、Node v24.22.1、隔离 HOME 沙箱

---

## 一、实测结论（先说跑通到哪一步）

| 步骤 | 结果 | 说明 |
|---|---|---|
| `bun install -g github:garrytan/gbrain` | ✅ 成功（第二次） | 42 秒，180 个包；第一次在 512MB 的 /tmp 里跑失败（ENOSPC，见坑 4）。官方现行 README 推荐带通道后缀的 `bun install -g github:garrytan/gbrain#stable`，本实测按原命令跑通，二者差异要写进教程 |
| 安装后检查 | ⚠️ 有坑 | 安装日志明确出现 `Blocked 1 postinstall`——Bun 会拦截全局安装的 postinstall，这正是官方 #218 坑的源头 |
| `gbrain init --pglite --no-embedding` | ✅ 成功 | PGLite 本地脑建好，schema v297，自动应用 292 条迁移，`doctor` 100/100；但提示还有 5 条 setup 迁移待手动补跑 |
| `gbrain apply-migrations --yes --no-autopilot-install` | ✅ 成功 | 5 条（0.11.0 / 0.14.0 / 0.22.4 / 0.28.0 / 0.53.0）全部补完，退出码 0；其中 0.53.0 留了一项 shared-skills 的「host action required」，不处理不影响基础记忆，但官方流程没算完全闭环 |
| 跨会话记忆最小例子 | ✅ 跑通 | 会话 1 导入 1 篇笔记 + `remember` 存 1 条事实；换一个全新进程（会话 2）用 `search`/`recall`/`get` 全部原样读回 |
| 自然语言提问（无 Key） | ❌ 不通（符合官方设计） | `gbrain query "我的教程项目代号是什么…"` 返回 No results（keyword_zero）；`gbrain think` 明确报无 LLM、无 ANTHROPIC_API_KEY，只给证据不给合成答案。**关键词记忆 ≠ 脑子，这是教程必须讲清的最大预期差** |
| 接入 Claude Code / Codex（MCP） | ⏸ 未实测 | 沙箱里没有这两个客户端，只核对了官方命令（`gbrain setup claude-code` / `codex`、`gbrain serve --surface verbs`）；教程发布前需在一台装了客户端的机器上补测并截图 |
| 语义搜索 / Supabase | ⏸ 未实测（无 Key） | 只核对官方文档与价格（见第五节），未充值、未填任何 Key |

**一句话结论**：无 Key 的 PGLite 记忆层是真能跑通的，30 分钟内可完成；但「让 AI 用人话回答你过去的事」这一步必须要嵌入 Key（语义）+ 聊天模型 Key（合成），教程的付费分层正好卡在这里：免费能演示，付费教进阶。

---

## 二、实测日志（可直接改编为教程正文）

### 2.1 环境检查
```bash
bun --version # 1.4.4 —— 官方要求 Bun 1.4.4 或更新，低于此版本每条 gbrain 命令都会拒绝并提示升级
node --version # v24.22.1（非必需，排查用）
```

### 2.2 安装
```bash
bun install -g github:garrytan/gbrain
# 实测输出节选：
# bun add v1.4.4
# Resolved, downloaded and extracted [702]
# installed gbrain@github:garrytan/gbrain#358e347 with binaries: - gbrain
# 180 packages installed [42.10s]
# Blocked 1 postinstall. Run `bun pm -g untrusted` for details. ← 别慌，见坑 3
gbrain --version # gbrain 0.60.645.0
```
注意：为隔离实测，本轮把 HOME 指到独立沙箱目录，`gbrain` 二进制落在 `$HOME/.bun/bin/gbrain`。普通用户用自己的 HOME 即可，但要确认 `$HOME/.bun/bin` 在 PATH 里，否则会「装上了却敲不出命令」。

### 2.3 初始化（无 Key / PGLite）
```bash
gbrain doctor
# 未建脑时输出：Error [no_brain]: No brain configured. Fix: gbrain init --pglite --no-embedding

gbrain init --pglite --no-embedding
# 实测输出节选：
# Setting up local brain with PGLite (no server needed)...
# Setting up brain schema (v297)... 292 migration(s) applied
# [init] Migrations: 14 recorded as not needed on a new brain;
# 5 setup migration(s) pending (0.11.0, 0.14.0, 0.22.4, 0.28.0, 0.53.0).
# Finish setup: gbrain apply-migrations --yes --no-autopilot-install
# Brain ready at ~/.gbrain/brain.pglite — 0 pages. Engine: PGLite (local Postgres).
# Skills: 75 loaded

gbrain apply-migrations --yes --no-autopilot-install # 补跑上面 5 条，退出码 0
gbrain doctor # Overall health score: 100/100（空脑时）
```

初始化时它还会当场抛一个选择题：搜索模式 conservative / balanced / tokenmax。无 Key 时自动暂定 conservative，并给出成本矩阵（10K 次查询/月，下游模型读检索结果的输入成本）：

| 模式（单次预算） | Haiku 级 | Sonnet 级 | Opus 级 |
|---|---|---|---|
| conservative（4K） | $4/月 | $80/月 | $160/月 |
| balanced（10–12K） | $10/月 | $200/月 | $400/月 |
| tokenmax（20K/无上限） | $20/月 | $400/月 | $800/月 |

（此表为实测版本 init 时打印的估算，模型标价随版本变化，教程里注明「以你安装时终端打印为准」。嵌入、重排、查询扩展是另一笔服务商费用，不含在内。）

### 2.4 跨会话记忆最小例子（本教程的演示主线）

会话 1（写入）：
```bash
mkdir -p notes
# notes/yuan-profile.md 内容：
# # Yuan 测试笔记
# 我的项目代号是蓝月亮计划，目标是给每个 AI Agent 装上长期记忆。
# 我偏好用中文写教程，定价习惯是 69 到 99 元。

gbrain import./notes/ --no-embed
# Import complete (1.2s): 1 pages imported, 1 chunks created, 0 errors

gbrain remember "教程项目代号是蓝月亮计划，偏好中文教程定价69到99元" --entity yuan
# ❌ 第一次这样写会报错：Error [invalid_params]: Missing required parameter: provenance

gbrain remember "教程项目代号是蓝月亮计划，偏好中文教程定价69到99元" \
--entity yuan \
--provenance "实测会话1，用户测试笔记 notes/yuan-profile.md，2026-11-10" \
--content-origin tool_output
# ✅ remembered as fact #1 / entity: yuan
# note: no embedding provider — duplicate detection degraded（无嵌入时只能查完全重复，近义重复会存两条）
```

会话 2（全新进程、换到无关目录、零重新导入 —— 模拟「下一次打开 AI」）：
```bash
gbrain search "蓝月亮计划" --json
# ✅ 命中 yuan-profile，chunk_text 原文返回，evidence: keyword_exact

gbrain recall yuan
# ✅ 📌 id=1 [yuan] 教程项目代号是蓝月亮计划，偏好中文教程定价69到99元 (fact, conf 1.00)

gbrain get yuan-profile
# ✅ 完整读回笔记原文（含 frontmatter：type: note）

gbrain query "我的教程项目代号是什么，定价习惯是多少？"
# ❌ No results. (degraded: embed_unavailable, keyword_zero)
# —— 换个说法就找不到了：关键词检索不是语义检索
```

---

## 三、坑清单（教程核心卖点，按小白踩中顺序排）

1. **npm 同名陷阱**：GBrain 不在 npm 分发，npm 上的 `gbrain` 是无关包。`npm install -g gbrain` / `bun add -g gbrain` 会装错还可能盖掉真二进制。只认 `bun install -g github:garrytan/gbrain`（官方现行推荐 `#stable` 通道）或 `git clone + bun install && bun link`。装错的解法：先卸载错包再重装，`gbrain doctor` 能识别。
2. **Bun 版本门槛**：必须 Bun 1.4.4+，旧版先 `bun upgrade`。教程第一步就要教查版本，否则后面全是看不懂的拒绝信息。
3. **Blocked postinstall 与 #218 schema 坑（官方已知）**：Bun 会拦截全局安装的 postinstall（实测日志实锤 `Blocked 1 postinstall`），导致 schema 迁移可能没自动跑，表现为 `gbrain doctor` 报 `schema_version: 0` 或 “Schema not migrated”。官方解法只有一条：`gbrain apply-migrations --yes --no-autopilot-install`。退出码 0 = 已到最新；退出码 1 且提示 `migrations_pending` = 还卡着，**不要反复再跑 `--yes`**，改跑 `gbrain doctor --json` 看它给的 fix；若原因是 `wedged`（同一条迁移连续失败 ≥3 次），重装没用，要按 doctor 给的修复处理根因后跑 `gbrain apply-migrations --force-retry <version>`。跨 v0.53.0 升级还要先 `gbrain apply-migrations --dry-run --json` 看 shared-skills 迁移说明。
4. **磁盘空间**：依赖 180 个包、约 570MB 起。实测第一次把沙箱放在 512MB 的 /tmp 里直接 ENOSPC 失败（一堆 `error: ENOSPC extracting tarball`，含关键包 `@electric-sql/pglite`）。教用户装到空间充足的磁盘，别装进小容量的临时盘/系统盘红线区。
5. **新脑也要补迁移**：全新 `init` 后仍有 5 条 setup 迁移 pending（实测：0.11.0、0.14.0、0.22.4、0.28.0、0.53.0），不补跑，后台 Minions、Takes/Think 等功能不完整。补跑命令在 init 输出里，多数人会划过去没看见。另：0.53.0 补完仍留 shared-skills 的 host action 提示，基础使用不受影响，别被吓到去删库重装。
6. **`remember` 必填出处**：存事实必须带 `--provenance`（来源说明），建议同时带 `--content-origin`（user_said / tool_output / inferred）。这是 GBrain 的「事实带来源、可纠正可撤回」设计，不是 bug；`gbrain forget <fact-id>` 可撤回。
7. **无嵌入时去重降级**：无 Key 时 `remember` 只查完全重复，换个说法存同一件事会存出两条。习惯：存之前先 `recall <entity>` 看一眼。
8. **关键词搜不到同义说法（最重要的预期管理）**：无 Key 模式是 tsvector 关键词检索。实测搜「蓝月亮计划」秒中，自然语言问句零结果。教程话术必须诚实：无 Key 版是「记得住、按词找得回」，想要「换个说法也懂」必须开嵌入（下一节）。
9. **搜索模式别乱选 tokenmax**：init 时会让选模式，三档在 10K 查询/月下下游成本差可达 200 倍（见 2.3 表）。新手默认 conservative，觉得召回不够再升 balanced；改法：`gbrain config set search.mode <模式>`，随时用 `gbrain search modes` 看当前生效值。
10. **`doctor` 分数会「变差」吓人**：空脑 100/100，导入 1 篇孤立笔记后 brain_score 掉到 25/100（embed 0/35、links 0/25、orphans 0/15 等维度扣分）。这不是坏了，是在提示你：还没开嵌入、页面之间还没建链接。教程要专门解释分数构成，避免用户以为安装失败去删 `~/.gbrain`——官方明确说维度不匹配等问题也不需要删库，`doctor` 会给修复命令。
11. **嵌入维度一旦定下不能改配置硬切**：换嵌入服务商/模型时若维度不同（如 Voyage 1024 vs OpenAI 1536），`gbrain config set embedding_model` 会被拒绝，必须走 `gbrain migrate embeddings --to <provider>:<model> --dim <N>`（可 `--dry-run` 预览成本、可续跑）；首次导入报 `expected N dimensions, not M` 同理，照 doctor 给的命令修，不要删库。
12. **PGLite 的边界**：本地脑适合个人、约 50K 页以内、单机使用；后台 Minions 守护进程需要真 Postgres，PGLite 只支持内联执行（`submit --follow`）。要多设备/团队/24 小时在线，就得毕业迁移：`gbrain migrate --to supabase`（先出 plan、核对 hash 再 `--yes --expect <plan_hash>`，不是一条命令闷头跑）。
13. **历史迁移暗坑（老用户升级才遇得到，教程附录提一句即可）**：v0.12.2 之前的 Postgres/Supabase 老脑升级时有 JSONB 双重编码问题，官方会在 `post-upgrade` 时自动跑 `gbrain repair-jsonb`；gbrain-base 旧 schema 包升级到 gbrain-base-v2 要走 `jobs submit unify-types` 且默认 dry-run，确认后再加 apply。原则：升级先看 `skills/migrations/v<版本>.md` 再动手。

---

## 四、个人知识库搭建（教程的第二卖点）

1. **页面形状**：默认 schema 包 `gbrain-base-v2`，15 类（person / company / project / concept / source / email / note 等），用目录前缀+frontmatter 决定类型；不合口味可用 `gbrain schema detect → suggest → review-candidates` 按你现有文件夹反推，或 `gbrain schema use gbrain-recommended` 换扩展包。换包是重新解释，不是搬家，换回去不丢数据。
2. **三种喂数据方式**：① 随手记事实 `remember`（带出处）；② 整夹导入 `gbrain import ~/notes/ --no-embed`（Obsidian/Markdown 笔记友好）；③ 页面直写 `gbrain put <slug>` + `timeline-add` + `link` 建人物-公司-项目关系，关系建起来后才能玩图谱问答（如 “who works at X”）。
3. **让 Agent 真用上**：光装 CLI 不等于你的 AI 会主动查脑。官方路径是 `gbrain setup claude-code`（或 codex）一步写入 MCP 入口（`gbrain serve`）+ brain-first 指令 + 钩子并做冒烟测试；其他支持 stdio MCP 的客户端手动配 `"$(command -v gbrain)" serve --surface starter --advertise verbs`。教程要强调验证动作：在新会话里让 Agent 复述一条你只存进脑里的事实，复述对了才算接通。
4. **维护节奏**：`gbrain doctor` 定期体检；页面多了之后再考虑 autopilot/dream 夜间整理（去重、修引用、找矛盾）——那属于进阶篇，且需要模型 Key 支撑，别在基础篇承诺。

---

## 五、进阶功能与真实成本（全部来自官方文档，未实测充值）

| 进阶项 | 需要什么 | 官方价格/说明 |
|---|---|---|
| 语义检索（换说法也能找） | 嵌入 API Key，新装默认 Voyage `voyage-4`@1024 维，`VOYAGE_API_KEY` 一把钥同时管嵌入和重排 | Voyage $0.06/百万 token；OpenAI $0.13；Google $0.025；OpenRouter $0.02；MiniMax $0.07；Mistral $0.10（官方 providers 文档表）。个人笔记量级通常每月几毛到几美元；本地 Ollama/llama.cpp/LM Studio 方案 $0 但要自己跑模型。开启方式：新脑带 Key 重 init，或按 init 提示 `gbrain init --force --embedding-model voyage:voyage-4 --embedding-dimensions 1024` 在原脑启用（属付费重建索引，先 dry-run 看成本） |
| 重排（结果更准） | 同 Voyage Key，默认 `rerank-3-lite` | $0.02/百万 token（`rerank-3` 为 $0.05） |
| 合成回答 `gbrain think`（带引用+缺口分析） | 聊天模型 Key，默认 Anthropic（实测无 Key 时点名要 ANTHROPIC_API_KEY） | 按所选模型订阅/API 计费；大头是下游读检索结果的成本，见 2.3 模式表。另可用 OpenRouter 一把 Key 打通多家聊天模型 |
| 查询扩展 `--expand` | 同聊天模型 | 每次扩展 = 一次模型调用，单独计费；无 Key 脑永不扩展 |
| 多设备/团队/大规模 | Supabase（Postgres+pgvector）或自托管 Postgres；`gbrain migrate --to supabase` 迁移 | Supabase 免费档可用，Pro $25/月（官方个人脑教程口径：Supabase free to $25）。PGLite 约 50K 页内没必要迁 |
| 全套 24 小时个人 Agent（官方 personal-brain 教程，非 gbrain 本体） | Render 托管 + Supabase + OpenAI/Anthropic Key | 官方估算约 $100–150/月起（Render Pro 约 $85 + Supabase $0–25 + OpenAI $5–20 + Anthropic $50 起、用量浮动）。**教程必须把这笔账和「只要记忆层」的 $0 起步分开讲，这是最容易被骂过度宣传的地方** |
| 官方托管 gbrain.io | 不装任何东西，浏览器选套餐 | 本轮未核价，教程只提「有官方托管可选」，不写具体价格 |

国内用户补充（写进教程加分项）：官方 providers 列表含阿里 DashScope、智谱 Zhipu 两个国内通道（价格以官方文档为准、标注 varies），以及完全本地的 Ollama 路线——不想用海外 Key 的人并非无路可走。

---

## 六、教程大纲（定价 9.9 元）

- 第 0 章 先说清它是什么、不是什么：给现有 AI Agent 加一个你自己掌控的、带出处的长期记忆；不是聊天机器人、不是笔记软件平替。适合谁/不适合谁（见卖点文案）。
- 第 1 章 30 分钟跑通无 Key 版：查 Bun 版本 → 正确安装命令（避开 npm 陷阱）→ init → 补迁移 → doctor 体检。
- 第 2 章 让它记住你：remember 的正确姿势（entity/provenance/content-origin）、import 导入现有笔记、forget 撤回与纠正。
- 第 3 章 跨会话验证：关掉重开、换个 Agent，新会话复述测试——亲手证明「它真记住了」。
- 第 4 章 接进你的 Agent：Claude Code / Codex 的 setup 一条龙、其他 MCP 客户端手动配法、brain-first 为什么必须写。
- 第 5 章 个人知识库搭建：schema 包怎么选、人物/公司/项目三类页面的建法、link 关系与时间线、从我的实测笔记到一个能问答的小知识库。
- 第 6 章 诚实的一章：无 Key 的边界——同义搜不到、去重降级、分数为什么低；三个真实失败截图说话。
- 第 7 章 进阶付费篇：开嵌入（Voyage/国内通道/本地 Ollama 三路线与成本表）、搜索模式怎么选、think 合成回答、迁 Supabase 的时机与步骤、夜间 dream 整理是什么。
- 附录 A 故障急救：#218/wedged 处理流程图、维度不匹配、doctor 分数解读、升级前必读 migrations 说明。
- 附录 B 成本总账：$0 版 / 几美元版 / 全套版三档清单，含「哪些钱是给服务商的、教程不代收」声明。

定价：统一 9.9 元（2026-11-11 用户拍板，全线教程统一价），全内容不分档。

## 七、卖点文案草案

**标题候选**
1. 《让你的 AI 记住你：GBrain 中文实测部署 + 个人知识库搭建》
2. 《给 Claude Code / Codex 装一个不会失忆的大脑——GBrain 小白到进阶全记录》
3. 《每次都要重新自我介绍？给你的 Agent 加长期记忆，30 分钟无 Key 跑通》

**痛点开场（详情页首屏）**
> 你是不是也这样：昨天刚跟 AI 说完的项目背景，今天开新会话又要打一遍；换了一个 Agent，之前攒的偏好、人脉、决定全归零。GBrain 是 YC 总裁 Garry Tan 给自己 Agent 做的记忆层——155,795 页、24,589 个人的生产脑就是它。这份教程是我在干净环境里从安装命令开始、一坑一坑踩完写出来的中文实测版：装错包、迁移卡住、搜不到同义词，每一个坑都有截图和解法。

**你将得到**
- 一条验证过的安装路径（含官方没在 README 首屏讲透的补迁移步骤）
- 一套可照抄的记忆用法：存事实带出处、导入旧笔记、纠正与撤回
- Claude Code / Codex 接入与「新会话复述」验收法
- 一份诚实的成本账：$0 能做到哪、哪一步开始花钱、花给谁
- 故障急救附录：#218、wedged、维度不匹配的正确处理顺序

**不适合你，如果**：你还没在用任何 AI Agent（先有 Agent，再谈记忆）；你期待装完就像电影里的贾维斯（无 Key 版只是关键词记忆，进阶要配 Key 并付服务商的钱）；你只想要现成托管服务（那直接看官方 gbrain.io）。

**合规声明（详情页必带）**：GBrain 为 MIT 开源项目，本教程出售的是作者的实测经验、中文讲解与配套清单，不包含、也不冒充出售软件本身；项目版权归原作者，仓库链接与协议说明见教程内。

## 八、需截图清单（发布前补拍，按章节编号）

1. [1章] `bun --version` 输出（1.4.4）
2. [1章] 安装成功输出，**特意框出 `Blocked 1 postinstall` 那行**（先吓后解，信任感来源）
3. [1章] `gbrain --version`（0.60.645.0，以发布时版本为准重拍）
4. [1章] init 输出中「5 setup migration(s) pending」段落
5. [1章] apply-migrations 五条逐条 complete 的输出
6. [1章] doctor 100/100（空脑）整屏
7. [2章] remember 缺 provenance 报错整屏
8. [2章] remember 成功 `remembered as fact #1` + degraded dedup 提示
9. [2章] import 完成统计（1 pages / 1 chunks / 0 errors）
10. [3章] 新会话/新终端里 search 命中 JSON（框出 keyword_exact）
11. [3章] recall 输出那一行事实
12. [3章] get 读回笔记全文
13. [6章] query 自然语言问句 No results（keyword_zero）——诚实截图，别删
14. [6章] think 无 Key 时的 Gaps 输出（no LLM available）
15. [6章] 导入笔记后 doctor brain_score 25/100 的构成明细
16. [4章]（待补测）Claude Code / Codex 里 setup 完成 + 新会话复述成功的对话截图 —— **发布前必须补，本轮未测**
17. [7章]（待补）`gbrain search modes` 三档对比、开启嵌入前后同一问句的搜索结果对比 —— 需一把真实 Key 后补

## 九、未跑通/未验证清单（不得在教程里冒充已测）

- MCP 接入 Claude Code、Codex 及其他客户端的端到端体验（仅核对官方命令与 serve 参数）。
- 开启嵌入后的语义检索效果、中文语义检索质量（中文向量效果是国内受众最关心的问题，需用 Voyage/DashScope/智谱/Ollama 分别补测后再写结论）。
- `gbrain think` 有 Key 时的合成回答质量与实际单次成本。
- Supabase 迁移全流程与多人/多设备同步。
- dream/autopilot 夜间整理、技能包（skills）体系的实际价值。
- Windows 平台安装差异（本轮仅 Linux 实测；Bun 在 Windows 的路径与 postinstall 行为需另测或在教程中明确标注「以 macOS/Linux 为准」）。

---
实测原始日志存档（与本草稿同目录的隐藏测试目录）：`~/workspace/github-trending/.gbrain-test/`（install.log、init.log、apply-migrations.log、doctor-after-init.log、session1-*.log、session2-*.log）。测试脑为隔离沙箱，未动用户真实 `~/.gbrain`，未在 GitHub 做任何写操作。
