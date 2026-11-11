# 第一轮 8 个候选总汇总（2026-11-10）

| # | 项目 | 状态 | 草稿 | 建议定价 | 上架前还差什么 |
|---|------|------|------|----------|----------------|
| 1 | homeassistant-ai/ha-mcp | ✅ 真 HA 全链路跑通（读写+重启持久性） | ha-mcp-draft.md | 9.9 | 云端客户端实连截图（需真实账号+公网域名）、Docker/App 路径补图 |
| 2 | vercel-labs/agent-browser | ✅ 真跑通（直连下载失败实锤，镜像方案跑通，最小场景闭环） | agent-browser-draft.md | 9.9 | Win/Mac 真机各补测一台，才能写「全平台实测」 |
| 3 | we-promise/sure | ⚠️ 文档+源码核实版（沙箱无 Docker，未真机部署） | sure-draft.md | 9.9 | 在 NAS 或 VPS 真跑一遍；支付宝/微信账单 CSV 字段映射表 |
| 4 | anefzaoui/misterdownloadmanager | ⚠️ 文档核实版（Windows 软件，Linux 无法真机测），已升至 v0.5.2 | misterdownloadmanager-draft.md | 9.9 | Windows 真机复核清单 11 项；版本迭代极快，需跟版更新 |
| 5 | garrytan/gbrain | ✅ 真跑通（无 Key 本地模式，跨会话记忆验证） | gbrain-draft.md | 9.9 | 中文语义效果、接 Claude Code 端到端补测 |
| 6 | milind-soni/OpenMausBot | ◐ 部分跑通（npx 服务+配对真跑通，Docker 路线仅文件核验） | openmausbot-draft.md | 9.9 | 真 VPS 补测 Docker+Caddy+真机配对；默认 Full access 必须改审批 |
| 7 | sherlock-project/sherlock | ✅ 真跑通，改免费引流（不做付费） | sherlock-free-content-draft.md | 免费 | 发布前合规自查；演示只用公开测试用户名 |
| 8 | anomalyco/hex | ⏸️ 暂缓（中文听写实测优秀，但桌面端无真机可测，Win/Linux 仍 Beta） | hex-verification.md | 9.9 | Apple 芯片 Mac/Windows 真机跑完桌面流程，或 Windows 脱离 Beta/完成签名 |

## 推荐上架顺序
1. 先上 1号 ha-mcp（成色最好、增速最快、中文空白最干净）
2. 次上 5号 gbrain 或 2号 agent-browser（同为 AI 进阶付费人群，可互相导流）
3. 4号 misterdownloadmanager 要快（热度窗口短），但必须先过 Windows 真机复核
4. 3号 Sure 是长尾款，不急，等有 NAS/VPS 真机环境补测后上
5. 7号 sherlock 免费内容可随时先发，为付费款引流

## 需要用户决定的
- 先做哪一个的上架定稿（把草稿扩成完整图文教程）
- 上架平台：知识星球还是小报童（或两个都挂）
- 能否提供真机环境：Windows 电脑（4号、2号）、NAS/VPS（3号、6号）、Apple 芯片 Mac（8号）
